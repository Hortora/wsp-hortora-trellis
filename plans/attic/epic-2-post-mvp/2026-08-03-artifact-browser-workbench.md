# Artifact Browser and Workbench Shell Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #15 — Trellis: Artifact Browser (B6b, post-MVP)
**Issue group:** #15

**Goal:** Replace the hash router with a dock-bar workbench shell and add
an artifact browser panel that scans workspace/project directories and
renders markdown content.

**Architecture:** A Lit `trellis-workbench` component replaces `app.ts`'s
`route()` function with a vertical dock bar and lazy panel management.
Existing views become panels with zero rewrite. A new `trellis-artifact-panel`
uses the platform's `renderMarkdown()` from `channel-activity` for content
display. Backend `ArtifactScanner` walks known workspace/project paths and
`ArtifactResource` serves the list and content via REST.

**Tech Stack:** Java 21, Quarkus 3.x, Lit (TypeScript), marked + DOMPurify
(via `@casehubio/channel-activity`), esbuild

## Global Constraints

- Java 21 — records, pattern matching
- Package root: `io.hortora.trellis`
- New backend types in `io.hortora.trellis.artifact`
- IntelliJ MCP required for all source file operations
- Content endpoint max file size: 1 MB
- `quarkus.http.host=localhost` (sidecar is local-only)
- Pre-release: breaking API changes cost nothing
- Sidebar sort: alphabetical by name within type (not modifiedAt)
- `renderMarkdown()` from `@casehubio/channel-activity` — not `pages-runtime`

---

### Task 1: ArtifactEntry + ArtifactScanner

Backend scanning logic. Walks workspace and project directories for markdown
artifacts. Handles broken `proj/` symlink gracefully.

**Files:**
- Create: `src/main/java/io/hortora/trellis/artifact/ArtifactEntry.java`
- Create: `src/main/java/io/hortora/trellis/artifact/ArtifactScanner.java`
- Test: `src/test/java/io/hortora/trellis/artifact/ArtifactScannerTest.java`

**Interfaces:**
- Produces: `ArtifactEntry(String type, String name, String path, Instant modifiedAt)`
- Produces: `ArtifactScanner.scan(Path workspaceRoot): List<ArtifactEntry>`

- [ ] **Step 1: Write the ArtifactScanner test**

Create test file with temp filesystem:

```java
package io.hortora.trellis.artifact;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class ArtifactScannerTest {

    @TempDir Path tempDir;

    private ArtifactScanner scanner = new ArtifactScanner();

    private Path setupWorkspace() throws IOException {
        var ws = tempDir.resolve("workspace");
        Files.createDirectories(ws);
        return ws;
    }

    private Path setupProject(Path ws) throws IOException {
        var proj = tempDir.resolve("project");
        Files.createDirectories(proj);
        Files.createSymbolicLink(ws.resolve("proj"), proj);
        return proj;
    }

    @Test
    void findsSpecsInWorkspaceAndProject() throws IOException {
        var ws = setupWorkspace();
        var proj = setupProject(ws);
        Files.createDirectories(ws.resolve("specs"));
        Files.writeString(ws.resolve("specs/design-a.md"), "# Spec A");
        Files.createDirectories(proj.resolve("docs/specs"));
        Files.writeString(proj.resolve("docs/specs/design-b.md"), "# Spec B");

        var entries = scanner.scan(ws);
        var specs = entries.stream().filter(e -> e.type().equals("spec")).toList();
        assertEquals(2, specs.size());
    }

    @Test
    void findsSingleFileArtifacts() throws IOException {
        var ws = setupWorkspace();
        var proj = setupProject(ws);
        Files.writeString(ws.resolve("HANDOFF.md"), "# Handoff");
        Files.createDirectories(proj.resolve("docs"));
        Files.writeString(proj.resolve("docs/ARC42STORIES.MD"), "# Design");

        var entries = scanner.scan(ws);
        assertTrue(entries.stream().anyMatch(e -> e.type().equals("handover")));
        assertTrue(entries.stream().anyMatch(e -> e.type().equals("design")));
    }

    @Test
    void handlesMissingDirectoriesGracefully() throws IOException {
        var ws = setupWorkspace();
        setupProject(ws);
        var entries = scanner.scan(ws);
        assertTrue(entries.isEmpty());
    }

    @Test
    void handlesBrokenProjSymlink() throws IOException {
        var ws = setupWorkspace();
        Files.createSymbolicLink(ws.resolve("proj"), tempDir.resolve("nonexistent"));
        Files.createDirectories(ws.resolve("specs"));
        Files.writeString(ws.resolve("specs/design-a.md"), "# Spec A");

        var entries = scanner.scan(ws);
        assertEquals(1, entries.size());
        assertEquals("spec", entries.get(0).type());
    }

    @Test
    void skipsIndexFiles() throws IOException {
        var ws = setupWorkspace();
        setupProject(ws);
        Files.createDirectories(ws.resolve("specs"));
        Files.writeString(ws.resolve("specs/INDEX.md"), "# Index");
        Files.writeString(ws.resolve("specs/real-spec.md"), "# Real");

        var entries = scanner.scan(ws);
        assertEquals(1, entries.size());
        assertEquals("real-spec", entries.get(0).name());
    }

    @Test
    void sortsByTypeOrderThenAlphabetically() throws IOException {
        var ws = setupWorkspace();
        setupProject(ws);
        Files.createDirectories(ws.resolve("specs"));
        Files.createDirectories(ws.resolve("adr"));
        Files.writeString(ws.resolve("specs/z-spec.md"), "# Z");
        Files.writeString(ws.resolve("specs/a-spec.md"), "# A");
        Files.writeString(ws.resolve("adr/adr-001.md"), "# ADR");

        var entries = scanner.scan(ws);
        assertEquals("spec", entries.get(0).type());
        assertEquals("a-spec", entries.get(0).name());
        assertEquals("z-spec", entries.get(1).name());
        assertEquals("adr", entries.get(2).type());
    }

    @Test
    void handlesNoProjSymlink() throws IOException {
        var ws = setupWorkspace();
        Files.createDirectories(ws.resolve("blog"));
        Files.writeString(ws.resolve("blog/entry.md"), "# Blog");

        var entries = scanner.scan(ws);
        assertEquals(1, entries.size());
        assertEquals("blog", entries.get(0).type());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=ArtifactScannerTest`
