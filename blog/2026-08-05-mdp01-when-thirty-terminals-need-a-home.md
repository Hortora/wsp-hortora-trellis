---
layout: post
title: "When Thirty Terminals Need a Home"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [trellis]
tags: [architecture, dockview, xterm, electron, multi-window, workspace]
series: trellis-build
---

*Part of a series on [#28 — Workspace view](https://github.com/Hortora/trellis/issues/28). Previous: [When Your Terminals Stop Lying](2026-08-04-mdp01-when-your-terminals-stop-lying.md).*

Trellis has been running against CaseHub's ~30 repos for about a week now. The single-panel workbench — one terminal visible at a time, click the dock bar to switch — works fine for a handful of repos. It falls apart when you're running agents across seven or eight related repos simultaneously, watching one finish while typing in another, and trying to remember which tab had the engine repo and which had the ledger.

The workflow I actually use looks nothing like what Trellis offered. I position overlapping groups of terminals on screen so I can glance at one while working in another. When an agent finishes in the background, I click that window to the front, check the output, and switch back. The existing single-panel model forced me to navigate linearly through something that's inherently spatial.

## The frame model

The core idea: floating rectangles within the workspace panel, each containing tabbed terminals. I'm calling them "frames" to avoid overloading "window" (which already means the Electron BrowserWindow) and "panel" (which means a dock-bar destination like Garden or Artifacts).

The terminology took some thought. The hierarchy is Workbench → Panel → Frame → Tab — four levels, no overloading. Each frame is a Dockview floating group. Each tab references a terminal by name and connects via WebSocket to the existing tmux backend. The existing org-dashboard (repo cards, slot status, epic progress) moves to its own "Dashboard" dock-bar panel. The workspace panel becomes the frame canvas.

I wanted free-form positioning as the default. No snapping, no magnetic edges, no tiling constraints. Organisers exist as one-shot arrangers — apply "Grid" or "Side by side" and you're immediately back in free mode. Snap is opt-in via Shift during drag, so it never surprises you. I've been burned too many times by window managers that helpfully snap something to an edge I didn't want.

## Dockview and the WebGL problem

Claude did a web research sweep on in-browser MDI window managers. The findings narrowed the field quickly: GoldenLayout is dead, WinBox has no tabs, Lumino (JupyterLab) is more framework than library. Dockview stood out — vanilla TypeScript, zero dependencies, MIT license, and the Cate IDE project already runs xterm.js with WebGL inside Dockview floating panels. That's the exact integration path we needed.

The harder constraint isn't the library — it's the GPU. Chrome and Electron allow 16 WebGL contexts per GPU process. With 30+ repos, each needing a terminal, you blow through that limit immediately. The solution is three-tier renderer management: WebGL for the focused terminal (full GPU acceleration), Canvas for visible-but-inactive terminals (CPU-rendered, lighter), and no renderer at all for background tabs (renderer disposed, Terminal buffer kept alive so scrollback accumulates). The WebGL budget is coordinated across Electron windows via IPC — a main-process `acquire/release/demote` protocol with a pending queue and LRU demotion.

The xterm.js gotchas were well-documented in the research: never call `fit()` during drag (causes width collapse to 1px), use `transform: translate()` instead of `top/left` for movement (GPU-composited, no layout thrash), and block `pointer-events` on all terminal containers during frame drag (otherwise the terminal steals the drag events). These aren't in the xterm.js docs — they come from years of GitHub issues.

## The adversarial review

The spec went through a full adversarial design review — three independent dimensions (coherence, structure, robustness) running in parallel, each doing 3-4 rounds. 51 issues surfaced in total. Every one was resolved.

The findings that most improved the design: `Ctrl+[` produces ESC (ASCII 27), which is essential for vim and tmux — all the navigation shortcuts had to move from Ctrl-based to Cmd+Opt-based modifiers. The shutdown save protocol needed a flush-before-quit sequence because the existing `before-quit` handler called `closeAll()` before any layout save could fire, destroying window tracking. And the persistence model needed splitting into three independent electron-store keys (groups, layout, keymap) to prevent layout auto-saves from corrupting group data.

The robustness review caught a pipe-pane takeover bug that was already latent in the existing code: if two WebSocket connections opened for the same tmux session, `tmux pipe-pane` would redirect output to the second connection's FIFO, silently killing the first. The fix is a session-keyed `activeBySession` map in `TerminalWebSocket` that closes the previous connection with close code 4001 ("session-takeover") before setting up the new pipe.

## What it looks like now

The implementation spans all three layers — Java sidecar (readiness endpoint, terminal creation atomicity, session takeover), Electron shell (LayoutStore with typed methods, WebGL IPC, menu accelerators, shutdown protocol), and TypeScript frontend (Dockview frame manager, terminal lifecycle, keyboard navigation, organisers, persistence).

Keyboard navigation covers the full hierarchy: `Cmd+Shift+[/]` cycles tabs within a frame, `Cmd+Opt+[/]` cycles frames (with spatial arrow-key navigation for directional movement), and `Cmd+Ctrl+[/]` cycles between browser windows. Numbered jumps (`Cmd+1-9` for tabs, `Cmd+Opt+1-9` for frames) give direct access. The tab hover flyout surfaces metadata on demand — branch, path, agent state, memory, last few lines of output — so you never lose track of what a terminal is doing.

Frames can be pinned (always-on-top and position-locked — useful for monitoring a running agent) or detached to separate Electron windows and brought back. The full layout persists across restarts, including which terminals were connected.

The agent control plane (#27) is filed as a separate issue. That's the next layer: letting a Trellis-level coordinating agent navigate frames, query terminal state, and send input — programmatically, not at the DOM level. The workspace view gives it the frame/tab model to navigate. The question now is whether 6-8 MCP tools with a rich data model can cover the full surface, or whether the interaction patterns will force something more granular.
