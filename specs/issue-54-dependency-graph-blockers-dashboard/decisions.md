# Decisions — #54 Dependency Graph + Blockers Dashboard

## D1: Issue body parsing for dependency extraction

**Choice:** Parse dependency relationships from cached issue bodies — `blocked by #N`, `depends on #N`, `## Blocked by` sections, epic checklists with `blocked by` annotations.
**Alternatives:**
- Label-based only — simple but misses most dependencies since they're rarely labeled
- GitHub timeline API — most complete (catches implicit cross-references) but requires API calls per issue and isn't cached
- Comment parsing — catches blockers added later, but requires caching comments (extra API calls per issue)
**Rationale:** Hortora/casehub conventions consistently declare dependencies in issue bodies, not comments or labels. Body text is already cached locally in `github_issue_cache.body`. Covers the intentional, explicit dependencies that actually block work.
**Trade-offs:** Misses blockers declared in comments after issue creation. Acceptable for first cut — comment scanning can be added later without architectural change.
**Sources:** github_issue_cache schema, sample issues #226 (full dependency graph), #282 (## Blocked by section), #190 (epic child checklist)
**Exploration:** quick
**Status:** captured

## D2: Single-repo graph scope

**Choice:** Dependencies within one repo, scoped by the workspace root folder. Cross-repo references shown as external links.
**Alternatives:**
- Cross-repo graph — full dependency graph across all org repos. More useful for multi-repo work but needs enrichment cache populated for all repos.
- Active work only — only show dependencies for issues with active worklog items. Smallest scope but misses upcoming work.
**Rationale:** Simpler first cut. Cross-repo can be added later by widening the repo filter.
**Trade-offs:** Cross-repo blockers (e.g., trellis blocked by soredium#282) appear as unresolved external references, not as nodes in the graph.
**Sources:** github_issue_cache schema (issue_repo column enables filtering)
**Exploration:** quick
**Status:** captured

## D3: Three-column status view layout

**Choice:** BLOCKED (red, with blocker chain) | UNBLOCKED (green, recently resolved) | CLEAR (no dependencies). Critical path highlighted at top.
**Alternatives:**
- Directed graph visualization — interactive node-graph with edges. More visual but harder to scan quickly.
- Table with dependency column — pages-data-table with filterable columns. Less visual, more data-dense.
**Rationale:** Familiar kanban-like layout. Immediately shows what's stuck, what just freed up, and what's clear. Critical path at top drives action.
**Trade-offs:** Doesn't show the full graph structure — a chain A→B→C→D appears as "D blocked by C" not as a visual chain. The critical path section compensates.
**Sources:** User request: "things blocked and what's blocking them, things now unblocked"
**Exploration:** quick
**Status:** captured

## D4: Shared DependencyService, separate panels

**Choice:** A DependencyService builds and caches the dependency graph. The blockers dashboard reads it for display. The intelligence adapters read it for transition detection. Two views, one source of truth.
**Alternatives:**
- Merge into intelligence panel — extend intelligence to show the graph. Risk: bloated panel trying to be two things.
- Independent with cross-links — separate data paths, linked via navigation. Simpler but duplicates data access.
**Rationale:** The dependency graph IS the foundation the intelligence layer needs. Both the dashboard (current state) and intelligence (state transitions) are views of the same data. Single service avoids divergence.
**Trade-offs:** DependencyService becomes a shared dependency — changes affect both consumers.
**Depends on:** D1 (data source determines what DependencyService parses)
**Sources:** EnrichmentAdapter hardcoded OPEN bug, UnblockedWorkGanglion detection logic
**Exploration:** quick
**Status:** captured

## D5: Simple chain-length critical path

**Choice:** Critical path = longest chain of unresolved blocking dependencies. Pure graph algorithm via topological sort.
**Alternatives:**
- Enrichment-weighted — weight edges by scale + complexity from enrichment data. More accurate but depends on enrichment being populated.
**Rationale:** Easy to understand and verify. No dependency on enrichment data being populated. Chain length is the right first-order signal — a 5-deep blocking chain is critical regardless of individual issue sizes.
**Trade-offs:** Treats all issues as equal weight. A chain of 3 XL/High issues isn't distinguished from 3 XS/Low ones.
**Sources:** Issue #226 dependency graph (demonstrates real chain structure)
**Exploration:** quick
**Status:** captured

## D6: Workspace root folder as scope ("space")

**Choice:** The root folder passed to trellis defines the "space". Trellis scans for git repos under it, resolves GitHub owner/repo from git remotes, and that set of repos scopes all queries to `github_issue_cache`. All panels share the same scoped set.
**Alternatives:**
- Hardcoded repo list — configure which repos to show in CLAUDE.md or settings. Simple but static.
- Worklog repos table — use the existing `repos` table mapping. Already populated but `github_repo` column is empty for all entries.
**Rationale:** Dynamic, zero-config, matches how the user thinks about trellis ("point it at a folder"). Switching roots changes the space and all panels update.
**Trade-offs:** Requires scanning the filesystem and resolving git remotes. Cached after first scan via FileWatcherService (directory-watcher).
**Sources:** repos table (2543 entries, 0 with github_repo populated), FileWatcherService (io.methvin directory-watcher already in POM)
**Exploration:** quick
**Status:** captured

## D7: Filesystem watcher for repo discovery

**Choice:** Use the existing FileWatcherService (backed by io.methvin:directory-watcher) to detect repos appearing or disappearing under the root. Same infrastructure already used for workspace file changes.
**Alternatives:**
- On-demand only — scan when root changes or panel requests data. Simpler but misses new repos until next explicit scan.
**Rationale:** Platform standard. Already a dependency. Keeps the space up to date without polling or manual refresh.
**Trade-offs:** Watcher overhead for something that rarely changes. Acceptable — directory-watcher is efficient for low-change directories.
**Sources:** pom.xml (io.methvin:directory-watcher:0.19.1), FileWatcherService.java
**Exploration:** quick
**Status:** captured
