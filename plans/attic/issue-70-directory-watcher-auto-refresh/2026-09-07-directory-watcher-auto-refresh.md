# Directory Watcher Auto-Refresh Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #70 — Use directory-watcher to auto-refresh repo and slot display on file changes
**Issue group:** #70

**Goal:** Wire filesystem change detection to panel auto-refresh so workspace state updates are reflected in the UI without manual navigation.

**Architecture:** `FileWatcherService` already detects file changes and broadcasts SSE events, but no panel subscribes. This plan adds path-to-domain classification, debounced partial rescans, and frontend SSE subscriptions so each panel re-fetches only when its data domain changes.

**Tech Stack:** Java 21 (Quarkus 3.x), TypeScript (Lit, vitest), io.methvin directory-watcher 0.19.1, pages-push EventBroadcaster

## Global Constraints

- Java 21 records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Frontend: vitest for tests, esbuild build
- All timing values from `PreferencesService`, never hardcoded
- Use `ide_insert_member` / `ide_replace_member` for Java edits
- Use `ide_refactor_rename` for renames, never bash mv

---

## Batch 1: Backend core components

### Task 1: PathDomainClassifier

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/PathDomainClassifier.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/scanner/PathDomainClassifierTest.java`

**Interfaces:**
- Consumes: nothing (standalone pure function)
- Produces: `PathDomainClassifier.classify(Path changedPath, Path watchRoot) → Set<Domain>`, `PathDomainClassifier.Domain` enum (`REPOS`, `SLOTS`, `PROTOCOLS`, `LIFECYCLE`, `WORKLOG`)

- [ ] **Step 1: Write the failing tests**

```java
package io.hortora.trellis.scanner;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;

import static io.hortora.trellis.scanner.PathDomainClassifier.Domain.*;
import static org.junit.jupiter.api.Assertions.*;

class PathDomainClassifierTest {

    private final PathDomainClassifier classifier = new PathDomainClassifier();

    @TempDir Path root;

