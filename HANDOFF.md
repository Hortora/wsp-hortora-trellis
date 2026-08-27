# Handoff — trellis

## Last Session

Batched three small issues on `issue-45-terminal-font-size-and-fixes`: per-frame terminal font size cycling (#45 — `FONT_SIZES` presets, `_frameFontSizes` map, cycle button in frame titlebar, layout persistence), repo-detail 409 fix (#30 — one-line guard matching workspace-view's `_ensureTerminalExists`), and provenance write-path contract tests (#21 — 10 tests validating `ProvenanceRecord` deserialization, GE-ID formats, enrichment edge cases). Landed as `2e55fbf` on main. Then set up branch `issue-49-pluggable-workbench-layout` for #49 — scaffold only, no code changes.

## Immediate Next Step

Branch `issue-49-pluggable-workbench-layout` is open for #49. Run `/work` to continue. **Before brainstorming the design, audit what pages now provides for workbenches** — pages has had significant recent improvements to workbench capabilities. Check `@casehubio/pages-runtime` and `@casehubio/pages-component` for new workbench layout primitives, split-frame support, or dock-bar APIs that may already solve parts of #49. Then brainstorm the design using what pages offers. Key file: `sidecar/src/main/webui/src/components/workbench.ts`.

## Cross-Module

No cross-module blockers. `pages-runtime` re-export gap (casehubio/casehub-pages#303) is still open but does not block #49.

## References

- Issue #49: `Hortora/trellis#49` — open, branch scaffolded
- Workbench source: `sidecar/src/main/webui/src/components/workbench.ts`
- Workspace-view (Dockview reference impl): `sidecar/src/main/webui/src/components/workspace-view.ts`
