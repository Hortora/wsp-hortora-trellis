# Lifecycle Step Progress for All Work Contexts

**Issue:** Hortora/trellis#87
**Date:** 2026-09-27

## Overview

Issue #68 implemented lifecycle step progress (end, pause, resume) with real-time SSE visibility, but scoped it exclusively to slots via `SlotAgentCoordinator`. Standalone repos — repos with active work but no slot — have no step progress visibility. This spec generalises the lifecycle progress infrastructure from slot-scoped to context-scoped, where a "context" is either a slot or a standalone repo.

## Identity Model — `contextId`

Replace the slot-specific `slotId` key with a unified `contextId` string throughout the stack:

| Context type | `contextId` format | Agent lookup |
|---|---|---|
| Slot | `slot-{N}` (e.g. `slot-3`) | `terminalRegistry.list().filter(t -> slotId.equals(t.slot()))` |
| Standalone repo | `repo-{repoName}` (e.g. `repo-engine`) | `terminalRegistry.list().filter(t -> repoName.equals(t.repo()))` |

The `contextId` is an opaque token. The coordinator parses the prefix to determine context type and agent lookup strategy. One agent per standalone repo path (confirmed constraint).

## Backend Changes

### OperationProgress record

Rename `slotId` field to `contextId`:

```java
public record OperationProgress(
    String operationId,
    String operationType,
    String contextId,        // was: slotId
    OperationState state,
    List<StepProgress> steps,
    Instant startedAt,
    Instant completedAt,
    String errorMessage
) { /* withStep, withState, withFailed unchanged except field name */ }
```

### LifecycleOperationTracker

Rename internal map from `slotToOperation` to `contextToOperation`. All methods that accepted `slotId` now accept `contextId`:

- `startOperation(String type, String contextId, List<String> stepNames)`
- `getActiveOperation(String contextId)`

SSE broadcast payloads change `slotId` → `contextId`.

Eviction, persistence, and startup recovery are unchanged — they operate on `operationId`, not the context key.

### LifecycleCoordinator (renamed from SlotAgentCoordinator)

Rename the class to `LifecycleCoordinator` to reflect its broadened scope. IntelliJ rename refactor updates all references.

**New helper method:**

```java
private List<TerminalInfo> findTerminals(String contextId) {
    if (contextId.startsWith("slot-")) {
        var slotId = contextId.substring(5);
        return terminalRegistry.list().stream()
                .filter(t -> slotId.equals(t.slot()))
                .toList();
    } else if (contextId.startsWith("repo-")) {
        var repoName = contextId.substring(5);
        return terminalRegistry.list().stream()
                .filter(t -> repoName.equals(t.repo()) && t.slot() == null)
                .toList();
    }
    return List.of();
}
```

The `repo-*` branch guards with `t.slot() == null` — a repo inside a slot is owned by the slot context, not addressable as a standalone repo. Null-safe: `repoName.equals(t.repo())` since `t.repo()` can be null.

The three agent methods (`stopAllAgents`, `shutdownAgents`, `resumeCoordinatorPausedAgents`) call `findTerminals(contextId)` instead of each doing their own slot filter. Rename them to drop the "Slot" prefix.

**Async methods** — parameter changes only:

```java
public OperationProgress coordinatedEndAsync(String contextId, Path workspaceRoot)
public OperationProgress coordinatedPauseAsync(String contextId, Path workspaceRoot)
public OperationProgress coordinatedResumeAsync(String contextId, Path workspaceRoot)
```

**Locking** — the `asyncSlotLocks` map becomes `asyncContextLocks`. Keyed on `contextId` — this naturally gives per-slot and per-repo mutual exclusion.

**Sync methods** (`coordinatedEnd`, `coordinatedPause`, `coordinatedResume`) — same rename of parameter from `slotId` to `contextId`. The `slotLocks` map becomes `contextLocks`.

### LifecycleManager

The async step methods (`endSteps`, `pauseSteps`, `resumeSteps`) accept `slotId` but don't use it — they only use `workspaceRoot` for locking and script execution. Rename the parameter to `contextId` for consistency but no logic change.

The sync methods (`end`, `pause`, `resume`) — same parameter rename.

### LifecycleResource

REST endpoints change from `/{slotId}` to `/{contextId}`:

