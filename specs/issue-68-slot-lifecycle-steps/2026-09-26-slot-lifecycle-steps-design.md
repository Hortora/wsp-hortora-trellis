# Slot Lifecycle Step Progress Display

**Issue:** Hortora/trellis#68
**Date:** 2026-09-26

## Overview

When a slot undergoes a lifecycle operation (end, pause, resume, start), the slot detail modal shows real-time step-by-step progress. Each operation is composed of discrete steps (e.g., end = stop agents → rebase → push → stamp), and the UI renders each step with done/active/pending/failed state plus expandable captured output.

## Architecture

### Backend — LifecycleOperationTracker

New `@ApplicationScoped` CDI bean in `io.hortora.trellis.lifecycle`:

```java
record StepProgress(
    String name,
    StepState state,     // PENDING, RUNNING, DONE, FAILED
    String summary,      // one-line description
    String stdout,       // captured raw stdout (null until step completes)
    String stderr,       // captured raw stderr (null until step completes)
    Instant startedAt,
    Instant completedAt
) {}

record OperationProgress(
    String operationId,
    String operationType,  // "end", "pause", "resume", "start"
    String slotId,
    OperationState state,  // RUNNING, COMPLETED, FAILED
    List<StepProgress> steps,
    Instant startedAt,
    Instant completedAt
) {}
```

`LifecycleOperationTracker` responsibilities:
- Holds `ConcurrentHashMap<String, OperationProgress>` for active/recent operations
- `startOperation(type, slotId, stepNames)` → creates OperationProgress, returns operationId (UUID)
- `stepStarted(operationId, stepName)` → updates step to RUNNING, broadcasts SSE
- `stepCompleted(operationId, stepName, stdout, stderr)` → updates step to DONE, broadcasts SSE
- `stepFailed(operationId, stepName, stdout, stderr)` → updates step to FAILED, marks operation FAILED, broadcasts SSE
- `operationCompleted(operationId)` → marks operation COMPLETED, broadcasts SSE
- `getProgress(operationId)` → returns current state (for GET endpoint)
- `getActiveOperation(slotId)` → returns active operation for a slot (for page refresh)
- Evicts completed operations after 5 minutes (they are short-lived, no need for persistence)

SSE publishing uses `EventBroadcaster.broadcast("lifecycle:progress", json)` where the JSON payload includes the operationId, the step name, the new state, and captured output.

### Step definitions per operation

Each lifecycle operation declares its step list upfront so the tracker (and the UI) know the full pipeline before execution begins:

| Operation | Steps |
|-----------|-------|
| **end** | `stop-agents` → `rebase` → `push` → `stamp` |
| **pause** | `shutdown-agents` → `commit-wip` → `push-and-stack` |
| **resume** | `checkout-branches` → `rebase` → `reset-wip` → `resume-agents` |
| **start** | `route` → `create-branches` → `scaffold` |

Agent coordination steps (`stop-agents`, `shutdown-agents`, `resume-agents`) come from `SlotAgentCoordinator`. Script steps come from `LifecycleManager`. The tracker treats them uniformly.

### LifecycleManager changes

Current: methods like `end()` are synchronous, called from the HTTP request thread inside `withLock()`, running scripts sequentially and returning an `OperationResult`.

New: methods accept a `LifecycleOperationTracker` and an `operationId`, report progress at each step boundary, and run on a managed executor thread.

```java
public void endAsync(String slotId, Path workspaceRoot,
                     LifecycleOperationTracker tracker, String operationId)
        throws IOException, InterruptedException, ConcurrentOperationException {
    withLock(workspaceRoot.toString(), "end", () -> {
        tracker.stepStarted(operationId, "rebase");
        var rebaseResult = scriptRunner.run("work-end", "land_branch.py",
                List.of("rebase", workspaceRoot.toString()));
        tracker.stepCompleted(operationId, "rebase",
                rebaseResult.rawStdout(), rebaseResult.stderr());
        if (!rebaseResult.success()) {
            tracker.stepFailed(operationId, "rebase",
                    rebaseResult.rawStdout(), rebaseResult.stderr());
            return rebaseResult;
        }

        tracker.stepStarted(operationId, "push");
        var pushResult = scriptRunner.run("work-end", "land_branch.py",
                List.of("push", workspaceRoot.toString()));
        // ... same pattern for each step
    });
}
```

**Lock transfer:** The `withLock()` call moves into the background thread — the lock is acquired and released within the async execution, not by the HTTP handler thread. This preserves the existing mutual exclusion guarantee.

### SlotAgentCoordinator changes

