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
- `startOperation(type, slotId, stepNames)` → creates OperationProgress, returns operationId (UUID). If an existing FAILED or COMPLETED operation exists for the same slotId, it is auto-dismissed (removed from memory and file) before the new operation is created
- `stepStarted(operationId, stepName)` → updates step to RUNNING, broadcasts SSE
- `stepCompleted(operationId, stepName, stdout, stderr)` → updates step to DONE, broadcasts SSE
- `stepFailed(operationId, stepName, stdout, stderr)` → updates step to FAILED, marks operation FAILED, broadcasts SSE
- `operationFailed(operationId, errorMessage)` → transitions any step currently in RUNNING state to FAILED (with the error message as stderr), then marks operation FAILED with error detail, broadcasts SSE. This enforces the invariant that no step remains RUNNING when the operation is FAILED — covering script timeouts (`IOException`), missing scripts, thread interruption, and any other exception thrown after `stepStarted()` but before `stepFailed()`/`stepCompleted()`
- `operationCompleted(operationId)` → marks operation COMPLETED, broadcasts SSE
- `getProgress(operationId)` → returns current state (for GET endpoint)
- `getActiveOperation(slotId)` → returns active operation for a slot (for page refresh). Returns the most recent operation regardless of state (RUNNING, COMPLETED, or FAILED within the eviction window)
- Eviction via `@Scheduled(every = "60s")` sweep with differentiated retention: COMPLETED operations evict after 5 minutes, FAILED operations evict after 1 hour. Eviction removes both the in-memory entry and the backing `.json` file
- **File-backed durability:** Each step transition writes the full `OperationProgress` to `.trellis/operations/{operationId}.json` in the workspace root. On startup, `LifecycleOperationTracker` scans this directory and loads any incomplete operations into memory (state = RUNNING or a step in RUNNING state). This handles sidecar restart mid-operation — the tracker recovers the last known state and marks the interrupted step as FAILED (since the background thread that was executing it no longer exists)
- File writes use atomic rename (`write to .tmp`, `rename to .json`) to avoid partial reads

**Immutability:** `OperationProgress` and `StepProgress` records use copy-on-write semantics. Each state transition creates a new `OperationProgress` instance with `List.copyOf(steps)` and atomically replaces the reference in the `ConcurrentHashMap`. `getProgress()` returns the current reference — an immutable snapshot that is never mutated. This is critical for thread safety: between the POST handler calling `getProgress()` and the HTTP framework serializing the response to JSON, the background thread may advance step state. The snapshot returned to the HTTP handler points to the old immutable state, preventing `ConcurrentModificationException` and ensuring the client receives a consistent view.

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

**`end`, `pause`, `resume`** are orchestrated by `SlotAgentCoordinator` via their respective `coordinatedXxxAsync` methods — async with step progress tracking.

**`start` remains synchronous** — no async orchestrator, no step progress UI. Rationale:
1. The slot doesn't exist during `start` — there's no `TrellisSlotDetail` modal to render progress in
2. `start` is invoked from workspace-level views (`TrellisEpicDashboard`, `EnhancedRecommendationCard`), not the slot detail sidebar
3. `start` has no agent coordination step and completes quickly (route → create-branches → scaffold are lightweight git operations)
4. `LifecycleResource.start()` continues to call `manager.start()` directly, returning `OperationResult` synchronously

Step progress for `start` in a workspace-level progress indicator is a future enhancement (tracked separately from this issue).

### LifecycleManager changes

Current: methods like `end()` are synchronous, called from the HTTP request thread inside `withLock()`, running scripts sequentially and returning an `OperationResult`.

**Existing synchronous methods (`end`, `pause`, `resume`, `start`) are retained.** They continue to work for the `LifecycleActionExecutor` path (coordinator action system), which requires synchronous `ActionResult execute(ProposedAction)` semantics. No changes to these methods.

New: async step variants are added for use from `SlotAgentCoordinator`. These methods acquire the workspace lock directly via `tryLock`/`unlock` (not via `withLock` — they skip CDI event firing, which the coordinator handles). They accept a `LifecycleOperationTracker` and an `operationId`, report progress at each step boundary, and throw `StepFailedException` on step failure.

```java
public void endSteps(String slotId, Path workspaceRoot,
                     LifecycleOperationTracker tracker, String operationId)
        throws IOException, InterruptedException, ConcurrentOperationException,
               StepFailedException {
    if (!tryLock(workspaceRoot.toString())) {
        throw new ConcurrentOperationException("Workspace operation in progress");
    }
    try {
        tracker.stepStarted(operationId, "rebase");
        var rebaseResult = scriptRunner.run("work-end", "land_branch.py",
                List.of("rebase", workspaceRoot.toString()));
        if (!rebaseResult.success()) {
            tracker.stepFailed(operationId, "rebase",
                    rebaseResult.rawStdout(), rebaseResult.stderr());
            throw new StepFailedException("rebase");
        }
        tracker.stepCompleted(operationId, "rebase",
                rebaseResult.rawStdout(), rebaseResult.stderr());

        tracker.stepStarted(operationId, "push");
        var pushResult = scriptRunner.run("work-end", "land_branch.py",
                List.of("push", workspaceRoot.toString()));
        // ... same pattern for each step
    } finally {
        unlock(workspaceRoot.toString());
    }
}
```

