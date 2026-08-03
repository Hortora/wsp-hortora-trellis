---
layout: post
title: "When the Coordinator Stops Asking Permission"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [coordinator, autonomy, concurrency, cas, architecture]
series: trellis-build
---

*Part of a series on [#18 — LLM Coordinator L3 + ISX](https://github.com/Hortora/trellis/issues/18). Previous: [When Your Agents Forget They're Mortal](2026-08-03-mdp01-when-your-agents-forget-theyre-mortal.md).*

The L2 coordinator was useful but needy. It could scan the dependency graph, spot that a slot was ready to merge, and propose the action — complete with rationale, risk classification, and an approve button in the advice card. Then it sat there. Waiting. The coordinator knew what to do but couldn't do it until someone clicked a button, and that someone was usually deep in a tmux session implementing the thing the coordinator was watching.

The gap between "knows what to do" and "does it" is entirely about trust. Not trust in the LLM's judgment — the risk classification is a static map, not an LLM opinion — but trust in the execution path. When you approve "merge slot X", you're trusting that the lifecycle manager will rebase cleanly, push to origin, and stamp the branch. You're trusting that a concurrent manual operation won't race with the automated one and corrupt state. You're trusting that the system won't enter a feedback loop where executing an action generates an event that triggers another proposal that auto-executes and generates another event.

Three levels of autonomy solve this incrementally. MANUAL is L2 behaviour — everything waits for a click. OBSERVATION adds a countdown: the action appears in the advice feed with a spinning timer, auto-executes after 30 seconds unless the user vetoes. AUTONOMOUS skips the countdown for low-risk actions and uses it only for high-risk ones. The levels are per-workspace preferences with a session toggle in the coordinator panel header — three buttons (MANUAL / OBS / AUTO) that call a REST endpoint and take effect immediately.

## The race that shapes everything

The interesting engineering problem isn't the autonomy model — it's the concurrency. When a countdown timer fires and calls `autoExecute()`, the user might simultaneously click "Approve" or "Reject" on the same card. Two threads, same action, both trying to transition from PROPOSED to a different state.

The fix is SQL compare-and-swap. Every state transition became:

```sql
UPDATE coordinator_actions SET status = ?
WHERE id = ? AND status = ?
```

The trailing `AND status = ?` is the CAS. If the row was already transitioned by a concurrent thread, the UPDATE affects zero rows and the caller returns without executing. No locks, no synchronisation, no distributed coordination — just a conditional write that the database enforces atomically. One thread wins, the other silently no-ops. The action executes exactly once.

This matters because `autoExecute()` deliberately bypasses the risk gate. In L2, `approve()` sends HIGH-risk actions to a CONFIRMING state that requires a second click. That's correct for manual approval — a human is there to confirm. But when the countdown fires, there's nobody to confirm. The autonomy resolver already evaluated whether this action should auto-execute; sending it to CONFIRMING with nobody listening would be a dead end. So `autoExecute()` goes PROPOSED → APPROVED → EXECUTING → COMPLETED in one path, and the CAS ensures that if someone vetoed during the countdown, the timer fires harmlessly.

## Countdowns that survive restarts

A countdown is a `ScheduledFuture` in memory. If the sidecar restarts, it's gone — and the action sits in PROPOSED forever, looking like it needs approval when it was supposed to auto-execute ten seconds ago.

The fix: persist `countdown_ends_at` in the database alongside the action. On startup, sweep all PROPOSED actions with a non-null deadline. Past the deadline? Execute immediately. Still in the future? Reschedule with the remaining time. The SSE payload carries the deadline too, so the frontend can recover the countdown timer after a page refresh — the circular spinner picks up where it left off, counting down from the right number.

## The backstop nobody should notice

Rate limiting autonomous execution sounds like throttling, but it's actually a circuit breaker. The coordinator proposes actions based on workspace events. An autonomous execution generates a lifecycle event. That event feeds back into the coordinator's accumulator. If the LLM decides the new state warrants another action, it proposes again. Without a guard, this is an unbounded loop.

The `SignificanceFilter` already blocks action-only event batches from triggering LLM calls — that's the primary defence from L2. The rate limiter is the backstop: a sliding window of timestamps, default five per minute. Exceed it and the next autonomous action falls back to observation countdown instead of immediate execution. A human clicking "Approve" resets the window — it's a trust signal that the coordinator is on track.

Under normal operation the rate limiter never fires. It catches the degenerate case where every proposed action's completion immediately triggers the next proposal — a tight loop that would look like useful work until it merged three slots in twelve seconds.

## What the user sees

The coordinator panel header gained three buttons — MANUAL, OBS, AUTO — with the active one highlighted. Clicking toggles the session override via REST. A "reset" link appears when the session diverges from the workspace preference.

Advice cards in observation mode show a spinning ring with a seconds countdown: "Auto-executing in 23s", flanked by "Approve Now" and "Veto" buttons. When the countdown expires, the card transitions to EXECUTING via SSE. Completed cards that auto-executed get a subtle "auto" badge — a visual distinction between "I approved this" and "the system handled it."

Toast notifications fire for autonomous completions — the user learns what happened even if they were looking at a different panel. Manual approvals don't notify; you already know.

The open question is whether the observation countdown duration should be action-type-specific. A slot merge might warrant 60 seconds of veto window; an epic-next probably needs 10. The current implementation uses a single global value from `~/.trellis/preferences.json`. That's probably wrong, but it's the kind of wrong that only shows up after real use — and it's a configuration change, not an architectural one.
