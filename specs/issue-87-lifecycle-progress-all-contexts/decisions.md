## D1: Identity model for lifecycle operations

**Choice:** Generalise `slotId` to `contextId` — a unified key covering both slot and standalone repo work contexts
**Alternatives:**
- Parallel standalone endpoints — keep slot infrastructure, add `/repo/{repoName}` endpoints with separate orchestrator. Duplicates error handling, two orchestration paths.
- Lift orchestration into LifecycleManager — merge coordinator logic into manager. Violates D6 from #68 (separation of concerns).
**Rationale:** `slotId` was always a proxy for "which work context". Making that explicit with a unified key means tracker, SSE, and frontend all use one filtering mechanism. For slots, `contextId = "slot-{N}"`. For standalone repos, `contextId = "repo-{repoName}"`. Agent lookup switches on context type internally.
**Trade-offs:** Touches every layer (record fields, tracker maps, REST paths, SSE payloads, frontend filtering), but each change is mechanical — rename and widen.
**Depends on:** None
**Sources:** `OperationProgress.java:9` (current slotId field), `LifecycleOperationTracker.java:34` (slotToOperation map), `SlotAgentCoordinator.java:88-118` (async orchestration keyed on slotId), `LifecycleResource.java:42-51` (REST path with slotId)
**Exploration:** quick
**Status:** captured

## D2: Coordinator class naming

**Choice:** Rename `SlotAgentCoordinator` to `LifecycleCoordinator`
**Alternatives:**
- Keep `SlotAgentCoordinator` — less churn but name becomes actively misleading when it handles standalone repos
**Rationale:** The class coordinates lifecycle operations for any work context (slot or standalone repo), not just slots. The name should reflect the actual responsibility. IntelliJ rename refactor updates all references.
**Trade-offs:** Touches imports and references across production and test code, but the refactor is mechanical and semantically correct
**Depends on:** D1 (contextId generalisation makes the slot-specific name wrong)
**Sources:** `SlotAgentCoordinator.java:21` (current class), `LifecycleResource.java:26` (injection site), `SlotAgentCoordinatorTest.java:21` (test class)
**Exploration:** quick
**Status:** captured

## D3: REST endpoint structure

**Choice:** Single endpoint with `contextId` path param — `/api/lifecycle/{end|pause|resume}/{contextId}`
**Alternatives:**
- Separate endpoint trees (`/api/lifecycle/slot/{slotId}/end`, `/api/lifecycle/repo/{repoName}/end`) — more REST-conventional but doubles endpoint count and requires two coordinator entry shapes
**Rationale:** The `contextId` is the key abstraction from D1 — the endpoint should use it directly. Prefix parsing (`slot-` vs `repo-`) is trivial and contained in the coordinator. One code path, one entry point shape.
**Trade-offs:** Backward-incompatible for slot-detail frontend — must update calls from `slotNumber` to `"slot-${slotNumber}"`. But it's code we control entirely.
**Depends on:** D1 (contextId identity model)
**Sources:** `LifecycleResource.java:42-79` (current `/{slotId}` endpoints), `slot-detail.ts:696-702` (current frontend calls)
**Exploration:** quick
**Status:** captured

## D4: Agent lookup strategy in the coordinator

**Choice:** Single `findTerminals(String contextId)` helper method that switches on prefix — `slot-*` filters by `t.slot()`, `repo-*` filters by `t.repo()` with `slot == null` guard
**Alternatives:**
- Strategy pattern with `ContextResolver` interface and per-type implementations — over-engineered for a two-variant switch on a string prefix
**Rationale:** The prefix switch is two branches. A helper method that returns a filtered terminal list keeps the coordinator simple. The three agent methods (`stopAll`, `shutdown`, `resume`) all call the same helper instead of each doing their own slot filter. The `repo-*` branch filters `t.repo().equals(repoName) && t.slot() == null` — a repo inside a slot is owned by the slot context, not addressable as a standalone repo. Null-safe: `repoName.equals(t.repo())` since `t.repo()` can be null.
**Trade-offs:** If more context types are added later, the switch grows — but that's a bridge to cross then, not now
**Depends on:** D1 (contextId identity model)
**Sources:** `SlotAgentCoordinator.java:205-252` (current slot-specific agent lookup methods), `TerminalInfo.java:3-10` (has both `slot` and `repo` fields)
**Exploration:** quick
**Status:** captured

## D5: Frontend lifecycle UI sharing

**Choice:** Extract a shared `<lifecycle-progress>` Lit component as a pure rendering component — receives `operation` as a property from the parent, dispatches `lifecycle-complete` event upward. Parent owns the SSE subscription and passes operation state down.
**Alternatives:**
- Component-owned SSE subscription — creates duplicate EventSource connections (parent already subscribes to agent:state + lifecycle:progress). Wastes connections and creates coordination problems.
- Copy lifecycle code into repo-detail — fast but two copies to maintain, diverge over time
- Lit mixin — awkward with `@state()` decorators and template composition, more ceremony than a component
**Rationale:** Follows Lit's unidirectional data flow model. Parent keeps a single EventSource for all topics, passes operation state as a property. Component is pure rendering — no SSE, no fetch, just template + styles. Eliminates duplication without adding connections.
**Trade-offs:** slot-detail.ts loses ~80 lines of inline lifecycle rendering which moves to a new file. Parent retains SSE handling and operation state management. repo-detail.ts needs to add SSE subscription for `lifecycle:progress` topic.
**Depends on:** D1 (contextId used for filtering), D3 (single endpoint shape)
**Sources:** `slot-detail.ts:460-493` (`_renderLifecycle`), `slot-detail.ts:261-310` (SSE subscription + event handling), `slot-detail.ts:69-84` (TS interfaces)
**Exploration:** quick
**Status:** revised — SSE ownership moved to parent per decision review finding