```
POST /api/lifecycle/end/{contextId}     -> coordinator.coordinatedEndAsync(contextId, ...)
POST /api/lifecycle/pause/{contextId}   -> coordinator.coordinatedPauseAsync(contextId, ...)
POST /api/lifecycle/resume/{contextId}  -> coordinator.coordinatedResumeAsync(contextId, ...)
GET  /api/lifecycle/operations/{operationId}          (unchanged)
GET  /api/lifecycle/operations?context={contextId}    (was: ?slot={slotId})
```

The `@PathParam` annotation changes from `slotId` to `contextId`. The query parameter on the GET endpoint changes from `slot` to `context`.

### Step definitions per context type

For standalone repos, the step lists are the same as for slots, except the agent coordination step names are context-appropriate:

| Operation | Slot steps | Standalone repo steps |
|---|---|---|
| **end** | `stop-agents` → `rebase` → `push` → `stamp` | `stop-agent` → `rebase` → `push` → `stamp` |
| **pause** | `shutdown-agents` → `commit-wip` → `push-and-stack` | `shutdown-agent` → `commit-wip` → `push-and-stack` |
| **resume** | `checkout-branches` → `rebase` → `reset-wip` → `resume-agents` | `checkout-branches` → `rebase` → `reset-wip` → `resume-agent` |

The script steps (rebase, push, stamp, commit-wip, push-and-stack, checkout-branches, reset-wip) are identical — they operate on `workspaceRoot`, which both contexts provide. The agent steps differ only in lookup strategy (handled by `findTerminals`).

Step name singularity (`stop-agent` vs `stop-agents`) is cosmetic — the coordinator can use the same names for both. Keep the plural form for simplicity since `findTerminals` returns a list regardless.

### MCP tool changes

`TrellisTools.trellis_lifecycle` — if the MCP tool currently accepts `slotId`, update to accept `contextId`. The tool dispatches to the same REST endpoint.

## Frontend Changes

### `lifecycle-progress.ts` — new shared component

Pure rendering component — receives operation state from the parent, renders the step progress section. No SSE subscription, no fetch. Parents own the SSE connection and pass operation state as a property.

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
  contextId: string;
  state: 'RUNNING' | 'COMPLETED' | 'FAILED';
  steps: StepProgress[];
  errorMessage?: string;
}

