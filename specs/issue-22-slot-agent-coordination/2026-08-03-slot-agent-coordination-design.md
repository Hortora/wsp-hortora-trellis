# Slot–Agent Pause/Resume Coordination and Memory Pressure Eviction

**Issue:** Hortora/trellis#22
**Date:** 2026-08-03
**Status:** Draft
**Parent:** Hortora/trellis#20 (Process Isolation)

## Problem

Slot pause/resume and agent pause/resume are independent operations. When a
user pauses a slot (commits WIP, pushes branches to stack), running Claude
agents in that slot continue consuming memory for no purpose. Resuming a slot
requires manually restarting each agent. Additionally, there is no visibility
into memory pressure across agents — no way to see which agents are candidates
for eviction when the system is under load.

## Design

### §1 SlotAgentCoordinator

New `@ApplicationScoped` service in `io.hortora.trellis.lifecycle` that
orchestrates the cascade between slot operations and agent operations.

```java
@ApplicationScoped
public class SlotAgentCoordinator {

    @Inject LifecycleManager lifecycleManager;
    @Inject AgentProcessManager agentProcessManager;
    @Inject TerminalRegistry terminalRegistry;
}
```

**`coordinatedPause(slotId, workspaceRoot)`:**

1. Find terminals where `terminal.slot() == slotId`
2. For each terminal with a RUNNING agent: call
   `agentProcessManager.gracefulShutdown(terminalName)` — logs failures,
   does not abort on individual agent failure
3. Delegate to `lifecycleManager.pause(slotId, workspaceRoot)` for git ops
   (commit WIP, push to stack)
4. Return the `OperationResult` from the git ops

**`coordinatedResume(slotId, workspaceRoot)`:**

1. Delegate to `lifecycleManager.resume(slotId, workspaceRoot)` for git ops
   (checkout branches, rebase, reset WIP)
2. If git ops failed, return immediately — do not restart agents on a broken
   workspace
3. Find terminals where `terminal.slot() == slotId`
4. For each terminal in PAUSED state: call
   `agentProcessManager.resumeAgent(terminalName)` — logs failures, does not
   abort on individual agent failure
5. Return the `OperationResult` from the git ops

**Lock ordering:** The coordinator does not acquire locks itself. It calls
through to `LifecycleManager` (workspace-keyed lock) and `AgentProcessManager`
methods (terminal-keyed locks). On pause, agents are shut down before
`LifecycleManager.pause()` acquires its lock. On resume,
`LifecycleManager.resume()` finishes and releases its lock before agent
restarts begin. No simultaneous lock holding — the constraint from the
process isolation spec (LifecycleManager lock first) is satisfied by
sequencing, not by nested locking.

**Failure handling:** Agent cascade failures are logged but never block the
slot operation. A stuck agent should not prevent workspace state from being
saved (pause) or restored (resume). The user sees agent state changes via SSE
and can manually intervene on failed agents.

**REST wiring:** `LifecycleResource.pause()` and `resume()` call the
coordinator instead of `LifecycleManager` directly. All other lifecycle
operations (start, end, slotCreate, slotMerge, epicSetup, epicNext) remain
unchanged — they do not involve agent coordination.

### §2 Graceful Agent Shutdown

New method on `AgentProcessManager` — `gracefulShutdown(terminalName)`:

1. Check current agent state — if not RUNNING or STARTING, return immediately
2. Send `Escape` key via `tmux.sendKeys(terminalName, "Escape")` — interrupts
   any active generation, returns Claude to the input prompt
3. Wait 500ms for Claude to return to prompt
4. Send `"/exit\n"` via `tmux.sendKeys(terminalName, "/exit\n")` — Claude's
   own clean shutdown
5. Poll `pane_current_command` up to 10s (every 500ms) — wait for a shell
   command to appear, indicating Claude has exited
6. If shell appears: process exited cleanly, mark PAUSED
7. If 10s timeout: fall back to `treeKill`, then mark PAUSED
8. Persist PAUSED via `@trellis_agent_state` tmux option (same mechanism as
   existing `pauseAgent`)

**Distinction from `pauseAgent`:** The existing `pauseAgent` does an immediate
`treeKill` — appropriate for the single-agent pause button in the UI where the
user explicitly wants an immediate stop. `gracefulShutdown` is used by the
coordinator for cascade pauses where the extra seconds for a clean exit are
worth it.

**Why Escape works:** `tmux send-keys Escape` is delivered to the terminal
regardless of whether Claude is streaming, idle, or mid-tool-execution. In
Claude Code, Escape interrupts the current generation and returns to the input
prompt. If Claude is already idle at the prompt, Escape is a no-op.

**Why poll `pane_current_command` instead of `ProcessHandle.onExit()`:** During
the exit sequence the process tree may restructure as MCP servers shut down.
Polling `pane_current_command` for a shell return is the reliable signal that
everything's done.

**No prompt to commit:** Claude sessions handle interruption as a normal
occurrence. `claude -c` on resume picks up where it left off. The WIP commit
in the slot pause operation captures whatever file state is on disk — no need
for Claude to commit its own work before exiting.

### §3 Memory Pressure Monitoring and Eviction Queue

Extends the existing `AgentMonitorScheduler` poll cycle. No new scheduler —
piggybacks on the 5-second poll that already collects RSS per agent.

#### Thresholds

| Level | Threshold | Effect |
|-------|-----------|--------|
| Per-agent warning | 500 MB | Amber badge (already exists) |
| Per-agent critical | 1 GB | Red badge + eviction candidate |
| System aggregate | 80% of total RAM | Top consumers become eviction candidates |

