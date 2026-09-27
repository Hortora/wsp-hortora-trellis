# Lifecycle Step Progress for All Work Contexts

**Issue:** Hortora/trellis#87
**Date:** 2026-09-27

## Overview

Issue #68 implemented lifecycle step progress (end, pause, resume) with real-time SSE visibility, but scoped it exclusively to slots via `SlotAgentCoordinator`. Standalone repos — repos with active work but no slot — have no step progress visibility. This spec generalises the lifecycle progress infrastructure from slot-scoped to context-scoped, where a "context" is either a slot or a standalone repo.

## Identity Model — `WorkContext`

Replace the slot-specific `slotId` key with a sealed `WorkContext` interface that gives compile-time exhaustiveness in Java code, with a string `contextId` as the wire/storage representation:

```java
public sealed interface WorkContext {
    record SlotContext(String slotId) implements WorkContext {}
    record RepoContext(String repoName) implements WorkContext {}

    default String key() {
        return switch (this) {
            case SlotContext s -> "slot-" + s.slotId();
            case RepoContext r -> "repo-" + r.repoName();
        };
    }

    static WorkContext parse(String contextId) {
        if (contextId.startsWith("slot-")) return new SlotContext(contextId.substring(5));
        if (contextId.startsWith("repo-")) return new RepoContext(contextId.substring(5));
        throw new IllegalArgumentException("Unknown context type: " + contextId);
    }
}
```

| Context type | `WorkContext` record | `key()` wire format | Agent lookup |
|---|---|---|---|
| Slot | `SlotContext("3")` | `"slot-3"` | `terminalRegistry.list().filter(t -> slotId.equals(t.slot()))` |
| Standalone repo | `RepoContext("engine")` | `"repo-engine"` | `terminalRegistry.list().filter(t -> repoName.equals(t.repo()) && t.slot() == null)` |

**Two-layer identity:** `WorkContext` is the coordination/tracking identity (which terminals to manage, which operation to track). `workspaceRoot` is the execution identity (which directory to run scripts in). LifecycleManager only uses `workspaceRoot` — it has no knowledge of `WorkContext`. The coordinator bridges these layers.

String `contextId` (`key()`) is used at wire boundaries: REST path params, SSE payloads, JSON-serialized `OperationProgress`, and frontend code. `WorkContext.parse()` converts from string to type at the system boundary (REST resource, JSON deserialization). Internal Java code uses the typed `WorkContext` throughout. One agent per standalone repo path (confirmed constraint).

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

**New helper method — pattern match on `WorkContext`:**

```java
private List<TerminalInfo> findTerminals(WorkContext ctx) {
    return switch (ctx) {
        case SlotContext s -> terminalRegistry.list().stream()
            .filter(t -> s.slotId().equals(t.slot())).toList();
        case RepoContext r -> terminalRegistry.list().stream()
            .filter(t -> r.repoName().equals(t.repo()) && t.slot() == null).toList();
    };
}
```

Compile-time exhaustiveness: if a third context type is added, the compiler forces a new case. The `RepoContext` branch guards with `t.slot() == null` — a repo inside a slot is owned by the slot context, not addressable as a standalone repo. Null-safe: `r.repoName().equals(t.repo())` — the non-null record component drives the `equals()` call, so null `t.repo()` returns `false` without NPE.

The three agent methods (`stopAllAgents`, `shutdownAgents`, `resumeCoordinatorPausedAgents`) call `findTerminals(ctx)` instead of each doing their own slot filter. Rename them to drop the "Slot" prefix.

**Async methods** — accept typed `WorkContext`:

```java
public OperationProgress coordinatedEndAsync(WorkContext context, Path workspaceRoot)
public OperationProgress coordinatedPauseAsync(WorkContext context, Path workspaceRoot)
public OperationProgress coordinatedResumeAsync(WorkContext context, Path workspaceRoot)
```

