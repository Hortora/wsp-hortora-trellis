# Slot–Agent Coordination Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #22 — Slot pause/resume ↔ agent pause/resume coordination
**Issue group:** #22

**Goal:** Wire slot pause/resume/end to automatically cascade to agent
processes, and add an advisory memory-pressure eviction queue.

**Architecture:** New `SlotAgentCoordinator` sits between REST/executor
layer and `LifecycleManager`, orchestrating agent shutdown before git ops
on pause, and agent restart after git ops on resume. Memory monitoring
extends the existing `AgentMonitorScheduler` poll cycle with an eviction
candidate queue broadcast via SSE.

**Tech Stack:** Java 21, Quarkus 3.x, Lit (TypeScript), SSE via
casehub-pages-push `EventBroadcaster`

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Lock ordering: coordinator slot lock → LifecycleManager workspace lock → AgentProcessManager terminal lock
- `@trellis_agent_state` tmux option values: `PAUSED` (user-initiated), `PAUSED_BY_COORDINATOR` (cascade-initiated)
- macOS-only — system memory via `memory_pressure` command, not JDK MXBean

---

### Task 1: Extend AgentState and AgentProcess for PAUSED_BY_COORDINATOR

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentState.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentProcess.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentProcessManager.java` (initializeFromBootstrap)
- Test: `sidecar/src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java`

**Interfaces:**
- Produces: `AgentState.PAUSED_BY_COORDINATOR` enum value, `AgentProcess.pausedByCoordinator(String command)` factory

- [ ] **Step 1: Write failing test — PAUSED_BY_COORDINATOR bootstrap recovery**

```java
@Test
void bootstrapRecoversPausedByCoordinatorState() throws Exception {
    when(tmux.getOption("t1", "@trellis_agent_state"))
            .thenReturn(Optional.of("PAUSED_BY_COORDINATOR"));
    var terminal = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    manager.initializeFromBootstrap(List.of(terminal));
    var snapshot = manager.getSnapshot("t1", terminal);
    assertNotNull(snapshot.process());
    assertEquals(AgentState.PAUSED_BY_COORDINATOR, snapshot.process().state());
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest#bootstrapRecoversPausedByCoordinatorState -DfailIfNoTests=false`
Expected: compilation error — `PAUSED_BY_COORDINATOR` does not exist

- [ ] **Step 3: Add PAUSED_BY_COORDINATOR to AgentState**

Add `PAUSED_BY_COORDINATOR` after `PAUSED` in the enum. Use `ide_edit_member`.

```java
public enum AgentState {
    IDLE,
    STARTING,
    RUNNING,
    PAUSED,
    PAUSED_BY_COORDINATOR
}
```

- [ ] **Step 4: Add factory method to AgentProcess**

Use `ide_insert_member` to add after the existing `paused` method:

```java
public static AgentProcess pausedByCoordinator(String command) {
    return new AgentProcess(0, AgentState.PAUSED_BY_COORDINATOR, 0, null, command);
}
```

- [ ] **Step 5: Update initializeFromBootstrap to recognize PAUSED_BY_COORDINATOR**

In `AgentProcessManager.initializeFromBootstrap`, change the condition from
`"PAUSED".equals(paused.get())` to handle both values:

```java
if (paused.isPresent()) {
    if ("PAUSED_BY_COORDINATOR".equals(paused.get())) {
        agents.put(terminal.name(), AgentProcess.pausedByCoordinator(null));
    } else if ("PAUSED".equals(paused.get())) {
        agents.put(terminal.name(), AgentProcess.paused(null));
    }
}
```

- [ ] **Step 6: Update processDiscovery to skip PAUSED_BY_COORDINATOR**

The monitor must not transition agents in PAUSED_BY_COORDINATOR state, same
as PAUSED. In `processDiscovery`, change:
```java
if (existingState == AgentState.PAUSED) return;
```
to:
```java
if (existingState == AgentState.PAUSED || existingState == AgentState.PAUSED_BY_COORDINATOR) return;
```

- [ ] **Step 7: Update broadcastState to handle PAUSED_BY_COORDINATOR**

The SSE broadcast sends `state` as `process.state().name()`. The frontend
will receive `"PAUSED_BY_COORDINATOR"` — the badge component needs to treat
it as paused. No backend change needed — the string propagates naturally.

- [ ] **Step 8: Run test — verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 9: Commit**

```bash
git -C sidecar add src/main/java/io/hortora/trellis/agent/AgentState.java \
  src/main/java/io/hortora/trellis/agent/AgentProcess.java \
  src/main/java/io/hortora/trellis/agent/AgentProcessManager.java \
  src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java
git -C sidecar commit -m "feat(#22): add PAUSED_BY_COORDINATOR state for cascade provenance  Refs #22"
```

---

### Task 2: Graceful Agent Shutdown

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentProcessManager.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java`

**Interfaces:**
- Consumes: `AgentState.PAUSED_BY_COORDINATOR`, `AgentProcess.pausedByCoordinator(String)` from Task 1
- Produces: `AgentProcessManager.gracefulShutdown(String terminalName)` — used by Task 3

- [ ] **Step 1: Write failing test — graceful shutdown sends Escape then /exit**

```java
@Test
void gracefulShutdownSendsEscapeThenExit() throws Exception {
    when(tmux.displayMessage("t1", "#{pane_current_command}"))
            .thenReturn("node")   // initial check in step 5
            .thenReturn("zsh");   // poll after /exit → shell appeared
    manager.setStarting("t1", "claude");
    // Simulate RUNNING state
    var terminal = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    var psOutput = "  100     1  50000 /usr/local/bin/node /usr/local/bin/claude\n";
    manager.pollTerminalWithPsOutput(terminal, psOutput);

    manager.gracefulShutdown("t1");

    var inOrder = org.mockito.Mockito.inOrder(tmux);
    inOrder.verify(tmux).sendKeys("t1", "Escape");
    inOrder.verify(tmux).sendKeys("t1", "/exit\n");
    var snapshot = manager.getSnapshot("t1", terminal);
    assertNotNull(snapshot.process());
    assertEquals(AgentState.PAUSED_BY_COORDINATOR, snapshot.process().state());
}
```

- [ ] **Step 2: Write failing test — graceful shutdown skips /exit if shell appears after Escape**

```java
@Test
void gracefulShutdownSkipsExitIfShellAppearsAfterEscape() throws Exception {
    when(tmux.displayMessage("t1", "#{pane_current_command}"))
            .thenReturn("zsh");   // shell already after Escape wait
    manager.setStarting("t1", "claude");
    var terminal = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    var psOutput = "  100     1  50000 /usr/local/bin/node /usr/local/bin/claude\n";
    manager.pollTerminalWithPsOutput(terminal, psOutput);

    manager.gracefulShutdown("t1");

    verify(tmux).sendKeys("t1", "Escape");
    verify(tmux, org.mockito.Mockito.never()).sendKeys(eq("t1"), eq("/exit\n"));
    assertEquals(AgentState.PAUSED_BY_COORDINATOR,
            manager.getSnapshot("t1", terminal).process().state());
}
```

- [ ] **Step 3: Write failing test — graceful shutdown is no-op for IDLE**

```java
@Test
void gracefulShutdownIsNoOpForIdleAgent() throws Exception {
    var terminal = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    manager.gracefulShutdown("t1");
    assertNull(manager.getSnapshot("t1", terminal).process());
}
```

- [ ] **Step 4: Write failing test — STARTING agent gets treeKill, not Escape**

```java
@Test
void gracefulShutdownUsesTreeKillForStartingAgent() throws Exception {
    manager.setStarting("t1", "claude");
    var terminal = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    // No Escape or /exit should be sent for STARTING
    manager.gracefulShutdown("t1");
    verify(tmux, org.mockito.Mockito.never()).sendKeys(eq("t1"), eq("Escape"));
    verify(tmux, org.mockito.Mockito.never()).sendKeys(eq("t1"), eq("/exit\n"));
    assertEquals(AgentState.PAUSED_BY_COORDINATOR,
            manager.getSnapshot("t1", terminal).process().state());
}
```

- [ ] **Step 5: Run tests — verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest -DfailIfNoTests=false`
Expected: compilation error — `gracefulShutdown` does not exist

- [ ] **Step 6: Implement gracefulShutdown**

Add to `AgentProcessManager` using `ide_insert_member`:

```java
public void gracefulShutdown(String terminalName) throws IOException, InterruptedException {
    var lock = lockFor(terminalName);
    lock.lock();
    try {
        var existing = agents.get(terminalName);
        if (existing == null) return;

        var state = existing.state();
        if (state != AgentState.RUNNING && state != AgentState.STARTING) return;

        if (state == AgentState.STARTING) {
            if (existing.pid() > 0) treeKill(terminalName, existing.pid());
            markPausedByCoordinator(terminalName, existing.command());
            return;
        }

        tmux.sendKeys(terminalName, "Escape");
        Thread.sleep(500);

        var cmd = tmux.displayMessage(terminalName, "#{pane_current_command}").trim();
        if (!SHELL_COMMANDS.contains(cmd)) {
            tmux.sendKeys(terminalName, "/exit\n");
            long deadline = System.currentTimeMillis() + 10_000;
            while (System.currentTimeMillis() < deadline) {
                Thread.sleep(500);
                cmd = tmux.displayMessage(terminalName, "#{pane_current_command}").trim();
                if (SHELL_COMMANDS.contains(cmd)) break;
            }
            if (!SHELL_COMMANDS.contains(cmd) && existing.pid() > 0) {
                treeKill(terminalName, existing.pid());
            }
        }

        markPausedByCoordinator(terminalName, existing.command());
    } finally {
        lock.unlock();
    }
}

private void markPausedByCoordinator(String terminalName, String command) throws IOException, InterruptedException {
    tmux.setOption(terminalName, "@trellis_agent_state", "PAUSED_BY_COORDINATOR");
    agents.put(terminalName, AgentProcess.pausedByCoordinator(command));
    startingTimestamps.remove(terminalName);
    lastErrors.remove(terminalName);
    broadcastState(terminalName);
}
```

- [ ] **Step 7: Run tests — verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 8: Commit**

```bash
git -C sidecar add src/main/java/io/hortora/trellis/agent/AgentProcessManager.java \
  src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java
git -C sidecar commit -m "feat(#22): add gracefulShutdown with Escape→/exit→treeKill fallback  Refs #22"
```

---

### Task 3: SlotAgentCoordinator

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/SlotAgentCoordinator.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/SlotAgentCoordinatorTest.java`

**Interfaces:**
- Consumes: `AgentProcessManager.gracefulShutdown(String)` from Task 2,
  `AgentProcessManager.stopAgent(String)`, `AgentProcessManager.resumeAgent(String)`,
  `LifecycleManager.pause(String, Path)`, `LifecycleManager.resume(String, Path)`,
  `LifecycleManager.end(String, Path)`,
  `TerminalRegistry.list()` → `List<TerminalInfo>`,
  `TerminalInfo.slot()` → slot ID match,
  `AgentState.PAUSED_BY_COORDINATOR`
- Produces: `SlotAgentCoordinator.coordinatedPause(String slotId, Path workspaceRoot)`,
  `SlotAgentCoordinator.coordinatedResume(String slotId, Path workspaceRoot)`,
  `SlotAgentCoordinator.coordinatedEnd(String slotId, Path workspaceRoot)` — used by Tasks 4, 5

- [ ] **Step 1: Write failing test — coordinatedPause shuts down agents before git ops**

```java
@Test
void coordinatedPauseShutsDownAgentsBeforeGitOps() throws Exception {
    var t1 = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    var t2 = new TerminalInfo("t2", "/tmp", "slot-1", null, null);
    var t3 = new TerminalInfo("t3", "/tmp", "slot-2", null, null);
    when(registry.list()).thenReturn(List.of(t1, t2, t3));
    when(agentManager.getSnapshot("t1", t1)).thenReturn(
            new AgentSnapshot("t1", t1, runningAgent(), null));
    when(agentManager.getSnapshot("t2", t2)).thenReturn(
            new AgentSnapshot("t2", t2, runningAgent(), null));
    when(agentManager.getSnapshot("t3", t3)).thenReturn(
            new AgentSnapshot("t3", t3, runningAgent(), null));
    when(lifecycleManager.pause("slot-1", WORKSPACE))
            .thenReturn(new OperationResult(true, 0, Map.of(), ""));

    coordinator.coordinatedPause("slot-1", WORKSPACE);

    verify(agentManager).gracefulShutdown("t1");
    verify(agentManager).gracefulShutdown("t2");
    verify(agentManager, never()).gracefulShutdown("t3");
    var inOrder = inOrder(agentManager, lifecycleManager);
    inOrder.verify(agentManager).gracefulShutdown("t1");
    inOrder.verify(lifecycleManager).pause("slot-1", WORKSPACE);
}
```

- [ ] **Step 2: Write failing test — coordinatedResume restarts only PAUSED_BY_COORDINATOR agents**

```java
@Test
void coordinatedResumeRestartsOnlyCoordinatorPausedAgents() throws Exception {
    var t1 = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    var t2 = new TerminalInfo("t2", "/tmp", "slot-1", null, null);
    when(registry.list()).thenReturn(List.of(t1, t2));
    when(agentManager.getSnapshot("t1", t1)).thenReturn(
            new AgentSnapshot("t1", t1, AgentProcess.pausedByCoordinator("claude"), null));
    when(agentManager.getSnapshot("t2", t2)).thenReturn(
            new AgentSnapshot("t2", t2, AgentProcess.paused("claude"), null));
    when(lifecycleManager.resume("slot-1", WORKSPACE))
            .thenReturn(new OperationResult(true, 0, Map.of(), ""));

    coordinator.coordinatedResume("slot-1", WORKSPACE);

    verify(agentManager).resumeAgent("t1");
    verify(agentManager, never()).resumeAgent("t2");
}
```

- [ ] **Step 3: Write failing test — coordinatedResume skips agents on git failure**

```java
@Test
void coordinatedResumeSkipsAgentsOnGitFailure() throws Exception {
    when(lifecycleManager.resume("slot-1", WORKSPACE))
            .thenReturn(new OperationResult(false, 1, Map.of(), "rebase failed"));

    var result = coordinator.coordinatedResume("slot-1", WORKSPACE);

    assertFalse(result.success());
    verify(agentManager, never()).resumeAgent(any());
}
```

- [ ] **Step 4: Write failing test — concurrent operations on same slot rejected**

```java
@Test
void concurrentOperationsOnSameSlotRejected() throws Exception {
    when(registry.list()).thenReturn(List.of());
    when(lifecycleManager.pause("slot-1", WORKSPACE))
            .thenAnswer(inv -> {
                Thread.sleep(100);
                return new OperationResult(true, 0, Map.of(), "");
            });

    var future = Executors.newSingleThreadExecutor().submit(() -> {
        coordinator.coordinatedPause("slot-1", WORKSPACE);
        return null;
    });
    Thread.sleep(20);

    assertThrows(ConcurrentOperationException.class,
            () -> coordinator.coordinatedPause("slot-1", WORKSPACE));
    future.get();
}
```

- [ ] **Step 5: Write failing test — coordinatedEnd stops all agents including PAUSED**

```java
@Test
void coordinatedEndStopsAllAgentsIncludingPaused() throws Exception {
    var t1 = new TerminalInfo("t1", "/tmp", "slot-1", null, null);
    var t2 = new TerminalInfo("t2", "/tmp", "slot-1", null, null);
    when(registry.list()).thenReturn(List.of(t1, t2));
    when(agentManager.getSnapshot("t1", t1)).thenReturn(
            new AgentSnapshot("t1", t1, runningAgent(), null));
    when(agentManager.getSnapshot("t2", t2)).thenReturn(
            new AgentSnapshot("t2", t2, AgentProcess.paused("claude"), null));
    when(lifecycleManager.end("slot-1", WORKSPACE))
            .thenReturn(new OperationResult(true, 0, Map.of(), ""));

    coordinator.coordinatedEnd("slot-1", WORKSPACE);

    verify(agentManager).stopAgent("t1");
    verify(agentManager).stopAgent("t2");
}
```

- [ ] **Step 6: Run tests — verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=SlotAgentCoordinatorTest -DfailIfNoTests=false`
Expected: compilation error — class does not exist

- [ ] **Step 7: Implement SlotAgentCoordinator**

Create `sidecar/src/main/java/io/hortora/trellis/lifecycle/SlotAgentCoordinator.java`
using `ide_create_file`:

```java
package io.hortora.trellis.lifecycle;

import io.hortora.trellis.agent.AgentProcessManager;
import io.hortora.trellis.agent.AgentState;
import io.hortora.trellis.terminal.TerminalRegistry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.nio.file.Path;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

@ApplicationScoped
public class SlotAgentCoordinator {

    private static final Logger LOG = Logger.getLogger(SlotAgentCoordinator.class);

    @Inject LifecycleManager lifecycleManager;
    @Inject AgentProcessManager agentProcessManager;
    @Inject TerminalRegistry terminalRegistry;

    private final ConcurrentHashMap<String, ReentrantLock> slotLocks = new ConcurrentHashMap<>();

    public OperationResult coordinatedPause(String slotId, Path workspaceRoot)
            throws IOException, InterruptedException, ConcurrentOperationException {
        var lock = slotLocks.computeIfAbsent(slotId, k -> new ReentrantLock());
        if (!lock.tryLock()) {
            throw new ConcurrentOperationException("Coordinated operation in progress for slot: " + slotId);
        }
        try {
            shutdownSlotAgents(slotId);
            return lifecycleManager.pause(slotId, workspaceRoot);
        } finally {
            lock.unlock();
        }
    }

    public OperationResult coordinatedResume(String slotId, Path workspaceRoot)
            throws IOException, InterruptedException, ConcurrentOperationException {
        var lock = slotLocks.computeIfAbsent(slotId, k -> new ReentrantLock());
        if (!lock.tryLock()) {
            throw new ConcurrentOperationException("Coordinated operation in progress for slot: " + slotId);
        }
        try {
            var result = lifecycleManager.resume(slotId, workspaceRoot);
            if (!result.success()) return result;
            resumeCoordinatorPausedAgents(slotId);
            return result;
        } finally {
            lock.unlock();
        }
    }

    public OperationResult coordinatedEnd(String slotId, Path workspaceRoot)
            throws IOException, InterruptedException, ConcurrentOperationException {
        var lock = slotLocks.computeIfAbsent(slotId, k -> new ReentrantLock());
        if (!lock.tryLock()) {
            throw new ConcurrentOperationException("Coordinated operation in progress for slot: " + slotId);
        }
        try {
            stopAllSlotAgents(slotId);
            return lifecycleManager.end(slotId, workspaceRoot);
        } finally {
            lock.unlock();
        }
    }

    private void shutdownSlotAgents(String slotId) {
        var terminals = terminalRegistry.list().stream()
                .filter(t -> slotId.equals(t.slot()))
                .toList();
        terminals.parallelStream().forEach(t -> {
            var snapshot = agentProcessManager.getSnapshot(t.name(), t);
            if (snapshot.process() != null && snapshot.process().state() == AgentState.RUNNING) {
                try {
                    agentProcessManager.gracefulShutdown(t.name());
                } catch (Exception e) {
                    LOG.warnf("Failed to gracefully shutdown agent %s: %s", t.name(), e.getMessage());
                }
            }
        });
    }

    private void resumeCoordinatorPausedAgents(String slotId) {
        var terminals = terminalRegistry.list().stream()
                .filter(t -> slotId.equals(t.slot()))
                .toList();
        for (var t : terminals) {
            var snapshot = agentProcessManager.getSnapshot(t.name(), t);
            if (snapshot.process() != null
                    && snapshot.process().state() == AgentState.PAUSED_BY_COORDINATOR) {
                try {
                    agentProcessManager.resumeAgent(t.name());
                } catch (Exception e) {
                    LOG.warnf("Failed to resume agent %s: %s", t.name(), e.getMessage());
                }
            }
        }
    }

    private void stopAllSlotAgents(String slotId) {
        var terminals = terminalRegistry.list().stream()
                .filter(t -> slotId.equals(t.slot()))
                .toList();
        terminals.parallelStream().forEach(t -> {
            var snapshot = agentProcessManager.getSnapshot(t.name(), t);
            if (snapshot.process() != null) {
                try {
                    agentProcessManager.stopAgent(t.name());
                } catch (Exception e) {
                    LOG.warnf("Failed to stop agent %s: %s", t.name(), e.getMessage());
                }
            }
        });
    }
}
```

- [ ] **Step 8: Create test class**

Create `sidecar/src/test/java/io/hortora/trellis/lifecycle/SlotAgentCoordinatorTest.java`
with helper methods `runningAgent()`, mocks for `LifecycleManager`,
`AgentProcessManager`, `TerminalRegistry`, and constant `WORKSPACE = Path.of("/ws")`.
Wire coordinator via field injection in `@BeforeEach`.

- [ ] **Step 9: Run tests — verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=SlotAgentCoordinatorTest -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 10: Update resumeAgent to accept PAUSED_BY_COORDINATOR**

In `AgentProcessManager.resumeAgent`, the state check currently only allows
`PAUSED`. Update to also accept `PAUSED_BY_COORDINATOR`:

```java
if (existing == null || (existing.state() != AgentState.PAUSED
        && existing.state() != AgentState.PAUSED_BY_COORDINATOR)) {
    throw new IllegalStateException("Cannot resume agent in state: " +
            (existing != null ? existing.state() : "IDLE"));
}
```

Also clear the `@trellis_agent_state` tmux option (the existing code already
does this with `tmux.setOption(terminalName, "@trellis_agent_state", "")` —
verify it handles both values).

- [ ] **Step 11: Run all agent tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="AgentProcessManagerTest,SlotAgentCoordinatorTest" -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 12: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/lifecycle/SlotAgentCoordinator.java \
  sidecar/src/main/java/io/hortora/trellis/agent/AgentProcessManager.java \
  sidecar/src/test/java/io/hortora/trellis/lifecycle/SlotAgentCoordinatorTest.java
git commit -m "feat(#22): add SlotAgentCoordinator with slot-level locking  Refs #22"
```

---

### Task 4: Wire REST and LifecycleActionExecutor Through Coordinator

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleResource.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/LifecycleActionExecutor.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/coordinator/LifecycleActionExecutorTest.java`

**Interfaces:**
- Consumes: `SlotAgentCoordinator.coordinatedPause(String, Path)`,
  `SlotAgentCoordinator.coordinatedResume(String, Path)`,
  `SlotAgentCoordinator.coordinatedEnd(String, Path)` from Task 3
- Produces: REST endpoints `/api/lifecycle/pause/{slotId}` and
  `/api/lifecycle/resume/{slotId}` and `/api/lifecycle/end/{slotId}` now
  route through coordinator

- [ ] **Step 1: Write failing test — LifecycleActionExecutor routes pause through coordinator**

Add test to `LifecycleActionExecutorTest`:

```java
@Test
void pauseRoutedThroughCoordinator() {
    var coordinator = mock(SlotAgentCoordinator.class);
    var executor = new LifecycleActionExecutor(new LifecycleManager(), coordinator);
    // Should fail — constructor doesn't accept coordinator yet
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleActionExecutorTest#pauseRoutedThroughCoordinator -DfailIfNoTests=false`
Expected: compilation error — wrong constructor signature

- [ ] **Step 3: Update LifecycleActionExecutor**

Inject `SlotAgentCoordinator`. Route `lifecycle.pause`, `lifecycle.resume`,
`lifecycle.end` through the coordinator. All other action types continue to
use `LifecycleManager` directly.

```java
@Inject
public LifecycleActionExecutor(LifecycleManager manager, SlotAgentCoordinator coordinator) {
    this.manager = manager;
    this.coordinator = coordinator;
}
```

In the `dispatch` method, change three cases:
```java
case "lifecycle.end" -> coordinator.coordinatedEnd(p.get("slotId"), Path.of(p.get("workspaceRoot")));
case "lifecycle.pause" -> coordinator.coordinatedPause(p.get("slotId"), Path.of(p.get("workspaceRoot")));
case "lifecycle.resume" -> coordinator.coordinatedResume(p.get("slotId"), Path.of(p.get("workspaceRoot")));
```

- [ ] **Step 4: Update LifecycleResource**

Inject `SlotAgentCoordinator`. Route `pause()`, `resume()`, and `end()`
through the coordinator instead of calling `LifecycleManager` directly.

- [ ] **Step 5: Fix existing tests**

Update `LifecycleActionExecutorTest` stubs — the `StubLifecycleManager` and
`FailingLifecycleManager` need a coordinator parameter in the
`LifecycleActionExecutor` constructor. Use a mock or null coordinator for
tests that don't exercise the coordinator path.

- [ ] **Step 6: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="LifecycleActionExecutorTest,SlotAgentCoordinatorTest" -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleResource.java \
  sidecar/src/main/java/io/hortora/trellis/coordinator/LifecycleActionExecutor.java \
  sidecar/src/test/java/io/hortora/trellis/coordinator/LifecycleActionExecutorTest.java
git commit -m "feat(#22): route pause/resume/end through SlotAgentCoordinator  Refs #22"
```

---

### Task 5: Memory Pressure Monitoring and Eviction Queue

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/agent/EvictionCandidate.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/agent/EvictionReason.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/agent/MemoryPressureMonitor.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentMonitorScheduler.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentProcessManager.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/agent/EvictionResource.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/agent/MemoryPressureMonitorTest.java`

**Interfaces:**
- Consumes: `AgentProcessManager.getAllSnapshots()`, `EventBroadcaster.broadcast()`
- Produces: `MemoryPressureMonitor.evaluate(List<AgentSnapshot>)` → `List<EvictionCandidate>`,
  `GET /api/agents/eviction` → `List<EvictionCandidate>`,
  SSE topic `agent:eviction`

- [ ] **Step 1: Create domain types**

```java
// EvictionReason.java
package io.hortora.trellis.agent;
public enum EvictionReason { PER_AGENT_CRITICAL, SYSTEM_PRESSURE }

// EvictionCandidate.java
package io.hortora.trellis.agent;
import java.time.Instant;
public record EvictionCandidate(
        String terminalName,
        long memoryBytes,
        Instant firstExceeded,
        EvictionReason reason
) {}
```

- [ ] **Step 2: Write failing test — per-agent critical threshold**

```java
@Test
void agentExceeding1GBBecomesEvictionCandidate() {
    var snapshot = new AgentSnapshot("t1", terminal("t1"),
            new AgentProcess(100, AgentState.RUNNING, 1_200_000_000L, Instant.now(), "claude"), null);
    var candidates = monitor.evaluate(List.of(snapshot));
    assertEquals(1, candidates.size());
    assertEquals(EvictionReason.PER_AGENT_CRITICAL, candidates.get(0).reason());
}
```

- [ ] **Step 3: Write failing test — agent below threshold not a candidate**

```java
@Test
void agentBelow1GBNotCandidate() {
    var snapshot = new AgentSnapshot("t1", terminal("t1"),
            new AgentProcess(100, AgentState.RUNNING, 500_000_000L, Instant.now(), "claude"), null);
    var candidates = monitor.evaluate(List.of(snapshot));
    assertTrue(candidates.isEmpty());
}
```

- [ ] **Step 4: Write failing test — firstExceeded preserved across evaluations**

```java
@Test
void firstExceededPreservedAcrossEvaluations() {
    var snapshot = new AgentSnapshot("t1", terminal("t1"),
            new AgentProcess(100, AgentState.RUNNING, 1_200_000_000L, Instant.now(), "claude"), null);
    var first = monitor.evaluate(List.of(snapshot));
    var firstTime = first.get(0).firstExceeded();

    var second = monitor.evaluate(List.of(snapshot));
    assertEquals(firstTime, second.get(0).firstExceeded());
}
```

- [ ] **Step 5: Implement MemoryPressureMonitor**

```java
package io.hortora.trellis.agent;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class MemoryPressureMonitor {

    private static final long PER_AGENT_CRITICAL_BYTES = 1_073_741_824L; // 1 GB

    private final ConcurrentHashMap<String, Instant> firstExceeded = new ConcurrentHashMap<>();

    public List<EvictionCandidate> evaluate(List<AgentSnapshot> snapshots) {
        var candidates = new ArrayList<EvictionCandidate>();
        var currentTerminals = new HashSet<String>();
        var runningByRss = new ArrayList<AgentSnapshot>();

        for (var s : snapshots) {
            if (s.process() == null || s.process().state() != AgentState.RUNNING) continue;
            currentTerminals.add(s.terminalName());
            runningByRss.add(s);
            if (s.process().memoryBytes() >= PER_AGENT_CRITICAL_BYTES) {
                var exceeded = firstExceeded.computeIfAbsent(s.terminalName(), k -> Instant.now());
                candidates.add(new EvictionCandidate(
                        s.terminalName(), s.process().memoryBytes(), exceeded,
                        EvictionReason.PER_AGENT_CRITICAL));
            } else {
                firstExceeded.remove(s.terminalName());
            }
        }

        if (systemPressureDetected()) {
            long totalRss = runningByRss.stream().mapToLong(s -> s.process().memoryBytes()).sum();
            runningByRss.sort(Comparator.comparingLong((AgentSnapshot s) -> s.process().memoryBytes()).reversed());
            long target = totalRss / 2;
            long accumulated = 0;
            for (var s : runningByRss) {
                if (accumulated >= target) break;
                if (candidates.stream().noneMatch(c -> c.terminalName().equals(s.terminalName()))) {
                    var exceeded = firstExceeded.computeIfAbsent(s.terminalName(), k -> Instant.now());
                    candidates.add(new EvictionCandidate(
                            s.terminalName(), s.process().memoryBytes(), exceeded,
                            EvictionReason.SYSTEM_PRESSURE));
                }
                accumulated += s.process().memoryBytes();
            }
        }

        firstExceeded.keySet().removeIf(k -> !currentTerminals.contains(k));
        return candidates;
    }

    boolean systemPressureDetected() {
        try {
            var proc = new ProcessBuilder("memory_pressure").start();
            var output = new String(proc.getInputStream().readAllBytes());
            proc.waitFor();
            return output.contains("WARNING") || output.contains("CRITICAL");
        } catch (Exception e) {
            return false;
        }
    }
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=MemoryPressureMonitorTest -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 7: Wire into AgentMonitorScheduler**

Inject `MemoryPressureMonitor` into `AgentMonitorScheduler`. After the poll
cycle collects all snapshots, call `monitor.evaluate(snapshots)` and broadcast
the result as a batched SSE event on topic `agent:eviction`.

- [ ] **Step 8: Create EvictionResource**

```java
package io.hortora.trellis.agent;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@Path("/api/agents/eviction")
@Produces(MediaType.APPLICATION_JSON)
public class EvictionResource {

    @Inject MemoryPressureMonitor monitor;
    @Inject TerminalRegistry registry;
    @Inject AgentProcessManager processManager;

    @GET
    public Response list() {
        var snapshots = processManager.getAllSnapshots(registry.list());
        return Response.ok(monitor.evaluate(snapshots)).build();
    }
}
```

(Import `TerminalRegistry` from `io.hortora.trellis.terminal`.)

- [ ] **Step 9: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 10: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/agent/EvictionCandidate.java \
  sidecar/src/main/java/io/hortora/trellis/agent/EvictionReason.java \
  sidecar/src/main/java/io/hortora/trellis/agent/MemoryPressureMonitor.java \
  sidecar/src/main/java/io/hortora/trellis/agent/EvictionResource.java \
  sidecar/src/main/java/io/hortora/trellis/agent/AgentMonitorScheduler.java \
  sidecar/src/test/java/io/hortora/trellis/agent/MemoryPressureMonitorTest.java
git commit -m "feat(#22): add memory pressure monitoring and eviction queue  Refs #22"
```

---

### Task 6: Frontend — Eviction Badges and System Pressure Banner

**Files:**
- Modify: `sidecar/src/main/webui/src/components/agent-status-badge.ts`
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts`

**Interfaces:**
- Consumes: SSE topic `agent:eviction` → `List<EvictionCandidate>`,
  `POST /api/terminals/{name}/agent/pause` (Evict button),
  `AgentState` values including `PAUSED_BY_COORDINATOR`

- [ ] **Step 1: Update agent-status-badge to handle eviction and PAUSED_BY_COORDINATOR**

The badge component needs three changes:
1. Accept a new `evictionCandidate` boolean property
2. When `evictionCandidate` is true and state is RUNNING: show pulsing red badge
3. Treat `PAUSED_BY_COORDINATOR` the same as `PAUSED` for display (amber badge)
4. Change the 500 MB threshold from red (`warning`) to amber

```typescript
@property({ type: Boolean }) evictionCandidate = false;
```

Update the memory span class logic:
```typescript
const memoryClass = this.evictionCandidate ? 'critical'
    : this.memoryMb > 500 ? 'amber' : '';
```

Add CSS:
```css
.memory.amber { color: #fbbf24; font-weight: 600; }
.memory.critical { color: #f87171; font-weight: 600; animation: pulse 1.5s ease-in-out infinite; }
```

Handle `PAUSED_BY_COORDINATOR` in state display — normalize to `paused`:
```typescript
const normalizedState = this.state === 'PAUSED_BY_COORDINATOR' ? 'PAUSED' : this.state;
```

- [ ] **Step 2: Update slot-detail to subscribe to agent:eviction and show Evict button**

Add `_evictionCandidates` state to track current candidates. Subscribe to
`agent:eviction` SSE topic alongside existing subscriptions.

When rendering terminal entries, check if the terminal is an eviction
candidate. If so, pass `evictionCandidate=true` to the badge and show an
"Evict" button that calls `POST /api/terminals/{name}/agent/pause`.

Add a system pressure banner at the top of the terminal panel when any
`SYSTEM_PRESSURE` candidates exist — show total agent RSS.

- [ ] **Step 3: Build frontend**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: build succeeds with no errors

- [ ] **Step 4: Manual browser verification**

Start dev mode: `/opt/homebrew/bin/mvn -f sidecar/pom.xml quarkus:dev`
Verify:
- Memory badges show amber at 500 MB (not red)
- Eviction candidates (if any) show pulsing red
- PAUSED_BY_COORDINATOR displays as "paused" (amber badge)
- Evict button appears for eviction candidates
- System pressure banner appears when applicable

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/src/components/agent-status-badge.ts \
  sidecar/src/main/webui/src/views/slot-detail.ts
git commit -m "feat(#22): add eviction badges, system pressure banner, and Evict button  Refs #22"
```

---

### Task 7: Full Integration Test

**Files:**
- Modify: `sidecar/src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java`
- Run full test suite

- [ ] **Step 1: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -DfailIfNoTests=false`
Expected: all tests pass

- [ ] **Step 2: Build the full sidecar jar**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml package -DskipTests`
Expected: build succeeds

- [ ] **Step 3: Final commit if any cleanup needed**

```bash
git commit -m "chore(#22): integration cleanup  Refs #22"
```
