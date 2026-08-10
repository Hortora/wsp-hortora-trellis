# workspace-view Engine Refactor Design

**Issue:** #43 — Frame and tab management via Agent Control Plane
**Branch:** `issue-43-frame-tab-management`
**Scope:** Replace inline Dockview code in `workspace-view.ts` with `FloatingFrameEngine` + `DockviewBackend` from `@casehubio/pages-runtime`

## Goal

Reduce `workspace-view.ts` from ~2,400 lines to ~800-1,000 by delegating frame/tab state management, z-order, spatial navigation, organisers, and Dockview lifecycle to the pages-runtime engine. Trellis keeps terminal integration, MCP command handling, picker UI, persistence bridging, and renderer tiers.

## Architecture

```
workspace-view.ts (trellis)
  │
  ├── FloatingFrameEngine (pages-runtime)
  │     ├── createFrame / removeFrame / hideFrame / showFrame
  │     ├── addTab / removeTab / moveTab / setActiveTab
  │     ├── bringToFront / togglePin / focusDirection
  │     ├── applyOrganiser / captureLayout / restoreLayout
  │     └── uses: frame-zorder, frame-spatial-nav, frame-organisers
  │
  ├── DockviewBackend (pages-runtime)
  │     ├── attach(container, contentFactory)
  │     ├── renderFrame / removeFrame / updatePosition / updateSize
  │     ├── addTab / removeTab / setActiveTab / bringToFront
  │     ├── onFrameMove / onFrameResize / onTabDragOut / onTabReorder
  │     └── injects frame chrome (close dot, pin button)
  │
  └── Trellis-specific (stays in workspace-view.ts)
        ├── ContentFactory → creates terminal elements
        ├── Terminal lifecycle (WebSocket, connect, retry)
        ├── Renderer tiers (WebGL/Canvas/None via Electron IPC)
        ├── MCP handleCommand → delegates to engine
        ├── getUIState → reads from engine.frames
        ├── Picker UI (repos, slots, groups, organisers)
        ├── Groups save/load (REST/IPC persistence)
        ├── Layout persistence (save debounce, REST/IPC bridging)
        ├── Keyboard shortcuts → dispatch to engine methods
        ├── Agent SSE state tracking
        ├── Tab hover flyout
        └── Detach/reattach (Electron IPC)
```

## Type Mapping

### TabRef (trellis domain) ↔ FrameTabConfig (pages)

```typescript
// Trellis domain type — unchanged
interface TabRef { terminalName: string; type: 'repo' | 'slot' }

// Pages type
interface FrameTabConfig { key: string; label: string; icon?: string; content: Component }

// Conversion — localised to workspace-view
function toFrameTabConfig(tab: TabRef): FrameTabConfig {
  return {
    key: tab.terminalName,
    label: tab.terminalName.replace(/^(repo-|slot-)/, ''),
    content: {} as Component,  // unused — ContentFactory provides elements
  };
}

function toTabRef(tab: FrameTabConfig): TabRef {
  const type = tab.key.startsWith('slot-') ? 'slot' : 'repo';
  return { terminalName: tab.key, type };
}
```

### Frame identity

Trellis `frameId` (UUID) maps to pages `FrameLayout.key`. Generated in workspace-view, passed to `engine.createFrame({ key: frameId, tabs, ... })`.

### Layout persistence format

Trellis serializes `ShellLayout` for IPC/REST. The bridge:

```typescript
// Save: engine → trellis format
const frames = engine.captureLayout();
const shellLayout: ShellLayout = {
  id: windowId,
  bounds: window.getBounds(),
  isMain: true,
  frames: frames.map(f => ({
    id: f.key,
    groupId: frameGroupIds.get(f.key),
    order: f.order,
    position: f.position,
    size: f.size,
    zIndex: f.zIndex,
    pinned: f.pinned,
    tabs: f.tabs.map(toTabRef),
    activeTabIndex: f.tabs.findIndex(t => t.key === f.activeTabKey),
  })),
  lastActiveFrameId: focusedFrameId,
};

// Restore: trellis format → engine
const frameConfigs = shellLayout.frames.map(f => ({
  key: f.id,
  tabs: f.tabs.map(toFrameTabConfig),
  position: f.position,
  size: f.size,
  pinned: f.pinned,
}));
engine.restoreLayout(/* FrameLayout[] built from frameConfigs */);
```

