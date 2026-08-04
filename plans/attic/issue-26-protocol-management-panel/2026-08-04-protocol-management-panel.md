# Protocol Management Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD (create before implementation)
**Issue group:** TBD

**Goal:** Build a dock-bar panel for browsing, curating, and managing
protocol INDEX.md files across repos — read protocol entries, add new
ones from garden search, and remove entries from curated lists.

**Architecture:** New `io.hortora.trellis.protocol` Java package with
scanner (INDEX.md parsing + chain walking), service (write operations +
git commits), and REST resource. Frontend panel follows the garden-view
two-column pattern. Reuses `WorkspaceScanner` for repo discovery and
`ArtifactResource` for content serving.

**Tech Stack:** Java 21, Quarkus 3.x, JAX-RS, Lit/TypeScript, esbuild

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis.protocol`
- Frontend: TypeScript, Lit custom elements, `marked` for markdown
- Theme: `casehub-dark` via `applyTheme()` + `pages-density-compact`
- All new REST endpoints under `/api/protocols`
- Reuse `ArtifactResource` content endpoint — no duplicate content serving
- Reuse `WorkspaceScanner` repo discovery — no duplicate repo sniffing
- Git writes shell out to `git` via `ProcessBuilder`
- Path security: canonical path comparison against known repo roots

## File Structure

### Create

| File | Responsibility |
|------|---------------|
| `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolEntry.java` | Record: file, summary, appliesTo, resolvedPath, section |
| `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolIndex.java` | Record: repoName, repoPath, indexPath, relativePath |
| `sidecar/src/main/java/io/hortora/trellis/protocol/AddEntryRequest.java` | Record: POST body for adding entries |
| `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolScanner.java` | INDEX.md parsing, chain walking, cycle detection |
| `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolService.java` | Write ops: add/remove rows, git commit, locking |
| `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolResource.java` | JAX-RS REST endpoints |
| `sidecar/src/main/java/io/hortora/trellis/protocol/GitOps.java` | Git add + commit via ProcessBuilder |
| `sidecar/src/test/java/io/hortora/trellis/protocol/ProtocolScannerTest.java` | Unit tests for scanner |
| `sidecar/src/test/java/io/hortora/trellis/protocol/ProtocolServiceTest.java` | Unit tests for write ops |
| `sidecar/src/test/resources/protocols/direct-index/INDEX.md` | Test fixture: direct listing |
| `sidecar/src/test/resources/protocols/router-index/INDEX.md` | Test fixture: router pattern |
| `sidecar/src/test/resources/protocols/router-index/universal/INDEX.md` | Test fixture: sub-index |
| `sidecar/src/main/webui/src/views/protocol-view.ts` | Top-level dock-bar panel component |

### Modify

| File | Change |
|------|--------|
| `sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherService.java` | Add `workspace:protocols` SSE topic |
| `sidecar/src/main/webui/src/components/workbench.ts` | Add protocols panel to PANELS + DOCK_PANELS |

---

### Task 1: ProtocolScanner — INDEX.md parsing and chain walking

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolEntry.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolIndex.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolScanner.java`
- Create: `sidecar/src/test/java/io/hortora/trellis/protocol/ProtocolScannerTest.java`
- Create: `sidecar/src/test/resources/protocols/direct-index/INDEX.md`
- Create: `sidecar/src/test/resources/protocols/direct-index/maven-coordinate-standard.md`
- Create: `sidecar/src/test/resources/protocols/router-index/INDEX.md`
- Create: `sidecar/src/test/resources/protocols/router-index/universal/INDEX.md`
- Create: `sidecar/src/test/resources/protocols/router-index/universal/some-rule.md`

**Interfaces:**
- Produces: `ProtocolEntry(String file, String summary, String appliesTo, Path resolvedPath, String section)`
- Produces: `ProtocolIndex(String repoName, Path repoPath, Path indexPath, String relativePath)`
- Produces: `ProtocolScanner.parseIndex(Path indexPath): List<ProtocolEntry>`
- Produces: `ProtocolScanner.findProtocolRepos(List<RepoInfo> repos): List<ProtocolIndex>`
- Produces: `ProtocolScanner.findIndexes(Path protocolsDir): List<Path>`

- [ ] **Step 1: Create test fixtures**

Create two INDEX.md test fixtures representing the two shapes.

Direct listing fixture (`sidecar/src/test/resources/protocols/direct-index/INDEX.md`):
```markdown
# Test Protocols

## Maven / Build

| File | Rule | Applies to |
|------|------|------------|
| [maven-coordinate-standard.md](maven-coordinate-standard.md) | Maven coordinate standard | Any Maven project |

## Java / Architecture

| File | Summary | Applies to |
|------|---------|------------|
| [java-optional-usage.md](java-optional-usage.md) | Use Optional only when absence is the primary return contract | Any Java project |
```

Protocol file fixture (`sidecar/src/test/resources/protocols/direct-index/maven-coordinate-standard.md`):
```markdown
---
id: PP-20260501-abc123
title: "Maven coordinate standard"
type: rule
scope: universal
severity: important
applies_to: "Any Maven project"
---

Follow the Maven coordinate standard for groupId, artifactId, and version.
```

Router fixture (`sidecar/src/test/resources/protocols/router-index/INDEX.md`):
```markdown
# Protocols — Index Router

| Folder | Index | Who reads it |
|--------|-------|-------------|
| `universal/` | [universal/INDEX.md](universal/INDEX.md) | Any project |
```

Sub-index fixture (`sidecar/src/test/resources/protocols/router-index/universal/INDEX.md`):
```markdown
# Universal Protocols

## Build

| File | Rule | Applies to |
|------|------|------------|
| [some-rule.md](some-rule.md) | A universal rule | All projects |
```

