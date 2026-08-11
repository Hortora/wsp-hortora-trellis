# Worklog DB Reader Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #42 — Worklog DB reader — expose soredium work state through the model tree
**Issue group:** #42

**Goal:** Expose soredium's worklog.db lifecycle data through a JDBC service, REST endpoints, and the MCP model tree.

**Architecture:** Three components share one data source following the terminal pattern. `WorklogService` is the single JDBC reader for worklog.db (plus .plan filesystem reads). `WorklogResource` serves REST endpoints. `WorklogModelProvider` implements the `ModelProvider` SPI. The existing `BacklogResource` delegates to `WorklogService` instead of querying directly.

**Tech Stack:** Java 21, Quarkus 3.x, SQLite JDBC (org.xerial:sqlite-jdbc), quarkus-mcp-server

## Global Constraints

- Java 21 records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- All timestamps are ISO-8601 strings — pass through as-is, no `Instant` parsing
- SQLite test databases use `jdbc:sqlite:<tmpdir>/test.db` — never `:memory:` (GE-20260801-1148df)
- Normalize paths with `Path.resolve()` for macOS `/tmp` → `/private/tmp` (GE-20260730-e942d8)
- Read-only connection to worklog.db — trellis never writes
- Use `ide_move_file` for package moves, `ide_refactor_rename` for renames, `ide_insert_member`/`ide_replace_member` for code edits

---

### Task 1: Package reorganization and records

Move shared infrastructure from `backlog` to `worklog` package. Create the new domain records.

**Files:**
- Move: `src/main/java/io/hortora/trellis/backlog/WorklogDataSourceProducer.java` → `src/main/java/io/hortora/trellis/worklog/WorklogDataSourceProducer.java` (use `ide_move_file`)
- Move: `src/main/java/io/hortora/trellis/backlog/BacklogEntry.java` → `src/main/java/io/hortora/trellis/worklog/BacklogEntry.java` (use `ide_move_file`)
- Modify: `src/main/java/io/hortora/trellis/backlog/BacklogResource.java` — update imports
- Create: `src/main/java/io/hortora/trellis/worklog/WorklogEvent.java`
- Create: `src/main/java/io/hortora/trellis/worklog/WorkItem.java`
- Create: `src/main/java/io/hortora/trellis/worklog/WorkItemIssue.java`
- Create: `src/main/java/io/hortora/trellis/worklog/SlotInfo.java`
- Create: `src/main/java/io/hortora/trellis/worklog/PlanState.java`
- Test: `src/test/java/io/hortora/trellis/worklog/WorklogRecordsTest.java`

**Interfaces:**
- Consumes: nothing (foundational task)
- Produces: `WorklogEvent(long id, String timestamp, String eventType, Long workItemId, Long slotId, String repoPath, String metadata)`, `WorkItem(long id, String branch, String state, String location, Long slotId, String createdAt, String repoPath, String githubRepo, List<WorkItemIssue> issues)`, `WorkItemIssue(int issueNumber, String issueRepo, boolean isPrimary)`, `SlotInfo(long id, int slotNumber, String familyRoot, String state, String createdAt, String archivedAt)`, `PlanState(String activeIssue, int completed, int total)`, `BacklogEntry` (unchanged shape, new package), `WorklogDataSourceProducer` (unchanged logic, new package + `getDbPath()` accessor)

- [ ] **Step 1: Move WorklogDataSourceProducer to worklog package**

Use `ide_move_file`:
- Source: `src/main/java/io/hortora/trellis/backlog/WorklogDataSourceProducer.java`
- Target: `src/main/java/io/hortora/trellis/worklog/WorklogDataSourceProducer.java`

