# LLM Coordinator L3 — Autonomous Execution with Observation Mode

**Issue:** Hortora/trellis#18
**Date:** 2026-08-02
**Status:** Draft

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

New infrastructure: `~/.trellis/preferences.json` — a JSON file read
by a new `PreferencesService` (`@ApplicationScoped`). This is separate
from Quarkus `@ConfigMapping` (`CoordinatorConfig`), which provides
compile-time defaults from `application.properties`. Preferences are
user-editable at runtime; config properties are not.

```json
{
  "coordinator.autonomyLevel": "MANUAL",
  "coordinator.observationCountdownSeconds": 30,
  "coordinator.autonomyOverrides": {
    "slot.create": "GATED",
    "lifecycle.end": "GATED"
  }
}
```

`autonomyLevel` is per-workspace (keyed by workspace root path).
`observationCountdownSeconds` and `autonomyOverrides` are global.

`PreferencesService` reads the file on startup and on demand. It is
the single source for all user-configurable trellis settings — future
preferences (layout, theme, etc.) go here too.

### CoordinatorConfig (Unchanged)

Existing `CoordinatorConfig` properties (`enabled`, `windowTimeMs`,
`windowCount`, etc.) remain in `application.properties`. The new
autonomy settings live in preferences, not config — they are
user-preference concerns, not deployment concerns.

### Session State

`CoordinatorService` holds a volatile `AutonomyLevel sessionAutonomyLevel`
field. `null` means "use preference default." The coordinator panel toggle
writes to this via REST. On sidecar restart, it resets to null.

### REST API

```
GET  /api/coordinator/autonomy
     → { "level": "OBSERVATION", "source": "session"|"preference" }

POST /api/coordinator/autonomy?level=OBSERVATION
     → sets session override, returns updated state

POST /api/coordinator/autonomy/reset
     → clears session override, returns to preference default
```

## §3 Action Lifecycle Changes

The L2 state machine stays unchanged — no new states, no new
transitions. The difference is who triggers the transitions.

### ActionService.propose() — Autonomy-Aware Path

After creating the action in PROPOSED and broadcasting it, `propose()`
checks the effective autonomy:

```
1. Resolve current autonomy level (session override ?? preference default)
2. If MANUAL → return (L2 behaviour, action sits in PROPOSED)
3. Resolve effective policy for this action type:
   a. Check override map → if present, use it
   b. Fall back to RiskClassification.riskFor(actionType)
      → LOW maps to AUTONOMOUS policy, HIGH maps to GATED policy
4. If AUTONOMOUS level + AUTONOMOUS policy → call approve() immediately
5. If OBSERVATION level, or AUTONOMOUS level + GATED policy → schedule countdown
```

The action always hits PROPOSED first and is always broadcast via SSE.
The UI sees every action, even ones that auto-execute — the feed is a
complete audit record.

### Countdown Mechanism

A `ScheduledExecutorService` in `ActionService` (single-threaded, same
as the existing action executor thread).

```java
record PendingCountdown(String actionId, ScheduledFuture<?> future) {}
private final Map<String, PendingCountdown> countdowns = new ConcurrentHashMap<>();
```

When the timer fires: re-read the action from the DB. If still PROPOSED,
call `approve()`. If already REJECTED or EXPIRED (user vetoed or state
changed), do nothing. The DB is the source of truth — the timer doesn't
hold state.

### Veto During Countdown

The existing `reject()` method works unchanged. After rejection, the
timer fires, reads REJECTED from DB, and is a no-op. The `countdowns`
map entry is cleaned up on the next fire or on a periodic sweep.

### Auto-Expiry During Countdown

Same pattern — `expireStale()` transitions to EXPIRED, the timer fires
and sees a terminal state.

### SSE Payload Extension

When a countdown starts, the SSE broadcast for `coordinator:action`
includes a `countdownEndsAt` ISO-8601 timestamp. This is transient
metadata on the broadcast, not persisted in the `ProposedAction` record
or the `coordinator_actions` table. The frontend uses it to render the
countdown timer.

