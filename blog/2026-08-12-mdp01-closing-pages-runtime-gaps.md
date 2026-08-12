---
layout: post
title: "Closing the gap between pages-runtime and trellis"
date: 2026-08-12
entry_type: note
subtype: diary
projects: [trellis]
tags: [workspace, pages-runtime, floating-frames, zone-picker, organiser]
---

Three issues sat open — #44, #46, #50 — each describing work that was largely done but not wired through. The session was gap assessment, not greenfield.

**#44** (worklog bridge) turned out to be entirely complete. Issue #42 had already landed the full implementation: WorklogService, WorklogModelProvider, WorklogResource, BacklogResource delegation, 24 tests. The only gap was the schema contract — soredium's `docs/worklog.md` still said "Schema version: 1" while trellis requires version 2. I updated the doc to cover the V2 enrichment tables, the missing event types, and added a cross-language consumer section explaining how trellis reads the database. Closed.

**#46** (extract floating workspace to pages) and **#50** (pluggable frame chrome) were more interesting. The pages-runtime side was fully built — `FloatingFrameEngine`, `FloatingFrameBackend`, `wireFloatingWorkspace()`, `createFrameZonePicker`, organiser presets, pin drag lock — all there with tests. Trellis imported the engine and backend but wasn't consuming the newer features: no zone picker on frame chrome, no organiser toolbar, no detach button.

I tried the obvious approach first: import `createFrameZonePicker` from pages-runtime. This required restructuring `_initEngine()` to create the engine before `backend.attach()` — a lifecycle ordering trick, since the zone picker needs the engine reference but the extra buttons need to be passed at attach time. The engine can be created before attach because it doesn't call any backend methods until `createFrame()` is invoked later.

It compiled. It deployed. Then at runtime: `TypeError: e.onTitlebarDoubleClick is not a function`. The published Maven SNAPSHOT artifact (`casehub-pages-npm`) didn't include the newer zone picker module. I'd manually copied the files into the portal directory, but `mvn package` re-extracts the artifact and overwrites them. The "already unpacked" message in Maven's output was misleading — it means the jar hasn't changed, not that your files are safe. Cost about thirty minutes to track down because the build succeeds silently and the error surfaces at runtime in a minified stack trace.

We pivoted to inline implementations that use only the engine methods available in the published artifact. The zone picker is a 3x3 grid dropdown (↖↑↗←⊞→↙↓↘) that snaps frames to zones — built as an `extraButton` on the frame chrome. The organiser toolbar shows five preset buttons (side-by-side, stacked, grid, main-sidebar, focus) and auto-shows when two or more frames are visible. The toggle behaviour is the nice part: clicking a preset saves the current manual layout, applies the preset, and highlights the button. Clicking again restores the saved layout. If you move frames after applying a preset, the moved layout becomes the new restore target. Simple state machine — one snapshot map, one active-preset flag — but it makes the presets feel reversible rather than destructive.

The detach button (🗗) appears in Electron mode only and delegates to the existing `_detachFrame()` IPC path.

Manual event wiring stays — trellis has legitimate reasons not to use `wireFloatingWorkspace()`. Its close semantics are different (hide to attic, not remove), and it needs terminal-specific side effects on resize (fit) and on every event (save scheduling). The wiring is correct; the framework's convenience wrapper doesn't fit.

When the pages-npm artifact is next published, the full `createFrameZonePicker` with its 9-zone dropdown, titlebar double-click maximize, and resize-aware zone recomputation can replace the inline version. The inline code is a bridge, not a destination.
