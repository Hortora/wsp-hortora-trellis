# workspace-view Engine Refactor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #43 — Frame and tab management via Agent Control Plane
**Issue group:** #43

**Goal:** Replace inline Dockview code in `workspace-view.ts` (~2,400 lines) with `FloatingFrameEngine` + `DockviewBackend` from `@casehubio/pages-runtime`, reducing the file to ~800-1,000 lines.

**Architecture:** workspace-view delegates all frame/tab state management, z-order, spatial navigation, and organisers to the pages-runtime engine. Trellis keeps terminal integration, MCP command handling, picker UI, persistence bridging, renderer tiers, and Electron IPC. A type bridge converts between trellis's `TabRef` and the engine's `FrameTabConfig` at the boundary.

**Tech Stack:** TypeScript / Lit 3.x, `@casehubio/pages-runtime` (FloatingFrameEngine, DockviewBackend), vitest, Playwright

## Global Constraints

- No new dependencies — pages-runtime already in casehub-packages via portal resolution
- dockview-core becomes a transitive dependency through pages-runtime (removed from trellis package.json)
- Pre-release platform — breaking changes to internal APIs are fine
- No happy-dom mocks for Dockview behavior — Playwright e2e only for layout/DnD
- `terminalName` (not positional index) for tab identification in all interfaces
- All existing Playwright e2e tests must continue to pass

## Spec

`specs/issue-43-frame-tab-management/2026-08-10-workspace-engine-refactor-design.md`

---

### Task 1: Type bridge + engine initialization (additive)

**Files:**
- Modify: `sidecar/src/main/webui/src/components/workspace-view.ts`
- Test: `sidecar/src/main/webui/src/components/workspace-view.test.ts`

**Interfaces:**
- Consumes: `createFloatingFrameEngine` from `@casehubio/pages-runtime`, `createDockviewBackend` from `@casehubio/pages-runtime/dockview-backend`, `FloatingFrameBackend` from `@casehubio/pages-runtime/floating-frame-backend`, `FrameTabConfig`, `FrameLayout as PagesFrameLayout`, `ContentFactory` from `@casehubio/pages-component`
- Produces: `toFrameTabConfig(tab: TabRef): FrameTabConfig`, `toTabRef(tab: FrameTabConfig): TabRef` — module-level conversion functions. `_engine: FloatingFrameEngine`, `_backend: FloatingFrameBackend` — instance fields on `TrellisWorkspaceView`.

This task is purely additive — engine and backend are initialised alongside the existing Dockview code. No behavior changes. All existing tests pass unchanged.

- [ ] **Step 1: Write failing test for type bridge functions**

Add a new `describe` block at the end of `workspace-view.test.ts`:

```typescript
describe('type bridge', () => {
  it('converts TabRef to FrameTabConfig', () => {
    const { toFrameTabConfig } = await import('./workspace-view.js');
    const tab: any = { terminalName: 'repo-engine', type: 'repo' };
    const config = toFrameTabConfig(tab);
    expect(config.key).toBe('repo-engine');
    expect(config.label).toBe('engine');
  });

  it('converts slot TabRef with correct label', () => {
    const { toFrameTabConfig } = await import('./workspace-view.js');
    const tab: any = { terminalName: 'slot-3', type: 'slot' };
    const config = toFrameTabConfig(tab);
    expect(config.key).toBe('slot-3');
    expect(config.label).toBe('3');
  });

  it('converts FrameTabConfig back to TabRef', () => {
    const { toTabRef } = await import('./workspace-view.js');
    const config: any = { key: 'repo-engine', label: 'engine', content: {} };
    const tab = toTabRef(config);
    expect(tab.terminalName).toBe('repo-engine');
    expect(tab.type).toBe('repo');
  });

  it('infers slot type from key prefix', () => {
    const { toTabRef } = await import('./workspace-view.js');
    const config: any = { key: 'slot-3', label: '3', content: {} };
    const tab = toTabRef(config);
    expect(tab.type).toBe('slot');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd sidecar/src/main/webui test -- --run workspace-view`
Expected: FAIL — `toFrameTabConfig` is not exported

- [ ] **Step 3: Implement type bridge functions**

Add to `workspace-view.ts` after the `clampPosition` function (before the `TabRef` interface):

