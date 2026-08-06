# Agent Control Plane — Programmatic Observation and Interaction API

**Issue:** Hortora/trellis#27
**Date:** 2026-08-06
**Status:** Approved

## Problem

A Trellis installation will have its own coordinating agent. That agent needs
to observe and interact with everything a human can — navigate between panels,
query workspace/repo/slot/terminal metadata, read terminal scrollback, and
send input to any terminal. The terminal should not know whether a human or
agent is interacting with it.

The coordinating agent is the primary consumer but the API must also be usable
by external MCP clients. The design must accommodate growth: new entity types,
richer orchestration, and capabilities that don't exist yet — without
redesigning the API surface.

## Solution

A semantic model of the entire application state — backend and frontend —
exposed through a small set of MCP tools embedded in the Quarkus sidecar.
The model is derived from existing Java records and TypeScript interfaces at
runtime. No separate model definition files, no code generation, no drift.

The model is the extension point. As Trellis grows new capabilities, the model
grows new entity types and actions. The MCP tool surface stays stable.

**Library:** `quarkus-mcp-server` — embedded in the existing sidecar process.
MCP tools are CDI beans that inject existing service beans directly. Zero
serialisation overhead, single process.

## §1 Data Model

The application state is a single tree. Every UI element, every backend entity,
every available action is a node. The tree is assembled at query time from the
actual service beans and the pushed frontend UI state.

### Tree Structure

```
trellis
├── dock-bar
│   ├── dashboard    {active: true}
│   ├── workspace    {active: false}
│   ├── artifacts    {active: false}
│   ├── protocols    {active: false}
│   ├── garden       {active: false}
│   └── memory       {active: false}
├── panels
│   ├── dashboard
│   │   ├── repo-cards: [{name, branch, remoteUrl, actions: [...]}]
│   │   ├── slot-cards: [{number, issue, status, repos, actions: [...]}]
│   │   └── epic-cards: [{...}]
│   ├── workspace-view
│   │   └── frames: [{id, name, order, position, size, pinned, zIndex,
│   │                  tabs: [{terminalName, type, active}],
│   │                  actions: [close, pin, detach, save-as-group]}]
│   ├── artifacts
│   │   └── files: [{path, type, actions: [open]}]
│   ├── protocols
│   │   └── repos: [{name, entries: [{id, title, actions: [view, remove]}]}]
│   ├── garden
│   │   └── results: [{id, title, domain, actions: [view]}]
│   └── memory
│       └── entries: [{name, type, actions: [view]}]
├── terminals
│   └── [{name, workingDir, slot, repo, issue,
│          agent: {state, pid, memoryBytes, startedAt, command, lastError},
│          sessionLog: "/path/to/log",
│          actions: [send-input, read-log, start-agent, stop-agent,
│                    pause-agent, resume-agent, refresh-agent, destroy]}]
└── workspace
    ├── root: "/path/to/workspace"
    ├── repos: [{name, path, branch, remoteUrl}]
    ├── slots: [{number, path, issue, status, isEpic, repos}]
    ├── pauses: [{branch, issue, timestamp}]
    └── epics: [{...}]
```

### Node Properties

Each node has:
- **Identity** — a path in the tree (e.g. `terminals/engine`,
  `panels/workspace-view/frames/0/tabs/1`)
- **State** — current values, typed per node kind
- **Actions** — what can be done here, with parameter descriptions
- **Children** — nested nodes

The model is read-only data + action declarations. It never contains
behaviour — it is a snapshot of "what exists and what you can do with it."

### Composite Ownership

Backend state (terminals, agents, workspace, repos, slots) is authoritative
in the sidecar. Frontend state (dock-bar active panel, frame positions, tab
focus, panel content) is pushed from the Electron frontend to the sidecar
via `POST /api/model/ui-state` whenever it changes. The sidecar assembles
the composite tree on query.

### No Separate Model Definition

The Java records (`TerminalInfo`, `AgentSnapshot`, `WorkspaceModel`,
`RepoInfo`, `SlotInfo`) ARE the schema. Add a field to `TerminalInfo`, the
model grows. Add a new record type, wire it into the assembly method, the
model grows. The TypeScript interfaces (`FrameLayout`, `ShellLayout`,
`TabRef`) define the frontend state shape — whatever the frontend pushes,
the model returns.

## §2 Session Logs