Expected: FAIL — classes don't exist.

- [ ] **Step 3: Create ArtifactEntry record**

Use `ide_create_file`:
```java
package io.hortora.trellis.artifact;

import java.time.Instant;

public record ArtifactEntry(
        String type,
        String name,
        String path,
        Instant modifiedAt
) {}
```

- [ ] **Step 4: Create ArtifactScanner**

Use `ide_create_file`:
```java
package io.hortora.trellis.artifact;

import org.jboss.logging.Logger;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

public class ArtifactScanner {

    private static final Logger LOG = Logger.getLogger(ArtifactScanner.class);

    private static final List<ArtifactType> TYPES = List.of(
            new ArtifactType("spec", "specs", true, "docs/specs", true),
            new ArtifactType("adr", "adr", true, "docs/adr", true),
            new ArtifactType("plan", "plans", true, "docs/plans", true),
            new ArtifactType("blog", "blog", true, null, false),
            new ArtifactType("handover", null, false, null, false),
            new ArtifactType("design", null, false, null, false),
            new ArtifactType("journal", null, false, null, false)
    );

    private record ArtifactType(String type, String wsDir, boolean isDirectory,
                                 String projDir, boolean scanProject) {}

    public List<ArtifactEntry> scan(Path workspaceRoot) {
        Path projectRoot = resolveProjectRoot(workspaceRoot);
        var entries = new ArrayList<ArtifactEntry>();

        for (var at : TYPES) {
            if (at.isDirectory()) {
                if (at.wsDir() != null) {
                    scanDirectory(workspaceRoot.resolve(at.wsDir()), at.type(), entries);
                }
                if (at.scanProject() && projectRoot != null && at.projDir() != null) {
                    scanDirectory(projectRoot.resolve(at.projDir()), at.type(), entries);
                }
            } else {
                scanSingleFile(workspaceRoot, projectRoot, at.type(), entries);
            }
        }

        entries.sort((a, b) -> {
            int typeOrder = typeIndex(a.type()) - typeIndex(b.type());
            return typeOrder != 0 ? typeOrder : a.name().compareToIgnoreCase(b.name());
        });

        return entries;
    }

    private Path resolveProjectRoot(Path workspaceRoot) {
        var projLink = workspaceRoot.resolve("proj");
        if (!Files.isSymbolicLink(projLink)) {
            LOG.debugf("No proj/ symlink in workspace %s", workspaceRoot);
            return null;
        }
        try {
            var resolved = projLink.toRealPath();
            if (Files.isDirectory(resolved)) return resolved;
            LOG.warnf("proj/ symlink target does not exist: %s", resolved);
            return null;
        } catch (IOException e) {
            LOG.warnf("Failed to resolve proj/ symlink: %s", e.getMessage());
            return null;
        }
    }

    private void scanDirectory(Path dir, String type, List<ArtifactEntry> entries) {
        if (!Files.isDirectory(dir)) return;
        try (var stream = Files.walk(dir)) {
            stream.filter(Files::isRegularFile)
                  .filter(p -> p.toString().endsWith(".md"))
                  .filter(p -> !p.getFileName().toString().equals("INDEX.md"))
                  .forEach(p -> {
                      try {
                          var name = p.getFileName().toString().replaceFirst("\\.md$", "");
                          var modified = Files.getLastModifiedTime(p).toInstant();
                          entries.add(new ArtifactEntry(type, name, p.toAbsolutePath().toString(), modified));
                      } catch (IOException e) {
                          LOG.debugf("Failed to read %s: %s", p, e.getMessage());
                      }
                  });
        } catch (IOException e) {
            LOG.debugf("Failed to scan %s: %s", dir, e.getMessage());
        }
    }

    private void scanSingleFile(Path ws, Path proj, String type, List<ArtifactEntry> entries) {
        Path file = switch (type) {
            case "handover" -> ws.resolve("HANDOFF.md");
            case "design" -> proj != null ? proj.resolve("docs/ARC42STORIES.MD") : null;
            case "journal" -> ws.resolve("design/JOURNAL.md");
            default -> null;
        };
        if (file != null && Files.isRegularFile(file)) {
            try {
                var name = file.getFileName().toString().replaceFirst("\\.md$", "");
                var modified = Files.getLastModifiedTime(file).toInstant();
                entries.add(new ArtifactEntry(type, name, file.toAbsolutePath().toString(), modified));
            } catch (IOException e) {
                LOG.debugf("Failed to read %s: %s", file, e.getMessage());
            }
        }
    }

    private int typeIndex(String type) {
        for (int i = 0; i < TYPES.size(); i++) {
            if (TYPES.get(i).type().equals(type)) return i;
        }
        return TYPES.size();
    }
}
```

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=ArtifactScannerTest`
Expected: All 7 tests pass.

- [ ] **Step 6: Verify with IDE diagnostics**

Run: `ide_diagnostics` on `ArtifactScanner.java` and `ArtifactEntry.java`.

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/artifact/ArtifactEntry.java \
       sidecar/src/main/java/io/hortora/trellis/artifact/ArtifactScanner.java \
       sidecar/src/test/java/io/hortora/trellis/artifact/ArtifactScannerTest.java
git commit -m "feat(#15): add ArtifactEntry and ArtifactScanner

Refs #15"
```

