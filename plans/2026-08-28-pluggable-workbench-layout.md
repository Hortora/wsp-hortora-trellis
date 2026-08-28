# Pluggable Workbench Layout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #49 — Workbench layout: pluggable content area with optional dock bars
**Issue group:** #49

**Goal:** Migrate the trellis workbench from a monolithic Lit component with manual dock-bar rendering to a two-layer architecture: pages `dockWorkbench()` for dock bars, pages `Container` for a pluggable content area.

**Architecture:** The workbench uses `dockWorkbench()` + `renderComponent()` for dock bar rendering (no `loadSite()`). A `Container` from pages-runtime's frame-sandbox manages the content area with switchable layout modes (content, tabbed, split, free, accordion). Layout persists via `createRestLayoutStore()` backed by a new `/api/layouts/{key}` endpoint. SSE navigation dispatches `pages-dock-toggle` events.

**Tech Stack:** Lit 3, pages-runtime (frame-sandbox, zone-layout-engine), pages-ui (dsl/builders), pages-component (renderer), Quarkus 3.x (JAX-RS), vitest

## Global Constraints

- Java 21, Quarkus 3.x
- Package root: `io.hortora.trellis`
- Pages packages consumed via portal: resolutions in `.casehub-packages/`
- Frontend theme: `casehub-dark` via `applyTheme()` + `pages-density-compact`
- No `loadSite()` — trellis uses `dockWorkbench()` + `renderComponent()` directly
- All commits reference #49: `Refs #49`

---

## Batch 1: Backend — key-based layout persistence

### Task 1: Replace layout endpoints with `/api/layouts/{key}`

Replace `WorkspaceLayoutStore` + `WorkspaceLayoutResource` (which serve `/api/workspace/layout` and `/api/workspace/groups`) with a key-based `LayoutStore` + `LayoutResource` serving `/api/layouts/{key}`.

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/layout/WorkspaceLayoutStore.java` → rename to `LayoutStore`, change to key-based storage
- Modify: `sidecar/src/main/java/io/hortora/trellis/layout/WorkspaceLayoutResource.java` → rename to `LayoutResource`, change endpoints
- Modify: `sidecar/src/test/java/io/hortora/trellis/layout/WorkspaceLayoutResourceTest.java` → rename to `LayoutResourceTest`, update tests
- Rename: `WorkspaceLayoutStore` → `LayoutStore` (use `ide_refactor_rename`)
- Rename: `WorkspaceLayoutResource` → `LayoutResource` (use `ide_refactor_rename`)
- Rename: `WorkspaceLayoutResourceTest` → `LayoutResourceTest` (use `ide_refactor_rename`)

**Interfaces:**
- Produces: `GET /api/layouts/{key}` → returns JSON string or 404, `PUT /api/layouts/{key}` → stores JSON string (64KB limit), `DELETE /api/layouts/{key}` → removes stored layout. File storage under `.trellis/layouts/{key}.json`.

- [ ] **Step 1: Write failing tests for the new key-based endpoints**

Replace the existing 6 tests with tests for the new API:

```java
@QuarkusTest
class LayoutResourceTest {

    @TestHTTPResource
    URL baseUrl;

    Path tempDir;

    @BeforeEach
    void setUp() throws Exception {
        tempDir = Files.createTempDirectory("trellis-layout-test");
    }

    @AfterEach
    void tearDown() throws Exception {
        // clean up tempDir
    }

    @Test
    void getReturns404WhenNotFound() {
        given()
            .queryParam("root", tempDir.toString())
            .when().get("/api/layouts/workbench")
            .then().statusCode(404);
    }

    @Test
    void putThenGetRoundTrips() {
        String body = "{\"docks\":{\"workspace\":true}}";
        given()
            .queryParam("root", tempDir.toString())
            .contentType("application/json")
            .body(body)
            .when().put("/api/layouts/workbench")
            .then().statusCode(204);

        given()
            .queryParam("root", tempDir.toString())
            .when().get("/api/layouts/workbench")
            .then().statusCode(200)
            .body(equalTo(body));
    }