This auto-updates the package declaration and all import references (including `BacklogResource`'s `@Inject` and qualifier import).

- [ ] **Step 2: Move BacklogEntry to worklog package**

Use `ide_move_file`:
- Source: `src/main/java/io/hortora/trellis/backlog/BacklogEntry.java`
- Target: `src/main/java/io/hortora/trellis/worklog/BacklogEntry.java`

This auto-updates `BacklogResource`'s import and return type references.

- [ ] **Step 3: Add getDbPath() to WorklogDataSourceProducer**

Add to `WorklogDataSourceProducer.java` using `ide_insert_member`. Store the resolved path during init and expose it:

Add a field: `private Path resolvedDbPath;`

In `init()`, after resolving the path string, store it: `this.resolvedDbPath = path;`

Add accessor:

```java
public Path getDbPath() {
    return resolvedDbPath;
}
```

- [ ] **Step 4: Verify BacklogResource compiles after moves**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml compile -q`
Expected: BUILD SUCCESS. BacklogResource's imports should now reference `io.hortora.trellis.worklog.WorklogDataSourceProducer` and `io.hortora.trellis.worklog.BacklogEntry`.

- [ ] **Step 4: Create domain records**

Create `src/main/java/io/hortora/trellis/worklog/WorklogEvent.java`:

```java
package io.hortora.trellis.worklog;

public record WorklogEvent(long id, String timestamp, String eventType,
                            Long workItemId, Long slotId, String repoPath,
                            String metadata) {}
```

Create `src/main/java/io/hortora/trellis/worklog/WorkItemIssue.java`:

```java
package io.hortora.trellis.worklog;

public record WorkItemIssue(int issueNumber, String issueRepo, boolean isPrimary) {}
```

Create `src/main/java/io/hortora/trellis/worklog/WorkItem.java`:

```java
package io.hortora.trellis.worklog;

import java.util.List;

public record WorkItem(long id, String branch, String state, String location,
                       Long slotId, String createdAt, String repoPath,
                       String githubRepo, List<WorkItemIssue> issues) {}
```

Create `src/main/java/io/hortora/trellis/worklog/SlotInfo.java`:

```java
package io.hortora.trellis.worklog;

public record SlotInfo(long id, int slotNumber, String familyRoot, String state,
                       String createdAt, String archivedAt) {}
```

Create `src/main/java/io/hortora/trellis/worklog/PlanState.java`:

```java
package io.hortora.trellis.worklog;

public record PlanState(String activeIssue, int completed, int total) {}
```

- [ ] **Step 5: Write record round-trip test**

Create `src/test/java/io/hortora/trellis/worklog/WorklogRecordsTest.java`:

```java
package io.hortora.trellis.worklog;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class WorklogRecordsTest {

    @Test
    void worklogEventFieldsAccessible() {
        var event = new WorklogEvent(1, "2026-08-11T10:00:00Z", "work-start",
                42L, null, "/path/to/repo", "{\"key\":\"value\"}");
        assertEquals(1, event.id());
        assertEquals("work-start", event.eventType());
        assertNull(event.slotId());
    }

    @Test
    void workItemIncludesIssues() {
        var issues = List.of(
                new WorkItemIssue(42, "Hortora/trellis", true),
                new WorkItemIssue(43, "Hortora/trellis", false));
        var item = new WorkItem(1, "issue-42-worklog", "active", "primary",
                null, "2026-08-11T10:00:00Z", "/path", "Hortora/trellis", issues);
        assertEquals(2, item.issues().size());
        assertTrue(item.issues().get(0).isPrimary());
    }

    @Test
    void planStatePositionTracking() {
        var plan = new PlanState("#42", 2, 6);
        assertEquals("#42", plan.activeIssue());
        assertEquals(2, plan.completed());
        assertEquals(6, plan.total());
    }

    @Test
    void slotInfoFieldsAccessible() {
        var slot = new SlotInfo(1, 3, "/family/root", "active",
                "2026-08-11T10:00:00Z", null);
        assertEquals(3, slot.slotNumber());
        assertEquals("active", slot.state());
        assertNull(slot.archivedAt());
    }
}
```

- [ ] **Step 6: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogRecordsTest -q`
Expected: 4 tests PASS

- [ ] **Step 7: Run existing BacklogResourceTest to verify no regression**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=BacklogResourceTest -q`
Expected: All existing tests PASS (imports updated by ide_move_file)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/
git -C /Users/mdproctor/claude/hortora/trellis commit -m "refactor(#42): move backlog infrastructure to worklog package, add domain records

Refs #42"
```

---

### Task 2: WorklogService — core query layer

The central service bean with all JDBC queries, .plan parsing, schema version check, freshness detection, and summary cache.

**Files:**
- Create: `src/main/java/io/hortora/trellis/worklog/WorklogService.java`
- Test: `src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java`

**Interfaces:**
- Consumes: `WorklogDataSourceProducer` (`@WorklogDS DataSource`, `isDbAvailable()`), `GenerationCounter` (`increment()`), domain records from Task 1
- Produces: `WorklogService.recentEvents(String since, String type, int limit): List<WorklogEvent>`, `WorklogService.activeWork(): List<WorkItem>`, `WorklogService.workItemTimeline(String branch, String repoPath): List<WorklogEvent>`, `WorklogService.slotStatus(String familyRoot): List<SlotInfo>`, `WorklogService.backlogEntries(String repo): List<BacklogEntry>`, `WorklogService.planPosition(Path workspaceRoot): PlanState`, `WorklogService.summary(Path workspaceRoot): WorklogSummary`, `WorklogService.isDbAvailable(): boolean`

- [ ] **Step 1: Write test for recentEvents**

Create `src/test/java/io/hortora/trellis/worklog/WorklogServiceTest.java`:

```java
package io.hortora.trellis.worklog;

import io.hortora.trellis.mcp.GenerationCounter;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import org.sqlite.SQLiteDataSource;

import java.nio.file.Path;
import java.sql.Connection;
import java.sql.SQLException;

import static org.junit.jupiter.api.Assertions.*;

class WorklogServiceTest {

    @TempDir
    Path tmpDir;

    private Path dbPath;
    private WorklogService service;
    private GenerationCounter generation;
    private Connection seedConn;

    @BeforeEach
    void setUp() throws SQLException {
        dbPath = tmpDir.resolve("test-worklog.db");
        var ds = new SQLiteDataSource();
        ds.setUrl("jdbc:sqlite:" + dbPath);

        seedConn = ds.getConnection();
        seedConn.createStatement().execute("PRAGMA journal_mode=WAL");
        seedConn.createStatement().execute("PRAGMA foreign_keys=ON");
        createSchema(seedConn);
        seedData(seedConn);

        generation = new GenerationCounter();
        service = new WorklogService(ds, generation, dbPath);
    }

    private void createSchema(Connection conn) throws SQLException {
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS repos (
                id INTEGER PRIMARY KEY, path TEXT UNIQUE NOT NULL,
                workspace TEXT, family_root TEXT, github_repo TEXT, project_type TEXT)""");
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS slots (
                id INTEGER PRIMARY KEY, slot_number INTEGER NOT NULL,
                family_root TEXT NOT NULL, state TEXT NOT NULL DEFAULT 'active',
                created_at TEXT NOT NULL, archived_at TEXT,
                UNIQUE(slot_number, family_root))""");
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS work_items (
                id INTEGER PRIMARY KEY, branch TEXT NOT NULL,
                repo_id INTEGER NOT NULL REFERENCES repos(id),
                state TEXT NOT NULL DEFAULT 'active', location TEXT NOT NULL DEFAULT 'primary',
                slot_id INTEGER REFERENCES slots(id), work_path TEXT,
                created_at TEXT NOT NULL, ended_at TEXT,
                UNIQUE(branch, repo_id))""");
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS work_item_issues (
                work_item_id INTEGER NOT NULL REFERENCES work_items(id),
                issue_number INTEGER NOT NULL, issue_repo TEXT NOT NULL,
                is_primary INTEGER NOT NULL DEFAULT 0,
                PRIMARY KEY (work_item_id, issue_number, issue_repo))""");
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS events (
                id INTEGER PRIMARY KEY, timestamp TEXT NOT NULL,
                event_type TEXT NOT NULL, work_item_id INTEGER REFERENCES work_items(id),
                slot_id INTEGER REFERENCES slots(id), repo_path TEXT, metadata TEXT)""");
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS issue_enrichment (
                issue_number INTEGER NOT NULL, issue_repo TEXT NOT NULL,
                strategic_role TEXT, readiness TEXT, decay TEXT,
                blast_radius TEXT, cohesion TEXT, updated_at TEXT NOT NULL,
                PRIMARY KEY (issue_number, issue_repo))""");
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS trajectory_notes (
                id INTEGER PRIMARY KEY, issue_number INTEGER NOT NULL,
                issue_repo TEXT NOT NULL, note TEXT NOT NULL,
                source_branch TEXT, created_at TEXT NOT NULL)""");
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS github_issue_cache (
                issue_number INTEGER NOT NULL, issue_repo TEXT NOT NULL,
                title TEXT, state TEXT, labels TEXT, body TEXT,
                cached_at TEXT NOT NULL,
                PRIMARY KEY (issue_number, issue_repo))""");
        conn.createStatement().execute("PRAGMA user_version = 2");
    }

    private void seedData(Connection conn) throws SQLException {
        conn.createStatement().execute(
            "INSERT INTO repos (id, path, github_repo) VALUES (1, '/repo/a', 'Org/repoA')");
        conn.createStatement().execute(
            "INSERT INTO work_items (id, branch, repo_id, state, location, created_at) VALUES " +
            "(1, 'issue-42-worklog', 1, 'active', 'primary', '2026-08-11T10:00:00Z')," +
            "(2, 'issue-43-done', 1, 'ended', 'primary', '2026-08-10T10:00:00Z')");
        conn.createStatement().execute(
            "INSERT INTO work_item_issues VALUES (1, 42, 'Hortora/trellis', 1)," +
            "(1, 44, 'Hortora/trellis', 0)");
        conn.createStatement().execute(
            "INSERT INTO events (id, timestamp, event_type, work_item_id, repo_path) VALUES " +
            "(1, '2026-08-11T10:00:00Z', 'work-start', 1, '/repo/a')," +
            "(2, '2026-08-11T11:00:00Z', 'work-continue', 1, '/repo/a')," +
            "(3, '2026-08-10T10:00:00Z', 'work-start', 2, '/repo/a')," +
            "(4, '2026-08-10T12:00:00Z', 'work-end', 2, '/repo/a')");
        conn.createStatement().execute(
            "INSERT INTO slots (id, slot_number, family_root, state, created_at) VALUES " +
            "(1, 7, '/family/root', 'active', '2026-08-11T09:00:00Z')");
    }

    @AfterEach
    void tearDown() throws SQLException {
        if (seedConn != null) seedConn.close();
    }

    @Test
    void recentEventsReturnsDescending() {
        var events = service.recentEvents(null, null, 10);
        assertEquals(4, events.size());
        assertEquals("work-continue", events.get(0).eventType());
        assertEquals("work-start", events.get(1).eventType());
    }

    @Test
    void recentEventsFiltersByType() {
        var events = service.recentEvents(null, "work-start", 10);
        assertEquals(2, events.size());
        assertTrue(events.stream().allMatch(e -> "work-start".equals(e.eventType())));
    }

    @Test
    void recentEventsFiltersBySince() {
        var events = service.recentEvents("2026-08-11T00:00:00Z", null, 10);
        assertEquals(2, events.size());
    }

    @Test
    void recentEventsRespectsLimit() {
        var events = service.recentEvents(null, null, 2);
        assertEquals(2, events.size());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest -q`
Expected: FAIL — `WorklogService` class does not exist

- [ ] **Step 3: Create WorklogService with recentEvents**

Create `src/main/java/io/hortora/trellis/worklog/WorklogService.java`:

```java
package io.hortora.trellis.worklog;

import io.hortora.trellis.mcp.GenerationCounter;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import javax.sql.DataSource;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.attribute.FileTime;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

@ApplicationScoped
public class WorklogService {

    private static final java.util.logging.Logger LOG =
            java.util.logging.Logger.getLogger(WorklogService.class.getName());
    private static final com.fasterxml.jackson.databind.ObjectMapper MAPPER =
            new com.fasterxml.jackson.databind.ObjectMapper();

    private final DataSource dataSource;
    private final GenerationCounter generation;
    private final Path dbPath;
    private final AtomicReference<FileTime> lastMtime = new AtomicReference<>();
    volatile boolean dbAvailable;

    @Inject
    public WorklogService(@WorklogDataSourceProducer.WorklogDS DataSource dataSource,
                          GenerationCounter generation,
                          WorklogDataSourceProducer producer) {
        this.dataSource = dataSource;
        this.generation = generation;
        this.dbPath = producer.getDbPath();
        this.dbAvailable = producer.isDbAvailable();
    }

    WorklogService(DataSource dataSource, GenerationCounter generation, Path dbPath) {
        this.dataSource = dataSource;
        this.generation = generation;
        this.dbPath = dbPath;
        this.dbAvailable = true;
    }

    public boolean isDbAvailable() {
        return dbAvailable;
    }

    void checkFreshness() {
        if (!dbAvailable || dbPath == null) return;
        try {
            var mtime = Files.getLastModifiedTime(dbPath);
            var prev = lastMtime.getAndSet(mtime);
            if (prev != null && !prev.equals(mtime)) {
                generation.increment();
            }
        } catch (IOException ignored) {}
    }

    public List<WorklogEvent> recentEvents(String since, String type, int limit) {
        if (!dbAvailable) return List.of();
        checkFreshness();
        var clauses = new ArrayList<String>();
        var params = new ArrayList<Object>();
        if (since != null && !since.isBlank()) {
            clauses.add("timestamp >= ?");
            params.add(since);
        }
        if (type != null && !type.isBlank()) {
            clauses.add("event_type = ?");
            params.add(type);
        }
        var where = clauses.isEmpty() ? "" : " WHERE " + String.join(" AND ", clauses);
        var sql = "SELECT * FROM events" + where + " ORDER BY id DESC LIMIT ?";
        params.add(limit);

        var results = new ArrayList<WorklogEvent>();
        try (var conn = dataSource.getConnection();
             var stmt = conn.prepareStatement(sql)) {
            for (int i = 0; i < params.size(); i++) {
                stmt.setObject(i + 1, params.get(i));
            }
            try (var rs = stmt.executeQuery()) {
                while (rs.next()) {
                    results.add(mapEvent(rs));
                }
            }
        } catch (SQLException e) {
            LOG.warning("worklog query failed (recentEvents): " + e.getMessage());
            return List.of();
        }
        return results;
    }

    private WorklogEvent mapEvent(ResultSet rs) throws SQLException {
        return new WorklogEvent(
                rs.getLong("id"),
                rs.getString("timestamp"),
                rs.getString("event_type"),
                rs.getObject("work_item_id") != null ? rs.getLong("work_item_id") : null,
                rs.getObject("slot_id") != null ? rs.getLong("slot_id") : null,
                rs.getString("repo_path"),
                rs.getString("metadata"));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest -q`
Expected: 4 tests PASS

- [ ] **Step 5: Add tests for activeWork with two-pass join**

Append to `WorklogServiceTest.java`:

```java
@Test
void activeWorkExcludesEnded() {
    var items = service.activeWork();
    assertEquals(1, items.size());
    assertEquals("issue-42-worklog", items.get(0).branch());
    assertEquals("active", items.get(0).state());
}

@Test
void activeWorkIncludesIssues() {
    var items = service.activeWork();
    assertEquals(1, items.size());
    var issues = items.get(0).issues();
    assertEquals(2, issues.size());
    assertTrue(issues.stream().anyMatch(i -> i.issueNumber() == 42 && i.isPrimary()));
    assertTrue(issues.stream().anyMatch(i -> i.issueNumber() == 44 && !i.isPrimary()));
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest#activeWork* -q`
Expected: FAIL — `activeWork()` method does not exist

- [ ] **Step 7: Implement activeWork with two-pass strategy**

Add to `WorklogService.java` using `ide_insert_member`:

```java
public List<WorkItem> activeWork() {
    if (!dbAvailable) return List.of();
    checkFreshness();
    var items = new ArrayList<WorkItem>();
    var ids = new ArrayList<Long>();
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(
             "SELECT wi.id, wi.branch, wi.state, wi.location, wi.slot_id, " +
             "wi.created_at, r.path AS repo_path, r.github_repo " +
             "FROM work_items wi JOIN repos r ON wi.repo_id = r.id " +
             "WHERE wi.state != 'ended' ORDER BY wi.created_at")) {
        try (var rs = stmt.executeQuery()) {
            while (rs.next()) {
                var id = rs.getLong("id");
                ids.add(id);
                items.add(new WorkItem(id, rs.getString("branch"),
                        rs.getString("state"), rs.getString("location"),
                        rs.getObject("slot_id") != null ? rs.getLong("slot_id") : null,
                        rs.getString("created_at"), rs.getString("repo_path"),
                        rs.getString("github_repo"), List.of()));
            }
        }
        if (!ids.isEmpty()) {
            var issueMap = fetchIssuesForWorkItems(conn, ids);
            items = items.stream().map(wi ->
                    new WorkItem(wi.id(), wi.branch(), wi.state(), wi.location(),
                            wi.slotId(), wi.createdAt(), wi.repoPath(), wi.githubRepo(),
                            issueMap.getOrDefault(wi.id(), List.of()))
            ).toList();
        }
    } catch (SQLException e) {
        return List.of();
    }
    return items;
}

private LinkedHashMap<Long, List<WorkItemIssue>> fetchIssuesForWorkItems(
        Connection conn, List<Long> workItemIds) throws SQLException {
    var placeholders = String.join(",", workItemIds.stream().map(id -> "?").toList());
    var sql = "SELECT work_item_id, issue_number, issue_repo, is_primary " +
              "FROM work_item_issues WHERE work_item_id IN (" + placeholders + ")";
    var map = new LinkedHashMap<Long, List<WorkItemIssue>>();
    try (var stmt = conn.prepareStatement(sql)) {
        for (int i = 0; i < workItemIds.size(); i++) {
            stmt.setLong(i + 1, workItemIds.get(i));
        }
        try (var rs = stmt.executeQuery()) {
            while (rs.next()) {
                var wiId = rs.getLong("work_item_id");
                map.computeIfAbsent(wiId, k -> new ArrayList<>()).add(
                        new WorkItemIssue(rs.getInt("issue_number"),
                                rs.getString("issue_repo"),
                                rs.getInt("is_primary") == 1));
            }
        }
    }
    return map;
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest -q`
Expected: 6 tests PASS

- [ ] **Step 9: Add tests for slotStatus, workItemTimeline, backlogEntries**

Append to `WorklogServiceTest.java`:

```java
@Test
void slotStatusReturnsAll() {
    var slots = service.slotStatus(null);
    assertEquals(1, slots.size());
    assertEquals(7, slots.get(0).slotNumber());
    assertEquals("active", slots.get(0).state());
}

@Test
void slotStatusFiltersByFamilyRoot() {
    var slots = service.slotStatus("/nonexistent");
    assertEquals(0, slots.size());
}

@Test
void workItemTimelineReturnsEventsForBranch() {
    var events = service.workItemTimeline("issue-42-worklog", "/repo/a");
    assertEquals(2, events.size());
    assertEquals("work-start", events.get(0).eventType());
    assertEquals("work-continue", events.get(1).eventType());
}

@Test
void workItemTimelineReturnsEmptyForUnknown() {
    var events = service.workItemTimeline("no-such-branch", "/repo/a");
    assertEquals(0, events.size());
}
```

- [ ] **Step 10: Implement slotStatus and workItemTimeline**

Add to `WorklogService.java` using `ide_insert_member`:

```java
public List<SlotInfo> slotStatus(String familyRoot) {
    if (!dbAvailable) return List.of();
    checkFreshness();
    var results = new ArrayList<SlotInfo>();
    var sql = familyRoot != null && !familyRoot.isBlank()
            ? "SELECT * FROM slots WHERE family_root = ? ORDER BY slot_number"
            : "SELECT * FROM slots ORDER BY family_root, slot_number";
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(sql)) {
        if (familyRoot != null && !familyRoot.isBlank()) {
            stmt.setString(1, Path.of(familyRoot).toAbsolutePath().normalize().toString());
        }
        try (var rs = stmt.executeQuery()) {
            while (rs.next()) {
                results.add(new SlotInfo(rs.getLong("id"), rs.getInt("slot_number"),
                        rs.getString("family_root"), rs.getString("state"),
                        rs.getString("created_at"), rs.getString("archived_at")));
            }
        }
    } catch (SQLException | IOException e) {
        return List.of();
    }
    return results;
}

public List<WorklogEvent> workItemTimeline(String branch, String repoPath) {
    if (!dbAvailable) return List.of();
    checkFreshness();
    var resolvedPath = Path.of(repoPath);
    resolvedPath = resolvedPath.toAbsolutePath().normalize();
    var results = new ArrayList<WorklogEvent>();
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(
             "SELECT e.* FROM events e " +
             "JOIN work_items wi ON e.work_item_id = wi.id " +
             "JOIN repos r ON wi.repo_id = r.id " +
             "WHERE wi.branch = ? AND r.path = ? ORDER BY e.id")) {
        stmt.setString(1, branch);
        stmt.setString(2, resolvedPath.toString());
        try (var rs = stmt.executeQuery()) {
            while (rs.next()) {
                results.add(mapEvent(rs));
            }
        }
    } catch (SQLException e) {
        return List.of();
    }
    return results;
}
```

- [ ] **Step 11: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest -q`
Expected: 10 tests PASS

- [ ] **Step 12: Add backlogEntries and test**

Append to `WorklogServiceTest.java`:

```java
@Test
void backlogEntriesReturnsOpenIssues() throws SQLException {
    seedConn.createStatement().execute(
        "INSERT INTO github_issue_cache (issue_number, issue_repo, title, state, labels, cached_at) VALUES " +
        "(10, 'Test/repo', 'Open issue', 'OPEN', '[]', '2026-08-09T10:00:00Z')," +
        "(11, 'Test/repo', 'Closed issue', 'CLOSED', '[]', '2026-08-09T10:00:00Z')");
    var entries = service.backlogEntries("Test/repo");
    assertEquals(1, entries.size());
    assertEquals("Open issue", entries.get(0).title());
}
```

Add to `WorklogService.java` using `ide_insert_member` (same SQL as existing BacklogResource, moved here):

```java
public List<BacklogEntry> backlogEntries(String repo) {
    if (!dbAvailable) return List.of();
    checkFreshness();
    var baseSql = """
        SELECT c.issue_number, c.issue_repo, c.title, c.labels, c.cached_at,
               e.strategic_role, e.readiness, e.decay, e.blast_radius, e.cohesion, e.updated_at,
               t.note AS trajectory_note, t.created_at AS trajectory_at
        FROM github_issue_cache c
        LEFT JOIN issue_enrichment e
          ON c.issue_number = e.issue_number AND c.issue_repo = e.issue_repo
        LEFT JOIN trajectory_notes t
          ON t.id = (
            SELECT id FROM trajectory_notes t2
            WHERE t2.issue_number = c.issue_number AND t2.issue_repo = c.issue_repo
            ORDER BY t2.id DESC LIMIT 1
          )
        WHERE c.state = 'OPEN'
        """;
    var sql = repo != null && !repo.isBlank()
            ? baseSql + " AND c.issue_repo = ? ORDER BY c.issue_repo, c.issue_number"
            : baseSql + " ORDER BY c.issue_repo, c.issue_number";
    var results = new ArrayList<BacklogEntry>();
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(sql)) {
        if (repo != null && !repo.isBlank()) {
            stmt.setString(1, repo);
        }
        try (var rs = stmt.executeQuery()) {
            while (rs.next()) {
                results.add(mapBacklogEntry(rs));
            }
        }
    } catch (SQLException e) {
        return List.of();
    }
    return results;
}