## §4 Circuit Breaker & Safety

### Existing Mitigation (Unchanged)

`SignificanceFilter` suppresses batches containing only action/lifecycle
events when the count exceeds 5 per window. Terminal state events
(EXPIRED, REJECTED) are excluded from the accumulator entirely.

### Autonomous Execution Rate Limit

`ActionService` tracks autonomous executions per rolling window.
Default: 5 actions per 60 seconds (configurable via preferences).

When the limit is hit, subsequent actions fall back to observation mode
(countdown) instead of auto-executing. The rate limit resets when:
- The window rolls over
- A human explicitly approves an action (signal that the coordinator
  is on track)

```java
private final AtomicInteger autonomousActionsInWindow = new AtomicInteger(0);
```

This is a backstop, not a throttle. Under normal operation it never
fires. It catches the degenerate case where the coordinator proposes,
executes, observes the result, and immediately proposes again in a
tight loop.

### Prompt Extension

`CoordinatorPrompts.systemPrompt()` is extended to inform the LLM of
the current autonomy mode:

```
The coordinator is in {OBSERVATION|AUTONOMOUS} mode. LOW-risk actions
will {execute after a countdown|execute immediately}. Be more selective
about proposing actions — only propose when confidence is high, since
actions may execute without explicit approval.
```

This makes the LLM more conservative about proposals when it knows they
may auto-execute — a soft guard on top of the hard rate limit.

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

- **AutonomyResolutionTest** — effective policy resolution: risk-based
  defaults, override promotes LOW→GATED, override demotes
  HIGH→AUTONOMOUS, missing override falls through to risk. Autonomy
  level × policy matrix (MANUAL/OBSERVATION/AUTONOMOUS ×
  GATED/AUTONOMOUS) produces correct behaviour.
- **CountdownTest** — schedule countdown, verify fires and auto-approves.
  Veto during countdown, timer is no-op. Expiry during countdown, timer
  is no-op. Race condition: action already executed, no double-execute.
- **RateLimitTest** — autonomous executions within limit proceed.
  Exceeding limit falls back to observation. Window rollover resets
  counter. Manual approval resets counter.
- **ActionServiceAutonomyTest** — propose() in MANUAL mode leaves
  PROPOSED. propose() in AUTONOMOUS + LOW-risk auto-executes.
  propose() in AUTONOMOUS + HIGH-risk schedules countdown. propose()
  in OBSERVATION mode always schedules countdown.
- **SessionAutonomyToggleTest** — session override takes precedence
  over preference. Reset clears override. Default used when no override.

### Integration Tests (`@QuarkusTest`)

- **CoordinatorResourceAutonomyTest** — GET/POST autonomy endpoints.
  Toggle level, verify subsequent proposals follow the new level.
  Reset returns to default.
- **ObservationCountdownIntegrationTest** — propose action in
  observation mode, verify SSE includes `countdownEndsAt`, verify
  auto-execution after delay, verify veto cancels execution.

### Frontend

- Manual browser verification: mode toggle, countdown timer rendering,
  auto badge on autonomous completions, veto during countdown.

## §8 Scope Boundary

### In scope
- `AutonomyLevel` enum and effective policy resolution (hybrid risk +
  overrides)
- Preferences persistence for autonomy level, countdown duration,
  overrides
- Session-level autonomy toggle with REST API
- Observation countdown mechanism in `ActionService`
- Autonomous auto-execution path in `ActionService.propose()`
- Rate limiter as circuit breaker backstop
- Platform notification wiring for autonomous action completions
- Coordinator panel mode toggle
- Countdown timer and auto badge on advice cards
- Prompt extension for autonomy-aware LLM behaviour
- SSE payload extension with `countdownEndsAt`

### Out of scope
- ISX integration (paused — will be raised separately)
- Per-workspace override maps (overrides are global for now)
- Undo/rollback of autonomously executed actions
- Batch action proposals ("do all of these")
- Preferences UI panel (preferences are JSON-file-edited for now)
