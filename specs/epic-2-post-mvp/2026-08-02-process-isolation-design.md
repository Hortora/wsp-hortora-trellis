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

**Current state:** The rename is partially started — `TerminalResource.java` exists
but still has `@Path("/api/sessions")`. This spec completes the rename.

### Pause/Resume Disambiguation

"Pause/resume" exists at two independent layers in trellis:

| Operation | Component | What it does | API |
|-----------|-----------|-------------|-----|
| **Slot pause** | `LifecycleManager` | Commits WIP, pushes branches to stack — preserves git workspace | `POST /api/lifecycle/pause/{slotId}` |
| **Slot resume** | `LifecycleManager` | Checks out branches, rebases, resets WIP — restores git workspace | `POST /api/lifecycle/resume/{slotId}` |
| **Agent pause** | `AgentProcessManager` | Kills Claude process, marks PAUSED — frees memory | `POST /api/terminals/{name}/agent/pause` |
| **Agent resume** | `AgentProcessManager` | Starts `claude -c` — continues conversation | `POST /api/terminals/{name}/agent/resume` |

These are independent operations at different layers. Agent pause/resume manages
OS process lifecycle (issue #20's memory management). Slot pause/resume manages
git workspace state (existing feature). Neither triggers the other.

Coordination (e.g., slot pause auto-pausing its agents) is a future concern
tracked in Hortora/trellis#22.

## Architecture: Hybrid Process Management

Combines auto-start (immediate PID awareness when creating terminals) with
periodic discovery (self-healing, handles manual starts and crashes).

### Layer Model

| Layer | What | Component |
|-------|------|-----------|
| I/O transport | tmux session | `TmuxManager` (existing, renamed methods) |
| OS process | Claude CLI with PID/memory | `AgentProcessManager` (new) |
| LLM conversation | Programmatic agent turns | `AgentProvider` / `AgentSession` (existing platform API, used by coordinator) |

Issue #20 fills the OS process layer. It is independent of the platform
`AgentSession` used by the coordinator for programmatic LLM invocations.

## §1 Domain Model

### Renamed Types

```java
// Was: SessionInfo — pure terminal descriptor, no agent state
record TerminalInfo(
    String name,
    String workingDir,
    String slot,
    String repo,
    String issue
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
    AgentProcess process,   // null when IDLE
    String lastError        // set on STARTING→IDLE timeout, cleared on next start
)
```

## §2 Agent Process Management

**Package:** `io.hortora.trellis.agent`

New types (`AgentState`, `AgentProcess`, `AgentSnapshot`) and the unified
`AgentProcessManager` live in `io.hortora.trellis.agent` — distinct from
`terminal` (I/O transport) and `lifecycle` (work/branch operations).

`AgentProcessManager` — `@ApplicationScoped` service that owns both discovery
(scheduled polling) and control (start/stop/pause/resume). A single component
because monitoring and lifecycle operations share mutable state: the monitor
must respect lifecycle invariants (PAUSED), and lifecycle operations need PID
data from monitoring. Splitting them across classes adds coordination overhead
without architectural clarity.

### Process Discovery

Scheduled poll every 5 seconds.

### Discovery Mechanism

For each registered terminal:

1. `tmux display-message -t <name> -p '#{pane_pid}'` → shell PID
2. `tmux display-message -t <name> -p '#{pane_current_command}'` → foreground command
3. If command is a shell (`bash`/`zsh`/`sh`/`fish`/`dash`) → `IDLE`
4. If command is `claude` or `node` → agent running; walk process tree for PID and RSS

Pattern proven by claudony's `StatusAwareExpiryPolicy`.

### Process Tree Walk (macOS)

- `ps -eo pid=,ppid=,rss=,args=` → build parent→children map (RSS in kilobytes on macOS; ×1024 for `memoryBytes`)
- From `pane_pid`, recursively collect all descendants
- Filter for Claude root (`args` contains `claude` — `comm` shows `node`, not
  `claude`, because Claude Code is a Node.js application)
- Sum RSS of Claude root + all children (MCP servers, subagents)

**RSS overcounting:** RSS includes shared memory-mapped pages, counted once per
process even when pages are physically shared (e.g. Node.js runtime shared by
Claude and MCP servers). For a typical tree, RSS overreports by ~30-50%. The
500 MB threshold is calibrated against RSS-reported values, not actual physical
memory. Acceptable for a warning display — not used for automatic actions.

### State Transitions Detected by Monitor

| From | To | Trigger |
|------|----|---------|
| IDLE | RUNNING | Claude process appears (auto-started or manual) |
| RUNNING | IDLE | Claude process gone without explicit pause (crash or user exit) |
| STARTING | RUNNING | Claude process appears within 15s of start command |
| STARTING | IDLE | 15s timeout — process never appeared (failed start) |
| PAUSED | RUNNING | Claude process appears (manual start in paused terminal) |

**STARTING protection:** The monitor MUST NOT transition from STARTING to any
state other than RUNNING until the 15s timeout expires. If the monitor observes
no Claude process while the terminal is in STARTING state and less than 15s
have elapsed, it preserves STARTING.

**PAUSED invariant:** The monitor never transitions a terminal TO PAUSED — only
explicit lifecycle operations set PAUSED. The monitor MAY transition FROM PAUSED
to RUNNING if a Claude process is detected (e.g. user manually started Claude
in the terminal).

**Uncontrolled exit ambiguity:** The monitor cannot distinguish a Claude crash
from a user typing `/exit` — both result in the process disappearing. Lifecycle
operations (`stopAgent`, `pauseAgent`) set state directly and are unambiguous.
The monitor-detected `RUNNING → IDLE` covers all uncontrolled exits. If
crash-specific behavior (auto-restart, alerting) is needed later,
`ProcessHandle.onExit()` could capture exit codes.

### Bootstrap Initialization

When `TerminalRegistry.bootstrap()` discovers existing tmux sessions on startup,
`AgentProcessManager` has no entries yet. Until the first monitor cycle (within
5 seconds), `AgentSnapshot.process` is `null` for bootstrapped terminals —
effectively showing IDLE even if Claude is running. The first scheduled poll
corrects this. This brief inaccuracy is acceptable: IDLE→RUNNING is a safe
false-negative that blocks no user action.

### State Storage

- `ConcurrentHashMap<String, AgentProcess>` inside `AgentProcessManager`, keyed by terminal name
- State is authoritative in this single location — no denormalization into `TerminalInfo`
- On state changes, publishes to push topic `agent:state` via `TopicRegistry`
- `AgentSnapshot` composed at API response time from `TerminalInfo` + `AgentProcess`

**PAUSED persistence:** PAUSED state is persisted as a tmux user option
`@trellis_agent_state` on the terminal's tmux session. On `bootstrap()`,
`AgentProcessManager` reads this option; if the value is `PAUSED`, the terminal
is initialized in PAUSED state. This survives both sidecar and tmux server
restarts, consistent with how terminal metadata (`@trellis_slot`,
`@trellis_repo`, `@trellis_issue`) is already persisted and recovered.

**Clearing `@trellis_agent_state`:** Every transition OUT of PAUSED clears the
tmux option to prevent stale persistence across restarts:
- `resumeAgent` clears before sending keys
- `stopAgent` (from PAUSED) clears before setting IDLE
- Monitor `PAUSED → RUNNING` detection clears on transition
- `clearState` (terminal destruction) clears as part of teardown

**Bootstrap ordering:** `AgentProcessManager` MUST NOT start its scheduled
polling until `TerminalRegistry.bootstrap()` completes. Standard Quarkus
lifecycle ordering via `@Observes StartupEvent` with `@Priority`. The first
monitor cycle after bootstrap completes is the authoritative initial state.

### Lifecycle Operations

| Operation | Action | State Transition |
|-----------|--------|-----------------|
| `startAgent(terminal, opts)` | Construct command from `opts`, verify shell, send-keys | IDLE → STARTING → RUNNING |
| `stopAgent(terminal)` | Tree-kill (§Kill Semantics), clear `@trellis_agent_state` | RUNNING → IDLE, PAUSED → IDLE |
| `pauseAgent(terminal)` | Tree-kill, persist PAUSED via `@trellis_agent_state` | RUNNING → PAUSED |
| `resumeAgent(terminal)` | Clear `@trellis_agent_state`, verify shell, send-keys `claude -c` | PAUSED → STARTING → RUNNING |
| `refreshAgent(terminal)` | Set STARTING → tree-kill → wait → send `claude -c` | RUNNING → STARTING → RUNNING |

All methods are on `AgentProcessManager`. The same component owns both the
scheduled poll and the lifecycle operations — no cross-component state sharing.

**Command construction:** `startAgent` does not accept a freeform command
string. It accepts structured parameters:

```java
record StartAgentRequest(
    boolean resume,           // true → `claude -c`
    String prompt             // optional → `claude -p "<prompt>"`
)
```

The sidecar constructs the exact shell command. The `prompt` value is
embedded using POSIX single-quote escaping: wrap in single quotes, escape
internal single quotes as `'\''`. Example: prompt `Fix the "login" bug`
→ `claude -p 'Fix the "login" bug'`; prompt `Fix the 'auth' flow`
→ `claude -p 'Fix the '\''auth'\'' flow'`. Single quotes disable all
shell metacharacter interpretation (`$`, backticks, `!`, `\`, `"`).

This eliminates both command injection and correctness issues — the sidecar
never sends unescaped user-supplied text to the terminal via lifecycle
operations.

**Validation:** `resume` and `prompt` are mutually exclusive. Setting both
is a 400 Bad Request — you cannot continue an existing session and provide
a new initial prompt simultaneously. Valid combinations:
- `resume=false, prompt=null` → `claude` (fresh session)
- `resume=false, prompt="..."` → `claude -p "..."` (fresh session with prompt)
- `resume=true, prompt=null` → `claude -c` (continue session)

**Terminal state verification:** Before sending keys, lifecycle operations verify
that `pane_current_command` is a shell (same shell command set as the monitor).
If a non-shell process is in the foreground, the operation returns an error.
After a kill operation, a 500ms delay is inserted before sending keys to allow
the shell prompt to appear.

**Refresh IDLE suppression:** `refreshAgent` sets the state to STARTING before
killing the process. This prevents the monitor from observing a transient IDLE
state between kill and restart. Sequence: set STARTING → tree-kill → wait for
exit → 500ms delay → send `claude -c`. No intermediate IDLE event.

**Resume/refresh always use `claude -c`:** This is intentional. The `-c` flag
continues the most recent Claude session, restoring conversation history, model
selection, and tool state. The original start command is stored in
`AgentProcess.command` for display and diagnostics, but is not replayed on
resume — its effects (initial prompt, flags) are already part of the session
state that `-c` restores.

### Auto-Start on Terminal Creation

`TerminalRegistry.createTerminal()` accepts an optional `StartAgentRequest`.
If provided, calls `agentProcessManager.startAgent()` after tmux session
creation. Default: no auto-start (IDLE).

### Terminal Destruction Coordination

`TerminalRegistry.destroyTerminal()` coordinates the full teardown sequence:

1. Call `agentProcessManager.stopAgent()` if agent is RUNNING or STARTING
2. Call `agentProcessManager.clearState()` to remove process state (handles PAUSED)
3. Call `tmuxManager.killSession()` to destroy the tmux session
4. Remove terminal from the registry map

This mirrors the creation path where `TerminalRegistry.createTerminal()` already
calls `agentProcessManager.startAgent()`. The registry owns terminal lifecycle;
the process manager owns agent process lifecycle. The registry coordinates both.

### Kill Semantics

1. Build process tree from the Claude root PID (same tree walk as RSS summation)
2. Send SIGTERM to the Claude root PID via `ProcessHandle.of(pid).destroy()`
3. Wait up to 5 seconds for exit (monitor via `ProcessHandle.onExit()`)
4. If still alive, kill all descendants leaf-first via
   `ProcessHandle.of(childPid).destroyForcibly()`, then kill the root

`ProcessHandle.destroyForcibly()` sends SIGKILL to a single process — Java's
ProcessHandle API does not support process-group kills. The tree walk is the
only reliable approach because child processes (MCP servers) may create their
own process groups.

### Concurrency Control

Per-terminal `ReentrantLock` prevents concurrent lifecycle operations on the
same terminal. Same locking pattern as `LifecycleManager.withLock()` in
`io.hortora.trellis.lifecycle` (work/branch lifecycle — a separate domain).

## §4 REST API

### Renamed Endpoints

```
GET    /api/terminals              → List<AgentSnapshot>
GET    /api/terminals/{name}       → AgentSnapshot
POST   /api/terminals              → create terminal (optional auto-start)
DELETE /api/terminals/{name}       → destroy terminal (kills agent if running, clears PAUSED state)
POST   /api/terminals/{name}/input → send keystrokes to terminal (existing, renamed from /api/sessions)
```

### Agent Sub-Resource Endpoints

Agent operations are sub-resources of terminals — agents are always scoped
to a terminal (1:1 containment), and the URL structure reflects this:

```
POST   /api/terminals/{name}/agent/start    → start Claude in terminal
       body: { "resume": false, "prompt": "..." }  (both optional)
POST   /api/terminals/{name}/agent/stop     → stop Claude process
POST   /api/terminals/{name}/agent/pause    → kill Claude, mark PAUSED
POST   /api/terminals/{name}/agent/resume   → restart with claude -c
POST   /api/terminals/{name}/agent/refresh  → stop + restart with claude -c
GET    /api/terminals/{name}/agent/stats    → { pid, memoryBytes, uptime, state }
```

Implemented as a JAX-RS sub-resource: `TerminalResource` delegates
`/agent/*` paths to an `AgentSubResource` instance, keeping resource
classes focused.

### Live Updates

Agent state changes use the existing push infrastructure at `/api/push`
(topic-based SSE via `TopicRegistry` from `casehub-pages-push`):

- Topic: `agent:state` — emits `AgentSnapshot` JSON on state changes and on
  each monitor cycle (every 5s) for terminals with running/starting agents,
  ensuring displayed memory values are always current
- Topic: `agent:memory` — emits when memory crosses threshold

**Initial state:** On subscription, the `agent:state` topic emits a full
`AgentSnapshot` for every terminal — the subscriber starts with current state
rather than needing a separate REST call.

The frontend already subscribes to `/api/push` for coordinator events
(`coordinator:advice`, `coordinator:message`). Agent events are additional
topics on the same connection — no second SSE endpoint needed.

No `Last-Event-ID` replay — state is eventually consistent within one monitor
cycle (5s), making replay unnecessary for a local sidecar.

### Design Decisions

- `GET /api/terminals` returns `AgentSnapshot` (terminal + agent combined) —
  frontend doesn't need two calls
- No top-level `/api/agents` resource — agents exist only within terminals
- All agent operations return the updated `AgentSnapshot` for immediate UI
  refresh without waiting for monitor poll
- `POST /api/terminals` body gains an optional `StartAgentRequest` for auto-start
- **Network binding:** `quarkus.http.host=localhost` — the sidecar manages local
  tmux sessions and has no authentication layer; binding to all interfaces would
  expose the control API to the network

### Error Responses

| Condition | HTTP Status | Body |
|-----------|------------|------|
| Terminal not found | 404 | `{ "error": "terminal not found: <name>" }` |
| Invalid state transition (e.g., stop when IDLE, resume when RUNNING) | 409 Conflict | `{ "error": "cannot <op> agent in state <current>", "state": "<current>" }` |
| Concurrent lifecycle operation on same terminal | 409 Conflict | `{ "error": "operation already in progress for: <name>" }` |
| Contradictory start request (`resume=true` + `prompt` set) | 400 Bad Request | `{ "error": "resume and prompt are mutually exclusive" }` |

All agent operations return 409 for invalid state transitions rather than
silently succeeding. The response includes the current state so the frontend
can update its display without an additional GET.

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
| RUNNING | refresh, pause, stop |
| PAUSED | resume |
| IDLE | start (defaults to `claude`) |
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

App subscribes to `/api/push?topics=agent:state,agent:memory` on load (or
adds these topics to an existing push connection). Each event updates the
relevant terminal's agent state in-place. No full-page refresh.

## §6 Testing Strategy

### Unit Tests (no tmux required)

- **`AgentProcessManagerTest`** — mock `TmuxManager`, verify:
  - State transitions: IDLE→RUNNING on Claude detection, RUNNING→IDLE on
    process disappearance, PAUSED preserved across monitor cycles
  - Lifecycle operations: start sends correct command, stop kills correct PID,
    refresh sequences stop→start, concurrent operations rejected
- **`AgentProcessTest`** — verify process tree RSS summation with canned
  `ps` output. Edge cases: no children, deep tree, zombie processes

### Integration Tests (require tmux — `assumeTrue` guard)

- **`TerminalRegistryTest`** — renamed from `SessionRegistryTest`, same
  coverage, verify renamed API, verify destruction coordination (agent
  cleanup before tmux kill)
- **`AgentProcessManagerIntegrationTest`** — real tmux session with a
  long-running process, verify discovery and RSS reporting, verify IDLE
  detection on process exit, full start→running→pause→resume→stop cycle
  with a mock agent script

### REST Endpoint Tests (`@QuarkusTest`)

- **`TerminalResourceTest`** — renamed paths, `AgentSnapshot` response shape,
  agent sub-resource operations: start/stop/pause/resume/refresh return
  correct snapshots, 404 for invalid terminal, 409 for concurrent ops

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
