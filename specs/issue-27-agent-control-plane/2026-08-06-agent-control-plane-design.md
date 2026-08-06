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
├── dock-bar                                    [frontend · ephemeral · per-window]
│   ├── dashboard    {active: true}
│   ├── workspace    {active: false}
│   ├── artifacts    {active: false}
│   ├── protocols    {active: false}
│   ├── garden       {active: false}
│   └── memory       {active: false}
├── panels                                      [frontend · ephemeral · per-window]
│   ├── dashboard                               [content shape defined by frontend push]
│   │   ├── repo-cards: [{name, branch, remoteUrl, actions: [...]}]
│   │   ├── slot-cards: [{number, issue, status, repos, actions: [...]}]
│   │   └── epic-cards: [{...}]
│   ├── workspace-view
│   │   └── frames: [{id, groupId?, order, position, size, pinned, zIndex,
│   │                  activeTabIndex,
│   │                  tabs: [{terminalName, type: 'repo'|'slot'}],
│   │                  actions: [pin, focus]}]
│   ├── artifacts
│   │   └── files: [{path, type, actions: [open]}]
│   ├── protocols
│   │   └── repos: [{name, entries: [{id, title, actions: [view, remove]}]}]
│   ├── garden
│   │   └── results: [{id, title, domain, actions: [view]}]
│   └── memory
│       └── entries: [{name, type, actions: [view]}]
├── terminals                                   [backend · authoritative]
│   └── [{name, workingDir, slot, repo, issue,
│          agent: {state, pid, memoryBytes, startedAt, command, lastError},
│          sessionLog: "/path/to/log",
│          actions: [send-input, read-log, start-agent, stop-agent,
│                    graceful-shutdown-agent, pause-agent, resume-agent,
│                    refresh-agent, destroy]}]
└── workspace                                   [summary — full data via trellis_workspace]
    ├── root: "/path/to/workspace"
    ├── slotCount: 3
    └── activeSlot: "slot-1"
