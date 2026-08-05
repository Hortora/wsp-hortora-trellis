# Handoff — trellis

## Last Session

Built #28 (workspace view). Full multi-frame terminal workspace within the dock-bar workbench: Dockview-backed floating frames with tabbed terminals, keyboard navigation (tab/frame/spatial/cross-window), organiser presets (grid/stacked/side-by-side/main+sidebar/focus), persistent layouts, tab hover flyouts, frame pinning, detach/reattach to separate Electron windows. Backend: readiness endpoint, terminal creation atomicity, WebSocket session takeover with close code 4001, FIFO lifecycle. Electron: LayoutStore with typed methods and separate keys, shutdown save protocol, WebGL context budget IPC, menu accelerators. Also filed #27 (agent control plane — programmatic observation and interaction API for a Trellis-level coordinating agent).

## Immediate Next Step

Pick next feature from What's Next — LLM integration or velocity tracking.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #27 | Agent Control Plane — programmatic observation/interaction API | M | Med | Filed this session. Rich data model, ~6-8 MCP tools |
| #19 | Velocity Tracking + Projections | M | Med | Ready |
| #16 | Drafthouse Integration (B6c) | M | Med | Ready |
| #21 | End-to-end provenance test path | S | Low | Ready |

## References

- Issue: #28 (closed)
- Spec: `docs/specs/workspace-view/2026-08-05-workspace-view-design.md`
- Plan: `docs/plans/2026-08-05-workspace-view.md`
- Agent control plane issue: #27 (open)
