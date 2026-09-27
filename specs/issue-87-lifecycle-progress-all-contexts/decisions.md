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