```typescript
import type { FrameTabConfig, FrameLayout as PagesFrameLayout, ContentFactory, ContentFactoryResult } from '@casehubio/pages-component';
import type { FloatingFrameBackend } from '@casehubio/pages-runtime/floating-frame-backend';
import type { FloatingFrameEngine } from '@casehubio/pages-runtime/floating-frame-engine';

export function toFrameTabConfig(tab: TabRef): FrameTabConfig {
  return {
    key: tab.terminalName,
    label: tab.terminalName.replace(/^(repo-|slot-)/, ''),
    content: {} as any,
  };
}

export function toTabRef(tab: FrameTabConfig): TabRef {
  const type: 'repo' | 'slot' = tab.key.startsWith('slot-') ? 'slot' : 'repo';
  return { terminalName: tab.key, type };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn --cwd sidecar/src/main/webui test -- --run workspace-view`
Expected: PASS (type bridge tests pass, all existing tests still pass)

- [ ] **Step 5: Add engine + backend fields to class**

Add to `TrellisWorkspaceView` field declarations (after `_restoring`):

```typescript
private _engine: FloatingFrameEngine | null = null;
private _backend: FloatingFrameBackend | null = null;
```

- [ ] **Step 6: Verify build passes**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: No errors (warnings about tsconfig bases are pre-existing and fine)

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/webui/src/components/workspace-view.ts sidecar/src/main/webui/src/components/workspace-view.test.ts
git commit -m "feat(#43): type bridge + engine fields — additive prep for engine migration

Refs #43"
```

---

### Task 2: Core migration — replace Dockview internals with engine delegation

**Files:**
- Modify: `sidecar/src/main/webui/src/components/workspace-view.ts`
- Modify: `sidecar/src/main/webui/src/components/workspace-view.test.ts`

**Interfaces:**
- Consumes: `toFrameTabConfig`, `toTabRef` (Task 1), `createFloatingFrameEngine`, `FloatingFrameBackend`, `createDockviewBackend` (pages-runtime)
- Produces: All existing public methods preserved with same signatures — `createFrame()`, `hideFrame()`, `showFrame()`, `deleteFrame()`, `togglePin()`, `bringToFront()`, `applyOrganiser()`, `handleCommand()`, `getUIState()`, `focusFrame()`, `focusTab()`, `isTerminalOpen()`, `loadGroups()`. Internal state now comes from `this._engine.frames` instead of individual maps.

This is the core rewrite. The internal state maps (`_frameOrders`, `_frameTabs`, `_frameZIndices`, `_pinnedFrames`, `_framePositions`, `_frameSizes`, `_frameGroups`, `_groupToFrame`, `_frameActiveTab`, `_hiddenFrames`) are removed. All frame/tab state is read from `this._engine.frames`. The DockviewComponent mock in tests is replaced with a mock `FloatingFrameBackend`.

The state maps that **stay** are: `_activeTerminals`, `_connectedTerminals`, `_terminalElements` (Map of terminal name → element), `_frameGroupIds` (trellis group associations), `_focusedFrameId`, `_agentStates`, `_rendererTiers`, `_rendererAddons`, `_lastCommandResult`, save debounce fields, flyout fields, SSE fields, `_browserMode`.

- [ ] **Step 1: Replace the DockviewComponent mock with a mock FloatingFrameBackend factory**

Replace the entire `vi.mock('dockview-core', ...)` block (lines 5-98 of workspace-view.test.ts) with:

```typescript
import type { FloatingFrameBackend } from '@casehubio/pages-runtime/floating-frame-backend';

