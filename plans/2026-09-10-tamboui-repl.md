# Tamboui REPL Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #75 — Tamboui REPL for mechanical work lifecycle
**Issue group:** #75

**Goal:** Build a Tamboui-based Java REPL that runs inside tmux terminals, providing mechanical work lifecycle commands with optional LLM dispatch.

**Architecture:** Standalone lightweight Java CLI (no Quarkus) using Tamboui for TUI rendering. Runs inside tmux sessions managed by the Trellis sidecar. Commands defined in YAML, loaded at runtime. Lifecycle commands delegate to soredium Python via a new JSON Lines CLI wrapper. LLM dispatch via sidecar REST API.

**Tech Stack:** Java 21+, Tamboui 0.4.0 (tamboui-tui, tamboui-toolkit, tamboui-panama-backend), Jackson (YAML/JSON), java.net.http (HTTP client), Python 3.11+ (soredium CLI wrapper)

## Global Constraints

- Java 21+ — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis.repl`
- Tamboui 0.4.0 — `dev.tamboui:tamboui-tui`, `dev.tamboui:tamboui-toolkit`, `dev.tamboui:tamboui-panama-backend`
- No Quarkus, no CDI — plain `main()` entry point
- YAML command definitions loaded at runtime — no recompile to change commands
- All sidecar interaction via REST API — never import sidecar classes directly
- Bootstrap pattern follows incus-spawn: `TuiRunner.create(TuiConfig)` → `runner.run(eventHandler, renderFn)`

---

## Batch 1: TUI Skeleton

Working after this batch: Maven module builds, Tamboui app launches in a terminal, shows a status bar and input prompt, accepts text input, exits on `quit`.

### Task 1: Maven module and Tamboui TUI bootstrap

**Files:**
- Create: `repl/pom.xml`
- Create: `repl/src/main/java/io/hortora/trellis/repl/ReplMain.java`
- Create: `repl/src/main/java/io/hortora/trellis/repl/ReplApp.java`
- Modify: `pom.xml` (root) — add `<module>repl</module>`
- Test: `repl/src/test/java/io/hortora/trellis/repl/ReplAppTest.java`

**Interfaces:**
- Produces: `ReplMain.main(String[] args)` — parses CLI args, creates `ReplApp`, runs it
- Produces: `ReplApp(ReplConfig config)` — Tamboui app with event loop
- Produces: `record ReplConfig(String repo, String slot, String issue, int sidecarPort, String pairedTerminal)`

- [ ] **Step 1: Create root pom.xml module entry**

Add `<module>repl</module>` to the `<modules>` section in `/Users/mdproctor/claude/hortora/trellis/pom.xml`.

- [ ] **Step 2: Create repl/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>io.hortora</groupId>
    <artifactId>trellis-repl</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <tamboui.version>0.4.0</tamboui.version>
        <jackson.version>2.18.2</jackson.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>dev.tamboui</groupId>
            <artifactId>tamboui-tui</artifactId>
            <version>${tamboui.version}</version>
        </dependency>
        <dependency>
            <groupId>dev.tamboui</groupId>
            <artifactId>tamboui-toolkit</artifactId>
            <version>${tamboui.version}</version>
        </dependency>
        <dependency>
            <groupId>dev.tamboui</groupId>
            <artifactId>tamboui-panama-backend</artifactId>
            <version>${tamboui.version}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
            <version>${jackson.version}</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.11.4</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.27.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.4.2</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>io.hortora.trellis.repl.ReplMain</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.2</version>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Create ReplConfig record**

Create `repl/src/main/java/io/hortora/trellis/repl/ReplConfig.java`:

```java
package io.hortora.trellis.repl;

public record ReplConfig(
    String repo,
    String slot,
    String issue,
    int sidecarPort,
    String pairedTerminal
) {
    public static ReplConfig fromArgs(String[] args) {
        String repo = "", slot = "", issue = "", paired = "";
        int port = 0;
        for (int i = 0; i < args.length - 1; i++) {
            switch (args[i]) {
                case "--repo" -> repo = args[++i];
                case "--slot" -> slot = args[++i];
                case "--issue" -> issue = args[++i];
                case "--sidecar-port" -> port = Integer.parseInt(args[++i]);
                case "--paired-terminal" -> paired = args[++i];
            }
        }
        return new ReplConfig(repo, slot, issue, port, paired);
    }
}
```

- [ ] **Step 4: Create ReplMain entry point**

Create `repl/src/main/java/io/hortora/trellis/repl/ReplMain.java`:

```java
package io.hortora.trellis.repl;

public final class ReplMain {
    public static void main(String[] args) {
        var config = ReplConfig.fromArgs(args);
        var app = new ReplApp(config);
        app.run();
    }
}
```

- [ ] **Step 5: Create ReplApp with Tamboui bootstrap**

Create `repl/src/main/java/io/hortora/trellis/repl/ReplApp.java`:

Following the incus-spawn pattern — `TuiRunner.create(TuiConfig)` with `FastPanamaBackend`, event handler for `KeyEvent`/`TickEvent`, render function drawing status bar + input area.

```java
package io.hortora.trellis.repl;

import dev.tamboui.tui.*;
import dev.tamboui.toolkit.layout.*;
import dev.tamboui.toolkit.style.*;
import dev.tamboui.toolkit.text.*;
import dev.tamboui.toolkit.widgets.*;
import dev.tamboui.backend.panama.PanamaBackend;
import java.time.Duration;

public final class ReplApp {
    private final ReplConfig config;
    private final TextInputState inputState = new TextInputState();
    private boolean running = true;

    public ReplApp(ReplConfig config) {
        this.config = config;
    }

    public void run() {
        try (var runner = TuiRunner.create(TuiConfig.builder()
                .backend(new PanamaBackend())
                .tickRate(Duration.ofMillis(100))
                .build())) {
            runner.run(this::handleEvent, this::render);
        }
    }

    private boolean handleEvent(Event event, TuiRunner tui) {
        if (!running) {
            tui.quit();
            return false;
        }
        switch (event) {
            case KeyEvent key -> handleKey(key, tui);
            case TickEvent tick -> {} // status bar refresh
            default -> {}
        }
        return true;
    }

