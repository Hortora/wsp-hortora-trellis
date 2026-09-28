# Backlog Tree Table Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #71 — Backlog panel: tree table view for epic hierarchy
**Issue group:** #71

**Goal:** Replace the flat backlog table with a tree table that shows
parent/child issue hierarchy using GitHub's native sub-issues API.

**Architecture:** Three-layer change — enrichment script fetches
`subIssues` from GitHub GraphQL during cache refresh and stores
`parent_issue` in the SQLite cache; the sidecar reads it and includes
`parentKey` in `BacklogEntry`; the frontend passes `expandable` config
to `pages-data-table` which handles tree rendering natively.

**Tech Stack:** Python (enrichment.py), Java 21 / Quarkus (sidecar),
TypeScript / Lit (frontend), `pages-data-table` tree API

## Global Constraints

- Java 21 records — add field at end of record constructor
- `pages-data-table` `ExpandableConfig` requires `idColumn` and `parentColumn` as `ColumnId`
- `github_issue_cache` schema is in `soredium/scripts/worklog.py`
- Enrichment script is in `soredium/scripts/enrichment.py`
- All soredium changes go to `~/claude/hortora/soredium/`, not `~/.claude/skills/`

---

## Batch 1: Cache + Backend (sub-issue data pipeline)

### Task 1: Add `parent_issue` column to cache schema and enrichment refresh

**Files:**
- Modify: `~/claude/hortora/soredium/scripts/worklog.py:107-116` (schema)
- Modify: `~/claude/hortora/soredium/scripts/enrichment.py:208-247` (refresh_cache)
- Modify: `~/claude/hortora/soredium/scripts/enrichment.py:171-181` (upsert_cached_issue)
- Test: `~/claude/hortora/soredium/scripts/test_enrichment.py` (if exists, else inline verification)

**Interfaces:**
- Consumes: GitHub GraphQL API `subIssues` field
- Produces: `parent_issue TEXT` column in `github_issue_cache` rows — value is `"Owner/repo#N"` for children, `NULL` for root issues

- [ ] **Step 1: Add column to schema**

In `worklog.py`, add `parent_issue` to the `github_issue_cache` CREATE TABLE:

```sql
CREATE TABLE IF NOT EXISTS github_issue_cache (
    issue_number INTEGER NOT NULL,
    issue_repo   TEXT NOT NULL,
    title        TEXT,
    state        TEXT,
    labels       TEXT,
    body         TEXT,
    parent_issue TEXT,
    cached_at    TEXT NOT NULL,
    PRIMARY KEY (issue_number, issue_repo)
);
```

- [ ] **Step 2: Update `upsert_cached_issue` to accept `parent_issue`**

In `enrichment.py`, update the function signature and SQL:

```python
def upsert_cached_issue(conn: sqlite3.Connection, issue_number: int,
                        issue_repo: str, title: str, state: str,
                        labels: list[str], body: str,
                        parent_issue: str | None = None) -> None:
    conn.execute(
        "INSERT OR REPLACE INTO github_issue_cache "
        "(issue_number, issue_repo, title, state, labels, body, parent_issue, cached_at) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
        (issue_number, issue_repo, title, state,
         json.dumps(labels), body, parent_issue, _now()),
    )
    conn.commit()
```

- [ ] **Step 3: Fetch sub-issues during `refresh_cache`**

After the existing `gh issue list` call in `refresh_cache`, add a GraphQL
query to fetch sub-issue relationships:

