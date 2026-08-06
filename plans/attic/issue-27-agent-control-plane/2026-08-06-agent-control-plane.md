# Agent Control Plane Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #27 — Trellis Agent Control Plane — programmatic observation and interaction API
**Issue group:** #27

**Goal:** Expose Trellis's full application state as a semantic model through 6 MCP tools embedded in the Quarkus sidecar, enabling a coordinating agent to observe and control everything a human can.

**Architecture:** `quarkus-mcp-server` embedded in the sidecar. A `ModelProvider` SPI assembles the model tree from existing service beans at query time. `SessionLogger` tees terminal output to disk. Frontend pushes UI state to the sidecar via REST. Navigation commands flow over existing SSE infrastructure.

**Tech Stack:** Quarkus 3.x, quarkus-mcp-server, Java 21 records, SSE (casehub-pages-push), TypeScript/Lit (frontend)

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Existing service beans (`TerminalRegistry`, `AgentProcessManager`, `WorkspaceScanner`, `FileWatcherService`, `SlotAgentCoordinator`, `LifecycleManager`) are the authoritative API — MCP tools delegate to them
- MCP tool surface is exactly 6 tools — no more
- No separate model definition files — Java records and TS interfaces ARE the schema
- Error responses use MCP `isError: true` with text description, no structured codes
- Frontend state is opaque JSON — sidecar does not parse or validate `content`

---

### Task 1: quarkus-mcp-server dependency and bare tool bean

**Files:**
- Modify: `sidecar/pom.xml`
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/TrellisToolsTest.java`

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces: `TrellisTools` CDI bean with 6 `@Tool` stub methods returning placeholder strings

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class TrellisToolsTest {
    @Test
    void modelToolReturnsPlaceholder() {
        // TrellisTools should be injectable and return a non-null response
        var tools = CDI.current().select(TrellisTools.class).get();
        var result = tools.trellisModel(null);
        assertNotNull(result);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=TrellisToolsTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Add quarkus-mcp-server dependency to pom.xml**

Add to `<dependencies>` in `sidecar/pom.xml`:
```xml
<dependency>
    <groupId>io.quarkiverse.mcp</groupId>
    <artifactId>quarkus-mcp-server-sse</artifactId>
    <version>1.1.0</version>
</dependency>
```

- [ ] **Step 4: Create TrellisTools bean with 6 @Tool stubs**

```java
package io.hortora.trellis.mcp;

import dev.langchain4j.mcp.server.McpTool;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class TrellisTools {

    @Tool(description = "Query application state and discover available actions")
    public String trellisModel(@ToolArg(description = "Model path (e.g. 'terminals', 'terminals/engine')") String path) {
        return "{\"status\": \"not_implemented\"}";
    }

    @Tool(description = "Activate a UI element (panel, frame, tab)")
    public String trellisNavigate(@ToolArg(description = "Target model path") String target) {
        return "{\"status\": \"not_implemented\"}";
    }

    @Tool(description = "Terminal I/O (read log, send input, create, destroy)")
    public String trellisTerminal(
            @ToolArg(description = "Terminal name") String name,
            @ToolArg(description = "Operation: read-log, send-input, create, destroy, resize") String operation,
            @ToolArg(description = "Operation parameters as JSON") String params) {
        return "{\"status\": \"not_implemented\"}";
    }

    @Tool(description = "Agent lifecycle (start, stop, pause, resume)")
    public String trellisAgent(
            @ToolArg(description = "Terminal name") String terminal,
            @ToolArg(description = "Operation: start, stop, graceful-shutdown, pause, resume, refresh, stats, tree") String operation,
            @ToolArg(description = "Operation parameters as JSON") String params) {
        return "{\"status\": \"not_implemented\"}";
    }

    @Tool(description = "Slot and workspace lifecycle operations")
    public String trellisLifecycle(
            @ToolArg(description = "Operation: start, end, pause, resume, slot-create, slot-merge, epic-setup, epic-next") String operation,
            @ToolArg(description = "Operation parameters as JSON") String params) {
        return "{\"status\": \"not_implemented\"}";
    }

    @Tool(description = "Query workspace repos, slots, epics")
    public String trellisWorkspace(
            @ToolArg(description = "Workspace subpath") String path,
            @ToolArg(description = "Force fresh scan") Boolean refresh) {
        return "{\"status\": \"not_implemented\"}";
    }
}
```

Note: The exact `@Tool`/`@ToolArg` annotation import paths depend on the quarkus-mcp-server version. Check the actual artifact's API — it may use `io.quarkiverse.mcp.server.Tool` or a different package. Adjust imports during implementation.

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=TrellisToolsTest`
Expected: PASS

- [ ] **Step 6: Verify MCP server starts in dev mode**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml quarkus:dev`
Check: MCP SSE endpoint available at `/mcp/sse` (or whatever path quarkus-mcp-server configures). The 6 tools should be listed in the tool discovery response.

- [ ] **Step 7: Commit**

```bash
git -C sidecar add pom.xml src/main/java/io/hortora/trellis/mcp/TrellisTools.java src/test/java/io/hortora/trellis/mcp/TrellisToolsTest.java
git -C . commit -m "feat(#27): add quarkus-mcp-server dependency and bare TrellisTools bean with 6 @Tool stubs