    private void handleKey(KeyEvent key, TuiRunner tui) {
        if (key.code() == KeyCode.ENTER) {
            var input = inputState.text().trim();
            if ("quit".equals(input) || "exit".equals(input)) {
                running = false;
            }
            inputState.clear();
        } else {
            inputState.handleEvent(key);
        }
    }

    private void render(Frame frame) {
        var chunks = Layout.vertical(
            Constraint.length(1),  // status bar
            Constraint.min(1),     // output area
            Constraint.length(3)   // input area
        ).split(frame.area());

        // Status bar
        var statusText = String.format(" %s | %s | %s",
            config.repo().isEmpty() ? "no repo" : config.repo(),
            config.slot().isEmpty() ? "no slot" : config.slot(),
            config.issue().isEmpty() ? "no issue" : "#" + config.issue());
        frame.renderWidget(
            Paragraph.of(Line.of(Span.styled(statusText, Style.of().fg(Color.Cyan)))),
            chunks[0]);

        // Output area (empty for now)
        frame.renderWidget(
            Block.bordered().title(" trellis repl "),
            chunks[1]);

        // Input
        var inputWidget = TextInput.of()
            .block(Block.bordered().title(" > "));
        frame.renderStatefulWidget(inputWidget, chunks[2], inputState);
    }
}
```

- [ ] **Step 6: Write test for ReplConfig.fromArgs**

Create `repl/src/test/java/io/hortora/trellis/repl/ReplConfigTest.java`:

```java
package io.hortora.trellis.repl;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ReplConfigTest {
    @Test
    void parsesAllArgs() {
        var args = new String[]{
            "--repo", "/path/to/repo",
            "--slot", "slot-1",
            "--issue", "75",
            "--sidecar-port", "8080",
            "--paired-terminal", "trellis-engine-75"
        };
        var config = ReplConfig.fromArgs(args);
        assertThat(config.repo()).isEqualTo("/path/to/repo");
        assertThat(config.slot()).isEqualTo("slot-1");
        assertThat(config.issue()).isEqualTo("75");
        assertThat(config.sidecarPort()).isEqualTo(8080);
        assertThat(config.pairedTerminal()).isEqualTo("trellis-engine-75");
    }

    @Test
    void defaultsForMissingArgs() {
        var config = ReplConfig.fromArgs(new String[]{});
        assertThat(config.repo()).isEmpty();
        assertThat(config.sidecarPort()).isZero();
    }
}
```

- [ ] **Step 7: Run tests**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test`
Expected: PASS — ReplConfig tests green.