function createMockBackend(): FloatingFrameBackend & { _rendered: Map<string, any> } {
  const rendered = new Map<string, any>();
  return {
    _rendered: rendered,
    attach: vi.fn(),
    detach: vi.fn(),
    renderFrame: vi.fn((layout: any) => { rendered.set(layout.key, layout); }),
    removeFrame: vi.fn((key: string) => { rendered.delete(key); }),
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

// Inject mock backend factory into workspace-view before import
vi.mock('@casehubio/pages-runtime/dockview-backend', () => ({
  createDockviewBackend: vi.fn(() => Promise.resolve(createMockBackend())),
}));
```

- [ ] **Step 2: Rewrite _initDockview to use engine + backend**

Replace the existing `_initDockview()` method and the `firstUpdated()` Dockview initialization with:

```typescript
private async _initEngine() {
  const { createDockviewBackend } = await import('@casehubio/pages-runtime/dockview-backend');
  const { createFloatingFrameEngine } = await import('@casehubio/pages-runtime/floating-frame-engine');

  this._backend = await createDockviewBackend();

  const contentFactory: ContentFactory = (tab) => {
    const termEl = document.createElement('pages-component-terminal') as any;
    this._terminalElements.set(tab.key, termEl);
    return {
      element: termEl,
      dispose: () => {
        this._terminalElements.delete(tab.key);
        this._activeTerminals.delete(tab.key);
        this._connectedTerminals.delete(tab.key);
      },
    };
  };

  this._backend.attach(this._container!, contentFactory);

  this._backend.onFrameMove((key, pos) => {
    this._scheduleSave();
  });
  this._backend.onFrameResize((key, size) => {
    this._fitTerminalsInFrame(key);
    this._scheduleSave();
  });
  this._backend.onTabDragOut((fromFrame, tabKey, position) => {
    this._engine?.removeTab(fromFrame, tabKey);
    const tab = toTabRef({ key: tabKey, label: tabKey.replace(/^(repo-|slot-)/, ''), content: {} as any });
    this.createFrame([tab], undefined, undefined, { position });
  });
  this._backend.onTabReorder((_frameKey, _tabKeys) => {
    this._scheduleSave();
  });

  this._engine = createFloatingFrameEngine(this._backend);
}
```

- [ ] **Step 3: Rewrite createFrame to delegate to engine**

Replace the existing `createFrame` method body. The signature stays the same:

```typescript
createFrame(
  tabs: TabRef[],
  groupId?: string,
  name?: string,
  restore?: Partial<FrameLayout>,
): string {
  if (tabs.length === 0 && !restore) return '';

  const frameId = restore?.id ?? crypto.randomUUID();
  if (!this._engine) return '';

  const containerWidth = this._container?.clientWidth ?? 1200;
  const containerHeight = this._container?.clientHeight ?? 800;
  const fWidth = restore?.size?.width ?? 600;
  const fHeight = restore?.size?.height ?? 400;

  const existingPositions = [...this._engine.frames.values()]
    .filter(f => !f.hidden)
    .map(f => f.position);

  const position = restore?.position
    ? clampPosition(restore.position, { width: fWidth, height: fHeight },
        { width: containerWidth, height: containerHeight })
    : nextFramePosition(
        { width: containerWidth, height: containerHeight },
        { width: fWidth, height: fHeight },
        existingPositions,
      );

  const frameTabConfigs = tabs.map(toFrameTabConfig);
  this._engine.createFrame({
    key: frameId,
    tabs: frameTabConfigs,
    position,
    size: { width: fWidth, height: fHeight },
    pinned: restore?.pinned ?? false,
  });

  for (const tab of tabs) {
    this._activeTerminals.add(tab.terminalName);
  }

  if (groupId) this._frameGroupIds.set(frameId, groupId);
  if (!this._restoring) this._focusedFrameId = frameId;
  this._scheduleSave();
  return frameId;
}
```

- [ ] **Step 4: Rewrite hideFrame, showFrame, deleteFrame, togglePin, bringToFront**

```typescript
hideFrame(frameId: string) {
  if (!this._engine) return;
  const frame = this._engine.frames.get(frameId);
  if (!frame) return;
  this._engine.hideFrame(frameId);
  for (const tab of frame.tabs) {
    this._activeTerminals.delete(tab.key);
  }
  this._frameGroupIds.delete(frameId);
  if (this._focusedFrameId === frameId) this._focusedFrameId = null;
  this._scheduleSave();
}

showFrame(frameId: string) {
  if (!this._engine) return;
  this._engine.showFrame(frameId);
  const frame = this._engine.frames.get(frameId);
  if (frame) {
    for (const tab of frame.tabs) {
      this._activeTerminals.add(tab.key);
    }
  }
  this._focusedFrameId = frameId;
  this._scheduleSave();
}

deleteFrame(frameId: string) {
  if (!this._engine) return;
  const frame = this._engine.frames.get(frameId);
  if (frame) {
    for (const tab of frame.tabs) {
      this._activeTerminals.delete(tab.key);
    }
  }
  this._engine.removeFrame(frameId);
  this._frameGroupIds.delete(frameId);
  if (this._focusedFrameId === frameId) this._focusedFrameId = null;
  this._scheduleSave();
}

togglePin(frameId: string) {
  if (!this._engine) return;
  this._engine.togglePin(frameId);
  this._scheduleSave();
}

bringToFront(frameId: string) {
  if (!this._engine) return;
  this._engine.bringToFront(frameId);
  this._focusedFrameId = frameId;
}
```

- [ ] **Step 5: Rewrite applyOrganiser to delegate to engine**

```typescript
applyOrganiser(presetName: string) {
  if (!this._engine) return;
  const preset = presetName.toLowerCase().replace(/\s+/g, '-') as any;
  const validPresets = ['side-by-side', 'stacked', 'grid', 'main-sidebar', 'focus'];
  if (!validPresets.includes(preset)) return;

  const containerWidth = this._container?.clientWidth ?? 1200;
  const containerHeight = this._container?.clientHeight ?? 800;
  this._engine.applyOrganiser(preset, { width: containerWidth, height: containerHeight });
  this._scheduleSave();
}
```

- [ ] **Step 6: Rewrite _serializeLayout + _restoreLayout to use engine**

```typescript
private _serializeLayout(): ShellLayout {
  if (!this._engine) return { id: 'shell-1', bounds: { x: 0, y: 0, width: 1200, height: 800 }, isMain: true, frames: [], lastActiveFrameId: undefined };

  const engineFrames = this._engine.captureLayout();
  return {
    id: 'shell-1',
    bounds: { x: 0, y: 0, width: this._container?.clientWidth ?? 1200, height: this._container?.clientHeight ?? 800 },
    isMain: true,
    frames: engineFrames.map(f => ({
      id: f.key,
      groupId: this._frameGroupIds.get(f.key),
      order: f.order,
      position: f.position,
      size: f.size,
      zIndex: f.zIndex,
      pinned: f.pinned,
      tabs: f.tabs.map(toTabRef),
      activeTabIndex: f.tabs.findIndex(t => t.key === f.activeTabKey),
    })),
    lastActiveFrameId: this._focusedFrameId ?? undefined,
  };
}

private async _restoreLayout() {
  // Preserve existing IPC/REST load path unchanged — only the frame
  // creation loop below replaces the old per-frame createFrame() calls.
  // The variable `layout` comes from the existing load logic:
  //   Electron: window.trellis.loadLayout(path)
  //   Browser:  fetch(`/api/workspace/layout?root=${root}`)
  if (!this._engine || !layout?.windows?.[0]?.frames) return;
  this._restoring = true;

  const pagesFrames: PagesFrameLayout[] = layout.windows[0].frames.map((f: FrameLayout) => ({
    key: f.id,
    order: f.order,
    position: f.position,
    size: f.size,
    zIndex: f.zIndex,
    pinned: f.pinned,
    hidden: false,
    tabs: f.tabs.map(toFrameTabConfig),
    activeTabKey: f.tabs[f.activeTabIndex]?.terminalName ?? f.tabs[0]?.terminalName ?? '',
  }));

  this._engine.restoreLayout(pagesFrames);

  for (const f of layout.windows[0].frames) {
    for (const tab of f.tabs) {
      this._activeTerminals.add(tab.terminalName);
    }
    if (f.groupId) this._frameGroupIds.set(f.id, f.groupId);
  }

  this._focusedFrameId = layout.windows[0].lastActiveFrameId ?? null;
  this._restoring = false;
}
```

- [ ] **Step 7: Rewrite handleCommand to delegate to engine/backend**

Replace the `handleCommand` method. Key changes:
- `frame-create` → `this.createFrame(params.tabs, ...)`
- `frame-remove` → `this.hideFrame()` + `this.deleteFrame()`
- `frame-move` → `this._backend.updatePosition()` directly
- `frame-resize` → `this._backend.updateSize()` directly
- `frame-pin/unpin` → `this._engine.togglePin()`
- `tab-add` → `this._engine.addTab()` + `this._activeTerminals.add()`
- `tab-remove` → `this._engine.removeTab()` + `this._activeTerminals.delete()`
- `organiser-apply` → `this.applyOrganiser()`
- Frame existence check: `this._engine.frames.has(frameId)` instead of `this._frameOrders.has(frameId)`

```typescript
async handleCommand(command: string, params?: any): Promise<{ ok: boolean; error?: string; frameId?: string }> {
  let result: { ok: boolean; error?: string; frameId?: string };

  switch (command) {
    case 'frame-create': {
      const id = this.createFrame(params?.tabs ?? [], params?.groupId, params?.name, params);
      result = id ? { ok: true, frameId: id } : { ok: false, error: 'no valid tabs' };
      break;
    }
    case 'frame-remove': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      this.hideFrame(params.frameId);
      this.deleteFrame(params.frameId);
      result = { ok: true };
      break;
    }
    case 'frame-move': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      this._backend?.updatePosition(params.frameId, params.position);
      this._scheduleSave();
      setTimeout(() => this._fitTerminalsInFrame(params.frameId), 150);
      result = { ok: true };
      break;
    }
    case 'frame-resize': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      this._backend?.updateSize(params.frameId, params.size);
      this._scheduleSave();
      setTimeout(() => this._fitTerminalsInFrame(params.frameId), 150);
      result = { ok: true };
      break;
    }
    case 'frame-pin': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      const f = this._engine.frames.get(params.frameId)!;
      if (!f.pinned) this._engine.togglePin(params.frameId);
      result = { ok: true };
      break;
    }
    case 'frame-unpin': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      const f = this._engine.frames.get(params.frameId)!;
      if (f.pinned) this._engine.togglePin(params.frameId);
      result = { ok: true };
      break;
    }
    case 'frame-detach': {
      if (this._browserMode) { result = { ok: false, error: 'electron only' }; break; }
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      this._focusedFrameId = params.frameId;
      await this._detachFrame();
      result = { ok: true };
      break;
    }
    case 'frame-attach': {
      if (this._browserMode) { result = { ok: false, error: 'electron only' }; break; }
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      await this._attachToMainWindow(params.frameId);
      result = { ok: true };
      break;
    }
    case 'tab-add': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      if (!params?.tab?.terminalName) { result = { ok: false, error: 'tab.terminalName required' }; break; }
      if (this._activeTerminals.has(params.tab.terminalName)) { result = { ok: false, error: 'terminal already open' }; break; }
      this._engine.addTab(params.frameId, toFrameTabConfig(params.tab));
      this._activeTerminals.add(params.tab.terminalName);
      await this._ensureTerminalExists(params.tab.terminalName);
      this._scheduleSave();
      result = { ok: true };
      break;
    }
    case 'tab-remove': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      if (!params?.terminalName) { result = { ok: false, error: 'terminalName required' }; break; }
      const frame = this._engine.frames.get(params.frameId)!;
      if (!frame.tabs.find(t => t.key === params.terminalName)) { result = { ok: false, error: 'terminal not in frame' }; break; }
      this._engine.removeTab(params.frameId, params.terminalName);
      this._activeTerminals.delete(params.terminalName);
      const updated = this._engine.frames.get(params.frameId);
      if (!updated || updated.tabs.length === 0) {
        this.hideFrame(params.frameId);
        this.deleteFrame(params.frameId);
      }
      this._scheduleSave();
      result = { ok: true };
      break;
    }
    case 'group-save': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      if (!params?.name) { result = { ok: false, error: 'name required' }; break; }
      this._focusedFrameId = params.frameId;
      await this._saveFrameAsGroup(params.name);
      result = { ok: true };
      break;
    }
    case 'group-update': {
      if (!this._engine?.frames.has(params?.frameId)) { result = { ok: false, error: 'frame not found' }; break; }
      if (!this._frameGroupIds.has(params.frameId)) { result = { ok: false, error: 'frame has no group' }; break; }
      await this._updateGroup(params.frameId);
      result = { ok: true };
      break;
    }
    case 'group-delete': {
      if (!params?.groupId) { result = { ok: false, error: 'groupId required' }; break; }
      await this._deleteGroupById(params.groupId);
      result = { ok: true };
      break;
    }
    case 'organiser-apply': {
      const validPresets = ['side-by-side', 'stacked', 'grid', 'main-sidebar', 'focus'];
      if (!validPresets.includes(params?.preset)) { result = { ok: false, error: 'unknown preset: ' + params?.preset }; break; }
      this.applyOrganiser(params.preset);
      result = { ok: true };
      break;
    }
    default:
      result = { ok: false, error: 'unknown command: ' + command };
  }

  this._lastCommandResult = result;
  return result;
}
```

- [ ] **Step 8: Rewrite getUIState to read from engine.frames**

```typescript
getUIState() {
  const FRAME_ACTIONS = [
    { name: 'remove', source: 'backend', tool: 'trellis_workspace', operation: 'frame-remove' },
    { name: 'move', source: 'backend', tool: 'trellis_workspace', operation: 'frame-move' },
    { name: 'resize', source: 'backend', tool: 'trellis_workspace', operation: 'frame-resize' },
    { name: 'pin', source: 'backend', tool: 'trellis_workspace', operation: 'frame-pin' },
    { name: 'unpin', source: 'backend', tool: 'trellis_workspace', operation: 'frame-unpin' },
    { name: 'add-tab', source: 'backend', tool: 'trellis_workspace', operation: 'tab-add' },
    { name: 'detach', source: 'backend', tool: 'trellis_workspace', operation: 'frame-detach' },
    { name: 'attach', source: 'backend', tool: 'trellis_workspace', operation: 'frame-attach' },
  ];
  const TAB_ACTIONS = [
    { name: 'remove', source: 'backend', tool: 'trellis_workspace', operation: 'tab-remove' },
  ];
  const WORKSPACE_ACTIONS = [
    { name: 'create-frame', source: 'backend', tool: 'trellis_workspace', operation: 'frame-create' },
    { name: 'apply-organiser', source: 'backend', tool: 'trellis_workspace', operation: 'organiser-apply' },
  ];

  const frames = this._engine
    ? [...this._engine.frames.values()].filter(f => !f.hidden).map(f => ({
        id: f.key,
        order: f.order,
        position: f.position,
        size: f.size,
        zIndex: f.zIndex,
        pinned: f.pinned,
        actions: FRAME_ACTIONS,
        tabs: f.tabs.map((t, idx) => ({
          terminalName: t.key,
          type: t.key.startsWith('slot-') ? 'slot' : 'repo',
          tabIndex: idx,
          actions: TAB_ACTIONS,
        })),
        activeTabIndex: f.tabs.findIndex(t => t.key === f.activeTabKey),
      }))
    : [];

  const state: Record<string, unknown> = {
    frames,
    focusedFrameId: this._focusedFrameId,
    actions: WORKSPACE_ACTIONS,
  };
  if (this._lastCommandResult) {
    state.commandResult = this._lastCommandResult;
    this._lastCommandResult = null;
  }
  return state;
}
```

- [ ] **Step 9: Rewrite keyboard/tab navigation to use engine**

```typescript
private _nextTab() {
  if (!this._focusedFrameId || !this._engine) return;
  const frame = this._engine.frames.get(this._focusedFrameId);
  if (!frame || frame.tabs.length === 0) return;
  const currentIdx = frame.tabs.findIndex(t => t.key === frame.activeTabKey);
  const nextIdx = (currentIdx + 1) % frame.tabs.length;
  this._engine.setActiveTab(this._focusedFrameId, frame.tabs[nextIdx].key);
}

