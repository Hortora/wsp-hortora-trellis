# Process Isolation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #20 — Process isolation, memory monitoring, and session lifecycle
**Issue group:** #20

**Goal:** Add agent process lifecycle management to trellis — track Claude CLI
processes in tmux terminals, monitor memory, provide start/stop/pause/resume/refresh
controls via REST API and frontend.

**Architecture:** Hybrid process management. A single `AgentProcessManager` owns
both scheduled discovery (5s poll using tmux `pane_current_command` introspection)
and lifecycle operations (start/stop/pause/resume/refresh via send-keys and
tree-kill). Agent endpoints are sub-resources of terminals
(`/api/terminals/{name}/agent/*`). State changes push to the existing
`/api/push` SSE infrastructure via `EventBroadcaster`.

**Tech Stack:** Java 21, Quarkus 3.x, Lit (TypeScript), xterm.js, tmux,
casehub-pages-push (TopicRegistry, EventBroadcaster)

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- New types in `io.hortora.trellis.agent`
- IntelliJ MCP required for all source file operations
- `quarkus.http.host=localhost` (sidecar is local-only, no auth)
- RSS overcounts by ~30-50% for shared pages — 500 MB threshold is calibrated to RSS values
- `claude -c` always used for resume/refresh (intentional — restores full session state)
- Pre-release: breaking API changes cost nothing

---

### Task 1: Rename Session → Terminal

Codebase-wide rename using IntelliJ refactoring. No new functionality.

**Files:**
- Rename: `SessionInfo` → `TerminalInfo` (use `ide_refactor_rename`)
- Rename: `SessionRegistry` → `TerminalRegistry` (use `ide_refactor_rename`)
- Rename: `SessionRegistryTest` → `TerminalRegistryTest` (use `ide_refactor_rename`)
- Modify: `src/main/java/io/hortora/trellis/terminal/TerminalResource.java` — change `@Path` annotation
- Modify: `src/main/webui/src/views/slot-detail.ts` — rename interface and API path
- Test: `src/test/java/io/hortora/trellis/terminal/TerminalRegistryTest.java` (renamed)
- Test: `src/test/java/io/hortora/trellis/terminal/TmuxManagerTest.java` (unchanged)

**Interfaces:**
- Produces: `TerminalInfo(String name, String workingDir, String slot, String repo, String issue)` — same fields as old `SessionInfo`
- Produces: `TerminalRegistry` — same API surface as old `SessionRegistry`

- [ ] **Step 1: Rename SessionInfo → TerminalInfo**

Use `ide_refactor_rename` on `SessionInfo` at line 3 of
`src/main/java/io/hortora/trellis/terminal/SessionInfo.java`.
This renames the record, the file, and all references across the project.

- [ ] **Step 2: Rename SessionRegistry → TerminalRegistry**

Use `ide_refactor_rename` on `SessionRegistry` at line 13 of
`src/main/java/io/hortora/trellis/terminal/SessionRegistry.java`.
This renames the class, the file, and all references (TerminalResource,
TerminalWebSocket, test classes).

- [ ] **Step 3: Rename SessionRegistryTest → TerminalRegistryTest**

Use `ide_refactor_rename` on `SessionRegistryTest` at line 11 of
`src/test/java/io/hortora/trellis/terminal/SessionRegistryTest.java`.

- [ ] **Step 4: Update REST path annotation**

Use `ide_replace_text_in_file` on `TerminalResource.java`:
```
searchText: @Path("/api/sessions")
replaceText: @Path("/api/terminals")
```

- [ ] **Step 5: Update frontend interface and API path**

In `src/main/webui/src/views/slot-detail.ts`:

Rename the `SessionInfo` interface to `TerminalInfo` (line 14):
```typescript
interface TerminalInfo {
  name: string;
  workingDir: string | null;
  slot: string | null;
  repo: string | null;
  issue: string | null;
}
```

Update the `_sessions` state field (line 29):
```typescript
@state() private _terminals: TerminalInfo[] = [];
```

Update the API path (line 181):
```typescript
const res = await fetch('/api/terminals');
```

Update all references from `_sessions` to `_terminals` and `SessionInfo`
to `TerminalInfo` throughout the file.

- [ ] **Step 6: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: All tests pass with renamed types.

- [ ] **Step 7: Verify with IDE diagnostics**

