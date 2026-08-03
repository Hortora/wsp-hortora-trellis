---
layout: post
title: "When Your Agents Forget They're Mortal"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [process-management, memory, lifecycle, ai-agents, architecture]
series: trellis-build
---

*Part of a series on [#20 — Process isolation, memory monitoring, and session lifecycle](https://github.com/Hortora/trellis/issues/20). Previous: [Dependency Graphs and the Blackboard Coordinator](2026-08-01-mdp01-dependency-graphs-and-the-blackboard-coordinator.md).*

Running five Claude instances on a laptop with 16GB of RAM teaches you something fast: AI agents don't manage their own memory. Each one starts at 200MB, creeps to 400, then 600, and by the time you notice, three of them have leaked past half a gig and your fan sounds like a leaf blower. The only option is to kill them manually in tmux and restart with `claude -c`, hoping the conversation state survives. It always does — Claude Code stores sessions on disk — but the process is tedious and error-prone.

Trellis already managed terminals (tmux sessions with metadata) and git lifecycle (start/pause/resume/end branches). What it didn't have was any awareness of the Claude process running *inside* the terminal. It could create a tmux session, stream I/O through xterm.js, and destroy the session on close. But whether Claude was running, how much memory it was using, whether it had crashed — invisible.

## The naming problem that shaped the architecture

The first design question wasn't technical. It was linguistic.

Trellis had a `SessionInfo` record and a `SessionRegistry`. Claude Code uses "session" to mean a conversation history. The industry uses "agent" for an autonomous LLM-powered entity. If I kept calling everything a "session," three different concepts would share one word and every conversation about the system would be ambiguous.

I searched for consensus across the orchestration ecosystem — OpenAI's Agents SDK, LangChain, CrewAI, AutoGen, the A2A protocol, Claude Code's own documentation. The terminology has converged:

- **Agent** — the autonomous entity (Claude Code literally calls its dashboard `claude agents`)
- **Session** — the conversation state (what `claude -c` continues)
- **Terminal** — we added this for the tmux container, since neither "agent" nor "session" describes an I/O pipe

This isn't just naming. It's a layer model. Terminal is the I/O transport. Agent is the OS process with a PID and memory footprint. Session is the conversation that survives process restarts. Each layer has different lifecycle semantics — a terminal outlives its agent (pause kills the process, terminal stays), and a session outlives its terminal (restart the terminal, `-c` restores the conversation).

## Discovery, not ownership

The architectural choice that matters most: how does Trellis learn about Claude processes?

Two options. Trellis could *own* the process — start Claude itself, capture the PID at birth, track it from there. Clean, immediate, but fragile. What if the user started Claude manually? What if the process crashed and a new one appeared? The owned-process model breaks on every edge case.

The alternative is *discovery*. Every five seconds, poll each terminal: `tmux display-message -t <name> -p '#{pane_current_command}'`. If the foreground command is a shell (`zsh`, `bash`), no agent is running. If it's `node` (Claude Code is a Node.js application), walk the process tree from the pane PID, find the `claude` process by scanning `args`, and sum RSS across the whole tree — Claude plus its MCP servers and subagents.

This pattern already existed. Claudony — the casehub platform's session management layer — uses `StatusAwareExpiryPolicy` with exactly this introspection. It checks `#{pane_current_command}` against a set of known shell commands and treats anything else as an active foreground process. I ported the pattern directly. The proof that it works in production was already written.

The hybrid approach auto-starts Claude when you create a terminal (immediate PID awareness for the common case) and runs discovery as a self-healing fallback (handles manual starts, crashes, and every edge case the owned-process model can't). Auto-start is a convenience; discovery is the ground truth.

## What the process tree actually tells you

RSS — Resident Set Size — is what `ps` reports for memory. It overcounts by 30-50% for a typical Claude process tree because shared memory-mapped pages (the Node.js runtime, shared libraries) are counted once per process even when physically shared. Three processes sharing 100MB of mapped pages report 300MB of RSS.

This matters for setting thresholds. The 500MB warning threshold in the UI is calibrated against RSS-reported values, not actual physical memory. It's a heuristic — useful for spotting runaway growth, not for precise accounting. Acceptable for a warning display that triggers human action; would not be acceptable for an auto-kill trigger, which is why we don't have one.

The tree walk itself is straightforward: parse `ps -eo pid=,ppid=,rss=,args=`, build a parent-to-children map, recurse from the pane PID looking for a process whose args contain `claude`. Sum RSS for Claude plus all descendants. The `args` column is necessary because `comm` (the short command name) shows `node`, not `claude` — Claude Code is a Node.js application, and the process table doesn't know about the JavaScript entry point.

## Pause, resume, refresh — and the naming collision

"Pause" in Trellis already meant something: commit WIP, push branches to a stack, switch to main. That's slot pause — git workspace preservation. Agent pause is different: kill the Claude process, free the memory, mark the terminal as paused. The terminal stays alive. The conversation state is on disk. Resume starts `claude -c` and picks up where it left off.

These are independent operations at different layers that happen to share a verb. The spec includes a disambiguation table to make this explicit. Coordination — slot pause automatically pausing its agents — is a future concern, tracked separately. For now, they're orthogonal.

Refresh is the one that gets the most use: kill the Claude process and immediately restart with `-c`. One click to reclaim leaked memory without losing context. The implementation sets the state to STARTING before killing the process, so the monitor never sees a transient IDLE state between kill and restart — a race condition the design review caught.

## What the design review surfaced

The spec went through a standard adversarial review — coherence, structure, robustness, and cross-cutting dimensions. The ones that shaped the final design:

The structure review merged two components into one. I'd originally split monitoring (scheduled poll) and lifecycle operations (start/stop/pause/resume) into `ProcessMonitor` and `AgentLifecycleManager`. The reviewer correctly identified that they share mutable state — the lifecycle operations need PID data from monitoring, and the monitor must respect lifecycle invariants like PAUSED. Splitting them creates coordination overhead without architectural clarity. `AgentProcessManager` owns both.

The robustness review caught that `ProcessHandle.destroyForcibly()` sends SIGKILL to a single process. My spec said "send SIGKILL to the entire process group" — but Java's ProcessHandle API has no process-group concept. The fix is a leaf-first tree walk: kill descendants before the root, working inward from the leaves. MCP servers and subagents die before Claude, preventing orphaned children.

The cross-cutting review found that `pauseAgent` had the same transient-state race as `refreshAgent`. Kill the process, then set PAUSED — but the monitor runs every five seconds and might observe the gap. The fix is a `tryLock` per terminal: the monitor uses `tryLock` and skips terminals locked by lifecycle operations. At most one monitor cycle is delayed, which is invisible to users.

The open question now is whether agent lifecycle and slot lifecycle need coordination. Today they're independent — you can pause an agent without pausing the slot's git workspace, and vice versa. But a slot that pauses its git workspace while agents are still running is probably a mistake. That coordination is tracked as a separate issue, not crammed into this one.