private BacklogEntry mapBacklogEntry(ResultSet rs) throws SQLException {
    return new BacklogEntry(
            rs.getInt("issue_number"), rs.getString("issue_repo"),
            rs.getString("title"), parseLabels(rs.getString("labels")),
            rs.getString("cached_at"), rs.getString("strategic_role"),
            rs.getString("readiness"), rs.getString("decay"),
            rs.getString("blast_radius"), rs.getString("cohesion"),
            rs.getString("updated_at"), rs.getString("trajectory_note"),
            rs.getString("trajectory_at"));
}

private List<String> parseLabels(String json) {
    if (json == null || json.isBlank()) return List.of();
    try {
        return MAPPER.readValue(json, new com.fasterxml.jackson.core.type.TypeReference<>() {});
    } catch (Exception e) {
        return List.of();
    }
}
```

- [ ] **Step 13: Add planPosition and test**

Append to `WorklogServiceTest.java`:

```java
@Test
void planPositionParsesActiveLine() throws IOException {
    var planDir = tmpDir.resolve("design");
    Files.createDirectories(planDir);
    Files.writeString(planDir.resolve(".plan"), """
        # Work Plan
        ## Queue
        - [x] #40 — Done task
        - [ ] #42 — Active task ← active
        - [ ] #43 — Future task
        ## Session State
        Current: #42
        """);
    var state = service.planPosition(tmpDir);
    assertNotNull(state);
    assertEquals("#42", state.activeIssue());
    assertEquals(1, state.position());
    assertEquals(3, state.total());
}