---

### Task 2: ArtifactResource REST Endpoints

REST API for listing artifacts and serving content. Path validation and
size limit.

**Files:**
- Create: `src/main/java/io/hortora/trellis/artifact/ArtifactResource.java`
- Test: `src/test/java/io/hortora/trellis/artifact/ArtifactResourceTest.java`

**Interfaces:**
- Consumes: `ArtifactScanner.scan(Path): List<ArtifactEntry>`
- Produces: `GET /api/artifacts?root=...` → `List<ArtifactEntry>`
- Produces: `GET /api/artifacts/content?path=...&root=...` → `text/plain`

- [ ] **Step 1: Write the REST endpoint test**

Create test file:

```java
package io.hortora.trellis.artifact;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ArtifactResourceTest {

    static Path workspace;

    @TempDir
    static Path tempDir;

    @BeforeAll
    static void setup() throws IOException {
        workspace = tempDir.resolve("workspace");
        Files.createDirectories(workspace);
        var proj = tempDir.resolve("project");
        Files.createDirectories(proj.resolve("docs"));
        Files.createSymbolicLink(workspace.resolve("proj"), proj);
        Files.createDirectories(workspace.resolve("specs"));
        Files.writeString(workspace.resolve("specs/test-spec.md"), "# Test Spec\n\nContent here.");
        Files.writeString(proj.resolve("docs/ARC42STORIES.MD"), "# Design Doc");
    }

    @Test
    void listReturnsArtifacts() {
        given()
            .queryParam("root", workspace.toString())
        .when()
            .get("/api/artifacts")
        .then()
            .statusCode(200)
            .body("size()", greaterThan(0))
            .body("[0].type", notNullValue())
            .body("[0].name", notNullValue());
    }

    @Test
    void listReturns400WithoutRoot() {
        given()
        .when()
            .get("/api/artifacts")
        .then()
            .statusCode(400);
    }

    @Test
    void listReturns404ForMissingRoot() {
        given()
            .queryParam("root", "/nonexistent/path")
        .when()
            .get("/api/artifacts")
        .then()
            .statusCode(404);
    }

    @Test
    void contentServesMarkdown() {
        given()
            .queryParam("root", workspace.toString())
            .queryParam("path", workspace.resolve("specs/test-spec.md").toString())
        .when()
            .get("/api/artifacts/content")
        .then()
            .statusCode(200)
            .contentType("text/plain")
            .body(containsString("# Test Spec"));
    }

    @Test
    void contentReturns403ForPathTraversal() {
        given()
            .queryParam("root", workspace.toString())
            .queryParam("path", "/etc/passwd")
        .when()
            .get("/api/artifacts/content")
        .then()
            .statusCode(403);
    }

    @Test
    void contentReturns404ForMissingFile() {
        given()
            .queryParam("root", workspace.toString())
            .queryParam("path", workspace.resolve("specs/nonexistent.md").toString())
        .when()
            .get("/api/artifacts/content")
        .then()
            .statusCode(404);
    }

    @Test
    void contentReturnsEmptyListForEmptyWorkspace() throws IOException {
        var empty = tempDir.resolve("empty-ws");
        Files.createDirectories(empty);
        given()
            .queryParam("root", empty.toString())
        .when()
            .get("/api/artifacts")
        .then()
            .statusCode(200)
            .body("size()", is(0));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=ArtifactResourceTest`