    @Test
    void gitHeadChangeClassifiesAsRepos() {
        var changed = root.resolve("my-repo/.git/HEAD");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(REPOS), result);
    }

    @Test
    void gitRefsChangeClassifiesAsRepos() {
        var changed = root.resolve("my-repo/.git/refs/heads/main");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(REPOS), result);
    }

    @Test
    void slotFileClassifiesAsSlots() {
        var changed = root.resolve("slots/1/.slot");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(SLOTS), result);
    }

    @Test
    void phaseCompleteClassifiesAsSlots() {
        var changed = root.resolve("slots/2/.phase-a-complete");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(SLOTS), result);
    }

    @Test
    void protocolIndexClassifiesAsProtocols() {
        var changed = root.resolve("my-repo/docs/protocols/INDEX.md");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(PROTOCOLS), result);
    }

    @Test
    void planFileClassifiesAsLifecycle() {
        var changed = root.resolve("slots/1/wsp-repo/design/.plan");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(LIFECYCLE), result);
    }

    @Test
    void pauseStackClassifiesAsLifecycle() {
        var changed = root.resolve("slots/1/wsp-repo/design/.pause-stack");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(LIFECYCLE), result);
    }

    @Test
    void epicFileClassifiesAsLifecycle() {
        var changed = root.resolve("slots/1/wsp-repo/design/.epic");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(LIFECYCLE), result);
    }

    @Test
    void worklogDbClassifiesAsWorklog() {
        var changed = root.resolve("worklog.db");
        var result = classifier.classify(changed, root);
        assertEquals(java.util.Set.of(WORKLOG), result);
    }

    @Test
    void unrelatedFileReturnsEmpty() {
        var changed = root.resolve("my-repo/src/Main.java");
        var result = classifier.classify(changed, root);
        assertTrue(result.isEmpty());
    }

    @Test
    void buildArtifactReturnsEmpty() {
        var changed = root.resolve("my-repo/target/classes/Foo.class");
        var result = classifier.classify(changed, root);
        assertTrue(result.isEmpty());
    }

    @Test
    void lifecycleFileInsideSlotMatchesBothDomains() {
        var changed = root.resolve("slots/1/.plan");
        var result = classifier.classify(changed, root);
        assertTrue(result.contains(LIFECYCLE));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=PathDomainClassifierTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `PathDomainClassifier` does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.hortora.trellis.scanner;

import java.nio.file.Path;
import java.util.EnumSet;
import java.util.Set;

public class PathDomainClassifier {

    public enum Domain { REPOS, SLOTS, PROTOCOLS, LIFECYCLE, WORKLOG }

    public Set<Domain> classify(Path changedPath, Path watchRoot) {
        Set<Domain> domains = EnumSet.noneOf(Domain.class);
        String relStr = watchRoot.relativize(changedPath).toString();
        String fileName = changedPath.getFileName().toString();

        if (relStr.contains("/.git/HEAD") || relStr.contains("/.git/refs/")) {
            domains.add(Domain.REPOS);
        }

        if (relStr.startsWith("slots/") &&
            (".slot".equals(fileName) || ".phase-a-complete".equals(fileName))) {
            domains.add(Domain.SLOTS);
        }

        if (relStr.endsWith("/docs/protocols/INDEX.md")) {
            domains.add(Domain.PROTOCOLS);
        }

        if (".plan".equals(fileName) || ".pause-stack".equals(fileName) ||
            ".epic".equals(fileName)) {
            domains.add(Domain.LIFECYCLE);
        }

        if ("worklog.db".equals(fileName)) {
            domains.add(Domain.WORKLOG);
        }

        return domains;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=PathDomainClassifierTest`
Expected: all 12 tests PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/scanner/PathDomainClassifier.java sidecar/src/test/java/io/hortora/trellis/scanner/PathDomainClassifierTest.java
git commit -m "feat(#70): add PathDomainClassifier for file-to-domain routing Refs #70"
```

---

### Task 2: FileWatcherDebouncer

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherDebouncer.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/scanner/FileWatcherDebouncerTest.java`

**Interfaces:**
- Consumes: nothing (standalone state machine)
- Produces: `FileWatcherDebouncer(ScheduledExecutorService, int idleWaitSeconds, int maxWaitSeconds, Consumer<Set<Path>>)`, `FileWatcherDebouncer.onFileChanged(Path)`

- [ ] **Step 1: Write the failing tests**

```java
package io.hortora.trellis.scanner;

import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.*;

class FileWatcherDebouncerTest {

    private final ScheduledExecutorService scheduler =
            Executors.newSingleThreadScheduledExecutor();

    @Test
    void singleEventDrainsAfterIdleWait() throws Exception {
        List<Set<Path>> drained = new ArrayList<>();
        var debouncer = new FileWatcherDebouncer(scheduler, 1, 10, drained::add);

        debouncer.onFileChanged(Path.of("/ws/repo/.git/HEAD"));

        Thread.sleep(1500);
        assertEquals(1, drained.size());
        assertEquals(Set.of(Path.of("/ws/repo/.git/HEAD")), drained.getFirst());
    }

    @Test
    void burstEventsCoalesceIntoSingleDrain() throws Exception {
        List<Set<Path>> drained = new ArrayList<>();
        var debouncer = new FileWatcherDebouncer(scheduler, 1, 10, drained::add);

        debouncer.onFileChanged(Path.of("/ws/a"));
        Thread.sleep(200);
        debouncer.onFileChanged(Path.of("/ws/b"));
        Thread.sleep(200);
        debouncer.onFileChanged(Path.of("/ws/c"));

        Thread.sleep(1500);
        assertEquals(1, drained.size());
        assertEquals(3, drained.getFirst().size());
    }

    @Test
    void maxCapForcesEarlyDrain() throws Exception {
        List<Set<Path>> drained = new ArrayList<>();
        var debouncer = new FileWatcherDebouncer(scheduler, 2, 1, drained::add);

        debouncer.onFileChanged(Path.of("/ws/first"));
        Thread.sleep(1200);

        assertFalse(drained.isEmpty(), "Max cap should have forced drain before idle wait");
    }

    @Test
    void noDrainWhenNoEvents() throws Exception {
        List<Set<Path>> drained = new ArrayList<>();
        new FileWatcherDebouncer(scheduler, 1, 10, drained::add);

        Thread.sleep(1500);
        assertTrue(drained.isEmpty());
    }

    @Test
    void resetsAfterDrain() throws Exception {
        List<Set<Path>> drained = new ArrayList<>();
        var debouncer = new FileWatcherDebouncer(scheduler, 1, 10, drained::add);

        debouncer.onFileChanged(Path.of("/ws/first"));
        Thread.sleep(1500);
        assertEquals(1, drained.size());

        debouncer.onFileChanged(Path.of("/ws/second"));
        Thread.sleep(1500);
        assertEquals(2, drained.size());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=FileWatcherDebouncerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `FileWatcherDebouncer` does not exist

- [ ] **Step 3: Write the implementation**

```java
package io.hortora.trellis.scanner;

import java.nio.file.Path;
import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;
import java.util.function.Consumer;

public class FileWatcherDebouncer {

    private final ScheduledExecutorService scheduler;
    private final int idleWaitSeconds;
    private final int maxWaitSeconds;
    private final Consumer<Set<Path>> drainCallback;

    private final Object lock = new Object();
    private Set<Path> accumulated = new HashSet<>();
    private ScheduledFuture<?> idleTimer;
    private long firstEventTime = -1;

    public FileWatcherDebouncer(ScheduledExecutorService scheduler,
                                int idleWaitSeconds,
                                int maxWaitSeconds,
                                Consumer<Set<Path>> drainCallback) {
        this.scheduler = scheduler;
        this.idleWaitSeconds = idleWaitSeconds;
        this.maxWaitSeconds = maxWaitSeconds;
        this.drainCallback = drainCallback;
    }

    public void onFileChanged(Path path) {
        synchronized (lock) {
            accumulated.add(path);
            long now = System.currentTimeMillis();

            if (firstEventTime == -1) {
                firstEventTime = now;
            }

            if (now - firstEventTime >= maxWaitSeconds * 1000L) {
                drain();
                return;
            }

            if (idleTimer != null) {
                idleTimer.cancel(false);
            }
            idleTimer = scheduler.schedule(
                    this::drainFromTimer, idleWaitSeconds, TimeUnit.SECONDS);
        }
    }

    private void drainFromTimer() {
        synchronized (lock) {
            drain();
        }
    }

    private void drain() {
        if (accumulated.isEmpty()) return;
        Set<Path> batch = accumulated;
        accumulated = new HashSet<>();
        firstEventTime = -1;
        if (idleTimer != null) {
            idleTimer.cancel(false);
            idleTimer = null;
        }
        drainCallback.accept(batch);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=FileWatcherDebouncerTest`
Expected: all 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherDebouncer.java sidecar/src/test/java/io/hortora/trellis/scanner/FileWatcherDebouncerTest.java
git commit -m "feat(#70): add FileWatcherDebouncer with idle-wait and max cap Refs #70"
```

---

## Batch 2: Backend integration

### Task 3: Wire FileWatcherService with debouncer, classifier, and preferences

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/config/PreferencesService.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherService.java`
- Modify: `sidecar/src/test/java/io/hortora/trellis/config/PreferencesServiceTest.java`

**Interfaces:**
- Consumes: `PathDomainClassifier.classify(Path, Path)`, `FileWatcherDebouncer(ScheduledExecutorService, int, int, Consumer<Set<Path>>)`, `PreferencesService.watcherDebounceIdleSeconds()`, `PreferencesService.watcherDebounceMaxWaitSeconds()`, `PreferencesService.watcherFallbackRescanSeconds()`
- Produces: `FileWatcherService` now broadcasts domain-specific topics on debounced file changes; `workspace:lifecycle` and `workspace:worklog` topics added

- [ ] **Step 1: Add watcher config to PreferencesService — write failing test**

Add to `PreferencesServiceTest.java`:

```java
@Test
void readsWatcherConfig() throws IOException {
    var prefs = tempDir.resolve("preferences.json");
    Files.writeString(prefs, """
        {
          "watcher": {
            "debounceIdleSeconds": 5,
            "debounceMaxWaitSeconds": 30,
            "fallbackRescanSeconds": 120
          }
        }
        """);
    var service = new PreferencesService(prefs);

    assertEquals(5, service.watcherDebounceIdleSeconds());
    assertEquals(30, service.watcherDebounceMaxWaitSeconds());
    assertEquals(120, service.watcherFallbackRescanSeconds());
}

@Test
void watcherConfigDefaultsWhenMissing() {
    var service = new PreferencesService(tempDir.resolve("nonexistent.json"));

    assertEquals(3, service.watcherDebounceIdleSeconds());
    assertEquals(20, service.watcherDebounceMaxWaitSeconds());
    assertEquals(60, service.watcherFallbackRescanSeconds());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=PreferencesServiceTest`
Expected: compilation error — methods do not exist

- [ ] **Step 3: Implement PreferencesService watcher accessors**

Add to `PreferencesService.java`:

```java
public int watcherDebounceIdleSeconds() {
    return getWatcherInt("debounceIdleSeconds", 3);
}

public int watcherDebounceMaxWaitSeconds() {
    return getWatcherInt("debounceMaxWaitSeconds", 20);
}

public int watcherFallbackRescanSeconds() {
    return getWatcherInt("fallbackRescanSeconds", 60);
}

private int getWatcherInt(String key, int defaultValue) {
    var watcher = root.getJsonObject("watcher");
    if (watcher == null) return defaultValue;
    return watcher.getInt(key, defaultValue);
}
```

- [ ] **Step 4: Run PreferencesServiceTest to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=PreferencesServiceTest`
Expected: all tests PASS

- [ ] **Step 5: Expose WorkspaceScanner partial scan methods**

Change visibility of `scanRepos`, `scanSlots`, and `scanWorkspaces` from `private` to package-private in `WorkspaceScanner.java`:

```java
// Was: private List<RepoInfo> scanRepos(Path root) {
List<RepoInfo> scanRepos(Path root) {

// Was: private List<SlotInfo> scanSlots(Path root) {
List<SlotInfo> scanSlots(Path root) {

// Was: private void scanWorkspaces(Path root, List<PauseEntry> pauses, List<EpicInfo> epics) {
void scanWorkspaces(Path root, List<PauseEntry> pauses, List<EpicInfo> epics) {
```

- [ ] **Step 6: Refactor FileWatcherService**

Replace the full `rescan()` approach with debounced domain-scoped partial rescans. The refactored `FileWatcherService`:

```java
package io.hortora.trellis.scanner;

import io.casehub.pages.push.EventBroadcaster;
import io.hortora.trellis.config.PreferencesService;
import io.methvin.watcher.DirectoryWatcher;
import jakarta.annotation.PreDestroy;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.nio.file.Path;
import java.util.List;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

@ApplicationScoped
public class FileWatcherService {

    private static final Logger LOG = Logger.getLogger(FileWatcherService.class);

    private static final String TOPIC_REPOS      = "workspace:repos";
    private static final String TOPIC_SLOTS      = "workspace:slots";
    private static final String TOPIC_PROTOCOLS  = "workspace:protocols";
    private static final String TOPIC_LIFECYCLE  = "workspace:lifecycle";
    private static final String TOPIC_WORKLOG    = "workspace:worklog";

    @Inject WorkspaceScanner scanner;
    @Inject EventBroadcaster broadcaster;
    @Inject PreferencesService preferences;

    private final PathDomainClassifier classifier = new PathDomainClassifier();
    private final ConcurrentHashMap<Path, WatchState> watches = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(
            r -> { Thread t = new Thread(r, "trellis-watcher"); t.setDaemon(true); return t; });

    private DirectoryWatcher worklogWatcher;

    public void watch(Path root) {
        watches.computeIfAbsent(root, r -> {
            var state = new WatchState(r, scanner.scan(r));
            state.debouncer = new FileWatcherDebouncer(
                    scheduler,
                    preferences.watcherDebounceIdleSeconds(),
                    preferences.watcherDebounceMaxWaitSeconds(),
                    paths -> onDebouncedChange(r, paths));
            startWatcher(state);
            startRescanFallback(state);
            return state;
        });
    }

    public void watchWorklog() {
        Path hortoraDir = Path.of(System.getProperty("user.home"), ".hortora");
        if (!java.nio.file.Files.isDirectory(hortoraDir)) return;
        try {
            var debouncer = new FileWatcherDebouncer(
                    scheduler,
                    preferences.watcherDebounceIdleSeconds(),
                    preferences.watcherDebounceMaxWaitSeconds(),
                    paths -> {
                        boolean worklogChanged = paths.stream()
                                .anyMatch(p -> "worklog.db".equals(p.getFileName().toString()));
                        if (worklogChanged) {
                            broadcaster.broadcast(TOPIC_WORKLOG, "changed");
                        }
                    });
            worklogWatcher = DirectoryWatcher.builder()
                    .path(hortoraDir)
                    .listener(event -> debouncer.onFileChanged(event.path()))
                    .build();
            worklogWatcher.watchAsync();
            LOG.info("Worklog watcher started");
        } catch (IOException e) {
            LOG.warnf(e, "Failed to start worklog watcher");
        }
    }

    public void stopWatching(Path root) {
        var state = watches.remove(root);
        if (state != null) { state.stop(); }
    }

    @PreDestroy
    void shutdown() {
        scheduler.shutdownNow();
        watches.values().forEach(WatchState::stop);
        watches.clear();
        if (worklogWatcher != null) {
            try { worklogWatcher.close(); } catch (IOException ignored) {}
        }
    }

    public WorkspaceModel currentModel(Path root) {
        var state = watches.get(root);
        return state != null ? state.model : null;
    }

    public java.util.List<WorkspaceModel> allModels() {
        return watches.values().stream()
                .map(ws -> ws.model)
                .filter(java.util.Objects::nonNull)
                .toList();
    }

    public void onWorkspaceChanged(@Observes @WorkspaceChanged Path root) {
        fullRescan(root);
    }

    private void onDebouncedChange(Path root, Set<Path> changedPaths) {
        var state = watches.get(root);
        if (state == null) return;

        var domains = new java.util.HashSet<PathDomainClassifier.Domain>();
        for (Path p : changedPaths) {
            domains.addAll(classifier.classify(p, root));
        }

        if (domains.isEmpty()) return;

        WorkspaceModel oldModel;
        synchronized (state) {
            oldModel = state.model;
        }

        if (domains.contains(PathDomainClassifier.Domain.REPOS)) {
            var newRepos = scanner.scanRepos(root);
            if (!oldModel.repos().equals(newRepos)) {
                synchronized (state) {
                    state.model = state.model.withRepos(newRepos);
                }
                broadcaster.broadcast(TOPIC_REPOS, newRepos);
            }
        }

        if (domains.contains(PathDomainClassifier.Domain.SLOTS)) {
            var newSlots = scanner.scanSlots(root);
            if (!oldModel.slots().equals(newSlots)) {
                synchronized (state) {
                    state.model = state.model.withSlots(newSlots);
                }
                broadcaster.broadcast(TOPIC_SLOTS, newSlots);
            }
        }

        if (domains.contains(PathDomainClassifier.Domain.PROTOCOLS)) {
            broadcaster.broadcast(TOPIC_PROTOCOLS, "changed");
        }

        if (domains.contains(PathDomainClassifier.Domain.LIFECYCLE)) {
            var pauses = new java.util.ArrayList<PauseEntry>();
            var epics = new java.util.ArrayList<EpicInfo>();
            scanner.scanWorkspaces(root, pauses, epics);
            synchronized (state) {
                state.model = state.model.withPausesAndEpics(
                        List.copyOf(pauses), List.copyOf(epics));
            }
            broadcaster.broadcast(TOPIC_LIFECYCLE, "changed");
        }
    }

    private void fullRescan(Path root) {
        var state = watches.get(root);
        if (state == null) return;
        var newModel = scanner.scan(root);
        WorkspaceModel oldModel;
        synchronized (state) {
            oldModel = state.model;
            state.model = newModel;
        }
        if (!oldModel.repos().equals(newModel.repos())) {
            broadcaster.broadcast(TOPIC_REPOS, newModel.repos());
        }
        if (!oldModel.slots().equals(newModel.slots())) {
            broadcaster.broadcast(TOPIC_SLOTS, newModel.slots());
        }
        if (!oldModel.repos().equals(newModel.repos())) {
            broadcaster.broadcast(TOPIC_PROTOCOLS, "changed");
        }
    }

    private void startWatcher(WatchState state) {
        Thread.ofVirtual().name("trellis-watcher-init-" + state.root.getFileName()).start(() -> {
            if (state.stopped) return;
            try {
                var watcher = DirectoryWatcher.builder()
                        .path(state.root)
                        .listener(event -> state.debouncer.onFileChanged(event.path()))
                        .build();
                state.directoryWatcher = watcher;
                watcher.watchAsync();
                LOG.infof("Directory watcher started for %s", state.root);
            } catch (IOException e) {
                LOG.warnf(e, "Failed to start directory watcher for %s — fallback rescan will continue", state.root);
            }
        });
    }

    private void startRescanFallback(WatchState state) {
        int interval = preferences.watcherFallbackRescanSeconds();
        scheduler.scheduleAtFixedRate(() -> {
            if (state.stopped) return;
            try {
                fullRescan(state.root);
            } catch (Exception e) {
                LOG.warnf(e, "Rescan failed for %s", state.root);
            }
        }, interval, interval, TimeUnit.SECONDS);
    }

    private static class WatchState {
        final Path root;
        volatile WorkspaceModel model;
        volatile DirectoryWatcher directoryWatcher;
        volatile boolean stopped;
        FileWatcherDebouncer debouncer;

        WatchState(Path root, WorkspaceModel initialModel) {
            this.root = root;
            this.model = initialModel;
        }

        void stop() {
            stopped = true;
            if (directoryWatcher != null) {
                try { directoryWatcher.close(); } catch (IOException ignored) {}
            }
        }
    }
}
```

- [ ] **Step 7: Add `withRepos`, `withSlots`, `withPausesAndEpics` to WorkspaceModel**

The `WorkspaceModel` record needs copy methods for partial updates. Add:

```java
public WorkspaceModel withRepos(List<RepoInfo> repos) {
    return new WorkspaceModel(root, scannedAt, repos, slots, pauses, epics);
}

public WorkspaceModel withSlots(List<SlotInfo> slots) {
    return new WorkspaceModel(root, scannedAt, repos, slots, pauses, epics);
}

public WorkspaceModel withPausesAndEpics(List<PauseEntry> pauses, List<EpicInfo> epics) {
    return new WorkspaceModel(root, scannedAt, repos, slots, pauses, epics);
}
```

- [ ] **Step 8: Start worklog watcher from ScannerResource or startup**

Add `watchWorklog()` call in `FileWatcherService.watch()` — call it once on first workspace watch:

```java
// At end of watch() method, after computeIfAbsent:
if (watches.size() == 1 && worklogWatcher == null) {
    watchWorklog();
}
```

- [ ] **Step 9: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: all tests PASS (including existing WorkspaceScannerTest and PreferencesServiceTest)

- [ ] **Step 10: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/config/PreferencesService.java sidecar/src/test/java/io/hortora/trellis/config/PreferencesServiceTest.java sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherService.java sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceModel.java
git commit -m "feat(#70): wire debounced domain-scoped file watching with preferences config Refs #70"
```

---

## Batch 3: Frontend auto-refresh

### Task 4: workspace-sse.ts shared subscription helper

**Files:**
- Create: `sidecar/src/main/webui/src/services/workspace-sse.ts`
- Test: `sidecar/src/main/webui/src/services/workspace-sse.test.ts`

**Interfaces:**
- Consumes: `/api/push?topics=...` SSE endpoint
- Produces: `subscribeWorkspace(topics: string[], onTopicMessage: (topic: string) => void): () => void`

- [ ] **Step 1: Write the failing tests**

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { subscribeWorkspace, _resetForTest } from './workspace-sse';

class MockEventSource {
  static instances: MockEventSource[] = [];
  url: string;
  onmessage: ((e: MessageEvent) => void) | null = null;
  onerror: (() => void) | null = null;
  readyState = 0;
  listeners = new Map<string, Function[]>();

  constructor(url: string) {
    this.url = url;
    MockEventSource.instances.push(this);
  }

  addEventListener(type: string, fn: Function) {
    if (!this.listeners.has(type)) this.listeners.set(type, []);
    this.listeners.get(type)!.push(fn);
  }

  removeEventListener(type: string, fn: Function) {
    const fns = this.listeners.get(type);
    if (fns) this.listeners.set(type, fns.filter(f => f !== fn));
  }

  close() { this.readyState = 2; }

  simulateMessage(data: unknown) {
    const event = { data: JSON.stringify(data) } as MessageEvent;
    for (const fn of this.listeners.get('message') ?? []) { fn(event); }
  }
}

beforeEach(() => {
  MockEventSource.instances = [];
  vi.stubGlobal('EventSource', MockEventSource);
  _resetForTest();
});

afterEach(() => {
  vi.unstubAllGlobals();
});

describe('subscribeWorkspace', () => {
  it('creates shared EventSource on first subscribe', () => {
    subscribeWorkspace(['workspace:repos'], () => {});
    expect(MockEventSource.instances).toHaveLength(1);
    expect(MockEventSource.instances[0].url).toContain('workspace:repos');
  });

  it('reuses EventSource for second subscriber', () => {
    subscribeWorkspace(['workspace:repos'], () => {});
    subscribeWorkspace(['workspace:slots'], () => {});
    expect(MockEventSource.instances).toHaveLength(1);
  });

  it('dispatches to correct subscriber by topic', () => {
    const reposCb = vi.fn();
    const slotsCb = vi.fn();
    subscribeWorkspace(['workspace:repos'], reposCb);
    subscribeWorkspace(['workspace:slots'], slotsCb);

    MockEventSource.instances[0].simulateMessage({ topic: 'workspace:repos' });
    expect(reposCb).toHaveBeenCalledWith('workspace:repos');
    expect(slotsCb).not.toHaveBeenCalled();
  });

  it('closes EventSource when last subscriber unsubscribes', () => {
    const unsub1 = subscribeWorkspace(['workspace:repos'], () => {});
    const unsub2 = subscribeWorkspace(['workspace:slots'], () => {});

    unsub1();
    expect(MockEventSource.instances[0].readyState).not.toBe(2);

    unsub2();
    expect(MockEventSource.instances[0].readyState).toBe(2);
  });

  it('ignores non-JSON messages', () => {
    const cb = vi.fn();
    subscribeWorkspace(['workspace:repos'], cb);

    const event = { data: 'not json' } as MessageEvent;
    for (const fn of MockEventSource.instances[0].listeners.get('message') ?? []) {
      fn(event);
    }
    expect(cb).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd sidecar/src/main/webui && yarn test --run src/services/workspace-sse.test.ts`
Expected: FAIL — module does not exist

- [ ] **Step 3: Write the implementation**

```typescript
const ALL_WORKSPACE_TOPICS = [
  'workspace:repos',
  'workspace:slots',
  'workspace:protocols',
  'workspace:lifecycle',
  'workspace:worklog',
];

type Unsubscribe = () => void;

interface Subscription {
  topics: Set<string>;
  callback: (topic: string) => void;
}

let eventSource: EventSource | null = null;
const subscriptions = new Set<Subscription>();

function ensureConnection(): void {
  if (eventSource) return;
  const topicParams = ALL_WORKSPACE_TOPICS
      .map(t => `topics=${encodeURIComponent(t)}`)
      .join('&');
  eventSource = new EventSource(`/api/push?${topicParams}`);
  eventSource.addEventListener('message', handleMessage);
}

function handleMessage(e: Event): void {
  try {
    const msg = JSON.parse((e as MessageEvent).data);
    if (msg.topic) {
      for (const sub of subscriptions) {
        if (sub.topics.has(msg.topic)) {
          sub.callback(msg.topic);
        }
      }
    }
  } catch { /* ignore non-JSON */ }
}

function closeIfEmpty(): void {
  if (subscriptions.size === 0 && eventSource) {
    eventSource.close();
    eventSource = null;
  }
}

export function subscribeWorkspace(
  topics: string[],
  onTopicMessage: (topic: string) => void,
): Unsubscribe {
  const sub: Subscription = {
    topics: new Set(topics),
    callback: onTopicMessage,
  };
  subscriptions.add(sub);
  ensureConnection();
  return () => {
    subscriptions.delete(sub);
    closeIfEmpty();
  };
}

export function _resetForTest(): void {
  if (eventSource) { eventSource.close(); eventSource = null; }
  subscriptions.clear();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd sidecar/src/main/webui && yarn test --run src/services/workspace-sse.test.ts`
Expected: all 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/src/services/workspace-sse.ts sidecar/src/main/webui/src/services/workspace-sse.test.ts
git commit -m "feat(#70): add workspace-sse shared subscription helper Refs #70"
```

---

### Task 5: Wire panels to subscribe and auto-refresh

**Files:**
- Modify: `sidecar/src/main/webui/src/views/org-dashboard.ts`
- Modify: `sidecar/src/main/webui/src/views/repo-detail.ts`
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts`
- Modify: `sidecar/src/main/webui/src/views/protocol-view.ts`
- Modify: `sidecar/src/main/webui/src/views/backlog-panel.ts`
- Modify: `sidecar/src/main/webui/src/views/intelligence-panel.ts`
- Modify: `sidecar/src/main/webui/src/views/blockers-panel.ts`

**Interfaces:**
- Consumes: `subscribeWorkspace(topics, callback)` from `workspace-sse.ts`
- Produces: panels auto-refresh when their subscribed SSE topics fire

Each panel gets the same pattern: import `subscribeWorkspace`, subscribe in `connectedCallback`, store the unsubscribe function, call it in `disconnectedCallback`, callback invokes existing fetch method.

- [ ] **Step 1: Wire org-dashboard**

Add import and subscription to `org-dashboard.ts`:

```typescript
// Add import at top
import { subscribeWorkspace } from '../services/workspace-sse.js';

// Add field
private _unsubWorkspace: (() => void) | null = null;

// In connectedCallback, after existing code:
this._unsubWorkspace = subscribeWorkspace(
  ['workspace:repos', 'workspace:slots', 'workspace:lifecycle'],
  () => { if (this._root) this._scan(); }
);

// Add disconnectedCallback (does not exist yet):
override disconnectedCallback() {
  super.disconnectedCallback();
  this._unsubWorkspace?.();
}
```

- [ ] **Step 2: Wire repo-detail**

Add to `repo-detail.ts` — already has `connectedCallback`/`disconnectedCallback` and EventSource for `agent:state`:

```typescript
// Add import at top
import { subscribeWorkspace } from '../services/workspace-sse.js';

// Add field
private _unsubWorkspace: (() => void) | null = null;

// In connectedCallback, after existing code:
this._unsubWorkspace = subscribeWorkspace(
  ['workspace:repos'],
  () => this._loadRepo()
);

// In disconnectedCallback, add:
this._unsubWorkspace?.();
```

- [ ] **Step 3: Wire slot-detail**

Add to `slot-detail.ts` — already has `connectedCallback`/`disconnectedCallback` and EventSource for `agent:state,agent:eviction`:

```typescript
// Add import at top
import { subscribeWorkspace } from '../services/workspace-sse.js';

// Add field
private _unsubWorkspace: (() => void) | null = null;

// In connectedCallback, after existing code:
this._unsubWorkspace = subscribeWorkspace(
  ['workspace:slots'],
  () => this._loadSlot()
);

// In disconnectedCallback, add:
this._unsubWorkspace?.();
```

- [ ] **Step 4: Wire protocol-view**

Add to `protocol-view.ts` — has `connectedCallback` but no `disconnectedCallback`:

```typescript
// Add import at top
import { subscribeWorkspace } from '../services/workspace-sse.js';

// Add field
private _unsubWorkspace: (() => void) | null = null;

// In connectedCallback, after existing code:
this._unsubWorkspace = subscribeWorkspace(
  ['workspace:protocols'],
  () => this._loadRepos()
);

// Add disconnectedCallback:
override disconnectedCallback() {
  super.disconnectedCallback();
  this._unsubWorkspace?.();
}
```

- [ ] **Step 5: Wire backlog-panel**

Add to `backlog-panel.ts` — has `connectedCallback`/`disconnectedCallback` with `_refreshInterval`:

```typescript
// Add import at top
import { subscribeWorkspace } from '../services/workspace-sse.js';

// Add field
private _unsubWorkspace: (() => void) | null = null;

// In connectedCallback, after existing code:
this._unsubWorkspace = subscribeWorkspace(
  ['workspace:lifecycle', 'workspace:worklog'],
  () => this._load()
);

// In disconnectedCallback, add:
this._unsubWorkspace?.();
```

- [ ] **Step 6: Wire intelligence-panel**

Add to `intelligence-panel.ts` — has `connectedCallback` but no `disconnectedCallback`:

```typescript
// Add import at top
import { subscribeWorkspace } from '../services/workspace-sse.js';

// Add field
private _unsubWorkspace: (() => void) | null = null;

// In connectedCallback, after existing code:
this._unsubWorkspace = subscribeWorkspace(
  ['workspace:lifecycle', 'workspace:worklog'],
  () => this._loadFindings()
);

// Add disconnectedCallback:
override disconnectedCallback() {
  super.disconnectedCallback();
  this._unsubWorkspace?.();
}
```

- [ ] **Step 7: Wire blockers-panel**

Add to `blockers-panel.ts` — has `connectedCallback`/`disconnectedCallback` with `_refreshInterval`:

```typescript
// Add import at top
import { subscribeWorkspace } from '../services/workspace-sse.js';

// Add field
private _unsubWorkspace: (() => void) | null = null;

// In connectedCallback, after existing code:
this._unsubWorkspace = subscribeWorkspace(
  ['workspace:worklog'],
  () => this._load()
);

// In disconnectedCallback, add:
this._unsubWorkspace?.();
```

- [ ] **Step 8: Build frontend**

Run: `cd sidecar/src/main/webui && yarn build`
Expected: build succeeds with no errors

- [ ] **Step 9: Run frontend tests**

Run: `cd sidecar/src/main/webui && yarn test --run`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```bash
git add sidecar/src/main/webui/src/views/org-dashboard.ts sidecar/src/main/webui/src/views/repo-detail.ts sidecar/src/main/webui/src/views/slot-detail.ts sidecar/src/main/webui/src/views/protocol-view.ts sidecar/src/main/webui/src/views/backlog-panel.ts sidecar/src/main/webui/src/views/intelligence-panel.ts sidecar/src/main/webui/src/views/blockers-panel.ts
git commit -m "feat(#70): wire all panels to auto-refresh via workspace SSE Refs #70"
```

---

## References

- [2026-09-07-directory-watcher-auto-refresh-design.md] — design spec this plan implements
- [decisions.md] — D1-D6 design decisions
- [FileWatcherService.java] — existing watcher infrastructure
- [WorkspaceScanner.java] — existing scanner with per-domain private methods
- [PushResource.java] — SSE endpoint
- [PreferencesService.java] — runtime preferences pattern
- [sse-manager.ts] — pages-data shared EventSource management (reference pattern)
- [org-dashboard.ts] — dashboard panel (primary consumer)
- [backlog-panel.test.ts] — vitest test pattern reference
- [PreferencesServiceTest.java] — Java test pattern reference
- [GitHub #70] — problem statement and incremental refresh requirement