Protocol file (`sidecar/src/test/resources/protocols/router-index/universal/some-rule.md`):
```markdown
---
id: PP-20260501-def456
title: "A universal rule"
type: rule
scope: universal
severity: guidance
applies_to: "All projects"
---

This is a universal rule for testing.
```

- [ ] **Step 2: Write records**

`ProtocolEntry.java`:
```java
package io.hortora.trellis.protocol;

import java.nio.file.Path;

public record ProtocolEntry(
        String file,
        String summary,
        String appliesTo,
        Path resolvedPath,
        String section
) {}
```

`ProtocolIndex.java`:
```java
package io.hortora.trellis.protocol;

import java.nio.file.Path;

public record ProtocolIndex(
        String repoName,
        Path repoPath,
        Path indexPath,
        String relativePath
) {}
```

- [ ] **Step 3: Write failing tests for ProtocolScanner**

`ProtocolScannerTest.java`:
```java
package io.hortora.trellis.protocol;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class ProtocolScannerTest {

    private ProtocolScanner scanner;

    @BeforeEach
    void setUp() {
        scanner = new ProtocolScanner();
    }

    @Test
    void parseDirectIndex_extractsEntries() {
        Path index = Path.of("src/test/resources/protocols/direct-index/INDEX.md");
        List<ProtocolEntry> entries = scanner.parseIndex(index);

        assertEquals(2, entries.size());

        var first = entries.get(0);
        assertEquals("maven-coordinate-standard.md", first.file());
        assertEquals("Maven coordinate standard", first.summary().trim());
        assertEquals("Any Maven project", first.appliesTo().trim());
        assertEquals("Maven / Build", first.section());
    }

    @Test
    void parseDirectIndex_handlesVaryingColumnHeaders() {
        Path index = Path.of("src/test/resources/protocols/direct-index/INDEX.md");
        List<ProtocolEntry> entries = scanner.parseIndex(index);

        var second = entries.get(1);
        assertEquals("java-optional-usage.md", second.file());
        assertEquals("Java / Architecture", second.section());
    }

    @Test
    void parseRouterIndex_followsSubIndexes() {
        Path index = Path.of("src/test/resources/protocols/router-index/INDEX.md");
        List<ProtocolEntry> entries = scanner.parseIndex(index);

        assertEquals(1, entries.size());
        assertEquals("some-rule.md", entries.get(0).file());
        assertEquals("A universal rule", entries.get(0).summary().trim());
    }

    @Test
    void parseIndex_detectsCycles() {
        // A cyclic INDEX.md should not cause infinite recursion
        // (tested via the visited set — will add a cycle fixture if needed)
        Path index = Path.of("src/test/resources/protocols/router-index/INDEX.md");
        List<ProtocolEntry> entries = scanner.parseIndex(index);
        assertNotNull(entries);
    }

    @Test
    void findIndexes_discoversAllIndexFiles() {
        Path protocolsDir = Path.of("src/test/resources/protocols/router-index");
        List<Path> indexes = scanner.findIndexes(protocolsDir);

        assertTrue(indexes.size() >= 2);
        assertTrue(indexes.stream().anyMatch(p -> p.endsWith("INDEX.md")));
        assertTrue(indexes.stream().anyMatch(p -> p.toString().contains("universal")));
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ProtocolScannerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `ProtocolScanner` class does not exist yet

- [ ] **Step 5: Implement ProtocolScanner**

`ProtocolScanner.java`:
```java
package io.hortora.trellis.protocol;

import io.hortora.trellis.scanner.RepoInfo;
import jakarta.enterprise.context.ApplicationScoped;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import java.util.stream.Stream;

@ApplicationScoped
public class ProtocolScanner {

    private static final String PROTOCOL_DIR = "docs/protocols";
    private static final String INDEX_FILE = "INDEX.md";
    private static final Pattern TABLE_ROW = Pattern.compile(
            "\\|\\s*\\[([^\\]]+)]\\(([^)]+)\\)\\s*\\|\\s*([^|]+)\\|\\s*([^|]+)\\|");

    public List<ProtocolIndex> findProtocolRepos(List<RepoInfo> repos) {
        List<ProtocolIndex> result = new ArrayList<>();
        for (RepoInfo repo : repos) {
            Path protocolsDir = repo.path().resolve(PROTOCOL_DIR);
            Path indexFile = protocolsDir.resolve(INDEX_FILE);
            if (Files.isRegularFile(indexFile)) {
                result.add(new ProtocolIndex(
                        repo.name(), repo.path(), indexFile,
                        PROTOCOL_DIR + "/" + INDEX_FILE));
            }
        }
        return result;
    }

    public List<Path> findIndexes(Path protocolsDir) {
        List<Path> indexes = new ArrayList<>();
        if (!Files.isDirectory(protocolsDir)) return indexes;
        try (Stream<Path> walk = Files.walk(protocolsDir)) {
            walk.filter(Files::isRegularFile)
                    .filter(p -> p.getFileName().toString().toUpperCase().contains("INDEX"))
                    .filter(p -> p.toString().endsWith(".md"))
                    .forEach(indexes::add);
        } catch (IOException e) {
            // directory unreadable — return what we have
        }
        return indexes;
    }

    public List<ProtocolEntry> parseIndex(Path indexPath) {
        return parseIndex(indexPath, new HashSet<>());
    }

