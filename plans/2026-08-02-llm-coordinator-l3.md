# LLM Coordinator L3 — Autonomous Execution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #18 — Trellis: LLM Coordinator L3 + ISX (B8, post-MVP)
**Issue group:** #18

**Goal:** Add autonomous action execution with observation mode to the
coordinator, building on L2's action state machine and executor framework.

**Architecture:** Three autonomy levels (MANUAL/OBSERVATION/AUTONOMOUS)
controlled by a preferences file and a session toggle. A new
`AutonomyResolver` encapsulates policy resolution. `CountdownScheduler`
manages observation timers. `ActionService.autoExecute()` bypasses the
manual risk gate. All state transitions use optimistic locking (CAS).
Countdown deadlines are persisted for restart recovery.

**Tech Stack:** Java 21, Quarkus 3.x, SQLite, Lit, pages-push SSE

**Spec:** `specs/issue-18-llm-coordinator-l3-isx/2026-08-02-llm-coordinator-l3-design.md`

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- PreferencesService goes in `io.hortora.trellis.config` (not coordinator)
- All coordinator classes in `io.hortora.trellis.coordinator`
- No new `ActionStatus` enum values — L2 state machine unchanged
- All state transitions use CAS: `UPDATE ... WHERE status = ?`
- `ide-tooling` for all source file operations
- Existing tests must continue to pass after each task

---

## Task 1: Foundation — Enums, PreferencesService, AutonomyResolver

**Creates the autonomy model and preferences infrastructure. After this
task, the autonomy level and policy for any action type can be resolved
from preferences + session state.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/coordinator/AutonomyLevel.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/coordinator/AutonomyOverride.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/config/PreferencesService.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/coordinator/AutonomyResolver.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/config/PreferencesServiceTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/AutonomyResolverTest.java`

**Interfaces:**
- Produces: `AutonomyLevel` enum — `MANUAL`, `OBSERVATION`, `AUTONOMOUS`
- Produces: `AutonomyOverride` enum — `AUTONOMOUS`, `GATED`
- Produces: `PreferencesService.autonomyLevel(String workspace)` → `AutonomyLevel`
- Produces: `PreferencesService.autonomyOverride(String actionType)` → `AutonomyOverride` (nullable)
- Produces: `PreferencesService.observationCountdownSeconds()` → `int`
- Produces: `PreferencesService.rateLimitMaxActions()` → `int`
- Produces: `PreferencesService.rateLimitWindowSeconds()` → `int`
- Produces: `AutonomyResolver.resolveLevel(String workspace)` → `AutonomyLevel`
- Produces: `AutonomyResolver.resolvePolicy(String actionType)` → `AutonomyOverride`
- Produces: `AutonomyResolver.setSessionOverride(AutonomyLevel level)` / `clearSessionOverride()` / `sessionOverride()` → `AutonomyLevel`

**Steps:**

- [ ] **Step 1: Write AutonomyLevel and AutonomyOverride enums**

```java
// AutonomyLevel.java
package io.hortora.trellis.coordinator;
public enum AutonomyLevel { MANUAL, OBSERVATION, AUTONOMOUS }

