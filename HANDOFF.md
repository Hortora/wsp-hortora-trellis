# Handoff — trellis

## Last Session

Fixed all #43 DnD integration bugs: position persistence (Dockview clamps to zero-sized container in `firstUpdated` — fixed with double `requestAnimationFrame`), tab order (`.sort()` mutation), active tab tracking, drag-out, and drop zones (replaced with Dockview smooth reorder). Removed 22 fake mock tests, added 4 Playwright e2e tests against real Dockview. Agreed to port the floating workspace code to `casehub-pages` as a reusable component.

## Immediate Next Step

Open a GitHub issue on `mdproctor/casehub-pages` for `pages-floating-workspace`, create a slot, and begin porting the Dockview workspace code from `workspace-view.ts`. The pages repo is at `/Users/mdproctor/claude/casehub/pages`.

## Key Files

- `sidecar/src/main/webui/src/components/workspace-view.ts` — 2400 lines, contains all code to port
- `sidecar/src/main/webui/e2e/workspace-dnd.spec.ts` — 4 Playwright e2e tests (port these too)
- `sidecar/src/main/webui/esbuild.config.mjs` — content-hashed JS filenames
