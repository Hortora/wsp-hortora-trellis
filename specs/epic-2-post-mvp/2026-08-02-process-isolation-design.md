# Process Isolation, Memory Monitoring, and Session Lifecycle

**Issue:** Hortora/trellis#20
**Date:** 2026-08-02
**Status:** Approved

## Problem

Claude Code processes leak memory over time, and running multiple sessions
concurrently exhausts system RAM. There is no way to see per-session memory
usage, kill individual sessions cleanly, or restart them without losing context.

Trellis currently treats "sessions" as tmux terminals with no awareness of the
Claude process running inside them — no PID tracking, no memory monitoring, no
lifecycle controls.

## Terminology

Industry consensus and Claude Code's own terminology drive the naming:

| Term | Definition | Rationale |
|------|-----------|-----------|
| **Terminal** | The tmux session — an I/O container with a name, working directory, and metadata | "Terminal" is unambiguous; avoids collision with Claude's "session" concept |
| **Agent** | The Claude CLI process running inside a terminal — has a PID, memory footprint, and lifecycle state | Aligns with Claude Code's `claude agents` dashboard and industry-wide "agent" terminology |
| **Session** | Claude's conversation state (history, tool calls) — survives process restarts via `claude -c` | Claude's own term; trellis references session IDs but does not own this concept |

**Renames in trellis codebase:**
- `SessionInfo` → `TerminalInfo`
- `SessionRegistry` → `TerminalRegistry`
- REST: `/api/sessions` → `/api/terminals`
- Frontend interfaces and component props updated accordingly

## Architecture: Hybrid Process Management

Combines auto-start (immediate PID awareness when creating terminals) with
periodic discovery (self-healing, handles manual starts and crashes).

### Layer Model

| Layer | What | Component |
|-------|------|-----------|
| I/O transport | tmux session | `TmuxManager` (existing, renamed methods) |
| OS process | Claude CLI with PID/memory | `ProcessMonitor` + `AgentLifecycleManager` (new) |
| LLM conversation | Programmatic agent turns | `AgentProvider` / `AgentSession` (existing platform API, used by coordinator) |

Issue #20 fills the OS process layer. It is independent of the platform
`AgentSession` used by the coordinator for programmatic LLM invocations.

## §1 Domain Model

### Renamed Types

```java
// Was: SessionInfo
record TerminalInfo(
    String name,
    String workingDir,
    String slot,
    String repo,
    String issue,
    AgentState agentState   // denormalized for quick list queries
)
```

### New Types

```java
enum AgentState { IDLE, STARTING, RUNNING, PAUSED }

record AgentProcess(
    long pid,
    AgentState state,
    long memoryBytes,       // process tree RSS
    Instant startedAt,
    String command
)

// Combined view for API responses
record AgentSnapshot(
    String terminalName,
    TerminalInfo terminal,
    AgentProcess process    // null when IDLE
)
```

## §2 Process Discovery & Monitoring

`ProcessMonitor` — `@ApplicationScoped` scheduled service. Runs every 5 seconds.

### Discovery Mechanism

For each registered terminal:

1. `tmux display-message -t <name> -p '#{pane_pid}'` → shell PID
2. `tmux display-message -t <name> -p '#{pane_current_command}'` → foreground command
3. If command is a shell (`bash`/`zsh`/`sh`/`fish`/`dash`) → `IDLE`
4. If command is `claude` or `node` → agent running; walk process tree for PID and RSS

Pattern proven by claudony's `StatusAwareExpiryPolicy`.

### Process Tree Walk (macOS)

- `ps -eo pid=,ppid=,rss=` → build parent→children map
- From `pane_pid`, recursively collect all descendants
- Filter for Claude root (command contains `claude`)
- Sum RSS of Claude root + all children (MCP servers, subagents)

### State Transitions Detected by Monitor

| From | To | Trigger |
|------|----|---------|
| IDLE | RUNNING | Claude process appears (auto-started or manual) |
| RUNNING | IDLE | Claude process gone without explicit pause (crash or user exit) |
| STARTING | RUNNING | Claude process appears within 15s of start command |
| STARTING | IDLE | 15s timeout — process never appeared (failed start) |

The `PAUSED` state is only set by explicit lifecycle operations, never by the
monitor. This prevents a paused terminal from flipping to IDLE when the monitor
runs.

### State Storage

- `ConcurrentHashMap<String, AgentProcess>` inside `ProcessMonitor`, keyed by terminal name
- `TerminalRegistry` updated with denormalized `agentState` on each change
- SSE endpoint emits `AgentSnapshot` on state changes

## §3 Agent Lifecycle Operations

`AgentLifecycleManager` — `@ApplicationScoped` service.

### Operations

| Operation | Action | State Transition |
|-----------|--------|-----------------|
| `startAgent(terminal, command)` | `tmux send-keys -t <name> '<command>' Enter` | IDLE → STARTING → RUNNING |
| `stopAgent(terminal)` | `kill <pid>` (SIGTERM, SIGKILL after 5s) | RUNNING → IDLE |
| `pauseAgent(terminal)` | Same as stop, marks PAUSED | RUNNING → PAUSED |
| `resumeAgent(terminal)` | `tmux send-keys -t <name> 'claude -c' Enter` | PAUSED → STARTING → RUNNING |
| `refreshAgent(terminal)` | stop + start with `claude -c` | RUNNING → STARTING → RUNNING |