// AutonomyOverride.java
package io.hortora.trellis.coordinator;
public enum AutonomyOverride { AUTONOMOUS, GATED }
```

Use `ide_create_file` for both.

- [ ] **Step 2: Write PreferencesService failing tests**

`PreferencesServiceTest.java` — test cases:
- `readsValidPreferences` — write a valid `preferences.json` to a temp dir,
  construct `PreferencesService` with that path, verify `autonomyLevel("/ws/a")`
  returns `OBSERVATION`, `observationCountdownSeconds()` returns 30,
  `autonomyOverride("slot.create")` returns `GATED`, `rateLimitMaxActions()`
  returns 5.
- `defaultsOnMissingFile` — construct with non-existent path, verify
  `autonomyLevel("/ws")` returns `MANUAL`, `observationCountdownSeconds()`
  returns 30, `autonomyOverride("anything")` returns null.
- `defaultsOnMalformedJson` — write `{invalid` to temp file, verify defaults.
- `perWorkspaceAutonomyLevel` — two workspaces with different levels.
- `unknownWorkspaceDefaultsToManual` — workspace not in map returns MANUAL.

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=PreferencesServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compile error (class doesn't exist)

- [ ] **Step 3: Implement PreferencesService**

```java
package io.hortora.trellis.config;

@ApplicationScoped
public class PreferencesService {
    private static final Logger LOG = Logger.getLogger(PreferencesService.class);
    private static final Path DEFAULT_PATH =
        Path.of(System.getProperty("user.home"), ".trellis", "preferences.json");

    private volatile JsonObject root = JsonObject.EMPTY_JSON_OBJECT;
    private final Path path;

    public PreferencesService() { this(DEFAULT_PATH); }

    PreferencesService(Path path) {
        this.path = path;
        reload();
    }

    public void reload() {
        if (!Files.exists(path)) { root = JsonObject.EMPTY_JSON_OBJECT; return; }
        try (var reader = Json.createReader(Files.newInputStream(path))) {
            root = reader.readObject();
        } catch (Exception e) {
            LOG.warnf(e, "Failed to parse %s — using defaults", path);
            root = JsonObject.EMPTY_JSON_OBJECT;
        }
    }

    public AutonomyLevel autonomyLevel(String workspace) {
        var coord = root.getJsonObject("coordinator");
        if (coord == null) return AutonomyLevel.MANUAL;
        var levels = coord.getJsonObject("autonomyLevel");
        if (levels == null) return AutonomyLevel.MANUAL;
        var val = levels.getString(workspace, null);
        if (val == null) return AutonomyLevel.MANUAL;
        try { return AutonomyLevel.valueOf(val); }
        catch (IllegalArgumentException e) { return AutonomyLevel.MANUAL; }
    }

    public AutonomyOverride autonomyOverride(String actionType) {
        var coord = root.getJsonObject("coordinator");
        if (coord == null) return null;
        var overrides = coord.getJsonObject("autonomyOverrides");
        if (overrides == null) return null;
        var val = overrides.getString(actionType, null);
        if (val == null) return null;
        try { return AutonomyOverride.valueOf(val); }
        catch (IllegalArgumentException e) { return null; }
    }

    public int observationCountdownSeconds() {
        return getInt("observationCountdownSeconds", 30);
    }

    public int rateLimitMaxActions() {
        return getInt("rateLimitMaxActions", 5);
    }

    public int rateLimitWindowSeconds() {
        return getInt("rateLimitWindowSeconds", 60);
    }

    private int getInt(String key, int defaultValue) {
        var coord = root.getJsonObject("coordinator");
        if (coord == null) return defaultValue;
        return coord.getInt(key, defaultValue);
    }
}
```

Use `ide_create_file`.

- [ ] **Step 4: Run PreferencesService tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=PreferencesServiceTest`
Expected: all pass

- [ ] **Step 5: Write AutonomyResolver failing tests**

`AutonomyResolverTest.java` — test cases:
- `riskBasedDefaults` — LOW-risk action → AUTONOMOUS policy, HIGH-risk → GATED
- `overridePromotesLowToGated` — override `slot.create` = GATED, verify policy is GATED despite LOW risk
- `overrideDemotesHighToAutonomous` — override `lifecycle.end` = AUTONOMOUS, verify policy is AUTONOMOUS despite HIGH risk
- `missingOverrideFallsToRisk` — no override for `lifecycle.pause` → AUTONOMOUS (LOW risk)
- `sessionOverrideTakesPrecedence` — set session override AUTONOMOUS, verify resolveLevel returns AUTONOMOUS regardless of preference
- `clearSessionOverrideReturnsToPreference` — set then clear, verify preference used
- `nullSessionOverrideUsesPreference` — no session override, verify preference used

- [ ] **Step 6: Implement AutonomyResolver**

```java
package io.hortora.trellis.coordinator;

@ApplicationScoped
public class AutonomyResolver {
    @Inject PreferencesService preferences;

    private volatile AutonomyLevel sessionOverride;

    AutonomyResolver() {}

    // test constructor
    static AutonomyResolver forTest(PreferencesService preferences) {
        var r = new AutonomyResolver();
        r.preferences = preferences;
        return r;
    }

    public AutonomyLevel resolveLevel(String workspace) {
        if (sessionOverride != null) return sessionOverride;
        return preferences.autonomyLevel(workspace);
    }

    public AutonomyOverride resolvePolicy(String actionType) {
        var override = preferences.autonomyOverride(actionType);
        if (override != null) return override;
        return RiskClassification.riskFor(actionType) == RiskLevel.LOW
            ? AutonomyOverride.AUTONOMOUS : AutonomyOverride.GATED;
    }

    public void setSessionOverride(AutonomyLevel level) {
        this.sessionOverride = level;
    }

    public void clearSessionOverride() {
        this.sessionOverride = null;
    }

    public AutonomyLevel sessionOverride() {
        return sessionOverride;
    }
}
```

Use `ide_create_file`.

- [ ] **Step 7: Run AutonomyResolver tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AutonomyResolverTest`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/main/java/io/hortora/trellis/coordinator/AutonomyLevel.java sidecar/src/main/java/io/hortora/trellis/coordinator/AutonomyOverride.java sidecar/src/main/java/io/hortora/trellis/config/PreferencesService.java sidecar/src/main/java/io/hortora/trellis/coordinator/AutonomyResolver.java sidecar/src/test/java/io/hortora/trellis/config/PreferencesServiceTest.java sidecar/src/test/java/io/hortora/trellis/coordinator/AutonomyResolverTest.java
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#18): add autonomy model — enums, PreferencesService, AutonomyResolver  Refs #18"
```

---

## Task 2: Schema v3, CAS Transitions, Countdown Persistence

**Adds the `countdown_ends_at` column and converts all ActionService
state transitions to optimistic locking. After this task, races between
concurrent transitions are eliminated.**

**Files:**
- Create: `sidecar/src/main/resources/coordinator-schema-v3.sql`
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/CoordinatorSchemaManager.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/ProposedAction.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/ActionService.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/OptimisticLockTest.java`

**Interfaces:**
- Consumes: `ProposedAction` record (Task 1 enums)
- Produces: `ProposedAction.countdownEndsAt()` → `Instant` (nullable)
- Produces: `ActionService.updateStatusCas(String actionId, ActionStatus expected, ActionStatus target)` → `int` (rows affected)

**Steps:**

- [ ] **Step 1: Create schema v3 SQL**

```sql
-- coordinator-schema-v3.sql
ALTER TABLE coordinator_actions ADD COLUMN countdown_ends_at TEXT;
```

Use `ide_create_file` for the SQL file.

- [ ] **Step 2: Add v3 migration to CoordinatorSchemaManager**

Use `ide_insert_member` to add `applySchemaV3(Connection conn)` after
`applySchemaV2`. Update `SCHEMA_VERSION` to 3 via `ide_edit_member`.
Add the v3 call in `initialize()` with a version check.

- [ ] **Step 3: Add countdownEndsAt to ProposedAction**

Use `ide_edit_member` on `ProposedAction` (member = `ProposedAction`):

```java
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
        String executionResult,
        Instant countdownEndsAt
) {}
```

- [ ] **Step 4: Fix all ProposedAction construction sites**

Use `ide_find_references` on `ProposedAction` constructor to find all
call sites. Add `null` as the last argument to each existing constructor
call (the new `countdownEndsAt` field). Also update `readAction()` in
`ActionService` to read the new column.

- [ ] **Step 5: Write OptimisticLockTest**

Test cases:
- `casSucceedsWhenStatusMatches` — create PROPOSED action, updateStatusCas
  with expected=PROPOSED target=APPROVED → returns 1, action is APPROVED.
- `casFailsWhenStatusMismatch` — create PROPOSED action, transition to
  REJECTED, then updateStatusCas with expected=PROPOSED target=APPROVED →
  returns 0, action remains REJECTED.
- `concurrentCasOnlyOneWins` — create PROPOSED action, two threads call
  updateStatusCas simultaneously → exactly one returns 1, the other returns 0.

- [ ] **Step 6: Add updateStatusCas to ActionService**

Use `ide_insert_member` after `updateStatus`:

```java
int updateStatusCas(String actionId, ActionStatus expected, ActionStatus target) {
    try (var conn = dataSource.getConnection();
         var ps = conn.prepareStatement(
             "UPDATE coordinator_actions SET status = ?, resolved_at = ? " +
             "WHERE id = ? AND status = ?")) {
        ps.setString(1, target.name());
        ps.setString(2, target.isTerminal() ? Instant.now().toString() : null);
        ps.setString(3, actionId);
        ps.setString(4, expected.name());
        return ps.executeUpdate();
    } catch (SQLException e) {
        LOG.warnf(e, "CAS update failed for %s", actionId);
        return 0;
    }
}
```

- [ ] **Step 7: Convert existing transitions to use CAS**

Use `ide_replace_member` on `approve()`, `confirm()`, `reject()`,
`executeAction()` to use `updateStatusCas` instead of `transition()` for
the initial state check. The `transition()` method remains for
broadcast + persist but is called only after CAS succeeds.

- [ ] **Step 8: Run all existing tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl .`
Expected: all pass (existing behaviour preserved, CAS is transparent)

