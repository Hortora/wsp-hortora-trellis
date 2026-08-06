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
`offset` (start from line N for pagination). Returns raw text.

### Input Logging

Commands sent — whether by human via WebSocket or coordinator via `sendKeys`
— are also logged, interleaved with output so the log reads like a complete
session transcript. No differentiation by source — the terminal doesn't know
who typed, the log shouldn't either.

### Lifecycle

Log files are created on first WebSocket data write. They survive terminal
reconnection (append continues). Cleaned up on terminal destroy via
`TerminalRegistry.destroySession()` — configurable retention for post-mortem
analysis (default: delete with terminal).

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
}
```

`content` is `unknown` deliberately. Each panel decides what to expose. The
workspace view pushes its frames, tabs, positions. The artifacts panel pushes
its file list. The dashboard pushes its card states. The sidecar does not parse
or validate `content` — it stores and serves it as-is. When a panel adds new
state, the frontend push grows automatically. No backend change needed.

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
frontend. Returns the updated state of the target node.

**`trellis_terminal(name, operation, params?)`** — Terminal I/O. Operations:
`read-log` (params: lines, offset), `send-input` (params: text), `create`
(params: workingDir, repo, slot), `destroy`, `resize` (params: cols, rows).
Delegates to `TerminalRegistry`.

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

### Performance

`trellis_model()` with no path assembles the full tree but excludes heavy
content (session logs, terminal scrollback). `trellis_model(path=...)` returns
just that subtree with full detail including action descriptions. The workspace
scan is cached with a TTL — it does filesystem I/O and should not run on every
model query.

## §6 SSE Integration

The sidecar already has SSE infrastructure (`PushResource`, topic-based
subscriptions). The control plane uses this for pushing navigation commands
to the frontend and notifying MCP clients of state changes.

### Navigation Push (sidecar → frontend)

When `trellis_navigate` is called, the sidecar emits a `control:navigate`
SSE event with the target path. The frontend's workbench component listens
for this event and executes the navigation — switch panel, focus frame,
select tab. Same rendering path as a human click, different trigger. The
frontend then pushes its updated UI state back via `POST /api/model/ui-state`,
closing the loop.

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
| Session log read | `read-log` returns correct tail/offset of log file. Empty log returns empty. |
| UI state endpoint | POST stores state in memory. GET via model returns it. Overwrite replaces. |
| Navigation SSE | `trellis_navigate` emits `control:navigate` event with correct target path. |
| MCP tool bean | Each `@Tool` method delegates to correct service bean. Parameter validation. |

### Frontend Tests

| Component | Test |
|-----------|------|
| UI state push | Workbench pushes state on panel switch. Debounce fires within max-wait. |
| Navigation handler | `control:navigate` SSE event switches panel, focuses frame, selects tab. |
| Input logging | Human keystrokes via WebSocket appear in session log interleaved with output. |

### Integration Tests

| Area | Test |
|------|------|
| MCP round-trip | Connect MCP client → call `trellis_model` → verify response matches sidecar state |
| Navigate round-trip | `trellis_navigate` → frontend receives SSE → pushes UI state → model reflects change |
| Terminal I/O | `trellis_terminal(send-input)` → text appears in tmux session → logged to session file → `trellis_terminal(read-log)` returns it |