private _prevTab() {
  if (!this._focusedFrameId || !this._engine) return;
  const frame = this._engine.frames.get(this._focusedFrameId);
  if (!frame || frame.tabs.length === 0) return;
  const currentIdx = frame.tabs.findIndex(t => t.key === frame.activeTabKey);
  const prevIdx = (currentIdx - 1 + frame.tabs.length) % frame.tabs.length;
  this._engine.setActiveTab(this._focusedFrameId, frame.tabs[prevIdx].key);
}

private _jumpToTab(index: number) {
  if (!this._focusedFrameId || !this._engine) return;
  const frame = this._engine.frames.get(this._focusedFrameId);
  if (!frame || index < 0 || index >= frame.tabs.length) return;
  this._engine.setActiveTab(this._focusedFrameId, frame.tabs[index].key);
}

private _nextFrame() {
  if (!this._engine) return;
  const visible = [...this._engine.frames.entries()].filter(([, f]) => !f.hidden).sort((a, b) => a[1].order - b[1].order);
  if (visible.length === 0) return;
  const currentIdx = visible.findIndex(([k]) => k === this._focusedFrameId);
  const nextIdx = (currentIdx + 1) % visible.length;
  this._focusedFrameId = visible[nextIdx][0];
  this._engine.bringToFront(visible[nextIdx][0]);
}

