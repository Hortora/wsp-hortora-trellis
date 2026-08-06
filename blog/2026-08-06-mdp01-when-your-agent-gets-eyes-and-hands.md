---
layout: post
title: "When Your Agent Gets Eyes and Hands"
date: 2026-08-06
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [mcp, agent, control-plane, architecture, spi]
series: trellis-build
---

*Part of a series on [#27 — Agent Control Plane](https://github.com/Hortora/trellis/issues/27). Previous: [When Thirty Terminals Need a Home](2026-08-05-mdp01-when-thirty-terminals-need-a-home.md).*

Trellis can spin up thirty terminals across seven repos, track which agents are running, monitor memory pressure, coordinate slot lifecycle operations. But until today, the only thing that could see and act on any of that was a human sitting in front of the Electron window. The coordinating agent — the one that's supposed to orchestrate all of it — was blind.

The problem isn't access to individual APIs. The sidecar already has REST endpoints for terminals, agents, workspace scanning, lifecycle operations. The problem is that an LLM connecting via MCP sees the tool list at connection time, and that tool list is its entire understanding of what's available. If Trellis exposes one tool per operation, the tool list grows with every feature. If it exposes too few, the agent can't discover what's possible. Neither extreme scales.

## The raggable model

The design we landed on is what I'm calling "raggable MCP" — the LLM retrieves what it needs, when it needs it. Six tools, fixed. New capabilities grow the model, not the tool list.

`trellis_model` is the entry point. Call it with no path and you get the full application state as a tree — every terminal with its slot, repo, agent state, available actions. Every panel in the UI with its content. The workspace summary. A generation counter for freshness detection. Call it with a path like `terminals/engine` and you get just that subtree with full detail.

The actions on each node tell the agent what it can do. A terminal node carries `send-input`, `read-log`, `start-agent`, `pause-agent`, `destroy` — each one naming the tool and operation to call. The agent doesn't need to know the API surface in advance. It discovers it by reading the model.

The remaining five tools are category dispatchers: `trellis_terminal` for I/O operations, `trellis_agent` for agent lifecycle, `trellis_lifecycle` for slot coordination, `trellis_workspace` for full workspace queries, and `trellis_navigate` for UI navigation. Each accepts an operation string and parameters. New operations within a category don't add new tools — they appear as action declarations in the model tree.

## The extension mechanism is the SPI

The model tree is assembled at query time from `ModelProvider` implementations — one per domain. `TerminalModelProvider` pulls from `TerminalRegistry` and `AgentProcessManager`. `WorkspaceModelProvider` pulls from `FileWatcherService`. `UIStateModelProvider` holds whatever the frontend last pushed.

Adding a new domain — build monitoring, say — means writing a new `ModelProvider`. The tool bean and existing providers don't change. The model grows; the tool surface stays at six. I think this is the property that matters most for longevity. MCP tool lists are cheap at connection time but expensive to reason about. Keeping that list short and stable while the capability behind it grows is the whole point.

## What it actually does

The concrete capability is straightforward. A coordinating agent can now:

- **See everything** — list all terminals with their slot/repo association, agent state, memory usage, errors. Read any terminal's full session log. Query the workspace model. Check which UI panel is active.
- **Control agents** — start, stop, pause, resume, graceful-shutdown. The graceful path sends Escape, tries `/exit`, waits for clean exit, force-kills if needed. All operations go through `AgentProcessManager`'s locking, so concurrent operations on the same terminal are rejected cleanly.
- **Drive lifecycle** — start work, create slots, end slots, pause/resume with coordinated agent shutdown. These delegate to `SlotAgentCoordinator`, which shuts down all agents in a slot before pausing and restarts them after resuming.
- **Navigate the UI** — `trellis_navigate` emits a `control:navigate` SSE event with a correlation ID. The frontend executes the navigation and pushes its updated state with the same correlation ID. The tool blocks until acknowledgment or timeout. The agent knows the navigation succeeded because it gets the post-navigation state back.

The `GenerationCounter` ties it together. Every state mutation — terminal create/destroy, agent state transition, UI state push — increments a monotonic counter. The agent polls `trellis_model`, checks the generation number, and knows immediately whether anything changed. No diffing required.

## The gap that surfaced

Building this exposed a gap I should have seen earlier. The `WorkspaceModelProvider` gets its data from `FileWatcherService`, which scans the filesystem. That gives you the current layout — repos, slots, epics, pauses. But soredium's `worklog.db` tracks something the filesystem can't: work lifecycle events. When a slot was created, when work started and paused, what outcomes were recorded, how the `.plan` queue has progressed. The filesystem is a snapshot; the worklog is the event log.

The control plane infrastructure makes this easy to close — a `WorklogModelProvider` reading `worklog.db` via JDBC, same SPI, model tree grows a `worklog` branch. Filed as #42. The same data needs REST endpoints too, because the dashboard and slot panels should be displaying work history, not just current state.

The other gap is frames. The agent can list terminals and navigate panels, but it can't see or manage the workspace view's frame layout — which terminals are grouped together, how they're positioned, which tab is active. That's purely frontend Dockview state right now. #43 extends `control:navigate` with frame/tab management commands, using the same SSE correlation pattern. No new MCP tools needed.

The design question these gaps clarified: the six-tool surface holds because new capabilities become either new `ModelProvider` domains (observation) or new `control:*` SSE commands routed through `trellis_navigate` (mutation). The tools are the routing layer; the model is the capability layer. As long as that separation holds, the control plane scales without the coordinating agent's tool list growing.