Refs #27"
```

---

### Task 2: SessionLogger service bean

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/terminal/SessionLogger.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/terminal/TerminalRegistry.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/terminal/FifoRelay.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/terminal/TerminalWebSocket.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/terminal/TerminalInfo.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/terminal/SessionLoggerTest.java`

**Interfaces:**
- Consumes: `TerminalRegistry.list()`, `FifoRelay.relay()`
- Produces: `SessionLogger` — `append(byte[])`, `tailLines(String name, int lines)`, `tailLinesWithOffset(String name, int lines, int offset)`, `logPath(String name)`, `delete(String name)`. `TerminalInfo` gains a `sessionLog` field.

- [ ] **Step 1: Write the failing test for append + tail read**

```java
@QuarkusTest
class SessionLoggerTest {

    @Inject
    SessionLogger logger;

    @Test
    void appendAndTailRead(@TempDir Path tempDir) throws IOException {
        var logger = new SessionLogger(tempDir);
        logger.append("test-terminal", "line1\nline2\nline3\n".getBytes());

        var result = logger.tailLines("test-terminal", 2);
        assertEquals("line2\nline3\n", result);
    }

    @Test
    void tailReadEmptyLog(@TempDir Path tempDir) {
        var logger = new SessionLogger(tempDir);
        var result = logger.tailLines("nonexistent", 10);
        assertEquals("", result);
    }

    @Test
    void deleteRemovesLogFile(@TempDir Path tempDir) throws IOException {
        var logger = new SessionLogger(tempDir);
        logger.append("test-terminal", "data".getBytes());
        assertTrue(Files.exists(tempDir.resolve("test-terminal.log")));

        logger.delete("test-terminal");
        assertFalse(Files.exists(tempDir.resolve("test-terminal.log")));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=SessionLoggerTest`
Expected: FAIL — class not found

- [ ] **Step 3: Implement SessionLogger**

```java
package io.hortora.trellis.terminal;

import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.io.*;
import java.nio.file.*;

@ApplicationScoped
public class SessionLogger {

    private final Path sessionsDir;

    public SessionLogger(
            @ConfigProperty(name = "trellis.session-log.dir",
                    defaultValue = "${java.io.tmpdir}/trellis-sessions") Path sessionsDir) {
        this.sessionsDir = sessionsDir;
        try {
            Files.createDirectories(sessionsDir);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    // Test constructor
    SessionLogger(Path sessionsDir) {
        this.sessionsDir = sessionsDir;
    }

    public void append(String terminalName, byte[] data) {
        try {
            Files.write(logPath(terminalName), data,
                    StandardOpenOption.CREATE, StandardOpenOption.APPEND);
        } catch (IOException e) {
            // Log but don't fail the relay — session log is best-effort
            System.err.println("Session log write failed for " + terminalName + ": " + e);
        }
    }

    public String tailLines(String terminalName, int lines) {
        return tailLinesWithOffset(terminalName, lines, 0);
    }

    public String tailLinesWithOffset(String terminalName, int lines, int offset) {
        var path = logPath(terminalName);
        if (!Files.exists(path)) return "";

        try (var raf = new RandomAccessFile(path.toFile(), "r")) {
            long fileLength = raf.length();
            if (fileLength == 0) return "";

            // Seek backwards to find (lines + offset) newlines
            int totalLines = lines + offset;
            int newlinesFound = 0;
            long pos = fileLength - 1;

            while (pos > 0 && newlinesFound < totalLines) {
                raf.seek(pos);
                if (raf.readByte() == '\n') newlinesFound++;
                pos--;
            }

            long startPos = (pos == 0 && newlinesFound < totalLines) ? 0 : pos + 2;

            // Now find where to stop (skip the last 'offset' lines)
            if (offset > 0) {
                long endPos = fileLength;
                int skipLines = 0;
                long ep = fileLength - 1;
                while (ep > startPos && skipLines < offset) {
                    raf.seek(ep);
                    if (raf.readByte() == '\n') skipLines++;
                    ep--;
                }
                endPos = (skipLines < offset) ? startPos : ep + 2;
                raf.seek(startPos);
                byte[] buf = new byte[(int) (endPos - startPos)];
                raf.readFully(buf);
                return new String(buf);
            }

            raf.seek(startPos);
            byte[] buf = new byte[(int) (fileLength - startPos)];
            raf.readFully(buf);
            return new String(buf);
        } catch (IOException e) {
            return "";
        }
    }

    public Path logPath(String terminalName) {
        return sessionsDir.resolve(terminalName + ".log");
    }

    public void delete(String terminalName) {
        try {
            Files.deleteIfExists(logPath(terminalName));
        } catch (IOException e) {
            System.err.println("Session log delete failed for " + terminalName + ": " + e);
        }
    }

    public void appendMarker(String terminalName, String text) {
        var marker = "\033[?2004h" + text + "\033[?2004l";
        append(terminalName, marker.getBytes());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=SessionLoggerTest`
Expected: PASS