```python
def _fetch_sub_issues(issue_repo: str) -> dict[int, int]:
    """Returns {child_number: parent_number} for all sub-issue relationships."""
    owner, repo = issue_repo.split("/")
    query = """
    query($owner: String!, $repo: String!, $cursor: String) {
      repository(owner: $owner, name: $repo) {
        issues(first: 100, states: OPEN, after: $cursor) {
          pageInfo { hasNextPage endCursor }
          nodes {
            number
            subIssues(first: 50) {
              nodes { number }
            }
          }
        }
      }
    }
    """
    child_to_parent: dict[int, int] = {}
    cursor = None
    for _ in range(10):  # max 1000 issues
        variables = json.dumps({"owner": owner, "repo": repo, "cursor": cursor})
        result = subprocess.run(
            ["gh", "api", "graphql", "-f", f"query={query}", "-f", f"variables={variables}"],
            capture_output=True, text=True, timeout=30,
        )
        if result.returncode != 0:
            break
        data = json.loads(result.stdout)
        issues_data = data["data"]["repository"]["issues"]
        for issue in issues_data["nodes"]:
            for child in issue.get("subIssues", {}).get("nodes", []):
                child_to_parent[child["number"]] = issue["number"]
        if not issues_data["pageInfo"]["hasNextPage"]:
            break
        cursor = issues_data["pageInfo"]["endCursor"]
    return child_to_parent
```

- [ ] **Step 4: Wire sub-issues into refresh_cache**

After the existing issue list loop in `refresh_cache`, fetch and apply
sub-issue relationships:

```python
    # After the INSERT OR REPLACE loop and DELETE cleanup:
    sub_issues = _fetch_sub_issues(issue_repo)
    if sub_issues:
        with conn:
            for child_num, parent_num in sub_issues.items():
                parent_key = f"{issue_repo}#{parent_num}"
                conn.execute(
                    "UPDATE github_issue_cache SET parent_issue=? "
                    "WHERE issue_number=? AND issue_repo=?",
                    (parent_key, child_num, issue_repo),
                )
```

- [ ] **Step 5: Add schema migration for existing databases**

In `worklog.py`, add an ALTER TABLE after the CREATE TABLE block (SQLite
ignores ALTER if column already exists when wrapped in try/except):

```python
# In the init_db function or schema section:
try:
    conn.execute("ALTER TABLE github_issue_cache ADD COLUMN parent_issue TEXT")
except sqlite3.OperationalError:
    pass  # column already exists
```

- [ ] **Step 6: Verify manually**

```bash
python3 ~/claude/hortora/soredium/scripts/enrichment.py refresh --repo Hortora/trellis
python3 -c "
import sqlite3
conn = sqlite3.connect('$HOME/.hortora/worklog.db')
conn.row_factory = sqlite3.Row
rows = conn.execute('SELECT issue_number, issue_repo, parent_issue FROM github_issue_cache WHERE parent_issue IS NOT NULL AND issue_repo=\"Hortora/trellis\"').fetchall()
for r in rows:
    print(f'#{r[\"issue_number\"]} ({r[\"issue_repo\"]}) parent={r[\"parent_issue\"]}')
print(f'Total with parents: {len(rows)}')
"
```

Expected: issues that are sub-issues of #2 show `parent_issue=Hortora/trellis#2`.

- [ ] **Step 7: Commit soredium changes**

```bash
git -C ~/claude/hortora/soredium add scripts/worklog.py scripts/enrichment.py
git -C ~/claude/hortora/soredium commit -m "feat: fetch GitHub sub-issues during cache refresh

Store parent_issue in github_issue_cache for tree table display.
Uses GraphQL subIssues field with cursor pagination.

Refs Hortora/trellis#71"
```

