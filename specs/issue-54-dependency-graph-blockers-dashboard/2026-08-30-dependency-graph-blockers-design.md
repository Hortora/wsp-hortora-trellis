# Dependency Graph + Blockers Dashboard — Design Spec

**Issue:** Hortora/trellis#54
**Branch:** issue-54-dependency-graph-blockers-dashboard
**Date:** 2026-08-30

## 1. Architecture Overview

A dependency resolution layer that builds and caches an in-memory graph of issue-blocks-issue relationships, scoped to the repos visible from the workspace root. Two consumers read the graph: a blockers dashboard panel (visual, for humans) and the existing intelligence layer (detection, for LLM and humans).

### Three components

1. **DependencyService** — parses issue bodies from `github_issue_cache`, builds a directed graph of blocking relationships, resolves blocker state against the cache, computes critical path. Single CDI bean, shared by both consumers.

2. **Blockers dashboard panel** (`trellis-blockers-panel`) — three-column Lit component: BLOCKED (with blocker chain) | UNBLOCKED (recently resolved) | CLEAR (no dependencies). Critical path highlighted at top.

3. **Intelligence integration** — fix `EnrichmentAdapter` to resolve actual blocker state from the dependency graph instead of hardcoding `"OPEN"`. `UnblockedWorkGanglion` starts detecting real unblock events.

### Data flow

```
github_issue_cache (body, state, labels)
        │
        ▼
  DependencyService
  ├── parse bodies → extract edges (blocked-by relationships)
  ├── resolve state → look up each blocker's actual state in cache
  ├── build graph → directed graph (issue → blocked-by → issue)
  └── compute critical path → longest chain of unresolved blockers
        │                           │
        ▼                           ▼
  GET /api/dependencies    EnrichmentAdapter (real state)
        │                           │
        ▼                           ▼
  trellis-blockers-panel   UnblockedWorkGanglion (fires)
```

### Dependencies

- Existing: `WorklogService` (database access), `FileWatcherService` + `WorkspaceScanner` (repo discovery), `GenerationCounter` (cache invalidation)
- Existing: `EnrichmentAdapter`, `UnblockedWorkGanglion` (intelligence layer — modified, not new)
- No new external dependencies

## 2. Dependency Extraction (D1)

`DependencyService` parses the `body` column of `github_issue_cache` to extract blocking relationships. Three patterns, checked in order:

### 2.1 Inline references

Pattern: `blocked by #N`, `depends on #N`, `blocks #N` (case-insensitive)
Regex: `(?i)(?:blocked\s+by|depends\s+on)\s+(?:(?:[\w.-]+/[\w.-]+)?#(\d+))` (extracts blocker)
Regex: `(?i)blocks\s+(?:(?:[\w.-]+/[\w.-]+)?#(\d+))` (extracts blocked — reverse edge)

Cross-repo references (`blocked by owner/repo#N`) captured but rendered as external links since we only resolve state for repos in the current workspace scope (D2).

### 2.2 Blocked-by sections

Pattern: `## Blocked by` or `## Dependencies` markdown section headers, followed by list items containing `#N` references.

```markdown
## Blocked by
- #42 — auth migration must land first
- Hortora/soredium#282 — garden schema change
```

### 2.3 Epic checklist annotations

Pattern: checklist items in epic bodies with `blocked by` annotations.

```markdown
- [ ] #55 — user management (blocked by #42)
- [x] #50 — logging overhaul
```

### 2.4 What is NOT parsed

- Comments (body-only, per D1 rationale)
- Labels (checked as fallback only — `blocked` label with `#N` in label text)
- GitHub timeline API (requires per-issue API calls, not cached)

### 2.5 Data access