- [ ] **Step 5: Add sessionLog field to TerminalInfo**

Use `ide_edit_member` to add `String sessionLog` to the `TerminalInfo` record. Update all call sites that construct `TerminalInfo` (`TerminalRegistry.createSession()`, `TerminalRegistry.bootstrap()`).

- [ ] **Step 6: Wire SessionLogger into FifoRelay**

Modify `FifoRelay` to accept an optional `SessionLogger` and terminal name. In `relay()`, after each `sink.accept()`, call `logger.append(terminalName, data)`.

Modify `TerminalWebSocket.onOpen()` to pass `SessionLogger` and terminal name when constructing the `FifoRelay`.

- [ ] **Step 7: Wire cleanup into TerminalRegistry.destroySession()**

Add `SessionLogger` injection to `TerminalRegistry`. In `destroySession()`, call `sessionLogger.delete(name)` after destroying the tmux session.

- [ ] **Step 8: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: All tests pass

- [ ] **Step 9: Commit**

```bash
git commit -m "feat(#27): SessionLogger — append-only terminal session logs with tail-read

Refs #27"
```

---

### Task 3: ModelProvider SPI and TerminalModelProvider

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/ModelProvider.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/ActionDescriptor.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/TerminalModelProvider.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/GenerationCounter.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/TerminalModelProviderTest.java`

**Interfaces:**
- Consumes: `TerminalRegistry.list()`, `AgentProcessManager.getAllSnapshots()`, `SessionLogger.logPath()`
- Produces: `ModelProvider` interface (`domain()`, `summary()`, `resolve(subpath)`, `actionsFor(nodeType)`), `ActionDescriptor` record, `TerminalModelProvider`, `GenerationCounter`

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class TerminalModelProviderTest {

    @Inject
    TerminalModelProvider provider;

    @Test
    void domainIsTerminals() {
        assertEquals("terminals", provider.domain());
    }

    @Test
    void summaryReturnsTerminalList() {
        var summary = provider.summary();
        assertNotNull(summary);
        // Should be a list (may be empty if no terminals)
        assertTrue(summary instanceof java.util.List);
    }

    @Test
    void actionsForTerminalIncludesSendInput() {
        var actions = provider.actionsFor("terminal");
        assertTrue(actions.stream().anyMatch(a -> a.name().equals("send-input")));
        assertTrue(actions.stream().anyMatch(a -> a.name().equals("read-log")));
        assertTrue(actions.stream().anyMatch(a -> a.name().equals("start-agent")));
        // All should be backend source
        assertTrue(actions.stream().allMatch(a -> a.source().equals("backend")));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TerminalModelProviderTest`
Expected: FAIL

- [ ] **Step 3: Create ActionDescriptor record**

```java
package io.hortora.trellis.mcp;

public record ActionDescriptor(
        String name,
        String description,
        String source,
        String tool,
        String operation
) {
    public static ActionDescriptor backend(String name, String description, String tool, String operation) {
        return new ActionDescriptor(name, description, "backend", tool, operation);
    }
}
```

- [ ] **Step 4: Create ModelProvider interface**

```java
package io.hortora.trellis.mcp;

import java.util.List;

public interface ModelProvider {
    String domain();
    Object summary();
    Object resolve(String subpath);
    List<ActionDescriptor> actionsFor(String nodeType);
}
```

- [ ] **Step 5: Create GenerationCounter**

```java
package io.hortora.trellis.mcp;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.atomic.AtomicLong;

@ApplicationScoped
public class GenerationCounter {

    private final AtomicLong counter = new AtomicLong(0);

    public long increment() {
        return counter.incrementAndGet();
    }

    public long current() {
        return counter.get();
    }
}
```

- [ ] **Step 6: Create TerminalModelProvider**