```

The tree annotates each subtree's state characteristics:

- **backend · authoritative** — durable server-side state, always consistent,
  trustworthy for decision-making.
- **frontend · ephemeral · per-window** — pushed from the Electron frontend,
  absent when no frontend is connected, may differ across windows. See §3 for
  multi-window details.
- **summary** — lightweight metadata included in the model tree for
  discoverability. Full data requires the dedicated `trellis_workspace` tool.

Panel content subtrees (e.g. `repo-cards`, `frames`) document the shape of
what the frontend pushes. The sidecar does not parse or validate this content
— it stores and serves it as opaque JSON. Path resolution via
`trellis_model(path=...)` terminates at the panel level (e.g.
`panels/dashboard`), returning the full content blob. The agent navigates
within panel content client-side.

### Node Properties

Each node has:
- **Identity** — a path in the tree (e.g. `terminals/engine`,
  `panels/workspace-view/frames/0/tabs/1`)
- **State** — current values, typed per node kind
- **Actions** — what can be done here, with parameter descriptions.
  Actions carry a `source` field: `"backend"` actions are MCP-executable
  via category tools; `"frontend"` actions are informational, describing
  UI capabilities that the agent cannot invoke via MCP (see §5 Action
  Declarations).
- **Children** — nested nodes

The model is read-only data + action declarations. It never contains
behaviour — it is a snapshot of what exists, what you can do via MCP
tools, and what the frontend UI can do independently.

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

A `SessionLogger` service bean, keyed by terminal name, owns session log
writes. `TerminalRegistry` creates the logger when the terminal is created
and cleans it up on `destroySession()`. The `FifoRelay` (spawned by
`TerminalWebSocket.onOpen()`) calls `sessionLogger.append(bytes)` as it
relays FIFO data — the relay already has the byte stream, the logger just
tees it to disk.

The log file is `{sidecar-data-dir}/sessions/{terminalName}.log`.
Append-only. The file grows for the lifetime of the tmux session.

This separation means the session log is a terminal concern, not a WebSocket
concern. The logger persists across WebSocket reconnections — the terminal
outlives any single connection. The MCP `trellis_terminal(read-log)` tool
injects `SessionLogger` directly rather than reading raw files.

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

Panel content includes action declarations embedded by the frontend. The
`actions` lists shown in §1 (e.g. `[pin, focus]` on frames) are part of
the pushed content, not computed by the sidecar. Each panel component
knows its own action palette and includes it in its push with
`"source": "frontend"` on each action descriptor. The sidecar serves
these action declarations as part of the opaque content blob — it does
not interpret or validate them. MCP consumers use the `source` field to
distinguish these informational actions from MCP-executable backend
actions (see §5 Action Declarations).

The sidecar enforces a 64KB size limit on serialised `content` per panel.
Pushes exceeding this are rejected with HTTP 413. The `lastPushed` timestamp
lets MCP consumers detect stale UI state — if a panel hasn't pushed in the
last 30 seconds, the model response flags it as `stale: true`.

### Push Trigger

Debounced — same cadence as layout persistence (1s debounce, 5s max-wait).
Panel switches push immediately (no debounce). Heartbeat: every 15 seconds,
the frontend pushes its current UI state regardless of changes, refreshing
`lastPushed`. This ensures `trellis_navigate` can distinguish an idle
frontend (connected, no recent interaction) from an absent one (no frontend
process running). The 15-second interval is half the staleness threshold,
providing sufficient margin for delayed heartbeats.

### Multi-Window

Trellis currently runs a single window. The `ShellLayout` persistence format
supports a `windows` array for future multi-window support, but the model
tree and UI state push assume a single window. When multi-window is added,
each window will push its own UI state keyed by window ID, and the model
tree will nest under a `windows` collection.

### Data Flow

Unidirectional: frontend pushes, sidecar serves. The sidecar never queries
the frontend.

## §4 MCP Tools

Six `@Tool` methods on a thin CDI bean. The bean delegates model assembly
to domain-specific `ModelProvider` implementations (see §5) and operations
to existing service beans. Return types are the existing Java records —
serialised directly, no adapter layer.

The tool bean injects `Instance<ModelProvider>` for tree assembly and the
individual service beans for operations (`TerminalRegistry`,
`AgentProcessManager`, `SlotAgentCoordinator`, `LifecycleManager`,
`WorkspaceScanner`). Each operation tool method is a thin dispatcher — it
validates parameters and delegates to the appropriate service bean. The
model assembly logic lives in the providers, not the tool bean.

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
`target` is a model path. Before emitting the SSE event, the tool checks
whether any UI state has been pushed within the staleness threshold (30s).
If no recent UI state exists, the tool returns immediately with an error
`status: "no_frontend"` — no blocking, no timeout. When a frontend is
connected, the tool pushes a `control:navigate` SSE event with a
correlation ID. It blocks until the frontend acknowledges the navigation
by pushing updated UI state tagged with the same correlation ID, or until
a 5-second timeout expires. Returns the post-navigation state of the
target node on success, or an error with `status: "timeout"` if the
frontend does not acknowledge in time.

**`trellis_terminal(name, operation, params?)`** — Terminal I/O. Operations:
`read-log` (params: lines, offset), `send-input` (params: text), `create`
(params: workingDir, repo, slot), `destroy`, `resize` (params: cols, rows).
Delegates to `TerminalRegistry`. `resize` requires adding a
`resize(name, cols, rows)` method to `TerminalRegistry` — it already holds
`TmuxManager` and is the right abstraction for terminal operations
(currently `TerminalResource.resize()` injects `TmuxManager` directly,
bypassing the registry). All tmux operations propagate exit code
failures — `TmuxManager.run()` throws on non-zero exit, and the MCP tool
returns the error to the caller. No silent swallowing.

**`trellis_agent(terminal, operation, params?)`** — Agent lifecycle.
Operations: `start` (params: resume, prompt — matching
`StartAgentRequest`), `stop`, `graceful-shutdown`, `pause`, `resume`,
`refresh`, `stats`, `tree`. Delegates to `AgentProcessManager`.
`graceful-shutdown` sends Escape, tries `/exit`, waits up to 10s for
clean exit, force-kills if needed, and marks `PAUSED_BY_COORDINATOR` —
semantically distinct from `stop` which does an immediate tree-kill.

**`trellis_lifecycle(operation, params?)`** — Slot and workspace lifecycle.
Operations: `start` (params: workspaceRoot, branch, issue), `end`
(params: slotId, workspaceRoot), `pause` (params: slotId, workspaceRoot),
`resume` (params: slotId, workspaceRoot), `slot-create` (params:
workspaceRoot, args), `slot-merge` (params: slotId, workspaceRoot),
`epic-setup` (params: workspaceRoot, args), `epic-next` (params:
epicPath). `end`, `pause`, and `resume` delegate to
`SlotAgentCoordinator` (which coordinates agent shutdown/restart around
the lifecycle operation). All others delegate to `LifecycleManager`.
Workspace refresh is handled by `trellis_workspace(refresh=true)`, not
by a lifecycle operation.

**`trellis_workspace(path?, refresh?)`** — Full workspace queries. Repos,
slots, epics, pauses. `trellis_model(path="workspace")` returns a summary
(root, slotCount, activeSlot); this tool returns the complete workspace
model from `FileWatcherService` cache. Accepts `refresh: true` to force
a fresh filesystem scan bypassing the cache.

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

### Relationship to Coordinator Action Framework

Trellis has an existing programmatic execution path: the L2 coordinator
(issue #17) proposes actions via `ActionService`, which routes through
`ActionExecutor` implementations (`LifecycleActionExecutor`,
`AgentActionExecutor`) to the same service beans the MCP tools call. That
path provides governance: propose → approve → risk-gate → execute, with
full audit persistence in `coordinator_actions`.

The MCP tools are a **direct operational API**, equivalent to the REST
resources (`TerminalResource`, `LifecycleResource`, `ScannerResource`).
They call service beans directly, without the propose/approve/audit
pipeline. This is intentional:

- **REST resources** are the human-facing operational API. A human clicking
  "End Slot" in the UI calls `LifecycleResource` directly — no proposal,
  no approval, immediate execution.
- **MCP tools** are the agent-facing operational API. A coordinating agent
  calling `trellis_lifecycle(slot-end)` is the agent equivalent of the
  human click — direct execution.
- **Coordinator actions** are a governance layer. The LLM coordinator
  observes state and proposes actions for human approval. That flow uses
  `ActionService` for its propose/approve/execute lifecycle.

An MCP client that wants governed execution can use the coordinator's
REST API (`POST /api/coordinator/actions/{id}/approve`) — the governance
path is available, not bypassed. The MCP tools deliberately do not
duplicate it.

The three paths share the same service-layer terminal: `LifecycleManager`,
`AgentProcessManager`, `TerminalRegistry`. These service beans emit CDI
events (`WorkspaceChanged`, agent state changes) regardless of which path
triggered the operation, so the coordinator's event accumulator sees all
mutations — including MCP-initiated ones — for context assembly and
auto-expiry.

### Error Responses

MCP tool errors use the `isError: true` response format with a text
description. Error mapping:

| Condition | MCP response |
|-----------|-------------|
| Terminal/agent not found | `isError: true`, `"not found: {name}"` |
| Concurrent operation in progress | `isError: true`, `"concurrent operation: {details}"` |
| Invalid operation or params | `isError: true`, `"invalid: {details}"` |
| Agent already running (start conflict) | `isError: true`, `"agent already running in {terminal}"` |
| Tmux/process failure | `isError: true`, `"operation failed: {message}"` |
| No frontend connected (navigate) | `isError: true`, `"no frontend connected"` |
| Navigation timeout | `isError: true`, `"navigation timeout: {target}"` |

No structured error codes — the text description is sufficient for LLM
consumers. The coordinating agent doesn't branch on error codes; it reads
the error and decides what to do next.

## §5 Model Registry Architecture

The model tree is assembled at query time from actual service beans and
pushed UI state. No separate model definition. No code generation.

### ModelProvider Interface

Tree assembly is factored into domain-specific providers:

```java
interface ModelProvider {
    String domain();
    Object summary();
    Object resolve(String subpath);
    List<ActionDescriptor> actionsFor(String nodeType);
}
```

Each provider owns its subtree assembly and action declarations. The tool
bean's `trellis_model` method collects all providers via
`Instance<ModelProvider>` and dispatches path resolution by domain prefix.

### Backend Assembly

| Provider | Injects | Tree Branch |
|----------|---------|-------------|
| `TerminalModelProvider` | `TerminalRegistry`, `AgentProcessManager` | `terminals` |
| `WorkspaceModelProvider` | `FileWatcherService` | `workspace` (summary) |
| `UIStateModelProvider` | UI state (in-memory) | `dock-bar`, `panels` (computes `stale: true` when `lastPushed` exceeds 30s) |

The Java records ARE the schema. Add a field to `TerminalInfo`, the model
grows. Add a new record type, wire it into the relevant provider, the
model grows.

Adding a new domain (e.g. build monitoring) requires a new
`ModelProvider` implementation. The tool bean and existing providers are
unchanged.

### Generation Counter

Every model response includes a `generation` field — a monotonically
increasing `AtomicLong` incremented on any state mutation across the
service beans (`TerminalRegistry` create/destroy, `AgentProcessManager`
state transitions, `WorkspaceChanged` CDI events, UI state pushes). The
model assembly captures the current generation **before** any providers
are read and includes it in the response. This makes the generation a
lower bound on freshness — the assembled data is at least as recent as
the reported generation. If a mutation occurs during assembly, the next
query will report a higher generation, prompting the agent to use the
fresh response.

The contract: if the generation is the same as a previous query, no
mutations occurred between the two assembly starts and the data is
identical. If the generation increased, the agent should treat the newer
response as authoritative regardless of whether the increase reflects a
mutation during assembly or between queries.

### Frontend Assembly

The sidecar holds the latest UI state pushed via `POST /api/model/ui-state`.
The `trellis_model` method merges this into the tree under `dock-bar` and
`panels`. The UI state is opaque JSON — the sidecar does not parse it, just
nests it.

### Action Declarations

Each node type carries an `actions` list describing what can be done.
Two kinds of actions exist, distinguishable by their source:

- **Backend actions** — declared by the owning `ModelProvider`. Each
  provider returns action descriptors for its node types. These are
  MCP-executable: the agent calls the corresponding category tool
  (`trellis_terminal`, `trellis_agent`, `trellis_lifecycle`). Backend
  action descriptors include `"source": "backend"`.
- **Frontend actions** — embedded in the pushed UI state by the frontend
  itself (see §3). These are part of the opaque panel content and are
  NOT MCP-executable. They describe what the frontend UI can do with
  that element. Frontend action descriptors include `"source": "frontend"`
  (set by the frontend when pushing content). An MCP consumer seeing a
  frontend action knows it cannot invoke it via tool calls — it is
  informational, describing UI capabilities.

Backend action declarations are a static mapping per node type within
each provider. `TerminalModelProvider` declares `[send-input, read-log,
start-agent, stop-agent, ...]` because those methods exist on
`TerminalRegistry` and `AgentProcessManager`. Explicit, no annotation
magic.

**Maintenance surface for a new capability:** Adding a new service method
and exposing it requires updating (1) the service bean, (2) the action
mapping in the owning `ModelProvider`, and (3) the REST resource (for
REST clients). If the coordinator should know about it, the
`ActionExecutor` and `CoordinatorPrompts` also need updating. The MCP
tool surface (6 tools) does not change — new operations appear as action
declarations in the model, not as new tools.

### Path Resolution

Model paths like `terminals/engine` or `panels/workspace-view` are resolved
by walking the assembled tree. Path segments map to collection names and
identity keys (terminal name, panel name). Backend subtrees resolve to
arbitrary depth. Frontend panel content is opaque — path resolution
terminates at the panel level (e.g. `panels/dashboard`), returning the
full content blob. Deep panel paths (e.g. `frames/0/tabs/1`) are for
navigation commands (§6), not model queries. No separate routing table —
the tree structure IS the routing.

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
arrives within 5 seconds, the pending correlation entry is removed and
the tool returns a timeout error. Late acknowledgments arriving after
timeout are ignored — the correlation ID no longer exists in the pending
map.

For multi-window scenarios: each window checks whether the target path
resolves to content it hosts. Dock-bar navigation always targets the main
window. Frame/tab navigation targets whichever window hosts that frame
(determined by checking the window's local frame registry).

### State Change Notifications (sidecar → MCP clients)

The existing SSE topics already broadcast state changes. A coordinating
agent subscribes to these via the existing `GET /api/push?topics=...`
endpoint. No new event infrastructure needed.

| Topic | Source | Event |
|-------|--------|-------|
| `agent:state` | `AgentProcessManager` | Agent state transitions |
| `agent:eviction` | `AgentMonitorScheduler` | Memory-pressure agent eviction |
| `workspace:repos` | `FileWatcherService` | Repository changes detected |
| `workspace:slots` | `FileWatcherService` | Slot changes detected |
| `workspace:protocols` | `FileWatcherService` | Protocol file changes |
| `coordinator:advice` | `CoordinatorService` | Proactive advice generated |
| `coordinator:action` | `ActionService` | Action state transitions |
| `coordinator:message` | `CoordinatorService` | Coordinator conversation turns |
| `coordinator:notification` | `ActionService` | Autonomous action notifications |

### New SSE Topics

Two topic conventions coexist on the SSE transport:

- **Notification topics** (`entity:event`) — existing convention. The
  server broadcasts that something happened. Clients react at their
  discretion. Examples: `agent:state`, `workspace:repos`,
  `coordinator:action`.
- **Command topics** (`control:verb`) — new convention. The sidecar
  sends a directive that the frontend must execute. The frontend goes
  from observer to actor for these topics.

The `control:` prefix is the formalization — any topic under `control:*`
is a command. All other topics are notifications.

| Topic | Convention | Event | When |
|-------|-----------|-------|------|
| `control:navigate` | command | `{target, correlationId, windowId?}` | Navigation command to frontend |
| `ui:state` | notification | `{activePanel, panels}` | Frontend pushed new UI state |

The coordinating agent subscribes to `control:*` and `ui:*` to stay
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
- `SessionLogger` service bean for session log writes (§2)
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
2. Add a `ModelProvider` implementation for the new domain
3. Add action entries to the new provider's action mapping
4. The 6 MCP tools and existing providers do not change — the model
   grows, the tools stay stable

## §8 Testing Strategy

### Unit Tests (Java)

| Component | Test |
|-----------|------|
| Model assembly | Tree structure matches service bean data. Path resolution returns correct subtrees. |
| Action mapping | Every declared action maps to a real service method. Unknown actions return 404. |
| SessionLogger | Bytes written to FIFO appear in log file. Append survives reconnection. Logger persists across WebSocket reconnects. |
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