`BacklogEntry` does not currently expose the `body` column. Rather than inflating `BacklogEntry` (which is used for the backlog table and doesn't need bodies), add a dedicated query method to `WorklogService`:

```java
record IssueDependencyData(int issueNumber, String issueRepo,
                           String state, String body) {}

List<IssueDependencyData> issueDependencyData(List<String> repos)
```

Queries `github_issue_cache` for `issue_number, issue_repo, state, body` filtered by the workspace's repo set. Returns both OPEN and CLOSED issues — closed issues are needed to resolve blocker state.

## 3. Graph Model

### 3.1 Data structures

```java
record DependencyEdge(IssueRef from, IssueRef to, EdgeType type) {}
record IssueRef(int number, String repo) {}
enum EdgeType { BLOCKED_BY, BLOCKS }
enum IssueStatus { BLOCKED, UNBLOCKED, CLEAR }

record DependencyNode(IssueRef ref, String title, String issueState,
                      IssueStatus status, List<IssueRef> blockedBy,
                      List<IssueRef> blocking) {}

record DependencyGraph(List<DependencyNode> nodes,
                       List<DependencyEdge> edges,
                       List<IssueRef> criticalPath,
                       Map<IssueStatus, List<DependencyNode>> grouped) {}
```

### 3.2 Status classification

Each OPEN issue is classified into one of three statuses:

| Status | Condition |
|--------|-----------|
| **BLOCKED** | Has at least one OPEN blocker |
| **UNBLOCKED** | Had blockers, all now CLOSED |
| **CLEAR** | Never had any declared blockers |

CLOSED issues are not classified — they appear only as resolved blockers in the graph.

### 3.3 Scope (D2, D6)

The graph is scoped to repos discovered under the workspace root. `DependencyService` receives the repo set from `FileWatcherService.currentModel(root).repos()`, extracts `owner/repo` from each `RepoInfo.remoteUrl()`, and uses those to filter `github_issue_cache` queries.

Cross-repo references to repos outside the workspace scope appear as `DependencyNode` instances with `issueState = "EXTERNAL"` — visible but unresolved.

## 4. DependencyService (D4)

Single `@ApplicationScoped` CDI bean. Builds the graph on demand and caches it with `GenerationCounter`-based invalidation (same pattern as `WorklogSummary` caching in `WorklogService`).

### 4.1 API

```java
@ApplicationScoped
public class DependencyService {

    DependencyGraph buildGraph(Path workspaceRoot)
    // Returns cached graph if GenerationCounter hasn't changed

    List<DependencyNode> blocked(Path workspaceRoot)
    // Convenience: nodes with status BLOCKED, sorted by blocker chain depth (deepest first)

    List<DependencyNode> unblocked(Path workspaceRoot)
    // Convenience: nodes with status UNBLOCKED

    List<IssueRef> criticalPath(Path workspaceRoot)
    // The longest chain of unresolved blocking dependencies
}
```

### 4.2 Graph construction

1. Resolve workspace repos → `owner/repo` set via `FileWatcherService`
2. Query `WorklogService.issueDependencyData(repos)` for all issues (OPEN + CLOSED)
3. Parse each issue body → extract `DependencyEdge` list
4. Build adjacency maps (blocks / blocked-by)
5. Classify each OPEN issue → BLOCKED / UNBLOCKED / CLEAR
6. Compute critical path (§5)
7. Cache result, keyed on `GenerationCounter` value

### 4.3 Cache invalidation

`GenerationCounter` increments when `worklog.db` mtime changes (which happens when enrichment runs `enrichment.py refresh`). Same mechanism that invalidates `WorklogSummary`. No additional watcher needed — the enrichment cache and the dependency graph share the same invalidation signal.

## 5. Critical Path (D5)

Longest chain of unresolved blocking dependencies, computed via topological sort on the subgraph of OPEN issues with OPEN blockers.

### 5.1 Algorithm

1. Build subgraph: only OPEN issues that are BLOCKED (have at least one OPEN blocker)
2. Compute in-degree for each node (number of issues this node blocks)
3. Topological sort via Kahn's algorithm — but track the longest path to each node
4. The longest path in the DAG is the critical path

If the graph has cycles (A blocks B blocks A), report the cycle as a finding rather than crashing. Cycles indicate data quality issues in issue bodies.

### 5.2 Output

Ordered list of `IssueRef` from root blocker to deepest blocked issue. Example:

```
Critical path (depth 4):
  #11 (OPEN) → blocks #19 → blocks #42 → blocks #55
```

## 6. REST Endpoint

### 6.1 Full graph

`GET /api/dependencies?root=<workspace-root>`

Returns:
```json
{
  "criticalPath": [
    { "number": 11, "repo": "Hortora/trellis", "title": "Auth migration" }
  ],
  "blocked": [
    {
      "number": 55, "repo": "Hortora/trellis", "title": "User management",
      "issueState": "OPEN", "status": "BLOCKED",
      "blockedBy": [{ "number": 42, "repo": "Hortora/trellis", "state": "OPEN" }],
      "blocking": []
    }
  ],
  "unblocked": [
    {
      "number": 19, "repo": "Hortora/trellis", "title": "Work Intelligence",
      "issueState": "OPEN", "status": "UNBLOCKED",
      "blockedBy": [{ "number": 11, "repo": "Hortora/trellis", "state": "CLOSED" }],
      "blocking": [{ "number": 42, "repo": "Hortora/trellis" }]
    }
  ],
  "clear": [
    {
      "number": 53, "repo": "Hortora/trellis", "title": "CrossRepoAdapter PR cache",
      "issueState": "OPEN", "status": "CLEAR",
      "blockedBy": [], "blocking": []
    }
  ],
  "external": [
    { "number": 282, "repo": "Hortora/soredium", "state": "EXTERNAL" }
  ],
  "stats": {
    "totalIssues": 12, "blocked": 3, "unblocked": 2, "clear": 7,
    "criticalPathDepth": 4, "externalRefs": 1
  }
}
```

### 6.2 Resource

`DependencyResource` — follows the same pattern as `IntelligenceResource`. Delegates to `DependencyService.buildGraph(root)`.

```java
@Path("/api/dependencies")
@ApplicationScoped
public class DependencyResource {
    @GET @Produces(MediaType.APPLICATION_JSON)
    public Response get(@QueryParam("root") String root) { ... }
}
```

## 7. Frontend Panel (D3)

`trellis-blockers-panel` — registered in `workbench-panels.ts` alongside existing panels.

### 7.1 Layout

Three-column kanban-style view:

```
┌─────────────────────────────────────────────────────────┐
│ Critical Path: #11 → #19 → #42 → #55  (depth 4)       │
├──────────────┬──────────────────┬────────────────────────┤
│ BLOCKED (3)  │ UNBLOCKED (2)    │ CLEAR (7)             │
├──────────────┼──────────────────┼────────────────────────┤
│ #55 User mgmt│ #19 Intelligence │ #53 CrossRepo cache   │
│  ← #42 (OPEN)│  ← #11 (CLOSED) │                       │
│  ← #11 (OPEN)│                  │ #60 Doc overhaul      │
│              │ #30 API refactor │                       │
│ #42 Auth     │  ← #28 (CLOSED) │ ...                   │
│  ← #11 (OPEN)│                  │                       │
│              │                  │                       │
│ #99 Deploy   │                  │                       │
│  ← soredium  │                  │                       │
│    #282 (EXT)│                  │                       │
└──────────────┴──────────────────┴────────────────────────┘
```

### 7.2 Visual design

- **BLOCKED column:** red left border, each card shows issue + blocker chain with blocker state colour-coded (OPEN = red, CLOSED = green, EXTERNAL = grey)
- **UNBLOCKED column:** green left border, each card shows issue + resolved blockers (all green)
- **CLEAR column:** neutral border, compact list (no blocker detail needed)
- **Critical path banner:** top bar highlighting the longest blocking chain, issue numbers clickable
- Sorted: BLOCKED by blocker chain depth (deepest first), UNBLOCKED by most-recently-resolved, CLEAR by issue number
- Stats bar: counts per column + critical path depth
- External references rendered with repo prefix and grey colour to indicate unresolved scope

### 7.3 Data loading

Fetches from `GET /api/dependencies?root=<workspaceRoot>`. Auto-refreshes on `workspace:repos` SSE topic (repo set changed) and on a 60s interval (same pattern as backlog panel).

### 7.4 Panel registration

Added to `workbench-panels.ts` as `blockers` panel with icon and title. Receives `workspaceRoot` property from the workbench shell.

## 8. Intelligence Integration

### 8.1 Fix EnrichmentAdapter

Current: hardcodes `"state", "OPEN"` for all issues and extracts blockers only from labels.

Fixed: inject `DependencyService`, use it to resolve actual blocker state:

```java
public void emitIssueEvents(Path workspaceRoot) {
    var graph = dependencyService.buildGraph(workspaceRoot);
    for (DependencyNode node : graph.nodes()) {
        var blockerData = node.blockedBy().stream()
            .map(ref -> Map.<String, Object>of(
                "number", ref.number(),
                "state", resolveState(ref, graph)))
            .toList();
        var data = Map.<String, Object>of(
            "issueNumber", node.ref().number(),
            "issueRepo", node.ref().repo(),
            "state", node.issueState(),
            "title", node.title(),
            "blockedBy", blockerData
        );
        cloudEventBus.fireAsync(TrellisCloudEvents.enrichmentIssue(data));
    }
}
```

This means `UnblockedWorkGanglion` will see `"state": "CLOSED"` for resolved blockers and fire `DETECTED` for issues whose blockers have all been resolved. No changes needed to the ganglion itself — it already checks `allResolved`.

### 8.2 ModelProvider integration

`DependencyService` does NOT get its own `ModelProvider`. The dependency graph is an operational view, not an intelligence signal. The intelligence layer reads it through `EnrichmentAdapter` → `UnblockedWorkGanglion` → `WorkIntelligenceModelProvider`. The dashboard reads it directly via the REST endpoint.

If MCP access to the graph is needed later, a `DependencyModelProvider` can be added — but the current design avoids premature wiring.

## 9. Decisions

See [decisions.md](decisions.md) for the full decision log. Summary:

| # | Decision |
|---|----------|
| D1 | Parse dependencies from cached issue bodies — inline refs, sections, epic checklists |
| D2 | Single-repo graph scope, cross-repo as external links |
| D3 | Three-column status view (Blocked/Unblocked/Clear) with critical path |
| D4 | Shared DependencyService feeds both dashboard and intelligence |
| D5 | Simple chain-length critical path via topological sort |
| D6 | Workspace root folder = "space" — scopes all queries |
| D7 | FileWatcherService for repo discovery (already exists) |

## 10. Trade-offs Acknowledged

- **D1:** Misses blockers declared in comments. Acceptable — body conventions are consistent and comment scanning can be added later without architecture change.
- **D2:** Cross-repo blockers appear as unresolved external links, not full graph nodes. Widening scope later only requires passing more repos to the query.
- **D3:** Doesn't show full graph topology. The critical path banner compensates — for deeper exploration, a future graph visualization can be added as an overlay.
- **D4:** DependencyService becomes a shared dependency. Changes to the graph model affect both consumers. Acceptable trade-off vs. data duplication.
- **D5:** Treats all issues as equal weight. A weighted critical path (via enrichment scale/complexity) can be layered in later.

## 11. Implementation Order

1. **Data layer:** `IssueDependencyData` record, `WorklogService.issueDependencyData()` query method
2. **Graph model:** records (`DependencyEdge`, `IssueRef`, `DependencyNode`, `DependencyGraph`)
3. **DependencyService:** body parsing, graph construction, critical path computation, caching
4. **REST endpoint:** `DependencyResource` at `/api/dependencies`
5. **Intelligence fix:** update `EnrichmentAdapter` to use `DependencyService` for real blocker state
6. **Frontend panel:** `trellis-blockers-panel` with three-column layout
7. **Panel registration:** add to `workbench-panels.ts`

Steps 1-4 are backend, independently testable. Step 5 fixes the intelligence gap. Steps 6-7 are frontend.

## References

- [EnrichmentAdapter.java] — current hardcoded blocker state (the gap this fixes)
- [UnblockedWorkGanglion.java] — detection logic (functional, needs real data from fixed adapter)
- [WorklogService.java] — database access layer (extended with dependency query)
- [WorkIntelligenceModelProvider.java] — intelligence presentation (unchanged)
- [FileWatcherService.java] + [WorkspaceScanner.java] — repo discovery (reused, not modified)
- [backlog-panel.ts] — reference for pages-data-table patterns
- [intelligence-panel.ts] — reference for severity-grouped display
- [workbench-panels.ts] — panel registration
- Hortora/trellis#54 — focal issue
- Hortora/trellis#19 — predecessor (Work Intelligence)
- Hortora/trellis#53 — related (CrossRepoAdapter data sourcing)
