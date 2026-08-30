# Dependency Graph + Blockers Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #54 — Dependency graph + blockers dashboard — critical path visibility
**Issue group:** #54

**Goal:** Build a dependency resolution layer that parses blocking relationships from cached GitHub issue bodies, computes critical paths, exposes a REST endpoint, fixes the intelligence adapter to use real blocker state, and provides a three-column blockers dashboard panel.

**Architecture:** A `DependencyService` CDI bean parses issue bodies from `github_issue_cache` to build an in-memory directed graph of blocking relationships. It resolves blocker state against the cache, classifies issues as BLOCKED/UNBLOCKED/CLEAR, and computes the critical path (longest chain of unresolved blockers). The graph feeds both a new REST endpoint (`/api/dependencies`) consumed by a frontend panel and the existing `EnrichmentAdapter` (fixed to emit real blocker state instead of hardcoding `"OPEN"`).

**Tech Stack:** Java 21 (records, sealed interfaces), Quarkus 3.x, SQLite (worklog.db), Lit (TypeScript), vitest

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Dependencies sub-package: `io.hortora.trellis.dependencies`
- Tests use JUnit 5 + SQLiteDataSource (no Quarkus test harness for unit tests)
- Frontend tests use vitest — test pure logic functions, not Lit rendering
- `github_issue_cache` columns: `issue_number`, `issue_repo`, `title`, `state`, `labels`, `body`, `cached_at`
- `BacklogEntry` record must NOT be modified — it serves the backlog panel and doesn't need bodies
- `GenerationCounter` for cache invalidation — same pattern as `WorklogSummary` caching
- IntelliJ MCP project path: `/Users/mdproctor/claude/hortora/trellis/sidecar`

---

## Batch 1: Data Layer + Graph Model

### Task 1: Issue Dependency Data Record and Query Method

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/IssueDependencyData.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/worklog/WorklogService.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java`

**Interfaces:**
- Consumes: `WorklogService` existing pattern (DataSource, checkFreshness, dbAvailable guard)
- Produces: `IssueDependencyData(int issueNumber, String issueRepo, String title, String state, String body)` record; `WorklogService.issueDependencyData(List<String> repos)` returning `List<IssueDependencyData>`

- [ ] **Step 1: Write the failing test**

Add to `WorklogServiceTest.java`:

```java
@Test
void issueDependencyDataReturnsAllStatesForRepos() {
    var data = service.issueDependencyData(List.of("Test/repo"));
    assertEquals(2, data.size());
    var open = data.stream().filter(d -> d.issueNumber() == 10).findFirst().orElseThrow();
    assertEquals("OPEN", open.state());
    assertEquals("Test/repo", open.issueRepo());
    var closed = data.stream().filter(d -> d.issueNumber() == 11).findFirst().orElseThrow();
    assertEquals("CLOSED", closed.state());
}

@Test
void issueDependencyDataFiltersByRepo() {
    var data = service.issueDependencyData(List.of("Nonexistent/repo"));
    assertEquals(0, data.size());
}

@Test
void issueDependencyDataIncludesBody() throws SQLException {
    try (var conn = service_ds.getConnection()) {
        conn.createStatement().execute(
            "UPDATE github_issue_cache SET body = 'blocked by #11' WHERE issue_number = 10");
    }
    var data = service.issueDependencyData(List.of("Test/repo"));
    var open = data.stream().filter(d -> d.issueNumber() == 10).findFirst().orElseThrow();
    assertEquals("blocked by #11", open.body());
}

@Test
void issueDependencyDataReturnsEmptyWhenDbUnavailable() {
    service.dbAvailable = false;
    var data = service.issueDependencyData(List.of("Test/repo"));
    assertEquals(0, data.size());
}
```

Note: the test setUp needs to expose `ds` as a field (`service_ds`) so the body-update test can insert data. Add `private SQLiteDataSource service_ds;` as a field and assign in setUp.

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest#issueDependencyDataReturnsAllStatesForRepos -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — method does not exist

- [ ] **Step 3: Create the IssueDependencyData record**

Create `sidecar/src/main/java/io/hortora/trellis/dependencies/IssueDependencyData.java`:

```java
package io.hortora.trellis.dependencies;

public record IssueDependencyData(int issueNumber, String issueRepo,
                                  String title, String state, String body) {}
```

- [ ] **Step 4: Implement the query method in WorklogService**

Add to `WorklogService`:

```java
public List<io.hortora.trellis.dependencies.IssueDependencyData> issueDependencyData(List<String> repos) {
    if (!dbAvailable || repos.isEmpty()) return List.of();
    checkFreshness();
    var placeholders = String.join(",", repos.stream().map(r -> "?").toList());
    var sql = "SELECT issue_number, issue_repo, title, state, body FROM github_issue_cache WHERE issue_repo IN (" + placeholders + ")";
    var results = new ArrayList<io.hortora.trellis.dependencies.IssueDependencyData>();
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(sql)) {
        for (int i = 0; i < repos.size(); i++) {
            stmt.setString(i + 1, repos.get(i));
        }
        try (var rs = stmt.executeQuery()) {
            while (rs.next()) {
                results.add(new io.hortora.trellis.dependencies.IssueDependencyData(
                    rs.getInt("issue_number"), rs.getString("issue_repo"),
                    rs.getString("title"), rs.getString("state"), rs.getString("body")));
            }
        }
    } catch (SQLException e) {
        LOG.warning("worklog query failed (issueDependencyData): " + e.getMessage());
        return List.of();
    }
    return results;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/dependencies/IssueDependencyData.java sidecar/src/main/java/io/hortora/trellis/worklog/WorklogService.java sidecar/src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java
git commit -m "feat(#54): add IssueDependencyData record and query method

Refs #54"
```

### Task 2: Graph Model Records and Body Parser

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/IssueRef.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyEdge.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/IssueStatus.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyNode.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyGraph.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyParser.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/dependencies/DependencyParserTest.java`

**Interfaces:**
- Consumes: `IssueDependencyData` from Task 1
- Produces:
  - `IssueRef(int number, String repo)`
  - `DependencyEdge(IssueRef blocked, IssueRef blocker)` — "blocked is blocked by blocker"
  - `IssueStatus` enum: `BLOCKED, UNBLOCKED, CLEAR`
  - `DependencyNode(IssueRef ref, String title, String issueState, IssueStatus status, List<IssueRef> blockedBy, List<IssueRef> blocking)`
  - `DependencyGraph(List<DependencyNode> nodes, List<DependencyEdge> edges, List<IssueRef> criticalPath, Map<IssueStatus, List<DependencyNode>> grouped, Map<IssueRef, String> issueStates)` — `issueStates` maps every known ref (including closed issues) to its state string ("OPEN", "CLOSED", "EXTERNAL")
  - `DependencyParser.parseEdges(int issueNumber, String issueRepo, String body)` returning `List<DependencyEdge>`

- [ ] **Step 1: Write failing tests for the body parser**

Create `sidecar/src/test/java/io/hortora/trellis/dependencies/DependencyParserTest.java`:

