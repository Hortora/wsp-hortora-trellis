# Handoff — trellis

## Last Session

Batched three small issues on `issue-45-terminal-font-size-and-fixes`: per-frame terminal font size cycling (#45 — `FONT_SIZES` presets, `_frameFontSizes` map, cycle button in frame titlebar, layout persistence), repo-detail 409 fix (#30 — one-line guard matching workspace-view's `_ensureTerminalExists`), and provenance write-path contract tests (#21 — 10 tests validating `ProvenanceRecord` deserialization, GE-ID formats, enrichment edge cases). Landed as `2e55fbf` on main. Then set up branch `issue-49-pluggable-workbench-layout` for #49 — scaffold only, no code changes.

## Immediate Next Step

Branch `issue-49-pluggable-workbench-layout` is open for #49. Run `/work` to continue. Start with brainstorming — the workbench currently uses hardcoded single-panel switching (`_activePanel` string, `DOCK_PANELS` array in `workbench.ts`). The issue asks for pluggable layout models (single-frame, split-frame, free-layout via Dockview) and optional dock bars. Key file: `sidecar/src/main/webui/src/components/workbench.ts`.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Issue #49: `Hortora/trellis#49` — open, branch scaffolded
- Workbench source: `sidecar/src/main/webui/src/components/workbench.ts`
- Workspace-view (Dockview reference impl): `sidecar/src/main/webui/src/components/workspace-view.ts`
