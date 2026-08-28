# Decisions — #49 Pluggable Workbench Layout

## D1: Workspace view remains a dedicated dock-bar panel

**Choice:** Workspace view stays as a dock-bar panel, not a container entry
**Alternatives:**
- Workspace as a container entry — visible simultaneously with other panels in split/free mode, but risks layout conflicts between container LayoutStrategy and workspace's internal Dockview
- Workspace IS the content area — other panels dock around it, one Dockview instance for everything — couples two independent layout concerns
**Rationale:** Workspace manages complex internal state (terminals, frames, renderer tiers, detach/reattach). Isolating it as a dock-bar panel keeps its Dockview scope separate from the workbench content layout.
**Trade-offs:** Can't view workspace side-by-side with dashboard/backlog in the content area. Switching requires dock-bar click.
**Sources:** workbench.ts (current implementation), issue #49 design notes
**Exploration:** quick
**Status:** captured

## D2: Preserve current panel structure for initial migration

**Choice:** Keep all 8 dock-bar panels as-is, keep 3 hash sub-views (slot, epic, repo) as dashboard overlays. Reorganisation deferred to future incremental work.
**Alternatives:**
- Promote some panels to permanent content area entries (e.g., dashboard + backlog always visible) — adds scope, premature without user feedback on which combinations matter
- Demote rarely-used panels to content-only (no dock-bar button) — saves dock-bar space but changes muscle memory
**Rationale:** Migration should change the infrastructure, not the panel topology. Incremental reorganisation after the new layout is stable avoids coupling two changes.
**Trade-offs:** Initial result looks identical to today — the value is in the unlocked capability, not immediate visual change.
**Sources:** workbench.ts DOCK_PANELS array, PANELS record
**Exploration:** quick
**Status:** captured

## D3: Hybrid consumption — loadSite + dockWorkbench for dock bars, direct Container API for content area

**Choice:** Option C — use `dockWorkbench()` / `loadSite()` for dock bar rendering and zone management; create a `Container` explicitly for the content area with `createContainer()` and `createContainerToolbar()`
**Alternatives:**
- A: Full Pages DSL (`loadSite` + `dockWorkbench` + `hostPanel` centre) — simplest migration but defers content area layout modes, which is the core #49 requirement
- B: `ZoneLayoutEngine` + custom Lit rendering — more control but duplicates `renderComponent()` and `buildTreeFromZones()` (~150 lines of unnecessary custom code)
**Rationale:** #49 explicitly requires pluggable content area layout (single-frame, split-frame at minimum). The Container API delivers this directly — tabbed, split, free, accordion layout modes via `LayoutStrategy`, with `OrganiserToolbar` for switching. Dock bars are pure infrastructure; `dockWorkbench()` handles them without custom code. The two layers are independent.
**Trade-offs:** Slightly more integration code than pure Option A (Container setup + ContentFactory), but delivers the actual feature. Container API is i3wm-inspired recursive tiling — powerful but more surface area to understand.
**Depends on:** D1 (workspace as dock-bar panel — not a container entry)
**Sources:** pages-runtime frame-sandbox/container.d.ts, frame-sandbox/types.d.ts, issue #49 acceptance criteria
**Exploration:** deep-analysis
**Status:** captured

## D4: Trellis adopts pages layout persistence model

**Choice:** Trellis backend serves `/api/layouts/{key}` (GET/PUT/DELETE) using pages' `LayoutState` type. Frontend uses `createRestLayoutStore()` directly. Drop existing `/api/workspace/layout` and `/api/workspace/groups` endpoints.
**Alternatives:**
- Keep trellis endpoints, write adapter — shim code, two models for the same thing, pre-release so no backward compat needed
- Keep trellis endpoints, extend with container state — fragments layout state across 3+ endpoints as features grow
**Rationale:** Pages defines the canonical layout model (`LayoutState` with zones, docks, splits, containerState, frames). One state object, one key, one endpoint. Trellis consuming it directly means zero glue code and one source of truth for the persistence contract.
**Trade-offs:** Trellis backend endpoint rename — existing `/api/workspace/layout` and `/api/workspace/groups` callers need updating. No external consumers, so no migration concern.
**Sources:** pages-runtime rest-layout-store.ts, pages-component types.ts (LayoutState), trellis WorkspaceResource.java
**Exploration:** quick
**Status:** captured