```java
package io.hortora.trellis.dependencies;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class DependencyParserTest {

    @Test
    void parsesInlineBlockedBy() {
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis",
            "This is blocked by #42 and needs work");
        assertEquals(1, edges.size());
        assertEquals(new IssueRef(55, "Hortora/trellis"), edges.getFirst().blocked());
        assertEquals(new IssueRef(42, "Hortora/trellis"), edges.getFirst().blocker());
    }

    @Test
    void parsesInlineDependsOn() {
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis",
            "This depends on #11");
        assertEquals(1, edges.size());
        assertEquals(new IssueRef(11, "Hortora/trellis"), edges.getFirst().blocker());
    }

    @Test
    void parsesCrossRepoBlockedBy() {
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis",
            "blocked by Hortora/soredium#282");
        assertEquals(1, edges.size());
        assertEquals(new IssueRef(282, "Hortora/soredium"), edges.getFirst().blocker());
    }

    @Test
    void parsesBlockedBySection() {
        var body = """
            ## Context
            Some context here.
            
            ## Blocked by
            - #42 — auth migration must land first
            - Hortora/soredium#282 — garden schema change
            
            ## References
            Something else.
            """;
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis", body);
        assertEquals(2, edges.size());
        assertTrue(edges.stream().anyMatch(e -> e.blocker().equals(new IssueRef(42, "Hortora/trellis"))));
        assertTrue(edges.stream().anyMatch(e -> e.blocker().equals(new IssueRef(282, "Hortora/soredium"))));
    }

    @Test
    void parsesDependenciesSection() {
        var body = """
            ## Dependencies
            - #11 — must land first
            """;
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis", body);
        assertEquals(1, edges.size());
        assertEquals(new IssueRef(11, "Hortora/trellis"), edges.getFirst().blocker());
    }

    @Test
    void parsesEpicChecklistAnnotation() {
        var body = """
            ## Children
            - [ ] #55 — user management (blocked by #42)
            - [x] #50 — logging overhaul
            - [ ] #60 — deploy pipeline (depends on #11)
            """;
        var edges = DependencyParser.parseEdges(0, "Hortora/trellis", body);
        assertEquals(2, edges.size());
        assertTrue(edges.stream().anyMatch(e ->
            e.blocked().equals(new IssueRef(55, "Hortora/trellis")) &&
            e.blocker().equals(new IssueRef(42, "Hortora/trellis"))));
        assertTrue(edges.stream().anyMatch(e ->
            e.blocked().equals(new IssueRef(60, "Hortora/trellis")) &&
            e.blocker().equals(new IssueRef(11, "Hortora/trellis"))));
    }

    @Test
    void deduplicatesEdges() {
        var body = """
            blocked by #42
            
            ## Blocked by
            - #42
            """;
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis", body);
        assertEquals(1, edges.size());
    }

    @Test
    void returnsEmptyForNullBody() {
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis", null);
        assertTrue(edges.isEmpty());
    }

    @Test
    void returnsEmptyForBodyWithNoDeps() {
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis",
            "Just a regular issue with no dependencies.");
        assertTrue(edges.isEmpty());
    }

    @Test
    void doesNotParseReferencesAsDependencies() {
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis",
            "See also #42 and #11 for context");
        assertTrue(edges.isEmpty());
    }

    @Test
    void parsesMultipleInlineBlockedBy() {
        var edges = DependencyParser.parseEdges(55, "Hortora/trellis",
            "blocked by #11 and #22");
        assertEquals(2, edges.size());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=DependencyParserTest`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Create the graph model records**

Create `IssueRef.java`:
```java
package io.hortora.trellis.dependencies;

public record IssueRef(int number, String repo) {}
```

Create `DependencyEdge.java`:
```java
package io.hortora.trellis.dependencies;

public record DependencyEdge(IssueRef blocked, IssueRef blocker) {}
```

Create `IssueStatus.java`:
```java
package io.hortora.trellis.dependencies;

public enum IssueStatus { BLOCKED, UNBLOCKED, CLEAR }
```

Create `DependencyNode.java`:
```java
package io.hortora.trellis.dependencies;

import java.util.List;

public record DependencyNode(IssueRef ref, String title, String issueState,
                             IssueStatus status, List<IssueRef> blockedBy,
                             List<IssueRef> blocking) {}
```

Create `DependencyGraph.java`:
```java
package io.hortora.trellis.dependencies;

import java.util.List;
import java.util.Map;

public record DependencyGraph(List<DependencyNode> nodes,
                              List<DependencyEdge> edges,
                              List<IssueRef> criticalPath,
                              Map<IssueStatus, List<DependencyNode>> grouped,
                              Map<IssueRef, String> issueStates) {}
```

- [ ] **Step 4: Implement DependencyParser**

Create `DependencyParser.java`:
```java
package io.hortora.trellis.dependencies;

import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public final class DependencyParser {

    private static final Pattern INLINE_BLOCKED_BY = Pattern.compile(
        "(?i)(?:blocked\\s+by|depends\\s+on)\\s+(.+?)(?:\\.|$|\\n)", Pattern.MULTILINE);
    private static final Pattern ISSUE_REF = Pattern.compile(
        "(?:([\\w.-]+/[\\w.-]+))?#(\\d+)");
    private static final Pattern SECTION_HEADER = Pattern.compile(
        "^##\\s+(Blocked by|Dependencies)\\s*$", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);
    private static final Pattern CHECKLIST_WITH_DEP = Pattern.compile(
        "^-\\s+\\[[ x]]\\s+#(\\d+)\\s+.*?\\((?:blocked by|depends on)\\s+(.+?)\\)",
        Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);

    private DependencyParser() {}

    public static List<DependencyEdge> parseEdges(int issueNumber, String issueRepo, String body) {
        if (body == null || body.isBlank()) return List.of();

        var seen = new LinkedHashSet<DependencyEdge>();
        var defaultRef = new IssueRef(issueNumber, issueRepo);

        parseInlineRefs(body, defaultRef, seen);
        parseSections(body, defaultRef, seen);
        parseChecklistAnnotations(body, issueRepo, seen);

        return List.copyOf(seen);
    }

    private static void parseInlineRefs(String body, IssueRef blocked, LinkedHashSet<DependencyEdge> edges) {
        Matcher m = INLINE_BLOCKED_BY.matcher(body);
        while (m.find()) {
            String fragment = m.group(1);
            Matcher refs = ISSUE_REF.matcher(fragment);
            while (refs.find()) {
                String repo = refs.group(1) != null ? refs.group(1) : blocked.repo();
                int number = Integer.parseInt(refs.group(2));
                edges.add(new DependencyEdge(blocked, new IssueRef(number, repo)));
            }
        }
    }

    private static void parseSections(String body, IssueRef blocked, LinkedHashSet<DependencyEdge> edges) {
        Matcher header = SECTION_HEADER.matcher(body);
        while (header.find()) {
            int start = header.end();
            int end = body.indexOf("\n##", start);
            if (end == -1) end = body.length();
            String section = body.substring(start, end);
            for (String line : section.split("\n")) {
                String trimmed = line.trim();
                if (!trimmed.startsWith("-")) continue;
                Matcher refs = ISSUE_REF.matcher(trimmed);
                while (refs.find()) {
                    String repo = refs.group(1) != null ? refs.group(1) : blocked.repo();
                    int number = Integer.parseInt(refs.group(2));
                    edges.add(new DependencyEdge(blocked, new IssueRef(number, repo)));
                }
            }
        }
    }

    private static void parseChecklistAnnotations(String body, String defaultRepo, LinkedHashSet<DependencyEdge> edges) {
        Matcher m = CHECKLIST_WITH_DEP.matcher(body);
        while (m.find()) {
            int childNumber = Integer.parseInt(m.group(1));
            var childRef = new IssueRef(childNumber, defaultRepo);
            String fragment = m.group(2);
            Matcher refs = ISSUE_REF.matcher(fragment);
            while (refs.find()) {
                String repo = refs.group(1) != null ? refs.group(1) : defaultRepo;
                int number = Integer.parseInt(refs.group(2));
                edges.add(new DependencyEdge(childRef, new IssueRef(number, repo)));
            }
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=DependencyParserTest`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/dependencies/ sidecar/src/test/java/io/hortora/trellis/dependencies/
git commit -m "feat(#54): graph model records and body parser for dependency extraction