    @Test
    void deleteRemovesLayout() {
        String body = "{\"docks\":{}}";
        given()
            .queryParam("root", tempDir.toString())
            .contentType("application/json")
            .body(body)
            .when().put("/api/layouts/workbench")
            .then().statusCode(204);

        given()
            .queryParam("root", tempDir.toString())
            .when().delete("/api/layouts/workbench")
            .then().statusCode(204);

        given()
            .queryParam("root", tempDir.toString())
            .when().get("/api/layouts/workbench")
            .then().statusCode(404);
    }

    @Test
    void multipleKeysAreIndependent() {
        String body1 = "{\"key\":\"workbench\"}";
        String body2 = "{\"key\":\"sidebar\"}";

        given().queryParam("root", tempDir.toString())
            .contentType("application/json").body(body1)
            .when().put("/api/layouts/workbench").then().statusCode(204);

        given().queryParam("root", tempDir.toString())
            .contentType("application/json").body(body2)
            .when().put("/api/layouts/sidebar").then().statusCode(204);

        given().queryParam("root", tempDir.toString())
            .when().get("/api/layouts/workbench")
            .then().statusCode(200).body(equalTo(body1));

        given().queryParam("root", tempDir.toString())
            .when().get("/api/layouts/sidebar")
            .then().statusCode(200).body(equalTo(body2));
    }