Every terminal's output is appended to a log file on disk as it streams
through the WebSocket relay. The coordinating agent reads these to understand
what happened in any terminal — full history, not just what's visible.

### Write Path

`TerminalWebSocket` already reads from the FIFO and forwards to the WebSocket
client. Add a tee: every byte read from the FIFO also appends to
`{sidecar-data-dir}/sessions/{terminalName}.log`. Append-only. The file grows
for the lifetime of the tmux session.

### Encoding

Raw terminal output including ANSI escape sequences. No stripping, no parsing.
LLMs handle ANSI fine. Clean text is a presentation concern, not a storage
concern.

### Read Path

The MCP `trellis_terminal` tool with `operation=read-log` reads the file.
Parameters: `name` (terminal name), `lines` (last N lines, default 50),
`offset` (start from line N for pagination). A "line" is `\n`-delimited —
raw terminal output including `\r` and ANSI sequences within a line is
preserved. Returns raw text.

Implementation uses `RandomAccessFile` seeking backwards from the end of
the file to locate the requested line boundaries. Read cost is O(bytes
returned), not O(file size).

### Input Logging

Terminal echo handles most cases — input typed via WebSocket or sent via
`sendKeys` appears in the pipe-pane output and thus in the session log
whenever the terminal application echoes it. This covers normal shell
interaction.

For MCP-initiated input (`trellis_terminal` `send-input`), the sidecar
also writes a bracketed marker directly to the log file before sending the
keys to tmux: `\x1b[?2004h<input text>\x1b[?2004l`. This ensures the
coordinating agent can always reconstruct what it sent, even when the
terminal is in raw mode or suppresses echo (password prompts, curses
applications). Human-initiated input via WebSocket is NOT separately
logged — it flows through the terminal echo path only.

### Lifecycle

Log files are created on first WebSocket data write or first MCP
`send-input` operation. They survive terminal reconnection (append
continues).

Cleanup on terminal destroy: `TerminalRegistry.destroySession()` deletes
the log file at `{sidecar-data-dir}/sessions/{terminalName}.log`. The
`TerminalInfo` record tracks the log file path. Configurable retention
via `trellis.session-log.retain-after-destroy` (duration, default: `0s`
— delete immediately). Non-zero values move the file to
`{sidecar-data-dir}/sessions/archive/` with a TTL-based cleanup
scheduler.

## §3 Frontend State Push

The sidecar owns backend state. The full model includes UI state — which
panel is active, frame positions, tab focus, what each panel is showing.
The frontend pushes this to the sidecar so the model is composite.

### Endpoint

`POST /api/model/ui-state` — the frontend posts its current UI state as
a JSON object whenever it changes. The sidecar stores it in memory (not
persisted — UI state is ephemeral, reconstructed on reconnect).

### Shape

```typescript
interface UIState {
  activePanel: string;
  panels: Record<string, PanelState>;
}

interface PanelState {
  visible: boolean;
  content: unknown;
  lastPushed: number; // Date.now() at push time
}
```

`content` is `unknown` deliberately. Each panel decides what to expose. The
workspace view pushes its frames, tabs, positions. The artifacts panel pushes
its file list. The dashboard pushes its card states. The sidecar does not parse
or validate `content` — it stores and serves it as-is. When a panel adds new
state, the frontend push grows automatically. No backend change needed.

The sidecar enforces a 64KB size limit on serialised `content` per panel.
Pushes exceeding this are rejected with HTTP 413. The `lastPushed` timestamp
lets MCP consumers detect stale UI state — if a panel hasn't pushed in the
last 30 seconds, the model response flags it as `stale: true`.

### Push Trigger

Debounced — same cadence as layout persistence (1s debounce, 5s max-wait).
Panel switches push immediately (no debounce).

### Multi-Window

Each `BrowserWindow` pushes its own UI state, keyed by window ID. The model
tree includes all windows.

### Data Flow

Unidirectional: frontend pushes, sidecar serves. The sidecar never queries
the frontend.

## §4 MCP Tools

Six `@Tool` methods on a single CDI bean. The bean injects existing service
beans (`TerminalRegistry`, `AgentProcessManager`, `WorkspaceScanner`,
`SlotAgentCoordinator`). Return types are the existing Java records —
serialised directly, no adapter layer.

