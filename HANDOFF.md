# Handoff — trellis

## Last Session

Worked on #43 frame/tab management bugs. Fixed 7 of the original 7 bugs (terminal connect retry, lazy tab init, position persistence, garbled text, z-order, organiser sizes, scan-root MCP). Then moved into Dockview DnD integration work — significant uncommitted changes in workspace-view.ts (~450 lines of diff). Multiple issues remain unfixed despite extensive iteration.

## Immediate Next Step

The uncommitted diff has two unresolved Dockview integration problems that need first-principles debugging. **Do not iterate on symptoms — read the Dockview v7 source code for floating group DnD behaviour before writing any code.**

### Problem 1: Drag tab to empty space → create new floating frame

**Current approach (partially working):** `dragstart`/`dragend` event listeners on shadow root capture the panel ID, then `setTimeout(50)` calls `dv.addFloatingGroup(panel, {x,y})` to move the panel to a new floating group. `_syncTabsFromDockview()` detects new groups and registers them.

**What works:** First drag-out creates a frame correctly.

**What fails:** After dragging a tab out, subsequent tabs in the same frame cannot be dragged. Root cause: calling `dv.addFloatingGroup()` inside the `dragend` handler (even deferred by 50ms) corrupts Dockview's internal drag state for the source group. The remaining tabs lose their drag handles.

**What to investigate:** Read Dockview's `addFloatingGroup` source — does it lock the source group? Does it need a longer delay? Is there a Dockview event (`onDidRemovePanel`, `onDidAddGroup`) that fires AFTER the drag cleanup is fully complete? Or should we abandon the `dragend` approach entirely and use a different mechanism (e.g., right-click context menu "Move to new frame" which calls `_removeTab` + `createFrame` and works reliably)?

### Problem 2: Tab-bar drop zone indicators showing no-op positions

When dragging tab A in [A, B, C], Dockview shows drop indicators at ALL positions. Dropping at positions adjacent to A's original position doesn't change order — these are no-op indicators.

**Current approach:** MutationObserver watches for `.dv-drop-target-dropzone` elements and hides those at positions `idx` and `idx+1` (where `idx` is the dragged tab's index). Uses `getBoundingClientRect()` to determine which position a zone represents.

**What works:** 2-tab frames.

**What fails:** 3+ tab frames — intermittent. The MutationObserver fires but sometimes misidentifies positions. Real Dockview creates ONE zone at a time (not N+1 simultaneously), and the zone can appear in `.dv-tabs-container` or `.dv-void-container` (the empty space after tabs).

**What to investigate:** Whether Dockview's `dropOverlayModel` option can shape the tab drop zones. Whether there's a per-group hook to filter zones. Whether CSS `:has()` selectors can detect the adjacent-to-dragged condition without JavaScript.

### Problem 3: Refresh persistence

**Current approach:** `beforeunload` handler calls `_saveBeforeUnload()` with sync XHR. Debounced save every 500ms via `_doSave()`.

**What works:** Save path works — `_syncFrameBoundsFromDockview()` reads overlay positions correctly. Restore path works — `_restoreLayout()` applies saved positions to `addPanel({floating: {x,y}})`.

**What fails in dev mode:** `_saveBeforeUnload()` was overwriting good debounced saves with stale data (Dockview overlays return zeros during page teardown). Fixed by only saving if `hasPendingSave` (debounce timer is pending). But Quarkus dev mode hot-reloads the frontend on TS changes, which may still race. In production (Electron), sync IPC saves work reliably.

## Key Files

- `sidecar/src/main/webui/src/components/workspace-view.ts` — all frame/tab logic, 2100+ lines
- `sidecar/src/main/webui/src/components/workspace-view.test.ts` — 198 unit tests
- `sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java` — scan-root operation
- `docs/specs/issue-43-frame-tab-management/2026-08-06-frame-tab-management-design.md` — spec

## References

- Issue #43: Frame and tab management via Agent Control Plane
- Issue #45: Per-frame/workspace font size selection (logged, not started)
- Dockview v7 docs: `dockview-core` in `node_modules/dockview-core/dist/esm/`
