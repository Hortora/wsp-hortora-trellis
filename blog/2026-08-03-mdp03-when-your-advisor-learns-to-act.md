---
layout: post
title: "When Your Advisor Learns to Act"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [llm, coordinator, actions, state-machine]
series: epic-2-post-mvp
---

*Part of a series on [#2 — Epic: Trellis](https://github.com/Hortora/trellis/issues/2). Previous: [When Your Agents Forget They're Mortal](2026-08-03-mdp01-when-your-agents-forget-theyre-mortal.md).*

The L1 coordinator watches your epic and tells you things. Useful things — which issue to pick up next, when a dependency unblocks, why the algorithm's top recommendation might be wrong. But it's read-only. You read the advice card, nod, dismiss it, and go do the thing yourself.

L2 closes that loop. The coordinator now proposes actions you can approve with a button click, and the system executes them.

## The Model That Already Existed

The interesting thing about L2 is how little new architecture it needed. L1's `CoordinatorAdvice` record already had an `actionKey` field — nullable, unused, sitting there since day one. That was the hook. An advice card with an `actionKey` is a proposed action. One without is a read-only insight. The frontend renders approve/reject buttons when the key is present; dismiss-only when it isn't.

The new entity is `ProposedAction` — linked to advice by that key, carrying the action type, parameters, risk level, and a status that tracks the full lifecycle: proposed, approved, confirming, executing, completed, failed, rejected, expired.

## The State Machine and the Severity Gate

Not all actions are equal. Advancing to the next epic issue is low-risk — approve it and it executes immediately. Merging a slot into main is high-risk — approve it and you get a confirmation step first, showing you what will happen before you commit. The risk classification is a static map, not LLM-determined. The LLM proposes; the system classifies.

```
PROPOSED → APPROVED → EXECUTING → COMPLETED
                                 → FAILED
PROPOSED → CONFIRMING → EXECUTING  (HIGH risk path)
         → REJECTED
         → EXPIRED
```

The cancel path from CONFIRMING back to PROPOSED matters more than it looks. Without it, a user who clicks approve on a high-risk action and then thinks better of it has no way back except reject — which is a terminal state. Cancel lets you reconsider without losing the proposal.

## The Feedback Loop Problem

Here's where it got interesting. The coordinator observes events, accumulates them in a window, and when the batch is significant enough, asks the LLM whether to generate advice. L2 adds two new event types: `ActionStateChangedEvent` (an action transitioned state) and `LifecycleOperationEvent` (a lifecycle operation completed). These feed into the same accumulator so the LLM has richer context.

The problem: action state changes ARE events. The LLM sees them, might propose more actions, which generate more state changes, which trigger more LLM calls. Unbounded feedback loop.

The circuit breaker is simple: if a batch contains only action and lifecycle events — no workspace changes, no analysis updates, no issue events — and the count exceeds a threshold, the batch is classified as not significant. The coordinator only proposes new actions when real workspace activity accompanies the action churn. Terminal state transitions (expired, rejected) are excluded from the accumulator entirely — they're bookkeeping, not signals.

## Three Executor Categories

The `ActionExecutor` interface has three implementations today:

**LifecycleActionExecutor** wraps the existing `LifecycleManager` — start, end, pause, resume, slot create, slot merge, epic setup, epic next. The tricky bit is that `LifecycleManager` methods throw checked exceptions (`IOException`, `InterruptedException`, `ConcurrentOperationException`) while the executor interface returns `ActionResult`. The executor catches and converts.

**AgentActionExecutor** is a stub. It maps to the agent process management from issue #20 — start, stop, pause, resume, refresh. The interface is designed; the implementation returns "not yet available" until the process isolation work lands.

**AdvisoryActionExecutor** handles soft suggestions — "prioritise issue #5", "investigate this blocker". Executing an advisory action just records the acknowledgement. No side effects, but the audit trail captures that the user saw and accepted the recommendation.

## What the LLM Sees

The system prompt now includes the full action type catalogue with expected parameters. When the LLM identifies something actionable, it includes a nested `action` object in its JSON response alongside the advice. The parser uses `jakarta.json` — L1's hand-rolled string extraction couldn't handle nested objects, which was one of the first things the code review caught.

The context assembler also feeds pending actions and recent action outcomes back into the prompt, so the LLM doesn't propose duplicates and can learn from what worked and what failed.

## The Atomicity Catch

The advice and action must be created in the same SQLite transaction. If they're created separately, there's a window where the frontend receives an advice card via SSE with an `actionKey` that points to an action that doesn't exist yet. The SSE broadcasts go out after the transaction commits — both in the same call, advice first.

The interactive path (user asks a question, LLM responds with an action proposal) originally created the action with a random UUID as `adviceId` — violating the foreign key constraint. The proactive path got this right from the start because it created both atomically. The fix was straightforward: both paths now use the same `persistAdviceWithAction` method.

---

The coordinator can now observe, advise, AND act — with the human always in the loop. L3 is where that loop gets optional: autonomous execution without approval, with the coordinator deciding when it's confident enough to act on its own. The severity gate and circuit breaker from L2 become the guardrails for that autonomy.
