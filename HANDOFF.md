# Handoff — trellis

## Last Session

Closed branch `issue-43-frame-tab-management`. Deleted the legacy workspace-view (2,400 lines), migrated e2e Playwright tests from old internal maps (`_framePositions`, `_frameTabs`, `_frameActiveTab`, `_frameOrders`) to `_engine.frames` API. Fixed broken build — `workspace-view.ts` imported frame engine functions from `@casehubio/pages-runtime` package root, but they aren't re-exported from the index yet; switched to subpath imports (`dist/dockview-backend.js`, `dist/floating-frame-engine.js`, `dist/frame-boundaries.js`). Rewrote the tab-extraction e2e test to use `handleCommand` instead of mouse DnD since tab drag-out plumbing is now pages-runtime's concern. All tests green (81 unit + 4 e2e). Squashed 38 commits → 1, landed as `a41b83f` on main. Closes #43.

## Immediate Next Step

No active branch. Check `what-next` for prioritised issues, or pick from the backlog.

## Cross-Module

**Blocked by** (pages-runtime needs to ship before trellis can remove subpath import workaround):
- `pages-runtime` — re-export frame engine functions from package index (gates casehubio/casehub-pages#303) · XS · Low

## References

- Issue #43: `Hortora/trellis#43` — closed, landed as `a41b83f`
- Issue #50: `Hortora/trellis#50` — pages gaps (pluggable chrome, arrangement)
- Pages #303: `casehubio/casehub-pages#303` — engine auto-wiring + index re-exports