Internally, these pass `context.key()` to `tracker.startOperation()` (which needs the string form for map keys and JSON persistence) and `context` to `findTerminals()`.

**Locking** — the `asyncSlotLocks` map becomes `asyncContextLocks`. Keyed on `context.key()` — this naturally gives per-slot and per-repo mutual exclusion.

**Sync methods** (`coordinatedEnd`, `coordinatedPause`, `coordinatedResume`) — accept `WorkContext context`. The `slotLocks` map becomes `contextLocks`, keyed on `context.key()`.

### LifecycleManager

LifecycleManager operates on `workspaceRoot` only — it has no knowledge of `WorkContext`. The `slotId` parameter on all methods (`end`, `pause`, `resume`, `endSteps`, `pauseSteps`, `resumeSteps`) is never referenced in the method bodies. Rename to `contextId` (string form, passed through from the coordinator for signature consistency) but no logic change. The coordinator bridges the two identity layers: it resolves `WorkContext` → terminals for agent management, then delegates to LifecycleManager with just `workspaceRoot` for script execution.

### LifecycleResource

REST endpoints change from `/{slotId}` to `/{contextId}`. The resource parses the string path param into a typed `WorkContext` at the boundary:

```java
@POST
@Path("/end/{contextId}")
@Consumes(MediaType.APPLICATION_JSON)
public Response end(@PathParam("contextId") String contextId, WorkspaceRequest req) {
    try {
        var ctx = WorkContext.parse(contextId);
        var progress = coordinator.coordinatedEndAsync(ctx,
                                                       java.nio.file.Path.of(req.workspaceRoot()));
        return Response.accepted(progress).build();
    } catch (IllegalArgumentException e) {
        return Response.status(Response.Status.BAD_REQUEST)
                       .entity(Map.of("error", e.getMessage())).build();
    } catch (ConcurrentOperationException e) {
        return Response.status(Response.Status.CONFLICT)
                       .entity(Map.of("error", e.getMessage())).build();
    }
}
```

`WorkContext.parse()` is the single boundary where string→type conversion happens. Invalid context IDs (no recognised prefix) get HTTP 400. Same pattern for `pause` and `resume` endpoints.

```
POST /api/lifecycle/end/{contextId}     -> WorkContext.parse() -> coordinator.coordinatedEndAsync(ctx, ...)
POST /api/lifecycle/pause/{contextId}   -> WorkContext.parse() -> coordinator.coordinatedPauseAsync(ctx, ...)
POST /api/lifecycle/resume/{contextId}  -> WorkContext.parse() -> coordinator.coordinatedResumeAsync(ctx, ...)
GET  /api/lifecycle/operations/{operationId}          (unchanged)
GET  /api/lifecycle/operations?context={contextId}    (was: ?slot={slotId})
```

The query parameter on the GET endpoint changes from `slot` to `context`. The GET endpoint uses the string form directly for tracker lookup (no parsing needed — the tracker maps by string key).

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

- **Standalone repo with no terminal/agent:** `findTerminals(RepoContext("engine"))` returns empty list. Agent coordination step completes immediately (no agents to stop/shutdown/resume). Script steps run normally.
- **Repo inside a slot:** `findTerminals(RepoContext("engine"))` filters by `r.repoName().equals(t.repo()) && t.slot() == null`. A terminal belonging to slot 3 working on repo "engine" is NOT matched — it is owned by the slot context (`SlotContext("3")`), not addressable as a standalone repo.
- **Invalid contextId string (no `slot-` or `repo-` prefix):** `WorkContext.parse()` throws `IllegalArgumentException`, resource returns HTTP 400 Bad Request.
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
- Existing tests updated: pass `WorkContext.parse("slot-3")` instead of `"3"`, URL path update
- New: `coordinatedEndAsync(new RepoContext("engine"), workspaceRoot)` — finds terminal by repo name, executes steps
- New: `coordinatedEndAsync(new RepoContext("engine"), ...)` with no matching terminal — agent step completes immediately, script steps execute

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