- [ ] **Step 9: Run OptimisticLockTest**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=OptimisticLockTest`
Expected: all pass

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#18): add schema v3 (countdown_ends_at), CAS transitions  Refs #18"
```

---

## Task 3: CountdownScheduler

**Extracted countdown timer lifecycle. After this task, countdowns can
be scheduled, cancelled, and queried independently of ActionService.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/coordinator/CountdownScheduler.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/CountdownSchedulerTest.java`

**Interfaces:**
- Produces: `CountdownScheduler.schedule(String actionId, int seconds, Consumer<String> onFire)` → void
- Produces: `CountdownScheduler.cancel(String actionId)` → void
- Produces: `CountdownScheduler.cancelAll()` → void
- Produces: `CountdownScheduler.hasCountdown(String actionId)` → boolean
- Produces: `CountdownScheduler.deadline(String actionId)` → `Instant` (nullable)

**Steps:**

- [ ] **Step 1: Write CountdownSchedulerTest**

Test cases:
- `scheduleFiresCallback` — schedule 1-second countdown, verify callback
  fires with action ID. Use `CountDownLatch` for async assertion.
- `cancelPreventsCallback` — schedule 5-second countdown, cancel
  immediately, wait 2 seconds, verify callback never fired.
- `cancelAllClearsAllPending` — schedule 3 countdowns, cancelAll,
  verify none fire.
- `deadlineReturnsCorrectTime` — schedule 30-second countdown, verify
  `deadline()` is ~30 seconds from now (±1 second).
- `hasCountdownReturnsTrueWhilePending` — schedule, verify true, cancel,
  verify false.
- `exceptionInCallbackDoesNotKillScheduler` — schedule with a callback
  that throws, schedule another after, verify second one still fires.

- [ ] **Step 2: Implement CountdownScheduler**

```java
package io.hortora.trellis.coordinator;

