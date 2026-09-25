# Terminal Lifecycle Stability — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #86 — Repo-detail modal terminal lifecycle fragility
**Issue group:** #86

**Goal:** Make the terminal in repo-detail/slot-detail reliable — no disappearing terminals, no lost keyboard, no duplicate agent spawning.

**Architecture:** Three independent layers. Layer 1 fixes the vendored `pages-component-terminal` (configure guard + reconnect cap). Layer 2 separates terminal identity from agent state in the frontend consumers. Layer 3 adds locking and verification to the backend `refreshAgent`.

**Tech Stack:** TypeScript (Lit 3, xterm.js), Java 21 (Quarkus 3.x), vitest, JUnit 5

## Global Constraints

- Vendored component at `.casehub-packages/packages/pages-component-terminal/dist/PagesTerminal.js` — edit the dist JS directly (no TypeScript source available)
- `pages-component-terminal` already has reconnect with exponential backoff — just needs configure guard and max attempts
- All frontend tests use vitest + happy-dom
- Java tests use JUnit 5 + AssertJ + @QuarkusTest where needed

---

## Batch 1: Component Guard + Reconnect Cap

### Task 1: Configure guard in pages-component-terminal

**Files:**
- Modify: `.casehub-packages/packages/pages-component-terminal/dist/PagesTerminal.js:15-21`
- Test: `src/components/pages-terminal-configure.test.ts` (create)

**Interfaces:**
- Consumes: nothing
- Produces: `PagesTerminal.configure(props)` skips teardown when `wsUrl` unchanged

- [ ] **Step 1: Write the failing test**

Create `sidecar/src/main/webui/src/components/pages-terminal-configure.test.ts`:
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';