private _prevFrame() {
  if (!this._engine) return;
  const visible = [...this._engine.frames.entries()].filter(([, f]) => !f.hidden).sort((a, b) => a[1].order - b[1].order);
  if (visible.length === 0) return;
  const currentIdx = visible.findIndex(([k]) => k === this._focusedFrameId);
  const prevIdx = (currentIdx - 1 + visible.length) % visible.length;
  this._focusedFrameId = visible[prevIdx][0];
  this._engine.bringToFront(visible[prevIdx][0]);
}

private _spatialNav(direction: 'up' | 'down' | 'left' | 'right') {
  if (!this._engine) return;
  const target = this._engine.focusDirection(direction);
  if (target) {
    this._focusedFrameId = target;
    this._engine.bringToFront(target);
  }
}
```

- [ ] **Step 10: Rewrite focusFrame, focusTab, isTerminalOpen, _onCloseTab**

```typescript
focusFrame(frameId: string) {
  if (!this._engine?.frames.has(frameId)) return;
  this._focusedFrameId = frameId;
  this._engine.bringToFront(frameId);
}

focusTab(frameId: string, tabIndex: number) {
  if (!this._engine?.frames.has(frameId)) return;
  const frame = this._engine.frames.get(frameId)!;
  if (tabIndex >= 0 && tabIndex < frame.tabs.length) {
    this._engine.setActiveTab(frameId, frame.tabs[tabIndex].key);
  }
}