`coordinatedEnd`, `coordinatedPause`, `coordinatedResume` become the async orchestrators. They:
1. Accept the tracker and operationId
2. Report agent coordination as named steps (e.g., `stop-agents`)
3. Delegate to `LifecycleManager` async methods for the script steps
4. Run on a `@ManagedExecutor` thread pool

```java
public String coordinatedEndAsync(String slotId, Path workspaceRoot) {
    var lock = slotLocks.computeIfAbsent(slotId, k -> new ReentrantLock());
    if (!lock.tryLock()) {
        throw new ConcurrentOperationException("...");
    }

    var operationId = tracker.startOperation("end", slotId,
            List.of("stop-agents", "rebase", "push", "stamp"));

    executor.submit(() -> {
        try {
            tracker.stepStarted(operationId, "stop-agents");
            stopAllSlotAgents(slotId);
            tracker.stepCompleted(operationId, "stop-agents", "", "");

            lifecycleManager.endAsync(slotId, workspaceRoot, tracker, operationId);
            tracker.operationCompleted(operationId);
        } catch (Exception e) {
            tracker.operationFailed(operationId, e.getMessage());
        } finally {
            lock.unlock();
        }
    });

    return operationId;
}
```

### ScriptRunner changes

`OperationResult` currently stores parsed `Map<String, String> output` and `String stderr`. Add a `String rawStdout` field to preserve the unparsed stdout for display in the UI:

```java
public record OperationResult(
    boolean success,
    int exitCode,
    Map<String, String> output,
    String stderr,
    String rawStdout    // new — raw stdout before KEY=VALUE parsing
) {}
```

### REST API changes

**Modified endpoints** — return operationId instead of OperationResult:

```
POST /api/lifecycle/end/{slotId}     → { "operationId": "..." }
POST /api/lifecycle/pause/{slotId}   → { "operationId": "..." }
POST /api/lifecycle/resume/{slotId}  → { "operationId": "..." }
POST /api/lifecycle/start            → { "operationId": "..." }
```

HTTP 202 Accepted (not 200 OK) — the operation is accepted but not complete.
HTTP 409 Conflict — concurrent operation in progress (unchanged).

**New endpoint** — query current operation state:

```
GET /api/lifecycle/operations/{operationId}  → OperationProgress JSON
GET /api/lifecycle/operations?slot={slotId}  → OperationProgress JSON (active operation for slot)
```

The slot-scoped query is the primary access path for page refresh — the frontend stores the slotId, not the operationId.

### SSE topic

New topic: `lifecycle:progress`

Event payload:
```json
{
  "operationId": "uuid",
  "slotId": "3",
  "operationType": "end",
  "step": "rebase",
  "state": "RUNNING",
  "stdout": null,
  "stderr": null,
  "operationState": "RUNNING"
}
```

When a step completes or fails, `stdout` and `stderr` are populated with the captured output from that step.

Frontend subscribes via the existing `workspace-sse.ts` pattern — add `lifecycle:progress` to the topic list in `slot-detail.ts`'s `_subscribeEvents()`.

## Frontend — Slot Detail Sidebar

### TypeScript interfaces

```typescript
interface StepProgress {
  name: string;
  state: 'PENDING' | 'RUNNING' | 'DONE' | 'FAILED';
  summary: string;
  stdout: string | null;
  stderr: string | null;
}

interface OperationProgress {
  operationId: string;
  operationType: string;
  slotId: string;
  state: 'RUNNING' | 'COMPLETED' | 'FAILED';
  steps: StepProgress[];
}
```

### New state and SSE subscription

Add to `TrellisSlotDetail`:
```typescript
@state() private _operation: OperationProgress | null = null;
@state() private _expandedStep: string | null = null;
```

Subscribe to `lifecycle:progress` topic (filtered by `slotId === this.slotNumber`). On each event, update `_operation` with the new step state. When the operation completes or fails, keep the result visible for 10 seconds, then fade/collapse.

On `connectedCallback` (and after `_loadSlot`), fetch `GET /api/lifecycle/operations?slot={slotId}` to restore state after page refresh.

### Rendering — `_renderLifecycle()`

New method called from `_renderSidebar()` between the Status section and the Issues section. Only rendered when `_operation` is non-null.

Layout:
```
LIFECYCLE                          ← section header (h3, uppercase, matches sidebar style)
end                                ← operation type badge
  ✓ stop-agents                    ← done step (green check, muted text)
  ● rebase                         ← active step (blue dot, bright text, spinner)
  ○ push                           ← pending step (dim circle, dim text)
  ○ stamp                          ← pending step
  ▼ rebase output                  ← expand button (only on done/failed steps)
    ┌──────────────────────────┐
    │ SYNCED=yes               │   ← captured stdout in monospace, dark bg
    │ MODEL=single             │
    └──────────────────────────┘
```