**`StepFailedException`:** A checked exception thrown after `tracker.stepFailed()` has already been called. Signals to the coordinator that a step failed and the tracker has been notified — the coordinator's catch block is a no-op for this exception type. This prevents the caller from unconditionally calling `operationCompleted()` after a step failure.

**Workspace lock:** The async variants acquire `LifecycleManager.tryLock(workspaceRoot)` / `unlock(workspaceRoot)` directly — the same package-private lock used by `withLock()`. This provides mutual exclusion against non-coordinator operations (`slotCreate`, `slotMerge`, `epicSetup`, `epicNext`) that share the same workspace. Without this, the coordinator's per-slot Semaphore alone does NOT prevent concurrent git mutations from different operation types on the same workspace.

Lock acquisition order is consistent and deadlock-free:
1. Coordinator Semaphore(slotId) — acquired on HTTP thread, released on executor thread
2. LifecycleManager lock(workspaceRoot) — acquired and released on executor thread
3. Non-coordinator operations only acquire LifecycleManager lock(workspaceRoot) — no Semaphore

**CDI events:** In the async path, `LifecycleOperationEvent` and `WorkspaceChanged` events are fired by the coordinator after operation completion (see SlotAgentCoordinator changes). The async step methods have no CDI event responsibilities.

### SlotAgentCoordinator changes

**Existing synchronous methods (`coordinatedEnd`, `coordinatedPause`, `coordinatedResume`) are retained** for the `LifecycleActionExecutor` path. The action system calls these synchronously from its `action-executor` thread and translates the `OperationResult` into `ActionResult`. No changes needed to `ActionExecutor` or `LifecycleActionExecutor`.

New async orchestrators are added: `coordinatedEndAsync`, `coordinatedPauseAsync`, `coordinatedResumeAsync`. They:
1. Acquire a per-slot `Semaphore(1)` (not `ReentrantLock` — semaphores have no thread affinity, so the HTTP thread can acquire and the executor thread can release)
2. Create the operation in the tracker and snapshot the initial `OperationProgress`
3. Report agent coordination as named steps (e.g., `stop-agents`)
4. Delegate to `LifecycleManager` async step methods (which acquire the workspace lock internally)
5. Fire `LifecycleOperationEvent` CDI event and `WorkspaceChanged` event on completion
6. Run on a `@ManagedExecutor` thread pool

`start` is NOT included in the async pattern — see §Step definitions for rationale.

```java
private final ConcurrentHashMap<String, Semaphore> slotLocks = new ConcurrentHashMap<>();

public OperationProgress coordinatedEndAsync(String slotId, Path workspaceRoot) {
    var semaphore = slotLocks.computeIfAbsent(slotId, k -> new Semaphore(1));
    if (!semaphore.tryAcquire()) {
        throw new ConcurrentOperationException("...");
    }

    var operationId = tracker.startOperation("end", slotId,
            List.of("stop-agents", "rebase", "push", "stamp"));
    var progress = tracker.getProgress(operationId);

    executor.submit(() -> {
        try {
            tracker.stepStarted(operationId, "stop-agents");
            stopAllSlotAgents(slotId);
            tracker.stepCompleted(operationId, "stop-agents", "", "");

            lifecycleManager.endSteps(slotId, workspaceRoot, tracker, operationId);
            tracker.operationCompleted(operationId);
            fireLifecycleOperationEvent("end", true, null);
            fireWorkspaceChanged(workspaceRoot);
        } catch (StepFailedException e) {
            // tracker.stepFailed() already called inside endSteps — no action needed
            fireLifecycleOperationEvent("end", false, e.getMessage());
        } catch (Exception e) {
            tracker.operationFailed(operationId, e.getMessage());
            fireLifecycleOperationEvent("end", false, e.getMessage());
        } finally {
            semaphore.release();
        }
    });

    return progress;
}
```

The coordinator now owns both notification paths:
- **SSE** (`lifecycle:progress`): fired by the tracker on each step transition
- **CDI** (`LifecycleOperationEvent`): fired by the coordinator on operation completion, feeding the `EventRing` → `EventAccumulator` pipeline. This event continues to fire with the same semantics as today — `(operation, success, detail)` — preserving backward compatibility with the coordinator event pipeline

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

### Threading model