Expected: FAIL — resource class doesn't exist.

- [ ] **Step 3: Create ArtifactResource**

Use `ide_create_file`:

```java
package io.hortora.trellis.artifact;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.io.IOException;
import java.nio.file.Files;
import java.util.Map;

@Path("/api/artifacts")
@Produces(MediaType.APPLICATION_JSON)
public class ArtifactResource {

    private static final long MAX_CONTENT_SIZE = 1_048_576;

    @Inject
    ArtifactScanner scanner;

    @GET
    public Response list(@QueryParam("root") String root) {
        if (root == null || root.isBlank()) {
            return Response.status(400).entity(Map.of("error", "root query parameter is required")).build();
        }
        var rootPath = java.nio.file.Path.of(root);
        if (!Files.isDirectory(rootPath)) {
            return Response.status(404).entity(Map.of("error", "root directory not found: " + root)).build();
        }
        return Response.ok(scanner.scan(rootPath)).build();
    }

    @GET
    @Path("/content")
    @Produces(MediaType.TEXT_PLAIN)
    public Response content(@QueryParam("root") String root, @QueryParam("path") String path) {
        if (root == null || root.isBlank() || path == null || path.isBlank()) {
            return Response.status(400).entity("root and path query parameters are required").build();
        }
        var rootPath = java.nio.file.Path.of(root);
        if (!Files.isDirectory(rootPath)) {
            return Response.status(404).entity("root directory not found").build();
        }

        var filePath = java.nio.file.Path.of(path);
        if (!Files.isRegularFile(filePath)) {
            return Response.status(404).entity("file not found").build();
        }

        try {
            var realPath = filePath.toRealPath();
            var realRoot = rootPath.toRealPath();
            var projLink = rootPath.resolve("proj");
            java.nio.file.Path realProj = null;
            if (Files.isSymbolicLink(projLink)) {
                try { realProj = projLink.toRealPath(); } catch (IOException ignored) {}
            }

            boolean withinWorkspace = realPath.startsWith(realRoot);
            boolean withinProject = realProj != null && realPath.startsWith(realProj);
            if (!withinWorkspace && !withinProject) {
                return Response.status(403).entity("path outside allowed roots").build();
            }

            if (Files.size(filePath) > MAX_CONTENT_SIZE) {
                return Response.status(413).entity("file exceeds 1 MB limit").build();
            }

            return Response.ok(Files.readString(filePath)).build();
        } catch (IOException e) {
            return Response.serverError().entity("failed to read file: " + e.getMessage()).build();
        }
    }
}
```

- [ ] **Step 4: Make ArtifactScanner a CDI bean**

Use `ide_edit_member` on `ArtifactScanner.java` to add `@ApplicationScoped`:

Add `import jakarta.enterprise.context.ApplicationScoped;` and annotate the class
with `@ApplicationScoped`.

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=ArtifactResourceTest`
Expected: All 7 tests pass.

- [ ] **Step 6: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: All tests pass (including existing tests).

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/artifact/ArtifactResource.java \
       sidecar/src/main/java/io/hortora/trellis/artifact/ArtifactScanner.java \
       sidecar/src/test/java/io/hortora/trellis/artifact/ArtifactResourceTest.java
git commit -m "feat(#15): add ArtifactResource REST endpoints

Refs #15"
```

---

### Task 3: Workbench Shell

Replace the hash router in `app.ts` with a `trellis-workbench` Lit component.
Dock bar on the left, lazy panel creation, hash sync.

**Files:**
- Create: `src/main/webui/src/components/workbench.ts`
- Modify: `src/main/webui/src/app.ts` — replace route() with workbench

