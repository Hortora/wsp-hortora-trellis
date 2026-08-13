---
layout: post
title: "Three small things that share a branch"
date: 2026-08-13
entry_type: note
subtype: diary
projects: [trellis]
tags: [workspace-view, terminal, provenance, testing]
---

Sometimes a branch earns its name from the biggest item, but the interesting parts are elsewhere. This one batched three issues — per-frame terminal font size (#45), the repo-detail 409 fix (#30), and automated provenance testing (#21) — because none of them warranted their own branch lifecycle.

**The 409 that became an error message.** The workspace view already knew how to handle this: when `POST /api/terminals` returns 409 Conflict, the terminal exists — connect to it. One line in `_ensureTerminalExists`: `return r.status === 201 || r.status === 409`. The repo detail panel didn't have that guard. Navigate to a repo, navigate away, navigate back — red error message. The fix was literally adding `&& res.status !== 409` to one conditional. The pattern is worth naming because it's the kind of thing that survives code review. The happy path works, the sad path works, and the "already done" path looks like the sad path if you don't know the API convention.

**Font size as a frame property, not a workspace property.** The issue suggested both per-frame and per-workspace defaults. We started with per-frame only — a `_frameFontSizes` map keyed by frame ID, four presets (11, 13, 15, 18) that cycle on click. The interesting decision was where to put the control. Dockview's `extraButtons` API takes an `icon` and an `onClick` — designed for small icon buttons in the frame titlebar chrome. A dropdown would need custom DOM injection into the Dockview shadow root. A cycle button fits the existing API: click to advance, the button label shows the current size. It's less discoverable than a dropdown but zero custom layout code.

Persistence fell out naturally — `FrameLayout` already serialized position, size, pinned state, and tabs. Adding `fontSize?: number` to the interface, writing it in `_serializeLayout`, and restoring it in `createFrame` was mechanical. The `_applyFontSizeToFrame` method walks the frame's tabs, sets `terminal.options.fontSize` on each xterm instance, and calls `fit()` to reflow. Font size survives across sessions because it's part of the layout blob that goes to `/api/workspace/layout` or the Electron IPC.

**Where does provenance testing belong?** The write path is skill → engine MCP → SQLite. Trellis only reads provenance. So what can trellis test? Not the write — that's the engine's `GardenMcpToolsTest`. Not the skill invocation — that's an LLM prompt, not deterministic code. What trellis owns is the contract: does `ProvenanceRecord` deserialize correctly from the engine's JSON? Does enrichment handle null workspaces, multiple records, legacy and new GE-ID formats? Does the `specName` lifecycle work (empty on first call, populated on second)? Ten tests that validate the seam between engine and trellis, plus a tagged integration test template for post-hoc verification against a live engine.

The workspace-level default for font size is still open — this branch only landed per-frame. If demand emerges, it's a `workspaceDefaultFontSize` property that `_connectTerminal` falls back to when `_frameFontSizes` has no entry.