Refs #54"
```

## Batch 2: DependencyService + Critical Path

### Task 3: DependencyService — Graph Construction and Critical Path

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyService.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/dependencies/DependencyServiceTest.java`

**Interfaces:**
- Consumes: `WorklogService.issueDependencyData(List<String>)` from Task 1, `DependencyParser.parseEdges(...)` from Task 2, `FileWatcherService.currentModel(Path)` returning `WorkspaceModel`, `RepoInfo.remoteUrl()` for owner/repo extraction, `GenerationCounter.current()` for cache invalidation
- Produces:
  - `DependencyService.buildGraph(Path workspaceRoot)` → `DependencyGraph`
  - `DependencyService.blocked(Path workspaceRoot)` → `List<DependencyNode>` (sorted by blocker chain depth)
  - `DependencyService.unblocked(Path workspaceRoot)` → `List<DependencyNode>`
  - `DependencyService.criticalPath(Path workspaceRoot)` → `List<IssueRef>`
  - `DependencyService.extractOwnerRepo(String remoteUrl)` → `String` (static, package-private)

- [ ] **Step 1: Write failing tests**

Create `sidecar/src/test/java/io/hortora/trellis/dependencies/DependencyServiceTest.java`:

```java
package io.hortora.trellis.dependencies;

import io.hortora.trellis.mcp.GenerationCounter;
import io.hortora.trellis.scanner.FileWatcherService;
import io.hortora.trellis.scanner.RepoInfo;
import io.hortora.trellis.scanner.WorkspaceModel;
import io.hortora.trellis.worklog.WorklogService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import org.sqlite.SQLiteDataSource;

import java.nio.file.Path;
import java.sql.SQLException;
import java.time.Instant;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class DependencyServiceTest {

    @TempDir Path tmpDir;
    private DependencyService service;
    private WorklogService worklogService;
    private FileWatcherService fileWatcherService;
    private GenerationCounter generation;

    @BeforeEach
    void setUp() throws SQLException {
        var dbPath = tmpDir.resolve("test.db");
        var ds = new SQLiteDataSource();
        ds.setUrl("jdbc:sqlite:" + dbPath);
        try (var conn = ds.getConnection()) {
            conn.createStatement().execute("""
                CREATE TABLE github_issue_cache (
                    issue_number INTEGER NOT NULL, issue_repo TEXT NOT NULL,
                    title TEXT, state TEXT, labels TEXT, body TEXT,
                    cached_at TEXT NOT NULL, PRIMARY KEY (issue_number, issue_repo))""");
            conn.createStatement().execute("PRAGMA user_version = 2");
        }
        generation = new GenerationCounter();
        worklogService = new WorklogService(ds, generation, dbPath);

        fileWatcherService = mock(FileWatcherService.class);
        var model = new WorkspaceModel(tmpDir, Instant.now(),
            List.of(new RepoInfo("trellis", tmpDir.resolve("trellis"), "main",
                "git@github.com:Hortora/trellis.git")),
            List.of(), List.of(), List.of());
        when(fileWatcherService.currentModel(tmpDir)).thenReturn(model);

        service = new DependencyService(worklogService, fileWatcherService, generation);
    }

    private void insertIssue(int number, String repo, String state, String body) throws SQLException {
        var dbPath = tmpDir.resolve("test.db");
        var ds = new SQLiteDataSource();
        ds.setUrl("jdbc:sqlite:" + dbPath);
        try (var conn = ds.getConnection()) {
            conn.createStatement().execute(
                "INSERT OR REPLACE INTO github_issue_cache VALUES (" +
                number + ",'" + repo + "','Issue " + number + "','" + state +
                "','[]','" + (body != null ? body.replace("'", "''") : "") +
                "','2026-08-30T10:00:00Z')");
        }
    }

    @Test
    void emptyGraphWhenNoIssues() {
        var graph = service.buildGraph(tmpDir);
        assertNotNull(graph);
        assertTrue(graph.nodes().isEmpty());
        assertTrue(graph.edges().isEmpty());
        assertTrue(graph.criticalPath().isEmpty());
    }

    @Test
    void classifiesUnblockedIssue() throws SQLException {
        insertIssue(55, "Hortora/trellis", "OPEN", "blocked by #42");
        insertIssue(42, "Hortora/trellis", "CLOSED", null);
        var graph = service.buildGraph(tmpDir);
        var node55 = graph.nodes().stream()
            .filter(n -> n.ref().number() == 55).findFirst().orElseThrow();
        assertEquals(IssueStatus.UNBLOCKED, node55.status());
    }

    @Test
    void classifiesBlockedIssue() throws SQLException {
        insertIssue(55, "Hortora/trellis", "OPEN", "blocked by #42");
        insertIssue(42, "Hortora/trellis", "OPEN", null);
        var graph = service.buildGraph(tmpDir);
        var node55 = graph.nodes().stream()
            .filter(n -> n.ref().number() == 55).findFirst().orElseThrow();
        assertEquals(IssueStatus.BLOCKED, node55.status());
    }

    @Test
    void classifiesClearIssue() throws SQLException {
        insertIssue(53, "Hortora/trellis", "OPEN", "No dependencies here");
        var graph = service.buildGraph(tmpDir);
        var node53 = graph.nodes().stream()
            .filter(n -> n.ref().number() == 53).findFirst().orElseThrow();
        assertEquals(IssueStatus.CLEAR, node53.status());
    }

    @Test
    void computesCriticalPath() throws SQLException {
        insertIssue(11, "Hortora/trellis", "OPEN", null);
        insertIssue(19, "Hortora/trellis", "OPEN", "blocked by #11");
        insertIssue(42, "Hortora/trellis", "OPEN", "blocked by #19");
        insertIssue(55, "Hortora/trellis", "OPEN", "blocked by #42");
        var graph = service.buildGraph(tmpDir);
        assertEquals(4, graph.criticalPath().size());
        assertEquals(11, graph.criticalPath().getFirst().number());
        assertEquals(55, graph.criticalPath().getLast().number());
    }

    @Test
    void criticalPathIgnoresResolvedBlockers() throws SQLException {
        insertIssue(11, "Hortora/trellis", "CLOSED", null);
        insertIssue(19, "Hortora/trellis", "OPEN", "blocked by #11");
        insertIssue(42, "Hortora/trellis", "OPEN", "blocked by #19");
        var graph = service.buildGraph(tmpDir);
        assertTrue(graph.criticalPath().size() <= 1,
            "Critical path should not include resolved chains");
    }

    @Test
    void groupsByStatus() throws SQLException {
        insertIssue(55, "Hortora/trellis", "OPEN", "blocked by #42");
        insertIssue(42, "Hortora/trellis", "OPEN", null);
        insertIssue(19, "Hortora/trellis", "OPEN", "blocked by #11");
        insertIssue(11, "Hortora/trellis", "CLOSED", null);
        insertIssue(53, "Hortora/trellis", "OPEN", null);
        var graph = service.buildGraph(tmpDir);
        assertEquals(1, graph.grouped().get(IssueStatus.BLOCKED).size());
        assertEquals(1, graph.grouped().get(IssueStatus.UNBLOCKED).size());
        assertEquals(2, graph.grouped().get(IssueStatus.CLEAR).size());
    }

    @Test
    void excludesClosedIssuesFromNodes() throws SQLException {
        insertIssue(11, "Hortora/trellis", "CLOSED", null);
        insertIssue(19, "Hortora/trellis", "OPEN", "blocked by #11");
        var graph = service.buildGraph(tmpDir);
        assertTrue(graph.nodes().stream().noneMatch(n -> n.ref().number() == 11),
            "Closed issues should not appear as nodes");
    }

    @Test
    void cachedGraphReturnsSameInstanceWhenUnchanged() throws SQLException {
        insertIssue(53, "Hortora/trellis", "OPEN", null);
        var g1 = service.buildGraph(tmpDir);
        var g2 = service.buildGraph(tmpDir);
        assertSame(g1, g2);
    }

    @Test
    void cacheInvalidatesOnGenerationChange() throws SQLException {
        insertIssue(53, "Hortora/trellis", "OPEN", null);
        var g1 = service.buildGraph(tmpDir);
        generation.increment();
        var g2 = service.buildGraph(tmpDir);
        assertNotSame(g1, g2);
    }

    @Test
    void extractsOwnerRepoFromSshUrl() {
        assertEquals("Hortora/trellis",
            DependencyService.extractOwnerRepo("git@github.com:Hortora/trellis.git"));
    }

    @Test
    void extractsOwnerRepoFromHttpsUrl() {
        assertEquals("Hortora/trellis",
            DependencyService.extractOwnerRepo("https://github.com/Hortora/trellis.git"));
    }

    @Test
    void returnsNullForUnrecognisedUrl() {
        assertNull(DependencyService.extractOwnerRepo("https://other.host/repo"));
    }

    @Test
    void blockedReturnsSortedByDepth() throws SQLException {
        insertIssue(11, "Hortora/trellis", "OPEN", null);
        insertIssue(19, "Hortora/trellis", "OPEN", "blocked by #11");
        insertIssue(42, "Hortora/trellis", "OPEN", "blocked by #19");
        var blocked = service.blocked(tmpDir);
        assertEquals(2, blocked.size());
        assertEquals(42, blocked.getFirst().ref().number());
        assertEquals(19, blocked.get(1).ref().number());
    }

    @Test
    void returnsEmptyGraphWhenNoFileWatcher() {
        when(fileWatcherService.currentModel(tmpDir)).thenReturn(null);
        var graph = service.buildGraph(tmpDir);
        assertTrue(graph.nodes().isEmpty());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=DependencyServiceTest`