@Test
void planPositionReturnsNullWhenNoPlan() {
    var state = service.planPosition(tmpDir.resolve("nonexistent"));
    assertNull(state);
}
```

Add import `java.io.IOException` and `java.nio.file.Files` to test file.

Add to `WorklogService.java` using `ide_insert_member`:

```java
public PlanState planPosition(Path workspaceRoot) {
    if (workspaceRoot == null) return null;
    var planFile = workspaceRoot.resolve("design/.plan");
    if (!Files.exists(planFile)) return null;
    try {
        var lines = Files.readAllLines(planFile);
        String active = null;
        int done = 0;
        int total = 0;
        for (var line : lines) {
            var trimmed = line.trim();
            if (trimmed.startsWith("- [x]") || trimmed.startsWith("- [ ]")) {
                total++;
                if (trimmed.startsWith("- [x]")) done++;
                if (trimmed.contains("← active")) {
                    var match = trimmed.replaceAll(".*?(#\\d+).*", "$1");
                    active = match;
                }
            }
        }
        if (total == 0) return null;
        return new PlanState(active, done, total);
    } catch (IOException e) {
        return null;
    }
}
```

- [ ] **Step 14: Add freshness detection test**

Append to `WorklogServiceTest.java`:

```java
@Test
void freshnessDetectsFileChange() throws Exception {
    long gen1 = generation.current();
    service.recentEvents(null, null, 1);
    long gen2 = generation.current();

    Thread.sleep(50);
    seedConn.createStatement().execute(
        "INSERT INTO events (timestamp, event_type) VALUES ('2026-08-11T12:00:00Z', 'test')");
    seedConn.commit();

    Thread.sleep(1100);

    service.recentEvents(null, null, 1);
    long gen3 = generation.current();
    assertTrue(gen3 > gen2, "Generation should increment after DB modification");
}
```

- [ ] **Step 15: Add schema version check test**

Append to `WorklogServiceTest.java`:

```java
@Test
void schemaVersionTooOldDisablesService() throws SQLException {
    var oldDbPath = tmpDir.resolve("old-worklog.db");
    var ds = new SQLiteDataSource();
    ds.setUrl("jdbc:sqlite:" + oldDbPath);
    try (var conn = ds.getConnection()) {
        conn.createStatement().execute("PRAGMA user_version = 1");
        conn.createStatement().execute("CREATE TABLE dummy (id INTEGER)");
    }
    var oldService = WorklogService.withSchemaCheck(ds, generation, oldDbPath);
    assertFalse(oldService.isDbAvailable());
    assertEquals(List.of(), oldService.recentEvents(null, null, 10));
}
```

Add factory method to `WorklogService.java`:

```java
static WorklogService withSchemaCheck(DataSource ds, GenerationCounter gen, Path path) {
    try (var conn = ds.getConnection()) {
        var version = conn.createStatement()
                .executeQuery("PRAGMA user_version").getInt(1);
        if (version < 2) {
            var svc = new WorklogService(ds, gen, path);
            svc.dbAvailable = false;
            return svc;
        }
        if (version > 2) {
            System.out.println("WARN: worklog.db schema version " + version +
                    " is newer than expected (2) — continuing with best effort");
        }
    } catch (SQLException e) {
        var svc = new WorklogService(ds, gen, path);
        svc.dbAvailable = false;
        return svc;
    }
    return new WorklogService(ds, gen, path);
}
```

- [ ] **Step 16: Add dbAvailable guard test**

Append to `WorklogServiceTest.java`:

```java
@Test
void allMethodsReturnEmptyWhenUnavailable() throws SQLException {
    var unavailablePath = tmpDir.resolve("no-db.db");
    var ds = new SQLiteDataSource();
    ds.setUrl("jdbc:sqlite:" + unavailablePath);
    var svc = new WorklogService(ds, new GenerationCounter(), unavailablePath);
    svc.dbAvailable = false;

    assertEquals(List.of(), svc.recentEvents(null, null, 10));
    assertEquals(List.of(), svc.activeWork());
    assertEquals(List.of(), svc.slotStatus(null));
    assertEquals(List.of(), svc.backlogEntries(null));
    assertEquals(List.of(), svc.workItemTimeline("x", "/x"));
    assertNull(svc.planPosition(tmpDir));
}
```

- [ ] **Step 17: Run all WorklogService tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogServiceTest -q`
Expected: All tests PASS