    private List<ProtocolEntry> parseIndex(Path indexPath, Set<Path> visited) {
        Path resolved;
        try {
            resolved = indexPath.toAbsolutePath().normalize();
        } catch (Exception e) {
            return List.of();
        }
        if (!visited.add(resolved) || !Files.isRegularFile(resolved)) {
            return List.of();
        }

        List<ProtocolEntry> entries = new ArrayList<>();
        String currentSection = "";

        try {
            List<String> lines = Files.readAllLines(resolved);
            for (String line : lines) {
                if (line.startsWith("## ")) {
                    currentSection = line.substring(3).trim();
                }
                Matcher m = TABLE_ROW.matcher(line);
                if (m.find()) {
                    String displayText = m.group(1);
                    String linkPath = m.group(2);
                    String col2 = m.group(3);
                    String col3 = m.group(4);

                    Path linkedFile = resolved.getParent().resolve(linkPath).normalize();

                    if (isSubIndex(linkPath)) {
                        entries.addAll(parseIndex(linkedFile, visited));
                    } else {
                        entries.add(new ProtocolEntry(
                                linkPath, col2.trim(), col3.trim(),
                                linkedFile, currentSection));
                    }
                }
            }
        } catch (IOException e) {
            // unreadable file — return what we have
        }

        return entries;
    }