Run: `ide_diagnostics` on `TerminalResource.java`, `TerminalRegistry.java`,
`TerminalWebSocket.java` to confirm no compilation errors.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "refactor(#20): rename Session → Terminal throughout codebase"
```

---

### Task 2: Domain Model Types

Create the new types in `io.hortora.trellis.agent`.

**Files:**
- Create: `src/main/java/io/hortora/trellis/agent/AgentState.java`
- Create: `src/main/java/io/hortora/trellis/agent/AgentProcess.java`
- Create: `src/main/java/io/hortora/trellis/agent/StartAgentRequest.java`
- Create: `src/main/java/io/hortora/trellis/agent/AgentSnapshot.java`
- Test: `src/test/java/io/hortora/trellis/agent/StartAgentRequestTest.java`

**Interfaces:**
- Produces: `AgentState { IDLE, STARTING, RUNNING, PAUSED }`
- Produces: `AgentProcess(long pid, AgentState state, long memoryBytes, Instant startedAt, String command)`
- Produces: `StartAgentRequest(boolean resume, String prompt)`
- Produces: `AgentSnapshot(String terminalName, TerminalInfo terminal, AgentProcess process, String lastError)`

- [ ] **Step 1: Write the StartAgentRequest validation test**

Create test file:
```java
package io.hortora.trellis.agent;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class StartAgentRequestTest {

    @Test
    void freshSessionNoPrompt() {
        var req = new StartAgentRequest(false, null);
        assertFalse(req.resume());
        assertNull(req.prompt());
    }

    @Test
    void freshSessionWithPrompt() {
        var req = new StartAgentRequest(false, "Fix the login bug");
        assertEquals("Fix the login bug", req.prompt());
    }

    @Test
    void resumeSession() {
        var req = new StartAgentRequest(true, null);
        assertTrue(req.resume());
    }

    @Test
    void resumeWithPromptThrows() {
        assertThrows(IllegalArgumentException.class,
                () -> new StartAgentRequest(true, "some prompt"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=StartAgentRequestTest`
Expected: FAIL — classes don't exist.

- [ ] **Step 3: Create AgentState enum**

Use `ide_create_file`:
```java
package io.hortora.trellis.agent;

public enum AgentState {
    IDLE,
    STARTING,
    RUNNING,
    PAUSED
}
```

- [ ] **Step 4: Create AgentProcess record**

Use `ide_create_file`:
```java
package io.hortora.trellis.agent;

import java.time.Instant;

public record AgentProcess(
        long pid,
        AgentState state,
        long memoryBytes,
        Instant startedAt,
        String command
) {
    public static AgentProcess paused(String command) {
        return new AgentProcess(0, AgentState.PAUSED, 0, null, command);
    }
}
```

- [ ] **Step 5: Create StartAgentRequest record**

Use `ide_create_file`:
```java
package io.hortora.trellis.agent;

public record StartAgentRequest(boolean resume, String prompt) {
    public StartAgentRequest {
        if (resume && prompt != null) {
            throw new IllegalArgumentException("resume and prompt are mutually exclusive");
        }
    }
}
```

- [ ] **Step 6: Create AgentSnapshot record**

Use `ide_create_file`:
```java
package io.hortora.trellis.agent;

import io.hortora.trellis.terminal.TerminalInfo;

public record AgentSnapshot(
        String terminalName,
        TerminalInfo terminal,
        AgentProcess process,
        String lastError
) {}
```

- [ ] **Step 7: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=StartAgentRequestTest`
Expected: All 4 tests pass.

- [ ] **Step 8: Commit**

```bash
git add src/main/java/io/hortora/trellis/agent/ src/test/java/io/hortora/trellis/agent/
git commit -m "feat(#20): add agent domain model — AgentState, AgentProcess, StartAgentRequest, AgentSnapshot"
```

---

### Task 3: TmuxManager Enhancements + Process Tree Walker

Add `displayMessage` to TmuxManager for process discovery, and create a
utility for parsing `ps` output into a process tree.

**Files:**
- Modify: `src/main/java/io/hortora/trellis/terminal/TmuxManager.java` — add `displayMessage`
- Create: `src/main/java/io/hortora/trellis/agent/ProcessTreeWalker.java`
- Test: `src/test/java/io/hortora/trellis/terminal/TmuxManagerTest.java` — add displayMessage test
- Test: `src/test/java/io/hortora/trellis/agent/ProcessTreeWalkerTest.java`

**Interfaces:**
- Produces: `TmuxManager.displayMessage(String sessionName, String format): String`
- Produces: `ProcessTreeWalker.walk(long rootPid): ProcessTree`
- Produces: `ProcessTree(long claudePid, long totalRssBytes, List<Long> allPids)`

- [ ] **Step 1: Write TmuxManager.displayMessage test**

Add to `TmuxManagerTest.java`:
```java
@Test
void displayMessageReturnsPanePid() throws IOException, InterruptedException {
    manager.createSession(sessionName, "/tmp");
    String panePid = manager.displayMessage(sessionName, "#{pane_pid}");
    assertFalse(panePid.isBlank());
    assertTrue(panePid.matches("\\d+"), "Expected numeric PID, got: " + panePid);
}

@Test
void displayMessageReturnsPaneCurrentCommand() throws IOException, InterruptedException {
    manager.createSession(sessionName, "/tmp");
    String cmd = manager.displayMessage(sessionName, "#{pane_current_command}");
    assertFalse(cmd.isBlank());
    // Default shell in tmux pane
    assertTrue(cmd.matches("(bash|zsh|sh|dash|fish)"),
            "Expected shell command, got: " + cmd);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=TmuxManagerTest#displayMessage*`
Expected: FAIL — method doesn't exist.

- [ ] **Step 3: Implement displayMessage**

Use `ide_insert_member` on `TmuxManager.java`, after `capturePane`:
```java
public String displayMessage(String sessionName, String format)
        throws IOException, InterruptedException {
    var p = new ProcessBuilder("tmux", "display-message", "-t", sessionName, "-p", format)
            .redirectErrorStream(false).start();
    var output = new String(p.getInputStream().readAllBytes()).trim();
    p.waitFor();
    return output;
}
```

- [ ] **Step 4: Run TmuxManager tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=TmuxManagerTest`
Expected: All tests pass (including new displayMessage tests).

- [ ] **Step 5: Write ProcessTreeWalker test**

Create test file:
```java
package io.hortora.trellis.agent;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ProcessTreeWalkerTest {

    @Test
    void findsClaudeInSimpleTree() {
        // pane shell (pid 100) → claude (pid 101)
        String psOutput = """
              100     1  1024 /bin/zsh
              101   100 204800 /usr/local/bin/node /Users/user/.claude/local/claude
              """;
        var tree = ProcessTreeWalker.fromPsOutput(psOutput, 100);
        assertTrue(tree.isPresent());
        assertEquals(101, tree.get().claudePid());
        assertEquals(204800L * 1024, tree.get().totalRssBytes());
    }

    @Test
    void sumsChildProcessRss() {
        // claude (101) has MCP server child (102) and subagent (103)
        String psOutput = """
              100     1  1024 /bin/zsh
              101   100 204800 /usr/local/bin/node /Users/user/.claude/local/claude
              102   101 51200 /usr/local/bin/node mcp-server
              103   101 25600 /usr/local/bin/node subagent
              """;
        var tree = ProcessTreeWalker.fromPsOutput(psOutput, 100);
        assertTrue(tree.isPresent());
        assertEquals(101, tree.get().claudePid());
        assertEquals((204800L + 51200 + 25600) * 1024, tree.get().totalRssBytes());
        assertEquals(3, tree.get().allPids().size());
    }

    @Test
    void returnsEmptyWhenNoClaudeFound() {
        // Only a shell running
        String psOutput = """
              100     1  1024 /bin/zsh
              """;
        var tree = ProcessTreeWalker.fromPsOutput(psOutput, 100);
        assertTrue(tree.isEmpty());
    }

    @Test
    void handlesDeepTree() {
        // claude → child → grandchild
        String psOutput = """
              100     1  1024 /bin/zsh
              101   100 102400 /usr/local/bin/node /Users/user/.claude/local/claude
              102   101 20480 node child
              103   102 10240 node grandchild
              """;
        var tree = ProcessTreeWalker.fromPsOutput(psOutput, 100);
        assertTrue(tree.isPresent());
        assertEquals((102400L + 20480 + 10240) * 1024, tree.get().totalRssBytes());
        assertEquals(3, tree.get().allPids().size());
    }

    @Test
    void ignoresNonClaudeNodeProcess() {
        // node process that isn't claude
        String psOutput = """
              100     1  1024 /bin/zsh
              101   100 102400 /usr/local/bin/node /some/other/app.js
              """;
        var tree = ProcessTreeWalker.fromPsOutput(psOutput, 100);
        assertTrue(tree.isEmpty());
    }
}
```

- [ ] **Step 6: Run ProcessTreeWalker tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ProcessTreeWalkerTest`
Expected: FAIL — class doesn't exist.

- [ ] **Step 7: Implement ProcessTreeWalker**

Use `ide_create_file`:
```java
package io.hortora.trellis.agent;

import java.io.IOException;
import java.util.*;
import java.util.stream.Collectors;

public class ProcessTreeWalker {

    public record ProcessTree(long claudePid, long totalRssBytes, List<Long> allPids) {}

    public static Optional<ProcessTree> fromPsOutput(String psOutput, long rootPid) {
        var children = new HashMap<Long, List<long[]>>();
        var entries = new HashMap<Long, String[]>();

        for (String line : psOutput.lines().toList()) {
            var trimmed = line.trim();
            if (trimmed.isEmpty()) continue;
            var parts = trimmed.split("\\s+", 4);
            if (parts.length < 4) continue;
            try {
                long pid = Long.parseLong(parts[0]);
                long ppid = Long.parseLong(parts[1]);
                long rss = Long.parseLong(parts[2]);
                String args = parts[3];
                children.computeIfAbsent(ppid, k -> new ArrayList<>()).add(new long[]{pid, rss});
                entries.put(pid, new String[]{args, String.valueOf(rss)});
            } catch (NumberFormatException ignored) {}
        }

        Long claudePid = findClaude(rootPid, children, entries);
        if (claudePid == null) return Optional.empty();

        var allPids = new ArrayList<Long>();
        long totalRss = collectTree(claudePid, children, entries, allPids);

        return Optional.of(new ProcessTree(claudePid, totalRss * 1024, List.copyOf(allPids)));
    }

    public static Optional<ProcessTree> walk(long rootPid) throws IOException, InterruptedException {
        var p = new ProcessBuilder("ps", "-eo", "pid=,ppid=,rss=,args=")
                .redirectErrorStream(false).start();
        var output = new String(p.getInputStream().readAllBytes());
        p.waitFor();
        return fromPsOutput(output, rootPid);
    }

    private static Long findClaude(long pid, Map<Long, List<long[]>> children,
                                    Map<Long, String[]> entries) {
        var entry = entries.get(pid);
        if (entry != null && entry[0].contains("claude")) return pid;
        var kids = children.get(pid);
        if (kids == null) return null;
        for (long[] kid : kids) {
            var found = findClaude(kid[0], children, entries);
            if (found != null) return found;
        }
        return null;
    }

    private static long collectTree(long pid, Map<Long, List<long[]>> children,
                                     Map<Long, String[]> entries, List<Long> allPids) {
        allPids.add(pid);
        long rss = Long.parseLong(entries.get(pid)[1]);
        var kids = children.get(pid);
        if (kids != null) {
            for (long[] kid : kids) {
                rss += collectTree(kid[0], children, entries, allPids);
            }
        }
        return rss;
    }
}
```

- [ ] **Step 8: Run all tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="ProcessTreeWalkerTest,TmuxManagerTest"`
Expected: All tests pass.

- [ ] **Step 9: Commit**

```bash
git add src/main/java/io/hortora/trellis/terminal/TmuxManager.java src/main/java/io/hortora/trellis/agent/ProcessTreeWalker.java src/test/java/io/hortora/trellis/agent/ProcessTreeWalkerTest.java src/test/java/io/hortora/trellis/terminal/TmuxManagerTest.java
git commit -m "feat(#20): add TmuxManager.displayMessage and ProcessTreeWalker for agent discovery"
```

---

### Task 4: AgentProcessManager — Monitoring + State

Core monitoring service: scheduled poll, state storage, bootstrap
initialization, PAUSED persistence.

**Files:**
- Create: `src/main/java/io/hortora/trellis/agent/AgentProcessManager.java`
- Modify: `src/main/java/io/hortora/trellis/terminal/TmuxManager.java` — add `setOption`/`getOption` already exist
- Test: `src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java`

**Interfaces:**
- Consumes: `TmuxManager.displayMessage(name, format)`, `TmuxManager.getOption(name, key)`, `TmuxManager.setOption(name, key, value)`
- Consumes: `ProcessTreeWalker.walk(rootPid)`, `ProcessTreeWalker.fromPsOutput(output, rootPid)`
- Consumes: `TerminalRegistry.list()`, `TerminalRegistry.get(name)`
- Produces: `AgentProcessManager.getSnapshot(terminalName): AgentSnapshot`
- Produces: `AgentProcessManager.getAllSnapshots(registry): List<AgentSnapshot>`
- Produces: `AgentProcessManager.initializeFromBootstrap(terminals: List<TerminalInfo>)`

- [ ] **Step 1: Write monitoring unit test**

Create test file with mock TmuxManager:
```java
package io.hortora.trellis.agent;

import io.casehub.pages.push.EventBroadcaster;
import io.hortora.trellis.terminal.TerminalInfo;
import io.hortora.trellis.terminal.TmuxManager;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.List;
import java.util.Optional;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class AgentProcessManagerTest {

    TmuxManager tmux;
    EventBroadcaster broadcaster;
    AgentProcessManager manager;

    @BeforeEach
    void setUp() {
        tmux = mock(TmuxManager.class);
        broadcaster = mock(EventBroadcaster.class);
        manager = new AgentProcessManager(tmux, broadcaster);
    }

    @Test
    void idleWhenShellIsForegrounded() throws Exception {
        when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("zsh");
        when(tmux.displayMessage("t1", "#{pane_pid}")).thenReturn("100");

        var terminal = new TerminalInfo("t1", "/tmp", null, null, null);
        manager.pollTerminal(terminal);

        var snapshot = manager.getSnapshot("t1", terminal);
        assertNull(snapshot.process());
    }

    @Test
    void runningWhenClaudeDetected() throws Exception {
        when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("node");
        when(tmux.displayMessage("t1", "#{pane_pid}")).thenReturn("100");

        var terminal = new TerminalInfo("t1", "/tmp", null, null, null);
        String fakePsOutput = """
              100     1  1024 /bin/zsh
              101   100 204800 /usr/local/bin/node /Users/user/.claude/local/claude
              """;
        manager.pollTerminalWithPsOutput(terminal, fakePsOutput);

        var snapshot = manager.getSnapshot("t1", terminal);
        assertNotNull(snapshot.process());
        assertEquals(AgentState.RUNNING, snapshot.process().state());
        assertEquals(101, snapshot.process().pid());
    }

    @Test
    void pausedPreservedAcrossMonitorCycles() throws Exception {
        var terminal = new TerminalInfo("t1", "/tmp", null, null, null);
        manager.setPaused("t1", "claude");

        when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("zsh");
        when(tmux.displayMessage("t1", "#{pane_pid}")).thenReturn("100");

        manager.pollTerminal(terminal);

        var snapshot = manager.getSnapshot("t1", terminal);
        assertNotNull(snapshot.process());
        assertEquals(AgentState.PAUSED, snapshot.process().state());
    }

    @Test
    void bootstrapRestoresPausedState() throws Exception {
        when(tmux.getOption("t1", "@trellis_agent_state")).thenReturn(Optional.of("PAUSED"));

        var terminals = List.of(new TerminalInfo("t1", "/tmp", null, null, null));
        manager.initializeFromBootstrap(terminals);

        var snapshot = manager.getSnapshot("t1", terminals.get(0));
        assertNotNull(snapshot.process());
        assertEquals(AgentState.PAUSED, snapshot.process().state());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest`
Expected: FAIL — class doesn't exist.

- [ ] **Step 3: Implement AgentProcessManager (monitoring + state)**

Use `ide_create_file`:
```java
package io.hortora.trellis.agent;

import io.casehub.pages.push.EventBroadcaster;
import io.hortora.trellis.terminal.TerminalInfo;
import io.hortora.trellis.terminal.TerminalRegistry;
import io.hortora.trellis.terminal.TmuxManager;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

@ApplicationScoped
public class AgentProcessManager {

    private static final Logger LOG = Logger.getLogger(AgentProcessManager.class);
    private static final Set<String> SHELL_COMMANDS = Set.of("bash", "zsh", "sh", "dash", "fish");
    private static final long STARTING_TIMEOUT_MS = 15_000;

    private final TmuxManager tmux;
    private final EventBroadcaster broadcaster;
    private final ConcurrentHashMap<String, AgentProcess> agents = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> lastErrors = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Long> startingTimestamps = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ReentrantLock> locks = new ConcurrentHashMap<>();

    @Inject
    public AgentProcessManager(TmuxManager tmux, EventBroadcaster broadcaster) {
        this.tmux = tmux;
        this.broadcaster = broadcaster;
    }

    public void initializeFromBootstrap(List<TerminalInfo> terminals) {
        for (var terminal : terminals) {
            try {
                var paused = tmux.getOption(terminal.name(), "@trellis_agent_state");
                if (paused.isPresent() && "PAUSED".equals(paused.get())) {
                    agents.put(terminal.name(), AgentProcess.paused(null));
                }
            } catch (IOException | InterruptedException e) {
                LOG.debugf("Could not read agent state for %s: %s", terminal.name(), e.getMessage());
            }
        }
    }

    public void pollTerminal(TerminalInfo terminal) {
        var lock = locks.computeIfAbsent(terminal.name(), k -> new ReentrantLock());
        if (!lock.tryLock()) return;
        try {
            doPoll(terminal);
        } finally {
            lock.unlock();
        }
    }

    void pollTerminalWithPsOutput(TerminalInfo terminal, String psOutput) {
        try {
            var currentCommand = tmux.displayMessage(terminal.name(), "#{pane_current_command}").trim();
            var panePidStr = tmux.displayMessage(terminal.name(), "#{pane_pid}").trim();
            long panePid = Long.parseLong(panePidStr);
            processDiscovery(terminal.name(), currentCommand, panePid, psOutput);
        } catch (Exception e) {
            LOG.debugf("Poll failed for %s: %s", terminal.name(), e.getMessage());
        }
    }

    private void doPoll(TerminalInfo terminal) {
        try {
            var currentCommand = tmux.displayMessage(terminal.name(), "#{pane_current_command}").trim();
            var panePidStr = tmux.displayMessage(terminal.name(), "#{pane_pid}").trim();
            long panePid = Long.parseLong(panePidStr);
            processDiscovery(terminal.name(), currentCommand, panePid, null);
        } catch (Exception e) {
            LOG.debugf("Poll failed for %s: %s", terminal.name(), e.getMessage());
        }
    }

    private void processDiscovery(String name, String currentCommand, long panePid, String psOutput) {
        var existing = agents.get(name);
        var existingState = existing != null ? existing.state() : null;

        if (existingState == AgentState.PAUSED) return;

        if (existingState == AgentState.STARTING) {
            var startTime = startingTimestamps.get(name);
            if (startTime != null && System.currentTimeMillis() - startTime < STARTING_TIMEOUT_MS) {
                if (SHELL_COMMANDS.contains(currentCommand)) return;
            } else if (startTime != null) {
                agents.remove(name);
                startingTimestamps.remove(name);
                lastErrors.put(name, "Start timeout: no process appeared within 15s");
                broadcastState(name);
                return;
            }
        }

        if (SHELL_COMMANDS.contains(currentCommand)) {
            if (existingState == AgentState.RUNNING) {
                agents.remove(name);
                lastErrors.put(name, "Agent process ended — check terminal for details");
                broadcastState(name);
            } else if (existingState == AgentState.STARTING) {
                // timeout case handled above
            }
            return;
        }

        // Non-shell command — check for Claude
        Optional<ProcessTreeWalker.ProcessTree> tree;
        try {
            tree = psOutput != null
                    ? ProcessTreeWalker.fromPsOutput(psOutput, panePid)
                    : ProcessTreeWalker.walk(panePid);
        } catch (Exception e) {
            LOG.debugf("Tree walk failed for %s: %s", name, e.getMessage());
            return;
        }

        if (tree.isEmpty()) return;

        var pt = tree.get();
        var process = new AgentProcess(pt.claudePid(), AgentState.RUNNING,
                pt.totalRssBytes(), existing != null ? existing.startedAt() : Instant.now(),
                existing != null ? existing.command() : "claude");
        agents.put(name, process);
        startingTimestamps.remove(name);
        lastErrors.remove(name);
        broadcastState(name);
    }

    public AgentSnapshot getSnapshot(String terminalName, TerminalInfo terminal) {
        return new AgentSnapshot(terminalName, terminal, agents.get(terminalName),
                lastErrors.get(terminalName));
    }

    public List<AgentSnapshot> getAllSnapshots(List<TerminalInfo> terminals) {
        return terminals.stream()
                .map(t -> getSnapshot(t.name(), t))
                .toList();
    }

    public void setStarting(String name, String command) {
        agents.put(name, new AgentProcess(0, AgentState.STARTING, 0, Instant.now(), command));
        startingTimestamps.put(name, System.currentTimeMillis());
        lastErrors.remove(name);
        broadcastState(name);
    }

    public void setPaused(String name, String command) {
        agents.put(name, AgentProcess.paused(command));
        broadcastState(name);
    }

    public void clearState(String name) {
        agents.remove(name);
        lastErrors.remove(name);
        startingTimestamps.remove(name);
    }

    public ReentrantLock lockFor(String terminalName) {
        return locks.computeIfAbsent(terminalName, k -> new ReentrantLock());
    }

    private void broadcastState(String name) {
        try {
            var process = agents.get(name);
            broadcaster.broadcast("agent:state",
                    Map.of("terminalName", name,
                           "state", process != null ? process.state().name() : "IDLE",
                           "pid", process != null ? process.pid() : 0,
                           "memoryBytes", process != null ? process.memoryBytes() : 0));
        } catch (Exception e) {
            LOG.debugf("Failed to broadcast agent state for %s: %s", name, e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest`
Expected: All 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/hortora/trellis/agent/AgentProcessManager.java src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java
git commit -m "feat(#20): add AgentProcessManager — monitoring, state storage, bootstrap"
```

---

### Task 5: AgentProcessManager — Lifecycle Operations

Add start/stop/pause/resume/refresh with kill semantics, shell escaping,
concurrency control, and terminal state verification.

**Files:**
- Modify: `src/main/java/io/hortora/trellis/agent/AgentProcessManager.java`
- Test: `src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java` — add lifecycle tests

**Interfaces:**
- Consumes: `TmuxManager.sendKeys(name, text)`, `TmuxManager.displayMessage(name, format)`
- Consumes: `TmuxManager.setOption(name, key, value)`, `TmuxManager.getOption(name, key)`
- Consumes: `ProcessTreeWalker.walk(rootPid)`
- Produces: `AgentProcessManager.startAgent(terminalName, request): void`
- Produces: `AgentProcessManager.stopAgent(terminalName): void`
- Produces: `AgentProcessManager.pauseAgent(terminalName): void`
- Produces: `AgentProcessManager.resumeAgent(terminalName): void`
- Produces: `AgentProcessManager.refreshAgent(terminalName): void`

- [ ] **Step 1: Write lifecycle unit tests**

Add to `AgentProcessManagerTest.java`:
```java
@Test
void startAgentSendsClaudeCommand() throws Exception {
    when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("zsh");

    manager.startAgent("t1", new StartAgentRequest(false, null));

    verify(tmux).sendKeys(eq("t1"), eq("claude\n"));
    var snapshot = manager.getSnapshot("t1", new TerminalInfo("t1", "/tmp", null, null, null));
    assertEquals(AgentState.STARTING, snapshot.process().state());
}

@Test
void startAgentWithPromptEscapesSingleQuotes() throws Exception {
    when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("zsh");

    manager.startAgent("t1", new StartAgentRequest(false, "Fix the 'auth' flow"));

    verify(tmux).sendKeys(eq("t1"), eq("claude -p 'Fix the '\\''auth'\\'' flow'\n"));
}

@Test
void startAgentResumeSendsClaudeC() throws Exception {
    when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("zsh");

    manager.startAgent("t1", new StartAgentRequest(true, null));

    verify(tmux).sendKeys(eq("t1"), eq("claude -c\n"));
}

@Test
void startAgentRejectsNonShellForeground() throws Exception {
    when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("node");

    assertThrows(IllegalStateException.class,
            () -> manager.startAgent("t1", new StartAgentRequest(false, null)));
}

@Test
void stopAgentFromRunning() throws Exception {
    // Simulate running state
    var terminal = new TerminalInfo("t1", "/tmp", null, null, null);
    agents(manager).put("t1", new AgentProcess(101, AgentState.RUNNING, 204800, Instant.now(), "claude"));

    manager.stopAgent("t1");

    assertNull(agents(manager).get("t1"));
}

@Test
void pauseAgentPersistsTmuxOption() throws Exception {
    agents(manager).put("t1", new AgentProcess(101, AgentState.RUNNING, 204800, Instant.now(), "claude"));

    manager.pauseAgent("t1");

    verify(tmux).setOption("t1", "@trellis_agent_state", "PAUSED");
    assertEquals(AgentState.PAUSED, agents(manager).get("t1").state());
    assertEquals(0, agents(manager).get("t1").pid());
}

@Test
void resumeAgentClearsTmuxOption() throws Exception {
    agents(manager).put("t1", AgentProcess.paused("claude"));
    when(tmux.displayMessage("t1", "#{pane_current_command}")).thenReturn("zsh");

    manager.resumeAgent("t1");

    verify(tmux).setOption("t1", "@trellis_agent_state", "");
    verify(tmux).sendKeys(eq("t1"), eq("claude -c\n"));
    assertEquals(AgentState.STARTING, agents(manager).get("t1").state());
}

// Helper to access internal state for testing
@SuppressWarnings("unchecked")
private static java.util.concurrent.ConcurrentHashMap<String, AgentProcess> agents(
        AgentProcessManager manager) {
    try {
        var field = AgentProcessManager.class.getDeclaredField("agents");
        field.setAccessible(true);
        return (java.util.concurrent.ConcurrentHashMap<String, AgentProcess>) field.get(manager);
    } catch (Exception e) { throw new RuntimeException(e); }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest`
Expected: FAIL — lifecycle methods don't exist.

- [ ] **Step 3: Implement lifecycle operations**

Add to `AgentProcessManager.java` using `ide_insert_member`:

```java
public void startAgent(String terminalName, StartAgentRequest request)
        throws IOException, InterruptedException {
    verifyShellForeground(terminalName);
    String command = buildCommand(request);
    setStarting(terminalName, command);
    tmux.sendKeys(terminalName, command + "\n");
}

public void stopAgent(String terminalName) throws IOException, InterruptedException {
    var existing = agents.get(terminalName);
    if (existing == null) return;
    if (existing.state() == AgentState.PAUSED) {
        tmux.setOption(terminalName, "@trellis_agent_state", "");
        clearState(terminalName);
        broadcastState(terminalName);
        return;
    }
    if (existing.pid() > 0) {
        treeKill(terminalName, existing.pid());
    }
    clearState(terminalName);
    broadcastState(terminalName);
}

public void pauseAgent(String terminalName) throws IOException, InterruptedException {
    var existing = agents.get(terminalName);
    if (existing == null || existing.state() != AgentState.RUNNING) {
        throw new IllegalStateException("Cannot pause agent in state: " +
                (existing != null ? existing.state() : "IDLE"));
    }
    setPaused(terminalName, existing.command());
    tmux.setOption(terminalName, "@trellis_agent_state", "PAUSED");
    if (existing.pid() > 0) {
        treeKill(terminalName, existing.pid());
    }
    agents.put(terminalName, AgentProcess.paused(existing.command()));
    broadcastState(terminalName);
}

public void resumeAgent(String terminalName) throws IOException, InterruptedException {
    var existing = agents.get(terminalName);
    if (existing == null || existing.state() != AgentState.PAUSED) {
        throw new IllegalStateException("Cannot resume agent in state: " +
                (existing != null ? existing.state() : "IDLE"));
    }
    tmux.setOption(terminalName, "@trellis_agent_state", "");
    verifyShellForeground(terminalName);
    setStarting(terminalName, "claude -c");
    tmux.sendKeys(terminalName, "claude -c\n");
}

public void refreshAgent(String terminalName) throws IOException, InterruptedException {
    var existing = agents.get(terminalName);
    if (existing == null || existing.state() != AgentState.RUNNING) {
        throw new IllegalStateException("Cannot refresh agent in state: " +
                (existing != null ? existing.state() : "IDLE"));
    }
    setStarting(terminalName, "claude -c");
    if (existing.pid() > 0) {
        treeKill(terminalName, existing.pid());
    }
    Thread.sleep(500);
    tmux.sendKeys(terminalName, "claude -c\n");
}

private void verifyShellForeground(String terminalName) throws IOException, InterruptedException {
    var cmd = tmux.displayMessage(terminalName, "#{pane_current_command}").trim();
    if (!SHELL_COMMANDS.contains(cmd)) {
        throw new IllegalStateException("Terminal foreground is '" + cmd + "', not a shell");
    }
}

private String buildCommand(StartAgentRequest request) {
    if (request.resume()) return "claude -c";
    if (request.prompt() != null) {
        return "claude -p " + shellEscape(request.prompt());
    }
    return "claude";
}

static String shellEscape(String value) {
    return "'" + value.replace("'", "'\\''") + "'";
}

private void treeKill(String terminalName, long rootPid) {
    try {
        var treeOpt = ProcessTreeWalker.walk(rootPid);
        var handle = ProcessHandle.of(rootPid);
        if (handle.isPresent()) {
            handle.get().destroy();
            handle.get().onExit().orTimeout(5, java.util.concurrent.TimeUnit.SECONDS).join();
        }
        if (handle.isPresent() && handle.get().isAlive()) {
            treeOpt.ifPresent(tree -> {
                var reversed = new ArrayList<>(tree.allPids());
                Collections.reverse(reversed);
                for (long pid : reversed) {
                    ProcessHandle.of(pid).ifPresent(ProcessHandle::destroyForcibly);
                }
            });
        }
    } catch (Exception e) {
        LOG.warnf("Tree kill failed for %s (pid=%d): %s", terminalName, rootPid, e.getMessage());
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentProcessManagerTest`
Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/hortora/trellis/agent/AgentProcessManager.java src/test/java/io/hortora/trellis/agent/AgentProcessManagerTest.java
git commit -m "feat(#20): add agent lifecycle operations — start/stop/pause/resume/refresh with tree-kill"
```

---

### Task 6: REST API — Agent Sub-Resource + Terminal Updates

Update TerminalResource to return AgentSnapshot, add AgentSubResource,
wire push topics, add scheduled monitor poll, terminal destruction
coordination.

**Files:**
- Modify: `src/main/java/io/hortora/trellis/terminal/TerminalResource.java`
- Modify: `src/main/java/io/hortora/trellis/terminal/TerminalRegistry.java` — destruction coordination
- Create: `src/main/java/io/hortora/trellis/agent/AgentSubResource.java`
- Create: `src/main/java/io/hortora/trellis/agent/AgentMonitorScheduler.java`
- Test: `src/test/java/io/hortora/trellis/agent/AgentSubResourceTest.java`

**Interfaces:**
- Consumes: `AgentProcessManager.*`, `TerminalRegistry.*`, `EventBroadcaster.broadcast(topic, payload)`
- Produces: REST endpoints per spec §4

- [ ] **Step 1: Write REST endpoint tests**

Create test file:
```java
package io.hortora.trellis.agent;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.mockito.InjectMock;
import io.hortora.trellis.terminal.TerminalInfo;
import io.hortora.trellis.terminal.TerminalRegistry;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import static org.mockito.Mockito.*;

@QuarkusTest
class AgentSubResourceTest {

    @InjectMock
    TerminalRegistry registry;

    @InjectMock
    AgentProcessManager processManager;

    @Test
    void startAgentReturnsSnapshot() {
        var terminal = new TerminalInfo("t1", "/tmp", null, null, null);
        when(registry.get("t1")).thenReturn(Optional.of(terminal));
        var snapshot = new AgentSnapshot("t1", terminal,
                new AgentProcess(0, AgentState.STARTING, 0, null, "claude"), null);
        when(processManager.getSnapshot("t1", terminal)).thenReturn(snapshot);

        given()
            .contentType("application/json")
            .body("{}")
        .when()
            .post("/api/terminals/t1/agent/start")
        .then()
            .statusCode(200)
            .body("terminalName", equalTo("t1"))
            .body("process.state", equalTo("STARTING"));
    }

    @Test
    void startAgentReturns404ForUnknownTerminal() {
        when(registry.get("unknown")).thenReturn(Optional.empty());

        given()
            .contentType("application/json")
            .body("{}")
        .when()
            .post("/api/terminals/unknown/agent/start")
        .then()
            .statusCode(404)
            .body("error", containsString("terminal not found"));
    }

    @Test
    void pauseReturns409WhenIdle() throws Exception {
        var terminal = new TerminalInfo("t1", "/tmp", null, null, null);
        when(registry.get("t1")).thenReturn(Optional.of(terminal));
        doThrow(new IllegalStateException("Cannot pause agent in state: IDLE"))
                .when(processManager).pauseAgent("t1");

        given()
            .contentType("application/json")
        .when()
            .post("/api/terminals/t1/agent/pause")
        .then()
            .statusCode(409)
            .body("error", containsString("Cannot pause"));
    }

    @Test
    void listTerminalsReturnsSnapshots() {
        var terminal = new TerminalInfo("t1", "/tmp", null, null, null);
        when(registry.list()).thenReturn(java.util.List.of(terminal));
        var snapshot = new AgentSnapshot("t1", terminal, null, null);
        when(processManager.getSnapshot("t1", terminal)).thenReturn(snapshot);

        given()
        .when()
            .get("/api/terminals")
        .then()
            .statusCode(200)
            .body("size()", equalTo(1))
            .body("[0].terminalName", equalTo("t1"));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=AgentSubResourceTest`
Expected: FAIL — endpoints don't exist.

- [ ] **Step 3: Create AgentSubResource**

Use `ide_create_file`:
```java
package io.hortora.trellis.agent;

import io.hortora.trellis.terminal.TerminalInfo;
import io.hortora.trellis.terminal.TerminalRegistry;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.io.IOException;
import java.util.Map;

@Produces(MediaType.APPLICATION_JSON)
public class AgentSubResource {

    private final String terminalName;
    private final TerminalRegistry registry;
    private final AgentProcessManager processManager;

    public AgentSubResource(String terminalName, TerminalRegistry registry,
                            AgentProcessManager processManager) {
        this.terminalName = terminalName;
        this.registry = registry;
        this.processManager = processManager;
    }

    @POST
    @Path("/start")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response start(StartAgentRequest request) {
        return executeLifecycle("start", () -> {
            var req = request != null ? request : new StartAgentRequest(false, null);
            processManager.startAgent(terminalName, req);
        });
    }

    @POST
    @Path("/stop")
    public Response stop() {
        return executeLifecycle("stop", () -> processManager.stopAgent(terminalName));
    }

    @POST
    @Path("/pause")
    public Response pause() {
        return executeLifecycle("pause", () -> processManager.pauseAgent(terminalName));
    }

    @POST
    @Path("/resume")
    public Response resume() {
        return executeLifecycle("resume", () -> processManager.resumeAgent(terminalName));
    }

    @POST
    @Path("/refresh")
    public Response refresh() {
        return executeLifecycle("refresh", () -> processManager.refreshAgent(terminalName));
    }

    @GET
    @Path("/stats")
    public Response stats() {
        var terminal = registry.get(terminalName);
        if (terminal.isEmpty()) {
            return Response.status(404).entity(Map.of("error", "terminal not found: " + terminalName)).build();
        }
        return Response.ok(processManager.getSnapshot(terminalName, terminal.get())).build();
    }

    private Response executeLifecycle(String operation, LifecycleAction action) {
        var terminal = registry.get(terminalName);
        if (terminal.isEmpty()) {
            return Response.status(404).entity(Map.of("error", "terminal not found: " + terminalName)).build();
        }
        var lock = processManager.lockFor(terminalName);
        if (!lock.tryLock()) {
            return Response.status(409).entity(
                    Map.of("error", "operation already in progress for: " + terminalName)).build();
        }
        try {
            action.run();
            return Response.ok(processManager.getSnapshot(terminalName, terminal.get())).build();
        } catch (IllegalStateException e) {
            return Response.status(409).entity(Map.of("error", e.getMessage())).build();
        } catch (IllegalArgumentException e) {
            return Response.status(400).entity(Map.of("error", e.getMessage())).build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        } finally {
            lock.unlock();
        }
    }

    @FunctionalInterface
    interface LifecycleAction {
        void run() throws IOException, InterruptedException;
    }
}
```

- [ ] **Step 4: Update TerminalResource to return AgentSnapshot and delegate to AgentSubResource**

Use `ide_edit_member` on `TerminalResource.java` to update the class. The full
updated class:

```java
@Path("/api/terminals")
@Produces(MediaType.APPLICATION_JSON)
public class TerminalResource {

    @Inject
    TerminalRegistry registry;

    @Inject
    AgentProcessManager processManager;

    @GET
    public Response list() {
        var snapshots = registry.list().stream()
                .map(t -> processManager.getSnapshot(t.name(), t))
                .toList();
        return Response.ok(snapshots).build();
    }

    @GET
    @Path("/{name}")
    public Response get(@PathParam("name") String name) {
        return registry.get(name)
                .map(t -> Response.ok(processManager.getSnapshot(name, t)).build())
                .orElse(Response.status(Response.Status.NOT_FOUND)
                        .entity(Map.of("error", "terminal not found: " + name))
                        .build());
    }

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    public Response create(CreateTerminalRequest request) {
        // existing create logic, plus optional auto-start
        if (request.name() == null || request.name().isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST)
                    .entity(Map.of("error", "name is required")).build();
        }
        if (registry.get(request.name()).isPresent()) {
            return Response.status(Response.Status.CONFLICT)
                    .entity(Map.of("error", "terminal already exists: " + request.name())).build();
        }
        try {
            String workDir = request.workingDir() != null ? request.workingDir() : "/tmp";
            registry.createTerminal(request.name(), workDir, request.slot(),
                    request.repo(), request.issue());
            if (request.agent() != null) {
                processManager.startAgent(request.name(), request.agent());
            }
            var terminal = registry.get(request.name()).orElseThrow();
            return Response.status(Response.Status.CREATED)
                    .entity(processManager.getSnapshot(request.name(), terminal)).build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError()
                    .entity(Map.of("error", "failed to create terminal: " + e.getMessage())).build();
        }
    }

    @DELETE
    @Path("/{name}")
    public Response destroy(@PathParam("name") String name) {
        if (registry.get(name).isEmpty()) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", "terminal not found: " + name)).build();
        }
        try {
            registry.destroyTerminal(name);
            return Response.noContent().build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError()
                    .entity(Map.of("error", "failed to destroy terminal: " + e.getMessage())).build();
        }
    }

    @POST
    @Path("/{name}/input")
    @Consumes(MediaType.TEXT_PLAIN)
    public Response sendInput(@PathParam("name") String name, String text) {
        if (registry.get(name).isEmpty()) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity(Map.of("error", "terminal not found: " + name)).build();
        }
        try {
            registry.sendKeys(name, text);
            return Response.noContent().build();
        } catch (IOException | InterruptedException e) {
            return Response.serverError()
                    .entity(Map.of("error", "failed to send input: " + e.getMessage())).build();
        }
    }

    @Path("/{name}/agent")
    public AgentSubResource agent(@PathParam("name") String name) {
        return new AgentSubResource(name, registry, processManager);
    }

    public record CreateTerminalRequest(String name, String workingDir, String slot,
                                         String repo, String issue, StartAgentRequest agent) {}
}
```

- [ ] **Step 5: Add destroyTerminal to TerminalRegistry**

Use `ide_insert_member` on `TerminalRegistry.java`:
```java
public void destroyTerminal(String name) throws IOException, InterruptedException {
    // Agent cleanup is done by caller (TerminalResource) — registry only handles tmux
    tmux.killSession(name);
    sessions.remove(name);
}
```

Note: Rename internal field from `sessions` to `terminals` with `ide_refactor_rename`
on the field declaration. Also rename `createSession` → `createTerminal` and
`destroySession` → `destroyTerminal`.

- [ ] **Step 6: Create AgentMonitorScheduler**

Separate scheduler class that calls `AgentProcessManager.pollTerminal` for
each terminal:

```java
package io.hortora.trellis.agent;

import io.hortora.trellis.terminal.TerminalRegistry;
import io.quarkus.scheduler.Scheduled;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class AgentMonitorScheduler {

    @Inject TerminalRegistry registry;
    @Inject AgentProcessManager processManager;

    @Scheduled(every = "5s", delayed = "5s")
    void poll() {
        for (var terminal : registry.list()) {
            processManager.pollTerminal(terminal);
        }
    }
}
```

- [ ] **Step 7: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#20): add AgentSubResource REST API, monitor scheduler, terminal destruction coordination"
```

---

### Task 7: Frontend — Agent State Display and Lifecycle Controls

Add agent status badges, memory display, and lifecycle action buttons.

**Files:**
- Create: `src/main/webui/src/components/agent-status-badge.ts`
- Modify: `src/main/webui/src/components/terminal-tab-group.ts`
- Modify: `src/main/webui/src/views/slot-detail.ts`

**Interfaces:**
- Consumes: `GET /api/terminals` returning `AgentSnapshot[]`
- Consumes: `/api/push?topics=agent:state` SSE events
- Consumes: `POST /api/terminals/{name}/agent/{start|stop|pause|resume|refresh}`

- [ ] **Step 1: Create agent-status-badge component**

Use Write tool (new file):
```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('agent-status-badge')
export class AgentStatusBadge extends LitElement {

  @property() state: string = 'IDLE';
  @property({ type: Number }) memoryMb: number = 0;
  @property() lastError: string | null = null;

  static override styles = css`
    :host { display: inline-flex; align-items: center; gap: 0.4rem; font-size: 0.75rem; }

    .badge {
      display: inline-flex; align-items: center; gap: 0.25rem;
      padding: 0.1rem 0.5rem; border-radius: 4px; font-weight: 500;
    }
    .badge-running { background: #166534; color: #86efac; }
    .badge-paused { background: #854d0e; color: #fde68a; }
    .badge-idle { background: #374151; color: #9ca3af; }
    .badge-starting { background: #1e3a5f; color: #93c5fd; animation: pulse 1.5s ease-in-out infinite; }
    .badge-error { background: #7f1d1d; color: #fca5a5; }

    .memory { font-family: monospace; color: #9ca3af; }
    .memory.warning { color: #f87171; font-weight: 600; }

    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.5; }
    }
  `;

  override render() {
    const stateClass = this.lastError && this.state === 'IDLE' ? 'error' : this.state.toLowerCase();
    const label = this.lastError && this.state === 'IDLE' ? 'error' : this.state.toLowerCase();

    return html`
      <span class="badge badge-${stateClass}" title=${this.lastError ?? ''}>
        ${label}
      </span>
      ${this.state === 'RUNNING' && this.memoryMb > 0 ? html`
        <span class="memory ${this.memoryMb > 500 ? 'warning' : ''}">
          ${this.memoryMb} MB
        </span>
      ` : nothing}
    `;
  }
}
```

- [ ] **Step 2: Update terminal-tab-group with agent controls**

Modify `terminal-tab-group.ts` to show agent state and action buttons per tab.
Update the `TabEntry` interface:

```typescript
interface TabEntry {
  name: string;
  sessionName: string;
  agentState?: string;
  memoryMb?: number;
  lastError?: string | null;
}
```

Add action buttons in the tab bar and import the badge component.

- [ ] **Step 3: Update slot-detail to use AgentSnapshot**

Modify `slot-detail.ts`:
- Replace `SessionInfo` interface with data from `AgentSnapshot`
- Fetch from `/api/terminals` instead of `/api/sessions`
- Add agent lifecycle buttons (start/stop/pause/resume/refresh) using
  `POST /api/terminals/{name}/agent/{action}`
- Subscribe to push topic `agent:state` for live updates
- Show `agent-status-badge` per terminal

- [ ] **Step 4: Manual browser test**

Start dev mode: `/opt/homebrew/bin/mvn -f sidecar/pom.xml quarkus:dev`
Navigate to a slot detail view. Verify:
- Agent state badges render correctly for each state
- Memory display shows and turns red > 500 MB
- Action buttons are contextual (only valid actions shown)
- Lifecycle operations (start/pause/resume/refresh/stop) work

- [ ] **Step 5: Commit**

```bash
git add src/main/webui/src/components/agent-status-badge.ts src/main/webui/src/components/terminal-tab-group.ts src/main/webui/src/views/slot-detail.ts
git commit -m "feat(#20): add agent status badges, memory display, and lifecycle controls to frontend"
```

---

## Task Dependencies

```
Task 1 (rename) ─────────────────────────────────────┐
Task 2 (domain model) ──────────────────────────────┐ │
Task 3 (tmux + process tree) ──────────────────────┐│ │
                                                    ││ │
Task 4 (APM monitoring) ← depends on 2, 3 ────────┤│ │
                                                    ││ │
Task 5 (APM lifecycle) ← depends on 4 ────────────┤│ │
                                                    ││ │
Task 6 (REST API) ← depends on 1, 4, 5 ───────────┤│ │
                                                    ││ │
Task 7 (Frontend) ← depends on 6 ─────────────────┘┘ │
```

Tasks 1, 2, and 3 can run in parallel. Task 4 needs 2+3. Task 5 needs 4.
Task 6 needs 1+4+5. Task 7 needs 6.
