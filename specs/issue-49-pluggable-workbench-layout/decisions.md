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