## ContentFactory

workspace-view provides a `ContentFactory` to the backend:

```typescript
const contentFactory: ContentFactory = (tab: FrameTabConfig) => {
  const termEl = document.createElement('pages-component-terminal');
  this._terminalElements.set(tab.key, termEl);
  // Terminal connection is lazy — happens on panel activation
  return {
    element: termEl,
    dispose: () => {
      this._terminalElements.delete(tab.key);
      this._activeTerminals.delete(tab.key);
      this._connectedTerminals.delete(tab.key);
    },
  };
};
```

## What Gets Deleted

### Files deleted entirely
- `workspace-zorder.ts` + `workspace-zorder.test.ts` — replaced by `frame-zorder.ts` in pages-runtime
- `workspace-spatial-nav.ts` + `workspace-spatial-nav.test.ts` — replaced by `frame-spatial-nav.ts`
- `workspace-organisers.ts` + `workspace-organisers.test.ts` — replaced by `frame-organisers.ts`

### Methods deleted from workspace-view.ts (~29 Dockview-specific methods)
- `_initDockview()` → replaced by `createDockviewBackend()` + `backend.attach()`
- `_injectFrameChrome()` → backend handles internally
- `_subscribeOverlayEvents()` → backend's `onFrameMove`/`onFrameResize` callbacks
- `_syncTabsFromDockview()` → engine tracks tab state
- `_findFloatingOverlay()` → no longer needed
- `_applyZIndex()` → engine + backend handle z-order
- `_compactZOrder()` → engine uses `normalizeForSave()` internally
- `_readOverlayBounds()` → backend events provide position/size
- `_moveFrame()` → `engine` doesn't expose this directly; backend `updatePosition` used by organiser
- `_resizeFrame()` → same pattern
- `_serializeLayout()` → replaced by `engine.captureLayout()` + trellis bridge
- Internal state maps (`_frameOrders`, `_frameZIndices`, `_frameTabs`, `_framePositions`, `_frameSizes`, `_frameGroups`, `_pinnedFrames`, `_frameActiveTab`, `_hiddenFrames`) → engine's `frames` map
- `_panelIdFromTabElement()` → no longer needed
- `_findFrameByPanels()` → no longer needed

### Methods simplified
- `createFrame()` → thin wrapper around `engine.createFrame()`
- `hideFrame()` → delegates to `engine.hideFrame()`
- `showFrame()` → delegates to `engine.showFrame()`
- `deleteFrame()` → delegates to `engine.removeFrame()`
- `togglePin()` → delegates to `engine.togglePin()`
- `bringToFront()` → delegates to `engine.bringToFront()`
- `applyOrganiser()` → delegates to `engine.applyOrganiser()`

### Methods remapped
- `handleCommand()` — switch cases delegate to engine methods for state operations (createFrame, removeFrame, togglePin, addTab, removeTab, applyOrganiser). For frame-move and frame-resize, call `backend.updatePosition/updateSize` directly — the engine doesn't expose imperative move/resize (it reacts to backend events via callbacks).
- `getUIState()` — reads from `engine.frames` instead of internal maps
- `_handleKeydown()` — spatial nav dispatches to `engine.focusDirection()`, tab nav reads from engine state
- `_nextTab()` / `_prevTab()` / `_jumpToTab()` — use `engine.setActiveTab()` instead of `_frameActiveTab` map

## What Stays (file retained)
- `workspace-renderer-tiers.ts` + test — trellis-specific, no pages equivalent