#### Eviction Queue State

New field on `AgentProcessManager`:

```java
private final ConcurrentHashMap<String, EvictionCandidate> evictionCandidates =
        new ConcurrentHashMap<>();
```

```java
public record EvictionCandidate(
        String terminalName,
        long memoryBytes,
        Instant firstExceeded,
        EvictionReason reason
) {}

public enum EvictionReason { PER_AGENT_CRITICAL, SYSTEM_PRESSURE }
```

#### Monitor Logic (each poll cycle)

1. After collecting RSS for all agents, check per-agent thresholds
2. Agents exceeding 1 GB: add to eviction queue with `PER_AGENT_CRITICAL`
3. Check aggregate RSS against system threshold — use
   `OperatingSystemMXBean.getFreePhysicalMemorySize()` via
   `((com.sun.management.OperatingSystemMXBean) ManagementFactory.getOperatingSystemMXBean())`
   for system-wide physical memory (macOS-specific; trellis is macOS-only)
4. If system pressure: rank agents by RSS descending, add top consumers to
   eviction queue with `SYSTEM_PRESSURE`
5. When an agent drops below threshold (or is paused/stopped), remove from
   queue
6. Broadcast eviction queue changes on SSE topic `agent:eviction`

#### No Auto-Eviction

The eviction queue is purely advisory. The system never pauses an agent for
memory pressure on its own. The user sees candidates in the UI and decides
which to pause. This avoids the system killing an agent doing critical work.

### §4 Frontend

#### Eviction Queue Display

Integrated into the existing terminal tab group — not a separate view.

Each terminal entry already shows a memory badge that turns red at 500 MB.
Extended behaviour:

| Condition | Badge | Action |
|-----------|-------|--------|
| Warning (500 MB – 1 GB) | Amber | None — informational |
| Critical / eviction candidate | Red, pulsing | "Evict" button appears → calls `pauseAgent` |
| System pressure banner | — | Bar at top of terminal panel: "Agent memory: 3.2 GB / 4.0 GB limit" |

**SSE subscription:** Add `agent:eviction` to the existing push topic
subscription alongside `agent:state`. When an eviction event arrives, the
terminal entry gains/loses the eviction indicator.

#### Slot Pause/Resume UI

No changes needed. The existing pause/resume buttons on slots call the same
REST endpoints. The coordinator handles the cascade transparently — the only
visible difference is that after slot pause, agent state badges switch to
PAUSED, and after slot resume they switch to RUNNING. The user sees it happen
automatically.

#### No Protection Toggle

No "protect from eviction" toggle for now. The eviction queue is advisory —
the user can ignore candidates they don't want to pause. If protection becomes
needed, it's a single boolean on `TerminalInfo` added later.

### §5 REST API Changes

#### Modified Endpoints (internal routing only)

```
POST /api/lifecycle/pause/{slotId}    — unchanged URL, routes through coordinator
POST /api/lifecycle/resume/{slotId}   — unchanged URL, routes through coordinator
```

Response shape stays the same (`OperationResult`). Agent cascade failures are
logged, not surfaced in the response — the user sees agent state changes via
SSE in real time.

#### New Endpoint

```
GET /api/agents/eviction    → List<EvictionCandidate>
```

Snapshot of current eviction candidates for initial page load. After that,
SSE `agent:eviction` keeps the UI current.

No new endpoints for the cascade itself. The coordination is an
implementation detail of slot pause/resume.

## Testing Strategy

### Unit Tests

- **`SlotAgentCoordinatorTest`** — mock `LifecycleManager`,
  `AgentProcessManager`, `TerminalRegistry`. Verify:
  - Pause finds correct terminals by slot
  - Calls graceful shutdown before git ops
  - Resume calls git ops before agent restart
  - Resume does not restart agents if git ops failed
  - Failures in one agent don't block others
  - Terminals not belonging to the slot are untouched

- **`AgentProcessManagerTest` (graceful shutdown)** — mock `TmuxManager`.
  Verify:
  - Escape → /exit → poll sequence
  - treeKill fallback on timeout
  - No-op for agents not in RUNNING/STARTING state
  - PAUSED state persisted via `@trellis_agent_state`

- **Eviction queue** — feed canned RSS values to
  `AgentProcessManager.pollTerminalWithPsOutput()`. Verify:
  - Candidates appear at 1 GB threshold
  - Candidates removed when agent drops below threshold or is paused
  - System pressure candidates ranked by RSS descending
  - SSE events broadcast on queue changes

### Integration Tests (tmux required, `assumeTrue` guard)

- Full pause cascade: create terminals in a slot, start mock agents, call
  `coordinatedPause`, verify agents stopped and WIP committed
- Full resume cascade: from paused state, call `coordinatedResume`, verify
  branches restored and agents restarted with `claude -c`

### Frontend

- Manual browser verification of eviction badges, system pressure banner,
  and Evict button visibility

## Scope Boundary

### In scope
- SlotAgentCoordinator with coordinated pause/resume
- Graceful agent shutdown (Escape → /exit → fallback)
- Memory pressure monitoring and eviction queue
- Eviction queue UI (badges, banner, Evict button)
- SSE topic `agent:eviction`
- REST endpoint `GET /api/agents/eviction`

### Out of scope
- Auto-eviction (no automatic pause for memory — advisory only)
- Per-agent eviction protection toggle (future if needed)
- Configurable thresholds via UI (hardcoded for now, tuned by config property)
- Electron shell changes (sidecar-only)
