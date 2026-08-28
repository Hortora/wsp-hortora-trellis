# Pluggable Workbench Layout — Design Spec

**Issue:** Hortora/trellis#49
**Branch:** issue-49-pluggable-workbench-layout
**Date:** 2026-08-28

## 1. Architecture Overview

The workbench migrates from a monolithic Lit component with manual dock-bar rendering and single-panel switching to a two-layer architecture.

### 1.1 Dock Bar Layer

`dockWorkbench()` from pages-ui generates a component tree with left/right/bottom dock bars. `renderComponent()` renders it directly — no `loadSite()` (trellis has no pages site context; this is first pages-runtime adoption).

Each dock bar panel is a `DockPanelConfig`:
- `key` — panel identifier (e.g., `"workspace"`, `"dashboard"`)
- `icon`, `label` — dock bar button presentation
- `content` — `hostPanel(key)` referencing a registered Lit component
- `zone` — default zone (`"top"` or `"bottom"` within its side)
- `allowedZones` — constrains drag targets
- `fixed` — locks non-moveable panels (e.g., workspace)

Panel activation via `pages-dock-toggle` CustomEvent dispatch (the same mechanism `LiveSite.activateDockPanel()` uses internally). Zone rearrangement via `attachDockDrag()` with `pages-dock-rearrange` events feeding `ZoneLayoutEngine.movePanel()`.

Each dock bar (left, right, bottom) is independently hideable. When all panels in a side are closed, the dock bar's cascading collapse (from `dockWorkbench()`'s component-level toggle) hides the entire side — no manual hide logic needed. A side with no `DockPanelConfig` entries omitted from the config entirely. Dock bar visibility state persisted via `LayoutState.docks`.

### 1.2 Content Area Layer

A `Container` from pages-runtime's frame-sandbox, created with `createContainer()`. The container holds `Entry` objects — one per visible panel. A `ContentFactory` lazily creates Lit panel elements, replacing the current `_getOrCreatePanel()` cache.

Layout mode switchable at runtime via `container.setLayout()`:
- `"content"` — single panel (default, matches current behaviour)
- `"tabbed"` — tabbed panel group
- `"splith"` / `"splitv"` — horizontal/vertical split
- `"free"` — Dockview-backed free layout
- `"accordion"` — collapsible stack

`createContainerToolbar()` provides the mode-switching UI. Container is i3wm-inspired: recursive and regular — the same Container/Entry/LayoutStrategy abstraction applies at every nesting level.

### 1.3 Persistence

`createRestLayoutStore()` backed by a new `/api/layouts/{key}` endpoint. Single `LayoutState` object covers:
- `zones` — panel-to-zone assignments (feeds `ZoneLayoutEngine` savedZones)
- `docks` — panel open/closed state
- `splits` — split ratios
- `containerState` — content area layout mode + per-layout state
- `frames` — frame positions (if free layout used)

Key: `"workbench"` (extend to `"workbench::<root>"` for multi-workspace later).

### 1.4 Navigation

- **SSE `control:navigate`** — dispatches `pages-dock-toggle` for panel activation. Deep paths (e.g., `panels/workspace-view/frames/xyz/tabs/2`) activate workspace panel then forward sub-path to workspace-view's `focusFrame()`/`focusTab()`.
- **Hash routing** — pages' `panel=` DeepLink parsing via `parseFromUrl()` / `serializeToUrl()`. `#?panel=backlog` activates the backlog panel.
- **`control:workspace`** — unchanged, forwarded to workspace-view's `handleCommand()`.

## 2. Component Changes

### 2.1 trellis-workbench.ts — Rewrite

The current 360-line component is replaced with ~100-120 lines: config declaration, ContentFactory, Container setup, SSE wiring, UI state push.

#### What gets removed