@ApplicationScoped
public class CountdownScheduler {
    private static final Logger LOG = Logger.getLogger(CountdownScheduler.class);

    record PendingCountdown(String actionId, Instant deadline,
                            ScheduledFuture<?> future) {}

    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor(r -> {
            var t = new Thread(r, "countdown-scheduler");
            t.setDaemon(true);
            return t;
        });

    private final Map<String, PendingCountdown> countdowns =
        new ConcurrentHashMap<>();

    public void schedule(String actionId, int seconds,
                         Consumer<String> onFire) {
        var deadline = Instant.now().plusSeconds(seconds);
        var future = scheduler.schedule(() -> {
            try {
                countdowns.remove(actionId);
                onFire.accept(actionId);
            } catch (Exception e) {
                LOG.warnf(e, "Countdown callback failed for %s", actionId);
            }
        }, seconds, TimeUnit.SECONDS);
        var prev = countdowns.put(actionId, new PendingCountdown(actionId, deadline, future));
        if (prev != null) prev.future().cancel(false);
    }

    public void cancel(String actionId) {
        var removed = countdowns.remove(actionId);
        if (removed != null) removed.future().cancel(false);
    }

    public void cancelAll() {
        countdowns.values().forEach(c -> c.future().cancel(false));
        countdowns.clear();
    }

    public boolean hasCountdown(String actionId) {
        return countdowns.containsKey(actionId);
    }

    public Instant deadline(String actionId) {
        var c = countdowns.get(actionId);
        return c != null ? c.deadline() : null;
    }
}
```

Use `ide_create_file`.

- [ ] **Step 3: Run CountdownSchedulerTest**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=CountdownSchedulerTest`
Expected: all pass

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#18): add CountdownScheduler — timer lifecycle for observation mode  Refs #18"
```

---

## Task 4: ActionService Autonomy Integration

**Wires autonomy into ActionService: autoExecute(), autonomy-aware
propose(), sliding-window rate limiter, restart recovery, mode-switch
cancellation. This is the core task.**

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/ActionService.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/ActionServiceAutonomyTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/RateLimitTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/CountdownRecoveryTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/ModeSwitchTest.java`

