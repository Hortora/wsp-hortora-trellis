# Slot Lifecycle Step Progress Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #68 — Slot modal: show work lifecycle steps and captured output
**Issue group:** #68

**Goal:** Show real-time step-by-step progress for lifecycle operations (end, pause, resume) in the slot detail modal sidebar, with expandable captured output per step.

**Architecture:** New `LifecycleOperationTracker` CDI bean manages operation state with file-backed durability. `SlotAgentCoordinator` gains async orchestrator methods that publish step progress via the existing SSE `EventBroadcaster`. `LifecycleResource` returns HTTP 202 with initial `OperationProgress`. Frontend `slot-detail.ts` subscribes to `lifecycle:progress` SSE topic and renders a new sidebar section with done/active/pending/failed steps.

**Tech Stack:** Java 21, Quarkus 3.x, Jackson, Lit, TypeScript, SSE via casehub-pages-push `EventBroadcaster`

## Global Constraints

- Java 21 records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Existing synchronous methods in `LifecycleManager` and `SlotAgentCoordinator` are retained — the `LifecycleActionExecutor` path depends on them
- `start` remains synchronous — no async orchestration, no step progress UI (see spec §Step definitions)
- Lock acquisition order: Semaphore(slotId) → LifecycleManager lock(workspaceRoot), never reversed
- Copy-on-write immutability for all `OperationProgress`/`StepProgress` instances
- File-backed durability: atomic rename writes to `.trellis/operations/`

---

## Batch 1: Foundation — Records, Tracker, and ScriptRunner

### Task 1: Records and StepFailedException

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/StepState.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationState.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/StepProgress.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationProgress.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/StepFailedException.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationResult.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/ScriptRunner.java:63-73`
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/ScriptRunnerTest.java`

**Interfaces:**
- Produces: `StepState` enum (`PENDING`, `RUNNING`, `DONE`, `FAILED`), `OperationState` enum (`RUNNING`, `COMPLETED`, `FAILED`), `StepProgress` record, `OperationProgress` record, `StepFailedException` checked exception, `OperationResult.rawStdout()` accessor

- [ ] **Step 1: Create the enums**

Create `StepState.java`:
```java
package io.hortora.trellis.lifecycle;

public enum StepState { PENDING, RUNNING, DONE, FAILED }
```

Create `OperationState.java`:
```java
package io.hortora.trellis.lifecycle;

public enum OperationState { RUNNING, COMPLETED, FAILED }
```

- [ ] **Step 2: Create StepProgress record**

Create `StepProgress.java`:
```java
package io.hortora.trellis.lifecycle;

import java.time.Instant;

public record StepProgress(
    String name,
    StepState state,
    String summary,
    String stdout,
    String stderr,
    Instant startedAt,
    Instant completedAt
) {
    public StepProgress(String name) {
        this(name, StepState.PENDING, name, null, null, null, null);
    }

    public StepProgress withState(StepState state) {
        return new StepProgress(name, state, summary, stdout, stderr,
                state == StepState.RUNNING ? Instant.now() : startedAt,
                state == StepState.DONE || state == StepState.FAILED ? Instant.now() : completedAt);
    }

    public StepProgress withOutput(StepState state, String stdout, String stderr) {
        return new StepProgress(name, state, summary, stdout, stderr,
                startedAt, Instant.now());
    }
}
```

- [ ] **Step 3: Create OperationProgress record**

Create `OperationProgress.java`:
```java
package io.hortora.trellis.lifecycle;

import java.time.Instant;
import java.util.List;

public record OperationProgress(
    String operationId,
    String operationType,
    String slotId,
    OperationState state,
    List<StepProgress> steps,
    Instant startedAt,
    Instant completedAt,
    String errorMessage
) {
    public OperationProgress withStep(String stepName, StepProgress updated) {
        var newSteps = steps.stream()
                .map(s -> s.name().equals(stepName) ? updated : s)
                .toList();
        return new OperationProgress(operationId, operationType, slotId,
                state, List.copyOf(newSteps), startedAt, completedAt, errorMessage);
    }

    public OperationProgress withState(OperationState state) {
        return new OperationProgress(operationId, operationType, slotId,
                state, steps, startedAt,
                state != OperationState.RUNNING ? Instant.now() : completedAt,
                errorMessage);
    }

    public OperationProgress withFailed(String errorMessage) {
        var fixedSteps = steps.stream()
                .map(s -> s.state() == StepState.RUNNING
                        ? s.withOutput(StepState.FAILED, null, errorMessage)
                        : s)
                .toList();
        return new OperationProgress(operationId, operationType, slotId,
                OperationState.FAILED, List.copyOf(fixedSteps), startedAt,
                Instant.now(), errorMessage);
    }
}
```

- [ ] **Step 4: Create StepFailedException**

Create `StepFailedException.java`:
```java
package io.hortora.trellis.lifecycle;

public class StepFailedException extends Exception {
    public StepFailedException(String stepName) {
        super("Step failed: " + stepName);
    }
}
```

- [ ] **Step 5: Add rawStdout to OperationResult**