- [ ] **Step 18: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#42): WorklogService — JDBC reader with freshness, schema check, .plan parsing

Refs #42"
```

---

### Task 3: BacklogResource delegation refactor

Replace BacklogResource's direct SQL with delegation to WorklogService.

**Files:**
- Modify: `src/main/java/io/hortora/trellis/backlog/BacklogResource.java`
- Modify: `src/test/java/io/hortora/trellis/backlog/BacklogResourceTest.java` — update imports
- Test: existing `BacklogResourceTest` (regression)

**Interfaces:**
- Consumes: `WorklogService.backlogEntries(String repo): List<BacklogEntry>`, `WorklogService.isDbAvailable(): boolean`
- Produces: `GET /api/backlog` (unchanged REST contract)

- [ ] **Step 1: Rewrite BacklogResource to delegate**

Use `ide_replace_member` on `BacklogResource` class body. Replace the entire class with:

```java
package io.hortora.trellis.backlog;

import io.hortora.trellis.worklog.BacklogEntry;
import io.hortora.trellis.worklog.WorklogService;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;

import java.util.List;

@Path("/api/backlog")
@Produces(MediaType.APPLICATION_JSON)
public class BacklogResource {

    @Inject
    WorklogService worklogService;

    @GET
    public List<BacklogEntry> list(@QueryParam("repo") String repo) {
        return worklogService.backlogEntries(repo);
    }
}
```

- [ ] **Step 2: Update BacklogResourceTest imports**

The test uses `WorklogDataSourceProducer` which moved to `worklog` package. Update import using `ide_replace_member` or direct edit:

Replace `import io.hortora.trellis.backlog.WorklogDataSourceProducer;` with `import io.hortora.trellis.worklog.WorklogDataSourceProducer;`

- [ ] **Step 3: Run BacklogResourceTest**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=BacklogResourceTest -q`
Expected: All 6 existing tests PASS — same REST contract, same response shape