```java
package io.hortora.trellis.mcp;

import io.hortora.trellis.agent.AgentProcessManager;
import io.hortora.trellis.terminal.SessionLogger;
import io.hortora.trellis.terminal.TerminalRegistry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.*;

@ApplicationScoped
public class TerminalModelProvider implements ModelProvider {

    @Inject TerminalRegistry registry;
    @Inject AgentProcessManager processManager;
    @Inject SessionLogger sessionLogger;

    private static final List<ActionDescriptor> TERMINAL_ACTIONS = List.of(
            ActionDescriptor.backend("send-input", "Send text input to terminal", "trellis_terminal", "send-input"),
            ActionDescriptor.backend("read-log", "Read session log", "trellis_terminal", "read-log"),
            ActionDescriptor.backend("start-agent", "Start an agent in this terminal", "trellis_agent", "start"),
            ActionDescriptor.backend("stop-agent", "Stop the running agent", "trellis_agent", "stop"),
            ActionDescriptor.backend("graceful-shutdown-agent", "Gracefully shutdown agent", "trellis_agent", "graceful-shutdown"),
            ActionDescriptor.backend("pause-agent", "Pause the running agent", "trellis_agent", "pause"),
            ActionDescriptor.backend("resume-agent", "Resume a paused agent", "trellis_agent", "resume"),
            ActionDescriptor.backend("refresh-agent", "Refresh agent state", "trellis_agent", "refresh"),
            ActionDescriptor.backend("destroy", "Destroy this terminal", "trellis_terminal", "destroy")
    );

    @Override
    public String domain() {
        return "terminals";
    }

    @Override
    public Object summary() {
        var terminals = registry.list();
        var snapshots = processManager.getAllSnapshots(terminals);
        return snapshots.stream().map(s -> {
            var map = new LinkedHashMap<String, Object>();
            map.put("name", s.terminalName());
            map.put("workingDir", s.terminal().workingDir());
            map.put("slot", s.terminal().slot());
            map.put("repo", s.terminal().repo());
            map.put("issue", s.terminal().issue());
            map.put("sessionLog", sessionLogger.logPath(s.terminalName()).toString());
            if (s.process() != null) {
                var agent = new LinkedHashMap<String, Object>();
                agent.put("state", s.process().state().name());
                agent.put("pid", s.process().pid());
                agent.put("memoryBytes", s.process().memoryBytes());
                agent.put("startedAt", s.process().startedAt() != null ? s.process().startedAt().toString() : null);
                agent.put("command", s.process().command());
                map.put("agent", agent);
            }
            if (s.lastError() != null) {
                map.put("lastError", s.lastError());
            }
            map.put("actions", TERMINAL_ACTIONS);
            return map;
        }).toList();
    }

    @Override
    public Object resolve(String subpath) {
        if (subpath == null || subpath.isEmpty()) return summary();
        var terminal = registry.get(subpath);
        if (terminal.isEmpty()) return null;
        // Return single terminal with full detail
        var snapshot = processManager.getSnapshot(subpath, terminal.get());
        var map = new LinkedHashMap<String, Object>();
        map.put("name", snapshot.terminalName());
        map.put("workingDir", snapshot.terminal().workingDir());
        map.put("slot", snapshot.terminal().slot());
        map.put("repo", snapshot.terminal().repo());
        map.put("issue", snapshot.terminal().issue());
        map.put("sessionLog", sessionLogger.logPath(subpath).toString());
        if (snapshot.process() != null) {
            var agent = new LinkedHashMap<String, Object>();
            agent.put("state", snapshot.process().state().name());
            agent.put("pid", snapshot.process().pid());
            agent.put("memoryBytes", snapshot.process().memoryBytes());
            agent.put("startedAt", snapshot.process().startedAt() != null ? snapshot.process().startedAt().toString() : null);
            agent.put("command", snapshot.process().command());
            map.put("agent", agent);
        }
        if (snapshot.lastError() != null) map.put("lastError", snapshot.lastError());
        map.put("actions", TERMINAL_ACTIONS);
        return map;
    }

    @Override
    public List<ActionDescriptor> actionsFor(String nodeType) {
        if ("terminal".equals(nodeType)) return TERMINAL_ACTIONS;
        return List.of();
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TerminalModelProviderTest`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git commit -m "feat(#27): ModelProvider SPI, TerminalModelProvider, GenerationCounter

Refs #27"
```

---

### Task 4: WorkspaceModelProvider

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/WorkspaceModelProvider.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/WorkspaceModelProviderTest.java`