The tool list is the capability discovery layer — "raggable MCP." At
connection time the LLM sees 6 tool names with brief descriptions (cheap
context). When it needs to interact with a specific part of Trellis, it
calls `trellis_model` to discover available actions on the nodes it cares
about. Then it calls the appropriate category tool.

As Trellis grows new capabilities, the tool list stays at 6. New capabilities
appear in the model's action declarations, discovered on demand. Context cost
stays constant.

### Tool Definitions

**`trellis_model(path?)`** — Query application state and discover available
actions. No path → top-level summary (panels, terminal count, workspace root).
`path=terminals` → all terminals with agent state. `path=terminals/engine` →
that terminal's full state including available actions with parameter
descriptions. Excludes heavy content (session logs) from responses.

**`trellis_navigate(target)`** — Activate a UI element (panel, frame, tab).
`target` is a model path. Pushes a `control:navigate` SSE event to the
frontend with a correlation ID. The tool blocks until the frontend
acknowledges the navigation by pushing updated UI state tagged with the
same correlation ID, or until a 5-second timeout expires. Returns the
post-navigation state of the target node on success, or an error with
`status: "timeout"` if the frontend does not acknowledge in time.

**`trellis_terminal(name, operation, params?)`** — Terminal I/O. Operations:
`read-log` (params: lines, offset), `send-input` (params: text), `create`
(params: workingDir, repo, slot), `destroy`, `resize` (params: cols, rows).
Delegates to `TerminalRegistry`. All tmux operations propagate exit code
failures — `TmuxManager.run()` throws on non-zero exit, and the MCP tool
returns the error to the caller. No silent swallowing.

**`trellis_agent(terminal, operation, params?)`** — Agent lifecycle.
Operations: `start` (params: command, model, etc.), `stop`, `pause`,
`resume`, `refresh`, `stats`, `tree`. Delegates to `AgentProcessManager`.

**`trellis_lifecycle(operation, params?)`** — Slot and workspace lifecycle.
Operations: `slot-pause`, `slot-resume`, `slot-end`, `slot-create`,
`workspace-scan`. Delegates to `SlotAgentCoordinator` and
`LifecycleManager`.

**`trellis_workspace(path?)`** — Workspace-specific queries. Repos, slots,
epics, pauses. Separate from `trellis_model` because workspace data is
backend-only (no UI state merge) and the scanner is expensive — it does
filesystem I/O and should not run on every model query.

### Why Category Tools

MCP tools declare their capabilities via their definitions — name,
description, parameter schema. That IS the discovery mechanism. The LLM
client reads the tool list and knows what's available without calling
anything first.

Hiding everything behind a generic `trellis_act(target, action, params)`
would throw away MCP's native discovery. The LLM would need to call
`trellis_model()` first to learn what actions exist — an extra round-trip
that buries semantics in data instead of schema.

Category tools balance discoverability with stability: each tool is
self-describing (the LLM knows what Trellis can do from the tool list),
and operations within a category are parameters (new operations don't add
new tools).

## §5 Model Registry Architecture

The model tree is assembled at query time from actual service beans and
pushed UI state. No separate model definition. No code generation.

### Backend Assembly

The MCP tool bean's `trellis_model` method builds the tree by calling
existing service methods:

| Source | Method | Tree Branch |
|--------|--------|-------------|
| `TerminalRegistry` | `list()` | `terminals` |
| `AgentProcessManager` | `getAllSnapshots()` | merged into terminal nodes |
| `WorkspaceScanner` | `scan()` | `workspace` |
| UI state (in-memory) | — | `dock-bar`, `panels` |

The Java records ARE the schema. Add a field to `TerminalInfo`, the model
grows. Add a new record type, wire it into the assembly method, the model
grows. One place to change.

### Generation Counter

Every model response includes a `generation` field — a monotonically
increasing `AtomicLong` incremented on any state mutation across the
service beans (`TerminalRegistry` create/destroy, `AgentProcessManager`
state transitions, `WorkspaceChanged` CDI events, UI state pushes). The
model assembly captures the current generation before reading and
includes it in the response. The agent can compare generations across
calls to detect intervening mutations — if the generation hasn't changed,
the state is identical to the last query.

### Frontend Assembly

The sidecar holds the latest UI state pushed via `POST /api/model/ui-state`.
The `trellis_model` method merges this into the tree under `dock-bar` and
`panels`. The UI state is opaque JSON — the sidecar does not parse it, just
nests it.

