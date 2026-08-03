# LLM Coordinator L3 — Autonomous Execution with Observation Mode

**Issue:** Hortora/trellis#18
**Date:** 2026-08-02
**Status:** Approved
**Review:** Light — coherence, structure, robustness (30 findings addressed)

## Problem

The L2 coordinator (issue #17) proposes executable actions with
approve/reject UI. Every action requires explicit human approval before
execution. This gates delivery velocity on human response time — the
coordinator knows what to do but must wait for permission.

L3 closes the gap: the coordinator can execute actions autonomously,
with configurable human oversight ranging from full manual control to
autonomous execution with rate-limited safety.

## Design Decisions

- **Three autonomy levels:** MANUAL (L2 behaviour), OBSERVATION
  (countdown with veto window), AUTONOMOUS (immediate execution for
  low-risk, observation countdown for high-risk).
- **Hybrid risk/override policy:** Risk classification sets the default
  (LOW = autonomous, HIGH = gated). Per-action-type overrides promote
  or demote individual actions. Most users need zero configuration.
- **Session-level toggle:** Preference sets the default autonomy level
  per workspace. The coordinator panel has a live toggle that overrides
  it for the session. Resets to preference default on sidecar restart.
- **Observation countdown:** Server-side timer. Action enters PROPOSED,
  countdown runs, auto-executes unless the user vetoes. Veto = REJECTED
  (existing terminal state, no new states).
- **No new states:** The L2 state machine is unchanged. L3 changes
  *who* triggers transitions, not the transitions themselves.
- **Notifications:** Autonomous action completions fire platform
  notifications via the existing notifications API + blocks-ui component.

## §1 Autonomy Model

**Package:** `io.hortora.trellis.coordinator`

```java
enum AutonomyLevel { MANUAL, OBSERVATION, AUTONOMOUS }

enum AutonomyOverride { AUTONOMOUS, GATED }
```

### Effective Policy Resolution

For each proposed action, the system resolves whether it should be
autonomous or gated:

```
1. Check per-action-type override map → if present, use override
2. Fall back to risk classification → LOW = AUTONOMOUS, HIGH = GATED
```

### Behaviour Matrix

| Autonomy Level | Effective Policy | Behaviour |
|----------------|-----------------|-----------|
| MANUAL | any | Leave in PROPOSED (L2) |
| OBSERVATION | any | Schedule countdown |
| AUTONOMOUS | AUTONOMOUS | Execute immediately |
| AUTONOMOUS | GATED | Schedule countdown |

In AUTONOMOUS level, HIGH-risk actions that aren't overridden always
use the observation countdown — they never skip the countdown entirely.

## §2 Preferences & Session State

### Preferences

**Package:** `io.hortora.trellis.config`

New infrastructure: `~/.trellis/preferences.json` — a JSON file read
by a new `PreferencesService` (`@ApplicationScoped`). This is separate
from Quarkus `@ConfigMapping` (`CoordinatorConfig`), which provides
compile-time defaults from `application.properties`. Preferences are
user-editable at runtime; config properties are not.

`PreferencesService` is in the `config` package, not `coordinator` —
it is a general-purpose trellis service, not coordinator-specific.

```json
{
  "coordinator": {
    "autonomyLevel": {
      "/Users/dev/workspace-a": "OBSERVATION",
      "/Users/dev/workspace-b": "AUTONOMOUS"
    },
    "observationCountdownSeconds": 30,
    "autonomyOverrides": {
      "slot.create": "GATED",
      "lifecycle.end": "GATED"
    },
    "rateLimitMaxActions": 5,
    "rateLimitWindowSeconds": 60
  }
}
```

`autonomyLevel` is per-workspace — a map keyed by workspace root path.
Workspaces not in the map default to MANUAL. All other settings are
global.

**Error handling:** `PreferencesService` reads the file on startup and
caches the parsed result. On parse failure (malformed JSON, missing
file), it logs a warning and uses hardcoded defaults (MANUAL, 30s
countdown, no overrides, 5/60s rate limit). Re-reads on demand when
the REST API is called — a file watcher is not needed for a
manually-edited file.

### CoordinatorConfig (Unchanged)

Existing `CoordinatorConfig` properties (`enabled`, `windowTimeMs`,
`windowCount`, etc.) remain in `application.properties`. The new
autonomy settings live in preferences, not config — they are
user-preference concerns, not deployment concerns.

### AutonomyResolver

**Package:** `io.hortora.trellis.coordinator`

Encapsulates the full autonomy resolution logic — extracted from
`ActionService` to avoid making it a god class and to break the
circular dependency between `CoordinatorService` (which holds session
state) and `ActionService` (which needs to read it).

```java
@ApplicationScoped
public class AutonomyResolver {
    @Inject PreferencesService preferences;

    private volatile AutonomyLevel sessionOverride; // null = use preference

    public AutonomyLevel resolveLevel(String workspace) {
        if (sessionOverride != null) return sessionOverride;
        return preferences.autonomyLevel(workspace);
    }

    public AutonomyOverride resolvePolicy(String actionType) {
        var override = preferences.autonomyOverride(actionType);
        if (override != null) return override;
        return RiskClassification.riskFor(actionType) == RiskLevel.LOW
            ? AutonomyOverride.AUTONOMOUS : AutonomyOverride.GATED;
    }

    public void setSessionOverride(AutonomyLevel level) { ... }
    public void clearSessionOverride() { ... }
    public AutonomyLevel sessionOverride() { ... }
}
```

`ActionService` and `CoordinatorService` both inject
`AutonomyResolver`. Neither depends on the other for autonomy state.

### Session State

`AutonomyResolver` holds the volatile `sessionOverride` field. The
session override is global — one coordinator per sidecar, one autonomy
level. When multiple workspaces are active, the session override applies
to all of them (the per-workspace preference is the differentiation
mechanism, not the session toggle).

On sidecar restart, the session override resets to null (preference
default wins).

### REST API

```
GET  /api/coordinator/autonomy?workspace={ws}
     → { "level": "OBSERVATION", "source": "session"|"preference" }

POST /api/coordinator/autonomy?level=OBSERVATION
     → sets session override (global), returns updated state

POST /api/coordinator/autonomy/reset
     → clears session override, returns to preference default
```

## §3 Action Lifecycle Changes

The L2 state machine stays unchanged — no new enum values, no new
transitions. L3 changes *who* triggers transitions, not the transitions
themselves. (Note: overall system state does grow — session overrides,
countdown map, rate limiter, preferences — but `ActionStatus` is
untouched.)

### CountdownScheduler

**Package:** `io.hortora.trellis.coordinator`

Extracted from `ActionService` to keep responsibilities focused.
Owns the `ScheduledExecutorService` for countdown timers and the
in-memory countdown map.

```java
@ApplicationScoped
public class CountdownScheduler {
    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor(r -> {
            var t = new Thread(r, "countdown-scheduler");
            t.setDaemon(true);
            return t;
        });

    record PendingCountdown(String actionId, Instant deadline,
                            ScheduledFuture<?> future) {}
    private final Map<String, PendingCountdown> countdowns =
        new ConcurrentHashMap<>();

    void schedule(String actionId, int seconds,
                  Consumer<String> onFire) { ... }
    void cancel(String actionId) { ... }
    void cancelAll() { ... }
    boolean hasCountdown(String actionId) { ... }
    Instant deadline(String actionId) { ... }
}
```

The `onFire` callback is `ActionService::autoExecute`. The scheduler
wraps every timer callback in try-catch — uncaught exceptions must not
kill the `ScheduledExecutorService`.

This is separate from the existing single-thread `actionExecutor` in
`ActionService` (which runs blocking lifecycle operations). Timer
fires submit to the action executor; they do not block the scheduler
thread.

### ActionService.propose() — Autonomy-Aware Path

After creating the action in PROPOSED and broadcasting it, `propose()`
checks the effective autonomy via `AutonomyResolver`:

```
1. Resolve current autonomy level via autonomyResolver.resolveLevel(workspace)
2. If MANUAL → return (L2 behaviour, action sits in PROPOSED)
3. Resolve effective policy via autonomyResolver.resolvePolicy(actionType)
4. If AUTONOMOUS level + AUTONOMOUS policy → call autoExecute(actionId)
5. If OBSERVATION level, or AUTONOMOUS level + GATED policy →
   persist countdown_ends_at, schedule countdown via CountdownScheduler
```

The action always hits PROPOSED first and is always broadcast via SSE.
The UI sees every action, even ones that auto-execute — the feed is a
complete audit record.

### autoExecute() — Bypass the Risk Gate

`approve()` contains a risk gate: HIGH-risk actions go to CONFIRMING,
requiring a second confirmation step. This is correct for human-
initiated approval. But when the system decides to auto-execute (either
immediately or after a countdown), it has already evaluated the
autonomy policy — the risk gate would send GATED actions to CONFIRMING
with nobody to confirm.

```java
void autoExecute(String actionId) {
    // Optimistic lock: only transition if still PROPOSED
    int updated = updateStatusCas(actionId, PROPOSED, APPROVED);
    if (updated == 0) return; // already transitioned (veto, expiry, race)
    var action = getAction(actionId);
    executeAction(action); // submits to action executor thread
}
```

`autoExecute()` is an internal method — not exposed via REST. The user
approval path (`approve()`) is unchanged and still enforces the risk
gate for manual approvals.

### Optimistic Locking

All state transitions use compare-and-swap on the database:

```sql
UPDATE coordinator_actions SET status = ?, resolved_at = ?,
  execution_result = ? WHERE id = ? AND status = ?
```

The trailing `AND status = ?` is the CAS. If the row was already
transitioned (by a race between timer and manual approve, or between
timer and veto), the UPDATE affects 0 rows and the caller returns
without executing. This eliminates double-execution races.

### Countdown Persistence and Restart Recovery

`countdown_ends_at` is persisted in the `coordinator_actions` table
(new nullable column). This serves two purposes:

1. **Restart recovery:** On startup, `ActionService` sweeps PROPOSED
   actions with a non-null `countdown_ends_at`:
   - If the deadline has passed → call `autoExecute(actionId)`
   - If the deadline is in the future → reschedule with remaining time
2. **SSE reconnect recovery:** The REST endpoint
   `GET /api/coordinator/actions/{id}` returns `countdown_ends_at` so
   the frontend can recover the countdown timer after reconnection.

Schema addition:
```sql
ALTER TABLE coordinator_actions ADD COLUMN countdown_ends_at TEXT;
```

### Mode-Switch Countdown Cancellation

When the autonomy level changes to MANUAL (via the session toggle or
preference update), `ActionService` calls
`countdownScheduler.cancelAll()` and clears `countdown_ends_at` for
all PROPOSED actions with active countdowns. Actions remain in PROPOSED
— the user must now approve them manually.

When switching from MANUAL to OBSERVATION or AUTONOMOUS, existing
PROPOSED actions are NOT retroactively given countdowns. Only newly
proposed actions use the new level.

### approve() — Unchanged for Manual Path

`approve()` retains the risk gate for human-initiated approvals:
HIGH-risk → CONFIRMING, LOW-risk → EXECUTING. It also uses the CAS
update to prevent races with concurrent `autoExecute()`.

### Veto During Countdown

The existing `reject()` method works unchanged with CAS. After
rejection, the timer fires, the CAS fails (status is REJECTED, not
PROPOSED), and the timer is a no-op. The `countdowns` map entry is
cleaned up on fire.

### Auto-Expiry During Countdown

Same pattern — `expireStale()` transitions to EXPIRED via CAS, the
timer fires and the CAS fails.

### SSE Payload Extension

When a countdown starts, the SSE broadcast for `coordinator:action`
includes `countdownEndsAt` from the persisted column. The frontend
also recovers this value from the REST endpoint on SSE reconnect.

## §4 Circuit Breaker & Safety

### Existing Mitigation (Unchanged)

`SignificanceFilter` suppresses batches containing only action/lifecycle
events when the count exceeds the threshold per window. Terminal state
events (EXPIRED, REJECTED) are excluded from the accumulator entirely
— they are not added to the accumulator in the first place (filtered
in `CoordinatorEventObserver` before accumulation, not filtered after).

### Autonomous Execution Rate Limit

`ActionService` tracks autonomous executions using a sliding window
of timestamps:

```java
private final Deque<Instant> autonomousExecutionTimestamps =
    new ConcurrentLinkedDeque<>();
```

On each autonomous execution, the current timestamp is appended.
Before executing, timestamps older than the window duration are
pruned. If the remaining count exceeds the limit, the action falls
back to observation mode (countdown) instead of auto-executing.

Defaults: 5 actions per 60 seconds (configurable via preferences:
`coordinator.rateLimitMaxActions`, `coordinator.rateLimitWindowSeconds`).

The rate limit resets (clears all timestamps) when a human explicitly
approves an action — this is a signal that the coordinator is on track
and the rate limit was being overly cautious.

This is a backstop, not a throttle. Under normal operation it never
fires. It catches the degenerate case where the coordinator proposes,
executes, observes the result, and immediately proposes again in a
tight loop.

### Prompt Extension

`CoordinatorPrompts.systemPrompt()` becomes parameterized — it accepts
the current `AutonomyLevel` and includes mode-specific guidance:

- **MANUAL:** No additional prompt (L2 behaviour unchanged).
- **OBSERVATION:** "Actions will execute after a countdown unless the
  user vetoes. Be selective — only propose when confidence is high."
- **AUTONOMOUS:** "LOW-risk actions execute immediately. HIGH-risk
  actions execute after a countdown. Only propose actions when there
  is clear justification."

`CoordinatorService` passes the resolved autonomy level when
assembling the prompt. The existing `systemPrompt()` call sites
are updated to `systemPrompt(autonomyLevel)`.

## §5 Notifications

When an action auto-executes (observation countdown elapsed or immediate
autonomous execution), fire a notification via the existing platform
notifications API. The blocks-ui notification component handles rendering.

Notifications are wired from `ActionService` after an autonomous or
countdown-triggered execution completes (COMPLETED or FAILED). Manual
approvals do not fire notifications — the user already knows.

## §6 Frontend Changes

### Mode Toggle

Three-state toggle in the coordinator panel header: Manual | Observation
| Autonomous. Shows current level and source (preference vs session
override). Clicking a mode calls `POST /api/coordinator/autonomy?level=X`.
A reset link appears when the session overrides the preference.

### Countdown Timer on Advice Cards

Cards for actions in observation countdown show a circular countdown
timer (CSS animation driven by `countdownEndsAt` from the SSE payload).
Two buttons during countdown: **Approve Now** (skip the wait) and
**Veto** (reject).

When the countdown expires, the card transitions to EXECUTING via the
normal SSE state change update.

### Auto Badge

Actions that auto-executed (no countdown or countdown elapsed) show a
subtle "auto" badge on the COMPLETED card, distinguishing them from
manually-approved completions.

### No New Components

Mode toggle, countdown timer, and auto badge are template extensions
on `coordinator-panel.ts`. The existing state map tracks `ProposedAction`
per `actionKey`.

## §7 Testing Strategy

### Unit Tests (no tmux, no LLM)

- **AutonomyResolverTest** — effective policy resolution: risk-based
  defaults, override promotes LOW→GATED, override demotes
  HIGH→AUTONOMOUS, missing override falls through to risk. Autonomy
  level × policy matrix (MANUAL/OBSERVATION/AUTONOMOUS ×
  GATED/AUTONOMOUS). Session override takes precedence over preference.
  Reset clears override. Per-workspace preference lookup.
- **CountdownSchedulerTest** — schedule countdown, verify fires
  callback. Cancel before fire, verify no callback. cancelAll clears
  all pending. Deadline query returns correct time. Exception in
  callback does not kill scheduler.
- **CountdownRecoveryTest** — simulate restart: PROPOSED actions with
  countdown_ends_at in the past → autoExecute called. Future deadline
  → rescheduled with remaining time. Null countdown_ends_at → skipped.
- **OptimisticLockTest** — concurrent autoExecute and manual approve
  on same action: only one succeeds (CAS). Concurrent autoExecute and
  reject: reject wins, autoExecute is no-op. Concurrent autoExecute
  and expiry: expiry wins.
- **RateLimitTest** — autonomous executions within limit proceed.
  Exceeding limit falls back to observation countdown. Window slides
  — old timestamps pruned. Manual approval resets all timestamps.
- **ActionServiceAutonomyTest** — propose() in MANUAL mode leaves
  PROPOSED. propose() in AUTONOMOUS + LOW-risk auto-executes.
  propose() in AUTONOMOUS + HIGH-risk schedules countdown. propose()
  in OBSERVATION mode always schedules countdown. countdown_ends_at
  persisted on countdown schedule.
- **ModeSwitchTest** — switch to MANUAL cancels all countdowns, clears
  countdown_ends_at, actions remain PROPOSED. Switch from MANUAL to
  OBSERVATION does not retroactively countdown existing PROPOSED.
- **PreferencesServiceTest** — reads valid JSON. Defaults on missing
  file. Defaults on malformed JSON. Per-workspace autonomy lookup.
  Global overrides lookup. Rate limit config.

### Integration Tests (`@QuarkusTest`)

- **CoordinatorResourceAutonomyTest** — GET/POST autonomy endpoints.
  Toggle level, verify subsequent proposals follow the new level.
  Reset returns to default. Workspace parameter on GET.
- **ObservationCountdownIntegrationTest** — propose action in
  observation mode, verify countdown_ends_at persisted, verify
  auto-execution after delay, verify veto cancels execution. SSE
  reconnect: GET returns countdown_ends_at for recovery.

### Frontend

- Manual browser verification: mode toggle, countdown timer rendering,
  auto badge on autonomous completions, veto during countdown, timer
  recovery on page refresh.

## §8 Scope Boundary

### In scope
- `AutonomyLevel`, `AutonomyOverride` enums
- `AutonomyResolver` — encapsulates policy resolution, holds session state
- `CountdownScheduler` — countdown timer lifecycle, extracted from ActionService
- `PreferencesService` (`io.hortora.trellis.config`) — `~/.trellis/preferences.json`
- `ActionService.autoExecute()` — risk-gate-bypassing autonomous path
- Optimistic locking (CAS) on all action state transitions
- Countdown persistence (`countdown_ends_at` column) and restart recovery
- Mode-switch countdown cancellation
- Sliding-window rate limiter as circuit breaker backstop
- Platform notification wiring for autonomous action completions
- Parameterized `CoordinatorPrompts.systemPrompt(AutonomyLevel)`
- Coordinator panel mode toggle (session override)
- Countdown timer and auto badge on advice cards
- SSE payload + REST endpoint for countdown recovery on reconnect

### Out of scope
- ISX integration (paused — will be raised separately)
- Per-workspace override maps (overrides are global for now)
- Undo/rollback of autonomously executed actions
- Batch action proposals ("do all of these")
- Preferences UI panel (preferences are JSON-file-edited for now)
