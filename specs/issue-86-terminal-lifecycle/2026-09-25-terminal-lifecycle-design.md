# Terminal Lifecycle Stability — Design Spec

**Issue:** Hortora/trellis#86
**Date:** 2026-09-25
**Status:** Draft

## Problem

The repo-detail modal terminal has three interacting failure modes: the terminal element is destroyed during Lit re-renders (losing the WebSocket connection), keyboard focus is lost after any non-terminal interaction, and agent refresh can spawn duplicate processes. The terminal is the primary interaction surface — these failures make it unreliable.

## Root Causes

1. `pages-component-terminal.configure()` is destructive — tears down the WebSocket and xterm instance even when called with identical parameters. `terminal-tab-group` calls it on every SSE event (no dedup guard).
2. `_loadTerminal()` conflates terminal identity (stable) with agent state (volatile). Every agent poll creates a new snapshot object reference, triggering Lit re-renders that churn the terminal DOM.
3. Focus management relies on a `document.addEventListener('mousedown')` with coordinate bounds-checking. This misses clicks through shadow DOM boundaries and doesn't restore focus after sidebar interactions.
4. `refreshAgent()` uses a hardcoded 500ms sleep instead of verifying the shell is ready, holds no lock, and can produce concurrent `claude -c` commands.
5. `pages-component-terminal` has no WebSocket reconnection — if the pipe drops, the terminal goes dark permanently.

## Solution

Three layers, each independently testable.

### Layer 1 — `pages-component-terminal` (vendored component)

**Configure guard:** Inside `configure()`, compare the incoming `wsUrl` with the currently active config. If unchanged, return immediately — no teardown, no reinit. This protects all consumers (repo-detail, slot-detail, terminal-tab-group, terminal-pair-view) in one place.

```
configure(props):
  if (this._wsUrl === props.wsUrl && this._terminal):
    return  // already configured with same URL
  this._teardown()
  this._props = props
  this._init()
```

**Auto-reconnect:** On WebSocket `onclose`, check the close code:
- `1000` (normal close via `_teardown()`) — do not reconnect
- `4001` (session-takeover — another tab took over) — do not reconnect
- Any other code — reconnect with exponential backoff: 1s, 2s, 4s, max 3 attempts

On reconnect, call `_connect()` which creates a new WebSocket. The server's `onOpen` sets up a fresh pipe-pane with the resize-redraw trick, so the terminal gets a full screen refresh including current tmux pane content.

### Layer 2 — Frontend consumers

**State separation in repo-detail and slot-detail:**

Replace `_snapshot: AgentSnapshot` with two properties:
- `_terminalName: string` — drives terminal element configure. Changes only on terminal create/destroy.
- `_agentState: { state: string, memoryMb: number, lastError: string | null } | null` — drives sidebar badges/buttons. Changes on every SSE poll.

`_loadTerminal()` fetches the snapshot, extracts the terminal name and agent state separately, and only updates each property when the value actually changed:
```
const next = filtered[0] ?? null;
const name = next?.terminalName ?? '';
const state = next?.process ? { state: next.process.state, ... } : null;
if (name !== this._terminalName) this._terminalName = name;
if (JSON.stringify(state) !== JSON.stringify(this._agentState)) this._agentState = state;
```

The terminal template keys on `_terminalName`. The sidebar keys on `_agentState`. No cross-contamination.

**Focus restoration:**

Replace the `document.addEventListener('mousedown', bounds-check)` pattern with a simple `@click` handler on `.terminal-area`:

```html
<div class="terminal-area" @click=${() => this._focusTerminal()}>
  <pages-component-terminal ...></pages-component-terminal>
</div>
```

Remove the document-level mousedown listener entirely. The `@click` on the terminal area is simpler, works through shadow DOM, and doesn't need coordinate math. Add `_focusTerminal()` calls after `_agentAction` completes (already landed in commit eb5a14d).

This matches standard terminal UX — click the terminal to focus it.

**terminal-tab-group benefits passively** from Layer 1's configure guard. No consumer-level changes needed beyond the state split that slot-detail inherits.

### Layer 3 — `AgentProcessManager.refreshAgent()` (Java backend)

Add locking and shell verification:

```java
public void refreshAgent(String name) throws IOException, InterruptedException {
    var lock = lockFor(name);
    lock.lock();
    try {
        var existing = agents.get(name);
        if (existing == null || existing.state() != AgentState.RUNNING) return;
        setStarting(name, "claude -c");
        if (existing.pid() > 0) {
            treeKill(name, existing.pid());
        }
        verifyShellForeground(name);  // polls up to 5x at 200ms — replaces Thread.sleep(500)
        tmux.sendKeys(name, "claude -c\n");
    } finally {
        lock.unlock();
    }
}
```

Changes from current code:
- `lock.lock()` / `finally { lock.unlock(); }` — prevents concurrent refresh/poll/stop operations
- `verifyShellForeground(name)` replaces `Thread.sleep(500)` — adapts to actual shell readiness
- Lock prevents two concurrent refresh calls from both sending `claude -c`

## Testing

### Layer 1 — `pages-component-terminal`
- Unit: `configure()` twice with same wsUrl does not call `_teardown()`
- Unit: `onclose` code 1006 triggers reconnect; codes 1000 and 4001 do not
- Unit: reconnect backs off 1s → 2s → 4s, stops after 3 attempts

### Layer 2 — Frontend consumers
- Vitest: repo-detail renders terminal when `_terminalName` set, doesn't re-render when only `_agentState` changes
- Vitest: sidebar badge updates when `_agentState` changes
- Playwright e2e: click sidebar button → click terminal area → verify focus restored

### Layer 3 — Backend
- JUnit: `refreshAgent` calls `verifyShellForeground` after kill (verify call order)
- JUnit: concurrent `refreshAgent` calls — second blocks on lock, doesn't send duplicate `claude -c`

## Implementation Order

1. Layer 1 first (configure guard + reconnect) — eliminates the #1 source of terminal destruction
2. Layer 3 next (refreshAgent atomicity) — eliminates duplicate process spawning
3. Layer 2 last (state separation + focus) — cleans up the remaining re-render and focus issues

Each layer is independently deployable and testable.

## References

- repo-detail.ts — current terminal lifecycle, _loadTerminal, _focusTerminal, updated()
- terminal-tab-group.ts — no configure dedup (worst offender)
- TerminalWebSocket.java — WebSocket endpoint, pipe-pane, session takeover (code 4001)
- AgentProcessManager.java — refreshAgent, verifyShellForeground, lockFor
- pages-component-terminal — vendored, configure/_teardown/_init/_connect
- repo-terminal-integration design spec — original repo-detail architecture