**Interfaces:**
- Consumes: All existing view components (trellis-org-dashboard, trellis-slot-detail, etc.)
- Produces: `<trellis-workbench>` custom element with dock bar and panel management

- [ ] **Step 1: Create the workbench component**

Use `ide_create_file` for `src/main/webui/src/components/workbench.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import '../views/org-dashboard';
import '../views/slot-detail';
import '../views/epic-dashboard';
import '../views/garden-view';
import '../components/coordinator-panel';

interface PanelDef {
  icon: string;
  label: string;
  tag: string;
}

const PANELS: Record<string, PanelDef> = {
  workspace:   { icon: '\u{1F4C1}', label: 'Workspace',   tag: 'trellis-org-dashboard' },
  slot:        { icon: '\u{1F4CB}', label: 'Slot',         tag: 'trellis-slot-detail' },
  artifacts:   { icon: '\u{1F4C4}', label: 'Artifacts',    tag: 'trellis-artifact-panel' },
  garden:      { icon: '\u{1F33F}', label: 'Garden',       tag: 'trellis-garden-view' },
  coordinator: { icon: '\u{1F916}', label: 'Coordinator',  tag: 'trellis-coordinator-panel' },
  epic:        { icon: '⚡',    label: 'Epic',          tag: 'trellis-epic-dashboard' },
};

@customElement('trellis-workbench')
export class TrellisWorkbench extends LitElement {

  @property() workspaceRoot = '';

  @state() private _activePanel = 'workspace';
  @state() private _panelContext: Record<string, string> = {};

  private _panelCache = new Map<string, HTMLElement>();
  private _lastRoot = '';

  static override styles = css`
    :host {
      display: flex;
      height: 100%;
      font-family: system-ui, -apple-system, sans-serif;
    }

    .dock-bar {
      display: flex;
      flex-direction: column;
      width: 48px;
      background: #141414;
      border-right: 1px solid #333;
      padding: 4px 0;
      flex-shrink: 0;
    }

    .dock-btn {
      display: flex;
      align-items: center;
      justify-content: center;
      width: 48px;
      height: 44px;
      border: none;
      background: transparent;
      cursor: pointer;
      font-size: 18px;
      border-left: 2px solid transparent;
      transition: background 0.15s;
    }

    .dock-btn:hover { background: #222; }
    .dock-btn[data-active] {
      background: #1e1e1e;
      border-left-color: #3b82f6;
    }

    .panel-area {
      flex: 1;
      overflow: hidden;
      background: #1e1e1e;
    }

    .panel-area > * {
      display: block;
      width: 100%;
      height: 100%;
    }
  `;

  override connectedCallback() {
    super.connectedCallback();
    window.addEventListener('hashchange', this._onHashChange);
    this._parseHash();
  }

  override disconnectedCallback() {
    super.disconnectedCallback();
    window.removeEventListener('hashchange', this._onHashChange);
  }

  override updated(changed: Map<PropertyKey, unknown>) {
    if (changed.has('workspaceRoot') && this._lastRoot && this._lastRoot !== this.workspaceRoot) {
      this._panelCache.forEach(el => el.remove());
      this._panelCache.clear();
    }
    this._lastRoot = this.workspaceRoot;
  }

  private _onHashChange = () => { this._parseHash(); };

  private _parseHash() {
    const hash = location.hash;
    const ctx: Record<string, string> = {};

    const rootMatch = hash.match(/[?&]root=([^&]+)/);
    if (rootMatch) {
      this.workspaceRoot = decodeURIComponent(rootMatch[1]);
      ctx['root'] = this.workspaceRoot;
    }

    if (hash.match(/^#slot\/(\d+)/)) {
      const m = hash.match(/^#slot\/(\d+)/)!;
      this._activePanel = 'slot';
      ctx['slotNumber'] = m[1];
    } else if (hash.match(/^#epic\/([^/]+)\/([^/]+)\/(\d+)/)) {
      const m = hash.match(/^#epic\/([^/]+)\/([^/]+)\/(\d+)/)!;
      this._activePanel = 'epic';
      ctx['owner'] = m[1];
      ctx['repo'] = m[2];
      ctx['epicNumber'] = m[3];
    } else if (hash.match(/^#coordinator/)) {
      this._activePanel = 'coordinator';
      const epicParam = hash.match(/[?&]epic=([^&]+)/);
      if (epicParam) ctx['epicRef'] = decodeURIComponent(epicParam[1]);
    } else if (hash.match(/^#artifacts/)) {
      this._activePanel = 'artifacts';
    } else if (hash.match(/^#garden/)) {
      this._activePanel = 'garden';
    } else {
      this._activePanel = 'workspace';
    }

    this._panelContext = ctx;
  }

  private _activatePanel(id: string) {
    this._activePanel = id;
    const root = this.workspaceRoot ? `root=${encodeURIComponent(this.workspaceRoot)}` : '';
    location.hash = id === 'workspace' ? `#?${root}` : `#${id}?${root}`;
  }

  private _getOrCreatePanel(id: string): HTMLElement {
    let el = this._panelCache.get(id);
    if (!el) {
      const def = PANELS[id];
      if (!def) return document.createElement('div');
      el = document.createElement(def.tag);
      this._panelCache.set(id, el);
    }
    this._applyContext(el, id);
    return el;
  }

  private _applyContext(el: HTMLElement, panelId: string) {
    const ctx = this._panelContext;
    (el as any).workspaceRoot = this.workspaceRoot;
    if (panelId === 'slot' && ctx['slotNumber']) {
      (el as any).slotNumber = parseInt(ctx['slotNumber']);
    }
    if (panelId === 'epic') {
      (el as any).owner = ctx['owner'] ?? '';
      (el as any).repo = ctx['repo'] ?? '';
      (el as any).epicNumber = parseInt(ctx['epicNumber'] ?? '0');
    }
    if (panelId === 'coordinator' && ctx['epicRef']) {
      (el as any).epicRef = ctx['epicRef'];
    }
  }

  override render() {
    const panel = this._getOrCreatePanel(this._activePanel);
    return html`
      <div class="dock-bar">
        ${Object.entries(PANELS).map(([id, def]) => html`
          <button class="dock-btn"
                  title=${def.label}
                  ?data-active=${id === this._activePanel}
                  @click=${() => this._activatePanel(id)}>
            ${def.icon}
          </button>
        `)}
      </div>
      <div class="panel-area">${panel}</div>
    `;
  }
}
```

- [ ] **Step 2: Update app.ts to use the workbench**

Replace the contents of `app.ts`:

```typescript
import './components/workbench';
import './views/launcher';