**Interfaces:**
- Consumes: `AutonomyResolver` (Task 1)
- Consumes: `CountdownScheduler` (Task 3)
- Consumes: `PreferencesService` (Task 1)
- Consumes: `updateStatusCas()` (Task 2)
- Produces: `ActionService.autoExecute(String actionId)` → void
- Produces: `ActionService.cancelAllCountdowns()` → void
- Produces: `ActionService.recoverCountdowns(String workspace)` → void

**Steps:**

- [ ] **Step 1: Write ActionServiceAutonomyTest**

Test cases (mock AutonomyResolver, CountdownScheduler, ActionExecutor):
- `proposeInManualLeavesProposed` — resolver returns MANUAL, verify
  action stays PROPOSED, no countdown scheduled, no execution.
- `proposeAutonomousLowRiskAutoExecutes` — resolver returns AUTONOMOUS +
  AUTONOMOUS policy, mock executor succeeds, verify action reaches
  COMPLETED.
- `proposeAutonomousHighRiskSchedulesCountdown` — resolver returns
  AUTONOMOUS + GATED policy, verify countdown scheduled with
  `preferences.observationCountdownSeconds()`.
- `proposeObservationSchedulesCountdown` — resolver returns OBSERVATION,
  verify countdown scheduled regardless of policy.
- `countdownEndsAtPersistedOnSchedule` — verify `countdown_ends_at`
  column is set in DB when countdown is scheduled.
- `autoExecuteSkipsRiskGate` — HIGH-risk action, autoExecute called,
  verify it goes PROPOSED→APPROVED→EXECUTING→COMPLETED (not CONFIRMING).
- `autoExecuteCasPreventsDuplicateExecution` — transition action to
  REJECTED before autoExecute fires, verify autoExecute is no-op.

- [ ] **Step 2: Inject AutonomyResolver and CountdownScheduler into ActionService**

Use `ide_insert_member` for the new fields and update the constructor/forTest.

```java
@Inject AutonomyResolver autonomyResolver;
@Inject CountdownScheduler countdownScheduler;
@Inject PreferencesService preferences;
```

- [ ] **Step 3: Implement autoExecute()**

Use `ide_insert_member` after `executeAction`:

```java
void autoExecute(String actionId) {
    int updated = updateStatusCas(actionId, ActionStatus.PROPOSED, ActionStatus.APPROVED);
    if (updated == 0) return;
    var action = getAction(actionId);
    if (action == null) return;
    var approved = new ProposedAction(action.id(), action.category(),
        action.actionType(), action.params(), action.risk(), action.rationale(),
        ActionStatus.APPROVED, action.adviceId(), action.workspace(),
        action.proposedAt(), null, null, action.countdownEndsAt());
    broadcast(approved);
    executeAction(approved);
}
```

- [ ] **Step 4: Add autonomy-aware path to propose()**

Use `ide_replace_member` on both `propose()` overloads. After the
existing `persist` + `broadcast`, add:

```java
var level = autonomyResolver.resolveLevel(workspace);
if (level == AutonomyLevel.MANUAL) return action;

var policy = autonomyResolver.resolvePolicy(actionType);
if (level == AutonomyLevel.AUTONOMOUS && policy == AutonomyOverride.AUTONOMOUS) {
    if (isWithinRateLimit()) {
        recordAutonomousExecution();
        autoExecute(action.id());
    } else {
        scheduleCountdown(action);
    }
} else {
    scheduleCountdown(action);
}
return action;
```

- [ ] **Step 5: Implement scheduleCountdown() and rate limiter**

Use `ide_insert_member`:

```java
private final Deque<Instant> autonomousTimestamps = new ConcurrentLinkedDeque<>();

private void scheduleCountdown(ProposedAction action) {
    int seconds = preferences.observationCountdownSeconds();
    var deadline = Instant.now().plusSeconds(seconds);
    persistCountdownDeadline(action.id(), deadline);
    countdownScheduler.schedule(action.id(), seconds, this::autoExecute);
    var withDeadline = new ProposedAction(action.id(), action.category(),
        action.actionType(), action.params(), action.risk(), action.rationale(),
        action.status(), action.adviceId(), action.workspace(),
        action.proposedAt(), action.resolvedAt(), action.executionResult(), deadline);
    broadcast(withDeadline);
}

private boolean isWithinRateLimit() {
    pruneOldTimestamps();
    return autonomousTimestamps.size() < preferences.rateLimitMaxActions();
}

private void recordAutonomousExecution() {
    autonomousTimestamps.addLast(Instant.now());
}

private void pruneOldTimestamps() {
    var cutoff = Instant.now().minusSeconds(preferences.rateLimitWindowSeconds());
    while (!autonomousTimestamps.isEmpty() && autonomousTimestamps.peekFirst().isBefore(cutoff)) {
        autonomousTimestamps.pollFirst();
    }
}

public void resetRateLimit() {
    autonomousTimestamps.clear();
}
```