Expected: FAIL — DependencyService does not exist

- [ ] **Step 3: Implement DependencyService**

Create `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyService.java`:

```java
package io.hortora.trellis.dependencies;

import io.hortora.trellis.mcp.GenerationCounter;
import io.hortora.trellis.scanner.FileWatcherService;
import io.hortora.trellis.scanner.RepoInfo;
import io.hortora.trellis.worklog.WorklogService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.nio.file.Path;
import java.util.*;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import java.util.stream.Collectors;

@ApplicationScoped
public class DependencyService {

    private static final Pattern SSH_URL = Pattern.compile("git@github\\.com:(.+/.+?)(?:\\.git)?$");
    private static final Pattern HTTPS_URL = Pattern.compile("https://github\\.com/(.+/.+?)(?:\\.git)?$");
    private static final DependencyGraph EMPTY = new DependencyGraph(
        List.of(), List.of(), List.of(), Map.of(
            IssueStatus.BLOCKED, List.of(),
            IssueStatus.UNBLOCKED, List.of(),
            IssueStatus.CLEAR, List.of()),
        Map.of());

    private final WorklogService worklogService;
    private final FileWatcherService fileWatcherService;
    private final GenerationCounter generation;

    private volatile DependencyGraph cachedGraph;
    private volatile long cachedGeneration = -1;

    @Inject
    public DependencyService(WorklogService worklogService,
                             FileWatcherService fileWatcherService,
                             GenerationCounter generation) {
        this.worklogService = worklogService;
        this.fileWatcherService = fileWatcherService;
        this.generation = generation;
    }

    public DependencyGraph buildGraph(Path workspaceRoot) {
        long gen = generation.current();
        if (cachedGraph != null && gen == cachedGeneration) return cachedGraph;

        var model = fileWatcherService.currentModel(workspaceRoot);
        if (model == null) return EMPTY;

        var repos = model.repos().stream()
            .map(r -> extractOwnerRepo(r.remoteUrl()))
            .filter(Objects::nonNull)
            .toList();
        if (repos.isEmpty()) return EMPTY;

        var issueData = worklogService.issueDependencyData(repos);
        if (issueData.isEmpty()) return EMPTY;

        var repoSet = new HashSet<>(repos);
        var stateMap = new HashMap<IssueRef, String>();
        var titleMap = new HashMap<IssueRef, String>();
        for (var d : issueData) {
            var ref = new IssueRef(d.issueNumber(), d.issueRepo());
            stateMap.put(ref, d.state());
            titleMap.put(ref, d.title() != null ? d.title() : "");
        }

        var allEdges = new ArrayList<DependencyEdge>();
        for (var d : issueData) {
            allEdges.addAll(DependencyParser.parseEdges(d.issueNumber(), d.issueRepo(), d.body()));
        }

        var blockedByMap = new LinkedHashMap<IssueRef, List<IssueRef>>();
        var blockingMap = new LinkedHashMap<IssueRef, List<IssueRef>>();
        for (var edge : allEdges) {
            blockedByMap.computeIfAbsent(edge.blocked(), k -> new ArrayList<>()).add(edge.blocker());
            blockingMap.computeIfAbsent(edge.blocker(), k -> new ArrayList<>()).add(edge.blocked());
        }

        var nodes = new ArrayList<DependencyNode>();
        for (var d : issueData) {
            if (!"OPEN".equals(d.state())) continue;
            var ref = new IssueRef(d.issueNumber(), d.issueRepo());
            var blockers = blockedByMap.getOrDefault(ref, List.of());
            var blocking = blockingMap.getOrDefault(ref, List.of());

            IssueStatus status;
            if (blockers.isEmpty()) {
                status = IssueStatus.CLEAR;
            } else {
                boolean allResolved = blockers.stream()
                    .allMatch(b -> "CLOSED".equals(stateMap.getOrDefault(b, "EXTERNAL")));
                status = allResolved ? IssueStatus.UNBLOCKED : IssueStatus.BLOCKED;
            }
            nodes.add(new DependencyNode(ref, titleMap.getOrDefault(ref, ""),
                d.state(), status, List.copyOf(blockers), List.copyOf(blocking)));
        }

        var grouped = nodes.stream().collect(Collectors.groupingBy(
            DependencyNode::status, () -> new EnumMap<>(IssueStatus.class), Collectors.toList()));
        for (var s : IssueStatus.values()) grouped.putIfAbsent(s, List.of());

        var criticalPath = computeCriticalPath(nodes, stateMap);

        var graph = new DependencyGraph(List.copyOf(nodes), List.copyOf(allEdges),
            criticalPath, Map.copyOf(grouped), Map.copyOf(stateMap));
        cachedGraph = graph;
        cachedGeneration = gen;
        return graph;
    }

    public List<DependencyNode> blocked(Path workspaceRoot) {
        var graph = buildGraph(workspaceRoot);
        return graph.grouped().getOrDefault(IssueStatus.BLOCKED, List.of()).stream()
            .sorted(Comparator.<DependencyNode, Integer>comparing(n -> blockerChainDepth(n, graph))
                .reversed())
            .toList();
    }

    public List<DependencyNode> unblocked(Path workspaceRoot) {
        return buildGraph(workspaceRoot).grouped()
            .getOrDefault(IssueStatus.UNBLOCKED, List.of());
    }

    public List<IssueRef> criticalPath(Path workspaceRoot) {
        return buildGraph(workspaceRoot).criticalPath();
    }

    private int blockerChainDepth(DependencyNode node, DependencyGraph graph) {
        var nodeMap = graph.nodes().stream()
            .collect(Collectors.toMap(DependencyNode::ref, n -> n));
        return chainDepth(node.ref(), nodeMap, new HashSet<>());
    }

    private int chainDepth(IssueRef ref, Map<IssueRef, DependencyNode> nodeMap, Set<IssueRef> visited) {
        if (!visited.add(ref)) return 0;
        var node = nodeMap.get(ref);
        if (node == null) return 0;
        int max = 0;
        for (var blocker : node.blockedBy()) {
            max = Math.max(max, 1 + chainDepth(blocker, nodeMap, visited));
        }
        return max;
    }

    private List<IssueRef> computeCriticalPath(List<DependencyNode> nodes,
                                               Map<IssueRef, String> stateMap) {
        var blocked = nodes.stream()
            .filter(n -> n.status() == IssueStatus.BLOCKED)
            .collect(Collectors.toMap(DependencyNode::ref, n -> n));
        if (blocked.isEmpty()) return List.of();

        var allRefs = new HashSet<>(blocked.keySet());
        for (var node : blocked.values()) {
            for (var b : node.blockedBy()) {
                if ("OPEN".equals(stateMap.getOrDefault(b, "EXTERNAL"))) allRefs.add(b);
            }
        }

        var longestTo = new HashMap<IssueRef, Integer>();
        var predOn = new HashMap<IssueRef, IssueRef>();
        for (var ref : allRefs) longestTo.put(ref, 0);

        boolean changed = true;
        int iterations = 0;
        while (changed && iterations < allRefs.size()) {
            changed = false;
            iterations++;
            for (var node : blocked.values()) {
                for (var blocker : node.blockedBy()) {
                    if (!"OPEN".equals(stateMap.getOrDefault(blocker, "EXTERNAL"))) continue;
                    int newDist = longestTo.getOrDefault(blocker, 0) + 1;
                    if (newDist > longestTo.getOrDefault(node.ref(), 0)) {
                        longestTo.put(node.ref(), newDist);
                        predOn.put(node.ref(), blocker);
                        changed = true;
                    }
                }
            }
        }

        var deepest = longestTo.entrySet().stream()
            .max(Comparator.comparingInt(Map.Entry::getValue))
            .map(Map.Entry::getKey).orElse(null);
        if (deepest == null || longestTo.get(deepest) == 0) return List.of();

        var path = new ArrayList<IssueRef>();
        var current = deepest;
        var visited = new HashSet<IssueRef>();
        while (current != null && visited.add(current)) {
            path.add(current);
            current = predOn.get(current);
        }
        Collections.reverse(path);
        return List.copyOf(path);
    }

    static String extractOwnerRepo(String remoteUrl) {
        if (remoteUrl == null) return null;
        Matcher m = SSH_URL.matcher(remoteUrl);
        if (m.find()) return m.group(1);
        m = HTTPS_URL.matcher(remoteUrl);
        if (m.find()) return m.group(1);
        return null;
    }
}
```

