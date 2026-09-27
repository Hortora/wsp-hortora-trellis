## D1: Identity model for lifecycle operations

**Choice:** Sealed `WorkContext` interface with typed records for each context type, with string `contextId` as the wire/storage representation

```java
sealed interface WorkContext {
    record SlotContext(String slotId) implements WorkContext {}
    record RepoContext(String repoName) implements WorkContext {}

    String key(); // "slot-{N}" or "repo-{name}" — used in REST paths, SSE payloads, JSON

    static WorkContext parse(String contextId) { /* prefix switch, one boundary */ }
}
```

**Alternatives:**
- Stringly-typed `contextId` with prefix convention — pushes parsing into every consumer, no compile-time exhaustiveness when context types are added
- Parallel standalone endpoints — keep slot infrastructure, add `/repo/{repoName}` endpoints with separate orchestrator. Duplicates error handling, two orchestration paths.
- Lift orchestration into LifecycleManager — merge coordinator logic into manager. Violates D6 from #68 (separation of concerns).
**Rationale:** `slotId` was always a proxy for "which work context". A sealed interface gives compile-time exhaustiveness (compiler forces handling of new context types), eliminates string parsing at internal consumers, and makes the agent lookup strategy a natural pattern match (subsumes D4). The string `contextId` (`key()`) remains as the wire format — REST paths, SSE payloads, and persisted JSON use the string representation. Parsing happens once at the system boundary via `WorkContext.parse()`.

Two-layer identity: `WorkContext` is the coordination/tracking identity (which terminals to manage, which operation to track). `workspaceRoot` is the execution identity (which directory to run scripts in). LifecycleManager only uses `workspaceRoot` — it has no knowledge of `WorkContext`. The coordinator bridges these layers.
**Trade-offs:** One new sealed interface + two record types. Touches every layer (record fields, tracker maps, REST paths, SSE payloads, frontend filtering), but each change is mechanical — type at boundaries, typed everywhere else.
**Depends on:** None
**Sources:** `OperationProgress.java:9` (current slotId field), `LifecycleOperationTracker.java:34` (slotToOperation map), `SlotAgentCoordinator.java:88-118` (async orchestration keyed on slotId), `LifecycleResource.java:42-51` (REST path with slotId)
**Exploration:** quick → revised in review
**Status:** revised — sealed interface replaces stringly-typed contextId per review finding R1-02. D4 (agent lookup) subsumed.

## D2: Coordinator class naming

**Choice:** Rename `SlotAgentCoordinator` to `LifecycleCoordinator`
**Alternatives:**
- Keep `SlotAgentCoordinator` — less churn but name becomes actively misleading when it handles standalone repos
- `LifecycleAgentCoordinator` — captures agent aspect but verbose and emphasises an implementation detail over the class's primary identity
- `WorkContextCoordinator` — ties naming to the identity model; too abstract for a class that specifically coordinates lifecycle operations
**Rationale:** The class coordinates lifecycle operations for any work context (slot or standalone repo). The coordinator/manager distinction is conventional: LifecycleManager executes scripts, LifecycleCoordinator orchestrates the full flow (agent management + script delegation + progress tracking). The agent coordination role is an implementation detail of the orchestration, not the class's primary identity.
**Trade-offs:** Touches imports and references across production and test code, but the refactor is mechanical and semantically correct
**Depends on:** None (need to rename is independent of identity model choice — the class name becomes wrong the moment it handles standalone repos, regardless of how contexts are identified)
**Sources:** `SlotAgentCoordinator.java:21` (current class), `LifecycleResource.java:26` (injection site), `SlotAgentCoordinatorTest.java:21` (test class)
**Exploration:** quick
**Status:** revised — removed hard dependency on D1, added alternatives per review findings R1-07, R1-08

## D3: REST endpoint structure

**Choice:** Single endpoint with `contextId` path param — `/api/lifecycle/{end|pause|resume}/{contextId}`
**Alternatives:**
- Separate endpoint trees (`/api/lifecycle/slot/{slotId}/end`, `/api/lifecycle/repo/{repoName}/end`) — more REST-conventional at resource level but doubles method count (3 operations × 2 contexts = 6 methods vs 3). Adding a third context type requires 3 new methods vs adding a case to `WorkContext.parse()`.
- Separate REST resources with shared coordinator helper — preserves backward compatibility and appears type-safe at boundary, but the single endpoint with `WorkContext.parse()` achieves the same type safety without duplicating methods
**Rationale:** The `contextId` is the string representation of the `WorkContext` — the endpoint receives it, parses it once via `WorkContext.parse()` into a typed `WorkContext`, and passes it to the coordinator. One code path, one entry point shape, type-safe after the boundary parse. Extensible: new context types are handled by adding a case to `WorkContext.parse()`, not new endpoint methods.
**Trade-offs:** Backward-incompatible for slot-detail frontend — must update calls from `slotNumber` to `"slot-${slotNumber}"`. But it's code we control entirely.
**Depends on:** D1 (WorkContext identity model)
**Sources:** `LifecycleResource.java:42-79` (current `/{slotId}` endpoints), `slot-detail.ts:696-702` (current frontend calls)
**Exploration:** quick
**Status:** revised — added separate-resources alternative and extensibility rationale per review finding R1-10

## D4: Agent lookup strategy in the coordinator

