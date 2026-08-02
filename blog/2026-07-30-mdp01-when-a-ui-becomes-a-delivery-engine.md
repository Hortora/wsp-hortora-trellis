---
layout: post
title: "When a UI Becomes a Delivery Engine"
date: 2026-07-30
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [architecture, design-review, electron, quarkus, parallel-development]
series: trellis-build
---

The original ask was modest: "explore building a UI for work lifecycle management." Issue #117 on soredium. A dashboard showing slots, paused branches, epic progress. The kind of thing you build in a weekend with a REST endpoint and some HTML.

The brainstorming went somewhere else entirely.

The real problem isn't visibility — it's coordination. When you're running parallel development across 20+ repos with worktree slots, the bottleneck is the developer doing dependency graph traversal in their head. Which issue should I start next? Which one unblocks the most downstream work? Is this issue on the critical path, or can it wait? We answer those questions by intuition, and we get them wrong often enough that it matters.

A dashboard shows you state. A delivery engine tells you what to do next and why.

## From dashboard to advisor

We designed Trellis around progressive autonomy — four levels, each building on the one below. L0 is the dashboard: human decides everything, trellis shows state. L1 is the advisor: trellis recommends, human decides. L2 is the copilot: trellis proposes actions, human approves. L3 is the autopilot: trellis drives delivery, human steers at epic level. The MVP targets L1 — algorithmic recommendations with natural language reasoning from an LLM.

The architecture is an Electron shell backed by a Quarkus sidecar. Electron provides native multi-window management (drag-to-detach panels, multi-monitor layout persistence) that browser `window.open()` cannot reliably deliver. The sidecar is the persistent process — it survives Electron quit and reconnects on relaunch. Dynamic port allocation, health polling, crash recovery with exponential backoff. This is a pattern we've proven in sparge: Electron launches the jar, no Node.js application logic needed.

The sidecar houses seven services — workspace scanning, tmux terminal management, GitHub issue intelligence, work lifecycle operations, and more. The most interesting design choice is the Work Lifecycle Manager: it wraps soredium's existing Python scripts (`land_branch.py`, `pause_exec.py`, `resume_exec.py`) as REST endpoints, so button clicks call the same code as the CLI. Two-tier invocation: the button-click path is a fast lane for mechanical operations that deliberately skips code review. The full-featured path runs through a Claude Code session in the terminal panel. The UI labels the button "work-end (skip review)" to make the semantic difference visible.

## Ten rounds of adversarial review

Before writing any code, we ran the spec through ten rounds of adversarial design review — a separate Claude session, given the spec cold, tasked with finding everything wrong with it. Total cost: $44.47. The severity descent was clean: architectural decisions overturned in rounds 1-2, internal consistency in 3-4, factual accuracy in round 6 (a fresh session that verified script subcommand names against the actual codebase), then four consecutive APPROVEDs.

The most interesting exchange was about lifecycle operations. The spec said the sidecar would "invoke the work-start skill" on button click — but skills are LLM instruction sets, not executables. You can't call a skill from a REST endpoint. The reviewer caught the flaw, but then asserted that the lifecycle skills had no externalised scripts at all. That was wrong. All four lifecycle skills have heavily externalised Python scripts — `work-end/land_branch.py`, `work-pause/pause_exec.py`, `work-start/branch_create.py` — they just live at skill root level, not under a `scripts/` subdirectory. The reviewer had searched the wrong directory structure.

Both sides were partially right. The wording was wrong (you can't "invoke a skill" from Java). The architecture was sound (the scripts exist and are callable via ProcessBuilder). The fix: a complete script-to-subcommand mapping for every lifecycle operation, making explicit that no Claude Code process is needed.

The review also challenged the Electron decision. Every CaseHub application is browser-served Quarkus — Electron was unprecedented. The reviewer demonstrated that `window.open()`, `BroadcastChannel`, and `IndexedDB` handle the multi-window requirements. Fair point. But browser APIs genuinely cannot do programmatic cross-monitor window positioning or persistent layout memory across restarts. We kept Electron as the primary path with browser fallback documented as a future option.

Beyond the big debates, the review caught operational gaps that would have surfaced in production. Java's WatchService on macOS uses kqueue, which drops events under high filesystem throughput — periodic rescan every 60 seconds mitigates. There was a race condition between "REST API returns slot created" and "dashboard shows the slot" — direct CDI event notification from the lifecycle manager to the scanner bypasses the filesystem watcher entirely. The dependency graph needed cycle detection, because people do accidentally create circular dependencies in issue trackers.

The spec gained explicit failure modes for every service, and landed in its own repository — [hortora/trellis](https://github.com/Hortora/trellis) — with a 52-task implementation plan across 10 batches. Time to build it.
