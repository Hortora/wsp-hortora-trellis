# Handoff — trellis

## Last Session

Two issues completed, one started:

**#19 — Work Intelligence (CLOSED):** Built RAS-based intelligence system — four JavaSwitchGanglion facets (stalled-work, unblocked-work, forgotten-deferral, cross-repo-dependency), four data adapters, WorkIntelligenceModelProvider with severity mapping + summary/suggestion text, GET /api/intelligence endpoint, trellis-intelligence-panel Lit component. 45 tests. Landed as `012e01a` on main. Follow-up #53 filed for CrossRepoAdapter PR cache data sourcing.

**Runtime fixes during launch testing:** ganglions needed `@ApplicationScoped` for CDI discovery, `quarkus-micrometer` required by casehub-ras runtime, `@casehubio/pages-filter-bar` added to package.json resolutions (frontend builds now), sweep job CPU thrash fixed by disabling (`poll-interval=off`). Workbench shell has pre-existing `renderComponent` TypeError from stale `.casehub-packages`.

**#54 — Dependency Graph + Blockers Dashboard (IN PROGRESS, branch open):** Brainstorming complete, 7 decisions captured (D1-D7). No spec or implementation yet.

## Immediate Next Step

`work continue` on branch `issue-54-dependency-graph-blockers-dashboard`. Write the design spec from the 7 captured decisions, then invoke writing-plans.

## Key Decisions (D1-D7)

- D1: Issue body parsing for dependency extraction
- D2: Single-repo graph scope
- D3: Three-column status view (Blocked/Unblocked/Clear) with critical path
- D4: Shared DependencyService feeds both blockers dashboard and intelligence
- D5: Simple chain-length critical path
- D6: Workspace root folder = "space" — scopes all panels
- D7: FileWatcherService (directory-watcher) for repo discovery

## Cross-Module

- `.casehub-packages` portal packages stale — `renderComponent` TypeError blocks workbench UI. Needs full portal sync.
- Intelligence sweep job disabled (`poll-interval=off`). Needs throttling before re-enabling.

## References

- Decisions: `specs/issue-54-dependency-graph-blockers-dashboard/decisions.md`
- Intelligence spec: `specs/issue-19-velocity-tracking-projections/2026-08-29-work-intelligence-design.md`
- #53 — CrossRepoAdapter PR cache (deferred from #19)
- #54 — Dependency graph + blockers dashboard (active)
