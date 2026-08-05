# Handoff — trellis

## Last Session

Built #28 (workspace view) — full multi-frame terminal workspace with Dockview, keyboard nav, organisers, persistence, detach/reattach. Adversarial design review (51 issues, all resolved, $36). Full-stack implementation across Java/Electron/TypeScript. Fixed tilde expansion bug in protocol/artifact endpoints (#29 filed for Dockview shadow DOM rendering). Filed #27 (agent control plane). Blog published to mdproctor.github.io.

## Immediate Next Step

Fix #29 — Dockview v7 panels render at zero height inside Lit shadow DOM. This is the only blocker for the workspace view being visually functional. Investigation needed: shadow DOM CSS inheritance, Dockview theme variables, possible HTMLElement base class instead of LitElement.

## What's Left

- #29 Dockview shadow DOM rendering — zero-height panels · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #29 | Fix Dockview shadow DOM rendering | S | Med | Blocker for workspace view |
| #27 | Agent Control Plane — programmatic API | M | Med | Rich data model, ~6-8 MCP tools |
| #19 | Velocity Tracking + Projections | M | Med | Ready |
| #16 | Drafthouse Integration (B6c) | M | Med | Ready |
| #21 | End-to-end provenance test path | S | Low | Ready |

## References

- Issue: #28 (closed), #29 (open — Dockview fix), #27 (open — agent control plane)
- Spec: `docs/specs/workspace-view/2026-08-05-workspace-view-design.md`
- Plan: `docs/plans/2026-08-05-workspace-view.md`
- Blog: `blog/2026-08-05-mdp01-when-thirty-terminals-need-a-home.md`