**Choice:** Pattern match on `WorkContext` sealed interface in `findTerminals` helper method

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

**Subsumed by D1:** With the sealed `WorkContext` interface, agent lookup is a natural pattern match — not an independent design choice. Compile-time exhaustiveness guarantees all context types are handled. No string parsing, no strategy pattern. The `RepoContext` branch guards `t.slot() == null` to prevent matching terminals owned by a slot. Null-safe: `r.repoName().equals(t.repo())` — the non-null `repoName` drives the equals call.
**Depends on:** D1 (sealed WorkContext interface)
**Sources:** `SlotAgentCoordinator.java:205-252` (current slot-specific agent lookup methods), `TerminalInfo.java:3-10` (has both `slot` and `repo` fields)
**Exploration:** quick → subsumed in review
**Status:** revised — subsumed by D1 sealed interface, pattern match replaces string switch

## D5: Frontend lifecycle UI sharing

**Choice:** Extract a shared `<lifecycle-progress>` Lit component as a pure rendering component — receives `operation` as a property from the parent, dispatches `lifecycle-complete` event upward. Parent owns the SSE subscription and passes operation state down.
**Alternatives:**
- Component-owned SSE subscription — creates duplicate EventSource connections (parent already subscribes to agent:state + lifecycle:progress). Wastes connections and creates coordination problems.
- Copy lifecycle code into repo-detail — fast but two copies to maintain, diverge over time
- Lit mixin — awkward with `@state()` decorators and template composition, more ceremony than a component
**Rationale:** Follows Lit's unidirectional data flow model. Parent keeps a single EventSource for all topics, passes operation state as a property. Component is pure rendering — no SSE, no fetch, just template + styles. Eliminates duplication without adding connections.
**Trade-offs:** slot-detail.ts loses ~80 lines of inline lifecycle rendering which moves to a new file. Parent retains SSE handling and operation state management. repo-detail.ts needs substantial host-side integration: SSE subscription for `lifecycle:progress` topic, lifecycle buttons, `_operation` state, `_handleLifecycleEvent()`, and `_lifecycleAction()`.
**Depends on:** D1 (contextId used for filtering), D3 (single endpoint shape)
**Sources:** `slot-detail.ts:460-493` (`_renderLifecycle`), `slot-detail.ts:261-310` (SSE subscription + event handling), `slot-detail.ts:69-84` (TS interfaces)
**Exploration:** quick
**Status:** revised — SSE ownership moved to parent per decision review finding. Trade-offs updated to acknowledge repo-detail integration scope per R1-17.

## D6: Standalone repo async orchestration architecture

**Choice:** Extend the existing coordinator (LifecycleCoordinator) to accept all context types via `WorkContext`
**Alternatives:**
- Create a parallel coordinator for repos — violates the unification goal of #87, duplicates orchestration logic (agent management, progress tracking, locking, error handling)
- Make LifecycleManager itself async-capable for all contexts — violates D6 from #68 (separation of concerns between script execution and progress tracking). LifecycleManager operates on `workspaceRoot` only — adding context-awareness tangles execution with coordination.
**Rationale:** The coordinator already has the correct structure: agent coordination → LifecycleManager delegation → progress tracking via LifecycleOperationTracker. The async methods (`coordinatedEndAsync`, `coordinatedPauseAsync`, `coordinatedResumeAsync`) accept `WorkContext` and `Path workspaceRoot`. `WorkContext` drives terminal lookup (via `findTerminals`), `workspaceRoot` drives script execution. No structural change to the coordinator — only parameter widening. All existing callers (LifecycleResource, LifecycleActionExecutor) update to pass `WorkContext` or `contextId` instead of `slotId`.
**Trade-offs:** The coordinator grows in responsibility scope (from slot-only to all contexts), but the actual logic change is minimal — the `findTerminals` pattern match handles context-type-specific behavior.
**Depends on:** D1 (WorkContext identity model)
**Sources:** `SlotAgentCoordinator.java:88-180` (existing async methods), issue #87 ("async orchestration path should be workspace/repo-scoped, not slot-scoped")
**Exploration:** surfaced by review
**Status:** captured

## D7: LLM coordinator lifecycle path

**Choice:** Keep synchronous coordinator path for `LifecycleActionExecutor` — no step progress for LLM-triggered lifecycle operations
**Alternatives:**
- Switch executor to async path — fire-and-return-operationId, LLM coordinator polls or receives stream of step events. Changes the action model from blocking to async.
- Remove synchronous methods entirely — all callers use async. Simplifies coordinator API but forces executor into an async-then-block pattern.
**Rationale:** The LLM coordinator's action model is synchronous: propose action, execute, get result. It has no streaming progress UI — step visibility adds no value in this context. The synchronous path blocks until completion and returns a pass/fail result, which is what the executor needs. The async path exists for frontend callers that need real-time step progress via SSE. Both paths use the same `WorkContext` identity model. The executor's action params change from `slotId` to `contextId`.
**Trade-offs:** Two code paths in the coordinator (sync + async). If the LLM coordinator later needs step visibility (e.g., streaming tool output), the executor can migrate to the async path at that point.
**Depends on:** D1 (WorkContext identity model), D6 (coordinator handles all context types)
**Sources:** `LifecycleActionExecutor.java:56-64` (dispatch calls synchronous coordinator methods)
**Exploration:** surfaced by review
**Status:** captured