### Task 2: Add `parentKey` to `BacklogEntry` and `WorklogService`

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/worklog/BacklogEntry.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/worklog/WorklogService.java:242-280` (backlogEntries SQL + mapBacklogEntry)
- Test: `sidecar/src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java`

**Interfaces:**
- Consumes: `parent_issue` column from `github_issue_cache`
- Produces: `BacklogEntry.parentKey()` — `String` or `null`, serialised as JSON to the frontend

- [ ] **Step 1: Write failing test**

Add a test in `WorklogServiceTest` that verifies `parentKey` is populated
from the cache:

```java
@Test
void backlogEntryIncludesParentKey() {
    // Insert a parent and child into the test cache
    insertCachedIssue(conn, 2, "Org/repo", "Epic", "OPEN", "[]", "", null);
    insertCachedIssue(conn, 5, "Org/repo", "Child", "OPEN", "[]", "", "Org/repo#2");

    var entries = service.backlogEntries("Org/repo");
    var child = entries.stream()
        .filter(e -> e.issueNumber() == 5).findFirst().orElseThrow();
    assertThat(child.parentKey()).isEqualTo("Org/repo#2");

    var parent = entries.stream()
        .filter(e -> e.issueNumber() == 2).findFirst().orElseThrow();
    assertThat(parent.parentKey()).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest#backlogEntryIncludesParentKey
```

Expected: compilation error — `parentKey` not on `BacklogEntry` yet.

- [ ] **Step 3: Add `parentKey` to `BacklogEntry`**

Use `ide_edit_member` to update the record:

```java
public record BacklogEntry(
    int issueNumber,
    String issueRepo,
    String title,
    List<String> labels,
    String cachedAt,
    String strategicRole,
    String readiness,
    String decay,
    String blastRadius,
    String cohesion,
    String enrichedAt,
    String trajectoryNote,
    String trajectoryAt,
    String parentKey
) {}
```

- [ ] **Step 4: Update `backlogEntries` SQL to include `parent_issue`**

In `WorklogService.backlogEntries()`, add `c.parent_issue` to the SELECT:

```sql
SELECT c.issue_number, c.issue_repo, c.title, c.labels, c.cached_at,
       e.strategic_role, e.readiness, e.decay, e.blast_radius, e.cohesion, e.updated_at,
       t.note AS trajectory_note, t.created_at AS trajectory_at,
       c.parent_issue
FROM github_issue_cache c
...
```

- [ ] **Step 5: Update `mapBacklogEntry` to read `parent_issue`**

```java
private BacklogEntry mapBacklogEntry(ResultSet rs) throws SQLException {
    return new BacklogEntry(
            rs.getInt("issue_number"), rs.getString("issue_repo"),
            rs.getString("title"), parseLabels(rs.getString("labels")),
            rs.getString("cached_at"), rs.getString("strategic_role"),
            rs.getString("readiness"), rs.getString("decay"),
            rs.getString("blast_radius"), rs.getString("cohesion"),
            rs.getString("updated_at"), rs.getString("trajectory_note"),
            rs.getString("trajectory_at"), rs.getString("parent_issue"));
}
```

- [ ] **Step 6: Fix test helper — update `insertCachedIssue` to accept `parent_issue`**

Check the test helper method and add the `parent_issue` parameter.

- [ ] **Step 7: Run tests**

```bash
/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl .
```

Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/worklog/BacklogEntry.java \
       sidecar/src/main/java/io/hortora/trellis/worklog/WorklogService.java \
       sidecar/src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java
git commit -m "feat(#71): add parentKey to BacklogEntry for tree table hierarchy

Include parent_issue from github_issue_cache in backlog API response.

Refs #71"
```

---

## Batch 2: Frontend tree table

### Task 3: Wire tree table in backlog panel

**Files:**
- Modify: `sidecar/src/main/webui/src/views/backlog-panel.ts`
- Modify: `sidecar/src/main/webui/src/views/backlog-panel.test.ts`

**Interfaces:**
- Consumes: `BacklogEntry.parentKey` from `/api/backlog` JSON response
- Produces: Tree table rendering with expand/collapse in the backlog panel

- [ ] **Step 1: Write failing test**

In `backlog-panel.test.ts`, add test data with parent/child relationships
and a test for filtering with hierarchy:

```typescript
const TREE_ITEMS: BacklogItem[] = [
  { issueNumber: 2, issueRepo: 'Org/repo', title: 'Epic', labels: [], cachedAt: '2026-09-28T10:00:00Z', strategicRole: null, readiness: null, decay: null, blastRadius: null, cohesion: null, enrichedAt: null, trajectoryNote: null, trajectoryAt: null, parentKey: null },
  { issueNumber: 5, issueRepo: 'Org/repo', title: 'Child A', labels: [], cachedAt: '2026-09-28T10:00:00Z', strategicRole: 'quick-win', readiness: 'ready', decay: null, blastRadius: null, cohesion: null, enrichedAt: null, trajectoryNote: null, trajectoryAt: null, parentKey: 'Org/repo#2' },
  { issueNumber: 6, issueRepo: 'Org/repo', title: 'Child B', labels: [], cachedAt: '2026-09-28T10:00:00Z', strategicRole: null, readiness: null, decay: null, blastRadius: null, cohesion: null, enrichedAt: null, trajectoryNote: null, trajectoryAt: null, parentKey: 'Org/repo#2' },
  { issueNumber: 10, issueRepo: 'Org/repo', title: 'Standalone', labels: [], cachedAt: '2026-09-28T10:00:00Z', strategicRole: null, readiness: null, decay: null, blastRadius: null, cohesion: null, enrichedAt: null, trajectoryNote: null, trajectoryAt: null, parentKey: null },
];

it('includes parentKey in filter pass-through', () => {
  const result = applyFilters(TREE_ITEMS, {});
  expect(result).toHaveLength(4);
  expect(result.find(i => i.issueNumber === 5)?.parentKey).toBe('Org/repo#2');
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn vitest run src/views/backlog-panel.test.ts
```

Expected: FAIL — `parentKey` not on `BacklogItem` interface.

- [ ] **Step 3: Add `parentKey` to `BacklogItem` interface**

```typescript
export interface BacklogItem {
  // ... existing fields ...
  parentKey: string | null;
}
```

- [ ] **Step 4: Add `parentKey` to existing test data**

Add `parentKey: null` to all entries in the existing `ITEMS` test data
array so existing tests continue to compile.

- [ ] **Step 5: Add `parent` column and `expandable` config**

Add a `parent` column ID:

```typescript
const COL = {
  key: columnId('key'),
  parent: columnId('parent'),
  // ... rest unchanged
} as const;
```

In `_buildDataSet()`, add the parent column:

```typescript
{ id: COL.parent, type: ColumnType.TEXT, getValue: i => i.parentKey ?? '' },
```

In the `render()` method, add `expandable` and update `hiddenColumns`:

```typescript
<pages-data-table
  .embedded=${true}
  mode="paginated"
  .pageSize=${100}
  .dataSet=${this._buildDataSet()}
  .columnConfig=${this._columnConfig}
  .columnRenderers=${this._columnRenderers}
  .sortable=${true}
  .clientSort=${true}
  .getRowKey=${this._getRowKey}
  .hiddenColumns=${[COL.key, COL.parent] as any}
  .expandable=${{ idColumn: COL.key, parentColumn: COL.parent, defaultExpanded: 1 }}
  .emptyMessage=${'No backlog data.'}
  @row-activate=${this._handleRowActivate}
></pages-data-table>
```

- [ ] **Step 6: Run all backlog tests**

```bash
yarn vitest run src/views/backlog-panel.test.ts
```

Expected: all tests pass (including new tree test).

- [ ] **Step 7: Build and verify**

```bash
yarn build
```

Expected: compiles without errors (pre-existing unrelated errors excepted).

- [ ] **Step 8: Commit**

```bash
git add sidecar/src/main/webui/src/views/backlog-panel.ts \
       sidecar/src/main/webui/src/views/backlog-panel.test.ts
git commit -m "feat(#71): tree table view for epic hierarchy in backlog panel

Add parentKey to BacklogItem, wire expandable config to pages-data-table
for native tree rendering with expand/collapse. First level expanded by
default.

Closes #71"
```

## References

- [2026-09-28-backlog-tree-table-design.md] — design spec this plan implements
- `sidecar/src/main/webui/.casehub-packages/packages/pages-table/src/tree-builder.ts` — `ExpandableConfig`, `buildTreeIndex`
- `sidecar/src/main/java/io/hortora/trellis/worklog/WorklogService.java:242` — `backlogEntries()` SQL
- `sidecar/src/main/java/io/hortora/trellis/worklog/BacklogEntry.java` — record definition
- `sidecar/src/main/webui/src/views/backlog-panel.ts` — current flat table
- `~/claude/hortora/soredium/scripts/enrichment.py:208` — `refresh_cache`
- `~/claude/hortora/soredium/scripts/worklog.py:107` — cache schema
- GitHub #71 — focal issue