    @Test
    void missingRootReturns400() {
        given()
            .when().get("/api/layouts/workbench")
            .then().statusCode(400);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=LayoutResourceTest -pl .`
Expected: compilation failure (class doesn't exist yet)

- [ ] **Step 3: Rename classes via IntelliJ refactoring**

Use `ide_refactor_rename` for each class:
- `WorkspaceLayoutStore` → `LayoutStore`
- `WorkspaceLayoutResource` → `LayoutResource`
- `WorkspaceLayoutResourceTest` → `LayoutResourceTest`

- [ ] **Step 4: Implement LayoutStore with key-based storage**

Replace the body of `LayoutStore` (formerly `WorkspaceLayoutStore`):

```java
@ApplicationScoped
public class LayoutStore {

    private static final String DIR = ".trellis/layouts";

    public String load(Path workspaceRoot, String key) throws IOException {
        var file = workspaceRoot.resolve(DIR).resolve(key + ".json");
        if (!Files.exists(file)) return null;
        return Files.readString(file);
    }

    public void save(Path workspaceRoot, String key, String json) throws IOException {
        var dir = workspaceRoot.resolve(DIR);
        Files.createDirectories(dir);
        Files.writeString(dir.resolve(key + ".json"), json);
    }

    public void delete(Path workspaceRoot, String key) throws IOException {
        var file = workspaceRoot.resolve(DIR).resolve(key + ".json");
        Files.deleteIfExists(file);
    }
}
```

- [ ] **Step 5: Implement LayoutResource with `/api/layouts/{key}`**

Replace the body of `LayoutResource` (formerly `WorkspaceLayoutResource`):

```java
@Path("/api/layouts")
@ApplicationScoped
public class LayoutResource {

    @Inject LayoutStore store;

    @GET @Path("/{key}")
    @Produces(MediaType.APPLICATION_JSON)
    public Response get(@PathParam("key") String key, @QueryParam("root") String root) {
        if (root == null || root.isBlank()) return Response.status(400).build();
        try {
            String json = store.load(Path.of(root), key);
            if (json == null) return Response.status(404).build();
            return Response.ok(json).build();
        } catch (IOException e) {
            return Response.serverError().build();
        }
    }

    @PUT @Path("/{key}")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response put(@PathParam("key") String key, @QueryParam("root") String root, String body) {
        if (root == null || root.isBlank()) return Response.status(400).build();
        try {
            store.save(Path.of(root), key, body);
            return Response.noContent().build();
        } catch (IOException e) {
            return Response.serverError().build();
        }
    }

    @DELETE @Path("/{key}")
    public Response delete(@PathParam("key") String key, @QueryParam("root") String root) {
        if (root == null || root.isBlank()) return Response.status(400).build();
        try {
            store.delete(Path.of(root), key);
            return Response.noContent().build();
        } catch (IOException e) {
            return Response.serverError().build();
        }
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=LayoutResourceTest -pl .`
Expected: all 5 tests PASS

- [ ] **Step 7: Verify no references to old endpoints remain**

Use `ide_search_text` to search for `workspace/layout` and `workspace/groups` across `.java` and `.ts` files. If any remain in frontend code, note them — they'll be removed in Batch 2.

- [ ] **Step 8: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/layout/ sidecar/src/test/java/io/hortora/trellis/layout/
git commit -m "feat(#49): replace /api/workspace/layout with /api/layouts/{key}

Key-based layout persistence using LayoutStore + LayoutResource.
Drops /api/workspace/layout and /api/workspace/groups endpoints.
Storage under .trellis/layouts/{key}.json.

Refs #49"
```

---

## Batch 2: Frontend — Workbench rewrite with pages APIs

### Task 2: Panel config and ContentFactory

Extract panel configuration from the hardcoded `PANELS`/`DOCK_PANELS` into pages-compatible structures. Create the `ContentFactory` function. Set up `registerPanel()` calls.

**Files:**
- Create: `sidecar/src/main/webui/src/components/workbench-panels.ts`
- Create: `sidecar/src/main/webui/src/components/workbench-panels.test.ts`

**Interfaces:**
- Produces: `WORKBENCH_CONFIG: DockWorkbenchConfig`, `PANEL_TAGS: Record<string, string>`, `createPanelFactory(workspaceRoot: string): ContentFactory`, `registerAllPanels(): void`

- [ ] **Step 1: Write failing test for ContentFactory**

```typescript
// workbench-panels.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { createPanelFactory, PANEL_TAGS, registerAllPanels } from './workbench-panels.js';

describe('workbench-panels', () => {
  beforeEach(() => {
    registerAllPanels();
  });

  it('PANEL_TAGS maps all 8 dock panel keys to tag names', () => {
    const keys = Object.keys(PANEL_TAGS);
    expect(keys).toContain('workspace');
    expect(keys).toContain('dashboard');
    expect(keys).toContain('backlog');
    expect(keys).toContain('artifacts');
    expect(keys).toContain('garden');
    expect(keys).toContain('protocols');
    expect(keys).toContain('coordinator');
    expect(keys).toContain('memory');
    expect(keys.length).toBe(11); // 8 dock + 3 sub-views
  });

  it('ContentFactory creates element with correct tag', () => {
    const factory = createPanelFactory('/test/root');
    const entry = { key: 'dashboard', label: 'Dashboard' };
    const { element } = factory(entry as any);
    expect(element.tagName.toLowerCase()).toBe('trellis-org-dashboard');
  });

  it('ContentFactory sets workspaceRoot on element', () => {
    const factory = createPanelFactory('/test/root');
    const entry = { key: 'dashboard', label: 'Dashboard' };
    const { element } = factory(entry as any);
    expect((element as any).workspaceRoot).toBe('/test/root');
  });

  it('ContentFactory dispose removes element', () => {
    const factory = createPanelFactory('/test/root');
    const entry = { key: 'dashboard', label: 'Dashboard' };
    const { element, dispose } = factory(entry as any);
    const parent = document.createElement('div');
    parent.appendChild(element);
    expect(parent.children.length).toBe(1);
    dispose!();
    expect(parent.children.length).toBe(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd sidecar/src/main/webui && yarn test --run src/components/workbench-panels.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement workbench-panels.ts**

```typescript
// workbench-panels.ts
import { registerPanel } from '@casehubio/pages-runtime';
import type { ContentFactory, Entry } from '@casehubio/pages-runtime/dist/frame-sandbox';
import type { DockWorkbenchConfig } from '@casehubio/pages-ui/dist/dsl/builders.js';
import { dockWorkbench, hostPanel } from '@casehubio/pages-ui/dist/dsl/builders.js';

import '../views/org-dashboard.js';
import '../views/slot-detail.js';
import '../views/epic-dashboard.js';
import '../views/garden-view.js';
import '../views/artifact-panel.js';
import '../views/repo-detail.js';
import '../components/coordinator-panel.js';
import '../components/workspace-view.js';
import '../views/protocol-view.js';
import '../views/backlog-panel.js';

export const PANEL_TAGS: Record<string, string> = {
  workspace:   'trellis-workspace-view',
  dashboard:   'trellis-org-dashboard',
  slot:        'trellis-slot-detail',
  artifacts:   'trellis-artifact-panel',
  garden:      'trellis-garden-view',
  protocols:   'trellis-protocol-view',
  coordinator: 'trellis-coordinator-panel',
  memory:      'trellis-memory-panel',
  backlog:     'trellis-backlog-panel',
  epic:        'trellis-epic-dashboard',
  repo:        'trellis-repo-detail',
};

export function registerAllPanels(): void {
  for (const [key, tag] of Object.entries(PANEL_TAGS)) {
    registerPanel(key, tag);
  }
}

export function createPanelFactory(workspaceRoot: string): ContentFactory {
  return (entry: Entry) => {
    const tag = PANEL_TAGS[entry.key];
    if (!tag) throw new Error(`Unknown panel: ${entry.key}`);
    const el = document.createElement(tag);
    (el as any).workspaceRoot = workspaceRoot;
    return { element: el, dispose: () => el.remove() };
  };
}

const DOCK_PANELS: DockWorkbenchConfig['left'] = [
  { key: 'workspace',   label: 'Workspace',   icon: '\u{2B1A}', content: hostPanel('workspace'),   fixed: true, defaultOpen: true },
  { key: 'dashboard',   label: 'Dashboard',   icon: '\u{1F4C1}', content: hostPanel('dashboard') },
  { key: 'backlog',     label: 'Backlog',      icon: '\u{1F4CB}', content: hostPanel('backlog') },
  { key: 'artifacts',   label: 'Artifacts',    icon: '\u{1F4C4}', content: hostPanel('artifacts') },
  { key: 'garden',      label: 'Garden',       icon: '\u{1F33F}', content: hostPanel('garden') },
  { key: 'protocols',   label: 'Protocols',    icon: '\u{1F4DC}', content: hostPanel('protocols') },
  { key: 'coordinator', label: 'Coordinator',  icon: '\u{1F916}', content: hostPanel('coordinator') },
  { key: 'memory',      label: 'Memory',       icon: '\u{1F4CA}', content: hostPanel('memory') },
];

export { DOCK_PANELS };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd sidecar/src/main/webui && yarn test --run src/components/workbench-panels.test.ts`
Expected: all 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/src/components/workbench-panels.ts sidecar/src/main/webui/src/components/workbench-panels.test.ts
git commit -m "feat(#49): panel config, ContentFactory, and registerPanel setup

Extracts panel definitions from workbench into workbench-panels.ts.
DockWorkbenchConfig-compatible panel declarations.
ContentFactory creates Lit elements with workspaceRoot binding.

Refs #49"
```

---

### Task 3: Rewrite workbench with pages dock bars and Container

Replace the monolithic `trellis-workbench` Lit component with a two-layer architecture: `dockWorkbench()` + `renderComponent()` for dock bars, `createContainer()` for the content area.

**Files:**
- Modify: `sidecar/src/main/webui/src/components/workbench.ts` — full rewrite

**Interfaces:**
- Consumes: `DOCK_PANELS` (from workbench-panels.ts), `createPanelFactory(root: string): ContentFactory`, `registerAllPanels(): void`, `PANEL_TAGS: Record<string, string>`
- Produces: `TrellisWorkbench` Lit component with `workspaceRoot` property

- [ ] **Step 1: Write the rewritten workbench component**

Replace the entire body of `workbench.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import { renderComponent } from '@casehubio/pages-component';
import { dockWorkbench, hostPanel } from '@casehubio/pages-ui/dist/dsl/builders.js';
import { createRestLayoutStore } from '@casehubio/pages-runtime';
import { createContainer, createContainerToolbar } from '@casehubio/pages-runtime/dist/frame-sandbox';
import type { Container } from '@casehubio/pages-runtime/dist/frame-sandbox';
import { DOCK_PANELS, PANEL_TAGS, registerAllPanels, createPanelFactory } from './workbench-panels.js';

registerAllPanels();

@customElement('trellis-workbench')
export class TrellisWorkbench extends LitElement {

  static override shadowRootOptions = { ...LitElement.shadowRootOptions, delegatesFocus: true };

  @property() workspaceRoot = '';

  private _container: Container | null = null;
  private _rendered = false;
  private _lastRoot = '';
  private _heartbeatInterval: ReturnType<typeof setInterval> | null = null;
  private _eventSource: EventSource | null = null;
  private _pendingCorrelationId: string | null = null;

  static override styles = css`
    :host {
      display: block;
      height: 100%;
      width: 100%;
    }
    .workbench-root {
      height: 100%;
      width: 100%;
    }
  `;

  override connectedCallback() {
    super.connectedCallback();
    this._startHeartbeat();
    this._connectSSE();
  }

  override disconnectedCallback() {
    super.disconnectedCallback();
    this._stopHeartbeat();
    this._disconnectSSE();
    this._container?.dispose();
    this._container = null;
  }

  override updated(changed: Map<PropertyKey, unknown>) {
    if (changed.has('workspaceRoot') && this._lastRoot !== this.workspaceRoot) {
      this._lastRoot = this.workspaceRoot;
      this._rendered = false;
      this._container?.dispose();
      this._container = null;
    }
    if (!this._rendered && this.workspaceRoot) {
      this._renderWorkbench();
      this._rendered = true;
    }
  }

  private _renderWorkbench() {
    const root = this.shadowRoot!.querySelector('.workbench-root');
    if (!root) return;
    root.innerHTML = '';

    const factory = createPanelFactory(this.workspaceRoot);

    const contentArea = document.createElement('div');
    contentArea.id = '__dock-centre';
    contentArea.style.cssText = 'width:100%;height:100%;overflow:hidden;';

    const config = dockWorkbench({
      centre: { type: 'html', props: { id: '__dock-centre' } },
      left: DOCK_PANELS,
      storageKey: 'workbench',
    });

    renderComponent(config, root as HTMLElement);

    const centreMount = root.querySelector('#__dock-centre') ?? contentArea;

    this._container = createContainer({
      entries: [{ key: 'dashboard', label: 'Dashboard' }],
      layout: 'content',
      contentFactory: factory,
      callbacks: {
        onStateChange: () => this._pushUIStateImmediate(),
      },
    });
    this._container.mount(centreMount as HTMLElement);
  }

  private _activatePanel(key: string) {
    if (PANEL_TAGS[key]) {
      this.shadowRoot!.dispatchEvent(new CustomEvent('pages-dock-toggle', {
        bubbles: true, composed: true,
        detail: { panelId: key, visible: true },
      }));
    }
    this._pushUIStateImmediate();
  }

  private _buildUIState(): Record<string, unknown> {
    const state: Record<string, unknown> = {
      layoutMode: this._container?.organiser?.type ?? 'content',
      visiblePanels: this._container?.entries.map(e => e.key) ?? [],
    };
    if (this._pendingCorrelationId) {
      state['correlationId'] = this._pendingCorrelationId;
      this._pendingCorrelationId = null;
    }
    return state;
  }

  private _pushUIStateImmediate() {
    const state = this._buildUIState();
    fetch('/api/model/ui-state', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(state),
    }).catch(() => {});
  }

  private _startHeartbeat() {
    this._heartbeatInterval = setInterval(() => this._pushUIStateImmediate(), 15000);
  }

  private _stopHeartbeat() {
    if (this._heartbeatInterval) {
      clearInterval(this._heartbeatInterval);
      this._heartbeatInterval = null;
    }
  }

  private _connectSSE() {
    this._eventSource = new EventSource('/api/push?topics=control:navigate&topics=control:workspace');
    this._eventSource.addEventListener('message', (event: MessageEvent) => {
      try {
        const msg = JSON.parse(event.data);
        if (msg.topic === 'control:navigate' && msg.payload) {
          const payload = typeof msg.payload === 'string' ? JSON.parse(msg.payload) : msg.payload;
          this._handleNavigateEvent(payload);
        } else if (msg.topic === 'control:workspace' && msg.payload) {
          const payload = typeof msg.payload === 'string' ? JSON.parse(msg.payload) : msg.payload;
          this._handleWorkspaceCommand(payload);
        }
      } catch { /* ignore parse errors */ }
    });
  }

  private _disconnectSSE() {
    if (this._eventSource) {
      this._eventSource.close();
      this._eventSource = null;
    }
  }

  _handleNavigateEvent(payload: { target: string; correlationId?: string }) {
    const { target, correlationId } = payload;
    if (correlationId) this._pendingCorrelationId = correlationId;

    if (target.startsWith('dock-bar/')) {
      this._activatePanel(target.substring('dock-bar/'.length));
    } else if (target.startsWith('panels/')) {
      const parts = target.substring('panels/'.length).split('/');
      const panelId = parts[0] === 'workspace-view' ? 'workspace' : parts[0];
      this._activatePanel(panelId);
      if (panelId === 'workspace' && parts.length >= 3 && parts[1] === 'frames') {
        const wsEl = this.shadowRoot!.querySelector('trellis-workspace-view');
        if (wsEl && typeof (wsEl as any).focusFrame === 'function') {
          (wsEl as any).focusFrame(parts[2]);
          if (parts.length >= 5 && parts[3] === 'tabs') {
            (wsEl as any).focusTab(parts[2], parseInt(parts[4], 10));
          }
        }
      }
    }
    this._pushUIStateImmediate();
  }

  private async _handleWorkspaceCommand(
      payload: { command: string; params?: any; correlationId?: string }) {
    this._activatePanel('workspace');
    const wsView = this.shadowRoot!.querySelector('trellis-workspace-view');
    if (wsView && typeof (wsView as any).handleCommand === 'function') {
      await (wsView as any).handleCommand(payload.command, payload.params);
    }
    if (payload.correlationId) this._pendingCorrelationId = payload.correlationId;
    this._pushUIStateImmediate();
  }

  override render() {
    return html`<div class="workbench-root"></div>`;
  }
}
```

- [ ] **Step 2: Verify the frontend compiles**

Run: `cd sidecar/src/main/webui && yarn build`
Expected: build succeeds with no errors

- [ ] **Step 3: Run existing vitest suite to check for regressions**

Run: `cd sidecar/src/main/webui && yarn test --run`
Expected: existing tests pass (workbench has no unit tests currently — only the new workbench-panels tests from Task 2)

- [ ] **Step 4: Verify no remnants of old code**

Use `ide_search_text` in the trellis project for:
- `DOCK_PANELS` in workbench.ts (should not exist — moved to workbench-panels.ts)
- `_panelCache` (should not exist)
- `_parseHash` (should not exist)
- `_lastHash` (should not exist)
- `PanelDef` interface (should not exist)
- `.dock-bar` CSS class in workbench.ts (should not exist)
- `.dock-btn` CSS class in workbench.ts (should not exist)

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/src/components/workbench.ts
git commit -m "feat(#49): rewrite workbench with pages dock bars and Container

Two-layer architecture: dockWorkbench() + renderComponent() for dock
bars, createContainer() for pluggable content area. SSE navigation via
pages-dock-toggle events. Deep workspace paths forwarded to
workspace-view. UI state push includes layoutMode and visiblePanels.

Replaces 360 lines of manual dock-bar/panel-cache code with ~150 lines
of pages API consumption.

Refs #49"
```

---

### Task 4: Wire layout persistence and remove old endpoint references

Connect `createRestLayoutStore()` to the new `/api/layouts/{key}` endpoint. Remove any remaining references to the old `/api/workspace/layout` and `/api/workspace/groups` endpoints from the frontend.

**Files:**
- Modify: `sidecar/src/main/webui/src/components/workbench.ts` — add layout store wiring
- Modify: `sidecar/src/main/webui/src/components/workbench-panels.test.ts` — add persistence test

**Interfaces:**
- Consumes: `createRestLayoutStore()` from `@casehubio/pages-runtime`, `LayoutResource` at `/api/layouts/{key}` (from Task 1)

- [ ] **Step 1: Write failing test for layout store creation**

Add to `workbench-panels.test.ts`:

```typescript
import { createRestLayoutStore } from '@casehubio/pages-runtime';

describe('layout persistence', () => {
  it('createRestLayoutStore returns a LayoutStore with load/save/delete', () => {
    const store = createRestLayoutStore({ baseUrl: 'http://localhost:8080' });
    expect(store).toBeDefined();
    expect(typeof store.load).toBe('function');
    expect(typeof store.save).toBe('function');
    expect(typeof store.delete).toBe('function');
  });
});
```

- [ ] **Step 2: Run test to verify it passes**

Run: `cd sidecar/src/main/webui && yarn test --run src/components/workbench-panels.test.ts`
Expected: PASS (createRestLayoutStore is already provided by pages-runtime)

- [ ] **Step 3: Add layout store to workbench**

In `workbench.ts`, add layout store initialization in `_renderWorkbench()`:

```typescript
// At the top of _renderWorkbench(), after const factory:
const layoutStore = createRestLayoutStore({
  baseUrl: window.location.origin,
  tokenFn: () => null,
});
```

Pass `layoutStore` to the `loadSite` or use it to save/restore container state on mount and on `onStateChange`.

- [ ] **Step 4: Verify old endpoint references removed**

Use `ide_search_text` across the entire project for:
- `workspace/layout` — should only appear in git history, not in any current file
- `workspace/groups` — same
- `loadLayout` / `saveLayout` / `loadGroups` / `saveGroups` — old LayoutStore method names, should not exist

- [ ] **Step 5: Run full test suite**

Run backend: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Run frontend: `cd sidecar/src/main/webui && yarn test --run`
Expected: all tests pass

- [ ] **Step 6: Run the app and verify**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml quarkus:dev`

Verify in browser:
- Workbench renders with dock bar on the left
- Clicking dock bar buttons switches panels
- Content area shows the active panel
- Page refresh preserves layout state
- SSE navigation still works (test via MCP tool or direct API call)

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/webui/src/components/workbench.ts sidecar/src/main/webui/src/components/workbench-panels.test.ts
git commit -m "feat(#49): wire layout persistence via createRestLayoutStore

Connects to /api/layouts/{key} for layout save/restore.
Removes all references to old /api/workspace/layout and
/api/workspace/groups endpoints.

Refs #49"
```

---

## References

- [2026-08-28-pluggable-workbench-layout-design.md] — design spec this plan implements
- [workbench.ts] — current implementation (replacement target)
- [WorkspaceLayoutResource.java:12] — current layout REST resource
- [WorkspaceLayoutStore.java:9] — current file-based layout store
- [pages-runtime frame-sandbox] — Container, ContentFactory, LayoutStrategy types
- [pages-ui builders.ts] — dockWorkbench, DockWorkbenchConfig
- [GE-20260804-befd45] — dockWorkbench 3-primitive decomposition
- [GE-20260805-e3211c] — registerPanel import ordering
- [GE-20260809-aee002] — zone naming gotcha
- [GitHub #49] — focal issue
