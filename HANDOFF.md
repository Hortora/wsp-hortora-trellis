# Handoff — trellis

## Last Session

Replaced inline Dockview code in `workspace-view.ts` (2,400 → 873 lines) with `FloatingFrameEngine` + `DockviewBackend` from `@casehubio/pages-runtime`. All frame/tab state, z-order, spatial nav, and organisers delegated to the engine. Fixed shadow DOM CSS, position persistence, and frame chrome injection timing. Verified in running app — frames, DnD, drop zones, tab switching, persistence all work. Filed casehubio/casehub-pages#303 for engine-backend auto-wiring gaps and casehubio/parent#407 for the hierarchical MCP model protocol.

## Immediate Next Step

Continue iterating on frames. The legacy file (`workspace-view-legacy.ts`) is still in the repo for comparison — delete it when confident. The e2e Playwright tests (`e2e/workspace-dnd.spec.ts`) reference old internal APIs (`_framePositions`, `_frameTabs`, etc.) and need migrating to use `_engine.frames`. Run `/work` to continue on `issue-43-frame-tab-management`.

## Cross-Module

**Blocked by** (pages-runtime needs to ship before trellis can remove workarounds):
- `pages-runtime` — engine-backend auto-wiring: position/size sync, close/pin event handling, pluggable chrome, pin drag lock (gates casehubio/casehub-pages#303) · M · Med

## References

- Issue #43: `Hortora/trellis#43` — branch `issue-43-frame-tab-management`
- Issue #50: `Hortora/trellis#50` — pages gaps (pluggable chrome, arrangement)
- Pages #303: `casehubio/casehub-pages#303` — engine auto-wiring + chrome + tab-bar drag bug
- Parent #407: `casehubio/parent#407` — hierarchical MCP model protocol (RAGgable MCP)
- Spec: `specs/issue-43-frame-tab-management/2026-08-10-workspace-engine-refactor-design.md`
- Plan: `plans/2026-08-10-workspace-engine-refactor.md`
- Garden entries: GE-20260810-8ad59a, GE-20260810-ccd128, GE-20260810-2f9a5a (+ revised GE-20260803-756b3d)