@customElement('lifecycle-progress')
export class LifecycleProgress extends LitElement {
  @property({ type: Object }) operation: OperationProgress | null = null;
  @state() private _expandedStep: string | null = null;
}
```

**Responsibilities:**
- Render the lifecycle section (same template as current `_renderLifecycle()`)
- Manage expand/collapse state for step output (`_expandedStep`)
- Return `nothing` when `operation` is null

**Styles:** Move the lifecycle-specific CSS (step icons, spinner, output area) from `slot-detail.ts` into the component.

The `StepProgress` and `OperationProgress` interfaces are exported from this file and imported by both parents.

### `slot-detail.ts` changes

- Remove `_renderLifecycle()` template (moves to component)
- Keep `_operation`, `_handleLifecycleEvent()`, and `lifecycle:progress` in the existing EventSource subscription — parent owns SSE and operation state
- In `_renderSidebar()`, replace `${this._renderLifecycle()}` with:
  ```typescript
  html`<lifecycle-progress .operation=${this._operation}></lifecycle-progress>`
  ```
- `_lifecycleAction()` updates URL path to use `slot-${this.slotNumber}` as the contextId
- On operation complete/fail (in `_handleLifecycleEvent`): same logic as today — clear `_actionInProgress`, reload slot/terminals, auto-dismiss after 10s

### `repo-detail.ts` changes

- Add lifecycle buttons to `_renderToolbar()` — `pause` and `end` buttons, shown only when `repo.branch !== 'main'`
- Add `_operation` state, `_actionInProgress` state
- Add `lifecycle:progress` to the existing EventSource subscription in `_subscribeEvents()` (currently subscribes to `agent:state` only)
- Add `_handleLifecycleEvent(data)` — same logic as slot-detail, filtering by `contextId === "repo-${this.repoName}"`
- Add `lifecycle-progress` element to `_renderSidebar()`:
  ```typescript
  html`<lifecycle-progress .operation=${this._operation}></lifecycle-progress>`
  ```
- Add `_lifecycleAction()` method:
  ```typescript
  private async _lifecycleAction(name: string) {
    this._actionInProgress = name;
    try {
      const res = await fetch(`/api/lifecycle/${name}/repo-${this.repoName}`, {
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
      this._operation = await res.json() as OperationProgress;
    } catch (e) {
      this._error = `${name} failed: ${e}`;
      this._actionInProgress = null;
    }
  }
  ```
- Wire toolbar buttons: `@click=${() => this._lifecycleAction('end')}`, `@click=${() => this._lifecycleAction('pause')}`
- On operation complete/fail (in `_handleLifecycleEvent`): clear `_actionInProgress`, reload repo/terminal, auto-dismiss after 10s
- On `connectedCallback`: fetch `GET /api/lifecycle/operations?context=repo-${this.repoName}` to restore state after page refresh

### Component registration

Add `lifecycle-progress.ts` to the webui build. Import in both `slot-detail.ts` and `repo-detail.ts`.

## Edge Cases

- **Standalone repo with no terminal/agent:** `findTerminals` returns empty list. Agent coordination step completes immediately (no agents to stop/shutdown/resume). Script steps run normally.
- **Repo inside a slot:** `findTerminals("repo-engine")` filters by `t.repo().equals("engine") && t.slot() == null`. A terminal belonging to slot 3 working on repo "engine" is NOT matched — it is owned by the slot context (`slot-3`), not addressable as a standalone repo.
- **contextId not recognised (no `slot-` or `repo-` prefix):** Coordinator returns HTTP 400 Bad Request.
- **Repo on main branch:** `repo-detail.ts` hides lifecycle buttons when `repo.branch === 'main'`. No lifecycle operations are valid on main.
- **SSE reconnection:** Parent re-fetches `GET /api/lifecycle/operations?context={contextId}` on EventSource reconnect, passes recovered state to `<lifecycle-progress>` component.
- **Concurrent operations:** Same per-context Semaphore(1) as before, now keyed on `contextId`. HTTP 409 for conflicts.
- **Existing persisted operations with `slotId`:** On startup, `LifecycleOperationTracker.loadOnStartup()` reads JSON files. Existing files use `slotId` field name. Jackson deserialization will fail on the renamed field. Add `@JsonAlias("slotId")` on the `contextId` parameter in `OperationProgress` for backward compatibility during the transition. Old files will be evicted within 1 hour (FAILED_TTL).

## Testing

### Java unit tests

**LifecycleOperationTrackerTest:**
- Existing tests updated: `slotId` → `contextId` in all assertions and setup
- New: `startOperation` with `"repo-engine"` contextId works identically to `"slot-3"`
- New: `getActiveOperation("repo-engine")` returns correct operation
- New: concurrent operations on `"slot-3"` and `"repo-engine"` are independent

**LifecycleCoordinatorTest (renamed from SlotAgentCoordinatorTest):**
- Existing tests updated: parameter rename, URL path update
- New: `coordinatedEndAsync("repo-engine", workspaceRoot)` — finds terminal by repo name, executes steps
- New: `coordinatedEndAsync("repo-engine", ...)` with no matching terminal — agent step completes immediately, script steps execute

**LifecycleResourceTest:**
- Existing tests updated: path param from `"3"` to `"slot-3"`, query param from `slot=3` to `context=slot-3`
- New: `POST /api/lifecycle/end/repo-engine` returns 202 with OperationProgress

### Frontend

Verify visually via dev server. The `<lifecycle-progress>` component is exercised by:
1. Triggering pause/resume on a slot from slot-detail modal (existing flow, now via component)
2. Triggering end/pause on a standalone repo from repo-detail view (new flow)

## References

- `LifecycleOperationTracker.java` — operation tracking, SSE broadcast, persistence
- `SlotAgentCoordinator.java` → `LifecycleCoordinator.java` — async orchestration
- `LifecycleManager.java` — script execution step methods
- `LifecycleResource.java` — REST endpoints
- `OperationProgress.java` — progress record
- `TerminalInfo.java:6-7` — `slot` and `repo` fields for agent lookup
- `slot-detail.ts:460-493` — current `_renderLifecycle()` template
- `slot-detail.ts:261-310` — current SSE subscription and event handling
- `repo-detail.ts:271-310` — current sidebar (no lifecycle section)
- Issue #68 spec — `docs/specs/issue-68-slot-lifecycle-steps/2026-09-26-slot-lifecycle-steps-design.md`
- Issue #68 decisions — `docs/specs/issue-68-slot-lifecycle-steps/decisions.md` (D5-D7 on persistence/state)
