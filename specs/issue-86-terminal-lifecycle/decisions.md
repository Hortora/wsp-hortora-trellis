# Decisions — Issue #86: Terminal Lifecycle

## D1: Configure Dedup Location

**Choice:** Guard inside `pages-component-terminal.configure()` — skip teardown+reinit if wsUrl hasn't changed
**Alternatives:**
- Per-consumer guards (tab-group, repo-detail, pair-view each track last config) — scatters fix, misses future consumers
**Rationale:** The problem is that `configure()` is destructive when called with identical parameters. Fixing this at the source protects all consumers.
**Trade-offs:** Modifying a vendored component — must be maintained across upstream updates
**Sources:** terminal-tab-group.ts (no dedup), terminal-pair-view.ts (_configuredPrimary pattern), repo-detail.ts (_lastTerminalName pattern)
**Exploration:** quick
**Status:** captured

## D2: Terminal Identity vs Agent State Separation

**Choice:** Split `_snapshot` into `_terminalName` (string, drives configure) and `_agentState` (object, drives sidebar)
**Alternatives:**
- Lit `guard()` directive on terminal template — couples to Lit API, doesn't fix data model
**Rationale:** Terminal identity is stable (changes on create/destroy). Agent state is volatile (changes every 5s poll). Conflating them causes the terminal DOM to churn on every agent state update.
**Trade-offs:** More state properties to manage in repo-detail and slot-detail
**Sources:** repo-detail.ts:_loadTerminal, workspace-sse.ts agent:state topic
**Exploration:** quick
**Status:** captured

## D3: Focus Restoration Mechanism

**Choice:** `focusin`/`focusout` tracking on the repo-detail host — click inside `.terminal-area` always calls `_focusTerminal()`
**Alternatives:**
- Global focus trap — blocks sidebar interaction, needs exclusion lists
**Rationale:** Standard DOM focus events work through shadow DOM. No coordinate math, no document-level listeners. Sidebar buttons remain clickable; terminal regains focus on click.
**Trade-offs:** Focus isn't automatically restored — user must click the terminal area. But this matches standard terminal UX (click to focus).
**Sources:** repo-detail.ts:_mousedownHandler (current bounds-checking approach)
**Exploration:** quick
**Status:** captured

## D4: refreshAgent Atomicity

**Choice:** Acquire `lockFor(terminalName)` + call `verifyShellForeground` after `treeKill`, drop the hardcoded 500ms sleep
**Alternatives:**
- Increase sleep to 1000ms with retry — still a guess, doesn't adapt to actual readiness
**Rationale:** `verifyShellForeground` already exists and polls up to 5x at 200ms. Reusing it replaces guessing with verification. Lock prevents concurrent operations.
**Trade-offs:** Refresh takes slightly longer (up to 1s polling vs 500ms fixed) on slow systems
**Sources:** AgentProcessManager.java:verifyShellForeground, AgentProcessManager.java:refreshAgent, AgentProcessManager.java:lockFor
**Exploration:** quick
**Status:** captured

## D5: WebSocket Auto-Reconnect

**Choice:** Reconnect on abnormal close (not 1000, not 4001) with exponential backoff — 1s, 2s, 4s, max 3 attempts
**Alternatives:**
- Reconnect on all closes including session-takeover — two tabs would fight in a loop
**Rationale:** Code 1000 is intentional teardown. Code 4001 is session-takeover (another tab). All other codes are failures worth retrying. Three attempts cap prevents infinite loops against a dead sidecar.
**Trade-offs:** 3 attempts may not be enough for a sidecar that takes >7s to restart. But the sidecar health check takes <2s normally.
**Sources:** TerminalWebSocket.java:onOpen (session takeover code 4001), pages-component-terminal configure/_teardown
**Exploration:** quick
**Status:** captured
