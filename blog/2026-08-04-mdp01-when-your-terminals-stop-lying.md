---
title: "When Your Terminals Stop Lying"
date: 2026-08-04
entry_type: note
subtype: diary
phase: phase-update
projects: [trellis]
tags: [terminal, xterm, tmux, memory-management, pages-data-table]
status: draft
---

# When Your Terminals Stop Lying

The terminal display had been corrupting since we first wired tmux output through
WebSocket to xterm.js. Claude Code's Ink TUI would render for a moment, then text
would scatter — fragments of the status bar appearing mid-screen, cursor positions
drifting. The kind of bug where every screenshot looks different.

## The format mismatch

I'd been using `capture-pane` to replay terminal history when a WebSocket connected,
then `pipe-pane` for live output. These two produce fundamentally different data:
capture-pane gives you a screen snapshot (what the terminal looks like *right now*),
while pipe-pane gives you the raw output stream (what the application is *sending*).
Mixing them corrupts any TUI that uses cursor positioning — which is exactly what
Claude Code's Ink renderer does.

The fix was to drop capture-pane entirely. We start pipe-pane first, then force a
full screen redraw by cycling the tmux pane size — shrink one column, wait 50ms,
restore. The resize triggers SIGWINCH, which makes Ink re-render its entire screen
cleanly through the pipe. No history replay, no format mismatch, no corruption.

We also found that the FIFO reader was doing `new String(buf, 0, n)` on raw byte
chunks — splitting multi-byte UTF-8 characters at 4KB boundaries. Extracted that
into a `FifoRelay` class with `InputStreamReader` for proper charset handling.

## Memory management

The second piece was a memory management panel. When you're running a dozen Claude
agents across slots and repos, each spawning MCP servers and language servers,
memory adds up fast. I wanted visibility and control in one place.

The panel has two tables: active terminals at the top (with pause, resume, terminate),
and available repos/slots below (with bulk start). Click a row to see the process
tree — claude at 448MB, playwright-mcp at 131MB, intellij-mcp at 37MB. That
breakdown is what lets you understand where the memory actually goes.

We used `pages-data-table` for the tables instead of raw HTML — gets selection,
sorting, and efficient DOM updates for free. Had to work around a couple of
pages quirks: paginated mode fills the parent height regardless of content (filed
as casehub-pages#288), and the column settings pill feature silently replaces
custom renderers.

## Process lifecycle

The MCP server orphan problem turned out to be well-documented in Claude Code's
issue tracker. When Claude is killed, `StdioClientTransport.close()` only kills the
direct child — wrapper commands like `npx` mean the actual server is a grandchild
that becomes an orphan. Trellis's `treeKill()` already handles this correctly
(walks the process tree, kills children first), so pause kills everything. Claude
respawns fresh MCP servers on resume anyway — there's no benefit to keeping orphans alive.

We also found a race condition in `pauseAgent()`: the poll cycle could fire between
`treeKill()` and `agents.put(PAUSED)`, seeing shell + RUNNING and removing the agent
from the map. Moving the state update before the kill eliminates the window.

## What shipped

Thirty-two commits across 24 files. Terminal rendering that doesn't corrupt. A memory
panel that shows what's eating RAM and lets you do something about it. And two upstream
issues filed on casehub-pages for things the platform should handle better.
