---
layout: post
title: "Making lifecycle operations visible"
date: 2026-09-26
entry_type: note
subtype: diary
projects: [Hortora/trellis]
tags: [trellis, lifecycle, async, sse, electron]
---

Until now, clicking "end" on a slot in the trellis dashboard was a trust exercise. The button went grey, the UI froze for ten to thirty seconds, and eventually either the slot disappeared or an error appeared. No indication of what was happening — whether the rebase was stuck, the push was timing out, or the agent shutdown had stalled. The same fire-and-hope pattern for pause and resume.

I wanted the slot modal to show what's actually happening: each step rendered as it executes, with captured output available on expand. The kind of thing you'd see in a CI pipeline view — done, running, pending — but for lifecycle operations.

The interesting design question was how to decouple the HTTP request from the multi-step execution. The existing `LifecycleManager` runs scripts sequentially inside a locked section — rebase, push, stamp for an end operation. Making the frontend wait for all three to complete before getting any response meant no intermediate progress was possible. We needed the POST to return immediately and stream step events through a separate channel.

SSE was the obvious transport — trellis already uses `EventBroadcaster` for workspace file-watch events, agent state, and control commands. A new `lifecycle:progress` topic carries step transitions. The slot-detail component subscribes directly (not through the global workspace SSE — lifecycle progress is only relevant when a slot modal is open).

The trickier part was the execution model. `SlotAgentCoordinator` now uses `Semaphore(1)` instead of `ReentrantLock` for the async methods — the HTTP thread acquires the permit, the `ManagedExecutor` thread releases it in a `finally` block. ReentrantLock wouldn't work here because it has thread affinity: the thread that locked must be the thread that unlocks. A subtle distinction that the design review caught before it became a runtime `IllegalMonitorStateException`.

Claude caught a genuine bug during code review: `_nextEpic()` was calling the updated `_lifecycleAction()` method, which now parses the response as `OperationProgress`. But the epic-next endpoint still returns the old `OperationResult` — different JSON shape, no `steps` array. The template would have thrown at runtime trying to iterate `undefined`. A one-method contract change that silently broke an unchanged caller — the kind of thing that's easy to miss in a diff because the broken code didn't change.

The `LifecycleOperationTracker` carries a file-backed durability layer: each step transition atomically writes to `.trellis/operations/{id}.json` via tmp-then-rename. On sidecar restart, incomplete operations are loaded and any step left in RUNNING state is marked FAILED — the executing thread is gone, so the step can't complete. The frontend re-fetches via the GET endpoint and sees exactly where things stopped. It's a small amount of file I/O per operation (roughly seven writes for a four-step end), but the alternative — losing all progress visibility on a crash — didn't seem acceptable for an operation that mutates git state.

The frontend rendering follows the plan progress pattern that already exists in the sidebar — same icon vocabulary (checkmarks, dots, circles), same colour scheme, same expandable output idiom. A `LIFECYCLE` section appears between Status and Issues when an operation is running, collapses to nothing when idle. Failed steps auto-expand their stderr in a red-bordered monospace block. The whole section fades after ten seconds on success.

What this opens up: the same `OperationProgress` model could serve a workspace-level progress indicator for `start` operations — right now start is synchronous and fast, but epic setup with multiple repos could benefit from the same visibility. The tracker is generic enough that any future multi-step async operation can use it without new infrastructure.