Styling (consistent with existing plan section):
- Done steps: muted text, `✓` in green (`#86efac`)
- Active step: bright text, `●` in blue (`#93c5fd`), CSS spinner animation
- Pending steps: dim text (`#555`), `○` outline
- Failed step: `✗` in red (`#f87171`), error text, auto-expanded stderr
- Expand/collapse button: `0.7rem`, clickable, toggles `_expandedStep`
- Output area: `font-family: monospace; font-size: 0.7rem; background: #111; border: 1px solid #333; padding: 0.5rem; max-height: 150px; overflow-y: auto; white-space: pre-wrap`
- Section collapses (returns `nothing`) when `_operation` is null

### Lifecycle action flow changes

Current `_lifecycleAction()` awaits the full POST response. New flow:

```typescript
private async _lifecycleAction(name: string, url: string) {
  this._actionInProgress = name;
  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ workspaceRoot: this.workspaceRoot }),
    });
    if (!res.ok) {
      const body = await res.json().catch(() => null);
      this._error = body?.error ?? `${name} failed: HTTP ${res.status}`;
      return;
    }
    const { operationId } = await res.json();
    // SSE subscription handles step updates from here
    // _actionInProgress stays set until operation completes via SSE
  } catch (e) {
    this._error = `${name} failed: ${e}`;
    this._actionInProgress = null;
  }
}
```

Clear `_actionInProgress` when the SSE stream reports `operationState: 'COMPLETED'` or `'FAILED'`. On completion, reload slot data (`_loadSlot()`, `_loadTerminals()`).

## Edge Cases

- **Concurrent operation:** `SlotAgentCoordinator` lock prevents concurrent operations on the same slot. HTTP 409 returned. Frontend disables buttons via `_actionInProgress`.
- **Sidecar restart during operation:** In-memory state is lost. The background thread dies. Frontend SSE reconnects but no active operation exists. The operation is effectively cancelled. The slot's filesystem state is the source of truth — the user can retry.
- **Script timeout:** `ScriptRunner` has a 120s timeout per script. If a step times out, it's marked FAILED with the timeout error. The operation stops at the failed step.
- **Page refresh mid-operation:** `GET /api/lifecycle/operations?slot={slotId}` returns the current `OperationProgress`. Frontend picks up from the current state; subsequent SSE events update from there.
- **No active operation:** `_renderLifecycle()` returns `nothing`. The section is invisible — no empty state needed.
- **Operation completes while modal is closed:** The operation runs server-side regardless of modal visibility. When the modal reopens, the GET endpoint returns the completed result (if within the 5-minute eviction window) or nothing.
- **Multiple slots:** Each operation is scoped by slotId. The tracker supports concurrent operations on different slots.

## Testing

### Java unit tests

**LifecycleOperationTrackerTest:**
- `startOperation` creates progress with all steps PENDING
- `stepStarted` transitions step to RUNNING, broadcasts SSE
- `stepCompleted` transitions step to DONE, captures output
- `stepFailed` transitions step and operation to FAILED
- `operationCompleted` transitions operation to COMPLETED
- `getActiveOperation` returns active operation for slotId
- `getActiveOperation` returns null after eviction timeout
- Concurrent operations on different slots succeed
- Concurrent operations on same slot (shouldn't happen due to locking, but defensive)

**SlotAgentCoordinator async tests:**
- `coordinatedEndAsync` returns operationId immediately
- Steps execute in order on background thread
- Agent stop failure marks step FAILED, halts operation
- Lock is released after operation completes/fails

**ScriptRunner:**
- `rawStdout` preserved alongside parsed output

### Frontend

No unit tests for rendering (Lit template). Verify visually via dev server. The lifecycle section is exercised by triggering pause/resume on a slot from the slot detail modal.

## References

- `LifecycleManager.java:50-66` — current synchronous end() with sequential scripts
- `SlotAgentCoordinator.java:56-68` — coordinated end with agent shutdown
- `ScriptRunner.java:29-74` — script execution and KEY=VALUE parsing
- `OperationResult.java` — current result record
- `EventBroadcaster` (casehub-pages-push) — SSE broadcast mechanism
- `FileWatcherService.java:112` — broadcaster.broadcast pattern
- `slot-detail.ts:565-584` — current `_lifecycleAction()` implementation
- `slot-detail.ts:296-362` — `_renderSidebar()` structure
- `workspace-sse.ts` — shared SSE subscription helper
- `CoordinatorEvent.java:21-24` — existing LifecycleOperationEvent record
- Issue #67 spec — plan batch state in sidebar (UI pattern precedent)
