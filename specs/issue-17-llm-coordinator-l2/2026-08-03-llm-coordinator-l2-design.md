# LLM Coordinator L2 — Propose Actions

**Issue:** Hortora/trellis#17
**Date:** 2026-08-03
**Status:** Approved

## Problem

The L1 coordinator (issue #13) observes workspace activity and provides
read-only advice — insights, warnings, suggestions, status updates. Advice
cards can only be dismissed. The `actionKey` field on `CoordinatorAdvice`
exists but is unused.

L2 closes the loop: the coordinator proposes executable actions with
approve/reject UI, and approved actions are routed to the lifecycle manager,
agent process manager, or recorded as advisory acknowledgements.

## Design Decisions

- **Action scope:** Lifecycle operations, agent management, and advisory
  suggestions. The action model is extensible to new categories.
- **Execution model:** Severity-based. LOW-risk actions execute immediately
  on approve. HIGH-risk actions (branch end, slot merge, agent stop) show a
  confirmation step before executing.
- **UI placement:** Actions live in the existing advice feed. Advice cards
  with an `actionKey` gain approve/reject buttons. No separate panel section.
- **Proposal paths:** Both proactive (event-driven) and interactive
  (conversation) paths can propose actions.
- **Expiry:** Actions auto-expire when system state changes invalidate their
  preconditions (e.g. user manually merges the slot the coordinator proposed
  merging).
- **Audit trail:** Full — separate `coordinator_actions` table tracks every
  state transition.
- **Event accumulation:** Uses the existing blocks `EventAccumulator` with
  new `CoordinatorEvent` variants for action lifecycle and lifecycle
  operation observations.

## §1 Domain Model

**Package:** `io.hortora.trellis.coordinator`

```java
enum ActionCategory { LIFECYCLE, AGENT, ADVISORY }

enum ActionStatus {
    PROPOSED, APPROVED, CONFIRMING, EXECUTING,
    COMPLETED, FAILED, REJECTED, EXPIRED
}

enum RiskLevel { LOW, HIGH }

record ProposedAction(
    String id,
    ActionCategory category,
    String actionType,
    Map<String, String> params,
    RiskLevel risk,
    String rationale,
    ActionStatus status,
    String adviceId,
    Instant proposedAt,
    Instant resolvedAt,
    String executionResult
)
```

### Risk Classification

Static map, not LLM-determined:

| Risk | Action types |
|------|-------------|
| HIGH | `lifecycle.start`, `lifecycle.end`, `slot.merge`, `agent.stop` |
| LOW | Everything else |

### Relation to CoordinatorAdvice

When the LLM generates advice with an action payload, the coordinator
creates both a `CoordinatorAdvice` (what the user sees) and a
`ProposedAction` (what can be executed). The advice's `actionKey` equals
the action's `id`. Advice without an action payload continues to work
as L1 — dismiss only.

## §2 Event Pipeline Extension

New variants on the existing `CoordinatorEvent` sealed interface. These
feed into the same `EventAccumulator<CoordinatorEvent>` (blocks) so the
LLM has richer context when deciding whether to propose new actions.

```java
sealed interface CoordinatorEvent {
    // existing
    record WorkspaceChangedEvent(...)  implements CoordinatorEvent {}
    record AnalysisEvent(...)          implements CoordinatorEvent {}
    record IssueEvent(...)             implements CoordinatorEvent {}

    // new — L2
    record ActionStateChangedEvent(
        Instant timestamp, String key,
        String actionId, ActionStatus oldStatus, ActionStatus newStatus,
        String actionType
    ) implements CoordinatorEvent {}

    record LifecycleOperationEvent(
        Instant timestamp, String key,
        String operation,
        boolean success,
        String detail
    ) implements CoordinatorEvent {}
}
```

### Event Flow

1. **ActionStateChangedEvent** fires whenever a `ProposedAction`
   transitions state. The LLM sees outcomes and can propose follow-ups.
2. **LifecycleOperationEvent** fires when lifecycle operations complete
   — whether triggered by an approved action OR by the user directly via
   the existing REST API. Gives the coordinator visibility into manual
   operations for auto-expiry.
3. `CoordinatorEventObserver` gains new observer methods for these events.
4. `SignificanceFilter` is extended — action state changes and lifecycle
   operations are always significant.

### Auto-Expiry Mechanism

When a `LifecycleOperationEvent` arrives, `ActionService.expireStale()`
checks all PROPOSED actions whose `actionType` and params overlap with
the completed operation. For example, a successful `slotMerge` for slot X
expires any pending "merge slot X" action. This is a precondition check,
not an LLM call.

## §3 Action Executor Framework

```java
interface ActionExecutor {
    ActionCategory category();
    Set<String> supportedTypes();
    ActionResult execute(ProposedAction action);
}

record ActionResult(boolean success, String detail) {}
```

### LifecycleActionExecutor

Wraps `LifecycleManager`. Maps `actionType` to manager calls:

| actionType | LifecycleManager method | Risk |
|---|---|---|
| `lifecycle.start` | `start(workspace, branch, issue)` | HIGH |
| `lifecycle.end` | `end(slotId, workspace)` | HIGH |
| `lifecycle.pause` | `pause(slotId, workspace)` | LOW |
| `lifecycle.resume` | `resume(slotId, workspace)` | LOW |
| `slot.create` | `slotCreate(workspace, args)` | LOW |
| `slot.merge` | `slotMerge(slotId, workspace)` | HIGH |
| `epic.setup` | `epicSetup(workspace, args)` | LOW |
| `epic.next` | `epicNext(epicPath)` | LOW |

Params are drawn from `ProposedAction.params()`. Validated against the
expected param set before execution.

### AgentActionExecutor

Designed against the `AgentProcessManager` interface from the issue #20
spec. Stubbed until that implementation lands — returns "not yet available"
`ActionResult` for now.

| actionType | Future method | Risk |
|---|---|---|
| `agent.start` | `startAgent(terminal, opts)` | LOW |
| `agent.stop` | `stopAgent(terminal)` | HIGH |
| `agent.pause` | `pauseAgent(terminal)` | LOW |
| `agent.resume` | `resumeAgent(terminal)` | LOW |
| `agent.refresh` | `refreshAgent(terminal)` | LOW |

### AdvisoryActionExecutor

Non-executable suggestions. "Executing" an advisory action marks it as
acknowledged and records the acknowledgement. No side effects.

| actionType | Behaviour | Risk |
|---|---|---|
| `advisory.prioritise` | Record acknowledgement | LOW |
| `advisory.investigate` | Record acknowledgement | LOW |

### Dispatch Flow

1. User approves action → `ActionService.approve(actionId)`
2. If HIGH risk → status moves to CONFIRMING, frontend shows confirmation
3. If LOW risk (or confirmed) → `ActionService.execute(actionId)`
4. Service looks up the right `ActionExecutor` by category
5. Validates preconditions (action still APPROVED/CONFIRMING, target exists)
6. Calls `executor.execute(action)` → gets `ActionResult`
7. Updates status to COMPLETED or FAILED, fires `ActionStateChangedEvent`

## §4 Persistence & ActionService

### Schema

```sql
CREATE TABLE coordinator_actions (
    id               TEXT PRIMARY KEY,
    advice_id        TEXT NOT NULL,
    category         TEXT NOT NULL,
    action_type      TEXT NOT NULL,
    params           TEXT NOT NULL,
    risk             TEXT NOT NULL,
    rationale        TEXT NOT NULL,
    status           TEXT NOT NULL,
    workspace        TEXT NOT NULL,
    proposed_at      TEXT NOT NULL,
    resolved_at      TEXT,
    execution_result TEXT,
    FOREIGN KEY (advice_id) REFERENCES coordinator_advice(id)
);
```

Same SQLite database as `coordinator_advice`. Schema creation extends
`CoordinatorSchemaManager`.

### ActionService

`@ApplicationScoped`. Owns the full action lifecycle. Distinct from
`CoordinatorService` which owns advice generation and LLM interaction.

```java
@ApplicationScoped
public class ActionService {
    @Inject DataSource dataSource;
    @Inject EventBroadcaster broadcaster;
    @Inject Event<CoordinatorEvent> eventBus;
    @Inject Instance<ActionExecutor> executors;

    ProposedAction propose(String adviceId, ActionCategory category,
                           String actionType, Map<String,String> params,
                           String rationale, String workspace);
    ProposedAction approve(String actionId);
    ProposedAction confirm(String actionId);
    ProposedAction reject(String actionId);
    void expireStale(String actionType, Map<String,String> params);

    List<ProposedAction> pendingActions(String workspace);
    List<ProposedAction> actionHistory(String workspace, int limit);
    ProposedAction getAction(String actionId);
}
```

### SSE Topic

`coordinator:action` — emits `ProposedAction` on every state change.
Frontend subscribes alongside existing topics.

## §5 LLM Prompt Changes

### System Prompt Extension

The system prompt (in `CoordinatorPrompts`) is extended to describe
available action types, their parameters, and when to propose them:

```
You can propose executable actions. When you identify something actionable,
include an "action" object in your response alongside the advice:

{"type": "SUGGESTION", "title": "...", "body": "...", "actionKey": "<id>",
 "action": {
   "category": "LIFECYCLE|AGENT|ADVISORY",
   "actionType": "<type>",
   "params": { ... },
   "rationale": "<why this action>"
 }}
```

The full action type catalogue with expected params is included in the
system prompt so the LLM generates valid payloads.

### Context Assembler Additions

`CoordinatorContextAssembler` gains:
- `appendPendingActions(sb, workspace)` — shows the LLM what actions are
  already proposed, preventing duplicates.
- `appendActionHistory(sb, workspace)` — recent action outcomes (last 10),
  so the LLM can learn from completed/failed actions.

### Parsing

`CoordinatorService.parseAdviceResponse()` is extended to parse the nested
`action` object. When present, `CoordinatorService` calls
`actionService.propose()` after persisting the advice.

Both paths (proactive and interactive) parse action proposals from LLM
responses.

## §6 Frontend Changes

### Advice Card Extensions

Cards with a non-null `actionKey` render action controls. The component
tracks action state via a `Map<string, ProposedAction>`, updated by SSE.

| Action status | Card rendering |
|---|---|
| PROPOSED | Approve + Reject buttons alongside Dismiss |
| CONFIRMING | Card expands: description, consequences, Confirm + Cancel |
| EXECUTING | Spinner with "Executing..." |
| COMPLETED | Green checkmark + result summary |
| FAILED | Red badge + error message |
| REJECTED | Greyed out, auto-dismissed |
| EXPIRED | Removed from feed |

### SSE Subscription

Add `coordinator:action` to the existing push connection at
`/api/push?topics=coordinator:advice,coordinator:message,coordinator:action`.

### Data Flow

1. Advice arrives via `coordinator:advice` SSE with an `actionKey`
2. Component fetches `GET /api/coordinator/actions/{actionKey}`
3. State changes arrive via `coordinator:action` SSE, matched by `adviceId`
4. Button clicks → `POST /api/coordinator/actions/{id}/approve|reject|confirm`

### Confirmation for HIGH-risk

Not a separate modal. The card itself transforms — expands to show
consequences ("This will merge slot X into main and delete the worktree"),
with Confirm and Cancel buttons.

### Components Changed

- `coordinator-panel.ts` — extended advice card template, new `_approve`,
  `_reject`, `_confirm` methods, action state map
- No new components

## §7 REST API

New endpoints on `CoordinatorResource`:

```
GET    /api/coordinator/actions?workspace={ws}         → List<ProposedAction>
GET    /api/coordinator/actions/history?workspace={ws}  → List<ProposedAction>
GET    /api/coordinator/actions/{id}                    → ProposedAction
POST   /api/coordinator/actions/{id}/approve            → ProposedAction
POST   /api/coordinator/actions/{id}/reject             → ProposedAction
POST   /api/coordinator/actions/{id}/confirm            → ProposedAction
```

### Error Responses

| Condition | HTTP Status |
|---|---|
| Action not found | 404 |
| Invalid state transition | 409 Conflict |
| Execution failure | 200 with FAILED status in body |
| Agent executor not available | 200 with FAILED status |

## §8 Testing Strategy

### Unit Tests (no tmux, no LLM)

- **ActionServiceTest** — state machine: propose→approve→execute→complete,
  propose→reject, propose→expire. HIGH-risk CONFIRMING gate. Invalid
  transitions rejected.
- **LifecycleActionExecutorTest** — mock `LifecycleManager`, verify method
  dispatch per `actionType`, param extraction.
- **AgentActionExecutorTest** — all types return "not yet available".
- **AdvisoryActionExecutorTest** — acknowledge-only behaviour.
- **ActionExpiryTest** — `LifecycleOperationEvent` expires matching
  PROPOSED actions.
- **SignificanceFilterTest** — action and lifecycle events pass filter.

### Integration Tests (`@QuarkusTest`, mock LLM)

- **CoordinatorResourceTest** — extended: action endpoints round-trip,
  404/409 error cases.
- **ActionProposalIntegrationTest** — mock LLM returns advice with action
  payload, verify both `CoordinatorAdvice` and `ProposedAction` created,
  linked, broadcast via SSE.

### Frontend

- Manual browser verification of action buttons, CONFIRMING expansion,
  state transitions, SSE-driven updates.

## Scope Boundary

### In scope
- `ProposedAction` domain model and persistence
- `ActionService` lifecycle management
- `ActionExecutor` interface + lifecycle, agent (stubbed), advisory impls
- Event pipeline extension (two new `CoordinatorEvent` variants)
- Auto-expiry on state change
- LLM prompt extension for action proposals
- Frontend approve/reject/confirm on advice cards
- REST API for action operations
- Full audit trail in `coordinator_actions`

### Out of scope
- Agent management implementation (issue #20 — stubbed here)
- Autonomous execution without approval (L3, issue #18)
- Batch action proposals ("do all of these")
- Action undo/rollback
- Notification/alerting on action outcomes