- [ ] **Step 6: Run ActionServiceAutonomyTest**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ActionServiceAutonomyTest`
Expected: all pass

- [ ] **Step 7: Write and run RateLimitTest**

Test cases:
- `withinLimitProceeds` — 4 autonomous executions within window, 5th succeeds.
- `exceedingLimitFallsToCountdown` — 5 executions, 6th gets countdown.
- `windowSlidesPrunesOld` — execute 5, advance time past window, next succeeds.
- `manualApprovalResetsTimestamps` — execute 5, call resetRateLimit, next succeeds.

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=RateLimitTest`

- [ ] **Step 8: Write and run CountdownRecoveryTest**

Test cases:
- `pastDeadlineAutoExecutesOnRecovery` — PROPOSED action with countdown_ends_at in the past → autoExecute called.
- `futureDeadlineRescheduled` — PROPOSED action with countdown_ends_at 20s in the future → countdown scheduled with remaining time.
- `nullDeadlineSkipped` — PROPOSED action without countdown_ends_at → no action.
- `terminalStateSkipped` — COMPLETED action with countdown_ends_at → no action.

- [ ] **Step 9: Implement recoverCountdowns()**

Use `ide_insert_member`:

```java
public void recoverCountdowns(String workspace) {
    var actions = queryActionsWithCountdown(workspace);
    var now = Instant.now();
    for (var action : actions) {
        if (action.status() != ActionStatus.PROPOSED) continue;
        if (action.countdownEndsAt() == null) continue;
        if (action.countdownEndsAt().isBefore(now)) {
            autoExecute(action.id());
        } else {
            int remaining = (int) Duration.between(now, action.countdownEndsAt()).getSeconds();
            countdownScheduler.schedule(action.id(), Math.max(1, remaining), this::autoExecute);
        }
    }
}
```

- [ ] **Step 10: Write and run ModeSwitchTest**

Test cases:
- `switchToManualCancelsCountdowns` — schedule 2 countdowns, call cancelAllCountdowns, verify countdowns cancelled and countdown_ends_at cleared.
- `switchFromManualDoesNotRetroact` — 2 PROPOSED actions exist, switch to OBSERVATION, verify no countdowns scheduled for existing actions.

- [ ] **Step 11: Implement cancelAllCountdowns()**

Use `ide_insert_member`:

```java
public void cancelAllCountdowns() {
    countdownScheduler.cancelAll();
    try (var conn = dataSource.getConnection();
         var ps = conn.prepareStatement(
             "UPDATE coordinator_actions SET countdown_ends_at = NULL " +
             "WHERE status = 'PROPOSED' AND countdown_ends_at IS NOT NULL")) {
        ps.executeUpdate();
    } catch (SQLException e) {
        LOG.warnf(e, "Failed to clear countdown deadlines");
    }
}
```

- [ ] **Step 12: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl .`
Expected: all pass

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#18): add autonomous execution path — autoExecute, rate limiter, countdown recovery  Refs #18"
```

---

## Task 5: CoordinatorService, Resource, and Prompts

