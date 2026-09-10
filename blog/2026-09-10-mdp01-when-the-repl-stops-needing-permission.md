---
layout: post
title: "When the REPL Stops Needing Permission"
date: 2026-09-10
entry_type: note
subtype: diary
projects: [Hortora/trellis]
tags: [repl, tamboui, mechanical-first, soredium, design]
series: issue-75-tamboui-repl
---

# When the REPL Stops Needing Permission

I've been circling this problem for months. Trellis has terminals, agents, a coordinator — all the pieces for managing multi-repo work at scale. But every interaction goes through an LLM. Want to start work? Ask Claude. Pause? Ask Claude. Check status? Ask Claude. The LLM is doing mechanical work that a shell script could handle, at the cost of tokens and latency.

The soredium TUI was my first attempt at fixing this. Python, Textual, a portable `commands/` layer for lifecycle operations. It worked. But it lived outside Trellis — a separate tool that couldn't see terminals, agents, or the workspace. I put it on hold.

Today I came back to the idea with a different framing: what if the REPL lived inside Trellis? Same tmux infrastructure, same modals, same terminal rendering. But instead of an LLM in one terminal, you get a REPL in one and an LLM in the other. The REPL does the mechanical work. The LLM does the work that actually needs reasoning. Neither is mandatory.

## The Mechanical-First Principle

This is the core design decision. The work lifecycle — `work start`, `work pause`, `work resume`, `work end`, `work next` — runs without an LLM at all. These are state machine transitions: check the branch, read the `.plan`, call `git`, update the lifecycle state. No reasoning required.

The LLM becomes an augmentation layer. When you want a sweep (forage, protocol, doc-sync), a semantic squash, a code review, or a blog entry — those need reasoning. The REPL dispatches them to the paired LLM terminal by typing commands into it via `sendKeys`. From the LLM's perspective, a human typed the command. The session history accumulates naturally.

Work-end splits into two paths: mechanical (close, merge, push — done) or LLM-assisted (sweep, squash, review, close — richer but optional). You choose at the moment, not at architecture time.

## Why Tamboui

I wanted Java on the UI side. Tamboui is a TUI framework by Cedric Champeau — widgets, CSS, a fluent DSL, and a headless test harness called Pilot. I've already used it in incus-spawn. The bootstrap pattern is clean: `TuiRunner.create(TuiConfig)` → `runner.run(eventHandler, renderFn)`, event loop distinguishing `KeyEvent` and `TickEvent`, render function receiving a `Frame`.

The REPL is not Quarkus. That was a deliberate decision — the design review caught it before I could over-engineer. The REPL needs Tamboui and an HTTP client. CDI, REST endpoints, Quarkus lifecycle are overhead with no benefit. A plain `main()` entry point, Tamboui for the TUI, `java.net.http` for talking to the sidecar. Start-up in milliseconds, not seconds.

## Commands and Goals

Two interaction modes. **Commands** are atomic — type `work status` with tab completion, it runs. **Goals** are guided multi-step flows — express an intent like "start work", the REPL walks you through questions (which issue? ISX isolation? confirm?), shows a summary, then executes.

Both are defined in YAML, loaded at runtime. Adding a command is a config edit, not a recompile. The command tree is namespace-based — `work`, `git`, `project`, `llm` — with a unified text-and-click input widget where typing filters suggestions and mouse clicks select them directly. Same widget, two input modes.

## The Bridge Problem

Soredium's commands are Python modules with `execute()` functions that return event objects — `StatusReady`, `BranchCreated`, `CommandFailed`. They're not CLI entry points. You can't just shell out to them.

We built a JSON Lines CLI wrapper: `python3 -m cli status '{"cwd": "/path"}'`. Each event serialised as a JSON line to stdout. The Java `SorediumBridge` spawns the subprocess, reads the lines, and surfaces structured events to the TUI. It's a thin bridge — the Python logic stays in Python, the Java logic stays in Java, and the protocol between them is a stream of typed events.

## What's Built

The `repl/` module has:

- **ReplApp** — Tamboui TUI with status bar, output area, and input field. Processes commands via the registry, renders results, exits on `quit` or Escape.
- **CommandRegistry** — parses YAML command trees, resolves namespace paths (`work status` → handler), tab-completes partial input.
- **GoalRunner** — walks Q&A steps, collects answers, shows summary, executes composed commands on confirmation.
- **SorediumBridge** — subprocess invocation of the Python CLI wrapper, JSON Lines event parsing.
- **SidecarClient** — `java.net.http` client for the Trellis sidecar REST API plus SSE subscription for agent state.
- **HandlerDispatcher** — routes handler prefixes (`shell:`, `soredium:`, `llm:`) to execution backends.
- **StatusModel** — thread-safe status bar model, updated reactively from SSE events.

32 tests, all in the `repl/` module. The soredium CLI wrapper is a separate commit in the soredium repo.

## What's Next

The REPL core is functional but not yet wired into Trellis's UI. Three pieces remain:

The **SuggestionInput widget** — Tamboui's `TextInput` handles text entry, but the unified suggestion dropdown (text + clickable options) needs building. This is the core UX piece and needs hands-on exploration of Tamboui's widget composition model.

The **modal view integration** — the repo/slot detail modals need a split-view mode showing REPL and LLM terminals side by side, with tab mode as an alternative. This is frontend TypeScript work in the existing components.

And the **sidecar spawning** — Trellis needs to know how to create REPL terminals, spawn the Java process with the right args, and manage the paired terminal lifecycle.

The interesting thing about this design is what it implies for the broader system. If the work lifecycle is mechanical, then soredium's role shifts from "the thing that drives Claude" to "the portable command layer that anything can drive." The REPL drives it from inside Trellis. A CLI could drive it from a terminal. A web UI could drive it from a browser. The LLM becomes one consumer among many, not the orchestrator.