isTerminalOpen(terminalName: string): boolean {
  return this._activeTerminals.has(terminalName);
}

private _onCloseTab() {
  if (!this._focusedFrameId || !this._engine) return;
  const frame = this._engine.frames.get(this._focusedFrameId);
  if (!frame) return;
  const activeTab = frame.tabs.find(t => t.key === frame.activeTabKey);
  if (!activeTab) return;
  this._engine.removeTab(this._focusedFrameId, activeTab.key);
  this._activeTerminals.delete(activeTab.key);
  const updated = this._engine.frames.get(this._focusedFrameId);
  if (!updated || updated.tabs.length === 0) {
    this.hideFrame(this._focusedFrameId);
    this.deleteFrame(this._focusedFrameId);
  }
  this._scheduleSave();
}
```

- [ ] **Step 11: Rewrite _saveFrameAsGroup, _updateGroup to read from engine**

```typescript
private async _saveFrameAsGroup(name: string) {
  if (!this._focusedFrameId || !this._engine) return;
  const frame = this._engine.frames.get(this._focusedFrameId);
  if (!frame) return;
  const data = await this._loadGroupsData();
  const group = { id: crypto.randomUUID(), name, tabs: frame.tabs.map(toTabRef) };
  data.groups.push(group);
  await this._saveGroupsData(data);
  this._frameGroupIds.set(this._focusedFrameId, group.id);
}