- [ ] **Step 4: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=DependencyServiceTest,DependencyParserTest,WorklogServiceTest`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/dependencies/ sidecar/src/test/java/io/hortora/trellis/dependencies/ sidecar/src/main/java/io/hortora/trellis/worklog/WorklogService.java sidecar/src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java
git commit -m "feat(#54): DependencyService with graph construction and critical path

Refs #54"
```

## Batch 3: REST Endpoint + Intelligence Fix

### Task 4: DependencyResource REST Endpoint

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyResource.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/dependencies/DependencyResourceTest.java`

**Interfaces:**
- Consumes: `DependencyService.buildGraph(Path)` from Task 3
- Produces: `GET /api/dependencies?root=<workspace-root>` returning JSON with `blocked`, `unblocked`, `clear`, `criticalPath`, `external`, `stats` fields

- [ ] **Step 1: Write failing test**

Create `sidecar/src/test/java/io/hortora/trellis/dependencies/DependencyResourceTest.java`:

```java
package io.hortora.trellis.dependencies;

import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class DependencyResourceTest {

    @Test
    void returnsBadRequestWithoutRoot() {
        var service = mock(DependencyService.class);
        var resource = new DependencyResource(service);
        var response = resource.get(null);
        assertEquals(400, response.getStatus());
    }

    @Test
    void returnsBadRequestForBlankRoot() {
        var service = mock(DependencyService.class);
        var resource = new DependencyResource(service);
        var response = resource.get("  ");
        assertEquals(400, response.getStatus());
    }

    @Test
    void returnsGraphData() {
        var service = mock(DependencyService.class);
        var blocked = new DependencyNode(new IssueRef(55, "R"), "Title", "OPEN",
            IssueStatus.BLOCKED, List.of(new IssueRef(42, "R")), List.of());
        var clear = new DependencyNode(new IssueRef(53, "R"), "Clear", "OPEN",
            IssueStatus.CLEAR, List.of(), List.of());
        var graph = new DependencyGraph(List.of(blocked, clear), List.of(),
            List.of(new IssueRef(42, "R"), new IssueRef(55, "R")),
            Map.of(IssueStatus.BLOCKED, List.of(blocked),
                   IssueStatus.UNBLOCKED, List.of(),
                   IssueStatus.CLEAR, List.of(clear)),
            Map.of(new IssueRef(42, "R"), "OPEN", new IssueRef(55, "R"), "OPEN",
                   new IssueRef(53, "R"), "OPEN"));
        when(service.buildGraph(Path.of("/root"))).thenReturn(graph);

        var resource = new DependencyResource(service);
        var response = resource.get("/root");
        assertEquals(200, response.getStatus());
        assertNotNull(response.getEntity());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=DependencyResourceTest`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement DependencyResource**

Create `sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyResource.java`:

```java
package io.hortora.trellis.dependencies;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@Path("/api/dependencies")
@ApplicationScoped
public class DependencyResource {

    private final DependencyService service;

    @Inject
    public DependencyResource(DependencyService service) {
        this.service = service;
    }

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response get(@QueryParam("root") String root) {
        if (root == null || root.isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST).build();
        }
        var graph = service.buildGraph(java.nio.file.Path.of(root));
        var result = new LinkedHashMap<String, Object>();
        result.put("criticalPath", graph.criticalPath().stream()
            .map(this::refToMap).toList());
        result.put("blocked", graph.grouped()
            .getOrDefault(IssueStatus.BLOCKED, List.of()).stream()
            .map(this::nodeToMap).toList());
        result.put("unblocked", graph.grouped()
            .getOrDefault(IssueStatus.UNBLOCKED, List.of()).stream()
            .map(this::nodeToMap).toList());
        result.put("clear", graph.grouped()
            .getOrDefault(IssueStatus.CLEAR, List.of()).stream()
            .map(this::nodeToMap).toList());
        result.put("stats", Map.of(
            "totalIssues", graph.nodes().size(),
            "blocked", graph.grouped().getOrDefault(IssueStatus.BLOCKED, List.of()).size(),
            "unblocked", graph.grouped().getOrDefault(IssueStatus.UNBLOCKED, List.of()).size(),
            "clear", graph.grouped().getOrDefault(IssueStatus.CLEAR, List.of()).size(),
            "criticalPathDepth", graph.criticalPath().size()));
        return Response.ok(result).build();
    }

    private Map<String, Object> nodeToMap(DependencyNode node) {
        var map = new LinkedHashMap<String, Object>();
        map.put("number", node.ref().number());
        map.put("repo", node.ref().repo());
        map.put("title", node.title());
        map.put("issueState", node.issueState());
        map.put("status", node.status().name());
        map.put("blockedBy", node.blockedBy().stream().map(b -> refWithState(b, graph.issueStates())).toList());
        map.put("blocking", node.blocking().stream().map(this::refToMap).toList());
        return map;
    }

    private Map<String, Object> refToMap(IssueRef ref) {
        return Map.of("number", ref.number(), "repo", ref.repo());
    }

    private Map<String, Object> refWithState(IssueRef ref, Map<IssueRef, String> issueStates) {
        var state = issueStates.getOrDefault(ref, "EXTERNAL");
        return Map.of("number", ref.number(), "repo", ref.repo(), "state", state);
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=DependencyResourceTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/dependencies/DependencyResource.java sidecar/src/test/java/io/hortora/trellis/dependencies/DependencyResourceTest.java
git commit -m "feat(#54): REST endpoint GET /api/dependencies

Refs #54"
```

### Task 5: Fix EnrichmentAdapter to Use Real Blocker State

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/intelligence/EnrichmentAdapter.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/intelligence/IntelligenceSweepJob.java`
- Modify: `sidecar/src/test/java/io/hortora/trellis/intelligence/EnrichmentAdapterTest.java`

**Interfaces:**
- Consumes: `DependencyService.buildGraph(Path)` from Task 3, `DependencyGraph.nodes()`, `DependencyNode.blockedBy()`, `DependencyNode.ref()`
- Produces: Updated `EnrichmentAdapter.emitIssueEvents(Path workspaceRoot)` that emits real blocker state from the dependency graph. `UnblockedWorkGanglion` (unchanged) now sees `"CLOSED"` for resolved blockers.

- [ ] **Step 1: Write failing tests for new behaviour**

Update `EnrichmentAdapterTest.java` — add tests for the new graph-based approach:

```java
@Test
void emitsRealBlockerStateFromGraph() {
    var graphNode = new DependencyNode(
        new IssueRef(19, "Hortora/trellis"), "Intelligence", "OPEN",
        IssueStatus.UNBLOCKED,
        List.of(new IssueRef(11, "Hortora/trellis")),
        List.of());
    var graph = new DependencyGraph(List.of(graphNode), List.of(),
        List.of(), Map.of(
            IssueStatus.BLOCKED, List.of(),
            IssueStatus.UNBLOCKED, List.of(graphNode),
            IssueStatus.CLEAR, List.of()),
        Map.of(new IssueRef(19, "Hortora/trellis"), "OPEN",
               new IssueRef(11, "Hortora/trellis"), "CLOSED"));

    var stateMap = Map.of(new IssueRef(11, "Hortora/trellis"), "CLOSED");

    var events = new ArrayList<CloudEvent>();
    var adapter = new EnrichmentAdapter(event -> events.add(event));

    adapter.emitFromGraph(graph, stateMap);

    assertEquals(1, events.size());
    // Verify the blocker state is CLOSED, not hardcoded OPEN
    var data = new ObjectMapper().readValue(
        events.getFirst().getData().toBytes(), Map.class);
    var blockers = (List<Map<String, Object>>) data.get("blockedBy");
    assertEquals("CLOSED", blockers.getFirst().get("state"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=EnrichmentAdapterTest#emitsRealBlockerStateFromGraph`
Expected: FAIL — method does not exist

- [ ] **Step 3: Refactor EnrichmentAdapter**

Update `EnrichmentAdapter` to accept `DependencyService` and emit events with real blocker state:

The adapter needs a new method `emitFromGraph(DependencyGraph, Map<IssueRef, String>)` that replaces the hardcoded `"OPEN"` state. The existing `emitIssueEvents()` method is updated to take a `Path workspaceRoot` parameter and delegate to `DependencyService`.

Key changes:
- Add `DependencyService` as a constructor dependency
- New `emitIssueEvents(Path workspaceRoot)` builds graph, then iterates nodes emitting events with real blocker state from the graph's state map
- Keep `extractBlockers(BacklogEntry)` as a static fallback for label-based extraction (backwards compat)
- The old no-arg `emitIssueEvents()` is removed

- [ ] **Step 4: Update IntelligenceSweepJob**

`IntelligenceSweepJob.sweep()` calls `enrichmentAdapter.emitIssueEvents()` — update to pass workspace root. The sweep job needs to discover workspace roots. It already walks `~/claude` for `.plan` files — extract workspace roots from the same walk, or use `FileWatcherService.allModels()`.

```java
// In sweep():
var models = fileWatcherService.allModels();
for (var model : models) {
    enrichmentAdapter.emitIssueEvents(model.root());
}
```

Add `FileWatcherService` as a dependency to `IntelligenceSweepJob`.

- [ ] **Step 5: Run all intelligence tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=EnrichmentAdapterTest,UnblockedWorkGanglionTest,WorkIntelligenceModelProviderTest`
Expected: ALL PASS

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/intelligence/EnrichmentAdapter.java sidecar/src/main/java/io/hortora/trellis/intelligence/IntelligenceSweepJob.java sidecar/src/test/java/io/hortora/trellis/intelligence/EnrichmentAdapterTest.java
git commit -m "fix(#54): EnrichmentAdapter uses real blocker state from DependencyService

Refs #54"
```

## Batch 4: Frontend Panel

### Task 6: Blockers Dashboard Panel + Registration

**Files:**
- Create: `sidecar/src/main/webui/src/views/blockers-panel.ts`
- Create: `sidecar/src/main/webui/src/views/blockers-panel.test.ts`
- Modify: `sidecar/src/main/webui/src/components/workbench-panels.ts`

**Interfaces:**
- Consumes: `GET /api/dependencies?root=<workspaceRoot>` from Task 4
- Produces: `<trellis-blockers-panel>` Lit component registered in `workbench-panels.ts` as `blockers` panel

- [ ] **Step 1: Write failing tests for pure logic functions**

Create `sidecar/src/main/webui/src/views/blockers-panel.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { classifyNodes, criticalPathDepth } from './blockers-panel';
import type { DependencyNode, GraphResponse } from './blockers-panel';

const BLOCKED_NODE: DependencyNode = {
  number: 55, repo: 'R', title: 'User mgmt', issueState: 'OPEN',
  status: 'BLOCKED', blockedBy: [{ number: 42, repo: 'R' }], blocking: [],
};
const UNBLOCKED_NODE: DependencyNode = {
  number: 19, repo: 'R', title: 'Intelligence', issueState: 'OPEN',
  status: 'UNBLOCKED', blockedBy: [{ number: 11, repo: 'R' }], blocking: [],
};
const CLEAR_NODE: DependencyNode = {
  number: 53, repo: 'R', title: 'CrossRepo', issueState: 'OPEN',
  status: 'CLEAR', blockedBy: [], blocking: [],
};

describe('classifyNodes', () => {
  it('groups nodes by status', () => {
    const { blocked, unblocked, clear } = classifyNodes([BLOCKED_NODE, UNBLOCKED_NODE, CLEAR_NODE]);
    expect(blocked).toHaveLength(1);
    expect(blocked[0].number).toBe(55);
    expect(unblocked).toHaveLength(1);
    expect(clear).toHaveLength(1);
  });

  it('returns empty arrays for no nodes', () => {
    const { blocked, unblocked, clear } = classifyNodes([]);
    expect(blocked).toHaveLength(0);
    expect(unblocked).toHaveLength(0);
    expect(clear).toHaveLength(0);
  });
});

describe('criticalPathDepth', () => {
  it('returns 0 for empty path', () => {
    expect(criticalPathDepth([])).toBe(0);
  });

  it('returns length of path', () => {
    expect(criticalPathDepth([
      { number: 11, repo: 'R' },
      { number: 19, repo: 'R' },
      { number: 42, repo: 'R' },
    ])).toBe(3);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd sidecar/src/main/webui test -- --run blockers-panel.test`
Expected: FAIL — module does not exist

- [ ] **Step 3: Implement the blockers panel**

Create `sidecar/src/main/webui/src/views/blockers-panel.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';

export interface IssueRefData {
  number: number;
  repo: string;
  state?: string;
}

export interface DependencyNode {
  number: number;
  repo: string;
  title: string;
  issueState: string;
  status: 'BLOCKED' | 'UNBLOCKED' | 'CLEAR';
  blockedBy: IssueRefData[];
  blocking: IssueRefData[];
}

export interface GraphResponse {
  criticalPath: IssueRefData[];
  blocked: DependencyNode[];
  unblocked: DependencyNode[];
  clear: DependencyNode[];
  stats: {
    totalIssues: number;
    blocked: number;
    unblocked: number;
    clear: number;
    criticalPathDepth: number;
  };
}

export function classifyNodes(nodes: DependencyNode[]): {
  blocked: DependencyNode[];
  unblocked: DependencyNode[];
  clear: DependencyNode[];
} {
  const blocked: DependencyNode[] = [];
  const unblocked: DependencyNode[] = [];
  const clear: DependencyNode[] = [];
  for (const n of nodes) {
    if (n.status === 'BLOCKED') blocked.push(n);
    else if (n.status === 'UNBLOCKED') unblocked.push(n);
    else clear.push(n);
  }
  return { blocked, unblocked, clear };
}

export function criticalPathDepth(path: IssueRefData[]): number {
  return path.length;
}

@customElement('trellis-blockers-panel')
export class TrellisBlockersPanel extends LitElement {

  @property() workspaceRoot = '';
  @state() private _data: GraphResponse | null = null;
  @state() private _loading = true;
  @state() private _error: string | null = null;

  private _refreshInterval: ReturnType<typeof setInterval> | null = null;

  static override styles = css`
    :host { display: flex; flex-direction: column; height: 100%; font-family: system-ui, -apple-system, sans-serif; }

    .critical-path {
      padding: 8px 16px; background: #1a1a2e; border-bottom: 1px solid #333;
      font-size: 12px; display: flex; align-items: center; gap: 8px; flex-shrink: 0;
    }
    .critical-path-label { color: #f87171; font-weight: 600; text-transform: uppercase; font-size: 11px; letter-spacing: 0.05em; }
    .critical-path-chain { color: #ccc; font-family: monospace; }
    .critical-path-chain .arrow { color: #555; margin: 0 4px; }
    .critical-path-chain .issue { color: #60a5fa; }
    .critical-path-depth { color: #666; margin-left: auto; font-size: 11px; }

    .stats-bar {
      display: flex; gap: 16px; padding: 8px 16px;
      border-bottom: 1px solid #333; font-size: 12px; flex-shrink: 0;
    }
    .stat { display: flex; align-items: center; gap: 4px; }
    .stat-count { font-weight: 600; }
    .stat-count.blocked { color: #f87171; }
    .stat-count.unblocked { color: #4ade80; }
    .stat-count.clear { color: #9ca3af; }

    .columns {
      display: flex; flex: 1; min-height: 0; overflow: hidden;
    }
    .column {
      flex: 1; display: flex; flex-direction: column; overflow-y: auto;
      border-right: 1px solid #2a2a2a; padding: 8px;
    }
    .column:last-child { border-right: none; }
    .column-header {
      font-size: 11px; font-weight: 600; text-transform: uppercase;
      letter-spacing: 0.05em; padding: 4px 8px; margin-bottom: 8px;
    }
    .column-header.blocked { color: #f87171; }
    .column-header.unblocked { color: #4ade80; }
    .column-header.clear { color: #9ca3af; }

    .card {
      padding: 8px 12px; margin-bottom: 6px; border-radius: 4px;
      background: var(--vscode-editor-background, #252525);
      border-left: 3px solid;
    }
    .card.blocked { border-color: #f87171; }
    .card.unblocked { border-color: #4ade80; }
    .card.clear { border-color: #555; }

    .card-title { font-size: 13px; font-weight: 500; color: #e0e0e0; }
    .card-number { font-family: monospace; color: #60a5fa; margin-right: 6px; }
    .card-blockers { margin-top: 4px; font-size: 11px; }
    .blocker { display: flex; align-items: center; gap: 4px; padding: 1px 0; }
    .blocker-arrow { color: #555; }
    .blocker-ref { font-family: monospace; }
    .blocker-ref.open { color: #f87171; }
    .blocker-ref.closed { color: #4ade80; }
    .blocker-ref.external { color: #888; }

    .empty { color: #666; padding: 32px; text-align: center; font-style: italic; }
    .error { color: #f87171; padding: 16px; }
  `;

  override connectedCallback() {
    super.connectedCallback();
    this._load();
    this._refreshInterval = setInterval(() => this._load(), 60_000);
  }

  override disconnectedCallback() {
    super.disconnectedCallback();
    if (this._refreshInterval) {
      clearInterval(this._refreshInterval);
      this._refreshInterval = null;
    }
  }

  override updated(changed: Map<PropertyKey, unknown>) {
    if (changed.has('workspaceRoot') && this.workspaceRoot) this._load();
  }

  private async _load() {
    if (!this.workspaceRoot) return;
    this._loading = true;
    try {
      const resp = await fetch(`/api/dependencies?root=${encodeURIComponent(this.workspaceRoot)}`);
      if (resp.ok) {
        this._data = await resp.json();
        this._error = null;
      } else {
        this._error = `HTTP ${resp.status}`;
      }
    } catch (e) {
      this._error = `Failed: ${e}`;
    }
    this._loading = false;
  }

  override render() {
    if (this._loading && !this._data) return html`<div class="empty">Loading dependencies...</div>`;
    if (this._error && !this._data) return html`<div class="error">${this._error}</div>`;
    if (!this._data) return html`<div class="empty">No dependency data.</div>`;

    const { blocked, unblocked, clear, criticalPath, stats } = this._data;

    return html`
      ${criticalPath.length > 0 ? html`
        <div class="critical-path">
          <span class="critical-path-label">Critical Path</span>
          <span class="critical-path-chain">
            ${criticalPath.map((ref, i) => html`${i > 0 ? html`<span class="arrow">→</span>` : nothing}<span class="issue">#${ref.number}</span>`)}
          </span>
          <span class="critical-path-depth">depth ${criticalPath.length}</span>
        </div>
      ` : nothing}

      <div class="stats-bar">
        <div class="stat"><span class="stat-count blocked">${stats.blocked}</span> blocked</div>
        <div class="stat"><span class="stat-count unblocked">${stats.unblocked}</span> unblocked</div>
        <div class="stat"><span class="stat-count clear">${stats.clear}</span> clear</div>
        <div style="margin-left:auto;color:#666;font-size:11px">${stats.totalIssues} total</div>
      </div>

      <div class="columns">
        <div class="column">
          <div class="column-header blocked">Blocked (${blocked.length})</div>
          ${blocked.length === 0
            ? html`<div class="empty">No blocked issues</div>`
            : blocked.map(n => this._renderCard(n, 'blocked'))}
        </div>
        <div class="column">
          <div class="column-header unblocked">Unblocked (${unblocked.length})</div>
          ${unblocked.length === 0
            ? html`<div class="empty">No recently unblocked</div>`
            : unblocked.map(n => this._renderCard(n, 'unblocked'))}
        </div>
        <div class="column">
          <div class="column-header clear">Clear (${clear.length})</div>
          ${clear.length === 0
            ? html`<div class="empty">No clear issues</div>`
            : clear.map(n => this._renderCard(n, 'clear'))}
        </div>
      </div>
    `;
  }

  private _renderCard(node: DependencyNode, cls: string) {
    return html`
      <div class="card ${cls}">
        <div class="card-title">
          <span class="card-number">#${node.number}</span>${node.title}
        </div>
        ${node.blockedBy.length > 0 ? html`
          <div class="card-blockers">
            ${node.blockedBy.map(b => html`
              <div class="blocker">
                <span class="blocker-arrow">←</span>
                <span class="blocker-ref ${this._blockerClass(b)}">${b.repo !== node.repo ? `${b.repo}` : ''}#${b.number}</span>
              </div>
            `)}
          </div>
        ` : nothing}
      </div>
    `;
  }

  private _blockerClass(blocker: IssueRefData): string {
    if (blocker.state === 'CLOSED') return 'closed';
    if (blocker.state === 'OPEN') return 'open';
    return 'external';
  }
}
```

- [ ] **Step 4: Register the panel in workbench-panels.ts**

Add import and entries:

In imports section:
```typescript
import '../views/blockers-panel.js';
```

In `PANEL_TAGS`:
```typescript
blockers:    'trellis-blockers-panel',
```

In `DOCK_PANELS`:
```typescript
{ key: 'blockers',     label: 'Blockers',     icon: '\u{1F6A7}', content: hostPanel('blockers') },
```

Place the blockers entry after `backlog` in the DOCK_PANELS array.

- [ ] **Step 5: Run frontend tests**

Run: `yarn --cwd sidecar/src/main/webui test -- --run`
Expected: ALL PASS

- [ ] **Step 6: Run full backend test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: ALL PASS

- [ ] **Step 7: Build frontend to verify compilation**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: Build succeeds

- [ ] **Step 8: Commit**

```bash
git add sidecar/src/main/webui/src/views/blockers-panel.ts sidecar/src/main/webui/src/views/blockers-panel.test.ts sidecar/src/main/webui/src/components/workbench-panels.ts
git commit -m "feat(#54): blockers dashboard panel with three-column layout

Refs #54"
```

### Task 7: Update CLAUDE.md with new endpoints and conventions

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: completed implementation from Tasks 1-6
- Produces: updated project conventions documenting the new endpoint, panel, and service

- [ ] **Step 1: Add DependencyService and endpoint to CLAUDE.md conventions**

Add to the Key Conventions section:

```markdown
- `DependencyService` — builds in-memory dependency graph from `github_issue_cache` bodies, caches with `GenerationCounter`, scopes to workspace repos via `FileWatcherService`
- `GET /api/dependencies?root=...` — dependency graph with blocked/unblocked/clear classification and critical path
- Blockers panel (`trellis-blockers-panel`) — three-column kanban view (Blocked/Unblocked/Clear) with critical path banner, scoped to workspace repos
- `DependencyParser` — extracts blocking relationships from issue bodies (inline refs, `## Blocked by` sections, epic checklist annotations)
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add dependency graph conventions to CLAUDE.md

Refs #54"
```

## References

- [specs/issue-54-dependency-graph-blockers-dashboard/2026-08-30-dependency-graph-blockers-design.md] — design spec this plan implements
- [sidecar/src/main/java/io/hortora/trellis/intelligence/EnrichmentAdapter.java] — hardcoded blocker state (gap being fixed)
- [sidecar/src/main/java/io/hortora/trellis/intelligence/UnblockedWorkGanglion.java] — detection logic (unchanged, benefits from fix)
- [sidecar/src/main/java/io/hortora/trellis/worklog/WorklogService.java] — database access layer (extended)
- [sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherService.java] — repo discovery (reused)
- [sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java] — workspace model (reused)
- [sidecar/src/main/java/io/hortora/trellis/mcp/GenerationCounter.java] — cache invalidation
- [sidecar/src/main/webui/src/views/backlog-panel.ts] — reference for panel patterns
- [sidecar/src/main/webui/src/views/intelligence-panel.ts] — reference for severity-grouped display
- [sidecar/src/main/webui/src/components/workbench-panels.ts] — panel registration
- [sidecar/src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java] — test setup pattern (SQLite + schema)
- [sidecar/src/test/java/io/hortora/trellis/intelligence/EnrichmentAdapterTest.java] — existing adapter tests
- [GitHub #54] — focal issue
- [GitHub #19] — predecessor (Work Intelligence)