### Auto-Start on Terminal Creation

`TerminalRegistry.createTerminal()` accepts an optional `command` parameter.
If provided, calls `agentLifecycleManager.startAgent()` after tmux session
creation. Default: no auto-start (IDLE). Caller decides the command
(`claude`, `claude -c`, `claude -p "..."`).

### Kill Semantics

1. Send SIGTERM to Claude root PID
2. Wait up to 5 seconds for exit
3. If still alive, send SIGKILL to the entire process group
4. Implemented via `ProcessHandle.of(pid)` (Java 9+ API)

### Concurrency Control

Per-terminal `ReentrantLock` prevents concurrent lifecycle operations on the
same terminal. Same pattern as existing `LifecycleManager.withLock()`.

## §4 REST API

### Renamed Endpoints

```
GET    /api/terminals              → List<AgentSnapshot>
GET    /api/terminals/{name}       → AgentSnapshot
POST   /api/terminals              → create terminal (optional auto-start)
DELETE /api/terminals/{name}       → destroy terminal (kills agent if running)
```

### New Agent Endpoints

```
POST   /api/agents/{terminalName}/start    → start Claude in terminal
       body: { "command": "claude" }        (optional, defaults to "claude")
POST   /api/agents/{terminalName}/stop     → stop Claude process
POST   /api/agents/{terminalName}/pause    → kill Claude, mark PAUSED
POST   /api/agents/{terminalName}/resume   → restart with claude -c
POST   /api/agents/{terminalName}/refresh  → stop + restart with claude -c
GET    /api/agents/{terminalName}/stats    → { pid, memoryBytes, uptime, state }
```

### Live Updates

```
GET    /api/agents/events                  → SSE stream of AgentSnapshot changes
```

Emitted when ProcessMonitor detects state changes or memory crosses a
threshold. Frontend subscribes on load for reactive updates.

### Design Decisions

- `GET /api/terminals` returns `AgentSnapshot` (terminal + agent combined) —
  frontend doesn't need two calls
- Agent endpoints keyed by terminal name (1:1 relationship)
- All agent operations return the updated `AgentSnapshot` for immediate UI
  refresh without waiting for monitor poll
- `POST /api/terminals` body gains an optional `command` field for auto-start

## §5 Frontend

### Agent State Display

Each terminal entry shows:
- Name and issue reference (existing)
- State badge: `running` (green) · `paused` (amber) · `idle` (grey) · `starting` (blue pulse)
- Memory: `287 MB` — turns red when exceeding threshold (default 500 MB)
- Contextual action buttons

### State → Actions

| State | Buttons |
|-------|---------|
| RUNNING | refresh, pause |
| PAUSED | resume |
| IDLE | start |
| STARTING | spinner (no actions) |

### Memory Display

- Updated via SSE stream
- Compact badge next to terminal name
- Threshold warning at 500 MB (badge turns red)

### Components Changed

- `terminal-tab-group.ts` — add state badge, memory display, action buttons per tab
- `slot-detail.ts` — replace hardcoded pause/end with contextual agent lifecycle buttons
- New: `agent-status-badge.ts` — reusable state + memory badge component

### SSE Subscription

App subscribes to `/api/agents/events` on load. Each event updates the
relevant terminal's agent state in-place. No full-page refresh.

## §6 Testing Strategy

### Unit Tests (no tmux required)

- **`ProcessMonitorTest`** — mock `TmuxManager`, verify state transitions:
  IDLE→RUNNING on Claude detection, RUNNING→IDLE on process disappearance,
  PAUSED preserved across monitor cycles
- **`AgentLifecycleManagerTest`** — mock `TmuxManager` + `ProcessMonitor`,
  verify: start sends correct command, stop kills correct PID, refresh
  sequences stop→start, concurrent operations rejected
- **`AgentProcessTest`** — verify process tree RSS summation with canned
  `ps` output. Edge cases: no children, deep tree, zombie processes

### Integration Tests (require tmux — `assumeTrue` guard)

- **`TerminalRegistryTest`** — renamed from `SessionRegistryTest`, same
  coverage, verify renamed API
- **`ProcessMonitorIntegrationTest`** — real tmux session with a
  long-running process, verify discovery and RSS reporting, verify IDLE
  detection on process exit
- **`AgentLifecycleIntegrationTest`** — full start→running→pause→resume→stop
  cycle with a mock agent script

### REST Endpoint Tests (`@QuarkusTest`)

- **`AgentResourceTest`** — all endpoints: start/stop/pause/resume/refresh
  return correct snapshots, 404 for invalid terminal, 409 for concurrent ops
- **`TerminalResourceTest`** — renamed paths, `AgentSnapshot` response shape

### Frontend

- Manual browser verification of state badges, memory display, and action
  button visibility for each agent state

## Scope Boundary

### In scope
- Terminal/Agent rename throughout trellis (sidecar + frontend)
- Process discovery and monitoring
- Agent lifecycle operations (start/stop/pause/resume/refresh)
- REST API for lifecycle operations
- Frontend state display and action buttons
- SSE live updates

### Out of scope
- Claudony alignment (claudony uses its own Session model; no cross-repo rename)
- Auto-refresh on memory threshold (manual only for now)
- Conversation ID tracking (Claude manages this internally via `-c`)
- Electron shell changes (sidecar-only)