- [ ] **Step 8: Verify TUI compiles and builds**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml package -DskipTests`
Expected: BUILD SUCCESS — jar created with main class manifest.

- [ ] **Step 9: Commit**

```bash
git add repl/ pom.xml
git commit -m "feat(#75): scaffold repl Maven module with Tamboui TUI bootstrap Refs #75"
```

---

## Batch 2: Command Engine

Working after this batch: YAML-defined command tree loads at startup, namespace resolution with tab completion works, typing a command dispatches to a handler. The `quit`/`exit` commands work.

### Task 2: YAML command model and CommandRegistry

**Files:**
- Create: `repl/src/main/java/io/hortora/trellis/repl/command/CommandNode.java`
- Create: `repl/src/main/java/io/hortora/trellis/repl/command/GoalNode.java`
- Create: `repl/src/main/java/io/hortora/trellis/repl/command/CommandRegistry.java`
- Create: `repl/src/main/java/io/hortora/trellis/repl/command/CommandResult.java`
- Create: `repl/src/main/java/io/hortora/trellis/repl/command/CommandHandler.java`
- Create: `repl/src/main/resources/commands.yaml`
- Test: `repl/src/test/java/io/hortora/trellis/repl/command/CommandRegistryTest.java`

**Interfaces:**
- Consumes: nothing (standalone)
- Produces: `CommandRegistry.load(InputStream yaml)` — parses YAML into command tree
- Produces: `CommandRegistry.resolve(String input)` — returns `CommandNode` or null
- Produces: `CommandRegistry.complete(String prefix)` — returns `List<String>` completions
- Produces: `record CommandNode(String name, String description, String handler, Map<String, CommandNode> children)`
- Produces: `record GoalNode(String name, String description, List<GoalStep> steps, List<String> execute)`
- Produces: `sealed interface CommandResult permits CommandResult.Output, CommandResult.Error`
- Produces: `interface CommandHandler { CommandResult execute(String[] args, ReplConfig config); }`

- [ ] **Step 1: Write failing test for YAML parsing**

```java
@Test
void loadsCommandTreeFromYaml() {
    var yaml = """
        commands:
          work:
            description: "Work lifecycle"
            children:
              status:
                description: "Show status"
                handler: "soredium:status"
              pause:
                description: "Pause work"
                handler: "soredium:pause"
          git:
            description: "Git operations"
            children:
              status:
                description: "Git status"
                handler: "shell:git status"
        """;
    var registry = CommandRegistry.load(new ByteArrayInputStream(yaml.getBytes()));
    assertThat(registry.resolve("work status")).isNotNull();
    assertThat(registry.resolve("work status").handler()).isEqualTo("soredium:status");
    assertThat(registry.resolve("git status").handler()).isEqualTo("shell:git status");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test -Dtest=CommandRegistryTest#loadsCommandTreeFromYaml`
Expected: FAIL — `CommandRegistry` class not found.

- [ ] **Step 3: Implement CommandNode, GoalNode, CommandResult, CommandHandler**

`CommandNode.java`:
```java
package io.hortora.trellis.repl.command;

import java.util.Map;

public record CommandNode(
    String name,
    String description,
    String handler,
    Map<String, CommandNode> children
) {
    public boolean isLeaf() { return children == null || children.isEmpty(); }
}
```

`GoalNode.java`:
```java
package io.hortora.trellis.repl.command;

import java.util.List;
import java.util.Map;

public record GoalNode(
    String name,
    String description,
    List<GoalStep> steps,
    List<String> execute
) {
    public record GoalStep(String prompt, String source, String type, Object defaultValue) {}
}
```

`CommandResult.java`:
```java
package io.hortora.trellis.repl.command;

public sealed interface CommandResult {
    record Output(String text) implements CommandResult {}
    record Error(String message) implements CommandResult {}
}
```

`CommandHandler.java`:
```java
package io.hortora.trellis.repl.command;

import io.hortora.trellis.repl.ReplConfig;

@FunctionalInterface
public interface CommandHandler {
    CommandResult execute(String[] args, ReplConfig config);
}
```

- [ ] **Step 4: Implement CommandRegistry**

```java
package io.hortora.trellis.repl.command;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import java.io.InputStream;
import java.util.*;

public final class CommandRegistry {
    private final Map<String, CommandNode> roots;
    private final Map<String, GoalNode> goals;

    private CommandRegistry(Map<String, CommandNode> roots, Map<String, GoalNode> goals) {
        this.roots = roots;
        this.goals = goals;
    }

    @SuppressWarnings("unchecked")
    public static CommandRegistry load(InputStream yaml) {
        var mapper = new ObjectMapper(new YAMLFactory());
        try {
            var raw = mapper.readValue(yaml, Map.class);
            var commandsRaw = (Map<String, Object>) raw.getOrDefault("commands", Map.of());
            var goalsRaw = (Map<String, Object>) raw.getOrDefault("goals", Map.of());
            var roots = parseNodes(commandsRaw);
            var goals = parseGoals(goalsRaw);
            return new CommandRegistry(roots, goals);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load commands.yaml", e);
        }
    }

    public CommandNode resolve(String input) {
        var parts = input.trim().split("\\s+");
        var current = roots;
        CommandNode last = null;
        for (var part : parts) {
            var node = current.get(part);
            if (node == null) return last;
            last = node;
            current = node.children() != null ? node.children() : Map.of();
        }
        return last;
    }

    public List<String> complete(String prefix) {
        var parts = prefix.trim().split("\\s+", -1);
        if (parts.length == 0) return List.copyOf(roots.keySet());

        var current = roots;
        for (int i = 0; i < parts.length - 1; i++) {
            var node = current.get(parts[i]);
            if (node == null || node.children() == null) return List.of();
            current = node.children();
        }
        var partial = parts[parts.length - 1];
        return current.keySet().stream()
            .filter(k -> k.startsWith(partial))
            .sorted()
            .toList();
    }

    public Map<String, GoalNode> goals() { return goals; }
    public Map<String, CommandNode> roots() { return roots; }

    @SuppressWarnings("unchecked")
    private static Map<String, CommandNode> parseNodes(Map<String, Object> raw) {
        var result = new LinkedHashMap<String, CommandNode>();
        for (var entry : raw.entrySet()) {
            var val = (Map<String, Object>) entry.getValue();
            var desc = (String) val.getOrDefault("description", "");
            var handler = (String) val.get("handler");
            var childrenRaw = (Map<String, Object>) val.get("children");
            var children = childrenRaw != null ? parseNodes(childrenRaw) : Map.<String, CommandNode>of();
            result.put(entry.getKey(), new CommandNode(entry.getKey(), desc, handler, children));
        }
        return result;
    }

    @SuppressWarnings("unchecked")
    private static Map<String, GoalNode> parseGoals(Map<String, Object> raw) {
        var result = new LinkedHashMap<String, GoalNode>();
        for (var entry : raw.entrySet()) {
            var val = (Map<String, Object>) entry.getValue();
            var desc = (String) val.getOrDefault("description", "");
            var stepsRaw = (List<Map<String, Object>>) val.getOrDefault("steps", List.of());
            var steps = stepsRaw.stream().map(s -> new GoalNode.GoalStep(
                (String) s.get("prompt"), (String) s.get("source"),
                (String) s.get("type"), s.get("default")
            )).toList();
            var execute = (List<String>) val.getOrDefault("execute", List.of());
            result.put(entry.getKey(), new GoalNode(entry.getKey(), desc, steps, execute));
        }
        return result;
    }
}
```

- [ ] **Step 5: Write tab completion test**

```java
@Test
void completesPartialInput() {
    var yaml = """
        commands:
          work:
            description: "Work lifecycle"
            children:
              start:
                description: "Start work"
                handler: "soredium:start"
              status:
                description: "Show status"
                handler: "soredium:status"
              stop:
                description: "Stop work"
                handler: "soredium:stop"
        """;
    var registry = CommandRegistry.load(new ByteArrayInputStream(yaml.getBytes()));
    assertThat(registry.complete("work st")).containsExactly("start", "status", "stop");
    assertThat(registry.complete("work sta")).containsExactly("start", "status");
    assertThat(registry.complete("")).containsExactly("work");
}
```

- [ ] **Step 6: Run all tests**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test`
Expected: PASS

- [ ] **Step 7: Create initial commands.yaml**

Create `repl/src/main/resources/commands.yaml`:

```yaml
commands:
  work:
    description: "Work lifecycle"
    children:
      start:
        description: "Start work on an issue"
        handler: "soredium:start"
      continue:
        description: "Resume work on current branch"
        handler: "soredium:continue"
      pause:
        description: "Save state and switch away"
        handler: "soredium:pause"
      resume:
        description: "Restore from pause stack"
        handler: "soredium:resume"
      next:
        description: "Advance to next issue in queue"
        handler: "soredium:next"
      end:
        description: "Mechanical close (merge, push)"
        handler: "soredium:end"
      status:
        description: "Current branch, state, plan position"
        handler: "soredium:status"
      slot:
        description: "Slot operations"
        children:
          create:
            description: "Create a slot"
            handler: "soredium:slot-create"

  git:
    description: "Git operations"
    children:
      status:
        description: "Show working tree status"
        handler: "shell:git status"
      log:
        description: "Show commit log"
        handler: "shell:git log --oneline -20"
      diff:
        description: "Show working tree diff"
        handler: "shell:git diff"
      commit:
        description: "Commit with issue linkage"
        handler: "shell:git commit"
      branch:
        description: "List branches"
        handler: "shell:git branch"
      stash:
        description: "Stash management"
        handler: "shell:git stash"

  project:
    description: "Project operations"
    children:
      build:
        description: "Compile the project"
        handler: "shell:project-build"
      test:
        description: "Run tests"
        handler: "shell:project-test"
      run:
        description: "Start dev mode"
        handler: "shell:project-run"

  llm:
    description: "LLM dispatch"
    children:
      start:
        description: "Spawn agent in paired terminal"
        handler: "llm:start"
      stop:
        description: "Stop agent"
        handler: "llm:stop"
      sweep:
        description: "Full sweep (forage, protocol, doc-sync, blog)"
        handler: "llm:sweep"
      squash:
        description: "Dispatch git squash"
        handler: "llm:squash"
      review:
        description: "Dispatch code review"
        handler: "llm:review"
      blog:
        description: "Dispatch blog entry writing"
        handler: "llm:blog"
      end:
        description: "Full LLM-assisted work-end"
        handler: "llm:end"

goals:
  start-work:
    description: "Start working on an issue"
    steps:
      - prompt: "Which issue?"
        source: "github:open-issues"
        type: select
      - prompt: "ISX isolation?"
        type: confirm
        default: false
      - type: summary
      - type: confirm
        prompt: "Proceed?"
    execute:
      - "soredium:start --issue {issue}"
```

- [ ] **Step 8: Wire CommandRegistry into ReplApp**

Modify `ReplApp.java` to load commands.yaml on startup, use `CommandRegistry.complete()` for suggestions, and dispatch resolved commands via handler prefix routing.

- [ ] **Step 9: Commit**

```bash
git add repl/
git commit -m "feat(#75): YAML command registry with namespace resolution and tab completion Refs #75"
```

---

## Batch 3: Soredium Bridge

Working after this batch: A JSON Lines CLI wrapper exists in soredium. The Java SorediumBridge invokes it, parses events, and surfaces results in the REPL. `work status` works end-to-end.

### Task 3: Soredium JSON Lines CLI wrapper

**Files:**
- Create: `$SOREDIUM/cli/__init__.py`
- Create: `$SOREDIUM/cli/__main__.py`
- Test: `$SOREDIUM/tests/test_cli.py`

Where `$SOREDIUM` = `/Users/mdproctor/claude/hortora/soredium`

**Interfaces:**
- Consumes: soredium `commands/*.py` modules and their `execute()` functions
- Produces: CLI entry point `python3 -m cli <command> '<json-kwargs>'`
- Produces: JSON Lines to stdout — each line is `{"type": "<EventClassName>", "data": {...}}`
- Produces: exit code 0 on success, 1 on `CommandFailed` with `fatal=true`

- [ ] **Step 1: Write failing test**

```python
def test_cli_status_emits_jsonl(tmp_path):
    """CLI wrapper serialises StatusReady event to JSON Lines."""
    result = subprocess.run(
        [sys.executable, "-m", "cli", "status", json.dumps({"cwd": str(tmp_path)})],
        capture_output=True, text=True, cwd=SOREDIUM_ROOT
    )
    lines = [json.loads(l) for l in result.stdout.strip().splitlines() if l.strip()]
    types = [l["type"] for l in lines]
    assert "StatusReady" in types or "CommandFailed" in types
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_cli.py::test_cli_status_emits_jsonl -v`
Expected: FAIL — `cli` module not found.

- [ ] **Step 3: Create cli/__init__.py**

Empty file.

- [ ] **Step 4: Create cli/__main__.py**

```python
"""JSON Lines CLI wrapper for soredium commands."""
import json
import sys
from dataclasses import asdict
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))
sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "work-slot"))

from commands import events


def _serialise_event(event) -> str:
    type_name = type(event).__name__
    data = {}
    for field in event.__dataclass_fields__:
        val = getattr(event, field)
        if hasattr(val, '__dataclass_fields__'):
            val = asdict(val)
        elif isinstance(val, list) and val and hasattr(val[0], '__dataclass_fields__'):
            val = [asdict(v) for v in val]
        data[field] = val
    return json.dumps({"type": type_name, "data": data})


def main():
    if len(sys.argv) < 2:
        print(json.dumps({"type": "CommandFailed", "data": {
            "command": "", "step": None, "error": "Usage: python -m cli <command> [json-kwargs]",
            "detail": "", "recoverable": False
        }}))
        sys.exit(1)

    command = sys.argv[1]
    kwargs = json.loads(sys.argv[2]) if len(sys.argv) > 2 else {}

    try:
        import importlib
        mod = importlib.import_module(f"commands.{command}")

        exit_code = 0
        for event in mod.execute(**kwargs):
            print(_serialise_event(event), flush=True)
            if isinstance(event, events.CommandFailed) and not event.recoverable:
                exit_code = 1

        sys.exit(exit_code)
    except ModuleNotFoundError:
        print(json.dumps({"type": "CommandFailed", "data": {
            "command": command, "step": None,
            "error": f"Unknown command: {command}",
            "detail": "", "recoverable": False
        }}))
        sys.exit(1)
    except Exception as e:
        print(json.dumps({"type": "CommandFailed", "data": {
            "command": command, "step": None,
            "error": str(e), "detail": "", "recoverable": False
        }}))
        sys.exit(1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 5: Run test**

Run: `python3 -m pytest tests/test_cli.py -v`
Expected: PASS

- [ ] **Step 6: Commit (in soredium repo)**

```bash
git -C $SOREDIUM add cli/ tests/test_cli.py
git -C $SOREDIUM commit -m "feat: JSON Lines CLI wrapper for REPL bridge Refs Hortora/trellis#75"
```

### Task 4: Java SorediumBridge

**Files:**
- Create: `repl/src/main/java/io/hortora/trellis/repl/soredium/SorediumBridge.java`
- Create: `repl/src/main/java/io/hortora/trellis/repl/soredium/SorediumEvent.java`
- Test: `repl/src/test/java/io/hortora/trellis/repl/soredium/SorediumBridgeTest.java`

**Interfaces:**
- Consumes: soredium CLI wrapper (subprocess, JSON Lines protocol)
- Produces: `SorediumBridge(String sorediumPath, String cwd)`
- Produces: `SorediumBridge.execute(String command, Map<String, String> kwargs)` → `List<SorediumEvent>`
- Produces: `record SorediumEvent(String type, Map<String, Object> data)`

- [ ] **Step 1: Write failing test**

```java
@Test
void parsesJsonLinesOutput() {
    var lines = List.of(
        """{"type":"StepProgress","data":{"command":"status","step":"resolve","detail":"resolving context"}}""",
        """{"type":"StatusReady","data":{"branch":"main","state":"idle","on_main":true}}"""
    );
    var events = SorediumBridge.parseEvents(lines);
    assertThat(events).hasSize(2);
    assertThat(events.get(0).type()).isEqualTo("StepProgress");
    assertThat(events.get(1).type()).isEqualTo("StatusReady");
    assertThat(events.get(1).data().get("branch")).isEqualTo("main");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test -Dtest=SorediumBridgeTest`
Expected: FAIL

- [ ] **Step 3: Implement SorediumEvent and SorediumBridge**

`SorediumEvent.java`:
```java
package io.hortora.trellis.repl.soredium;

import java.util.Map;

public record SorediumEvent(String type, Map<String, Object> data) {}
```

`SorediumBridge.java`:
```java
package io.hortora.trellis.repl.soredium;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.*;
import java.util.*;

public final class SorediumBridge {
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private final String sorediumPath;
    private final String cwd;

    public SorediumBridge(String sorediumPath, String cwd) {
        this.sorediumPath = sorediumPath;
        this.cwd = cwd;
    }

    public List<SorediumEvent> execute(String command, Map<String, String> kwargs) throws Exception {
        var fullKwargs = new HashMap<>(kwargs);
        fullKwargs.put("cwd", cwd);
        var kwargsJson = MAPPER.writeValueAsString(fullKwargs);

        var pb = new ProcessBuilder("python3", "-m", "cli", command, kwargsJson)
            .directory(new File(sorediumPath))
            .redirectErrorStream(false);
        var process = pb.start();

        var lines = new BufferedReader(new InputStreamReader(process.getInputStream()))
            .lines().toList();
        process.waitFor();

        return parseEvents(lines);
    }

    @SuppressWarnings("unchecked")
    public static List<SorediumEvent> parseEvents(List<String> lines) {
        return lines.stream()
            .filter(l -> !l.isBlank())
            .map(line -> {
                try {
                    var map = MAPPER.readValue(line, new TypeReference<Map<String, Object>>() {});
                    var type = (String) map.get("type");
                    var data = (Map<String, Object>) map.getOrDefault("data", Map.of());
                    return new SorediumEvent(type, data);
                } catch (Exception e) {
                    return new SorediumEvent("ParseError", Map.of("raw", line, "error", e.getMessage()));
                }
            })
            .toList();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add repl/
git commit -m "feat(#75): SorediumBridge with JSON Lines subprocess protocol Refs #75"
```

---

## Batch 4: Sidecar Client and LLM Dispatch

Working after this batch: The REPL can query the sidecar REST API, subscribe to SSE events, and dispatch commands to the paired LLM terminal. `llm start`, `llm stop`, and `llm <freetext>` work.

### Task 5: SidecarClient — HTTP + SSE

**Files:**
- Create: `repl/src/main/java/io/hortora/trellis/repl/sidecar/SidecarClient.java`
- Test: `repl/src/test/java/io/hortora/trellis/repl/sidecar/SidecarClientTest.java`

**Interfaces:**
- Consumes: `ReplConfig.sidecarPort()`, `ReplConfig.pairedTerminal()`
- Produces: `SidecarClient(int port)`
- Produces: `SidecarClient.getTerminals()` → `String` (JSON)
- Produces: `SidecarClient.getTerminal(String name)` → `String` (JSON)
- Produces: `SidecarClient.sendInput(String terminal, String text)` → `void`
- Produces: `SidecarClient.startAgent(String terminal)` → `void`
- Produces: `SidecarClient.stopAgent(String terminal)` → `void`
- Produces: `SidecarClient.subscribeSSE(String topics, Consumer<String> listener)` → `void`

- [ ] **Step 1: Write test for URL construction**

```java
@Test
void constructsCorrectUrls() {
    var client = new SidecarClient(8080);
    assertThat(client.terminalUrl("engine-42"))
        .isEqualTo("http://localhost:8080/api/terminals/engine-42");
    assertThat(client.agentStartUrl("engine-42"))
        .isEqualTo("http://localhost:8080/api/terminals/engine-42/agent/start");
    assertThat(client.inputUrl("engine-42"))
        .isEqualTo("http://localhost:8080/api/terminals/engine-42/input");
}
```

- [ ] **Step 2: Implement SidecarClient**

```java
package io.hortora.trellis.repl.sidecar;

import java.net.URI;
import java.net.http.*;
import java.net.http.HttpResponse.BodyHandlers;
import java.util.function.Consumer;

public final class SidecarClient {
    private final HttpClient http = HttpClient.newHttpClient();
    private final String baseUrl;

    public SidecarClient(int port) {
        this.baseUrl = "http://localhost:" + port;
    }

    public String terminalUrl(String name) { return baseUrl + "/api/terminals/" + name; }
    public String agentStartUrl(String name) { return baseUrl + "/api/terminals/" + name + "/agent/start"; }
    public String agentStopUrl(String name) { return baseUrl + "/api/terminals/" + name + "/agent/stop"; }
    public String inputUrl(String name) { return baseUrl + "/api/terminals/" + name + "/input"; }

    public String getTerminal(String name) throws Exception {
        var req = HttpRequest.newBuilder(URI.create(terminalUrl(name))).GET().build();
        return http.send(req, BodyHandlers.ofString()).body();
    }

    public void sendInput(String terminal, String text) throws Exception {
        var req = HttpRequest.newBuilder(URI.create(inputUrl(terminal)))
            .header("Content-Type", "text/plain")
            .POST(HttpRequest.BodyPublishers.ofString(text + "\n"))
            .build();
        http.send(req, BodyHandlers.discarding());
    }

    public void startAgent(String terminal) throws Exception {
        var req = HttpRequest.newBuilder(URI.create(agentStartUrl(terminal)))
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString("{}"))
            .build();
        http.send(req, BodyHandlers.discarding());
    }

    public void stopAgent(String terminal) throws Exception {
        var req = HttpRequest.newBuilder(URI.create(agentStopUrl(terminal)))
            .POST(HttpRequest.BodyPublishers.ofString(""))
            .build();
        http.send(req, BodyHandlers.discarding());
    }
}
```

- [ ] **Step 3: Run tests**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test`
Expected: PASS

- [ ] **Step 4: Wire handler dispatch in ReplApp**

Modify `ReplApp.java` to route handler prefixes:
- `soredium:*` → `SorediumBridge.execute()`
- `shell:*` → `ProcessBuilder` subprocess
- `llm:*` → `SidecarClient` methods
- Display results in the output area

- [ ] **Step 5: Commit**

```bash
git add repl/
git commit -m "feat(#75): SidecarClient for REST API and LLM dispatch Refs #75"
```

---

## Batch 5: Full Command Wiring

Working after this batch: All four command namespaces (work, git, project, llm) execute end-to-end. The REPL dispatches to soredium for lifecycle, runs git/build commands as subprocesses, and sends text to the paired LLM terminal via the sidecar.

### Task 6: Handler dispatch and shell commands

**Files:**
- Create: `repl/src/main/java/io/hortora/trellis/repl/command/HandlerDispatcher.java`
- Modify: `repl/src/main/java/io/hortora/trellis/repl/ReplApp.java`
- Test: `repl/src/test/java/io/hortora/trellis/repl/command/HandlerDispatcherTest.java`

**Interfaces:**
- Consumes: `CommandRegistry.resolve()`, `SorediumBridge`, `SidecarClient`, `ReplConfig`
- Produces: `HandlerDispatcher(SorediumBridge bridge, SidecarClient sidecar, ReplConfig config)`
- Produces: `HandlerDispatcher.dispatch(String handler, String[] args)` → `CommandResult`

- [ ] **Step 1: Write failing test**

```java
@Test
void routesByHandlerPrefix() {
    var dispatcher = new HandlerDispatcher(null, null, testConfig());

    // shell: prefix runs a subprocess
    var result = dispatcher.dispatch("shell:echo hello", new String[]{});
    assertThat(result).isInstanceOf(CommandResult.Output.class);
    assertThat(((CommandResult.Output) result).text()).contains("hello");
}
```

- [ ] **Step 2: Implement HandlerDispatcher**

```java
package io.hortora.trellis.repl.command;

import io.hortora.trellis.repl.ReplConfig;
import io.hortora.trellis.repl.sidecar.SidecarClient;
import io.hortora.trellis.repl.soredium.SorediumBridge;
import java.io.*;
import java.util.*;

public final class HandlerDispatcher {
    private final SorediumBridge bridge;
    private final SidecarClient sidecar;
    private final ReplConfig config;

    public HandlerDispatcher(SorediumBridge bridge, SidecarClient sidecar, ReplConfig config) {
        this.bridge = bridge;
        this.sidecar = sidecar;
        this.config = config;
    }

    public CommandResult dispatch(String handler, String[] args) {
        var colonIdx = handler.indexOf(':');
        if (colonIdx < 0) return new CommandResult.Error("Invalid handler: " + handler);

        var prefix = handler.substring(0, colonIdx);
        var value = handler.substring(colonIdx + 1);

        return switch (prefix) {
            case "shell" -> executeShell(value, args);
            case "soredium" -> executeSoredium(value, args);
            case "llm" -> executeLlm(value, args);
            default -> new CommandResult.Error("Unknown handler prefix: " + prefix);
        };
    }

    private CommandResult executeShell(String command, String[] args) {
        try {
            var fullCommand = new ArrayList<>(List.of("sh", "-c", command + " " + String.join(" ", args)));
            var pb = new ProcessBuilder(fullCommand).directory(new File(config.repo())).redirectErrorStream(true);
            var process = pb.start();
            var output = new String(process.getInputStream().readAllBytes());
            process.waitFor();
            return new CommandResult.Output(output);
        } catch (Exception e) {
            return new CommandResult.Error(e.getMessage());
        }
    }

    private CommandResult executeSoredium(String command, String[] args) {
        if (bridge == null) return new CommandResult.Error("Soredium bridge not configured");
        try {
            var kwargs = new HashMap<String, String>();
            for (int i = 0; i < args.length - 1; i += 2) {
                kwargs.put(args[i].replaceFirst("^--", ""), args[i + 1]);
            }
            var events = bridge.execute(command, kwargs);
            var sb = new StringBuilder();
            for (var event : events) {
                sb.append(formatEvent(event)).append("\n");
            }
            return new CommandResult.Output(sb.toString());
        } catch (Exception e) {
            return new CommandResult.Error(e.getMessage());
        }
    }

    private CommandResult executeLlm(String operation, String[] args) {
        if (sidecar == null) return new CommandResult.Error("Sidecar not configured");
        var terminal = config.pairedTerminal();
        try {
            return switch (operation) {
                case "start" -> { sidecar.startAgent(terminal); yield new CommandResult.Output("Agent started"); }
                case "stop" -> { sidecar.stopAgent(terminal); yield new CommandResult.Output("Agent stopped"); }
                case "sweep" -> { sidecar.sendInput(terminal, "/forage SWEEP"); yield new CommandResult.Output("Sweep dispatched"); }
                case "squash" -> { sidecar.sendInput(terminal, "/git-squash"); yield new CommandResult.Output("Squash dispatched"); }
                case "review" -> { sidecar.sendInput(terminal, "/code-review"); yield new CommandResult.Output("Review dispatched"); }
                case "blog" -> { sidecar.sendInput(terminal, "/write-content diary"); yield new CommandResult.Output("Blog dispatched"); }
                case "end" -> { sidecar.sendInput(terminal, "work end"); yield new CommandResult.Output("LLM work-end dispatched"); }
                default -> { sidecar.sendInput(terminal, operation + " " + String.join(" ", args)); yield new CommandResult.Output("Sent to LLM"); }
            };
        } catch (Exception e) {
            return new CommandResult.Error("LLM dispatch failed: " + e.getMessage());
        }
    }

    private String formatEvent(io.hortora.trellis.repl.soredium.SorediumEvent event) {
        return switch (event.type()) {
            case "StepProgress" -> "→ " + event.data().getOrDefault("step", "") + ": " + event.data().getOrDefault("detail", "");
            case "StatusReady" -> String.format("Branch: %s | State: %s | Main: %s",
                event.data().get("branch"), event.data().get("state"), event.data().get("on_main"));
            case "CommandFailed" -> "ERROR: " + event.data().get("error") + " — " + event.data().get("detail");
            case "BranchCreated" -> "Branch created: " + event.data().get("branch");
            case "WorkEnded" -> "Work ended: " + event.data().get("branch");
            case "Paused" -> "Paused: " + event.data().get("branch");
            case "Resumed" -> "Resumed: " + event.data().get("branch");
            default -> event.type() + ": " + event.data();
        };
    }
}
```

- [ ] **Step 3: Run tests**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test`
Expected: PASS

- [ ] **Step 4: Wire HandlerDispatcher into ReplApp event handler**

Update `ReplApp.handleKey()` to:
1. On Enter: resolve input via `CommandRegistry.resolve()`
2. If resolved and has handler: `dispatcher.dispatch(handler, remainingArgs)`
3. Display result in output area
4. Clear input

- [ ] **Step 5: Commit**

```bash
git add repl/
git commit -m "feat(#75): handler dispatch — shell, soredium, and LLM command execution Refs #75"
```

---

## Batch 6: Goal Runner

Working after this batch: Goals execute as Q&A guided flows — each step prompts the user, collects answers, shows a summary, and executes the composed commands on confirmation.

### Task 7: GoalRunner with Q&A flow

**Files:**
- Create: `repl/src/main/java/io/hortora/trellis/repl/command/GoalRunner.java`
- Modify: `repl/src/main/java/io/hortora/trellis/repl/ReplApp.java`
- Test: `repl/src/test/java/io/hortora/trellis/repl/command/GoalRunnerTest.java`

**Interfaces:**
- Consumes: `GoalNode`, `HandlerDispatcher`, `ReplConfig`
- Produces: `GoalRunner(HandlerDispatcher dispatcher)`
- Produces: `GoalRunner.start(GoalNode goal)` — enters Q&A mode
- Produces: `GoalRunner.handleInput(String input)` — processes answer, advances to next step
- Produces: `GoalRunner.currentPrompt()` — returns current step's prompt
- Produces: `GoalRunner.isActive()` — true during a goal flow
- Produces: `GoalRunner.summary()` — returns formatted summary of all answers

- [ ] **Step 1: Write failing test**

```java
@Test
void collectsAnswersAndProducesSummary() {
    var goal = new GoalNode("test-goal", "Test", List.of(
        new GoalNode.GoalStep("Pick a repo", null, "text", null),
        new GoalNode.GoalStep("Pick an issue", null, "text", null),
        new GoalNode.GoalStep(null, null, "summary", null),
        new GoalNode.GoalStep("Proceed?", null, "confirm", null)
    ), List.of("soredium:start --repo {repo} --issue {issue}"));

    var runner = new GoalRunner(null);
    runner.start(goal);

    assertThat(runner.isActive()).isTrue();
    assertThat(runner.currentPrompt()).isEqualTo("Pick a repo");
    runner.handleInput("my-repo");
    assertThat(runner.currentPrompt()).isEqualTo("Pick an issue");
    runner.handleInput("42");
    // summary step auto-advances
    assertThat(runner.currentPrompt()).isEqualTo("Proceed?");
    assertThat(runner.summary()).contains("my-repo").contains("42");
}
```

- [ ] **Step 2: Implement GoalRunner**

```java
package io.hortora.trellis.repl.command;

import java.util.*;

public final class GoalRunner {
    private final HandlerDispatcher dispatcher;
    private GoalNode activeGoal;
    private int stepIndex;
    private final Map<String, String> answers = new LinkedHashMap<>();

    public GoalRunner(HandlerDispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }

    public void start(GoalNode goal) {
        this.activeGoal = goal;
        this.stepIndex = 0;
        this.answers.clear();
    }

    public boolean isActive() { return activeGoal != null; }

    public String currentPrompt() {
        if (!isActive() || stepIndex >= activeGoal.steps().size()) return null;
        var step = activeGoal.steps().get(stepIndex);
        if ("summary".equals(step.type())) return summary();
        return step.prompt();
    }

    public GoalNode.GoalStep currentStep() {
        if (!isActive() || stepIndex >= activeGoal.steps().size()) return null;
        return activeGoal.steps().get(stepIndex);
    }

    public CommandResult handleInput(String input) {
        if (!isActive()) return null;
        var step = activeGoal.steps().get(stepIndex);

        switch (step.type()) {
            case "text", "select" -> {
                var key = step.prompt().toLowerCase().replaceAll("[^a-z]", "_");
                answers.put(key, input);
                stepIndex++;
            }
            case "confirm" -> {
                if ("n".equalsIgnoreCase(input) || "no".equalsIgnoreCase(input)) {
                    activeGoal = null;
                    return new CommandResult.Output("Goal cancelled.");
                }
                stepIndex++;
            }
            case "summary" -> stepIndex++;
        }

        // Skip past summary steps (auto-display)
        while (isActive() && stepIndex < activeGoal.steps().size()
               && "summary".equals(activeGoal.steps().get(stepIndex).type())) {
            stepIndex++;
        }

        if (isActive() && stepIndex >= activeGoal.steps().size()) {
            return executeGoal();
        }
        return null;
    }

    public String summary() {
        var sb = new StringBuilder("Summary:\n");
        answers.forEach((k, v) -> sb.append("  ").append(k).append(": ").append(v).append("\n"));
        return sb.toString();
    }

    private CommandResult executeGoal() {
        var results = new StringBuilder();
        for (var cmd : activeGoal.execute()) {
            var expanded = cmd;
            for (var entry : answers.entrySet()) {
                expanded = expanded.replace("{" + entry.getKey() + "}", entry.getValue());
            }
            var parts = expanded.split("\\s+", 2);
            var handler = parts[0];
            var args = parts.length > 1 ? parts[1].split("\\s+") : new String[0];
            if (dispatcher != null) {
                var result = dispatcher.dispatch(handler, args);
                switch (result) {
                    case CommandResult.Output o -> results.append(o.text()).append("\n");
                    case CommandResult.Error e -> results.append("ERROR: ").append(e.message()).append("\n");
                }
            }
        }
        activeGoal = null;
        return new CommandResult.Output(results.toString());
    }
}
```

- [ ] **Step 3: Run tests**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test`
Expected: PASS

- [ ] **Step 4: Wire GoalRunner into ReplApp**

When the user types a goal name (e.g., `start-work`), enter goal mode. ReplApp renders the current prompt instead of the command prompt, and routes input through `GoalRunner.handleInput()` until the goal completes.

- [ ] **Step 5: Commit**

```bash
git add repl/
git commit -m "feat(#75): GoalRunner — Q&A guided flows with summary and execution Refs #75"
```

---

## Batch 7: Status Bar and SSE

Working after this batch: The REPL status bar shows real-time agent state, branch, and work state. SSE events from the sidecar update the status reactively.

### Task 8: SSE subscription and reactive status bar

**Files:**
- Modify: `repl/src/main/java/io/hortora/trellis/repl/sidecar/SidecarClient.java` — add SSE
- Create: `repl/src/main/java/io/hortora/trellis/repl/StatusModel.java`
- Modify: `repl/src/main/java/io/hortora/trellis/repl/ReplApp.java` — reactive status bar
- Test: `repl/src/test/java/io/hortora/trellis/repl/StatusModelTest.java`

**Interfaces:**
- Consumes: `SidecarClient.subscribeSSE()`, SSE `agent:state` events
- Produces: `StatusModel` — thread-safe mutable model for status bar display
- Produces: `StatusModel.updateAgentState(String state, long memoryBytes)`
- Produces: `StatusModel.updateBranch(String branch, String workState)`
- Produces: `StatusModel.render()` → `String` formatted status line

- [ ] **Step 1: Write failing test**

```java
@Test
void rendersStatusLine() {
    var model = new StatusModel("my-repo", "slot-1", "75");
    model.updateBranch("issue-75-repl", "active");
    model.updateAgentState("RUNNING", 256_000_000);
    assertThat(model.render()).contains("issue-75-repl")
        .contains("active")
        .contains("RUNNING")
        .contains("244MB");
}
```

- [ ] **Step 2: Implement StatusModel**

```java
package io.hortora.trellis.repl;

import java.util.concurrent.atomic.AtomicReference;

public final class StatusModel {
    private final String repo;
    private final String slot;
    private final String issue;
    private final AtomicReference<String> branch = new AtomicReference<>("main");
    private final AtomicReference<String> workState = new AtomicReference<>("idle");
    private final AtomicReference<String> agentState = new AtomicReference<>("IDLE");
    private final AtomicReference<Long> agentMemory = new AtomicReference<>(0L);
    private final AtomicReference<String> lastResult = new AtomicReference<>("");

    public StatusModel(String repo, String slot, String issue) {
        this.repo = repo;
        this.slot = slot;
        this.issue = issue;
    }

    public void updateBranch(String branch, String state) {
        this.branch.set(branch);
        this.workState.set(state);
    }

    public void updateAgentState(String state, long memoryBytes) {
        this.agentState.set(state);
        this.agentMemory.set(memoryBytes);
    }

    public void setLastResult(String result) { this.lastResult.set(result); }

    public String render() {
        var mem = agentMemory.get();
        var memStr = mem > 0 ? String.format("%dMB", mem / (1024 * 1024)) : "";
        var agent = agentState.get();
        var agentStr = "IDLE".equals(agent) ? "" : agent + (memStr.isEmpty() ? "" : " " + memStr);

        return String.format(" %s | %s | #%s | %s [%s] %s",
            repo.isEmpty() ? "-" : repo,
            branch.get(), issue.isEmpty() ? "-" : issue,
            workState.get(), agentStr.isEmpty() ? "no agent" : agentStr,
            lastResult.get());
    }
}
```

- [ ] **Step 3: Add SSE subscription to SidecarClient**

Add to `SidecarClient`:
```java
public void subscribeSSE(String topics, Consumer<String> listener) {
    var url = baseUrl + "/api/sse?topics=" + topics;
    Thread.startVirtualThread(() -> {
        try {
            var req = HttpRequest.newBuilder(URI.create(url)).GET().build();
            http.send(req, BodyHandlers.ofLines()).body()
                .filter(line -> line.startsWith("data:"))
                .map(line -> line.substring(5).trim())
                .forEach(listener);
        } catch (Exception e) {
            // Connection lost — will be retried on next tick
        }
    });
}
```

- [ ] **Step 4: Wire StatusModel into ReplApp**

Replace the hardcoded status bar in `render()` with `StatusModel.render()`. On startup, subscribe to `agent:state` SSE topic and update `StatusModel` on each event. The `TickEvent` handler triggers a re-render so the status bar updates reactively.

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f repl/pom.xml test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add repl/
git commit -m "feat(#75): reactive status bar with SSE agent state subscription Refs #75"
```

---

## References

- [2026-09-10-tamboui-repl-design.md] — design spec this plan implements
- [decisions.md] — 11 design decisions
- TerminalRegistry.java — tmux session management (sidecar)
- TerminalResource.java — REST API for terminals (sidecar)
- AgentSubResource.java — REST API for agent lifecycle (sidecar)
- soredium/commands/events.py — 22 event dataclasses for CLI wrapper
- incus-spawn/cli/src/.../ListCommand.java — Tamboui bootstrap pattern
- incus-spawn/cli/pom.xml — Tamboui Maven coordinates
- GitHub #75 — focal issue