const container = document.getElementById('app');

function route() {
  if (!container) return;
  const hash = location.hash;

  container.innerHTML = '';

  const rootMatch = hash.match(/[?&]root=([^&]+)/);
  if (!rootMatch && (hash === '' || hash === '#' || hash === '#launcher')) {
    const launcher = document.createElement('trellis-launcher');
    container.appendChild(launcher);
    return;
  }

  const workbench = document.createElement('trellis-workbench') as any;
  if (rootMatch) {
    workbench.workspaceRoot = decodeURIComponent(rootMatch[1]);
  }
  container.appendChild(workbench);
}

window.addEventListener('hashchange', route);
route();
```

- [ ] **Step 3: Verify with IDE diagnostics**

Run: `ide_diagnostics` on `src/main/webui/src/components/workbench.ts` and
`src/main/webui/src/app.ts`.

- [ ] **Step 4: Commit**

```bash
git add sidecar/src/main/webui/src/components/workbench.ts \
       sidecar/src/main/webui/src/app.ts
git commit -m "feat(#15): add workbench shell with dock bar, replacing hash router

Refs #15"
```

---

### Task 4: Artifact Panel

The artifact browser panel. Sidebar grouped by type, markdown rendering
via `renderMarkdown()` from `channel-activity`.

**Files:**
- Create: `src/main/webui/src/views/artifact-panel.ts`

**Interfaces:**
- Consumes: `GET /api/artifacts?root=...` → `ArtifactEntry[]`
- Consumes: `GET /api/artifacts/content?path=...&root=...` → `text/plain`
- Consumes: `renderMarkdown()` from `@casehubio/channel-activity`
- Produces: `<trellis-artifact-panel>` custom element

- [ ] **Step 1: Create the artifact panel component**

Use `ide_create_file` for `src/main/webui/src/views/artifact-panel.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { renderMarkdown } from '@casehubio/channel-activity';

interface ArtifactEntry {
  type: string;
  name: string;
  path: string;
  modifiedAt: string;
}

interface ArtifactGroup {
  type: string;
  label: string;
  entries: ArtifactEntry[];
  expanded: boolean;
}

const TYPE_LABELS: Record<string, string> = {
  spec: 'Specs',
  adr: 'ADRs',
  plan: 'Plans',
  blog: 'Blog',
  handover: 'Handovers',
  design: 'Design',
  journal: 'Journals',
};

const TYPE_ORDER = ['spec', 'adr', 'plan', 'blog', 'handover', 'design', 'journal'];

@customElement('trellis-artifact-panel')
export class TrellisArtifactPanel extends LitElement {

  @property() workspaceRoot = '';