**Wires the session toggle REST API, parameterizes the LLM prompt, and
connects mode switching to countdown cancellation.**

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/CoordinatorResource.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/CoordinatorService.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/CoordinatorPrompts.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/SessionAutonomyToggleTest.java`

**Interfaces:**
- Consumes: `AutonomyResolver` (Task 1)
- Consumes: `ActionService.cancelAllCountdowns()` (Task 4)
- Produces: `GET /api/coordinator/autonomy?workspace={ws}` → `{level, source}`
- Produces: `POST /api/coordinator/autonomy?level=X` → updated state
- Produces: `POST /api/coordinator/autonomy/reset` → preference default
- Produces: `CoordinatorPrompts.systemPrompt(AutonomyLevel)` → `String`

**Steps:**

- [ ] **Step 1: Write SessionAutonomyToggleTest**

Test cases (unit test against AutonomyResolver + CoordinatorResource):
- `getAutonomyReturnsPreferenceWhenNoOverride` — verify `source: "preference"`.
- `postAutonomySetsSessionOverride` — POST level=AUTONOMOUS, GET returns `source: "session"`.
- `resetClearsOverride` — set override, reset, verify preference returned.
- `switchToManualCancelsCountdowns` — set OBSERVATION, propose action (starts countdown), switch to MANUAL, verify `cancelAllCountdowns` called on ActionService.

- [ ] **Step 2: Add autonomy endpoints to CoordinatorResource**

Use `ide_insert_member` after `cancelAction`:

```java
@GET @Path("autonomy")
public Response autonomy(@QueryParam("workspace") String workspace) {
    var level = autonomyResolver.resolveLevel(workspace != null ? workspace : "default");
    var source = autonomyResolver.sessionOverride() != null ? "session" : "preference";
    return Response.ok(Map.of("level", level, "source", source)).build();
}

@POST @Path("autonomy")
public Response setAutonomy(@QueryParam("level") AutonomyLevel level) {
    var previous = autonomyResolver.resolveLevel("default");
    autonomyResolver.setSessionOverride(level);
    if (level == AutonomyLevel.MANUAL && previous != AutonomyLevel.MANUAL) {
        actionService.cancelAllCountdowns();
    }
    return autonomy(null);
}

@POST @Path("autonomy/reset")
public Response resetAutonomy() {
    var previous = autonomyResolver.sessionOverride();
    autonomyResolver.clearSessionOverride();
    if (previous != null && previous != AutonomyLevel.MANUAL) {
        actionService.cancelAllCountdowns();
    }
    return autonomy(null);
}
```

Inject `AutonomyResolver` into `CoordinatorResource`.

- [ ] **Step 3: Parameterize CoordinatorPrompts.systemPrompt()**

Use `ide_edit_member` on `systemPrompt`:

```java
public static String systemPrompt(AutonomyLevel level) {
    var base = """
        You are the Trellis Coordinator ...
        """; // existing content
    return switch (level) {
        case MANUAL -> base;
        case OBSERVATION -> base + """
            \nThe coordinator is in OBSERVATION mode. Actions will execute \
            after a countdown unless the user vetoes. Be selective — only \
            propose when confidence is high.""";
        case AUTONOMOUS -> base + """
            \nThe coordinator is in AUTONOMOUS mode. LOW-risk actions execute \
            immediately. HIGH-risk actions execute after a countdown. Only \
            propose actions when there is clear justification.""";
    };
}
```

- [ ] **Step 4: Update CoordinatorService call sites**

Use `ide_find_references` on `systemPrompt()` to find all call sites.
Update each to pass the resolved autonomy level:
`CoordinatorPrompts.systemPrompt(autonomyResolver.resolveLevel(workspace))`.

Inject `AutonomyResolver` into `CoordinatorService`.

- [ ] **Step 5: Wire approve() to reset rate limit**

In `ActionService.approve()`, after successful manual approval, call
`resetRateLimit()` — a human approve is a trust signal.

- [ ] **Step 6: Wire startup recovery**

In `CoordinatorService.start()`, after existing initialization, call
`actionService.recoverCountdowns(workspace)` for each known workspace.

- [ ] **Step 7: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl .`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#18): add session toggle, parameterized prompts, startup recovery  Refs #18"
```

---

## Task 6: Notification Wiring

**Fires platform notifications when actions auto-execute.**

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/ActionService.java`

**Interfaces:**
- Consumes: Platform notifications API (existing)
- Produces: Notification on autonomous COMPLETED or FAILED

**Steps:**

- [ ] **Step 1: Identify the notifications API**

Use `ide_find_class` to search for notification-related classes.
Use `ide_find_symbol` for `notify` or `notification` methods.
Determine the injection point and method signature.

- [ ] **Step 2: Wire notification into autoExecute**

After `executeAction()` completes (COMPLETED or FAILED), fire a
notification. The notification title should be the action type and
result ("Slot merged successfully" / "Lifecycle start failed").

Only fire for autonomous executions (autoExecute path), not for manual
approvals.

- [ ] **Step 3: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl .`
Expected: all pass

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#18): wire platform notifications for autonomous action completions  Refs #18"
```

