## D1: Transport mechanism for lifecycle step progress

**Choice:** SSE via existing TopicRegistry/EventBroadcaster on the `/api/push` endpoint
**Alternatives:**
- Dedicated SSE endpoint (like BootstrapperResource) — more isolation but adds a new endpoint and connection
- WebSocket channel — bidirectional capability unused since lifecycle progress is server→client only
**Rationale:** Fits the established broadcast pattern used by all panels (workspace:*, agent:state, control:*). Multiple UI components can subscribe. No new endpoint needed.
**Trade-offs:** Topic-based broadcast means all subscribers receive all lifecycle events; frontend filters by operationId
**Sources:** `PushResource.java:17`, `SseSessionSender.java`, `FileWatcherService.java:112` (broadcaster.broadcast pattern), `workspace-sse.ts`
**Exploration:** quick
**Status:** captured

## D2: Execution model for lifecycle operations

**Choice:** Fully async — POST returns immediately with an operationId, steps run on a background thread, progress streams via SSE
**Alternatives:**
- Blocking POST + SSE side-channel — simpler backend but risks HTTP timeout on long operations (rebase + push can take 30s+)
**Rationale:** Decouples HTTP request lifecycle from operation execution. No timeout risk. Frontend gets immediate feedback that the operation was accepted.
**Trade-offs:** More complex backend — need operation tracking, background executor, error propagation via SSE instead of HTTP response
**Sources:** `LifecycleManager.java:50-66` (current synchronous end() with 3 sequential scripts), `ScriptRunner.java:57` (120s timeout per script)
**Exploration:** quick
**Status:** captured

## D3: Output detail level in UI

**Choice:** Summary + expandable — each step shows one-line status, expand button reveals full stdout/stderr
**Alternatives:**
- Inline streaming — mini-terminal per step; noisy for the sidebar's limited space
- Summary only — just status badges with no output; insufficient for debugging failures
**Rationale:** Keeps the modal clean during normal operations while allowing debugging when a step fails. Matches the progressive disclosure pattern used elsewhere in trellis (e.g., agent status badges with expandable error detail).
**Trade-offs:** Captured output must be stored per-step on the server, increasing memory footprint per operation
**Sources:** `slot-detail.ts:348-353` (agent-status-badge pattern with expandable lastError)
**Exploration:** quick
**Status:** captured

## D4: UI placement of lifecycle progress

**Choice:** New sidebar section between Status and Issues, collapsed when no operation is running
**Alternatives:**
- Main content overlay — more room but takes over the terminal/content view
- Toast/notification bar — minimal footprint but less discoverable and no expandable output
**Rationale:** Consistent with the sidebar's role as the slot state dashboard. Status → Lifecycle → Issues → Plan → Repos → Terminals flows naturally. Collapsed-when-idle avoids visual noise.
**Trade-offs:** Sidebar space is limited — long step lists or verbose output need scrollable expand areas
**Sources:** `slot-detail.ts:296-362` (_renderSidebar structure)
**Exploration:** quick
**Status:** captured

## D5: State persistence across page refresh

**Choice:** Server-side state — LifecycleOperationTracker holds active operations in memory, GET endpoint returns current state on page load
**Alternatives:**
- Ephemeral only — progress lives only in SSE stream, page refresh loses step history until next event
**Rationale:** Users expect to see operation state after refreshing the page. A GET endpoint for current state + SSE for live updates is the standard pattern (mirrors how agent:state works).
**Trade-offs:** In-memory state — lost on sidecar restart. Acceptable because lifecycle operations are short-lived (seconds to minutes) and a restart would kill the operation anyway.
**Sources:** `AgentProcessManager` (precedent for in-memory state with SSE updates + GET snapshots)
**Exploration:** quick
**Status:** revised — see D7

## D6: Backend architecture — dedicated OperationTracker

**Choice:** Separate `LifecycleOperationTracker` CDI bean that owns operation progress state and SSE publishing
**Alternatives:**
- Inline tracking in LifecycleManager — fewer classes but tangles execution orchestration with progress tracking
**Rationale:** Separates concerns: LifecycleManager orchestrates script execution, OperationTracker owns progress state and event publishing. Tracker is independently testable and reusable for future async operations.
**Trade-offs:** One more class. LifecycleManager must call tracker at step boundaries, creating a coupling.
**Depends on:** D2 (fully async execution model)
**Sources:** `LifecycleManager.java` (current structure), `AgentProcessManager` (precedent for state-tracking CDI bean)
**Exploration:** quick
**Status:** captured

## D7: Operation state durability

**Choice:** File-backed log — write a JSON operation file to `.trellis/operations/{id}.json` at each step transition
**Alternatives:**
- Derive from filesystem — infer state from git branch positions and .plan lifecycle state; cheaper but can't determine which step failed or why
- Accept ephemeral — rely on operations being short-lived; simplest but weakest consistency
**Rationale:** Crash recovery requires knowing which step was running and what its output was. The filesystem (git, .plan) tells you the current state but not the operation context. A lightweight file log bridges this gap with minimal overhead.
**Trade-offs:** One file write per step transition (~7 writes for a 4-step operation). Negligible I/O cost. Files must be cleaned up after completion (same 5-minute eviction applies to disk).
**Depends on:** D5 (server-side state), D6 (dedicated OperationTracker)
**Sources:** `.trellis/layouts/` (precedent for file-backed state under .trellis directory)
**Exploration:** quick
**Status:** captured