private async _updateGroup(frameId: string) {
  const groupId = this._frameGroupIds.get(frameId);
  if (!groupId || !this._engine) return;
  const frame = this._engine.frames.get(frameId);
  if (!frame) return;
  const data = await this._loadGroupsData();
  const group = data.groups.find((g: any) => g.id === groupId);
  if (group) {
    group.tabs = frame.tabs.map(toTabRef);
    await this._saveGroupsData(data);
  }
}
```

- [ ] **Step 12: Remove deleted internal state maps and Dockview-specific methods**

Remove from class field declarations:
- `_frameOrders`, `_frameTabs`, `_frameGroups`, `_groupToFrame`, `_framePositions`, `_frameActiveTab`, `_frameSizes`, `_frameZIndices`, `_pinnedFrames`, `_hiddenFrames`, `_nextOrder`, `_normalMaxZ`, `_pinnedMaxZ`, `_draggingPanelId`, `_dragDidDrop`

Remove methods:
- `_initDockview`, `_injectFrameChrome` (or `_injectCloseDot`), `_subscribeOverlayEvents`, `_syncTabsFromDockview`, `_findFloatingOverlay`, `_applyZIndex`, `_compactZOrder`, `_readOverlayBounds`, `_moveFrame`, `_resizeFrame`, `_findFrameByPanels`, `_panelIdFromTabElement`

Remove imports:
- `DockviewComponent`, `DockviewGroupPanel`, `themeDark` from `'dockview-core'`
- `dockviewCSS` import
- `bringToFront as zBringToFront`, `compactFrames`, `normalizeForSave` from `'./workspace-zorder.js'`
- `findSpatialTarget` from `'./workspace-spatial-nav.js'`
- `PRESETS` from `'./workspace-organisers.js'`

Keep imports:
- `computeAllTiers`, `computeTransitions`, `type RendererTier` from `'./workspace-renderer-tiers.js'`
- `xtermCSS` import (still needed for terminal rendering)

- [ ] **Step 13: Update CSS — remove Dockview CSS import, keep xterm CSS**

Remove the `unsafeCSS(dockviewCSS)` from `static override styles`. The backend handles Dockview CSS injection. Keep `unsafeCSS(xtermCSS)` and all trellis-specific styles.

- [ ] **Step 14: Update tests — rewrite all test suites**

Rewrite test suites to use `engine.frames` for state assertions instead of internal maps. The general pattern changes from:

```typescript
// Old: access internal map
expect((el as any)._frameZIndices.get(f1)).toBeGreaterThan(10000);
// New: read from engine
expect((el as any)._engine.frames.get(f1).zIndex).toBeGreaterThan(10000);

// Old: internal tab map
expect((el as any)._frameTabs.get(f1).length).toBe(2);
// New: engine frames
expect((el as any)._engine.frames.get(f1).tabs.length).toBe(2);

// Old: internal pinned set
expect((el as any)._pinnedFrames.has(f1)).toBe(true);
// New: engine frames
expect((el as any)._engine.frames.get(f1).pinned).toBe(true);

// Old: internal active tab
expect((el as any)._frameActiveTab.get(f1)).toBe(1);
// New: engine activeTabKey
const frame = (el as any)._engine.frames.get(f1);
expect(frame.tabs.findIndex(t => t.key === frame.activeTabKey)).toBe(1);
```

Delete test suites that test removed Dockview internals:
- CSS injection tests (`dockview-theme-dark`, `.dv-scrollable` assertions)
- DockviewComponent constructor option tests (`createRightHeaderActionComponent`, `floatingGroupDragHandle`)
- `overlay.setBounds` tests
- `_syncTabsFromDockview` tests
- `onDidActivePanelChange` Dockview event tests
- `createComponent` factory tests (pointerdown activation)

Keep test suites that test trellis behavior (update internal map references):
- Toolbar, picker, flyout, groups, REST persistence, detach/reattach, renderer lifecycle, handleCommand validation, getUIState shape, type bridge, nextFramePosition, clampPosition

- [ ] **Step 15: Run all tests**

Run: `yarn --cwd sidecar/src/main/webui test -- --run`
Expected: PASS — all tests pass with the new engine

- [ ] **Step 16: Verify build**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: No errors

- [ ] **Step 17: Commit**

```bash
git add sidecar/src/main/webui/src/components/workspace-view.ts sidecar/src/main/webui/src/components/workspace-view.test.ts
git commit -m "feat(#43): replace inline Dockview with FloatingFrameEngine from pages-runtime