**Interfaces:**
- Consumes: `FileWatcherService.currentModel()`, `FileWatcherService.allModels()`
- Produces: `WorkspaceModelProvider` — summary returns `{root, slotCount, activeSlot}`, `resolve()` returns full `WorkspaceModel` data

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class WorkspaceModelProviderTest {

    @Inject
    WorkspaceModelProvider provider;

    @Test
    void domainIsWorkspace() {
        assertEquals("workspace", provider.domain());
    }

    @Test
    void summaryReturnsSummaryNotFullModel() {
        var summary = provider.summary();
        assertNotNull(summary);
        // Summary should be a map with root, slotCount — not full repo/slot lists
        assertTrue(summary instanceof Map);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=WorkspaceModelProviderTest`
Expected: FAIL

- [ ] **Step 3: Implement WorkspaceModelProvider**

```java
package io.hortora.trellis.mcp;

import io.hortora.trellis.scanner.FileWatcherService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.*;

@ApplicationScoped
public class WorkspaceModelProvider implements ModelProvider {

    @Inject FileWatcherService fileWatcher;

    @Override
    public String domain() {
        return "workspace";
    }

    @Override
    public Object summary() {
        var models = fileWatcher.allModels();
        if (models.isEmpty()) return Map.of();
        var model = models.getFirst();
        var map = new LinkedHashMap<String, Object>();
        map.put("root", model.root().toString());
        map.put("slotCount", model.slots().size());
        var activeSlot = model.slots().stream()
                .filter(s -> s.status().name().equals("ACTIVE"))
                .findFirst();
        map.put("activeSlot", activeSlot.map(s -> "slot-" + s.number()).orElse(null));
        return map;
    }

    @Override
    public Object resolve(String subpath) {
        var models = fileWatcher.allModels();
        if (models.isEmpty()) return null;
        var model = models.getFirst();
        if (subpath == null || subpath.isEmpty()) {
            return fullModel(model);
        }
        // Resolve sub-paths: repos, slots, epics, pauses
        return switch (subpath.split("/")[0]) {
            case "repos" -> model.repos();
            case "slots" -> model.slots();
            case "epics" -> model.epics();
            case "pauses" -> model.pauses();
            default -> null;
        };
    }

    private Map<String, Object> fullModel(io.hortora.trellis.scanner.WorkspaceModel model) {
        var map = new LinkedHashMap<String, Object>();
        map.put("root", model.root().toString());
        map.put("repos", model.repos());
        map.put("slots", model.slots());
        map.put("epics", model.epics());
        map.put("pauses", model.pauses());
        return map;
    }

    @Override
    public List<ActionDescriptor> actionsFor(String nodeType) {
        return List.of();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=WorkspaceModelProviderTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#27): WorkspaceModelProvider — summary + full model from FileWatcherService

Refs #27"
```

---

### Task 5: UI state endpoint and UIStateModelProvider

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/UIStateStore.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/UIStateResource.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/mcp/UIStateModelProvider.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/UIStateResourceTest.java`

**Interfaces:**
- Consumes: nothing external
- Produces: `POST /api/model/ui-state` endpoint, `UIStateStore` (in-memory), `UIStateModelProvider`

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class UIStateResourceTest {

    @Test
    void postAndGetUIState() {
        var state = """
                {"activePanel": "workspace", "panels": {"workspace": {"visible": true, "content": {}, "lastPushed": 1234}}}
                """;
        given()
                .contentType("application/json")
                .body(state)
                .when().post("/api/model/ui-state")
                .then().statusCode(204);

        // Verify via model provider
        var store = CDI.current().select(UIStateStore.class).get();
        var current = store.current();
        assertNotNull(current);
        assertTrue(current.containsKey("activePanel"));
    }

    @Test
    void rejectOversizedContent() {
        var huge = "{\"activePanel\": \"x\", \"panels\": {\"x\": {\"visible\": true, \"content\": \"" + "a".repeat(70000) + "\", \"lastPushed\": 1}}}";
        given()
                .contentType("application/json")
                .body(huge)
                .when().post("/api/model/ui-state")
                .then().statusCode(413);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=UIStateResourceTest`
Expected: FAIL

- [ ] **Step 3: Create UIStateStore**

```java
package io.hortora.trellis.mcp;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;

@ApplicationScoped
public class UIStateStore {

    private static final int MAX_CONTENT_SIZE = 65536;

    private final AtomicReference<Map<String, Object>> state = new AtomicReference<>();
    private final GenerationCounter generation;

    public UIStateStore(GenerationCounter generation) {
        this.generation = generation;
    }

    public void update(Map<String, Object> newState) {
        state.set(newState);
        generation.increment();
    }

    public Map<String, Object> current() {
        return state.get();
    }

    public static int maxContentSize() {
        return MAX_CONTENT_SIZE;
    }
}
```

- [ ] **Step 4: Create UIStateResource**

```java
package io.hortora.trellis.mcp;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.Map;

@Path("/api/model")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UIStateResource {

    @Inject UIStateStore store;
    @Inject ObjectMapper mapper;

    @POST
    @Path("/ui-state")
    public Response updateUIState(String body) {
        if (body.length() > UIStateStore.maxContentSize()) {
            return Response.status(413).entity("{\"error\": \"content too large\"}").build();
        }
        try {
            @SuppressWarnings("unchecked")
            var state = mapper.readValue(body, Map.class);
            store.update(state);
            return Response.noContent().build();
        } catch (Exception e) {
            return Response.status(400).entity("{\"error\": \"invalid JSON\"}").build();
        }
    }
}
```

- [ ] **Step 5: Create UIStateModelProvider**

```java
package io.hortora.trellis.mcp;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.*;

@ApplicationScoped
public class UIStateModelProvider implements ModelProvider {

    @Inject UIStateStore store;

    @Override
    public String domain() {
        return "ui";
    }

    @Override
    public Object summary() {
        var state = store.current();
        if (state == null) return Map.of("connected", false);
        var result = new LinkedHashMap<>(state);
        // Check staleness of each panel
        if (result.get("panels") instanceof Map<?, ?> panels) {
            var annotated = new LinkedHashMap<>();
            for (var entry : panels.entrySet()) {
                if (entry.getValue() instanceof Map<?, ?> panel) {
                    var copy = new LinkedHashMap<>((Map<?, ?>) panel);
                    var lastPushed = panel.get("lastPushed");
                    if (lastPushed instanceof Number ts) {
                        boolean stale = Instant.now().toEpochMilli() - ts.longValue() > 30_000;
                        copy.put("stale", stale);
                    }
                    annotated.put(entry.getKey(), copy);
                } else {
                    annotated.put(entry.getKey(), entry.getValue());
                }
            }
            result.put("panels", annotated);
        }
        return result;
    }

    @Override
    public Object resolve(String subpath) {
        var state = store.current();
        if (state == null) return null;
        if (subpath == null || subpath.isEmpty()) return summary();
        // Resolve dock-bar or panels/{name}
        if (subpath.startsWith("dock-bar")) {
            return state.get("activePanel");
        }
        if (subpath.startsWith("panels/")) {
            var panelName = subpath.substring("panels/".length()).split("/")[0];
            if (state.get("panels") instanceof Map<?, ?> panels) {
                return panels.get(panelName);
            }
        }
        return null;
    }

    @Override
    public List<ActionDescriptor> actionsFor(String nodeType) {
        return List.of();
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=UIStateResourceTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#27): UI state endpoint, UIStateStore, UIStateModelProvider

POST /api/model/ui-state accepts frontend state push. 64KB limit.
Staleness detection via lastPushed timestamp.

Refs #27"
```

---

### Task 6: trellis_model tool — model assembly and path resolution

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/TrellisModelTest.java`

**Interfaces:**
- Consumes: `Instance<ModelProvider>`, `GenerationCounter`
- Produces: `trellisModel(path)` — returns assembled model tree as JSON

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class TrellisModelTest {

    @Inject
    TrellisTools tools;

    @Test
    void modelWithNoPathReturnsTopLevel() {
        var result = tools.trellisModel(null);
        assertNotNull(result);
        // Should contain terminals, workspace, generation
        assertTrue(result.contains("terminals"));
        assertTrue(result.contains("workspace"));
        assertTrue(result.contains("generation"));
    }

    @Test
    void modelWithTerminalsPathReturnsTerminals() {
        var result = tools.trellisModel("terminals");
        assertNotNull(result);
        // Should be a list (may be empty)
        assertTrue(result.startsWith("[") || result.contains("terminals"));
    }

    @Test
    void modelWithInvalidPathReturnsError() {
        var result = tools.trellisModel("nonexistent/path");
        assertTrue(result.contains("not_found"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisModelTest`
Expected: FAIL (stubs return placeholder)

- [ ] **Step 3: Implement trellisModel in TrellisTools**

Wire `Instance<ModelProvider>` and `GenerationCounter` into `TrellisTools`. The `trellisModel` method:
1. Captures current generation
2. If no path: assembles full tree from all providers' `summary()` methods
3. If path: splits on `/`, finds the provider whose `domain()` matches the first segment, calls `resolve()` with the remainder
4. Wraps response with `generation` field
5. On invalid path: returns error with `not_found`

Use `ObjectMapper` for JSON serialisation.

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisModelTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#27): trellis_model — model assembly from providers with path resolution

Refs #27"
```

---

### Task 7: trellis_terminal and trellis_agent tools

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/terminal/TerminalRegistry.java` (add resize method)
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/TrellisTerminalTest.java`

**Interfaces:**
- Consumes: `TerminalRegistry`, `AgentProcessManager`, `SessionLogger`
- Produces: Working `trellisTerminal()` and `trellisAgent()` methods

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class TrellisTerminalTest {

    @Inject TrellisTools tools;

    @Test
    void readLogReturnsSessionContent() {
        // Depends on SessionLogger being wired — test with a known terminal
        var result = tools.trellisTerminal("nonexistent", "read-log", "{\"lines\": 10}");
        // Should return empty string, not error (no log = empty)
        assertNotNull(result);
    }

    @Test
    void invalidOperationReturnsError() {
        var result = tools.trellisTerminal("test", "invalid-op", null);
        assertTrue(result.contains("isError") || result.contains("invalid"));
    }

    @Test
    void agentStatsForNonexistentTerminal() {
        var result = tools.trellisAgent("nonexistent", "stats", null);
        assertTrue(result.contains("not found") || result.contains("isError"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisTerminalTest`
Expected: FAIL

- [ ] **Step 3: Add resize method to TerminalRegistry**

```java
public void resize(String name, int cols, int rows) {
    tmux.run("resize-pane", "-t", name, "-x", String.valueOf(cols), "-y", String.valueOf(rows));
}
```

- [ ] **Step 4: Implement trellisTerminal dispatch**

Parse `params` JSON. Switch on `operation`:
- `read-log` → `sessionLogger.tailLines(name, lines)` / `tailLinesWithOffset(name, lines, offset)`
- `send-input` → `sessionLogger.appendMarker(name, text)` then `registry.sendKeys(name, text)`
- `create` → `registry.createSession(name, workingDir, slot, repo, issue)`
- `destroy` → `registry.destroySession(name)`
- `resize` → `registry.resize(name, cols, rows)`
- default → error response

Wrap exceptions with MCP `isError` responses.

- [ ] **Step 5: Implement trellisAgent dispatch**

Parse `params` JSON. Switch on `operation`:
- `start` → parse `StartAgentRequest`, call `processManager.startAgent(terminal, request)`
- `stop` → `processManager.stopAgent(terminal)`
- `graceful-shutdown` → `processManager.gracefulShutdown(terminal)`
- `pause` → `processManager.pauseAgent(terminal)`
- `resume` → `processManager.resumeAgent(terminal)`
- `refresh` → `processManager.refreshAgent(terminal)`
- `stats` → `processManager.getSnapshot(terminal, registry.get(terminal).orElseThrow())`
- `tree` → delegate to `AgentSubResource.tree()` logic
- default → error response

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisTerminalTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#27): trellis_terminal and trellis_agent — terminal I/O and agent lifecycle MCP tools

Refs #27"
```

---

### Task 8: trellis_lifecycle and trellis_workspace tools

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/TrellisLifecycleTest.java`

**Interfaces:**
- Consumes: `SlotAgentCoordinator`, `LifecycleManager`, `FileWatcherService`
- Produces: Working `trellisLifecycle()` and `trellisWorkspace()` methods

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class TrellisLifecycleTest {

    @Inject TrellisTools tools;

    @Test
    void invalidLifecycleOperationReturnsError() {
        var result = tools.trellisLifecycle("invalid-op", null);
        assertTrue(result.contains("invalid"));
    }

    @Test
    void workspaceQueryReturnsModel() {
        var result = tools.trellisWorkspace(null, false);
        assertNotNull(result);
        // Should return workspace data or empty if no workspace watched
    }

    @Test
    void workspaceWithRefreshForcesScan() {
        var result = tools.trellisWorkspace(null, true);
        assertNotNull(result);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisLifecycleTest`
Expected: FAIL

- [ ] **Step 3: Implement trellisLifecycle dispatch**

Parse `params` JSON. Switch on `operation`:
- `start` → `lifecycleManager.start(workspaceRoot, branch, issue)`
- `end` → `coordinator.coordinatedEnd(slotId, workspaceRoot)`
- `pause` → `coordinator.coordinatedPause(slotId, workspaceRoot)`
- `resume` → `coordinator.coordinatedResume(slotId, workspaceRoot)`
- `slot-create` → `lifecycleManager.slotCreate(workspaceRoot, args)`
- `slot-merge` → `lifecycleManager.slotMerge(slotId, workspaceRoot)`
- `epic-setup` → `lifecycleManager.epicSetup(workspaceRoot, args)`
- `epic-next` → `lifecycleManager.epicNext(epicPath)`
- default → error response

- [ ] **Step 4: Implement trellisWorkspace**

If `refresh` is true, call `fileWatcher.onWorkspaceChanged(root)` to invalidate cache, then return full model. Otherwise return `fileWatcher.allModels()` data. If `path` is specified, resolve sub-path (repos, slots, epics, pauses).

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisLifecycleTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#27): trellis_lifecycle and trellis_workspace — slot lifecycle and workspace query MCP tools

Refs #27"
```

---

### Task 9: trellis_navigate — SSE navigation with correlation ID

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/mcp/UIStateStore.java` (add correlation tracking)
- Modify: `sidecar/src/main/java/io/hortora/trellis/mcp/UIStateResource.java` (accept correlationId)
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/TrellisNavigateTest.java`

**Interfaces:**
- Consumes: `EventBroadcaster`, `UIStateStore`
- Produces: Working `trellisNavigate()` with correlation-based acknowledgment

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class TrellisNavigateTest {

    @Inject TrellisTools tools;
    @Inject UIStateStore store;

    @Test
    void navigateWithNoFrontendReturnsError() {
        // No UI state pushed — should return no_frontend error
        var result = tools.trellisNavigate("dock-bar/workspace");
        assertTrue(result.contains("no_frontend") || result.contains("no frontend"));
    }

    @Test
    void navigateWithStaleFrontendReturnsError() {
        // Push state with old timestamp
        store.update(Map.of("activePanel", "dashboard", "panels", Map.of()));
        // Wait for staleness (or mock the timestamp)
        var result = tools.trellisNavigate("dock-bar/workspace");
        // Should detect stale frontend
        assertNotNull(result);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisNavigateTest`
Expected: FAIL

- [ ] **Step 3: Add correlation tracking to UIStateStore**

Add a `ConcurrentHashMap<String, CompletableFuture<Map<String, Object>>>` for pending navigations. Methods: `registerNavigation(correlationId)` returns a `CompletableFuture`, `acknowledgeNavigation(correlationId, state)` completes it. Timeout cleanup: entries older than 10s are purged.

- [ ] **Step 4: Update UIStateResource to accept correlationId**

When the POST body contains a `correlationId` field, call `store.acknowledgeNavigation(correlationId, state)`.

- [ ] **Step 5: Implement trellisNavigate**

1. Check UIStateStore for recent push (within 30s). If none → return `no_frontend` error.
2. Generate UUID correlation ID.
3. Register pending navigation via `store.registerNavigation(correlationId)`.
4. Emit `control:navigate` SSE event via `EventBroadcaster` with `{target, correlationId}`.
5. Await the `CompletableFuture` with 5s timeout.
6. On completion → return post-navigation state.
7. On timeout → clean up pending entry, return `timeout` error.

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisNavigateTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#27): trellis_navigate — SSE navigation with correlation-based acknowledgment

Refs #27"
```

---

### Task 10: Frontend — UI state push and navigation handler

**Files:**
- Modify: `sidecar/src/main/webui/src/components/workbench.ts`
- Test: `sidecar/src/main/webui/src/components/workbench.test.ts` (add tests)

**Interfaces:**
- Consumes: `POST /api/model/ui-state`, `control:navigate` SSE topic
- Produces: Workbench pushes UI state on panel switch and 15s heartbeat. Handles `control:navigate` events.

- [ ] **Step 1: Write the failing test**

```typescript
describe('UI state push', () => {
    it('should push UI state on panel switch', async () => {
        // Mock fetch
        const fetchCalls: any[] = [];
        globalThis.fetch = async (url: string, opts: any) => {
            fetchCalls.push({ url, body: JSON.parse(opts.body) });
            return new Response(null, { status: 204 });
        };

        const el = await fixture(html`<trellis-workbench></trellis-workbench>`);
        // Trigger panel switch
        el._switchPanel('workspace');
        await aTimeout(100);

        const pushCall = fetchCalls.find(c => c.url.includes('/api/model/ui-state'));
        expect(pushCall).to.exist;
        expect(pushCall.body.activePanel).to.equal('workspace');
    });
});

describe('Navigation handler', () => {
    it('should switch panel on control:navigate event', async () => {
        const el = await fixture(html`<trellis-workbench></trellis-workbench>`);
        // Simulate SSE navigate event
        el._handleNavigateEvent({ target: 'dock-bar/artifacts', correlationId: 'test-123' });
        expect(el._activePanel).to.equal('artifacts');
    });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd sidecar/src/main/webui test`
Expected: FAIL — methods don't exist

- [ ] **Step 3: Add UI state push to workbench**

Add to `trellis-workbench`:
- `_pushUIState()` method that POSTs current state to `/api/model/ui-state`
- Call on panel switch (immediate)
- Debounced push on other state changes (1s debounce, 5s max-wait)
- 15s heartbeat interval via `setInterval`
- Include `correlationId` in push if one is pending

Each panel component contributes its `content` via a `getUIState()` method (returns `unknown`). The workbench collects from all cached panels.

- [ ] **Step 4: Add navigation event handler**

Subscribe to `control:navigate` SSE topic (via the existing EventSource connection). On receiving the event:
- Parse `target` path
- Switch panel if target starts with `dock-bar/`
- Focus frame/tab if target starts with `panels/workspace-view/frames/`
- Push updated UI state with the `correlationId` from the event

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn --cwd sidecar/src/main/webui test`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test && yarn --cwd sidecar/src/main/webui test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#27): frontend UI state push and control:navigate handler

Workbench pushes UI state on panel switch + 15s heartbeat.
Handles control:navigate SSE for agent-driven navigation.

Refs #27"
```

---

### Task 11: Wire GenerationCounter into service beans and integration test

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/terminal/TerminalRegistry.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentProcessManager.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/GenerationCounterTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/mcp/IntegrationTest.java`

**Interfaces:**
- Consumes: `GenerationCounter`
- Produces: Generation counter increments on terminal create/destroy, agent state changes, workspace changes

- [ ] **Step 1: Write the failing test**

```java
@QuarkusTest
class GenerationCounterTest {

    @Inject GenerationCounter counter;
    @Inject TrellisTools tools;

    @Test
    void modelResponseIncludesGeneration() {
        var result = tools.trellisModel(null);
        assertTrue(result.contains("\"generation\""));
    }

    @Test
    void generationIncrementsOnMutation() {
        long before = counter.current();
        // Any mutation should increment — UI state push is easiest
        var store = CDI.current().select(UIStateStore.class).get();
        store.update(Map.of("activePanel", "test"));
        long after = counter.current();
        assertTrue(after > before);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=GenerationCounterTest`
Expected: FAIL (generation not wired)

- [ ] **Step 3: Wire GenerationCounter into TerminalRegistry**

Inject `GenerationCounter` into `TerminalRegistry`. Call `generation.increment()` in `createSession()` and `destroySession()`.

- [ ] **Step 4: Wire GenerationCounter into AgentProcessManager**

Inject `GenerationCounter` into `AgentProcessManager`. Call `generation.increment()` in `broadcastState()` (covers all agent state transitions).

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=GenerationCounterTest`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#27): wire GenerationCounter into TerminalRegistry and AgentProcessManager

Generation increments on terminal create/destroy and agent state changes.
Model responses include generation for freshness detection.

Refs #27"
```

---

### Task 12: CLAUDE.md update and final verification

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: all previous tasks
- Produces: updated CLAUDE.md with MCP conventions

- [ ] **Step 1: Update CLAUDE.md with MCP conventions**

Add to Key Conventions:
```
- `quarkus-mcp-server` embedded in sidecar — 6 `@Tool` methods on `TrellisTools` CDI bean
- MCP tool surface is stable at 6 tools — new capabilities extend the model, not the tool list
- `ModelProvider` SPI for model tree assembly — one provider per domain
- `SessionLogger` appends terminal output to `{data-dir}/sessions/{name}.log` — append-only, tail-read via RandomAccessFile
- `POST /api/model/ui-state` — frontend pushes UI state, sidecar serves as opaque JSON
- `control:navigate` SSE topic — command convention for agent-driven UI navigation
```

- [ ] **Step 2: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test && yarn --cwd sidecar/src/main/webui test`
Expected: All tests pass

- [ ] **Step 3: Commit**

```bash
git commit -m "docs(#27): CLAUDE.md — MCP control plane conventions

Refs #27"
```