The background executor is a `@ManagedExecutor`-injected instance:

```java
@Inject ManagedExecutor executor;
```

This is the first `ManagedExecutor` usage in the project — the existing `ActionService` uses a plain `Executors.newSingleThreadExecutor()`. `ManagedExecutor` is the correct Quarkus pattern for new thread pools: it propagates CDI context and participates in graceful shutdown. CDI context propagation is not strictly required here (all beans are `@ApplicationScoped` and `Event.fire()`/`Event.fireAsync()` work from any thread with application-scoped observers), but adopting the managed pattern avoids subtle issues if request-scoped beans are introduced in the future.

### REST API changes

**Async endpoints** — return full `OperationProgress` so the frontend has immediate renderable state:

```
POST /api/lifecycle/end/{slotId}     → OperationProgress JSON (HTTP 202 Accepted)
POST /api/lifecycle/pause/{slotId}   → OperationProgress JSON (HTTP 202 Accepted)
POST /api/lifecycle/resume/{slotId}  → OperationProgress JSON (HTTP 202 Accepted)
```

HTTP 202 Accepted — the operation is accepted but not complete. The response body is the full `OperationProgress` with all steps in PENDING state and the operation in RUNNING state. SSE events update from this initial state.

HTTP 409 Conflict — concurrent operation in progress (unchanged).

**Synchronous endpoint** — `start` returns `OperationResult` as before (unchanged from current behavior):

```
POST /api/lifecycle/start            → OperationResult JSON (HTTP 200 OK)
```

`LifecycleResource.start()` continues to call `manager.start()` directly. No async orchestration needed (see §Step definitions for rationale).

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

**Operation-level failure** (from `operationFailed`) uses the same topic with `step: null`:
```json
{
  "operationId": "uuid",
  "slotId": "3",
  "operationType": "end",
  "step": null,
  "state": null,
  "stdout": null,
  "stderr": null,
  "operationState": "FAILED",
  "errorMessage": "Workspace operation in progress"
}
```

The frontend SSE handler checks `step` first: if non-null, update the matching step in `_operation.steps`. If null, update `_operation.state` directly and display `errorMessage` as the failure reason.

Frontend subscribes directly in `slot-detail.ts`'s `_subscribeEvents()` — add `lifecycle:progress` to the existing `EventSource` URL alongside `agent:state,agent:eviction`. This is a component-scoped subscription, NOT added to the global `ALL_WORKSPACE_TOPICS` in `workspace-sse.ts` (lifecycle progress is only relevant to the slot detail view).

On SSE reconnection (EventSource `onerror` → automatic reconnect), re-fetch `GET /api/lifecycle/operations?slot={slotId}` to recover any missed events during the connection gap.

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
      this._actionInProgress = null;
      return;
    }
    // POST returns full OperationProgress — all steps PENDING, operation RUNNING
    this._operation = await res.json() as OperationProgress;
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

- **Concurrent operation:** `SlotAgentCoordinator` semaphore prevents concurrent operations on the same slot. HTTP 409 returned. Frontend disables buttons via `_actionInProgress`.
- **Retry after failure:** When the user re-clicks a lifecycle action after a FAILED operation, `startOperation` auto-dismisses the previous FAILED operation for the same slot (removes from memory and file). The new operation starts fresh. No separate "dismiss" or "retry" UI is needed — the existing lifecycle buttons serve as the retry mechanism.
- **Sidecar restart during operation:** On restart, `LifecycleOperationTracker` loads incomplete operations from `.trellis/operations/`. Any step in RUNNING state is marked FAILED (the thread is gone). The frontend re-fetches via the GET endpoint and sees the partially-completed operation with the interrupted step marked as failed. The user can see where it stopped and retry.
- **Script timeout:** `ScriptRunner` has a 120s timeout per script. The timeout throws `IOException`, which propagates to the coordinator's `catch (Exception e)` → `tracker.operationFailed()`. `operationFailed` transitions the RUNNING step to FAILED (with the timeout error) before marking the operation FAILED. The same mechanism handles missing script files and thread interruption.
- **Page refresh mid-operation:** `GET /api/lifecycle/operations?slot={slotId}` returns the current `OperationProgress`. Frontend picks up from the current state; subsequent SSE events update from there.
- **No active operation:** `_renderLifecycle()` returns `nothing`. The section is invisible — no empty state needed.
- **Operation completes while modal is closed:** The operation runs server-side regardless of modal visibility. When the modal reopens, the GET endpoint returns the completed result (if within the eviction window) or nothing.
- **Multiple slots:** Each operation is scoped by slotId. The tracker supports concurrent operations on different slots.
- **FAILED operation visibility:** FAILED operations remain visible for 1 hour (vs 5 minutes for COMPLETED), giving the user time to investigate stderr output. They are auto-dismissed when a new operation starts for the same slot.

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