describe('PagesTerminal configure guard', () => {
  let el: any;

  beforeEach(async () => {
    await import('@casehubio/pages-component-terminal');
    el = document.createElement('pages-component-terminal');
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  it('does not teardown when configure called with same wsUrl', () => {
    const props = {
      wsUrl: 'ws://localhost:9777/ws/terminal/test/{cols}/{rows}',
      theme: { background: '#1e1e1e' },
      fontSize: 13,
    };
    el.configure(props);
    const terminal = el._terminal;
    el.configure(props);
    expect(el._terminal).toBe(terminal);
  });

  it('tears down and reinits when wsUrl changes', () => {
    el.configure({
      wsUrl: 'ws://localhost:9777/ws/terminal/test-a/{cols}/{rows}',
      fontSize: 13,
    });
    const terminal1 = el._terminal;
    el.configure({
      wsUrl: 'ws://localhost:9777/ws/terminal/test-b/{cols}/{rows}',
      fontSize: 13,
    });
    expect(el._terminal).not.toBe(terminal1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/components/pages-terminal-configure.test.ts`
Expected: first test FAILS — `_terminal` is a different instance because configure always tears down

- [ ] **Step 3: Implement the configure guard**

Edit `PagesTerminal.js` — replace the `configure` method:

Current (lines 15-21):
```javascript
configure(props) {
    this._props = props;
    if (this._connected) {
        this._teardown();
        this._init();
    }
}
```

New:
```javascript
configure(props) {
    if (this._props?.wsUrl === props.wsUrl && this._terminal) {
        return;
    }
    this._props = props;
    if (this._connected) {
        this._teardown();
        this._init();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/components/pages-terminal-configure.test.ts`
Expected: both tests PASS

- [ ] **Step 5: Commit**

```bash
git add .casehub-packages/packages/pages-component-terminal/dist/PagesTerminal.js
git add src/components/pages-terminal-configure.test.ts
git commit -m "feat(#86): configure guard in pages-component-terminal — skip teardown on same wsUrl Refs #86"
```

### Task 2: Reconnect max attempts cap

**Files:**
- Modify: `.casehub-packages/packages/pages-component-terminal/dist/PagesTerminal.js:127-141`
- Modify: `src/components/pages-terminal-configure.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `_scheduleReconnect()` stops after 3 attempts

- [ ] **Step 1: Write the failing test**

Add to `pages-terminal-configure.test.ts`:
```typescript
describe('reconnect cap', () => {
  it('stops reconnecting after 3 attempts', () => {
    el.configure({
      wsUrl: 'ws://localhost:9777/ws/terminal/test/{cols}/{rows}',
      fontSize: 13,
    });
    el._retries = 3;
    el._scheduleReconnect();
    expect(el._reconnectTimer).toBeUndefined();
  });

  it('allows reconnect when retries below cap', () => {
    el.configure({
      wsUrl: 'ws://localhost:9777/ws/terminal/test/{cols}/{rows}',
      fontSize: 13,
    });
    el._retries = 2;
    el._scheduleReconnect();
    expect(el._reconnectTimer).not.toBeUndefined();
    clearTimeout(el._reconnectTimer);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/components/pages-terminal-configure.test.ts`
Expected: first test FAILS — `_reconnectTimer` is defined (no cap exists)

- [ ] **Step 3: Add max attempts guard**

Edit `PagesTerminal.js` `_scheduleReconnect` method — add early return at the top:

```javascript
_scheduleReconnect() {
    if (this._retries >= 3) {
        this._dispatchEvent("terminal-disconnected", { reason: "max-retries" });
        return;
    }
    const delay = Math.min(1000 * Math.pow(2, this._retries), 30000);
    // ... rest unchanged
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/components/pages-terminal-configure.test.ts`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add .casehub-packages/packages/pages-component-terminal/dist/PagesTerminal.js
git add src/components/pages-terminal-configure.test.ts
git commit -m "feat(#86): cap reconnect at 3 attempts in pages-component-terminal Refs #86"
```

## Batch 2: Backend — refreshAgent Atomicity

### Task 3: Lock and verifyShellForeground in refreshAgent

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentProcessManager.java:194-206`
- Modify: `sidecar/src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java`

**Interfaces:**
- Consumes: existing `lockFor()`, `verifyShellForeground()`, `treeKill()` methods
- Produces: `refreshAgent()` that is atomic and verified

- [ ] **Step 1: Write the failing test**

Add to `AgentProcessManagerTest.java`:
```java
@Test
void refreshAgentCallsVerifyShellForegroundAfterKill() throws Exception {
    var terminal = new TerminalInfo("t1", "/tmp", null, null, null, null);
    manager.setStarting("t1", "claude");
    manager.pollTerminalWithPsOutput(terminal, "12345 12344 200000 claude");

    var callOrder = new java.util.ArrayList<String>();
    when(tmux.getPaneCommand("t1")).thenAnswer(inv -> {
        callOrder.add("getPaneCommand");
        return "zsh";
    });

    manager.refreshAgent("t1");
    assertThat(callOrder).contains("getPaneCommand");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=AgentProcessManagerTest#refreshAgentCallsVerifyShellForegroundAfterKill -Dsurefire.useFile=false`
Expected: FAIL — current refreshAgent uses Thread.sleep, not verifyShellForeground

- [ ] **Step 3: Implement the fix**

Use `ide_replace_text_in_file` on `AgentProcessManager.java`:

Replace:
```java
public void refreshAgent(String terminalName) throws IOException, InterruptedException {
    var existing = agents.get(terminalName);
    if (existing == null || existing.state() != AgentState.RUNNING) {
        throw new IllegalStateException("Cannot refresh agent in state: " +
                                        (existing != null ? existing.state() : "IDLE"));
    }
    setStarting(terminalName, "claude -c");
    if (existing.pid() > 0) {
        treeKill(terminalName, existing.pid());
    }
    try {Thread.sleep(500);} catch (InterruptedException ignored) {}
    tmux.sendKeys(terminalName, "claude -c\n");
}
```

With:
```java
public void refreshAgent(String terminalName) throws IOException, InterruptedException {
    var lock = lockFor(terminalName);
    lock.lock();
    try {
        var existing = agents.get(terminalName);
        if (existing == null || existing.state() != AgentState.RUNNING) {
            throw new IllegalStateException("Cannot refresh agent in state: " +
                                            (existing != null ? existing.state() : "IDLE"));
        }
        setStarting(terminalName, "claude -c");
        if (existing.pid() > 0) {
            treeKill(terminalName, existing.pid());
        }
        verifyShellForeground(terminalName);
        tmux.sendKeys(terminalName, "claude -c\n");
    } finally {
        lock.unlock();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=AgentProcessManagerTest -Dsurefire.useFile=false`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/agent/AgentProcessManager.java
git add sidecar/src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java
git commit -m "feat(#86): refreshAgent atomicity — lock + verifyShellForeground Refs #86"
```

## Batch 3: Frontend — State Separation + Focus

### Task 4: Split _snapshot into _terminalName + _agentState in repo-detail

**Files:**
- Modify: `sidecar/src/main/webui/src/views/repo-detail.ts`

**Interfaces:**
- Consumes: `/api/terminals` REST response (AgentSnapshot[])
- Produces: `_terminalName` (string) drives terminal rendering; `_agentState` (object) drives sidebar

- [ ] **Step 1: Replace state declarations**

In `repo-detail.ts`, use `ide_replace_text_in_file`:

Replace:
```typescript
@state() private _snapshot: AgentSnapshot | null = null;
@state() private _snapshots: AgentSnapshot[] = [];
```

With:
```typescript
@state() private _terminalName = '';
@state() private _agentState: { state: string; memoryMb: number; lastError: string | null } | null = null;
@state() private _snapshots: AgentSnapshot[] = [];
```

- [ ] **Step 2: Update _loadTerminal to split concerns**

Replace `_loadTerminal`:
```typescript
private async _loadTerminal() {
    try {
      const params = new URLSearchParams();
      if (this.repoName) params.set('repo', this.repoName);
      const res = await fetch(`/api/terminals?${params}`);
      if (!res.ok) return;
      const all: AgentSnapshot[] = await res.json();
      const filtered = all.filter(s => !s.terminal.slot);
      this._snapshots = filtered;
      const next = filtered[0] ?? null;
      const name = next?.terminalName ?? '';
      if (name !== this._terminalName) {
        this._terminalName = name;
      }
      const agentState = next?.process
        ? { state: next.process.state, memoryMb: Math.round(next.process.memoryBytes / (1024 * 1024)), lastError: next.lastError }
        : null;
      const prev = this._agentState;
      if (agentState?.state !== prev?.state || agentState?.memoryMb !== prev?.memoryMb || agentState?.lastError !== prev?.lastError) {
        this._agentState = agentState;
      }
    } catch { /* ignore */ }
  }
```

- [ ] **Step 3: Update updated() to use _terminalName**

Replace the `_snapshot` check in `updated()`:
```typescript
if (changed.has('_terminalName') && this._terminalName &&
    this._terminalName !== this._lastTerminalName) {
  this.updateComplete.then(() => {
    const el = this.renderRoot.querySelector('#repo-terminal') as any;
    if (el) {
      this._lastTerminalName = this._terminalName;
      const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
      el.configure({
        wsUrl: `${proto}//${location.host}/ws/terminal/${this._terminalName}/{cols}/{rows}`,
        theme: { background: '#1e1e1e', foreground: '#cccccc', cursor: '#aeafad' },
        fontSize: 13,
        fontFamily: "'JetBrains Mono', 'Fira Code', 'Cascadia Code', monospace",
      });
    }
  });
}
```

- [ ] **Step 4: Update render to use _terminalName and _agentState**

Update the template conditional from `this._snapshot` to `this._terminalName`. Update sidebar to use `this._agentState` instead of `this._snapshot.process`.

- [ ] **Step 5: Run all frontend tests**

Run: `npx vitest run`
Expected: all existing tests PASS (pair-view tests don't depend on repo-detail internals)

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/webui/src/views/repo-detail.ts
git commit -m "feat(#86): split _snapshot into _terminalName + _agentState in repo-detail Refs #86"
```

### Task 5: Simplify focus — replace mousedown with @click

**Files:**
- Modify: `sidecar/src/main/webui/src/views/repo-detail.ts`

**Interfaces:**
- Consumes: `_focusTerminal()` method (existing)
- Produces: click on terminal area always focuses the terminal

- [ ] **Step 1: Remove document mousedown listener setup from updated()**

Remove the `_mousedownHandler` setup from the `updated()` method — the entire block that creates the handler, removes the old one, and adds it to `document`.

- [ ] **Step 2: Remove _mousedownHandler field and disconnectedCallback cleanup**

Remove the `_mousedownHandler` private field and its cleanup in `disconnectedCallback`.

- [ ] **Step 3: Verify the @click handler is already on .terminal-area**

The terminal area div already has `@click=${() => this._focusTerminal()}` from a prior commit. Verify this is present. If the paired path also needs it, add there too.

- [ ] **Step 4: Run full test suite**

Run: `npx vitest run`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/src/views/repo-detail.ts
git commit -m "feat(#86): simplify focus — remove document mousedown, keep @click on terminal-area Refs #86"
```

### Task 6: Apply state separation to slot-detail

**Files:**
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts`

**Interfaces:**
- Consumes: `/api/terminals` REST response
- Produces: sidebar updates independently from terminal element

- [ ] **Step 1: Update _loadTerminals to avoid unnecessary snapshot array replacement**

In `slot-detail.ts`, modify `_loadTerminals` to compare terminal names before replacing `_snapshots`:
```typescript
private async _loadTerminals() {
    try {
      const params = new URLSearchParams();
      params.set('slot', String(this.slotNumber));
      const res = await fetch(`/api/terminals?${params}`);
      if (!res.ok) return;
      const next: AgentSnapshot[] = await res.json();
      const prevNames = this._snapshots.map(s => s.terminalName).join(',');
      const nextNames = next.map(s => s.terminalName).join(',');
      if (prevNames !== nextNames) {
        this._snapshots = next;
      }
    } catch { /* ignore */ }
  }
```

This prevents the `tabs` array from being recreated on every SSE event, which (combined with the Layer 1 configure guard) stops `terminal-tab-group` from tearing down the terminal.

- [ ] **Step 2: Run tests**

Run: `npx vitest run`
Expected: all tests PASS

- [ ] **Step 3: Commit**

```bash
git add sidecar/src/main/webui/src/views/slot-detail.ts
git commit -m "feat(#86): slot-detail — skip snapshot update when terminal names unchanged Refs #86"
```

## References

- [2026-09-25-terminal-lifecycle-design.md] — design spec this plan implements
- [repo-detail.ts] — current terminal lifecycle
- [terminal-tab-group.ts] — no configure dedup
- [PagesTerminal.js] — vendored component with existing reconnect logic
- [AgentProcessManager.java] — refreshAgent, verifyShellForeground, lockFor
- [TerminalWebSocket.java] — WebSocket session takeover (code 4001)
- [GitHub #86] — tracking issue