Delegate frame/tab state, z-order, spatial nav, organisers to pages-runtime
engine. workspace-view retains terminal lifecycle, MCP commands, persistence
bridging, picker UI, renderer tiers, and Electron IPC.

Refs #43"
```

---

### Task 3: Delete replaced utility modules

**Files:**
- Delete: `sidecar/src/main/webui/src/components/workspace-zorder.ts` (use `ide_refactor_safe_delete`)
- Delete: `sidecar/src/main/webui/src/components/workspace-zorder.test.ts` (bash rm — test file)
- Delete: `sidecar/src/main/webui/src/components/workspace-spatial-nav.ts` (use `ide_refactor_safe_delete`)
- Delete: `sidecar/src/main/webui/src/components/workspace-spatial-nav.test.ts` (bash rm — test file)
- Delete: `sidecar/src/main/webui/src/components/workspace-organisers.ts` (use `ide_refactor_safe_delete`)
- Delete: `sidecar/src/main/webui/src/components/workspace-organisers.test.ts` (bash rm — test file)

**Interfaces:**
- Consumes: Task 2 removed all imports of these modules from workspace-view.ts
- Produces: Nothing — pure cleanup

- [ ] **Step 1: Verify no remaining imports of deleted modules**

Use `ide_find_references` on each module to confirm no remaining consumers:
- `workspace-zorder` — should have 0 references after Task 2
- `workspace-spatial-nav` — should have 0 references
- `workspace-organisers` — should have 0 references

- [ ] **Step 2: Delete the files**

Use `ide_refactor_safe_delete` for each `.ts` source file. Use bash `rm` for test files (not source code — no reference tracking needed).

- [ ] **Step 3: Run all tests**

Run: `yarn --cwd sidecar/src/main/webui test -- --run`
Expected: PASS — no tests depend on deleted modules

- [ ] **Step 4: Verify build**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
git add -A sidecar/src/main/webui/src/components/
git commit -m "chore(#43): delete workspace utility modules replaced by pages-runtime

workspace-zorder, workspace-spatial-nav, workspace-organisers now live in
@casehubio/pages-runtime as frame-zorder, frame-spatial-nav, frame-organisers.

Refs #43"
```

---

### Task 4: Remove dockview-core direct dependency + CLAUDE.md update

**Files:**
- Modify: `sidecar/src/main/webui/package.json`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: dockview-core is now a transitive dep through pages-runtime
- Produces: Cleaned dependency tree, updated CLAUDE.md conventions

- [ ] **Step 1: Remove dockview-core from package.json**

In `sidecar/src/main/webui/package.json`, remove the line:
```json
"dockview-core": "^7.0.4",
```

- [ ] **Step 2: Run yarn install**

Run: `yarn --cwd sidecar/src/main/webui install`
Expected: Success — dockview-core still resolved as transitive dep through pages-runtime

- [ ] **Step 3: Run all tests + build**

Run: `yarn --cwd sidecar/src/main/webui test -- --run && yarn --cwd sidecar/src/main/webui build`
Expected: All pass

- [ ] **Step 4: Update CLAUDE.md**

Add to Key Conventions section, after the existing workspace view entries:

```markdown
- workspace-view delegates frame/tab state to `FloatingFrameEngine` + `DockviewBackend` from `@casehubio/pages-runtime` — trellis provides a `ContentFactory` for terminal elements and bridges persistence format
```

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/package.json CLAUDE.md
git commit -m "chore(#43): remove dockview-core direct dep, update CLAUDE.md

dockview-core is now a transitive dependency through @casehubio/pages-runtime.
CLAUDE.md updated with engine delegation convention.

Refs #43"
```

---

## Task Dependencies

```
Task 1 (type bridge + engine fields) ─► Task 2 (core migration) ─► Task 3 (delete utils) ─► Task 4 (deps + docs)
```

All tasks are sequential — each depends on the previous.

## Execution Order

Sequential (inline): 1 → 2 → 3 → 4