### Action Declarations

Each node type carries an `actions` list describing what can be done. These
are computed from the service bean's public methods. A terminal node declares
`[send-input, read-log, start-agent, stop-agent, ...]` because those methods
exist on `TerminalRegistry` and `AgentProcessManager`.

The action list is a static mapping per node type in the tool bean — a
`Map<NodeType, List<ActionDescriptor>>`. When a new method is added to a
service bean and needs to be exposed, it is added to this mapping. One place,
explicit, no annotation magic.

### Path Resolution

Model paths like `terminals/engine` or `panels/workspace-view/frames/0` are
resolved by walking the assembled tree. Path segments map to collection names
and identity keys (terminal name, frame id, panel name). No separate routing
table — the tree structure IS the routing.

**Error contract:**
- **Invalid path** (no matching node at any depth): error response with
  `error: "not_found"` and `path` echoing the requested path.
- **Partially valid path** (prefix resolves, remainder doesn't): returns the
  deepest valid node with `resolvedPath` indicating how far resolution got
  and `unresolvedSuffix` containing the remaining segments.
- **Numeric segments**: always identity keys, not indices. `frames/0` means
  "the frame with ID 0", not "the first frame." Frame IDs are stable
  identifiers assigned at creation. If no frame has ID 0, it's a not-found
  error.

### Performance

`trellis_model()` with no path assembles the full tree but excludes heavy
content (session logs, terminal scrollback). `trellis_model(path=...)` returns
just that subtree with full detail including action descriptions.

### Workspace Scan Cache

The workspace scan is cached because it does filesystem I/O. Cache design:
- TTL: 30 seconds (configurable via `trellis.workspace-scan.cache-ttl`)
- Invalidation: the cache is evicted on `@WorkspaceChanged` CDI events,
  which `LifecycleManager` already fires after every lifecycle operation
  (`slotCreate`, `start`, `end`, `pause`, `resume`). After eviction, the
  next `trellis_model()` or `trellis_workspace()` call triggers a fresh scan.
- Manual refresh: `trellis_workspace()` accepts a `refresh: true` parameter
  to bypass the cache.

## §6 SSE Integration

The sidecar already has SSE infrastructure (`PushResource`, topic-based
subscriptions). The control plane uses this for pushing navigation commands
to the frontend and notifying MCP clients of state changes.

### Navigation Push (sidecar → frontend)

When `trellis_navigate` is called, the sidecar generates a correlation ID
and emits a `control:navigate` SSE event with `{target, correlationId,
windowId?}`. The `windowId` field is optional — when omitted, all windows
evaluate the target; when present, only the targeted window acts.

The frontend's workbench component listens for `control:navigate` events
and translates model paths to internal navigation:

| Model path prefix | Frontend action |
|-------------------|-----------------|
| `dock-bar/{panel}` | `location.hash = '#{panel}?root=...'` |
| `panels/workspace-view` | Hash to `#workspace`, then delegate to workspace-view |
| `panels/workspace-view/frames/{id}` | `_jumpToFrame(id)` on the workspace-view component |
| `panels/workspace-view/frames/{id}/tabs/{idx}` | `_jumpToFrame(id)` then `_jumpToTab(idx)` |
| `panels/{other}` | `location.hash = '#{other}?root=...'` |

After executing the navigation, the frontend pushes its updated UI state
via `POST /api/model/ui-state` with the `correlationId` included. The
sidecar matches the correlation ID to the pending `trellis_navigate` call
and unblocks it with the post-navigation state. If no acknowledgment
arrives within 5 seconds, the tool returns a timeout error.

For multi-window scenarios: each window checks whether the target path
resolves to content it hosts. Dock-bar navigation always targets the main
window. Frame/tab navigation targets whichever window hosts that frame
(determined by checking the window's local frame registry).

### State Change Notifications (sidecar → MCP clients)

The existing SSE topics (`agent:state`, `workspace:repos`, `workspace:slots`)
already broadcast state changes. A coordinating agent subscribes to these
via the existing `GET /api/push?topics=...` endpoint. No new event
infrastructure needed.

### New SSE Topics

| Topic | Event | When |
|-------|-------|------|
| `control:navigate` | `{target, result}` | Navigation command executed |
| `control:action` | `{target, action, result}` | Action completed |
| `model:ui-state` | `{activePanel, panels}` | Frontend pushed new UI state |

The coordinating agent subscribes to `control:*` and `model:*` to stay
informed without polling. Poll for full state, subscribe for deltas.

### SSE Connection Recovery

SSE connections can drop (network, process restart, timeout). Recovery
strategy: on reconnect, the agent calls `trellis_model()` to get the full
current state, then resumes SSE subscription. No event replay needed — the
model is always the source of truth; SSE is an optimisation to avoid
polling. The generation counter (§5) lets the agent detect whether state
changed during the disconnect without comparing full model snapshots.

### No New Transport

The MCP server is embedded in the sidecar. SSE is existing infrastructure.
Navigation events flow over the same SSE channels the frontend already uses.
The control plane adds new event types, not new infrastructure.

## §7 Scope Boundary

### In scope

- `quarkus-mcp-server` dependency added to sidecar
- MCP tool bean with 6 `@Tool` methods (§4)
- Model assembly from existing service beans (§5)
- Session log tee in `TerminalWebSocket` (§2)
- `POST /api/model/ui-state` endpoint (§3)
- Frontend UI state push from workbench component (§3)
- `control:navigate` SSE event + frontend handler (§6)
- Tests: tool bean unit tests, session log write/read, UI state
  round-trip, navigation SSE

### Out of scope

- Blocks chat integration for terminal interaction — future evolution
  of `trellis_terminal` write path
- Custom panel action declarations — panels exposing their own action
  schemas dynamically
- MCP authentication/authorization — single-user local app, no auth yet
- Multi-client MCP coordination — one coordinating agent at a time
- Session log rotation/archival — append-only is fine at current scale
- MCP-native change notifications — SSE covers this, MCP Notifications
  spec can layer on later

### Extension Path

When Trellis grows a new capability (e.g. build monitoring):
1. Add the service bean with its records (`BuildInfo`, `BuildResult`)
2. Wire it into the model assembly in the tool bean
3. Add action entries to the action mapping
4. The 6 MCP tools do not change — the model grows, the tools stay stable

## §8 Testing Strategy

### Unit Tests (Java)

| Component | Test |
|-----------|------|
| Model assembly | Tree structure matches service bean data. Path resolution returns correct subtrees. |
| Action mapping | Every declared action maps to a real service method. Unknown actions return 404. |
| Session log tee | Bytes written to FIFO appear in log file. Append survives reconnection. |
| Session log read | `read-log` returns correct tail/offset of log file. Empty log returns empty. RandomAccessFile tail-read cost is O(bytes), not O(file). |
| Session log cleanup | `destroySession()` deletes log file. Retention config moves to archive. |
| Input logging | MCP `send-input` writes bracketed marker to log before sending keys. |
| Generation counter | Mutations increment generation. Model response includes current generation. |
| Workspace cache | Cache invalidated on WorkspaceChanged. TTL evicts stale entries. |
| Path resolution errors | Invalid path returns error. Partial path returns deepest valid node. |
| UI state size limit | Content exceeding 64KB rejected with 413. lastPushed timestamp present. |
| UI state endpoint | POST stores state in memory. GET via model returns it. Overwrite replaces. |
| Navigation SSE | `trellis_navigate` emits `control:navigate` event with correct target path. |
| MCP tool bean | Each `@Tool` method delegates to correct service bean. Parameter validation. |

### Frontend Tests

| Component | Test |
|-----------|------|
| UI state push | Workbench pushes state on panel switch. Debounce fires within max-wait. |
| Navigation handler | `control:navigate` SSE event switches panel, focuses frame, selects tab. Correlation ID round-trip completes. |
| Navigation timeout | No frontend acknowledgment within 5s returns timeout error. |
| Multi-window nav | Window-targeted events only handled by correct window. |
| Input logging | Human keystrokes via WebSocket appear in session log interleaved with output. |

### Integration Tests

| Area | Test |
|------|------|
| MCP round-trip | Connect MCP client → call `trellis_model` → verify response matches sidecar state |
| Navigate round-trip | `trellis_navigate` → frontend receives SSE → pushes UI state with correlation ID → tool unblocks → model reflects change |
| SSE recovery | Drop SSE → reconnect → `trellis_model()` returns current state → generation counter detects changes |
| Terminal I/O | `trellis_terminal(send-input)` → text appears in tmux session → logged to session file → `trellis_terminal(read-log)` returns it |