- [ ] **Step 4: Compile full project**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/
git -C /Users/mdproctor/claude/hortora/trellis commit -m "refactor(#42): BacklogResource delegates to WorklogService

Refs #42"
```

---

### Task 4: WorklogModelProvider

MCP model tree integration via the ModelProvider SPI.

**Files:**
- Create: `src/main/java/io/hortora/trellis/mcp/WorklogModelProvider.java`
- Create: `src/main/java/io/hortora/trellis/worklog/WorklogSummary.java`
- Test: `src/test/java/io/hortora/trellis/mcp/WorklogModelProviderTest.java`

**Interfaces:**
- Consumes: `WorklogService` (all query methods), `FileWatcherService.allModels()` for workspace root
- Produces: `ModelProvider` implementation — domain `"worklog"`, auto-discovered via `Instance<ModelProvider>` in `TrellisTools`

- [ ] **Step 1: Create WorklogSummary record**

Create `src/main/java/io/hortora/trellis/worklog/WorklogSummary.java`:

```java
package io.hortora.trellis.worklog;

public record WorklogSummary(int activeWorkItems, int recentEventCount,
                              WorklogEvent latestEvent, PlanState planPosition,
                              int slotsActive) {}
```

- [ ] **Step 2: Add summary method to WorklogService**

Add to `WorklogService.java` using `ide_insert_member`:

```java
private volatile WorklogSummary cachedSummary;
private volatile long cachedSummaryTime;
private static final long SUMMARY_TTL_MS = 5000;