Modify `OperationResult.java` — add `rawStdout` field:
```java
public record OperationResult(
        boolean success,
        int exitCode,
        Map<String, String> output,
        String stderr,
        String rawStdout
) {}
```

- [ ] **Step 6: Update ScriptRunner to capture rawStdout**

Modify `ScriptRunner.java:63-73` — pass raw stdout to OperationResult:
```java
var stdout = stdoutFuture.join();
var stderr = stderrFuture.join();
int exitCode = process.exitValue();

var output = parseKeyValues(stdout);

if (exitCode != 0) {
    LOG.warnf("Script %s/%s exited with code %d: %s", skillDir, scriptName, exitCode, stderr);
}

return new OperationResult(exitCode == 0, exitCode, output, stderr, stdout);
```

- [ ] **Step 7: Fix existing ScriptRunner test**

Update `ScriptRunnerTest.java` — existing tests create `OperationResult` directly. Add the 5th `rawStdout` parameter to any existing test constructors.

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ScriptRunnerTest`
Expected: PASS

- [ ] **Step 8: Fix any other OperationResult construction sites**

Search for `new OperationResult(` across the codebase and add the `rawStdout` parameter. The callers that construct `OperationResult` manually (typically in tests like `LifecycleManagerTest`, `SlotAgentCoordinatorTest`) need updating.

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml compile`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/main/java/io/hortora/trellis/lifecycle/StepState.java sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationState.java sidecar/src/main/java/io/hortora/trellis/lifecycle/StepProgress.java sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationProgress.java sidecar/src/main/java/io/hortora/trellis/lifecycle/StepFailedException.java sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationResult.java sidecar/src/main/java/io/hortora/trellis/lifecycle/ScriptRunner.java sidecar/src/test/
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#68): add lifecycle progress records, StepFailedException, OperationResult.rawStdout Refs #68"
```

### Task 2: LifecycleOperationTracker

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleOperationTracker.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleOperationTrackerTest.java`

**Interfaces:**
- Consumes: `StepState`, `OperationState`, `StepProgress`, `OperationProgress` from Task 1; `EventBroadcaster` from casehub-pages-push
- Produces: `LifecycleOperationTracker` CDI bean with methods: `startOperation(type, slotId, stepNames) → String operationId`, `stepStarted(operationId, stepName)`, `stepCompleted(operationId, stepName, stdout, stderr)`, `stepFailed(operationId, stepName, stdout, stderr)`, `operationCompleted(operationId)`, `operationFailed(operationId, errorMessage)`, `getProgress(operationId) → OperationProgress`, `getActiveOperation(slotId) → OperationProgress`

- [ ] **Step 1: Write the test — startOperation creates progress with all steps PENDING**

Create `LifecycleOperationTrackerTest.java`:
```java
package io.hortora.trellis.lifecycle;

import io.casehub.pages.push.EventBroadcaster;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class LifecycleOperationTrackerTest {

    private LifecycleOperationTracker tracker;
    private StubBroadcaster broadcaster;
    @TempDir Path tempDir;

    @BeforeEach
    void setUp() {
        broadcaster = new StubBroadcaster();
        tracker = new LifecycleOperationTracker(broadcaster, tempDir);
    }

    @Test
    void startOperationCreatesAllStepsPending() {
        var opId = tracker.startOperation("end", "3",
                List.of("stop-agents", "rebase", "push", "stamp"));
        var progress = tracker.getProgress(opId);

        assertNotNull(progress);
        assertEquals("end", progress.operationType());
        assertEquals("3", progress.slotId());
        assertEquals(OperationState.RUNNING, progress.state());
        assertEquals(4, progress.steps().size());
        assertTrue(progress.steps().stream()
                .allMatch(s -> s.state() == StepState.PENDING));
        assertEquals("stop-agents", progress.steps().get(0).name());
    }

    static class StubBroadcaster extends EventBroadcaster {
        List<String> topics = new java.util.ArrayList<>();
        List<Object> payloads = new java.util.ArrayList<>();

        StubBroadcaster() { super(null); }

        @Override
        public void broadcast(String topic, Object payload) {
            topics.add(topic);
            payloads.add(payload);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleOperationTrackerTest#startOperationCreatesAllStepsPending`
Expected: FAIL — `LifecycleOperationTracker` class does not exist

- [ ] **Step 3: Implement LifecycleOperationTracker**

Create `LifecycleOperationTracker.java`:
```java
package io.hortora.trellis.lifecycle;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.pages.push.EventBroadcaster;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Stream;

@ApplicationScoped
public class LifecycleOperationTracker {

    private static final Logger LOG = Logger.getLogger(LifecycleOperationTracker.class);
    private static final ObjectMapper MAPPER = new ObjectMapper()
            .registerModule(new JavaTimeModule());
    private static final Duration COMPLETED_TTL = Duration.ofMinutes(5);
    private static final Duration FAILED_TTL = Duration.ofHours(1);

    private final ConcurrentHashMap<String, OperationProgress> operations = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> slotToOperation = new ConcurrentHashMap<>();
    private final EventBroadcaster broadcaster;
    private final Path operationsDir;

    @Inject
    public LifecycleOperationTracker(EventBroadcaster broadcaster,
            @ConfigProperty(name = "trellis.data.dir", defaultValue = ".trellis")
            Path dataDir) {
        this.broadcaster = broadcaster;
        this.operationsDir = dataDir.resolve("operations");
    }

    LifecycleOperationTracker(EventBroadcaster broadcaster, Path operationsDir) {
        this.broadcaster = broadcaster;
        this.operationsDir = operationsDir;
    }

    public String startOperation(String type, String slotId, List<String> stepNames) {
        var existing = slotToOperation.get(slotId);
        if (existing != null) {
            var prev = operations.get(existing);
            if (prev != null && prev.state() != OperationState.RUNNING) {
                operations.remove(existing);
                slotToOperation.remove(slotId);
                deleteFile(existing);
            }
        }

        var operationId = UUID.randomUUID().toString();
        var steps = stepNames.stream()
                .map(StepProgress::new)
                .toList();
        var progress = new OperationProgress(operationId, type, slotId,
                OperationState.RUNNING, List.copyOf(steps), Instant.now(), null, null);
        operations.put(operationId, progress);
        slotToOperation.put(slotId, operationId);
        persist(progress);
        return operationId;
    }

    public void stepStarted(String operationId, String stepName) {
        operations.computeIfPresent(operationId, (id, op) -> {
            var step = findStep(op, stepName).withState(StepState.RUNNING);
            var updated = op.withStep(stepName, step);
            persist(updated);
            broadcastStep(updated, step);
            return updated;
        });
    }

    public void stepCompleted(String operationId, String stepName,
                              String stdout, String stderr) {
        operations.computeIfPresent(operationId, (id, op) -> {
            var step = findStep(op, stepName).withOutput(StepState.DONE, stdout, stderr);
            var updated = op.withStep(stepName, step);
            persist(updated);
            broadcastStep(updated, step);
            return updated;
        });
    }

    public void stepFailed(String operationId, String stepName,
                           String stdout, String stderr) {
        operations.computeIfPresent(operationId, (id, op) -> {
            var step = findStep(op, stepName).withOutput(StepState.FAILED, stdout, stderr);
            var updated = op.withStep(stepName, step).withState(OperationState.FAILED);
            persist(updated);
            broadcastStep(updated, step);
            return updated;
        });
    }

    public void operationCompleted(String operationId) {
        operations.computeIfPresent(operationId, (id, op) -> {
            var updated = op.withState(OperationState.COMPLETED);
            persist(updated);
            broadcastOperation(updated);
            return updated;
        });
    }

    public void operationFailed(String operationId, String errorMessage) {
        operations.computeIfPresent(operationId, (id, op) -> {
            var updated = op.withFailed(errorMessage);
            persist(updated);
            broadcastOperation(updated);
            return updated;
        });
    }

    public OperationProgress getProgress(String operationId) {
        return operations.get(operationId);
    }

    public OperationProgress getActiveOperation(String slotId) {
        var opId = slotToOperation.get(slotId);
        return opId != null ? operations.get(opId) : null;
    }

    @Scheduled(every = "60s")
    void evictStale() {
        var now = Instant.now();
        operations.entrySet().removeIf(e -> {
            var op = e.getValue();
            if (op.state() == OperationState.COMPLETED
                    && op.completedAt() != null
                    && Duration.between(op.completedAt(), now).compareTo(COMPLETED_TTL) > 0) {
                slotToOperation.remove(op.slotId(), e.getKey());
                deleteFile(e.getKey());
                return true;
            }
            if (op.state() == OperationState.FAILED
                    && op.completedAt() != null
                    && Duration.between(op.completedAt(), now).compareTo(FAILED_TTL) > 0) {
                slotToOperation.remove(op.slotId(), e.getKey());
                deleteFile(e.getKey());
                return true;
            }
            return false;
        });
    }

    void loadOnStartup() {
        if (!Files.isDirectory(operationsDir)) return;
        try (Stream<Path> files = Files.list(operationsDir)) {
            files.filter(f -> f.toString().endsWith(".json")).forEach(f -> {
                try {
                    var op = MAPPER.readValue(f.toFile(), OperationProgress.class);
                    if (op.state() == OperationState.RUNNING) {
                        op = op.withFailed("Sidecar restarted during operation");
                    }
                    operations.put(op.operationId(), op);
                    slotToOperation.put(op.slotId(), op.operationId());
                } catch (IOException e) {
                    LOG.warnf("Failed to load operation file %s: %s", f, e.getMessage());
                }
            });
        } catch (IOException e) {
            LOG.warnf("Failed to scan operations directory: %s", e.getMessage());
        }
    }

    private StepProgress findStep(OperationProgress op, String stepName) {
        return op.steps().stream()
                .filter(s -> s.name().equals(stepName))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Unknown step: " + stepName));
    }

    private void persist(OperationProgress op) {
        try {
            Files.createDirectories(operationsDir);
            var tmp = operationsDir.resolve(op.operationId() + ".tmp");
            var target = operationsDir.resolve(op.operationId() + ".json");
            MAPPER.writeValue(tmp.toFile(), op);
            Files.move(tmp, target, StandardCopyOption.ATOMIC_MOVE,
                    StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            LOG.warnf("Failed to persist operation %s: %s", op.operationId(), e.getMessage());
        }
    }

    private void deleteFile(String operationId) {
        try {
            Files.deleteIfExists(operationsDir.resolve(operationId + ".json"));
        } catch (IOException e) {
            LOG.debugf("Failed to delete operation file %s: %s", operationId, e.getMessage());
        }
    }

    private void broadcastStep(OperationProgress op, StepProgress step) {
        broadcaster.broadcast("lifecycle:progress", Map.of(
                "operationId", op.operationId(),
                "slotId", op.slotId(),
                "operationType", op.operationType(),
                "step", step.name(),
                "state", step.state().name(),
                "stdout", step.stdout() != null ? step.stdout() : "",
                "stderr", step.stderr() != null ? step.stderr() : "",
                "operationState", op.state().name()));
    }

    private void broadcastOperation(OperationProgress op) {
        var payload = new java.util.HashMap<String, Object>();
        payload.put("operationId", op.operationId());
        payload.put("slotId", op.slotId());
        payload.put("operationType", op.operationType());
        payload.put("step", null);
        payload.put("state", null);
        payload.put("operationState", op.state().name());
        payload.put("errorMessage", op.errorMessage());
        broadcaster.broadcast("lifecycle:progress", payload);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleOperationTrackerTest#startOperationCreatesAllStepsPending`
Expected: PASS

- [ ] **Step 5: Add remaining tracker tests**

Add to `LifecycleOperationTrackerTest.java`:
```java
@Test
void stepStartedTransitionsToRunning() {
    var opId = tracker.startOperation("end", "3", List.of("rebase", "push"));
    tracker.stepStarted(opId, "rebase");
    var progress = tracker.getProgress(opId);

    assertEquals(StepState.RUNNING, progress.steps().get(0).state());
    assertEquals(StepState.PENDING, progress.steps().get(1).state());
    assertNotNull(progress.steps().get(0).startedAt());
    assertEquals(1, broadcaster.topics.size());
    assertEquals("lifecycle:progress", broadcaster.topics.get(0));
}

@Test
void stepCompletedCapturesOutput() {
    var opId = tracker.startOperation("end", "3", List.of("rebase"));
    tracker.stepStarted(opId, "rebase");
    tracker.stepCompleted(opId, "rebase", "SYNCED=yes", "");
    var step = tracker.getProgress(opId).steps().get(0);

    assertEquals(StepState.DONE, step.state());
    assertEquals("SYNCED=yes", step.stdout());
}

@Test
void stepFailedMarksOperationFailed() {
    var opId = tracker.startOperation("end", "3", List.of("rebase", "push"));
    tracker.stepStarted(opId, "rebase");
    tracker.stepFailed(opId, "rebase", "", "error: conflict");
    var progress = tracker.getProgress(opId);

    assertEquals(StepState.FAILED, progress.steps().get(0).state());
    assertEquals(OperationState.FAILED, progress.state());
    assertEquals("error: conflict", progress.steps().get(0).stderr());
}

@Test
void operationFailedTransitionsRunningStepToFailed() {
    var opId = tracker.startOperation("end", "3", List.of("rebase", "push"));
    tracker.stepStarted(opId, "rebase");
    tracker.operationFailed(opId, "Script timed out");
    var progress = tracker.getProgress(opId);

    assertEquals(StepState.FAILED, progress.steps().get(0).state());
    assertEquals(OperationState.FAILED, progress.state());
    assertEquals("Script timed out", progress.errorMessage());
}

@Test
void getActiveOperationBySlotId() {
    var opId = tracker.startOperation("end", "3", List.of("rebase"));
    var active = tracker.getActiveOperation("3");

    assertNotNull(active);
    assertEquals(opId, active.operationId());
}

@Test
void startOperationAutoDismissesPreviousCompleted() {
    var opId1 = tracker.startOperation("end", "3", List.of("rebase"));
    tracker.operationCompleted(opId1);
    var opId2 = tracker.startOperation("pause", "3", List.of("commit-wip"));

    assertNull(tracker.getProgress(opId1));
    assertNotNull(tracker.getProgress(opId2));
}

@Test
void filePersistenceOnStartup() throws Exception {
    var opId = tracker.startOperation("end", "3", List.of("rebase"));
    tracker.stepStarted(opId, "rebase");

    var tracker2 = new LifecycleOperationTracker(broadcaster, tempDir);
    tracker2.loadOnStartup();
    var recovered = tracker2.getProgress(opId);

    assertNotNull(recovered);
    assertEquals(OperationState.FAILED, recovered.state());
    assertEquals(StepState.FAILED, recovered.steps().get(0).state());
}

@Test
void operationCompletedTransitionsState() {
    var opId = tracker.startOperation("end", "3", List.of("rebase"));
    tracker.stepStarted(opId, "rebase");
    tracker.stepCompleted(opId, "rebase", "", "");
    tracker.operationCompleted(opId);
    var progress = tracker.getProgress(opId);

    assertEquals(OperationState.COMPLETED, progress.state());
    assertNotNull(progress.completedAt());
}
```

- [ ] **Step 6: Run all tracker tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleOperationTrackerTest`
Expected: PASS (all 8 tests)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleOperationTracker.java sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleOperationTrackerTest.java
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#68): LifecycleOperationTracker with file-backed durability and SSE broadcasting Refs #68"
```

## Batch 2: Async Orchestration — LifecycleManager + SlotAgentCoordinator

### Task 3: LifecycleManager async step methods

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleManager.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleManagerTest.java`

**Interfaces:**
- Consumes: `LifecycleOperationTracker` from Task 2, `StepFailedException` from Task 1
- Produces: `endSteps(slotId, workspaceRoot, tracker, operationId)`, `pauseSteps(slotId, workspaceRoot, tracker, operationId)`, `resumeSteps(slotId, workspaceRoot, tracker, operationId)` — all throw `IOException`, `InterruptedException`, `ConcurrentOperationException`, `StepFailedException`

- [ ] **Step 1: Write the test — endSteps reports step progress**

Add to `LifecycleManagerTest.java` (or create if it doesn't exist). Use a stub `LifecycleOperationTracker` or the real one with a `StubBroadcaster`:

```java
@Test
void endStepsReportsProgressToTracker() throws Exception {
    var broadcaster = new StubBroadcaster();
    var tracker = new LifecycleOperationTracker(broadcaster, tempDir);
    var opId = tracker.startOperation("end", "1",
            List.of("rebase", "push", "stamp"));

    // stubScriptRunner returns success for all scripts
    manager.endSteps("1", workspaceRoot, tracker, opId);

    var progress = tracker.getProgress(opId);
    assertEquals(3, progress.steps().size());
    assertTrue(progress.steps().stream()
            .allMatch(s -> s.state() == StepState.DONE));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleManagerTest#endStepsReportsProgressToTracker`
Expected: FAIL — `endSteps` method does not exist

- [ ] **Step 3: Implement endSteps, pauseSteps, resumeSteps**

Add to `LifecycleManager.java`:

```java
public void endSteps(String slotId, Path workspaceRoot,
                     LifecycleOperationTracker tracker, String operationId)
        throws IOException, InterruptedException, ConcurrentOperationException,
               StepFailedException {
    if (!tryLock(workspaceRoot.toString())) {
        throw new ConcurrentOperationException("Workspace operation in progress");
    }
    try {
        runTrackedStep(tracker, operationId, "rebase",
                () -> scriptRunner.run("work-end", "land_branch.py",
                        List.of("rebase", workspaceRoot.toString())));
        runTrackedStep(tracker, operationId, "push",
                () -> scriptRunner.run("work-end", "land_branch.py",
                        List.of("push", workspaceRoot.toString())));
        runTrackedStep(tracker, operationId, "stamp",
                () -> scriptRunner.run("work-end", "land_branch.py",
                        List.of("stamp", workspaceRoot.toString())));
        fireWorkspaceChanged(workspaceRoot);
    } finally {
        unlock(workspaceRoot.toString());
    }
}

public void pauseSteps(String slotId, Path workspaceRoot,
                       LifecycleOperationTracker tracker, String operationId)
        throws IOException, InterruptedException, ConcurrentOperationException,
               StepFailedException {
    if (!tryLock(workspaceRoot.toString())) {
        throw new ConcurrentOperationException("Workspace operation in progress");
    }
    try {
        runTrackedStep(tracker, operationId, "commit-wip",
                () -> scriptRunner.run("work-pause", "pause_exec.py",
                        List.of("commit-wip", workspaceRoot.toString())));
        runTrackedStep(tracker, operationId, "push-and-stack",
                () -> scriptRunner.run("work-pause", "pause_exec.py",
                        List.of("push-and-stack", workspaceRoot.toString())));
        fireWorkspaceChanged(workspaceRoot);
    } finally {
        unlock(workspaceRoot.toString());
    }
}

public void resumeSteps(String slotId, Path workspaceRoot,
                        LifecycleOperationTracker tracker, String operationId)
        throws IOException, InterruptedException, ConcurrentOperationException,
               StepFailedException {
    if (!tryLock(workspaceRoot.toString())) {
        throw new ConcurrentOperationException("Workspace operation in progress");
    }
    try {
        runTrackedStep(tracker, operationId, "checkout-branches",
                () -> scriptRunner.run("work-resume", "resume_exec.py",
                        List.of("checkout-branches", workspaceRoot.toString())));
        runTrackedStep(tracker, operationId, "rebase",
                () -> scriptRunner.run("work-resume", "resume_exec.py",
                        List.of("rebase", workspaceRoot.toString())));
        runTrackedStep(tracker, operationId, "reset-wip",
                () -> scriptRunner.run("work-resume", "resume_exec.py",
                        List.of("reset-wip", workspaceRoot.toString())));
        fireWorkspaceChanged(workspaceRoot);
    } finally {
        unlock(workspaceRoot.toString());
    }
}

private void runTrackedStep(LifecycleOperationTracker tracker, String operationId,
                            String stepName, ScriptOperation scriptOp)
        throws IOException, InterruptedException, StepFailedException {
    tracker.stepStarted(operationId, stepName);
    var result = scriptOp.run();
    if (!result.success()) {
        tracker.stepFailed(operationId, stepName, result.rawStdout(), result.stderr());
        throw new StepFailedException(stepName);
    }
    tracker.stepCompleted(operationId, stepName, result.rawStdout(), result.stderr());
}

@FunctionalInterface
interface ScriptOperation {
    OperationResult run() throws IOException, InterruptedException;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleManagerTest#endStepsReportsProgressToTracker`
Expected: PASS

- [ ] **Step 5: Add test for step failure halting operation**

```java
@Test
void endStepsThrowsStepFailedOnScriptFailure() {
    // stubScriptRunner returns failure for "push" step
    var tracker = new LifecycleOperationTracker(new StubBroadcaster(), tempDir);
    var opId = tracker.startOperation("end", "1",
            List.of("rebase", "push", "stamp"));

    assertThrows(StepFailedException.class,
            () -> manager.endSteps("1", workspaceRoot, tracker, opId));

    var progress = tracker.getProgress(opId);
    assertEquals(StepState.DONE, progress.steps().get(0).state());
    assertEquals(StepState.FAILED, progress.steps().get(1).state());
    assertEquals(StepState.PENDING, progress.steps().get(2).state());
}
```

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleManagerTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleManager.java sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleManagerTest.java
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#68): LifecycleManager async step methods with tracked progress Refs #68"
```

### Task 4: SlotAgentCoordinator async orchestrators + REST API

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/SlotAgentCoordinator.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleResource.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/SlotAgentCoordinatorTest.java`

**Interfaces:**
- Consumes: `LifecycleOperationTracker` from Task 2, `LifecycleManager.endSteps/pauseSteps/resumeSteps` from Task 3
- Produces: `coordinatedEndAsync(slotId, workspaceRoot) → OperationProgress`, `coordinatedPauseAsync(slotId, workspaceRoot) → OperationProgress`, `coordinatedResumeAsync(slotId, workspaceRoot) → OperationProgress`; REST endpoints returning HTTP 202 with `OperationProgress`; `GET /api/lifecycle/operations/{operationId}` and `GET /api/lifecycle/operations?slot={slotId}`

- [ ] **Step 1: Write the test — coordinatedEndAsync returns progress immediately**

Add to `SlotAgentCoordinatorTest.java`:
```java
@Test
void coordinatedEndAsyncReturnsProgressImmediately() {
    var progress = coordinator.coordinatedEndAsync("3", workspaceRoot);

    assertNotNull(progress);
    assertEquals("end", progress.operationType());
    assertEquals("3", progress.slotId());
    assertEquals(OperationState.RUNNING, progress.state());
    assertEquals(4, progress.steps().size());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=SlotAgentCoordinatorTest#coordinatedEndAsyncReturnsProgressImmediately`
Expected: FAIL — method does not exist

- [ ] **Step 3: Implement async orchestrators in SlotAgentCoordinator**

Add to `SlotAgentCoordinator.java`:
- Inject `LifecycleOperationTracker tracker` and `ManagedExecutor executor`
- Change `slotLocks` from `ConcurrentHashMap<String, ReentrantLock>` to `ConcurrentHashMap<String, Semaphore>`
- Add `coordinatedEndAsync`, `coordinatedPauseAsync`, `coordinatedResumeAsync`
- Add `fireLifecycleOperationEvent` and `fireWorkspaceChanged` helper methods (CDI events moved from LifecycleManager to coordinator for async path)
- Keep existing synchronous methods unchanged (they still use `ReentrantLock` — add a separate `syncLocks` map if needed to avoid conflict with the new `Semaphore` map)

Follow the exact pattern from the spec §SlotAgentCoordinator changes code block.

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=SlotAgentCoordinatorTest#coordinatedEndAsyncReturnsProgressImmediately`
Expected: PASS

- [ ] **Step 5: Update LifecycleResource — async endpoints + operations query**

Modify `LifecycleResource.java`:
- Inject `LifecycleOperationTracker tracker`
- Change `end()`, `pause()`, `resume()` to call `coordinator.coordinatedXxxAsync()` and return HTTP 202 with `OperationProgress`
- Add `GET /api/lifecycle/operations/{operationId}` → `tracker.getProgress(operationId)`
- Add `GET /api/lifecycle/operations?slot={slotId}` → `tracker.getActiveOperation(slotId)`
- Keep `start()` unchanged (synchronous)

```java
@POST
@Path("/end/{slotId}")
@Consumes(MediaType.APPLICATION_JSON)
public Response end(@PathParam("slotId") String slotId, WorkspaceRequest request) {
    try {
        var progress = coordinator.coordinatedEndAsync(slotId,
                java.nio.file.Path.of(request.workspaceRoot()));
        return Response.accepted(progress).build();
    } catch (ConcurrentOperationException e) {
        return Response.status(Response.Status.CONFLICT)
                .entity(Map.of("error", e.getMessage())).build();
    }
}

@GET
@Path("/operations/{operationId}")
public Response getOperation(@PathParam("operationId") String operationId) {
    var progress = tracker.getProgress(operationId);
    if (progress == null) return Response.status(Response.Status.NOT_FOUND).build();
    return Response.ok(progress).build();
}

@GET
@Path("/operations")
public Response getOperationBySlot(@QueryParam("slot") String slotId) {
    var progress = tracker.getActiveOperation(slotId);
    if (progress == null) return Response.ok().build();
    return Response.ok(progress).build();
}
```

- [ ] **Step 6: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=SlotAgentCoordinatorTest,LifecycleManagerTest,LifecycleOperationTrackerTest`
Expected: PASS

- [ ] **Step 7: Compile full project**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml compile`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/main/java/io/hortora/trellis/lifecycle/ sidecar/src/test/java/io/hortora/trellis/lifecycle/
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#68): async SlotAgentCoordinator orchestrators + REST API for lifecycle operations Refs #68"
```

## Batch 3: Frontend — Lifecycle Sidebar Section

### Task 5: Frontend lifecycle progress display

**Files:**
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts`

**Interfaces:**
- Consumes: `POST /api/lifecycle/{end|pause|resume}/{slotId}` returning `OperationProgress` (HTTP 202), `GET /api/lifecycle/operations?slot={slotId}`, SSE topic `lifecycle:progress`

- [ ] **Step 1: Add TypeScript interfaces and state**

At the top of `slot-detail.ts`, add after the existing `AgentSnapshot` interface:

```typescript
interface StepProgress {
  name: string;
  state: 'PENDING' | 'RUNNING' | 'DONE' | 'FAILED';
  summary: string;
  stdout: string | null;
  stderr: string | null;
}

interface OperationProgress {
  operationId: string;
  operationType: string;
  slotId: string;
  state: 'RUNNING' | 'COMPLETED' | 'FAILED';
  steps: StepProgress[];
  errorMessage?: string;
}
```

Add state properties to `TrellisSlotDetail`:
```typescript
@state() private _operation: OperationProgress | null = null;
@state() private _expandedStep: string | null = null;
```

- [ ] **Step 2: Update _subscribeEvents to include lifecycle:progress**

Modify `_subscribeEvents()` — add `lifecycle:progress` to the EventSource URL:
```typescript
this._eventSource = new EventSource('/api/push?topics=agent:state,agent:eviction,lifecycle:progress');
```

Add handling in the message listener for `lifecycle:progress` events:
```typescript
if (msg.topic === 'lifecycle:progress') {
  const data = typeof msg.data === 'string' ? JSON.parse(msg.data) : msg.data;
  if (data.slotId === String(this.slotNumber)) {
    if (data.step) {
      // Step-level update
      if (this._operation) {
        this._operation = {
          ...this._operation,
          state: data.operationState,
          steps: this._operation.steps.map(s =>
            s.name === data.step
              ? { ...s, state: data.state, stdout: data.stdout, stderr: data.stderr }
              : s
          ),
        };
      }
    } else {
      // Operation-level update
      if (this._operation) {
        this._operation = {
          ...this._operation,
          state: data.operationState,
          errorMessage: data.errorMessage,
        };
      }
      if (data.operationState === 'COMPLETED' || data.operationState === 'FAILED') {
        this._actionInProgress = null;
        this._loadSlot();
        this._loadTerminals();
        if (data.operationState === 'COMPLETED') {
          setTimeout(() => { this._operation = null; }, 10000);
        }
      }
    }
  }
}
```

- [ ] **Step 3: Add lifecycle operation restore on load**

In `_loadSlot()`, after the slot data fetch, also fetch active operation:
```typescript
try {
  const opRes = await fetch(`/api/lifecycle/operations?slot=${this.slotNumber}`);
  if (opRes.ok) {
    const body = await opRes.json();
    if (body && body.operationId) {
      this._operation = body as OperationProgress;
      if (body.state === 'RUNNING') {
        this._actionInProgress = body.operationType;
      }
    }
  }
} catch { /* ignore */ }
```

- [ ] **Step 4: Update _lifecycleAction to use async flow**

Replace the current `_lifecycleAction` method:
```typescript
private async _lifecycleAction(name: string, url: string) {
  this._actionInProgress = name;
  this._expandedStep = null;
  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ workspaceRoot: this.workspaceRoot }),
    });
    if (!res.ok) {
      const body = await res.json().catch(() => null);
      this._error = body?.error ?? `${name} failed: HTTP ${res.status}`;
      this._actionInProgress = null;
      return;
    }
    this._operation = await res.json() as OperationProgress;
  } catch (e) {
    this._error = `${name} failed: ${e}`;
    this._actionInProgress = null;
  }
}
```

- [ ] **Step 5: Add CSS styles for lifecycle section**

Add to `static override styles`:
```css
.lifecycle-step {
  display: flex; align-items: baseline; gap: 0.4rem;
  font-size: 0.8rem; padding: 0.15rem 0;
}
.lifecycle-step-done { color: #666; }
.lifecycle-step-active { color: #e5e5e5; }
.lifecycle-step-pending { color: #555; }
.lifecycle-step-failed { color: #fca5a5; }
.lifecycle-icon-done { color: #86efac; }
.lifecycle-icon-active { color: #93c5fd; }
.lifecycle-icon-pending { color: #555; }
.lifecycle-icon-failed { color: #f87171; }

@keyframes lifecycle-spin {
  to { transform: rotate(360deg); }
}
.lifecycle-spinner {
  display: inline-block; animation: lifecycle-spin 1s linear infinite;
}

.lifecycle-expand {
  font-size: 0.7rem; color: #888; cursor: pointer; padding: 0.1rem 0;
  background: none; border: none;
}
.lifecycle-expand:hover { color: #ccc; }

.lifecycle-output {
  font-family: monospace; font-size: 0.7rem; background: #111;
  border: 1px solid #333; padding: 0.5rem; margin: 0.3rem 0 0.5rem;
  max-height: 150px; overflow-y: auto; white-space: pre-wrap;
  color: #ccc;
}

.lifecycle-type-badge {
  display: inline-flex; padding: 0.1rem 0.4rem; border-radius: 3px;
  font-size: 0.65rem; font-weight: 500; background: #1e3a5f; color: #93c5fd;
  margin-bottom: 0.5rem;
}
```

- [ ] **Step 6: Implement _renderLifecycle**

Add the rendering method:
```typescript
private _renderLifecycle() {
  const op = this._operation;
  if (!op) return nothing;

  return html`
    <div class="sidebar-section">
      <h3>Lifecycle</h3>
      <span class="lifecycle-type-badge">${op.operationType}</span>
      ${op.steps.map(s => html`
        <div class="lifecycle-step lifecycle-step-${s.state === 'RUNNING' ? 'active' : s.state.toLowerCase()}">
          <span class="${s.state === 'DONE' ? 'lifecycle-icon-done' :
                        s.state === 'RUNNING' ? 'lifecycle-icon-active lifecycle-spinner' :
                        s.state === 'FAILED' ? 'lifecycle-icon-failed' :
                        'lifecycle-icon-pending'}">
            ${s.state === 'DONE' ? '✓' :
              s.state === 'RUNNING' ? '●' :
              s.state === 'FAILED' ? '✗' : '○'}
          </span>
          <span>${s.name}</span>
        </div>
        ${(s.state === 'DONE' || s.state === 'FAILED') && (s.stdout || s.stderr) ? html`
          <button class="lifecycle-expand"
                  @click=${() => { this._expandedStep = this._expandedStep === s.name ? null : s.name; }}>
            ${this._expandedStep === s.name ? '▼' : '▶'} output
          </button>
          ${this._expandedStep === s.name ? html`
            <div class="lifecycle-output">${s.stdout || s.stderr || ''}</div>
          ` : nothing}
        ` : nothing}
        ${s.state === 'FAILED' && this._expandedStep !== s.name ? html`
          <div class="lifecycle-output" style="border-color:#991b1b">${s.stderr || s.stdout || ''}</div>
        ` : nothing}
      `)}
      ${op.errorMessage ? html`
        <div style="font-size:0.75rem;color:#f87171;margin-top:0.5rem">${op.errorMessage}</div>
      ` : nothing}
    </div>
  `;
}
```

- [ ] **Step 7: Wire _renderLifecycle into _renderSidebar**

In `_renderSidebar()`, insert `${this._renderLifecycle()}` between the Status section and the Issues section:

```typescript
// After the Status badge section, before the Issues section:
${this._renderLifecycle()}
```

- [ ] **Step 8: Build frontend**

Run: `yarn --cwd /Users/mdproctor/claude/hortora/trellis/sidecar/src/main/webui build`
Expected: BUILD SUCCESS (no TypeScript errors)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/src/main/webui/src/views/slot-detail.ts
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#68): lifecycle step progress sidebar section in slot detail modal Refs #68"
```

### Task 6: Integration verification

**Files:**
- No new files — verification only

**Interfaces:**
- Consumes: All of the above

- [ ] **Step 1: Run full backend test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: All tests PASS

- [ ] **Step 2: Build full package**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml package -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 3: Run frontend tests**

Run: `yarn --cwd /Users/mdproctor/claude/hortora/trellis/sidecar/src/main/webui test`
Expected: All tests PASS

- [ ] **Step 4: Commit any fixes**

If any compilation or test failures were found and fixed, commit them.

## References

- [2026-09-26-slot-lifecycle-steps-design.md] — design spec this plan implements
- [LifecycleManager.java] — current synchronous lifecycle methods
- [SlotAgentCoordinator.java] — current coordinated lifecycle methods
- [ScriptRunner.java] — script execution infrastructure
- [OperationResult.java] — current result record
- [LifecycleResource.java] — current REST endpoints
- [slot-detail.ts] — current slot detail modal frontend
- [workspace-sse.ts] — SSE subscription pattern
- [GitHub #68] — focal issue
- [GitHub #67] — related plan batch state spec (UI pattern precedent)