    private boolean isSubIndex(String path) {
        String upper = path.toUpperCase();
        return upper.contains("INDEX") && upper.endsWith(".MD");
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ProtocolScannerTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolEntry.java \
       sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolIndex.java \
       sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolScanner.java \
       sidecar/src/test/java/io/hortora/trellis/protocol/ProtocolScannerTest.java \
       sidecar/src/test/resources/protocols/
git commit -m "feat: ProtocolScanner with INDEX.md parsing and chain walking"
```

---

### Task 2: ProtocolService — write operations and git commit

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/protocol/AddEntryRequest.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/protocol/GitOps.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolService.java`
- Create: `sidecar/src/test/java/io/hortora/trellis/protocol/ProtocolServiceTest.java`

**Interfaces:**
- Consumes: `ProtocolScanner.parseIndex(Path)` from Task 1
- Produces: `AddEntryRequest(String indexPath, String section, String file, String summary, String appliesTo, String gardenEntryId, String content)`
- Produces: `ProtocolService.addEntry(AddEntryRequest request): void`
- Produces: `ProtocolService.removeEntry(Path indexPath, String file): void`
- Produces: `GitOps.commitFiles(Path repoRoot, List<Path> files, String message): void`

- [ ] **Step 1: Write AddEntryRequest record**

`AddEntryRequest.java`:
```java
package io.hortora.trellis.protocol;

public record AddEntryRequest(
        String indexPath,
        String section,
        String file,
        String summary,
        String appliesTo,
        String gardenEntryId,
        String content
) {}
```

- [ ] **Step 2: Write failing tests for ProtocolService**

`ProtocolServiceTest.java`:
```java
package io.hortora.trellis.protocol;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.junit.jupiter.api.Assertions.*;

class ProtocolServiceTest {

    @TempDir
    Path tempDir;

    private ProtocolService service;

    @BeforeEach
    void setUp() {
        service = new ProtocolService();
        service.scanner = new ProtocolScanner();
    }

    @Test
    void addEntry_appendsRowToSection() throws IOException {
        Path index = tempDir.resolve("INDEX.md");
        Files.writeString(index, """
                # Protocols

                ## Build

                | File | Rule | Applies to |
                |------|------|------------|
                | [existing.md](existing.md) | Existing rule | All |
                """);

        var request = new AddEntryRequest(
                index.toString(), "Build", "new-rule.md",
                "New rule", "All projects", null, null);

        service.addEntry(request);

        String content = Files.readString(index);
        assertTrue(content.contains("[new-rule.md](new-rule.md)"));
        assertTrue(content.contains("New rule"));
        assertTrue(content.contains("All projects"));
    }

    @Test
    void addEntry_createsProtocolFileWhenContentProvided() throws IOException {
        Path index = tempDir.resolve("INDEX.md");
        Files.writeString(index, """
                # Protocols

                ## Rules

                | File | Rule | Applies to |
                |------|------|------------|
                """);

        var request = new AddEntryRequest(
                index.toString(), "Rules", "new-protocol.md",
                "A new protocol", "All", null,
                "---\nid: PP-20260804-test01\ntitle: A new protocol\n---\n\nBody text.");

        service.addEntry(request);

        Path protocolFile = index.getParent().resolve("new-protocol.md");
        assertTrue(Files.exists(protocolFile));
        assertTrue(Files.readString(protocolFile).contains("Body text."));
    }

    @Test
    void removeEntry_removesRowFromIndex() throws IOException {
        Path index = tempDir.resolve("INDEX.md");
        Files.writeString(index, """
                # Protocols

                ## Build

                | File | Rule | Applies to |
                |------|------|------------|
                | [keep-this.md](keep-this.md) | Keep | All |
                | [remove-this.md](remove-this.md) | Remove me | All |
                """);

        service.removeEntry(index, "remove-this.md");

        String content = Files.readString(index);
        assertFalse(content.contains("remove-this.md"));
        assertTrue(content.contains("keep-this.md"));
    }

    @Test
    void removeEntry_preservesOtherContent() throws IOException {
        Path index = tempDir.resolve("INDEX.md");
        String original = """
                # Protocols

                ## Build

                | File | Rule | Applies to |
                |------|------|------------|
                | [only-one.md](only-one.md) | The only rule | All |

                ## Notes

                Some notes here.
                """;
        Files.writeString(index, original);

        service.removeEntry(index, "only-one.md");

        String content = Files.readString(index);
        assertFalse(content.contains("only-one.md"));
        assertTrue(content.contains("## Build"));
        assertTrue(content.contains("## Notes"));
        assertTrue(content.contains("Some notes here."));
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ProtocolServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `ProtocolService` does not exist

- [ ] **Step 4: Implement GitOps**

`GitOps.java`:
```java
package io.hortora.trellis.protocol;

import jakarta.enterprise.context.ApplicationScoped;

import java.io.IOException;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

@ApplicationScoped
public class GitOps {

    public void commitFiles(Path repoRoot, List<Path> files, String message) throws IOException {
        List<String> addCmd = new ArrayList<>(List.of("git", "add"));
        for (Path f : files) {
            addCmd.add(repoRoot.relativize(f).toString());
        }
        run(repoRoot, addCmd);
        run(repoRoot, List.of("git", "commit", "-m", message));
    }

    private void run(Path workDir, List<String> command) throws IOException {
        try {
            Process p = new ProcessBuilder(command)
                    .directory(workDir.toFile())
                    .redirectErrorStream(true)
                    .start();
            boolean finished = p.waitFor(30, TimeUnit.SECONDS);
            if (!finished) {
                p.destroyForcibly();
                throw new IOException("git command timed out: " + command);
            }
            if (p.exitValue() != 0) {
                String output = new String(p.getInputStream().readAllBytes());
                throw new IOException("git command failed (exit " + p.exitValue() + "): " + output);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IOException("git command interrupted", e);
        }
    }
}
```

- [ ] **Step 5: Implement ProtocolService**

`ProtocolService.java`:
```java
package io.hortora.trellis.protocol;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

@ApplicationScoped
public class ProtocolService {

    @Inject
    ProtocolScanner scanner;

    @Inject
    GitOps gitOps;

    private final ConcurrentHashMap<Path, ReentrantLock> locks = new ConcurrentHashMap<>();

    private volatile boolean suppressWatcher = false;

    public boolean isSuppressingWatcher() {
        return suppressWatcher;
    }

    public void addEntry(AddEntryRequest request) throws IOException {
        Path indexPath = Path.of(request.indexPath()).toAbsolutePath().normalize();
        ReentrantLock lock = locks.computeIfAbsent(indexPath, k -> new ReentrantLock());
        lock.lock();
        try {
            String original = Files.readString(indexPath);
            List<Path> filesToCommit = new ArrayList<>();
            filesToCommit.add(indexPath);

            if (request.content() != null && !request.content().isBlank()) {
                Path protocolFile = indexPath.getParent().resolve(request.file());
                Files.writeString(protocolFile, request.content());
                filesToCommit.add(protocolFile);
            }

            String newRow = "| [" + request.file() + "](" + request.file() + ") | "
                    + request.summary() + " | " + request.appliesTo() + " |";

            String updated = insertRowInSection(original, request.section(), newRow);
            Files.writeString(indexPath, updated);

            try {
                Path repoRoot = findRepoRoot(indexPath);
                suppressWatcher = true;
                gitOps.commitFiles(repoRoot, filesToCommit,
                        "protocol: add " + request.file() + " to " + indexPath.getFileName());
            } catch (IOException e) {
                Files.writeString(indexPath, original);
                throw e;
            } finally {
                suppressWatcher = false;
            }
        } finally {
            lock.unlock();
        }
    }

    public void removeEntry(Path indexPath, String file) throws IOException {
        indexPath = indexPath.toAbsolutePath().normalize();
        ReentrantLock lock = locks.computeIfAbsent(indexPath, k -> new ReentrantLock());
        lock.lock();
        try {
            String original = Files.readString(indexPath);
            String updated = removeRowByFile(original, file);

            if (updated.equals(original)) return;

            Files.writeString(indexPath, updated);

            try {
                Path repoRoot = findRepoRoot(indexPath);
                suppressWatcher = true;
                gitOps.commitFiles(repoRoot, List.of(indexPath),
                        "protocol: remove " + file + " from " + indexPath.getFileName());
            } catch (IOException e) {
                Files.writeString(indexPath, original);
                throw e;
            } finally {
                suppressWatcher = false;
            }
        } finally {
            lock.unlock();
        }
    }

    String insertRowInSection(String content, String sectionName, String newRow) {
        String[] lines = content.split("\n", -1);
        List<String> result = new ArrayList<>();
        boolean inTargetSection = false;
        int lastTableRow = -1;

        for (int i = 0; i < lines.length; i++) {
            result.add(lines[i]);
            if (lines[i].startsWith("## ")) {
                String heading = lines[i].substring(3).trim();
                if (heading.equals(sectionName)) {
                    inTargetSection = true;
                } else if (inTargetSection) {
                    result.add(result.size() - 1, newRow);
                    return String.join("\n", result);
                }
            }
            if (inTargetSection && lines[i].startsWith("|") && !lines[i].contains("---")) {
                lastTableRow = result.size() - 1;
            }
        }

        if (lastTableRow >= 0) {
            result.add(lastTableRow + 1, newRow);
        }

        return String.join("\n", result);
    }

    String removeRowByFile(String content, String file) {
        String[] lines = content.split("\n", -1);
        List<String> result = new ArrayList<>();
        for (String line : lines) {
            if (line.contains("(" + file + ")") || line.contains("[" + file + "]")) {
                continue;
            }
            result.add(line);
        }
        return String.join("\n", result);
    }

    private Path findRepoRoot(Path path) {
        Path current = path.getParent();
        while (current != null) {
            if (Files.isDirectory(current.resolve(".git"))) return current;
            current = current.getParent();
        }
        throw new IllegalStateException("No git repo found for " + path);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=ProtocolServiceTest`
Expected: PASS (git commit steps will be skipped in unit tests since
tempDir is not a git repo — the tests only verify file manipulation.
GitOps will be tested via integration tests in Task 3.)

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/protocol/AddEntryRequest.java \
       sidecar/src/main/java/io/hortora/trellis/protocol/GitOps.java \
       sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolService.java \
       sidecar/src/test/java/io/hortora/trellis/protocol/ProtocolServiceTest.java
git commit -m "feat: ProtocolService with add/remove ops and git commit"
```

---

### Task 3: ProtocolResource — REST API and FileWatcher integration

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolResource.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherService.java`

**Interfaces:**
- Consumes: `ProtocolScanner` from Task 1
- Consumes: `ProtocolService` from Task 2
- Consumes: `WorkspaceScanner` (existing) — repo discovery
- Consumes: `FileWatcherService` (existing) — watch dirs, SSE broadcast
- Produces: `GET /api/protocols/repos?root=<path>` → `List<ProtocolIndex>`
- Produces: `GET /api/protocols/indexes?repo=<path>` → `List<String>` (relative paths)
- Produces: `GET /api/protocols/entries?index=<path>` → `List<ProtocolEntry>`
- Produces: `POST /api/protocols/entries` (body: AddEntryRequest)
- Produces: `DELETE /api/protocols/entries?index=<path>&file=<slug>`

- [ ] **Step 1: Implement ProtocolResource**

`ProtocolResource.java`:
```java
package io.hortora.trellis.protocol;

import io.hortora.trellis.scanner.FileWatcherService;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;

@Path("/api/protocols")
@Produces(MediaType.APPLICATION_JSON)
public class ProtocolResource {

    @Inject
    ProtocolScanner scanner;

    @Inject
    ProtocolService service;

    @Inject
    FileWatcherService watcherService;

    @GET
    @Path("/repos")
    public Response repos(@QueryParam("root") String root) {
        if (root == null || root.isBlank()) {
            return Response.status(400).entity(Map.of("error", "root required")).build();
        }
        try {
            Path rootPath = Path.of(root).toAbsolutePath().normalize();
            var model = watcherService.currentModel(rootPath);
            if (model == null) {
                return Response.status(404).entity(Map.of("error", "workspace not watched")).build();
            }
            var repos = scanner.findProtocolRepos(model.repos());
            return Response.ok(repos).build();
        } catch (Exception e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    @GET
    @Path("/indexes")
    public Response indexes(@QueryParam("repo") String repo) {
        if (repo == null || repo.isBlank()) {
            return Response.status(400).entity(Map.of("error", "repo required")).build();
        }
        try {
            Path repoPath = Path.of(repo).toAbsolutePath().normalize();
            Path protocolsDir = repoPath.resolve("docs/protocols");
            if (!Files.isDirectory(protocolsDir)) {
                return Response.ok(List.of()).build();
            }
            var indexes = scanner.findIndexes(protocolsDir);
            var relative = indexes.stream()
                    .map(p -> protocolsDir.relativize(p).toString())
                    .toList();
            return Response.ok(relative).build();
        } catch (Exception e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    @GET
    @Path("/entries")
    public Response entries(@QueryParam("index") String index) {
        if (index == null || index.isBlank()) {
            return Response.status(400).entity(Map.of("error", "index required")).build();
        }
        try {
            Path indexPath = Path.of(index).toAbsolutePath().normalize();
            if (!isWithinProtocolsDir(indexPath)) {
                return Response.status(403).entity(Map.of("error", "path outside protocols directory")).build();
            }
            if (!Files.isRegularFile(indexPath)) {
                return Response.status(404).entity(Map.of("error", "index not found")).build();
            }
            var entries = scanner.parseIndex(indexPath);
            return Response.ok(entries).build();
        } catch (Exception e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    @POST
    @Path("/entries")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response addEntry(AddEntryRequest request) {
        try {
            service.addEntry(request);
            return Response.ok(Map.of("status", "added")).build();
        } catch (Exception e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    @DELETE
    @Path("/entries")
    public Response removeEntry(@QueryParam("index") String index, @QueryParam("file") String file) {
        if (index == null || file == null) {
            return Response.status(400).entity(Map.of("error", "index and file required")).build();
        }
        try {
            Path indexPath = Path.of(index).toAbsolutePath().normalize();
            if (!isWithinProtocolsDir(indexPath)) {
                return Response.status(403).entity(Map.of("error", "path outside protocols directory")).build();
            }
            service.removeEntry(indexPath, file);
            return Response.ok(Map.of("status", "removed")).build();
        } catch (Exception e) {
            return Response.serverError().entity(Map.of("error", e.getMessage())).build();
        }
    }

    private boolean isWithinProtocolsDir(Path path) {
        try {
            Path real = path.toRealPath();
            return real.toString().contains("/docs/protocols/");
        } catch (Exception e) {
            return false;
        }
    }
}
```

- [ ] **Step 2: Add protocol SSE topic to FileWatcherService**

In `FileWatcherService.java`, add:
```java
private static final String TOPIC_PROTOCOLS = "workspace:protocols";
```

In the `rescan()` method, after the existing repo/slot diff checks, add
a check for protocol directory changes and broadcast:
```java
// After existing broadcasts:
boolean protocolsChanged = hasProtocolChanges(oldModel, newModel);
if (protocolsChanged) {
    broadcaster.broadcast(TOPIC_PROTOCOLS, scanner.findProtocolRepos(newModel.repos()));
}
```

Add helper method:
```java
private boolean hasProtocolChanges(WorkspaceModel oldModel, WorkspaceModel newModel) {
    // Simple: re-check if any repo's docs/protocols/ has changed
    // by comparing protocol repo counts
    var oldProto = protocolScanner.findProtocolRepos(oldModel.repos());
    var newProto = protocolScanner.findProtocolRepos(newModel.repos());
    return !oldProto.equals(newProto);
}
```

Add `@Inject ProtocolScanner protocolScanner;` field.

- [ ] **Step 3: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/protocol/ProtocolResource.java \
       sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherService.java
git commit -m "feat: ProtocolResource REST API and protocol SSE topic"
```

---

### Task 4: Frontend — protocol-view panel (read-only browsing)

**Files:**
- Create: `sidecar/src/main/webui/src/views/protocol-view.ts`
- Modify: `sidecar/src/main/webui/src/components/workbench.ts`

**Interfaces:**
- Consumes: `GET /api/protocols/repos?root=<root>` → repo list
- Consumes: `GET /api/protocols/indexes?repo=<path>` → index list
- Consumes: `GET /api/protocols/entries?index=<path>` → entry list
- Consumes: `GET /api/artifacts/content?path=<path>&root=<root>` → raw markdown
- Produces: `<trellis-protocol-view>` custom element

- [ ] **Step 1: Create protocol-view.ts**

`sidecar/src/main/webui/src/views/protocol-view.ts`:
```typescript
import { LitElement, html, css, PropertyValues } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { marked } from 'marked';

interface ProtocolIndex {
  repoName: string;
  repoPath: string;
  indexPath: string;
  relativePath: string;
}

interface ProtocolEntry {
  file: string;
  summary: string;
  appliesTo: string;
  resolvedPath: string;
  section: string;
}

@customElement('trellis-protocol-view')
export class ProtocolView extends LitElement {

  @property() workspaceRoot = '';

  @state() private _repos: ProtocolIndex[] = [];
  @state() private _selectedRepo: ProtocolIndex | null = null;
  @state() private _indexes: string[] = [];
  @state() private _selectedIndex = '';
  @state() private _entries: ProtocolEntry[] = [];
  @state() private _selectedEntry: ProtocolEntry | null = null;
  @state() private _detailContent = '';
  @state() private _loading = false;

  static styles = css`
    :host {
      display: grid;
      grid-template-columns: 1fr 1fr;
      grid-template-rows: minmax(0, 1fr);
      gap: 1rem;
      padding: 1rem;
      height: 100%;
      box-sizing: border-box;
      font-family: system-ui, sans-serif;
      color: #eee;
      overflow: hidden;
    }
    .left, .right {
      overflow-y: auto;
      background: #1a1a2e;
      border-radius: 8px;
      padding: 1rem;
    }
    h3 { margin: 0 0 0.5rem 0; color: #8888cc; font-size: 0.9rem; }
    .repo-list, .index-list { margin-bottom: 1rem; }
    .repo-item, .index-item {
      padding: 0.4rem 0.6rem;
      cursor: pointer;
      border-radius: 4px;
      margin-bottom: 2px;
      font-size: 0.85rem;
    }
    .repo-item:hover, .index-item:hover { background: #2a2a4e; }
    .repo-item.selected, .index-item.selected { background: #3a3a6e; }
    .entry-row {
      display: flex;
      align-items: center;
      padding: 0.4rem 0.6rem;
      cursor: pointer;
      border-radius: 4px;
      margin-bottom: 2px;
      font-size: 0.85rem;
      gap: 0.5rem;
    }
    .entry-row:hover { background: #2a2a4e; }
    .entry-row.selected { background: #3a3a6e; }
    .entry-file { color: #aaf; flex-shrink: 0; }
    .entry-summary { flex: 1; color: #ccc; }
    .entry-applies { color: #888; font-size: 0.75rem; flex-shrink: 0; }
    .section-header {
      color: #666;
      font-size: 0.75rem;
      text-transform: uppercase;
      margin: 0.8rem 0 0.3rem 0;
      letter-spacing: 0.05em;
    }
    .remove-btn {
      background: none;
      border: 1px solid #633;
      color: #c66;
      border-radius: 3px;
      cursor: pointer;
      font-size: 0.7rem;
      padding: 2px 6px;
      flex-shrink: 0;
    }
    .remove-btn:hover { background: #633; color: #fcc; }
    .detail-content {
      font-size: 0.85rem;
      line-height: 1.5;
    }
    .detail-content h1, .detail-content h2, .detail-content h3 {
      color: #aaf;
    }
    .detail-content code { background: #2a2a4e; padding: 1px 4px; border-radius: 3px; }
    .detail-content pre {
      background: #12122a;
      padding: 0.8rem;
      border-radius: 4px;
      overflow-x: auto;
    }
    .badge {
      display: inline-block;
      padding: 2px 8px;
      border-radius: 3px;
      font-size: 0.7rem;
      margin-right: 4px;
      margin-bottom: 4px;
    }
    .badge-scope { background: #2a4a2a; color: #8c8; }
    .badge-severity { background: #4a2a2a; color: #c88; }
    .badge-type { background: #2a2a4a; color: #88c; }
    .badges { margin-bottom: 0.8rem; }
    .empty { color: #666; font-style: italic; font-size: 0.85rem; }
  `;

  connectedCallback() {
    super.connectedCallback();
    if (this.workspaceRoot) this._loadRepos();
  }

  updated(changed: PropertyValues) {
    if (changed.has('workspaceRoot') && this.workspaceRoot) {
      this._loadRepos();
    }
  }

  private async _loadRepos() {
    try {
      const res = await fetch(`/api/protocols/repos?root=${encodeURIComponent(this.workspaceRoot)}`);
      if (res.ok) this._repos = await res.json();
    } catch { /* unavailable */ }
  }

  private async _selectRepo(repo: ProtocolIndex) {
    this._selectedRepo = repo;
    this._selectedIndex = '';
    this._entries = [];
    this._selectedEntry = null;
    this._detailContent = '';
    try {
      const res = await fetch(`/api/protocols/indexes?repo=${encodeURIComponent(repo.repoPath)}`);
      if (res.ok) {
        this._indexes = await res.json();
        if (this._indexes.length === 1) {
          this._selectIndex(this._indexes[0]);
        }
      }
    } catch { /* unavailable */ }
  }

  private async _selectIndex(indexRelPath: string) {
    if (!this._selectedRepo) return;
    this._selectedIndex = indexRelPath;
    this._selectedEntry = null;
    this._detailContent = '';
    const fullPath = this._selectedRepo.repoPath + '/docs/protocols/' + indexRelPath;
    try {
      const res = await fetch(`/api/protocols/entries?index=${encodeURIComponent(fullPath)}`);
      if (res.ok) this._entries = await res.json();
    } catch { /* unavailable */ }
  }

  private async _selectEntry(entry: ProtocolEntry) {
    this._selectedEntry = entry;
    this._loading = true;
    try {
      const res = await fetch(
        `/api/artifacts/content?path=${encodeURIComponent(entry.resolvedPath)}&root=${encodeURIComponent(this.workspaceRoot)}`
      );
      if (res.ok) {
        this._detailContent = await res.text();
      }
    } catch { /* unavailable */ }
    this._loading = false;
  }

  private async _removeEntry(entry: ProtocolEntry, e: Event) {
    e.stopPropagation();
    if (!this._selectedRepo || !this._selectedIndex) return;
    const fullPath = this._selectedRepo.repoPath + '/docs/protocols/' + this._selectedIndex;
    try {
      await fetch(
        `/api/protocols/entries?index=${encodeURIComponent(fullPath)}&file=${encodeURIComponent(entry.file)}`,
        { method: 'DELETE' }
      );
      this._selectIndex(this._selectedIndex);
    } catch { /* unavailable */ }
  }

  private _parseFrontmatter(content: string): { meta: Record<string, string>; body: string } {
    const match = content.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);
    if (!match) return { meta: {}, body: content };
    const meta: Record<string, string> = {};
    for (const line of match[1].split('\n')) {
      const colonIdx = line.indexOf(':');
      if (colonIdx > 0) {
        meta[line.substring(0, colonIdx).trim()] = line.substring(colonIdx + 1).trim().replace(/^"(.*)"$/, '$1');
      }
    }
    return { meta, body: match[2] };
  }

  private _renderEntries() {
    if (this._entries.length === 0) {
      return html`<p class="empty">No protocol entries found.</p>`;
    }
    let currentSection = '';
    const items: unknown[] = [];
    for (const entry of this._entries) {
      if (entry.section && entry.section !== currentSection) {
        currentSection = entry.section;
        items.push(html`<div class="section-header">${currentSection}</div>`);
      }
      const selected = this._selectedEntry?.file === entry.file;
      items.push(html`
        <div class="entry-row ${selected ? 'selected' : ''}" @click=${() => this._selectEntry(entry)}>
          <span class="entry-file">${entry.file.replace('.md', '')}</span>
          <span class="entry-summary">${entry.summary}</span>
          <span class="entry-applies">${entry.appliesTo}</span>
          <button class="remove-btn" @click=${(e: Event) => this._removeEntry(entry, e)}>remove</button>
        </div>
      `);
    }
    return items;
  }

  private _renderDetail() {
    if (!this._selectedEntry) {
      return html`<p class="empty">Select a protocol entry to view its content.</p>`;
    }
    if (this._loading) {
      return html`<p class="empty">Loading...</p>`;
    }
    const { meta, body } = this._parseFrontmatter(this._detailContent);
    return html`
      <div class="badges">
        ${meta.scope ? html`<span class="badge badge-scope">${meta.scope}</span>` : ''}
        ${meta.severity ? html`<span class="badge badge-severity">${meta.severity}</span>` : ''}
        ${meta.type ? html`<span class="badge badge-type">${meta.type}</span>` : ''}
      </div>
      <div class="detail-content" .innerHTML=${marked.parse(body)}></div>
    `;
  }

  render() {
    return html`
      <div class="left">
        <h3>Repos with Protocols</h3>
        <div class="repo-list">
          ${this._repos.length === 0
            ? html`<p class="empty">No repos with protocols found.</p>`
            : this._repos.map(r => html`
              <div class="repo-item ${this._selectedRepo?.repoName === r.repoName ? 'selected' : ''}"
                   @click=${() => this._selectRepo(r)}>
                ${r.repoName}
              </div>
            `)}
        </div>

        ${this._selectedRepo && this._indexes.length > 1 ? html`
          <h3>Index Files</h3>
          <div class="index-list">
            ${this._indexes.map(idx => html`
              <div class="index-item ${this._selectedIndex === idx ? 'selected' : ''}"
                   @click=${() => this._selectIndex(idx)}>
                ${idx}
              </div>
            `)}
          </div>
        ` : ''}

        ${this._selectedIndex ? html`
          <h3>Protocol Entries</h3>
          ${this._renderEntries()}
        ` : ''}
      </div>

      <div class="right">
        <h3>Protocol Detail</h3>
        ${this._renderDetail()}
      </div>
    `;
  }
}
```

- [ ] **Step 2: Register panel in workbench.ts**

In `workbench.ts`, add to `PANELS`:
```typescript
protocols: { icon: '\u{1F4DC}', label: 'Protocols', tag: 'trellis-protocol-view' },
```

Add `'protocols'` to `DOCK_PANELS` after `'garden'`:
```typescript
const DOCK_PANELS = ['workspace', 'artifacts', 'garden', 'protocols', 'coordinator', 'memory'];
```

Add import at top:
```typescript
import '../views/protocol-view';
```

Add hash routing in `_parseHash()` for `#protocols`.

Add `workspaceRoot` context in `_applyContext()` (same as other panels).

- [ ] **Step 3: Build frontend**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: Build succeeds

- [ ] **Step 4: Manual test**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml quarkus:dev`
Navigate to the protocols panel. Verify:
- Repos with `docs/protocols/INDEX.md` appear in the list
- Clicking a repo shows its indexes
- Clicking an index shows parsed protocol entries
- Clicking an entry shows the rendered markdown detail with badges

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/src/views/protocol-view.ts \
       sidecar/src/main/webui/src/components/workbench.ts
git commit -m "feat: protocol management panel with browse and detail view"
```

---

### Task 5: Frontend — add from garden search

**Files:**
- Modify: `sidecar/src/main/webui/src/views/protocol-view.ts`

**Interfaces:**
- Consumes: `GET /api/garden/search?q=...` → garden entries
- Consumes: `POST /api/protocols/entries` → add entry
- Consumes: All state from Task 4

- [ ] **Step 1: Add garden search state and UI to protocol-view.ts**

Add state properties:
```typescript
@state() private _showAddSearch = false;
@state() private _addQuery = '';
@state() private _gardenResults: any[] = [];
@state() private _searchingGarden = false;
```

Add methods:
```typescript
private async _searchGarden() {
  if (!this._addQuery.trim()) return;
  this._searchingGarden = true;
  try {
    const res = await fetch(`/api/garden/search?q=${encodeURIComponent(this._addQuery)}`);
    if (res.ok) {
      const data = await res.json();
      this._gardenResults = data.results || [];
    }
  } catch { /* unavailable */ }
  this._searchingGarden = false;
}

private async _addFromGarden(gardenEntry: any) {
  if (!this._selectedRepo || !this._selectedIndex) return;
  const slug = gardenEntry.id.replace(/\//g, '-').replace('.md', '') + '.md';
  const fullPath = this._selectedRepo.repoPath + '/docs/protocols/' + this._selectedIndex;
  const body = {
    indexPath: fullPath,
    section: '',
    file: slug,
    summary: gardenEntry.title,
    appliesTo: gardenEntry.domain || 'All',
    gardenEntryId: gardenEntry.id,
    content: `---\nid: PP-${new Date().toISOString().slice(0,10).replace(/-/g,'')}-${Math.random().toString(36).slice(2,8)}\ntitle: "${gardenEntry.title}"\ntype: rule\nscope: universal\nseverity: guidance\napplies_to: "${gardenEntry.domain || 'All'}"\ngarden_ref: "${gardenEntry.id}"\n---\n\n${gardenEntry.body || gardenEntry.title}\n`
  };
  try {
    await fetch('/api/protocols/entries', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    });
    this._showAddSearch = false;
    this._gardenResults = [];
    this._addQuery = '';
    this._selectIndex(this._selectedIndex);
  } catch { /* unavailable */ }
}
```

Add to the left column render, after the entries list:
```typescript
${this._selectedIndex ? html`
  <button class="add-btn" @click=${() => { this._showAddSearch = !this._showAddSearch; }}>
    + Add from Garden
  </button>
  ${this._showAddSearch ? html`
    <div class="add-search">
      <input type="text" .value=${this._addQuery}
             @input=${(e: any) => { this._addQuery = e.target.value; }}
             @keydown=${(e: KeyboardEvent) => { if (e.key === 'Enter') this._searchGarden(); }}
             placeholder="Search garden entries..." />
      <button @click=${() => this._searchGarden()}>Search</button>
      ${this._searchingGarden ? html`<p class="empty">Searching...</p>` : ''}
      ${this._gardenResults.map(r => html`
        <div class="entry-row" @click=${() => this._addFromGarden(r)}>
          <span class="entry-file">${r.title}</span>
          <span class="entry-applies">${r.domain || ''}</span>
        </div>
      `)}
    </div>
  ` : ''}
` : ''}
```

Add CSS for the add button and search:
```css
.add-btn {
  margin-top: 0.5rem;
  background: #2a4a2a;
  color: #8c8;
  border: 1px solid #3a6a3a;
  border-radius: 4px;
  padding: 0.4rem 0.8rem;
  cursor: pointer;
  font-size: 0.8rem;
}
.add-btn:hover { background: #3a6a3a; }
.add-search { margin-top: 0.5rem; }
.add-search input {
  width: 70%;
  padding: 0.3rem 0.5rem;
  background: #12122a;
  border: 1px solid #333;
  color: #eee;
  border-radius: 4px;
  font-size: 0.8rem;
}
.add-search button {
  padding: 0.3rem 0.6rem;
  background: #2a2a4e;
  border: 1px solid #444;
  color: #aaf;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.8rem;
}
```

- [ ] **Step 2: Build frontend**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: Build succeeds

- [ ] **Step 3: Manual test**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml quarkus:dev`
Test the add flow:
- Click "Add from Garden" button
- Search for a garden entry
- Click a result to add it as a protocol
- Verify the entry appears in the list and the INDEX.md was updated

Test the remove flow:
- Click "remove" on an entry
- Verify it disappears from the list
- Verify the INDEX.md was updated (the .md file still exists on disk)

- [ ] **Step 4: Commit**

```bash
git add sidecar/src/main/webui/src/views/protocol-view.ts
git commit -m "feat: garden search integration for adding protocols"
```