public WorklogSummary summary(Path workspaceRoot) {
    if (!dbAvailable) return new WorklogSummary(0, 0, null, null, 0);
    checkFreshness();
    long now = System.currentTimeMillis();
    if (cachedSummary != null && (now - cachedSummaryTime) < SUMMARY_TTL_MS) {
        return cachedSummary;
    }
    var active = activeWork();
    var recent = recentEvents(null, null, 1);
    var slots = slotStatus(null);
    var plan = planPosition(workspaceRoot);
    int activeSlots = (int) slots.stream().filter(s -> "active".equals(s.state())).count();
    int eventCount = countEvents();
    var summary = new WorklogSummary(active.size(), eventCount,
            recent.isEmpty() ? null : recent.get(0), plan, activeSlots);
    cachedSummary = summary;
    cachedSummaryTime = now;
    return summary;
}

private int countEvents() {
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement("SELECT COUNT(*) FROM events")) {
        try (var rs = stmt.executeQuery()) {
            return rs.next() ? rs.getInt(1) : 0;
        }
    } catch (SQLException e) {
        return 0;
    }
}
```

- [ ] **Step 3: Write WorklogModelProvider test**

Create `src/test/java/io/hortora/trellis/mcp/WorklogModelProviderTest.java`:

```java
package io.hortora.trellis.mcp;

