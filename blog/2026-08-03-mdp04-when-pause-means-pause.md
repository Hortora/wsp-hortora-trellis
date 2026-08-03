---
layout: post
title: "When Pause Means Pause"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [lifecycle, memory, coordination]
---

Trellis has had slot pause/resume since the first epic — commit WIP, push branches to stack, switch to something else. And since #20 landed, it's had agent lifecycle management — start, stop, pause, resume individual Claude processes. Two systems, completely independent. Pause a slot and the agents keep running, burning memory for no reason. Resume a slot and you're manually restarting each agent.

The obvious fix is wiring them together. The less obvious part is where the wire goes.

I considered three approaches: direct calls from `LifecycleManager` into `AgentProcessManager`, CDI events, or a dedicated coordinator. The first couples two domains that currently know nothing about each other. The second has a timing problem — agent shutdown needs to complete *before* git ops run, and CDI events are async by default. The third adds a class but keeps both existing components unchanged.

`SlotAgentCoordinator` sits between the REST layer and the existing managers. On pause: shut down agents first, then commit WIP. On resume: restore workspace first, then restart agents. The sequencing is the whole point — it's why the coordinator exists rather than being an event listener.

The graceful shutdown path is worth describing. My first thought was a simple SIGTERM, but the existing `treeKill` does that already — 5 seconds of grace, then SIGKILL. What I actually wanted was for Claude to exit cleanly through its own shutdown path. The sequence: send Escape (interrupts any active generation), check if Claude already exited, send `/exit` if not, poll for the shell prompt, fall back to `treeKill` on timeout. Escape works because `tmux send-keys` delivers it regardless of whether Claude is streaming.

One thing the design review caught that I'd missed: resume was going to restart every paused agent in the slot, including ones the user had manually paused. If I pause an agent because it's misbehaving, then pause the slot, then resume the slot — that agent should stay paused. We added `PAUSED_BY_COORDINATOR` as a distinct state. The coordinator marks agents it pauses; on resume it only restarts those. User-paused agents are left alone.

The memory pressure side landed differently than I expected. I started with the idea of auto-restarting agents when their RSS gets too high. But the more I thought about it, the less I wanted the system making that decision. An agent at 800 MB might be mid-way through something important. What I want is visibility — the system tells me who's heavy, I decide what to evict.

So the eviction queue is advisory. The monitor (same 5-second poll that already collects RSS) checks per-agent thresholds and system pressure, ranks candidates, broadcasts the list via SSE. The frontend shows pulsing red badges on eviction candidates and a system pressure banner when aggregate memory is high. An "Evict" button appears — it's just a pause. No automatic action.

The system memory check needed platform-specific handling. `OperatingSystemMXBean.getFreePhysicalMemorySize()` is useless on macOS — the OS aggressively caches to RAM, so "free" memory is routinely under 200 MB even when there's no real pressure. We use `memory_pressure` instead, which actually understands macOS's compressed memory system.

The lock ordering constraint from the process isolation spec (#20) turned out to be easy to satisfy. The spec says `LifecycleManager` lock before `AgentProcessManager` lock. The coordinator doesn't nest locks at all — it sequences operations. Agents first, then git ops. No simultaneous holding. The coordinator does have its own slot-keyed lock to prevent concurrent pause/resume on the same slot from interleaving.

The design review surfaced ten findings worth addressing. The slot-level lock, the PAUSED_BY_COORDINATOR provenance, and parallel agent shutdowns (to avoid O(n × timeout) blocking on the REST thread) were the ones that changed the spec. The macOS memory API issue would have been a runtime surprise.

The open question is whether the eviction queue needs a protection mechanism — a way to mark certain agents as "don't recommend for eviction." For now, the queue is advisory and the user can just ignore candidates. If that becomes friction, it's a boolean on `TerminalInfo`.