## What Stays (in workspace-view.ts)
- ContentFactory implementation
- `_connectTerminal()`, `_ensureTerminalExists()` — terminal WebSocket lifecycle
- `_setupAgentSSE()`, `_handleAgentStateEvent()` — agent state tracking
- `_showTabFlyout()`, `_hideTabFlyout()`, `_populateFlyout()` — tab hover preview
- `_fetchWorkspace()` — workspace API integration
- `_setupKeyboard()`, `_handleKeydown()` — keyboard handler (simplified, delegates to engine)
- `_setupShortcutIPC()`, `_setupDetachIPC()`, `_setupFlushHandler()`, `_setupWebglIPC()` — Electron IPC
- `_detachFrame()`, `_attachToMainWindow()` — cross-window frame transfer
- `_showPicker()`, `_dismissPicker()`, `_showOrganiserPicker()`, `_showFramesList()` — picker UI
- `_showFrameContextMenu()`, `_promptSaveAsGroup()` — context menu
- `_saveFrameAsGroup()`, `_updateGroup()`, `_deleteGroup()`, `_deleteGroupById()` — group CRUD
- `_loadGroupsData()`, `_saveGroupsData()` — persistence bridging
- `_doSave()`, `_scheduleSave()` — layout save debounce (now calls `engine.captureLayout()`)
- `_restoreLayout()` — reads persisted layout, calls `engine.restoreLayout()`
- `handleCommand()` — MCP command dispatch (remapped to engine)
- `getUIState()` — agent control plane observation
- `focusFrame()`, `focusTab()` — navigation targets
- `isTerminalOpen()` — active terminal check
- `loadGroups()` — group data access
- `_updateRendererTiers()`, `_acquireWebgl()`, `_releaseWebgl()`, `_applyRendererTier()` — renderer management
- `_onNewFrame()`, `_onNewTab()`, `_onCloseTab()` — toolbar/shortcut handlers
- `_addTab()`, `_removeTab()` — terminal-aware tab lifecycle (ensures terminal exists, connects WebSocket)
- `nextFramePosition()`, `clampPosition()` — exported utilities (used by createFrame positioning)

## Initialization Flow

```
firstUpdated()
  ├── createDockviewBackend()          → async, lazy-loads dockview-core
  ├── backend.attach(container, contentFactory)
  ├── createFloatingFrameEngine(backend, savedLayout)
  ├── backend.onFrameMove(cb)          → update trellis position cache, scheduleSave
  ├── backend.onFrameResize(cb)        → update trellis size cache, fit terminals, scheduleSave
  ├── backend.onTabDragOut(cb)         → create new frame from dragged tab
  ├── backend.onTabReorder(cb)         → scheduleSave
  ├── _setupKeyboard()
  ├── _setupAgentSSE()
  ├── _restoreLayout()                 → engine.restoreLayout(saved)
  └── (Electron only) _setupShortcutIPC(), _setupDetachIPC(), etc.
```

## Test Strategy

Replace the 100-line `DockviewComponent` mock with a simple `FloatingFrameBackend` mock:

```typescript
function createMockBackend(): FloatingFrameBackend {
  return {
    attach: vi.fn(),
    detach: vi.fn(),
    renderFrame: vi.fn(),
    removeFrame: vi.fn(),
    updatePosition: vi.fn(),
    updateSize: vi.fn(),
    bringToFront: vi.fn(),
    addTab: vi.fn(),
    removeTab: vi.fn(),
    setActiveTab: vi.fn(),
    onFrameMove: vi.fn(),
    onFrameResize: vi.fn(),
    onTabDragOut: vi.fn(),
    onTabReorder: vi.fn(),
    dispose: vi.fn(),
    unwrap: () => null,
  };
}
```

Tests that verified Dockview-specific behavior (overlay.setBounds, panel ordering, floating group internals) are replaced by tests that verify the engine's public API is called correctly. The engine itself is tested in pages-runtime.

### Test categories after refactor
- **Keep as-is:** picker UI, flyout data, REST persistence, keyboard shortcuts, renderer tiers, detach/reattach, groups CRUD, groups in picker, getUIState shape, handleCommand validation
- **Rewrite:** z-order (verify via engine.frames instead of internal maps), pinning (same), tab navigation (same), organiser integration (same), layout restore (use engine.restoreLayout), frame chrome (backend handles, verify events)
- **Delete:** CSS injection tests for dockview theme (backend handles), DockviewComponent constructor option tests, overlay.setBounds tests

## Dependencies

- `dockview-core` moves from trellis `package.json` direct dependency to transitive through `@casehubio/pages-runtime`. Remove from trellis `package.json`.
- `@casehubio/pages-runtime` already in casehub-packages portal resolution — updated this session with floating-frame files.
- `@casehubio/pages-component` already in casehub-packages — updated with `FrameLayout`, `FrameTabConfig`, `ContentFactory` types.

## Risk

**Low.** The engine and backend were extracted from this exact file. The API surface maps 1:1. The main risk is persistence format compatibility — the bridge code must preserve backward compatibility with existing saved layouts (position, size, z-index, tab order, pinned state, groups). A round-trip test verifies this.