| Current code | Replaced by |
|---|---|
| `PANELS` record (icon, label, tag) | `DockPanelConfig[]` declarations |
| `DOCK_PANELS` array | `DockWorkbenchConfig.left` initial zone assignment |
| `_panelCache` + `_getOrCreatePanel()` | `ContentFactory` function |
| `_activePanel` state + show/hide rendering | Container manages visibility |
| `_parseHash()` with 10+ regex branches | `parseFromUrl()` + `panel=` |
| `_activatePanel()` + `_lastHash` | `pages-dock-toggle` event dispatch |
| `_handleNavigateEvent()` deep path parsing | Panel-level activation + workspace-view forwarding |
| Custom CSS for `.dock-bar` + `.panel-area` | Pages renders dock bars via `renderComponent()` |
| `render()` with dock-bar buttons + panel divs | Mount point for `renderComponent()` + Container |

#### What stays (trellis-specific)

| Code | Why it stays |
|---|---|
| `_handleWorkspaceCommand()` | SSE → workspace-view forwarding, trellis-specific |
| `_buildUIState()` + `_doPushUIState()` | UI state push to sidecar — **updated** to include `layoutMode` (container's current layout type) and `visiblePanels` (list of panel keys currently visible in the container) |
| `_connectSSE()` / `_disconnectSSE()` | SSE transport |
| `_startHeartbeat()` / `_stopHeartbeat()` | Heartbeat for UI state |

#### New imports (first pages-runtime usage in trellis)

```typescript
import { dockWorkbench } from "@casehubio/pages-ui/dist/dsl/builders.js";
import { renderComponent } from "@casehubio/pages-component";
import { registerPanel, createRestLayoutStore } from "@casehubio/pages-runtime";
import { createContainer, createContainerToolbar } from "@casehubio/pages-runtime/dist/frame-sandbox";
```

#### ContentFactory

Replaces `_getOrCreatePanel()`. Maps panel keys to Lit element creation:

```typescript
const contentFactory: ContentFactory = (entry: Entry) => {
  const tag = PANEL_TAGS[entry.key];
  const el = document.createElement(tag);
  (el as any).workspaceRoot = this.workspaceRoot;
  return { element: el, dispose: () => el.remove() };
};
```

`PANEL_TAGS` is a simple `Record<string, string>` replacing the current `PANELS` record — only key → tag-name, no icon/label (those move to `DockPanelConfig`).

#### Panel registration

Before rendering, register all panels:

```typescript
for (const [key, tag] of Object.entries(PANEL_TAGS)) {
  registerPanel(key, tag);
}
```

### 2.2 Panel components — unchanged

All panel Lit components (`trellis-org-dashboard`, `trellis-workspace-view`, etc.) are unchanged. They remain self-contained elements with `workspaceRoot` property binding.

### 2.3 Context application

The current `_applyContext()` method sets panel-specific properties (slotNumber, owner, repo, epicNumber, etc.) based on hash routing context. Two concerns:

1. **Initial properties** — `ContentFactory` sets `workspaceRoot` when creating the element. This is the only property every panel needs at creation time.

2. **Navigation-driven properties** (slotNumber, owner, repo, epicNumber) — these are set by the dashboard component itself when it handles sub-view hash routes (`#slot/42`, `#epic/owner/repo/5`, `#repo/name`). The workbench no longer applies context to panels — each panel reads its own navigation state from the hash. The dashboard already renders slot, epic, and repo as internal sub-views; it just needs to own the hash parsing for its sub-routes instead of receiving properties from the workbench.

This eliminates the `_applyContext()` coupling — the workbench doesn't need to know each panel's property API.

## 3. Backend Changes

### 3.1 LayoutResource.java — new

Key-value store for `LayoutState` JSON:
- `GET /api/layouts/{key}` — returns stored JSON or 404
- `PUT /api/layouts/{key}` — stores JSON blob (opaque to backend, size-limited)
- `DELETE /api/layouts/{key}` — removes stored layout

Storage: file-based under `.trellis/layouts/{key}.json` in workspace root.

### 3.2 WorkspaceResource.java — remove layout endpoints

Drop `GET/PUT /api/workspace/layout` and `GET/PUT /api/workspace/groups`. Other WorkspaceResource endpoints stay.

### 3.3 TrellisTools.java — update layout path references

The MCP tool's workspace operations that reference layout state update to use `/api/layouts/{key}`. No MCP tool contract change.

### 3.4 No changes to

- `UIStateResource` (`POST /api/model/ui-state`)
- SSE push infrastructure
- `TerminalRegistry`, `SessionLogger`

## 4. Migration & Deletion Checklist

### Files modified
- `sidecar/src/main/webui/src/components/workbench.ts` — full rewrite
- `sidecar/src/main/java/.../WorkspaceResource.java` — remove layout/groups endpoints
- `sidecar/src/main/java/.../TrellisTools.java` — update layout path references

### Files created
- `sidecar/src/main/java/.../LayoutResource.java` — new REST resource

### Dead code removal verification
- No references to `/api/workspace/layout` or `/api/workspace/groups` in any `.ts` or `.java` file
- No `DOCK_PANELS` array or `PanelDef` interface
- No manual dock-bar CSS (`.dock-bar`, `.dock-btn`) in workbench
- No `_panelCache`, `_getOrCreatePanel`, `_activePanel` state
- No `_parseHash` with regex branches
- No `_lastHash` map

### Functional verification
- Hash routing works via `panel=` parameter
- SSE `control:navigate` activates panels via `pages-dock-toggle`
- SSE `control:workspace` still reaches workspace-view
- UI state push reports active panel state and layout mode
- Layout persists across page refresh via `/api/layouts/workbench`
- Dock bar panels rearrangeable via drag-and-drop
- Content area layout mode switchable (content, tabbed, split at minimum)

## 5. Decisions

See [decisions.md](decisions.md) for the full decision log. Summary:

| # | Decision |
|---|----------|
| D1 | Workspace remains a dedicated dock-bar panel (not a container entry) |
| D2 | Current panel structure preserved; reorganise incrementally later |
| D3 | Hybrid: `dockWorkbench()` + `renderComponent()` for dock bars, `Container` for content area. No `loadSite()`. |
| D4 | Trellis adopts pages `/api/layouts/{key}` + `LayoutState`. Key: `"workbench"`. |
| D5 | Panel-level navigation only. Deep paths forwarded to workspace-view directly (YAGNI on generic interface). |

## 6. Trade-offs Acknowledged

- **D1:** Workspace can't be viewed side-by-side with other panels in the content area. Acceptable — workspace manages complex internal Dockview state that would conflict with the content Container.
- **D2:** Initial result looks identical to today. The value is in unlocked capability, not immediate visual change.
- **D3:** Must dispatch `pages-dock-toggle` events directly instead of `site.activateDockPanel()`. Small cost for avoiding the full `loadSite()` dependency chain.
- **D5:** One workspace-specific `if` branch in the navigate handler. Extract interface when a second panel needs deep navigation.

## References

- [workbench.ts](/Users/mdproctor/claude/hortora/trellis/sidecar/src/main/webui/src/components/workbench.ts) — current implementation (replacement target)
- [pages-runtime zone-layout-engine.ts](pages-runtime/src/zone-layout-engine.ts) — ZoneLayoutEngine API
- [pages-runtime frame-sandbox/types.d.ts](pages-runtime/dist/frame-sandbox/types.d.ts) — Container, Entry, LayoutStrategy, ContentFactory types
- [pages-ui builders.ts](pages-ui/src/dsl/builders.ts) — dockWorkbench, DockWorkbenchConfig, DockPanelConfig
- [pages-runtime rest-layout-store.ts](pages-runtime/src/rest-layout-store.ts) — createRestLayoutStore
- [GE-20260804-befd45] — Pages dockWorkbench 3-primitive decomposition technique
- [GE-20260805-e3211c] — hostPanel requires import AND registerPanel before render
- [GE-20260809-aee002] — Zone naming gotcha (left-bottom = 2nd zone in left column)
- [GE-20260809-44b2a6] — Dockview floating position clamped to zero in Lit firstUpdated
- [GE-20260809-14d2f9] — Playwright visual TDD for layout verification
- Hortora/trellis#49 — issue description and acceptance criteria
- Hortora/trellis#46 — related: extract pages-floating-workspace
- Hortora/trellis#43 — related: frame and tab management via Agent Control Plane