---

## Task 7: Frontend — Mode Toggle, Countdown Timer, Auto Badge

**Adds the autonomy mode toggle, countdown timer rendering, and auto
badge to the coordinator panel.**

**Files:**
- Modify: `sidecar/src/main/webui/src/components/coordinator-panel.ts`

**Interfaces:**
- Consumes: `GET/POST /api/coordinator/autonomy` (Task 5)
- Consumes: `coordinator:action` SSE with `countdownEndsAt` (Task 4)

**Steps:**

- [ ] **Step 1: Add AutonomyLevel type and state**

Add to the top of `coordinator-panel.ts`:

```typescript
type AutonomyLevel = 'MANUAL' | 'OBSERVATION' | 'AUTONOMOUS';

interface AutonomyState {
    level: AutonomyLevel;
    source: 'session' | 'preference';
}
```

Add `@state()` field: `private autonomy: AutonomyState = { level: 'MANUAL', source: 'preference' };`

- [ ] **Step 2: Fetch autonomy state on connectedCallback**

In `connectedCallback`, after existing setup, fetch:
`GET /api/coordinator/autonomy?workspace=${this.workspaceRoot}`
and set `this.autonomy`.

- [ ] **Step 3: Add mode toggle to header**

In the `render()` method, add a three-button toggle group in the panel
header. Each button calls `POST /api/coordinator/autonomy?level=X` and
updates `this.autonomy`. Show a reset link when `source === 'session'`.

Style the active button distinctly. Use `MANUAL` / `OBS` / `AUTO` as
compact labels.

- [ ] **Step 4: Add countdownEndsAt to ProposedAction interface**

```typescript
interface ProposedAction {
    // ... existing fields
    countdownEndsAt: string | null;
}
```

- [ ] **Step 5: Add countdown timer to advice cards**

For cards where `actions.get(advice.actionKey)?.countdownEndsAt` is set
and status is `PROPOSED`, render a circular countdown timer. Use a CSS
animation based on the remaining seconds. Show "Approve Now" and "Veto"
buttons alongside the timer.

Start a `setInterval(1000)` when the card mounts with a countdown to
update the visual. Clear on unmount or when status changes.

- [ ] **Step 6: Add auto badge to completed cards**

When a COMPLETED card's action was auto-executed (no manual approval in
the transition history — approximated by checking if the elapsed time
between PROPOSED and COMPLETED is less than the countdown duration + 5s),
show a small "auto" badge.

Simpler approach: track which actions were auto-executed by checking if
the action transitioned from PROPOSED directly to APPROVED→COMPLETED
without user interaction. Use a `Set<string>` of auto-executed action IDs,
populated when `autoExecute` broadcasts the APPROVED transition.

- [ ] **Step 7: Handle SSE reconnect countdown recovery**

When the SSE reconnects, re-fetch
`GET /api/coordinator/actions?workspace=${this.workspaceRoot}` and check
each PROPOSED action for `countdownEndsAt`. Restart countdown timers for
any with future deadlines.

- [ ] **Step 8: Build and verify**

```bash
cd /Users/mdproctor/claude/hortora/trellis/sidecar/src/main/webui && yarn build
```

Expected: no build errors.

- [ ] **Step 9: Manual browser verification**

Start the sidecar in dev mode, verify:
- Mode toggle renders and switches
- Countdown timer appears on observation-mode actions
- Auto badge shows on autonomously completed cards
- Timer recovers after page refresh

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#18): add coordinator panel mode toggle, countdown timer, auto badge  Refs #18"
```

---

## Dependency Summary

```
Task 1 (enums, prefs, resolver) ──┬──→ Task 4 (autonomy integration)
                                  │         ↑
Task 2 (schema, CAS) ────────────┘         │
                                            │
Task 3 (countdown scheduler) ──────────────┘
                                            │
Task 4 ──→ Task 5 (service, resource, prompts)
           Task 6 (notifications)
           Task 7 (frontend)

Tasks 5, 6, 7 can run in parallel after Task 4.
```

## Parallelism Guide

| Phase | Tasks | Parallel? |
|-------|-------|-----------|
| 1 | Tasks 1, 2, 3 | Yes — independent foundations |
| 2 | Task 4 | Sequential — integrates 1+2+3 |
| 3 | Tasks 5, 6, 7 | Yes — independent consumers of Task 4 |