import io.hortora.trellis.worklog.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class WorklogModelProviderTest {

    private WorklogModelProvider provider;
    private StubWorklogService stubService;

    @BeforeEach
    void setUp() {
        stubService = new StubWorklogService();
        provider = new WorklogModelProvider(stubService, (io.hortora.trellis.scanner.FileWatcherService) null);
    }

    @Test
    void domainIsWorklog() {
        assertEquals("worklog", provider.domain());
    }

    @Test
    void summaryReturnsSummaryMap() {
        var result = provider.summary();
        assertNotNull(result);
        assertTrue(result instanceof java.util.Map);
    }

    @Test
    void resolveEventsReturnsEventList() {
        var result = provider.resolve("events");
        assertNotNull(result);
        assertTrue(result instanceof List);
    }

    @Test
    void resolveWorkItemsReturnsActiveWork() {
        var result = provider.resolve("work-items");
        assertNotNull(result);
    }

    @Test
    void resolveSlotsReturnsSlotList() {
        var result = provider.resolve("slots");
        assertNotNull(result);
    }

    @Test
    void resolveBacklogReturnsList() {
        var result = provider.resolve("backlog");
        assertNotNull(result);
    }

    @Test
    void resolveUnknownReturnsNull() {
        assertNull(provider.resolve("nonexistent"));
    }

    @Test
    void actionsForReturnsEmpty() {
        assertTrue(provider.actionsFor("worklog").isEmpty());
    }

    static class StubWorklogService extends WorklogService {
        StubWorklogService() {
            super(null, new GenerationCounter(), (Path) null);
        }

        @Override public boolean isDbAvailable() { return true; }
        @Override public List<WorklogEvent> recentEvents(String s, String t, int l) { return List.of(); }
        @Override public List<WorkItem> activeWork() { return List.of(); }
        @Override public List<SlotInfo> slotStatus(String f) { return List.of(); }
        @Override public List<BacklogEntry> backlogEntries(String r) { return List.of(); }
        @Override public List<WorklogEvent> workItemTimeline(String b, String r) { return List.of(); }
        @Override public PlanState planPosition(Path p) { return null; }
        @Override public WorklogSummary summary(Path p) {
            return new WorklogSummary(0, 0, null, null, 0);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogModelProviderTest -q`
Expected: FAIL — `WorklogModelProvider` does not exist

- [ ] **Step 5: Implement WorklogModelProvider**

Create `src/main/java/io/hortora/trellis/mcp/WorklogModelProvider.java`:

```java
package io.hortora.trellis.mcp;

import io.hortora.trellis.scanner.FileWatcherService;
import io.hortora.trellis.worklog.WorklogService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.nio.file.Path;
import java.util.LinkedHashMap;
import java.util.List;

@ApplicationScoped
public class WorklogModelProvider implements ModelProvider {

    private final WorklogService worklogService;
    private final FileWatcherService fileWatcher;

    @Inject
    public WorklogModelProvider(WorklogService worklogService, FileWatcherService fileWatcher) {
        this.worklogService = worklogService;
        this.fileWatcher = fileWatcher;
    }

    @Override
    public String domain() {
        return "worklog";
    }

    @Override
    public Object summary() {
        var root = resolveWorkspaceRoot();
        var s = worklogService.summary(root);
        var map = new LinkedHashMap<String, Object>();
        map.put("activeWorkItems", s.activeWorkItems());
        map.put("recentEventCount", s.recentEventCount());
        if (s.latestEvent() != null) {
            var event = new LinkedHashMap<String, Object>();
            event.put("type", s.latestEvent().eventType());
            event.put("timestamp", s.latestEvent().timestamp());
            map.put("latestEvent", event);
        }
        if (s.planPosition() != null) {
            var plan = new LinkedHashMap<String, Object>();
            plan.put("active", s.planPosition().activeIssue());
            plan.put("completed", s.planPosition().completed() + "/" + s.planPosition().total());
            map.put("planPosition", plan);
        }
        map.put("slotsActive", s.slotsActive());
        return map;
    }

    @Override
    public Object resolve(String subpath) {
        if (subpath == null || subpath.isEmpty()) return summary();
        return switch (subpath.split("/")[0]) {
            case "events" -> worklogService.recentEvents(null, null, 50);
            case "work-items" -> worklogService.activeWork();
            case "slots" -> worklogService.slotStatus(null);
            case "backlog" -> worklogService.backlogEntries(null);
            default -> null;
        };
    }

    @Override
    public List<ActionDescriptor> actionsFor(String nodeType) {
        return List.of();
    }

    private Path resolveWorkspaceRoot() {
        if (fileWatcher == null) return null;
        var models = fileWatcher.allModels();
        return models.isEmpty() ? null : models.getFirst().root();
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogModelProviderTest -q`
Expected: 8 tests PASS

- [ ] **Step 7: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -q`
Expected: All tests PASS (including ModelProviderTest which verifies provider discovery)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#42): WorklogModelProvider — model tree integration via ModelProvider SPI

Refs #42"
```

---

### Task 5: WorklogResource — REST endpoints

REST endpoints with query parameter support, mirroring model subpaths.

**Files:**
- Create: `src/main/java/io/hortora/trellis/worklog/WorklogResource.java`
- Test: `src/test/java/io/hortora/trellis/worklog/WorklogResourceTest.java`

**Interfaces:**
- Consumes: `WorklogService` (all query methods)
- Produces: `GET /api/worklog/events`, `GET /api/worklog/work-items`, `GET /api/worklog/work-items/{branch}/timeline`, `GET /api/worklog/slots`

- [ ] **Step 1: Write REST integration test**

Create `src/test/java/io/hortora/trellis/worklog/WorklogResourceTest.java`:

```java
package io.hortora.trellis.worklog;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.sqlite.SQLiteDataSource;

import java.sql.Connection;
import java.sql.SQLException;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assumptions.assumeTrue;

@QuarkusTest
class WorklogResourceTest {

    @Inject
    WorklogDataSourceProducer producer;

    @Inject
    WorklogService worklogService;

    private Connection writableConn;

    @BeforeEach
    void seedData() throws SQLException {
        assumeTrue(producer.isDbAvailable(), "worklog.db not available — skipping");
        var ds = new SQLiteDataSource();
        ds.setUrl("jdbc:sqlite:" + producer.getDbPath());
        writableConn = ds.getConnection();

        writableConn.createStatement().execute(
            "INSERT OR IGNORE INTO repos (id, path, github_repo) VALUES " +
            "(9990, '/test/worklog-rest', 'Test/worklog-rest')");
        writableConn.createStatement().execute(
            "INSERT OR IGNORE INTO work_items (id, branch, repo_id, state, location, created_at) VALUES " +
            "(9990, 'test-rest-branch', 9990, 'active', 'primary', '2026-08-11T10:00:00Z')");
        writableConn.createStatement().execute(
            "INSERT OR IGNORE INTO work_item_issues VALUES (9990, 99, 'Test/rest', 1)");
        writableConn.createStatement().execute(
            "INSERT INTO events (timestamp, event_type, work_item_id, repo_path) VALUES " +
            "('2026-08-11T10:00:00Z', 'work-start', 9990, '/test/worklog-rest')");
        writableConn.createStatement().execute(
            "INSERT OR IGNORE INTO slots (id, slot_number, family_root, state, created_at) VALUES " +
            "(9990, 999, '/test/family', 'active', '2026-08-11T09:00:00Z')");
    }

    @AfterEach
    void cleanData() throws SQLException {
        if (writableConn == null) return;
        try {
            writableConn.createStatement().execute(
                "DELETE FROM events WHERE repo_path = '/test/worklog-rest'");
            writableConn.createStatement().execute(
                "DELETE FROM work_item_issues WHERE work_item_id = 9990");
            writableConn.createStatement().execute(
                "DELETE FROM work_items WHERE id = 9990");
            writableConn.createStatement().execute(
                "DELETE FROM repos WHERE id = 9990");
            writableConn.createStatement().execute(
                "DELETE FROM slots WHERE id = 9990");
        } finally {
            writableConn.close();
        }
    }

    @Test
    void eventsEndpointReturnsEvents() {
        given()
            .when().get("/api/worklog/events")
            .then().statusCode(200)
            .body("size()", greaterThan(0))
            .body("[0].eventType", notNullValue());
    }

    @Test
    void eventsFiltersByType() {
        given().queryParam("type", "work-start")
            .when().get("/api/worklog/events")
            .then().statusCode(200)
            .body("eventType", everyItem(is("work-start")));
    }

    @Test
    void eventsRespectsLimit() {
        given().queryParam("limit", 1)
            .when().get("/api/worklog/events")
            .then().statusCode(200)
            .body("size()", is(1));
    }

    @Test
    void workItemsReturnsActiveOnly() {
        given()
            .when().get("/api/worklog/work-items")
            .then().statusCode(200)
            .body("state", everyItem(not(is("ended"))));
    }

    @Test
    void slotsEndpointReturnsSlots() {
        given()
            .when().get("/api/worklog/slots")
            .then().statusCode(200)
            .body("size()", greaterThan(0));
    }

    @Test
    void timelineRequiresRepoPath() {
        given()
            .when().get("/api/worklog/work-items/test-rest-branch/timeline")
            .then().statusCode(400);
    }

    @Test
    void timelineReturnsEventsForBranch() {
        given().queryParam("repoPath", "/test/worklog-rest")
            .when().get("/api/worklog/work-items/test-rest-branch/timeline")
            .then().statusCode(200)
            .body("size()", greaterThan(0))
            .body("[0].eventType", is("work-start"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogResourceTest -q`
Expected: FAIL — 404 on `/api/worklog/*` endpoints

- [ ] **Step 3: Implement WorklogResource**

Create `src/main/java/io/hortora/trellis/worklog/WorklogResource.java`:

```java
package io.hortora.trellis.worklog;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.List;

@Path("/api/worklog")
@Produces(MediaType.APPLICATION_JSON)
public class WorklogResource {

    @Inject
    WorklogService worklogService;

    @GET
    @Path("/events")
    public List<WorklogEvent> events(
            @QueryParam("since") String since,
            @QueryParam("type") String type,
            @QueryParam("limit") @DefaultValue("50") int limit) {
        return worklogService.recentEvents(since, type, limit);
    }

    @GET
    @Path("/work-items")
    public List<WorkItem> workItems() {
        return worklogService.activeWork();
    }

    @GET
    @Path("/work-items/{branch}/timeline")
    public Response timeline(
            @PathParam("branch") String branch,
            @QueryParam("repoPath") String repoPath) {
        if (repoPath == null || repoPath.isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity("{\"error\":\"repoPath query parameter is required\"}")
                    .build();
        }
        return Response.ok(worklogService.workItemTimeline(branch, repoPath)).build();
    }

    @GET
    @Path("/slots")
    public List<SlotInfo> slots(@QueryParam("familyRoot") String familyRoot) {
        return worklogService.slotStatus(familyRoot);
    }
}
```

- [ ] **Step 4: Add getDbPath to WorklogDataSourceProducer**

The `WorklogResourceTest` needs `producer.getDbPath()` to create a writable connection for seeding. Add to `WorklogDataSourceProducer.java` using `ide_insert_member`:

```java
public Path getDbPath() {
    var resolved = dbPath.replace("${user.home}", System.getProperty("user.home"));
    return java.nio.file.Path.of(resolved);
}
```

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorklogResourceTest -q`
Expected: 7 tests PASS

- [ ] **Step 6: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -q`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#42): WorklogResource — REST endpoints with query parameter filtering

Refs #42"
```
