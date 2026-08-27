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
