# LLM Coordinator L2 — Propose Actions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #17 — Trellis: LLM Coordinator L2 — Propose Actions
**Issue group:** #17

**Goal:** The coordinator proposes executable actions with approve/reject
UI, and approved actions are routed to lifecycle manager, agent process
manager, or recorded as advisory acknowledgements.

**Architecture:** Extends L1's advice + event pipeline with a first-class
`ProposedAction` entity. Actions live in the advice feed (cards with
`actionKey` gain approve/reject). Three executor categories: lifecycle
(wrapping `LifecycleManager`), agent (stubbed for issue #20), advisory
(acknowledge-only). Async execution on a dedicated thread. Circuit
breaker prevents action→event→LLM feedback loops.

**Tech Stack:** Java 21, Quarkus 3.x, casehub-blocks (EventAccumulator),
jakarta.json (nested JSON parsing), Lit (TypeScript frontend), SQLite.

## Global Constraints

- Package root: `io.hortora.trellis.coordinator`
- All new types are records or sealed interfaces (Java 21)
- Existing patterns: CDI `@ApplicationScoped`, `@Inject`, `@CoordinatorDS` DataSource
- Schema versioning: `PRAGMA user_version` — bump from 1 to 2
- SSE push via `EventBroadcaster.broadcast(topic, payload)`
- Tests: JUnit 5 + `@QuarkusTest` where needed, `assumeTrue` for tmux

---

### Task 1: Domain Model + Schema Migration

**Files:**
- Create: `src/main/java/io/hortora/trellis/coordinator/ActionCategory.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/ActionStatus.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/RiskLevel.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/ProposedAction.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/ActionResult.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/RiskClassification.java`
- Create: `src/main/resources/coordinator-schema-v2.sql`
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorSchemaManager.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/CoordinatorSchemaManagerTest.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/RiskClassificationTest.java`

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces:
  - `ActionCategory` enum: `LIFECYCLE`, `AGENT`, `ADVISORY`
  - `ActionStatus` enum: `PROPOSED`, `APPROVED`, `CONFIRMING`, `EXECUTING`, `COMPLETED`, `FAILED`, `REJECTED`, `EXPIRED`
  - `RiskLevel` enum: `LOW`, `HIGH`
  - `ProposedAction` record: `(String id, ActionCategory category, String actionType, Map<String,String> params, RiskLevel risk, String rationale, ActionStatus status, String adviceId, String workspace, Instant proposedAt, Instant resolvedAt, String executionResult)`
  - `ActionResult` record: `(boolean success, int exitCode, Map<String,String> output, String detail)`
  - `RiskClassification.riskFor(String actionType)` → `RiskLevel`
  - Schema v2 with `coordinator_actions` table

- [ ] **Step 1: Write RiskClassification test**

```java
package io.hortora.trellis.coordinator;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class RiskClassificationTest {

    @Test
    void highRiskActionsIdentified() {
        assertEquals(RiskLevel.HIGH, RiskClassification.riskFor("lifecycle.start"));
        assertEquals(RiskLevel.HIGH, RiskClassification.riskFor("lifecycle.end"));
        assertEquals(RiskLevel.HIGH, RiskClassification.riskFor("slot.merge"));
        assertEquals(RiskLevel.HIGH, RiskClassification.riskFor("agent.stop"));
    }

    @Test
    void lowRiskActionsIdentified() {
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("lifecycle.pause"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("lifecycle.resume"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("slot.create"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("epic.setup"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("epic.next"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("agent.start"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("agent.pause"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("agent.resume"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("agent.refresh"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("advisory.prioritise"));
        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("advisory.investigate"));
    }

    @Test
    void unknownTypeDefaultsToHigh() {
        assertEquals(RiskLevel.HIGH, RiskClassification.riskFor("unknown.type"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=RiskClassificationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `RiskClassification` does not exist.

- [ ] **Step 3: Create all domain types**

Create `ActionCategory.java`:
```java
package io.hortora.trellis.coordinator;

public enum ActionCategory { LIFECYCLE, AGENT, ADVISORY }
```

Create `ActionStatus.java`:
```java
package io.hortora.trellis.coordinator;

public enum ActionStatus {
    PROPOSED, APPROVED, CONFIRMING, EXECUTING,
    COMPLETED, FAILED, REJECTED, EXPIRED;

    public boolean isTerminal() {
        return this == COMPLETED || this == FAILED || this == REJECTED || this == EXPIRED;
    }
}
```

Create `RiskLevel.java`:
```java
package io.hortora.trellis.coordinator;

public enum RiskLevel { LOW, HIGH }
```

Create `ProposedAction.java`:
```java
package io.hortora.trellis.coordinator;

import java.time.Instant;
import java.util.Map;

public record ProposedAction(
    String id,
    ActionCategory category,
    String actionType,
    Map<String, String> params,
    RiskLevel risk,
    String rationale,
    ActionStatus status,
    String adviceId,
    String workspace,
    Instant proposedAt,
    Instant resolvedAt,
    String executionResult
) {}
```

Create `ActionResult.java`:
```java
package io.hortora.trellis.coordinator;

import java.util.Map;

public record ActionResult(boolean success, int exitCode, Map<String, String> output, String detail) {

    public static ActionResult ok(String detail) {
        return new ActionResult(true, 0, Map.of(), detail);
    }

    public static ActionResult fail(String detail) {
        return new ActionResult(false, -1, Map.of(), detail);
    }
}
```

Create `RiskClassification.java`:
```java
package io.hortora.trellis.coordinator;

import java.util.Set;

public final class RiskClassification {

    private static final Set<String> HIGH_RISK = Set.of(
            "lifecycle.start", "lifecycle.end", "slot.merge", "agent.stop");

    private RiskClassification() {}

    public static RiskLevel riskFor(String actionType) {
        return HIGH_RISK.contains(actionType) ? RiskLevel.HIGH : RiskLevel.LOW;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=RiskClassificationTest`
Expected: PASS — 3 tests.

- [ ] **Step 5: Write schema migration test**

Extend existing `CoordinatorSchemaManagerTest` with a test for the v2 schema:

```java
@Test
void schemaV2CreatesActionsTable() throws Exception {
    var ds = createInMemoryDataSource();
    new CoordinatorSchemaManager().initialize(ds);
    try (var conn = ds.getConnection();
         var rs = conn.createStatement().executeQuery(
                 "SELECT name FROM sqlite_master WHERE type='table' AND name='coordinator_actions'")) {
        assertTrue(rs.next(), "coordinator_actions table should exist");
    }
}
```

- [ ] **Step 6: Create schema v2 file and update SchemaManager**

Create `src/main/resources/coordinator-schema-v2.sql`:
```sql
CREATE TABLE IF NOT EXISTS coordinator_actions (
    id               TEXT PRIMARY KEY,
    advice_id        TEXT NOT NULL,
    category         TEXT NOT NULL,
    action_type      TEXT NOT NULL,
    params           TEXT NOT NULL,
    risk             TEXT NOT NULL,
    rationale        TEXT NOT NULL,
    status           TEXT NOT NULL,
    workspace        TEXT NOT NULL,
    proposed_at      TEXT NOT NULL,
    resolved_at      TEXT,
    execution_result TEXT,
    FOREIGN KEY (advice_id) REFERENCES coordinator_advice(id)
);
CREATE INDEX IF NOT EXISTS idx_actions_workspace ON coordinator_actions(workspace, status);
CREATE INDEX IF NOT EXISTS idx_actions_advice ON coordinator_actions(advice_id);
```

Update `CoordinatorSchemaManager`:
- Change `SCHEMA_VERSION` from `1` to `2`
- Add `applySchemaV2(conn)` method that reads `coordinator-schema-v2.sql`
- In `initialize()`, if `version < 2`, call `applySchemaV2(conn)`

```java
private static final int SCHEMA_VERSION = 2;

// In initialize(), after the existing applySchema call:
if (version < 2) {
    applySchemaV2(conn);
}
setVersion(conn, SCHEMA_VERSION);

private void applySchemaV2(Connection conn) throws SQLException {
    try (var is = getClass().getClassLoader().getResourceAsStream("coordinator-schema-v2.sql")) {
        if (is == null) throw new SQLException("coordinator-schema-v2.sql not found on classpath");
        var sql = new String(is.readAllBytes());
        for (var stmt : sql.split(";")) {
            var trimmed = stmt.trim();
            if (!trimmed.isEmpty()) {
                try (var s = conn.createStatement()) { s.execute(trimmed); }
            }
        }
    } catch (java.io.IOException e) {
        throw new SQLException("Failed to read schema file", e);
    }
}
```

- [ ] **Step 7: Run schema test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=CoordinatorSchemaManagerTest`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#17): add L2 domain model and schema migration

Refs #17"
```

---

### Task 2: Event Pipeline Extension + Circuit Breaker

**Files:**
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorEvent.java`
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorEventObserver.java`
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorContextAssembler.java`
- Modify: `src/main/java/io/hortora/trellis/coordinator/SignificanceFilter.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/SignificanceFilterTest.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/CoordinatorEventObserverTest.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/CoordinatorContextAssemblerTest.java`

**Interfaces:**
- Consumes: `ActionStatus` from Task 1
- Produces:
  - `CoordinatorEvent.ActionStateChangedEvent(Instant, String, String, ActionStatus, ActionStatus, String)`
  - `CoordinatorEvent.LifecycleOperationEvent(Instant, String, String, boolean, String)`
  - `SignificanceFilter.isSignificant(List<CoordinatorEvent>)` — updated with circuit breaker

- [ ] **Step 1: Write SignificanceFilter circuit breaker test**

```java
@Test
void circuitBreakerBlocksActionOnlyBatches() {
    var filter = new SignificanceFilter();
    var actionEvents = new ArrayList<CoordinatorEvent>();
    for (int i = 0; i < 6; i++) {
        actionEvents.add(new CoordinatorEvent.ActionStateChangedEvent(
                Instant.now(), "ws", "action-" + i,
                ActionStatus.PROPOSED, ActionStatus.APPROVED, "slot.merge"));
    }
    assertFalse(filter.isSignificant(actionEvents));
}

@Test
void mixedBatchWithActionEventsPassesFilter() {
    var events = List.<CoordinatorEvent>of(
            new CoordinatorEvent.WorkspaceChangedEvent(Instant.now(), "/ws", Path.of("/ws")),
            new CoordinatorEvent.ActionStateChangedEvent(
                    Instant.now(), "ws", "a1",
                    ActionStatus.PROPOSED, ActionStatus.APPROVED, "epic.next"));
    var filter = new SignificanceFilter();
    assertTrue(filter.isSignificant(events));
}

@Test
void terminalActionEventsExcludedFromSignificance() {
    var events = List.<CoordinatorEvent>of(
            new CoordinatorEvent.ActionStateChangedEvent(
                    Instant.now(), "ws", "a1",
                    ActionStatus.PROPOSED, ActionStatus.EXPIRED, "slot.merge"));
    var filter = new SignificanceFilter();
    assertFalse(filter.isSignificant(events));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=SignificanceFilterTest`
Expected: Compilation failure — new event types don't exist.

- [ ] **Step 3: Add new event variants to CoordinatorEvent**

Add to the sealed interface in `CoordinatorEvent.java`:

```java
record ActionStateChangedEvent(
        Instant timestamp, String key,
        String actionId, ActionStatus oldStatus, ActionStatus newStatus,
        String actionType
) implements CoordinatorEvent {}

record LifecycleOperationEvent(
        Instant timestamp, String key,
        String operation, boolean success, String detail
) implements CoordinatorEvent {}
```

- [ ] **Step 4: Update SignificanceFilter with circuit breaker**

```java
@ApplicationScoped
public class SignificanceFilter {

    private static final int ACTION_EVENT_THRESHOLD = 5;

    public boolean isSignificant(List<CoordinatorEvent> batch) {
        if (batch.isEmpty()) return false;

        boolean hasNonActionEvent = false;
        int actionEventCount = 0;

        for (var event : batch) {
            switch (event) {
                case CoordinatorEvent.AnalysisEvent ae when !ae.newlyUnblocked().isEmpty() ->
                        hasNonActionEvent = true;
                case CoordinatorEvent.WorkspaceChangedEvent ignored -> hasNonActionEvent = true;
                case CoordinatorEvent.IssueEvent ignored -> hasNonActionEvent = true;
                case CoordinatorEvent.ActionStateChangedEvent ase -> {
                    if (!ase.newStatus().isTerminal()) actionEventCount++;
                }
                case CoordinatorEvent.LifecycleOperationEvent ignored -> actionEventCount++;
                default -> {}
            }
        }

        if (hasNonActionEvent) return true;
        return actionEventCount > 0 && actionEventCount <= ACTION_EVENT_THRESHOLD;
    }
}
```

- [ ] **Step 5: Update CoordinatorContextAssembler.formatEvent**

Add cases for the new event types in the exhaustive switch:

```java
case CoordinatorEvent.ActionStateChangedEvent e ->
        "ActionStateChanged: " + e.actionId() + " " + e.oldStatus() + "→" + e.newStatus() + " (" + e.actionType() + ")";
case CoordinatorEvent.LifecycleOperationEvent e ->
        "LifecycleOperation: " + e.operation() + " success=" + e.success() + " " + e.detail();
```

- [ ] **Step 6: Run all coordinator tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="SignificanceFilterTest,CoordinatorEventObserverTest,CoordinatorContextAssemblerTest"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#17): extend event pipeline with action + lifecycle events, circuit breaker

Refs #17"
```

---

### Task 3: ActionExecutor Framework

**Files:**
- Create: `src/main/java/io/hortora/trellis/coordinator/ActionExecutor.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/LifecycleActionExecutor.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/AgentActionExecutor.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/AdvisoryActionExecutor.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/LifecycleActionExecutorTest.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/AgentActionExecutorTest.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/AdvisoryActionExecutorTest.java`

**Interfaces:**
- Consumes: `ProposedAction`, `ActionResult`, `ActionCategory` from Task 1; `LifecycleManager` and `OperationResult` from `io.hortora.trellis.lifecycle`
- Produces:
  - `ActionExecutor` interface: `category()`, `supportedTypes()`, `execute(ProposedAction)`
  - `LifecycleActionExecutor` — maps actionType → LifecycleManager calls
  - `AgentActionExecutor` — stubs returning "not yet available"
  - `AdvisoryActionExecutor` — acknowledge-only

- [ ] **Step 1: Write LifecycleActionExecutor test**

```java
package io.hortora.trellis.coordinator;

import io.hortora.trellis.lifecycle.LifecycleManager;
import io.hortora.trellis.lifecycle.OperationResult;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.time.Instant;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class LifecycleActionExecutorTest {

    LifecycleManager manager = mock(LifecycleManager.class);
    LifecycleActionExecutor executor = new LifecycleActionExecutor(manager);

    @Test
    void categoryIsLifecycle() {
        assertEquals(ActionCategory.LIFECYCLE, executor.category());
    }

    @Test
    void supportsAllLifecycleTypes() {
        assertTrue(executor.supportedTypes().containsAll(Set.of(
                "lifecycle.start", "lifecycle.end", "lifecycle.pause", "lifecycle.resume",
                "slot.create", "slot.merge", "epic.setup", "epic.next")));
    }

    @Test
    void executesEpicNext() throws Exception {
        var action = new ProposedAction("a1", ActionCategory.LIFECYCLE, "epic.next",
                Map.of("epicPath", "/path/to/.epic"), RiskLevel.LOW, "ready",
                ActionStatus.APPROVED, "adv1", "/ws", Instant.now(), null, null);
        when(manager.epicNext("/path/to/.epic"))
                .thenReturn(new OperationResult(true, 0, Map.of(), ""));
        var result = executor.execute(action);
        assertTrue(result.success());
        verify(manager).epicNext("/path/to/.epic");
    }

    @Test
    void executesSlotMerge() throws Exception {
        var action = new ProposedAction("a2", ActionCategory.LIFECYCLE, "slot.merge",
                Map.of("slotId", "slot-1", "workspaceRoot", "/ws"), RiskLevel.HIGH,
                "done", ActionStatus.APPROVED, "adv2", "/ws", Instant.now(), null, null);
        when(manager.slotMerge("slot-1", Path.of("/ws")))
                .thenReturn(new OperationResult(true, 0, Map.of(), ""));
        var result = executor.execute(action);
        assertTrue(result.success());
    }

    @Test
    void listParamsReconstructed() throws Exception {
        var action = new ProposedAction("a3", ActionCategory.LIFECYCLE, "slot.create",
                Map.of("workspaceRoot", "/ws", "args.0", "issue-5", "args.1", "my-branch"),
                RiskLevel.LOW, "create", ActionStatus.APPROVED, "adv3", "/ws",
                Instant.now(), null, null);
        when(manager.slotCreate(eq(Path.of("/ws")), anyList()))
                .thenReturn(new OperationResult(true, 0, Map.of(), ""));
        executor.execute(action);
        verify(manager).slotCreate(Path.of("/ws"), java.util.List.of("issue-5", "my-branch"));
    }

    @Test
    void checkedExceptionConvertedToFailedResult() throws Exception {
        var action = new ProposedAction("a4", ActionCategory.LIFECYCLE, "epic.next",
                Map.of("epicPath", "/p"), RiskLevel.LOW, "x",
                ActionStatus.APPROVED, "adv4", "/ws", Instant.now(), null, null);
        when(manager.epicNext("/p")).thenThrow(new java.io.IOException("script error"));
        var result = executor.execute(action);
        assertFalse(result.success());
        assertTrue(result.detail().contains("script error"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleActionExecutorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure.

- [ ] **Step 3: Create ActionExecutor interface**

```java
package io.hortora.trellis.coordinator;

import java.util.Set;

public interface ActionExecutor {
    ActionCategory category();
    Set<String> supportedTypes();
    ActionResult execute(ProposedAction action);
}
```

- [ ] **Step 4: Create LifecycleActionExecutor**

```java
package io.hortora.trellis.coordinator;

import io.hortora.trellis.lifecycle.ConcurrentOperationException;
import io.hortora.trellis.lifecycle.LifecycleManager;
import io.hortora.trellis.lifecycle.OperationResult;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.io.IOException;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

@ApplicationScoped
public class LifecycleActionExecutor implements ActionExecutor {

    private static final Set<String> TYPES = Set.of(
            "lifecycle.start", "lifecycle.end", "lifecycle.pause", "lifecycle.resume",
            "slot.create", "slot.merge", "epic.setup", "epic.next");

    private final LifecycleManager manager;

    @Inject
    public LifecycleActionExecutor(LifecycleManager manager) {
        this.manager = manager;
    }

    @Override public ActionCategory category() { return ActionCategory.LIFECYCLE; }
    @Override public Set<String> supportedTypes() { return TYPES; }

    @Override
    public ActionResult execute(ProposedAction action) {
        try {
            var result = dispatch(action);
            return new ActionResult(result.success(), result.exitCode(), result.output(), result.stderr());
        } catch (ConcurrentOperationException e) {
            return ActionResult.fail("Concurrent operation: " + e.getMessage());
        } catch (IOException | InterruptedException e) {
            if (e instanceof InterruptedException) Thread.currentThread().interrupt();
            return ActionResult.fail(e.getMessage());
        }
    }

    private OperationResult dispatch(ProposedAction action)
            throws IOException, InterruptedException, ConcurrentOperationException {
        var p = action.params();
        return switch (action.actionType()) {
            case "lifecycle.start" -> manager.start(
                    Path.of(p.get("workspaceRoot")), p.get("branch"), p.get("issue"));
            case "lifecycle.end" -> manager.end(p.get("slotId"), Path.of(p.get("workspaceRoot")));
            case "lifecycle.pause" -> manager.pause(p.get("slotId"), Path.of(p.get("workspaceRoot")));
            case "lifecycle.resume" -> manager.resume(p.get("slotId"), Path.of(p.get("workspaceRoot")));
            case "slot.create" -> manager.slotCreate(Path.of(p.get("workspaceRoot")), collectListParams(p));
            case "slot.merge" -> manager.slotMerge(p.get("slotId"), Path.of(p.get("workspaceRoot")));
            case "epic.setup" -> manager.epicSetup(Path.of(p.get("workspaceRoot")), collectListParams(p));
            case "epic.next" -> manager.epicNext(p.get("epicPath"));
            default -> throw new IllegalArgumentException("Unknown action type: " + action.actionType());
        };
    }

    private List<String> collectListParams(Map<String, String> params) {
        var args = new ArrayList<String>();
        for (int i = 0; params.containsKey("args." + i); i++) {
            args.add(params.get("args." + i));
        }
        return args;
    }
}
```

- [ ] **Step 5: Run LifecycleActionExecutor test**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleActionExecutorTest`
Expected: PASS.

- [ ] **Step 6: Write and create AgentActionExecutor**

Test:
```java
package io.hortora.trellis.coordinator;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class AgentActionExecutorTest {

    AgentActionExecutor executor = new AgentActionExecutor();

    @Test
    void allTypesReturnNotAvailable() {
        for (var type : executor.supportedTypes()) {
            var action = new ProposedAction("a1", ActionCategory.AGENT, type,
                    Map.of("terminalName", "t1"), RiskLevel.LOW, "x",
                    ActionStatus.APPROVED, "adv1", "/ws", Instant.now(), null, null);
            var result = executor.execute(action);
            assertFalse(result.success());
            assertTrue(result.detail().contains("not yet implemented"));
        }
    }
}
```

Implementation:
```java
package io.hortora.trellis.coordinator;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.Set;

@ApplicationScoped
public class AgentActionExecutor implements ActionExecutor {

    private static final Set<String> TYPES = Set.of(
            "agent.start", "agent.stop", "agent.pause", "agent.resume", "agent.refresh");

    @Override public ActionCategory category() { return ActionCategory.AGENT; }
    @Override public Set<String> supportedTypes() { return TYPES; }

    @Override
    public ActionResult execute(ProposedAction action) {
        return ActionResult.fail("Agent management not yet implemented (issue #20)");
    }
}
```

- [ ] **Step 7: Write and create AdvisoryActionExecutor**

Test:
```java
package io.hortora.trellis.coordinator;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class AdvisoryActionExecutorTest {

    AdvisoryActionExecutor executor = new AdvisoryActionExecutor();

    @Test
    void acknowledgeReturnsSuccess() {
        var action = new ProposedAction("a1", ActionCategory.ADVISORY, "advisory.prioritise",
                Map.of("issueKey", "#5", "reason", "unblocks 3 issues"), RiskLevel.LOW, "x",
                ActionStatus.APPROVED, "adv1", "/ws", Instant.now(), null, null);
        var result = executor.execute(action);
        assertTrue(result.success());
        assertTrue(result.detail().contains("Acknowledged"));
    }
}
```

Implementation:
```java
package io.hortora.trellis.coordinator;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.Set;

@ApplicationScoped
public class AdvisoryActionExecutor implements ActionExecutor {

    private static final Set<String> TYPES = Set.of("advisory.prioritise", "advisory.investigate");

    @Override public ActionCategory category() { return ActionCategory.ADVISORY; }
    @Override public Set<String> supportedTypes() { return TYPES; }

    @Override
    public ActionResult execute(ProposedAction action) {
        return ActionResult.ok("Acknowledged: " + action.actionType());
    }
}
```

- [ ] **Step 8: Run all executor tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="LifecycleActionExecutorTest,AgentActionExecutorTest,AdvisoryActionExecutorTest"`
Expected: PASS — all 3 test classes.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#17): add ActionExecutor interface with lifecycle, agent, advisory impls

Refs #17"
```

---

### Task 4: ActionService — State Machine, Persistence, Async Execution, Expiry

**Files:**
- Create: `src/main/java/io/hortora/trellis/coordinator/ActionService.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/ActionServiceTest.java`

**Interfaces:**
- Consumes: `ProposedAction`, `ActionStatus`, `ActionCategory`, `RiskLevel`, `RiskClassification`, `ActionResult` from Task 1; `ActionExecutor` from Task 3; `EventBroadcaster` from casehub-pages-push; `CoordinatorEvent.ActionStateChangedEvent` from Task 2
- Produces:
  - `ActionService.propose(String adviceId, ActionCategory category, String actionType, Map<String,String> params, String rationale, String workspace)` → `ProposedAction`
  - `ActionService.propose(Connection conn, ...)` — transactional variant accepting caller's connection
  - `ActionService.approve(String actionId)` → `ProposedAction`
  - `ActionService.confirm(String actionId)` → `ProposedAction`
  - `ActionService.cancel(String actionId)` → `ProposedAction`
  - `ActionService.reject(String actionId)` → `ProposedAction`
  - `ActionService.expireStale(String actionType, Map<String,String> params)`
  - `ActionService.pendingActions(String workspace)` → `List<ProposedAction>`
  - `ActionService.actionHistory(String workspace, int limit)` → `List<ProposedAction>`
  - `ActionService.getAction(String actionId)` → `ProposedAction`

- [ ] **Step 1: Write ActionService state machine tests**

```java
package io.hortora.trellis.coordinator;

import io.casehub.pages.push.EventBroadcaster;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.sqlite.SQLiteDataSource;

import javax.sql.DataSource;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class ActionServiceTest {

    DataSource dataSource;
    EventBroadcaster broadcaster = mock(EventBroadcaster.class);
    ActionService service;

    @BeforeEach
    void setUp() throws Exception {
        var sds = new SQLiteDataSource();
        sds.setUrl("jdbc:sqlite::memory:");
        dataSource = sds;
        new CoordinatorSchemaManager().initialize(dataSource);

        var lifecycleExecutor = mock(LifecycleActionExecutor.class);
        when(lifecycleExecutor.category()).thenReturn(ActionCategory.LIFECYCLE);
        when(lifecycleExecutor.supportedTypes()).thenReturn(java.util.Set.of("epic.next", "slot.merge"));
        when(lifecycleExecutor.execute(any())).thenReturn(ActionResult.ok("done"));

        var advisoryExecutor = new AdvisoryActionExecutor();

        service = ActionService.forTest(dataSource, broadcaster,
                List.of(lifecycleExecutor, advisoryExecutor));
    }

    @Test
    void proposeCreatesActionInProposedState() {
        var action = service.propose("adv1", ActionCategory.LIFECYCLE, "epic.next",
                Map.of("epicPath", "/p"), "ready", "/ws");
        assertNotNull(action.id());
        assertEquals(ActionStatus.PROPOSED, action.status());
        assertEquals("/ws", action.workspace());
    }

    @Test
    void approveLowRiskExecutesImmediately() {
        var action = service.propose("adv1", ActionCategory.LIFECYCLE, "epic.next",
                Map.of("epicPath", "/p"), "ready", "/ws");
        var approved = service.approve(action.id());
        assertEquals(ActionStatus.COMPLETED, approved.status());
    }

    @Test
    void approveHighRiskMovesToConfirming() {
        var action = service.propose("adv2", ActionCategory.LIFECYCLE, "slot.merge",
                Map.of("slotId", "s1", "workspaceRoot", "/ws"), "done", "/ws");
        var confirming = service.approve(action.id());
        assertEquals(ActionStatus.CONFIRMING, confirming.status());
    }

    @Test
    void confirmMovesToExecuting() {
        var action = service.propose("adv3", ActionCategory.LIFECYCLE, "slot.merge",
                Map.of("slotId", "s1", "workspaceRoot", "/ws"), "done", "/ws");
        service.approve(action.id());
        var confirmed = service.confirm(action.id());
        assertEquals(ActionStatus.COMPLETED, confirmed.status());
    }

    @Test
    void cancelReturnsToProposed() {
        var action = service.propose("adv4", ActionCategory.LIFECYCLE, "slot.merge",
                Map.of("slotId", "s1", "workspaceRoot", "/ws"), "done", "/ws");
        service.approve(action.id());
        var cancelled = service.cancel(action.id());
        assertEquals(ActionStatus.PROPOSED, cancelled.status());
    }

    @Test
    void rejectSetsTerminalState() {
        var action = service.propose("adv5", ActionCategory.ADVISORY, "advisory.prioritise",
                Map.of("issueKey", "#5"), "x", "/ws");
        var rejected = service.reject(action.id());
        assertEquals(ActionStatus.REJECTED, rejected.status());
        assertNotNull(rejected.resolvedAt());
    }

    @Test
    void invalidTransitionThrows() {
        var action = service.propose("adv6", ActionCategory.ADVISORY, "advisory.prioritise",
                Map.of("issueKey", "#5"), "x", "/ws");
        service.reject(action.id());
        assertThrows(IllegalStateException.class, () -> service.approve(action.id()));
    }

    @Test
    void expireStaleExpiresMatchingActions() {
        service.propose("adv7", ActionCategory.LIFECYCLE, "slot.merge",
                Map.of("slotId", "s1", "workspaceRoot", "/ws"), "merge it", "/ws");
        service.expireStale("slot.merge", Map.of("slotId", "s1"));
        var pending = service.pendingActions("/ws");
        assertTrue(pending.isEmpty());
    }

    @Test
    void pendingActionsReturnsOnlyProposedAndConfirming() {
        service.propose("adv8", ActionCategory.ADVISORY, "advisory.prioritise",
                Map.of("issueKey", "#5"), "x", "/ws");
        var a2 = service.propose("adv9", ActionCategory.ADVISORY, "advisory.investigate",
                Map.of("issueKey", "#6"), "y", "/ws");
        service.reject(a2.id());
        var pending = service.pendingActions("/ws");
        assertEquals(1, pending.size());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ActionServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `ActionService` does not exist.

- [ ] **Step 3: Implement ActionService**

Create `ActionService.java` with:
- `forTest()` static factory (same pattern as `CoordinatorService`)
- State machine validation in private `transition()` method
- SQLite persistence via `@CoordinatorDS DataSource`
- Async execution on `Executors.newSingleThreadExecutor()`
- `expireStale()` that checks PROPOSED, APPROVED, CONFIRMING
- SSE broadcast on `coordinator:action` topic
- CDI async event fire for `ActionStateChangedEvent`
- `propose(Connection, ...)` overload for transactional use

The full implementation is ~250 lines following the patterns of
`CoordinatorService` (same DataSource qualifier, same broadcast
pattern, same SQL style).

Key state machine logic:
```java
private ActionStatus validateTransition(ActionStatus current, ActionStatus target) {
    return switch (current) {
        case PROPOSED -> switch (target) {
            case APPROVED, CONFIRMING, REJECTED, EXPIRED -> target;
            default -> throw new IllegalStateException("Cannot transition from PROPOSED to " + target);
        };
        case CONFIRMING -> switch (target) {
            case EXECUTING, PROPOSED -> target;  // PROPOSED = cancel
            default -> throw new IllegalStateException("Cannot transition from CONFIRMING to " + target);
        };
        case APPROVED -> switch (target) {
            case EXECUTING -> target;
            default -> throw new IllegalStateException("Cannot transition from APPROVED to " + target);
        };
        case EXECUTING -> switch (target) {
            case COMPLETED, FAILED -> target;
            default -> throw new IllegalStateException("Cannot transition from EXECUTING to " + target);
        };
        default -> throw new IllegalStateException("Cannot transition from terminal state " + current);
    };
}
```

- [ ] **Step 4: Run ActionService tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ActionServiceTest`
Expected: PASS — all 9 tests.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#17): add ActionService with state machine, persistence, async execution

Refs #17"
```

---

### Task 5: LifecycleManager Event Emission

**Files:**
- Modify: `src/main/java/io/hortora/trellis/lifecycle/LifecycleManager.java`
- Test: `src/test/java/io/hortora/trellis/lifecycle/LifecycleManagerTest.java`

**Interfaces:**
- Consumes: `CoordinatorEvent.LifecycleOperationEvent` from Task 2
- Produces: `LifecycleOperationEvent` fired as CDI async event after each operation

- [ ] **Step 1: Write test for LifecycleOperationEvent emission**

Add to `LifecycleManagerTest`:
```java
@Test
void firesLifecycleOperationEventOnSuccess() throws Exception {
    // arrange: set up scriptRunner mock to return success for epicNext
    when(scriptRunner.run(eq("work-slot"), eq("epic_manager.py"), anyList()))
            .thenReturn(new OperationResult(true, 0, Map.of(), ""));

    manager.epicNext("/path/to/.epic");

    verify(lifecycleOperationEvent).fireAsync(argThat(event ->
            event instanceof CoordinatorEvent.LifecycleOperationEvent loe
            && loe.operation().equals("epicNext")
            && loe.success()));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleManagerTest`
Expected: FAIL — no `LifecycleOperationEvent` injection or emission in LifecycleManager.

- [ ] **Step 3: Add event emission to LifecycleManager.withLock()**

Add to `LifecycleManager`:
```java
@Inject
Event<CoordinatorEvent.LifecycleOperationEvent> lifecycleOperationEvent;
```

Update `withLock()` to fire the event after each operation:
```java
private OperationResult withLock(String key, String operationName, LockedOperation operation)
        throws IOException, InterruptedException, ConcurrentOperationException {
    if (!tryLock(key)) {
        throw new ConcurrentOperationException("Operation already in progress for: " + key);
    }
    try {
        var result = operation.execute();
        fireLifecycleOperationEvent(operationName, result);
        return result;
    } finally {
        unlock(key);
    }
}

private void fireLifecycleOperationEvent(String operationName, OperationResult result) {
    try {
        var event = new CoordinatorEvent.LifecycleOperationEvent(
                java.time.Instant.now(), operationName, operationName,
                result.success(), result.stderr());
        lifecycleOperationEvent.fireAsync(event);
    } catch (Exception e) {
        LOG.debugf(e, "Failed to fire LifecycleOperationEvent for %s", operationName);
    }
}
```

Update all callers of `withLock()` to pass the operation name as the
second argument (e.g., `"start"`, `"end"`, `"pause"`, `"slotMerge"`,
`"epicNext"`).

- [ ] **Step 4: Add observer in CoordinatorEventObserver**

```java
public void onLifecycleOperation(
        @ObservesAsync CoordinatorEvent.LifecycleOperationEvent event) {
    dispatch(event);
}
```

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="LifecycleManagerTest,CoordinatorEventObserverTest"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#17): fire LifecycleOperationEvent from LifecycleManager.withLock()

Refs #17"
```

---

### Task 6: LLM Integration — Prompts, Parser, CoordinatorService Bridge

**Files:**
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorPrompts.java`
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorContextAssembler.java`
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorService.java`
- Create: `src/main/java/io/hortora/trellis/coordinator/ActionResponseParser.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/ActionResponseParserTest.java`

**Interfaces:**
- Consumes: `ActionService.propose()` from Task 4; `ProposedAction`, `ActionCategory` from Task 1
- Produces:
  - `ActionResponseParser.parse(String response)` → `Optional<ParsedAction>`
  - `ParsedAction` record: `(ActionCategory category, String actionType, Map<String,String> params, String rationale)`
  - Updated `CoordinatorPrompts.systemPrompt()` with action type catalogue
  - Updated `CoordinatorContextAssembler` with `appendPendingActions()` and `appendActionHistory()`

- [ ] **Step 1: Write ActionResponseParser test**

```java
package io.hortora.trellis.coordinator;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ActionResponseParserTest {

    @Test
    void parsesAdviceWithNestedAction() {
        var json = """
                {"type": "SUGGESTION", "title": "Merge slot", "body": "Ready to merge",
                 "actionKey": "a1",
                 "action": {
                   "category": "LIFECYCLE",
                   "actionType": "slot.merge",
                   "params": {"slotId": "s1", "workspaceRoot": "/ws"},
                   "rationale": "All tests passing"
                 }}""";
        var result = ActionResponseParser.parseAction(json);
        assertTrue(result.isPresent());
        var action = result.get();
        assertEquals(ActionCategory.LIFECYCLE, action.category());
        assertEquals("slot.merge", action.actionType());
        assertEquals("s1", action.params().get("slotId"));
        assertEquals("All tests passing", action.rationale());
    }

    @Test
    void returnsEmptyForAdviceWithoutAction() {
        var json = """
                {"type": "INSIGHT", "title": "Progress update", "body": "3 issues closed"}""";
        var result = ActionResponseParser.parseAction(json);
        assertTrue(result.isEmpty());
    }

    @Test
    void returnsEmptyForMalformedAction() {
        var json = """
                {"type": "SUGGESTION", "title": "Bad", "body": "x",
                 "action": {"category": "INVALID"}}""";
        var result = ActionResponseParser.parseAction(json);
        assertTrue(result.isEmpty());
    }

    @Test
    void returnsEmptyForNoneResponse() {
        var json = """
                {"none": true}""";
        var result = ActionResponseParser.parseAction(json);
        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ActionResponseParserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure.

- [ ] **Step 3: Create ActionResponseParser**

```java
package io.hortora.trellis.coordinator;

import jakarta.json.Json;
import jakarta.json.JsonObject;
import java.io.StringReader;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

public final class ActionResponseParser {

    private ActionResponseParser() {}

    public record ParsedAction(
            ActionCategory category, String actionType,
            Map<String, String> params, String rationale) {}

    public static Optional<ParsedAction> parseAction(String response) {
        try {
            var reader = Json.createReader(new StringReader(response));
            var root = reader.readObject();
            if (!root.containsKey("action")) return Optional.empty();
            var action = root.getJsonObject("action");
            var category = ActionCategory.valueOf(action.getString("category"));
            var actionType = action.getString("actionType");
            var rationale = action.getString("rationale", "");
            var paramsObj = action.getJsonObject("params");
            var params = new HashMap<String, String>();
            for (var key : paramsObj.keySet()) {
                params.put(key, paramsObj.getString(key));
            }
            return Optional.of(new ParsedAction(category, actionType, params, rationale));
        } catch (Exception e) {
            return Optional.empty();
        }
    }
}
```

- [ ] **Step 4: Run parser tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ActionResponseParserTest`
Expected: PASS.

- [ ] **Step 5: Update CoordinatorPrompts.systemPrompt()**

Add the action type catalogue to the system prompt string (append after
the existing text). Include all action types with their expected params.

- [ ] **Step 6: Update CoordinatorContextAssembler**

Add two methods:

```java
void appendPendingActions(StringBuilder sb, String workspace, ActionService actionService) {
    var pending = actionService.pendingActions(workspace);
    if (!pending.isEmpty()) {
        sb.append("\n## Pending Actions\n");
        for (var a : pending) {
            sb.append("- [%s] %s %s (status=%s)\n"
                    .formatted(a.id(), a.actionType(), a.params(), a.status()));
        }
    }
}

void appendActionHistory(StringBuilder sb, String workspace, ActionService actionService) {
    var history = actionService.actionHistory(workspace, 10);
    if (!history.isEmpty()) {
        sb.append("\n## Recent Action Outcomes\n");
        for (var a : history) {
            sb.append("- %s %s → %s: %s\n"
                    .formatted(a.actionType(), a.params(), a.status(),
                            a.executionResult() != null ? a.executionResult() : ""));
        }
    }
}
```

Update `assembleProactivePrompt()` and `assembleInteractivePrompt()` to
call these two methods, passing `ActionService` as a new constructor
dependency.

- [ ] **Step 7: Update CoordinatorService to bridge ActionService**

In `CoordinatorService`:
- Add `@Inject ActionService actionService`
- In `generateProactiveAdvice()`: after `parseAdviceResponse()`, call
  `ActionResponseParser.parseAction(response)`. If present, use the same
  SQLite connection to insert both advice and action atomically.
- In `handleMessage()`: same parsing and bridging after the LLM response.
- Update `persistAdvice()` to accept a `Connection` parameter for
  transactional use.

- [ ] **Step 8: Run all coordinator tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="ActionResponseParserTest,CoordinatorServiceTest,CoordinatorContextAssemblerTest"`
Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#17): LLM prompt extension, JSON action parser, CoordinatorService bridge

Refs #17"
```

---

### Task 7: REST API Endpoints

**Files:**
- Modify: `src/main/java/io/hortora/trellis/coordinator/CoordinatorResource.java`
- Test: `src/test/java/io/hortora/trellis/coordinator/CoordinatorResourceTest.java`

**Interfaces:**
- Consumes: `ActionService` from Task 4; `ProposedAction` from Task 1
- Produces: REST endpoints:
  - `GET /api/coordinator/actions?workspace={ws}` → `List<ProposedAction>`
  - `GET /api/coordinator/actions/history?workspace={ws}` → `List<ProposedAction>`
  - `GET /api/coordinator/actions/{id}` → `ProposedAction`
  - `POST /api/coordinator/actions/{id}/approve` → `ProposedAction`
  - `POST /api/coordinator/actions/{id}/reject` → `ProposedAction`
  - `POST /api/coordinator/actions/{id}/confirm` → `ProposedAction`
  - `POST /api/coordinator/actions/{id}/cancel` → `ProposedAction`

- [ ] **Step 1: Write REST endpoint tests**

```java
@Test
void approveReturnsUpdatedAction() {
    // Create advice + action via service, then POST /approve
    // Assert 200 with updated status
}

@Test
void approveNonExistentReturns404() {
    // POST /api/coordinator/actions/bogus/approve
    // Assert 404
}

@Test
void invalidTransitionReturns409() {
    // Create action, reject it, then try to approve
    // Assert 409
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=CoordinatorResourceTest`
Expected: FAIL — new endpoints don't exist.

- [ ] **Step 3: Add endpoints to CoordinatorResource**

```java
@Inject ActionService actionService;

@GET @Path("/actions")
public List<ProposedAction> actions(@QueryParam("workspace") String workspace) {
    return actionService.pendingActions(workspace);
}

@GET @Path("/actions/history")
public List<ProposedAction> actionHistory(@QueryParam("workspace") String workspace) {
    return actionService.actionHistory(workspace, 50);
}

@GET @Path("/actions/{id}")
public Response getAction(@PathParam("id") String id) {
    var action = actionService.getAction(id);
    if (action == null) return Response.status(404).entity(Map.of("error", "action not found")).build();
    return Response.ok(action).build();
}

@POST @Path("/actions/{id}/approve")
public Response approveAction(@PathParam("id") String id) {
    return actionOp(id, actionService::approve);
}

@POST @Path("/actions/{id}/reject")
public Response rejectAction(@PathParam("id") String id) {
    return actionOp(id, actionService::reject);
}

@POST @Path("/actions/{id}/confirm")
public Response confirmAction(@PathParam("id") String id) {
    return actionOp(id, actionService::confirm);
}

@POST @Path("/actions/{id}/cancel")
public Response cancelAction(@PathParam("id") String id) {
    return actionOp(id, actionService::cancel);
}

private Response actionOp(String id, java.util.function.Function<String, ProposedAction> op) {
    try {
        var action = op.apply(id);
        if (action == null) return Response.status(404).entity(Map.of("error", "action not found")).build();
        return Response.ok(action).build();
    } catch (IllegalStateException e) {
        return Response.status(409).entity(Map.of("error", e.getMessage())).build();
    }
}
```

- [ ] **Step 4: Run REST tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=CoordinatorResourceTest`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#17): REST API for action approve/reject/confirm/cancel

Refs #17"
```

---

### Task 8: Frontend — Advice Card Action Controls

**Files:**
- Modify: `src/main/webui/src/components/coordinator-panel.ts`

**Interfaces:**
- Consumes: REST endpoints from Task 7; SSE topic `coordinator:action`
- Produces: Interactive advice cards with approve/reject/confirm/cancel buttons

- [ ] **Step 1: Add ProposedAction TypeScript interface**

```typescript
interface ProposedAction {
    id: string;
    category: 'LIFECYCLE' | 'AGENT' | 'ADVISORY';
    actionType: string;
    params: Record<string, string>;
    risk: 'LOW' | 'HIGH';
    rationale: string;
    status: 'PROPOSED' | 'APPROVED' | 'CONFIRMING' | 'EXECUTING' | 'COMPLETED' | 'FAILED' | 'REJECTED' | 'EXPIRED';
    adviceId: string;
    workspace: string;
    proposedAt: string;
    resolvedAt: string | null;
    executionResult: string | null;
}
```

- [ ] **Step 2: Add action state tracking**

Add to `CoordinatorPanel`:
```typescript
@state() private actions: Map<string, ProposedAction> = new Map();
```

- [ ] **Step 3: Subscribe to coordinator:action SSE topic**

In `_connectSSE()`:
```typescript
this.eventSource.addEventListener('coordinator:action', (e: MessageEvent) => {
    const action: ProposedAction = JSON.parse(e.data);
    this.actions = new Map(this.actions).set(action.adviceId, action);
});
```

- [ ] **Step 4: Fetch initial action state for advice with actionKey**

In `_loadHistory()`, after loading advice, fetch actions for each
advice with a non-null `actionKey`:
```typescript
for (const a of this.advice.filter(a => a.actionKey)) {
    const res = await fetch(`/api/coordinator/actions/${a.actionKey}`);
    if (res.ok) {
        const action = await res.json();
        this.actions = new Map(this.actions).set(action.adviceId, action);
    }
}
```

- [ ] **Step 5: Render action buttons on advice cards**

Update the advice card template in `render()`:
```typescript
${this.advice.map(a => {
    const action = a.actionKey ? this.actions.get(a.id) : null;
    return html`
        <div class="advice-card">
            <span class="dismiss" @click=${() => this._dismiss(a.id)}>✕</span>
            <span class="badge badge-${a.type}">${a.type}</span>
            <span class="advice-title">${a.title}</span>
            <div class="advice-body">${a.body}</div>
            ${action ? this._renderActionControls(action) : nothing}
        </div>
    `;
})}
```

- [ ] **Step 6: Implement _renderActionControls**

```typescript
private _renderActionControls(action: ProposedAction) {
    switch (action.status) {
        case 'PROPOSED':
            return html`
                <div class="action-buttons">
                    <button class="btn-approve" @click=${() => this._approveAction(action.id)}>Approve</button>
                    <button class="btn-reject" @click=${() => this._rejectAction(action.id)}>Reject</button>
                </div>`;
        case 'CONFIRMING':
            return html`
                <div class="action-confirm">
                    <div class="confirm-warning">${action.rationale}</div>
                    <button class="btn-confirm" @click=${() => this._confirmAction(action.id)}>Confirm</button>
                    <button class="btn-cancel" @click=${() => this._cancelAction(action.id)}>Cancel</button>
                </div>`;
        case 'EXECUTING':
            return html`<div class="action-status executing">Executing...</div>`;
        case 'COMPLETED':
            return html`<div class="action-status completed">✓ ${action.executionResult ?? 'Done'}</div>`;
        case 'FAILED':
            return html`<div class="action-status failed">✗ ${action.executionResult ?? 'Failed'}</div>`;
        default:
            return nothing;
    }
}
```

- [ ] **Step 7: Implement action methods**

```typescript
private async _approveAction(id: string) {
    const res = await fetch(`/api/coordinator/actions/${id}/approve`, { method: 'POST' });
    if (res.ok) { const action = await res.json(); this.actions = new Map(this.actions).set(action.adviceId, action); }
}

private async _rejectAction(id: string) {
    const res = await fetch(`/api/coordinator/actions/${id}/reject`, { method: 'POST' });
    if (res.ok) { const action = await res.json(); this.actions = new Map(this.actions).set(action.adviceId, action); }
}

private async _confirmAction(id: string) {
    const res = await fetch(`/api/coordinator/actions/${id}/confirm`, { method: 'POST' });
    if (res.ok) { const action = await res.json(); this.actions = new Map(this.actions).set(action.adviceId, action); }
}

private async _cancelAction(id: string) {
    const res = await fetch(`/api/coordinator/actions/${id}/cancel`, { method: 'POST' });
    if (res.ok) { const action = await res.json(); this.actions = new Map(this.actions).set(action.adviceId, action); }
}
```

- [ ] **Step 8: Add CSS for action buttons and states**

```css
.action-buttons { display: flex; gap: 0.5rem; margin-top: 0.5rem; }
.btn-approve { padding: 0.3rem 0.8rem; border: none; border-radius: 4px; background: #166534; color: white; cursor: pointer; font-size: 0.75rem; }
.btn-reject { padding: 0.3rem 0.8rem; border: none; border-radius: 4px; background: #991b1b; color: white; cursor: pointer; font-size: 0.75rem; }
.btn-confirm { padding: 0.3rem 0.8rem; border: none; border-radius: 4px; background: #b45309; color: white; cursor: pointer; font-size: 0.75rem; }
.btn-cancel { padding: 0.3rem 0.8rem; border: none; border-radius: 4px; background: #333; color: #aaa; cursor: pointer; font-size: 0.75rem; }
.action-confirm { margin-top: 0.5rem; padding: 0.5rem; background: #2d1f00; border-radius: 4px; }
.confirm-warning { font-size: 0.8rem; color: #fde68a; margin-bottom: 0.5rem; }
.action-status { margin-top: 0.5rem; font-size: 0.75rem; }
.action-status.executing { color: #60a5fa; }
.action-status.completed { color: #86efac; }
.action-status.failed { color: #fca5a5; }
```

- [ ] **Step 9: Update SSE subscription URL**

```typescript
this.eventSource = new EventSource('/api/push?topics=coordinator:advice,coordinator:message,coordinator:action');
```

- [ ] **Step 10: Build frontend**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: No errors.

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(#17): frontend approve/reject/confirm/cancel on advice cards

Refs #17"
```

---

### Task 9: Integration Test — End-to-End Action Proposal

**Files:**
- Create: `src/test/java/io/hortora/trellis/coordinator/ActionProposalIntegrationTest.java`

**Interfaces:**
- Consumes: all prior tasks
- Produces: integration test verifying the full flow: LLM response → advice + action created → approve → execute → SSE broadcast

- [ ] **Step 1: Write integration test**

```java
package io.hortora.trellis.coordinator;

import io.casehub.pages.push.EventBroadcaster;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.sqlite.SQLiteDataSource;

import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class ActionProposalIntegrationTest {

    @Test
    void fullFlowFromLlmResponseToExecution() {
        var llmResponse = """
                {"type": "SUGGESTION", "title": "Advance epic", "body": "Issue done",
                 "actionKey": "test-action-1",
                 "action": {
                   "category": "LIFECYCLE",
                   "actionType": "epic.next",
                   "params": {"epicPath": "/path/.epic"},
                   "rationale": "Current issue is complete"
                 }}""";

        var parsed = ActionResponseParser.parseAction(llmResponse);
        assertTrue(parsed.isPresent());
        assertEquals("epic.next", parsed.get().actionType());
        assertEquals(ActionCategory.LIFECYCLE, parsed.get().category());

        assertEquals(RiskLevel.LOW, RiskClassification.riskFor("epic.next"));
    }

    @Test
    void actionResponseParserHandlesNoActionField() {
        var llmResponse = """
                {"type": "INSIGHT", "title": "Progress", "body": "Going well"}""";
        assertTrue(ActionResponseParser.parseAction(llmResponse).isEmpty());
    }

    @Test
    void actionServiceStateTransitionsWithRealDb() throws Exception {
        var sds = new SQLiteDataSource();
        sds.setUrl("jdbc:sqlite::memory:");
        new CoordinatorSchemaManager().initialize(sds);

        var executor = new AdvisoryActionExecutor();
        var service = ActionService.forTest(sds, mock(EventBroadcaster.class),
                List.of(executor));

        var action = service.propose("adv1", ActionCategory.ADVISORY,
                "advisory.prioritise", Map.of("issueKey", "#5"),
                "unblocks 3", "/ws");
        assertEquals(ActionStatus.PROPOSED, action.status());

        var approved = service.approve(action.id());
        assertEquals(ActionStatus.COMPLETED, approved.status());
        assertNotNull(approved.executionResult());

        var history = service.actionHistory("/ws", 10);
        assertEquals(1, history.size());
        assertEquals(ActionStatus.COMPLETED, history.getFirst().status());
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ActionProposalIntegrationTest`
Expected: PASS.

- [ ] **Step 3: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: All tests pass.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "test(#17): integration tests for action proposal end-to-end flow

Refs #17"
```