  @state() private _groups: ArtifactGroup[] = [];
  @state() private _selectedPath: string | null = null;
  @state() private _content = '';
  @state() private _loading = false;
  @state() private _listLoading = false;
  @state() private _error: string | null = null;

  private _contentCache = new Map<string, string>();
  private _lastRoot = '';

  static override styles = css`
    :host { display: flex; height: 100%; font-family: system-ui, -apple-system, sans-serif; }

    .sidebar {
      width: 260px; min-width: 200px; max-width: 400px;
      background: #1a1a1a; border-right: 1px solid #333;
      overflow-y: auto; flex-shrink: 0; padding: 0.5rem 0;
    }

    .group-header {
      display: flex; align-items: center; gap: 0.4rem;
      padding: 0.4rem 1rem; cursor: pointer; user-select: none;
      font-size: 0.75rem; font-weight: 600; color: #888;
      text-transform: uppercase; letter-spacing: 0.05em;
    }
    .group-header:hover { color: #aaa; }
    .group-header .chevron { font-size: 0.6rem; transition: transform 0.15s; }
    .group-header .chevron.expanded { transform: rotate(90deg); }
    .group-header .count { color: #555; font-weight: 400; }

    .artifact-item {
      padding: 0.3rem 1rem 0.3rem 2rem;
      font-size: 0.8rem; color: #ccc; cursor: pointer;
      white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
    }
    .artifact-item:hover { background: #252525; }
    .artifact-item[data-selected] { background: #1e3a5f; color: #93c5fd; }

    .content-pane {
      flex: 1; overflow-y: auto; padding: 2rem;
      color: #ccc; line-height: 1.6;
    }

    .content-pane .placeholder { color: #555; font-size: 0.9rem; }
    .content-pane .error { color: #f87171; }

    .content-pane :is(h1, h2, h3, h4, h5, h6) { color: #eee; margin-top: 1.5em; }
    .content-pane h1 { font-size: 1.5rem; border-bottom: 1px solid #333; padding-bottom: 0.3rem; }
    .content-pane h2 { font-size: 1.25rem; }
    .content-pane code { background: #2a2a2a; padding: 0.15em 0.4em; border-radius: 3px; font-size: 0.9em; }
    .content-pane pre { background: #2a2a2a; padding: 1rem; border-radius: 6px; overflow-x: auto; }
    .content-pane pre code { background: none; padding: 0; }
    .content-pane table { border-collapse: collapse; margin: 1em 0; }
    .content-pane th, .content-pane td { border: 1px solid #444; padding: 0.4rem 0.8rem; }
    .content-pane th { background: #2a2a2a; }
    .content-pane blockquote { border-left: 3px solid #444; margin: 1em 0; padding: 0.5em 1em; color: #999; }
    .content-pane a { color: #60a5fa; }
    .content-pane img { max-width: 100%; }

    .spinner { color: #666; font-size: 0.85rem; }
  `;

  override updated(changed: Map<PropertyKey, unknown>) {
    if (changed.has('workspaceRoot') && this.workspaceRoot !== this._lastRoot) {
      this._lastRoot = this.workspaceRoot;
      this._contentCache.clear();
      this._selectedPath = null;
      this._content = '';
      this._loadArtifacts();
    }
  }

  override connectedCallback() {
    super.connectedCallback();
    if (this.workspaceRoot) this._loadArtifacts();
  }

  private async _loadArtifacts() {
    if (!this.workspaceRoot) return;
    this._listLoading = true;
    try {
      const res = await fetch(`/api/artifacts?root=${encodeURIComponent(this.workspaceRoot)}`);
      if (!res.ok) { this._error = `Failed to load artifacts: ${res.status}`; return; }
      const entries: ArtifactEntry[] = await res.json();
      this._groups = this._groupEntries(entries);
    } catch (e) {
      this._error = `Failed to load artifacts: ${e}`;
    } finally {
      this._listLoading = false;
    }
  }

  private _groupEntries(entries: ArtifactEntry[]): ArtifactGroup[] {
    const byType = new Map<string, ArtifactEntry[]>();
    for (const e of entries) {
      const list = byType.get(e.type) ?? [];
      list.push(e);
      byType.set(e.type, list);
    }
    return TYPE_ORDER
      .filter(t => byType.has(t))
      .map(t => ({
        type: t,
        label: TYPE_LABELS[t] ?? t,
        entries: byType.get(t)!,
        expanded: true,
      }));
  }

