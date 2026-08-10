# Handoff — trellis

## Last Session

Slot 5 (issue-46-pages-floating-workspace) completed and merged to main in casehub-pages. The new pages component provides a workbench with docks, panels, draggable toolbar buttons for dock positioning, configurable drag (can be disabled), free-layout floating frames, and draggable tabs — all demonstrated in pages examples. Trellis #43 branch has all DnD fixes committed but trellis has NOT yet been updated to consume the new pages component.

## Immediate Next Step

Update trellis to consume the new `pages-floating-workspace` component from casehub-pages instead of the inline Dockview code in `workspace-view.ts`. This means:

1. **Update casehub-packages** — pull the latest pages SNAPSHOT into `sidecar/src/main/webui/.casehub-packages/` so the new component is available via portal resolutions
2. **Replace workspace-view.ts internals** — the 2400-line file currently contains inline Dockview management, position persistence, tab reorder, drag-out, frame chrome, and drop zone handling. All of this is now in the pages component. Replace with a thin wrapper that:
   - Creates a `pages-floating-workspace` element
   - Passes a content factory (terminal panels via `createComponent`)
   - Handles terminal-specific concerns (WebSocket, xterm, agent SSE, renderer tiers)
   - Delegates frame/tab CRUD to the pages component's API
   - Keeps MCP command handler (`handleCommand`) and `getUIState`
3. **Update Playwright e2e tests** — the 4 tests in `e2e/workspace-dnd.spec.ts` should still pass against the pages component
4. **Remove Dockview direct dependency** — `dockview-core` moves from trellis's `package.json` to pages's (it's already there from the port)

## What the pages component provides

The new component (built in slot 5) handles:
- Dockview lifecycle and floating group management
- Position/size persistence via pluggable store
- Smooth iTerm2-style tab reorder (`tabAnimation: 'smooth'`)
- Tab drag-out to new frame (mouse-outside-frame boundary check)
- Event-driven position tracking (`overlay.onDidChangeEnd`)
- Double `requestAnimationFrame` before restore (container sizing)
- Frame chrome (close dot, pin, context menu)
- Configurable docks with draggable toolbar buttons
- Dock drag can be disabled per-dock

## What stays in trellis workspace-view.ts

- Terminal integration (WebSocket, xterm, connect/retry, renderer tiers)
- Picker UI (repos, slots, groups, organisers)
- MCP command handler (`handleCommand` switch)
- `getUIState` for agent control plane
- Agent SSE state tracking
- Keyboard shortcuts specific to trellis
- Flyout (tab hover preview)
- Detach/reattach (Electron IPC)

## Key constraints

- No happy-dom mocks for Dockview behavior — Playwright e2e only
- Instrument (console.log) before fixing — observe the failure, don't theorize
- One fix → one user verification → course correct
- Content-hashed JS filenames (`esbuild entryNames: [name]-[hash]`) prevent stale cache

## References

- Issue #43 (DnD bugs): `Hortora/trellis#43` — branch `issue-43-frame-tab-management`
- Issue #46 (pages port): `Hortora/trellis#46` — slot 5 complete, merged to pages main
- Pages repo: `/Users/mdproctor/claude/casehub/pages`
- Slot 5 path: `/Users/mdproctor/claude/hortora/slots/5/`
- Garden entries: GE-20260809-6bede4, GE-20260809-44b2a6, GE-20260809-f0c43a, GE-20260809-6821a6
