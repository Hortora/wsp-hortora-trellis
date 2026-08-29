# Handoff — trellis

## Last Session

Two issues completed, one started:

**#49 — Pluggable Workbench Layout (CLOSED):** Rewrote trellis-workbench from 360-line monolithic Lit component to two-layer pages-runtime architecture: `dockWorkbench()` + `renderComponent()` for dock bars with `ZoneLayoutEngine` drag rearrangement, `Container` from frame-sandbox for pluggable content area (content/tabbed/splith/splitv modes). Backend: replaced `/api/workspace/layout` + `/api/workspace/groups` with key-based `/api/layouts/{key}` endpoint with path traversal protection. Fixed pre-existing Quarkus startup failures (StaticCoordinatorRouting return type, SmallRye config mapping validation). 11 commits squashed to 4, landed on main. Also fixed `artifact_promote.py` and `close_artifacts.py` in soredium (commit message missing issue ref + empty staged check).

**#19 — Work Intelligence (IN PROGRESS, branch open):** Reframed from velocity tracking to proactive work intelligence — RAS-based detection of stalled, forgotten, unblocked, and cross-repo dependency gaps. Design spec and implementation plan written. 3 batches, 7 tasks. No implementation started — ready for Batch 1 Task 1 (add RAS dependencies + CloudEvent factory).

## Immediate Next Step

`work continue` on branch `issue-19-velocity-tracking-projections`. Plan at `plans/2026-08-29-work-intelligence.md`. Start with executing-plans Batch 1 Task 1.

## Cross-Module

- `blocks-ui-core` in `.casehub-packages/` is out of date — `kpi-metric-row` imports `emitPagesEvent` and `renderSparkline` which aren't exported. Blocks `yarn build` and `quarkus:dev`. Needs portal sync.
- Soredium fix for `artifact_promote.py` committed on `issue-306-slot-manager-decomposition` branch — may need merging to main.

## References

- Design spec: `specs/issue-19-velocity-tracking-projections/2026-08-29-work-intelligence-design.md`
- Plan: `plans/2026-08-29-work-intelligence.md`
- Decisions: `specs/issue-19-velocity-tracking-projections/decisions.md`
- Garden: GE-20260828-74bbb5 (createRestLayoutStore no query params), GE-20260424-4b7aa2 revised (SRCFG00050 fix confirmed)