  private async _selectArtifact(path: string) {
    this._selectedPath = path;
    this._error = null;

    const cached = this._contentCache.get(path);
    if (cached) { this._content = cached; return; }

    this._loading = true;
    try {
      const res = await fetch(`/api/artifacts/content?path=${encodeURIComponent(path)}&root=${encodeURIComponent(this.workspaceRoot)}`);
      if (!res.ok) { this._error = `Failed to load content: ${res.status}`; this._loading = false; return; }
      const text = await res.text();
      this._contentCache.set(path, text);
      this._content = text;
    } catch (e) {
      this._error = `Failed to load content: ${e}`;
    } finally {
      this._loading = false;
    }
  }

  private _toggleGroup(type: string) {
    this._groups = this._groups.map(g =>
      g.type === type ? { ...g, expanded: !g.expanded } : g
    );
  }

  override render() {
    return html`
      <div class="sidebar">
        ${this._listLoading ? html`<div class="spinner" style="padding:1rem">Loading...</div>` :
          this._groups.length === 0 ? html`<div class="spinner" style="padding:1rem">No artifacts found</div>` :
          this._groups.map(g => html`
            <div class="group-header" @click=${() => this._toggleGroup(g.type)}>
              <span class="chevron ${g.expanded ? 'expanded' : ''}">&#9654;</span>
              ${g.label}
              <span class="count">(${g.entries.length})</span>
            </div>
            ${g.expanded ? g.entries.map(e => html`
              <div class="artifact-item"
                   ?data-selected=${e.path === this._selectedPath}
                   @click=${() => this._selectArtifact(e.path)}>
                ${e.name}
              </div>
            `) : nothing}
          `)}
      </div>
      <div class="content-pane">
        ${this._loading ? html`<div class="spinner">Loading...</div>` :
          this._error ? html`<div class="error">${this._error}</div>` :
          !this._selectedPath ? html`<div class="placeholder">Select an artifact to view</div>` :
          nothing}
        ${!this._loading && !this._error && this._selectedPath ? html`
          <div .innerHTML=${renderMarkdown(this._content)}></div>
        ` : nothing}
      </div>
    `;
  }
}
```

- [ ] **Step 2: Import the artifact panel in the workbench**

Add to the top of `src/main/webui/src/components/workbench.ts`:
```typescript
import '../views/artifact-panel';
```

- [ ] **Step 3: Verify with IDE diagnostics**

Run: `ide_diagnostics` on `artifact-panel.ts`.

- [ ] **Step 4: Commit**

```bash
git add sidecar/src/main/webui/src/views/artifact-panel.ts \
       sidecar/src/main/webui/src/components/workbench.ts
git commit -m "feat(#15): add artifact panel with sidebar navigation and markdown rendering

Refs #15"
```

---

### Task 5: Frontend Verification with Playwright

Manual browser test of the full workbench — dock bar navigation, artifact
browsing, markdown rendering, and existing view compatibility.

**Files:** None — verification only.

- [ ] **Step 1: Start dev server**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml quarkus:dev -Dquarkus.http.port=8081 -Ddebug=false`

Wait for `Listening on: http://localhost:8081`.

- [ ] **Step 2: Navigate to workbench**

Open `http://localhost:8081/#?root=<workspace-root>` where workspace root is
the trellis workspace path.

Verify:
- Dock bar renders on the left with 6 icon buttons
- Workspace panel is active by default
- Clicking each dock icon switches the panel

- [ ] **Step 3: Test artifact panel**

Click the 📄 artifacts dock icon.

Verify:
- Sidebar shows artifact types (Specs, ADRs, Plans, etc.) as collapsible groups
- Clicking a group header toggles expansion
- Clicking an artifact loads and renders its markdown content
- Markdown tables, headings, code blocks render correctly
- "Select an artifact to view" placeholder shows when nothing is selected

- [ ] **Step 4: Test existing views still work**

Click through workspace, slot, garden, epic dock icons.

Verify each existing view renders without errors.

- [ ] **Step 5: Test hash deep linking**

Navigate directly to `#artifacts?root=...` — verify artifact panel activates.
Navigate to `#garden` — verify garden panel activates.
Navigate to an unknown hash — verify workspace panel activates as fallback.

- [ ] **Step 6: Commit (if any fixes applied)**

```bash
git add -A
git commit -m "fix(#15): address issues found during browser verification

Refs #15"
```

---

## Task Dependencies

```
Task 1 (ArtifactScanner) ─────────────────────┐
                                                │
Task 2 (ArtifactResource) ← depends on 1 ─────┤
                                                │
Task 3 (Workbench Shell) ─────────────────────┐│
                                               ││
Task 4 (Artifact Panel) ← depends on 2, 3 ───┤│
                                               ││
Task 5 (Playwright) ← depends on 3, 4 ───────┘┘
```

Tasks 1 and 3 can run in parallel. Task 2 needs 1. Task 4 needs 2 and 3.
Task 5 needs everything.
