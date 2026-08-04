# Memory Management Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #24 — Repo terminal integration
**Issue group:** #24

**Goal:** Add a dock-bar memory management panel showing all terminals
with process tree memory, individual/bulk pause/resume/terminate, and
a side panel for process tree inspection.

**Architecture:** New `GET /api/terminals/{name}/agent/tree` endpoint
exposes per-process entries from `ProcessTreeWalker`. Frontend is a
single Lit component (`trellis-memory-panel`) registered in the
workbench dock bar. Uses existing `agent:state` SSE for live updates.

**Tech Stack:** Java 21 (Quarkus), TypeScript (Lit), xterm.js ecosystem

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Frontend: Lit web components, esbuild, yarn berry
- Reuse `agent-status-badge` for state rendering
- All commits reference #24

---

### Task 1: Extend ProcessTreeWalker to expose per-process entries

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/agent/ProcessEntry.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/ProcessTreeWalker.java`
- Modify: `sidecar/src/test/java/io/hortora/trellis/agent/ProcessTreeWalkerTest.java` (or create if absent)

**Interfaces:**
- Consumes: existing `ProcessTreeWalker.fromPsOutput(String, long)`
- Produces: `ProcessEntry(long pid, long ppid, long rssBytes, String command)` record; `ProcessTree` gains `List<ProcessEntry> entries` field

- [ ] **Step 1: Write the failing test**

Check if test file exists:
```bash
ls sidecar/src/test/java/io/hortora/trellis/agent/ProcessTreeWalkerTest.java
```

Create or add to the test file:

```java
@Test
void fromPsOutputReturnsProcessEntries() {
    String ps = """
            1234   100  51200 /bin/zsh
            1235  1234 262144 claude --resume
            1236  1235  46080 node playwright-mcp
            1237  1235  35840 node intellij-mcp
            """;
    var tree = ProcessTreeWalker.fromPsOutput(ps, 1234);

    assertTrue(tree.isPresent());
    var entries = tree.get().entries();
    assertEquals(3, entries.size());

    var claude = entries.stream().filter(e -> e.pid() == 1235).findFirst().orElseThrow();
    assertEquals(1234, claude.ppid());
    assertEquals(262144 * 1024, claude.rssBytes());
    assertTrue(claude.command().contains("claude"));

    var playwright = entries.stream().filter(e -> e.pid() == 1236).findFirst().orElseThrow();
    assertEquals(1235, playwright.ppid());
    assertTrue(playwright.command().contains("playwright"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=ProcessTreeWalkerTest#fromPsOutputReturnsProcessEntries -Dsurefire.useFile=false`

Expected: FAIL — `ProcessTree` has no `entries()` method

- [ ] **Step 3: Create ProcessEntry record**

Use `ide_create_file`:

```java
package io.hortora.trellis.agent;

public record ProcessEntry(long pid, long ppid, long rssBytes, String command) {}
```

- [ ] **Step 4: Extend ProcessTree and collectTree**

Use `ide_replace_text_in_file` to update the `ProcessTree` record:

```java
public record ProcessTree(long claudePid, long totalRssBytes, List<Long> allPids, List<ProcessEntry> entries) {}
```

Update `fromPsOutput` — add an `entries` list and populate it in `collectTree`:

```java
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
            entries.put(pid, new String[]{args, String.valueOf(rss), String.valueOf(ppid)});
        } catch (NumberFormatException ignored) {}
    }

    Long claudePid = findClaude(rootPid, children, entries);
    if (claudePid == null) return Optional.empty();

    var allPids = new ArrayList<Long>();
    var processEntries = new ArrayList<ProcessEntry>();
    long totalRss = collectTree(claudePid, children, entries, allPids, processEntries);

    return Optional.of(new ProcessTree(claudePid, totalRss * 1024, List.copyOf(allPids), List.copyOf(processEntries)));
}
```

Update `collectTree` to populate `processEntries`:

```java
private static long collectTree(long pid, Map<Long, List<long[]>> children,
                                 Map<Long, String[]> entries, List<Long> allPids,
                                 List<ProcessEntry> processEntries) {
    allPids.add(pid);
    var entry = entries.get(pid);
    long rss = Long.parseLong(entry[1]);
    long ppid = Long.parseLong(entry[2]);
    processEntries.add(new ProcessEntry(pid, ppid, rss * 1024, entry[0]));
    var kids = children.get(pid);
    if (kids != null) {
        for (long[] kid : kids) {
            rss += collectTree(kid[0], children, entries, allPids, processEntries);
        }
    }
    return rss;
}
```

Fix any existing callers of the old `collectTree` signature (it now takes 5 args).

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=ProcessTreeWalkerTest -Dsurefire.useFile=false`

Expected: PASS

- [ ] **Step 6: Run all agent tests to check for regressions**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest="ProcessTreeWalkerTest,AgentProcessManagerTest" -Dsurefire.useFile=false`

If any existing tests break due to the new `entries` parameter on `ProcessTree`, fix them — they'll need the 4th constructor arg.

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/agent/ProcessEntry.java \
       sidecar/src/main/java/io/hortora/trellis/agent/ProcessTreeWalker.java \
       sidecar/src/test/java/io/hortora/trellis/agent/ProcessTreeWalkerTest.java
git commit -m "feat(#24): expose per-process entries in ProcessTreeWalker"
```

---

### Task 2: Add GET /api/terminals/{name}/agent/tree endpoint

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/agent/AgentSubResource.java`
- Modify: `sidecar/src/test/java/io/hortora/trellis/terminal/TerminalResourceTest.java` (or agent test)

**Interfaces:**
- Consumes: `ProcessTreeWalker.walk(long pid)` returning `ProcessTree` with `entries()`; `AgentProcessManager.getSnapshot()`
- Produces: `GET /api/terminals/{name}/agent/tree` returning JSON `{ rootPid, totalBytes, processes: [{pid, ppid, rssBytes, command}] }`

- [ ] **Step 1: Write the failing test**

```java
@Test
void treeEndpointReturns404ForUnknownTerminal() {
    given()
        .when()
            .get("/api/terminals/nonexistent/agent/tree")
        .then()
            .statusCode(404);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TerminalResourceTest#treeEndpointReturns404ForUnknownTerminal -Dsurefire.useFile=false`

Expected: FAIL — 404 but for the wrong reason (no route matched)

- [ ] **Step 3: Add tree endpoint to AgentSubResource**

Use `ide_insert_member` to add after the `stats` method:

```java
@GET
@Path("/tree")
public Response tree() {
    var terminal = registry.get(terminalName);
    if (terminal.isEmpty()) {
        return Response.status(404).entity(Map.of("error", "terminal not found: " + terminalName)).build();
    }
    var snapshot = processManager.getSnapshot(terminalName, terminal.get());
    if (snapshot.process() == null || snapshot.process().pid() <= 0) {
        return Response.ok(Map.of("rootPid", 0, "totalBytes", 0, "processes", List.of())).build();
    }
    try {
        var treeOpt = ProcessTreeWalker.walk(snapshot.process().pid());
        if (treeOpt.isEmpty()) {
            return Response.ok(Map.of("rootPid", 0, "totalBytes", 0, "processes", List.of())).build();
        }
        var tree = treeOpt.get();
        var processes = tree.entries().stream()
                .map(e -> Map.of(
                        "pid", e.pid(),
                        "ppid", e.ppid(),
                        "rssBytes", e.rssBytes(),
                        "command", e.command()))
                .toList();
        return Response.ok(Map.of(
                "rootPid", tree.claudePid(),
                "totalBytes", tree.totalRssBytes(),
                "processes", processes
        )).build();
    } catch (IOException | InterruptedException e) {
        return Response.serverError().entity(Map.of("error", e.getMessage())).build();
    }
}
```

Add missing imports (`List`, `ProcessTreeWalker`) if needed.

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TerminalResourceTest -Dsurefire.useFile=false`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/agent/AgentSubResource.java \
       sidecar/src/test/java/io/hortora/trellis/terminal/TerminalResourceTest.java
git commit -m "feat(#24): add process tree endpoint GET /agent/tree"
```

---

### Task 3: Create memory-panel.ts frontend component

**Files:**
- Create: `sidecar/src/main/webui/src/components/memory-panel.ts`
- Modify: `sidecar/src/main/webui/src/components/workbench.ts`
- Modify: `sidecar/src/main/webui/src/app.ts` (import)

**Interfaces:**
- Consumes: `GET /api/terminals` returning `AgentSnapshot[]`; `GET /api/terminals/{name}/agent/tree`; `POST /api/terminals/{name}/agent/pause`; `POST /api/terminals/{name}/agent/resume`; `DELETE /api/terminals/{name}`; `EventSource('/api/push?topics=agent:state')`
- Produces: `<trellis-memory-panel>` custom element with `workspaceRoot` property

- [ ] **Step 1: Create memory-panel.ts with table rendering**

Use `ide_create_file` to create `sidecar/src/main/webui/src/components/memory-panel.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import './agent-status-badge';

interface Terminal {
  name: string;
  workingDir: string | null;
  slot: string | null;
  repo: string | null;
  issue: string | null;
}

interface Process {
  pid: number;
  state: string;
  memoryBytes: number;
  startedAt: string | null;
  command: string | null;
}

interface Snapshot {
  terminalName: string;
  terminal: Terminal;
  process: Process | null;
  lastError: string | null;
}

interface ProcessEntry {
  pid: number;
  ppid: number;
  rssBytes: number;
  command: string;
}

interface TreeResponse {
  rootPid: number;
  totalBytes: number;
  processes: ProcessEntry[];
}

type SortKey = 'type' | 'slot' | 'repo' | 'state' | 'memory';
type SortDir = 'asc' | 'desc';

@customElement('trellis-memory-panel')
export class TrellisMemoryPanel extends LitElement {

  @property() workspaceRoot = '';

  @state() private _snapshots: Snapshot[] = [];
  @state() private _selected = new Set<string>();
  @state() private _sortKey: SortKey = 'memory';
  @state() private _sortDir: SortDir = 'desc';
  @state() private _activeRow: string | null = null;
  @state() private _tree: TreeResponse | null = null;
  @state() private _actionInProgress: string | null = null;

  private _eventSource: EventSource | null = null;

  override connectedCallback() {
    super.connectedCallback();
    this._load();
    this._eventSource = new EventSource('/api/push?topics=agent:state');
    this._eventSource.addEventListener('agent:state', () => this._load());
  }

  override disconnectedCallback() {
    super.disconnectedCallback();
    this._eventSource?.close();
    this._eventSource = null;
  }

  private async _load() {
    try {
      const res = await fetch('/api/terminals');
      if (res.ok) this._snapshots = await res.json();
    } catch { /* ignore */ }
  }

  private _sorted(): Snapshot[] {
    const dir = this._sortDir === 'asc' ? 1 : -1;
    return [...this._snapshots].sort((a, b) => {
      switch (this._sortKey) {
        case 'type': return dir * (this._type(a)).localeCompare(this._type(b));
        case 'slot': return dir * (a.terminal.slot ?? '').localeCompare(b.terminal.slot ?? '');
        case 'repo': return dir * (a.terminal.repo ?? a.terminalName).localeCompare(b.terminal.repo ?? b.terminalName);
        case 'state': return dir * (a.process?.state ?? 'IDLE').localeCompare(b.process?.state ?? 'IDLE');
        case 'memory': return dir * ((a.process?.memoryBytes ?? 0) - (b.process?.memoryBytes ?? 0));
        default: return 0;
      }
    });
  }

  private _type(s: Snapshot): string {
    return s.terminal.slot ? 'slot' : 'repo';
  }

  private _toggleSort(key: SortKey) {
    if (this._sortKey === key) {
      this._sortDir = this._sortDir === 'asc' ? 'desc' : 'asc';
    } else {
      this._sortKey = key;
      this._sortDir = key === 'memory' ? 'desc' : 'asc';
    }
  }

  private _toggleSelect(name: string) {
    const next = new Set(this._selected);
    if (next.has(name)) next.delete(name); else next.add(name);
    this._selected = next;
  }

  private _toggleAll() {
    if (this._selected.size === this._snapshots.length) {
      this._selected = new Set();
    } else {
      this._selected = new Set(this._snapshots.map(s => s.terminalName));
    }
  }

  private _totalMemory(snapshots: Snapshot[]): number {
    return snapshots.reduce((sum, s) => sum + (s.process?.memoryBytes ?? 0), 0);
  }

  private _formatBytes(bytes: number): string {
    if (bytes === 0) return '0 MB';
    if (bytes >= 1024 * 1024 * 1024) return (bytes / (1024 * 1024 * 1024)).toFixed(1) + ' GB';
    return Math.round(bytes / (1024 * 1024)) + ' MB';
  }

  private async _pause(name: string) {
    this._actionInProgress = name;
    try {
      await fetch(`/api/terminals/${name}/agent/pause`, { method: 'POST' });
      await this._load();
    } finally { this._actionInProgress = null; }
  }

  private async _resume(name: string) {
    this._actionInProgress = name;
    try {
      await fetch(`/api/terminals/${name}/agent/resume`, { method: 'POST' });
      await this._load();
    } finally { this._actionInProgress = null; }
  }

  private async _terminate(name: string) {
    this._actionInProgress = name;
    try {
      await fetch(`/api/terminals/${name}`, { method: 'DELETE' });
      this._selected.delete(name);
      if (this._activeRow === name) { this._activeRow = null; this._tree = null; }
      await this._load();
    } finally { this._actionInProgress = null; }
  }

  private async _bulkAction(action: 'pause' | 'resume' | 'terminate') {
    const names = [...this._selected];
    for (const name of names) {
      if (action === 'pause') await this._pause(name);
      else if (action === 'resume') await this._resume(name);
      else await this._terminate(name);
    }
    this._selected = new Set();
  }

  private async _selectRow(name: string) {
    if (this._activeRow === name) {
      this._activeRow = null;
      this._tree = null;
      return;
    }
    this._activeRow = name;
    this._tree = null;
    try {
      const res = await fetch(`/api/terminals/${name}/agent/tree`);
      if (res.ok) this._tree = await res.json();
    } catch { /* ignore */ }
  }

  private _sortIndicator(key: SortKey): string {
    if (this._sortKey !== key) return '';
    return this._sortDir === 'asc' ? ' ▲' : ' ▼';
  }

  static override styles = css`
    :host { display: flex; height: 100%; font-family: system-ui, -apple-system, sans-serif; }
    .main { flex: 1; display: flex; flex-direction: column; min-width: 0; }

    .header {
      display: flex; align-items: center; gap: 1rem; padding: 0.75rem 1rem;
      background: #1a1a1a; border-bottom: 1px solid #333; flex-shrink: 0;
    }
    .header h2 { margin: 0; font-size: 1rem; font-weight: 600; }
    .header .spacer { flex: 1; }
    .header .total { font-size: 0.85rem; color: #9ca3af; font-family: monospace; }

    .bulk-actions { display: flex; gap: 0.5rem; }
    .bulk-btn {
      padding: 0.25rem 0.6rem; border: 1px solid #444; border-radius: 4px;
      background: #2a2a2a; color: #666; cursor: not-allowed; font-size: 0.75rem;
    }
    .bulk-btn.enabled { color: #ccc; cursor: pointer; }
    .bulk-btn.enabled:hover { background: #333; }
    .bulk-btn.enabled.danger { border-color: #991b1b; color: #fca5a5; }
    .bulk-btn.enabled.danger:hover { background: #450a0a; }

    .table-wrap { flex: 1; overflow: auto; }
    table { width: 100%; border-collapse: collapse; font-size: 0.8rem; }
    th {
      position: sticky; top: 0; background: #1e1e1e; padding: 0.5rem 0.75rem;
      text-align: left; color: #888; font-weight: 500; cursor: pointer;
      border-bottom: 1px solid #333; user-select: none; white-space: nowrap;
    }
    th:hover { color: #ccc; }
    td { padding: 0.4rem 0.75rem; border-bottom: 1px solid #262626; color: #ccc; }
    tr:hover td { background: #262626; }
    tr.selected td { background: #1e3a5f; }
    tr.active td { background: #2a2a2a; border-left: 2px solid #3b82f6; }

    .type-badge {
      display: inline-flex; padding: 0.1rem 0.4rem; border-radius: 3px;
      font-size: 0.7rem; font-weight: 500;
    }
    .type-slot { background: #166534; color: #86efac; }
    .type-repo { background: #1e3a5f; color: #93c5fd; }

    .memory { font-family: monospace; white-space: nowrap; }
    .memory.high { color: #fbbf24; font-weight: 600; }
    .memory.critical { color: #f87171; font-weight: 600; }

    .action-btn {
      padding: 0.2rem 0.5rem; border: 1px solid #444; border-radius: 3px;
      background: #2a2a2a; color: #ccc; cursor: pointer; font-size: 0.7rem;
      margin-right: 0.25rem;
    }
    .action-btn:hover { background: #333; }
    .action-btn:disabled { opacity: 0.3; cursor: not-allowed; }
    .action-btn.danger { border-color: #991b1b; color: #fca5a5; }
    .action-btn.danger:hover { background: #450a0a; }

    .footer {
      padding: 0.5rem 1rem; background: #1a1a1a; border-top: 1px solid #333;
      font-size: 0.8rem; color: #9ca3af; flex-shrink: 0;
    }

    .sidebar {
      width: 300px; background: #1e1e1e; border-left: 1px solid #333;
      padding: 1rem; overflow-y: auto; flex-shrink: 0;
    }
    .sidebar h3 {
      margin: 0 0 0.75rem; font-size: 0.85rem; font-weight: 600;
      color: #aaa; text-transform: uppercase; letter-spacing: 0.05em;
    }
    .process-entry {
      display: flex; justify-content: space-between; align-items: baseline;
      padding: 0.3rem 0; font-size: 0.75rem; border-bottom: 1px solid #262626;
    }
    .process-entry.child { padding-left: 1rem; }
    .process-cmd { color: #ccc; font-family: monospace; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 180px; }
    .process-mem { color: #9ca3af; font-family: monospace; white-space: nowrap; }
    .process-pid { color: #666; font-size: 0.7rem; margin-right: 0.5rem; }
    .tree-total {
      margin-top: 0.75rem; padding-top: 0.5rem; border-top: 1px solid #444;
      font-size: 0.8rem; font-family: monospace; color: #ccc;
      display: flex; justify-content: space-between;
    }

    .empty { color: #666; padding: 3rem; text-align: center; font-style: italic; }

    input[type="checkbox"] { accent-color: #3b82f6; }
  `;

  override render() {
    const sorted = this._sorted();
    const hasSelection = this._selected.size > 0;
    const allSelected = this._selected.size === this._snapshots.length && this._snapshots.length > 0;
    const selectedSnapshots = this._snapshots.filter(s => this._selected.has(s.terminalName));
    const selectedMemory = this._totalMemory(selectedSnapshots);
    const totalMemory = this._totalMemory(this._snapshots);

    return html`
      <div class="main">
        <div class="header">
          <h2>Memory</h2>
          <div class="bulk-actions">
            <button class="bulk-btn ${hasSelection ? 'enabled' : ''}"
                    ?disabled=${!hasSelection || !!this._actionInProgress}
                    @click=${() => this._bulkAction('pause')}>Pause Selected</button>
            <button class="bulk-btn ${hasSelection ? 'enabled' : ''}"
                    ?disabled=${!hasSelection || !!this._actionInProgress}
                    @click=${() => this._bulkAction('resume')}>Resume Selected</button>
            <button class="bulk-btn ${hasSelection ? 'enabled danger' : ''}"
                    ?disabled=${!hasSelection || !!this._actionInProgress}
                    @click=${() => this._bulkAction('terminate')}>Terminate Selected</button>
          </div>
          <span class="spacer"></span>
          <span class="total">Total: ${this._formatBytes(totalMemory)} across ${this._snapshots.length} terminals</span>
        </div>

        ${this._snapshots.length === 0
          ? html`<div class="empty">No terminal sessions.</div>`
          : html`
            <div class="table-wrap">
              <table>
                <thead><tr>
                  <th><input type="checkbox" .checked=${allSelected}
                             .indeterminate=${hasSelection && !allSelected}
                             @change=${this._toggleAll}></th>
                  <th @click=${() => this._toggleSort('type')}>Type${this._sortIndicator('type')}</th>
                  <th @click=${() => this._toggleSort('slot')}>Slot${this._sortIndicator('slot')}</th>
                  <th @click=${() => this._toggleSort('repo')}>Repo${this._sortIndicator('repo')}</th>
                  <th @click=${() => this._toggleSort('state')}>State${this._sortIndicator('state')}</th>
                  <th @click=${() => this._toggleSort('memory')}>Memory${this._sortIndicator('memory')}</th>
                  <th>Actions</th>
                </tr></thead>
                <tbody>
                  ${sorted.map(s => this._renderRow(s))}
                </tbody>
              </table>
            </div>
          `}

        ${hasSelection ? html`
          <div class="footer">
            Selected: ${this._formatBytes(selectedMemory)} (${this._selected.size} of ${this._snapshots.length} terminals)
          </div>
        ` : nothing}
      </div>

      ${this._activeRow && this._tree ? this._renderSidebar() : nothing}
    `;
  }

  private _renderRow(s: Snapshot) {
    const name = s.terminalName;
    const type = this._type(s);
    const mem = s.process?.memoryBytes ?? 0;
    const memMb = Math.round(mem / (1024 * 1024));
    const state = s.process?.state ?? 'IDLE';
    const isRunning = state === 'RUNNING';
    const isPaused = state === 'PAUSED' || state === 'PAUSED_BY_COORDINATOR';
    const busy = this._actionInProgress === name;

    return html`
      <tr class="${this._selected.has(name) ? 'selected' : ''} ${this._activeRow === name ? 'active' : ''}"
          @click=${() => this._selectRow(name)}>
        <td @click=${(e: Event) => { e.stopPropagation(); this._toggleSelect(name); }}>
          <input type="checkbox" .checked=${this._selected.has(name)}
                 @change=${() => this._toggleSelect(name)}>
        </td>
        <td><span class="type-badge type-${type}">${type}</span></td>
        <td>${s.terminal.slot ?? '—'}</td>
        <td>${s.terminal.repo ?? name}</td>
        <td><agent-status-badge .state=${state} .memoryMb=${0} .lastError=${s.lastError ?? null}></agent-status-badge></td>
        <td class="memory ${memMb > 500 ? 'critical' : memMb > 300 ? 'high' : ''}">${this._formatBytes(mem)}</td>
        <td @click=${(e: Event) => e.stopPropagation()}>
          ${isRunning ? html`
            <button class="action-btn" ?disabled=${busy} @click=${() => this._pause(name)}>Pause</button>
          ` : nothing}
          ${isPaused ? html`
            <button class="action-btn" ?disabled=${busy} @click=${() => this._resume(name)}>Resume</button>
          ` : nothing}
          <button class="action-btn danger" ?disabled=${busy} @click=${() => this._terminate(name)}>Terminate</button>
        </td>
      </tr>
    `;
  }

  private _renderSidebar() {
    const tree = this._tree!;
    const rootPid = tree.rootPid;
    return html`
      <div class="sidebar">
        <h3>Process Tree</h3>
        ${tree.processes.length === 0
          ? html`<div style="color: #666; font-size: 0.8rem;">No processes running.</div>`
          : html`
            ${tree.processes.map(p => html`
              <div class="process-entry ${p.ppid === rootPid || p.pid === rootPid ? '' : 'child'}">
                <div>
                  <span class="process-pid">${p.pid}</span>
                  <span class="process-cmd" title=${p.command}>${p.command.split('/').pop()?.split(' ')[0] ?? p.command}</span>
                </div>
                <span class="process-mem">${this._formatBytes(p.rssBytes)}</span>
              </div>
            `)}
            <div class="tree-total">
              <span>Total</span>
              <span>${this._formatBytes(tree.totalBytes)}</span>
            </div>
          `}
      </div>
    `;
  }
}
```

- [ ] **Step 2: Register in workbench dock bar**

Use `ide_replace_text_in_file` on `workbench.ts` to add the memory panel to PANELS:

Add after the coordinator entry:
```typescript
memory:      { icon: '\u{1F4CA}', label: 'Memory',       tag: 'trellis-memory-panel' },
```

Add `'memory'` to the `DOCK_PANELS` array:
```typescript
const DOCK_PANELS = ['workspace', 'artifacts', 'garden', 'coordinator', 'memory'];
```

Add hash parsing in `_parseHash()`, after the garden block:
```typescript
} else if (hash.match(/^#memory/)) {
  this._activePanel = 'memory';
```

- [ ] **Step 3: Add import in app.ts**

Use `ide_replace_text_in_file` on `app.ts` to add import:

```typescript
import './components/memory-panel';
```

Add after the existing component imports.

- [ ] **Step 4: Build frontend**

Run: `yarn --cwd sidecar/src/main/webui build`

Expected: builds with no errors (warnings OK)

- [ ] **Step 5: Build and run sidecar, verify panel loads**

Run:
```bash
/opt/homebrew/bin/mvn -f sidecar/pom.xml package -DskipTests -q
```

Start sidecar and navigate to `http://localhost:{port}/#memory` — panel should render with the table.

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/webui/src/components/memory-panel.ts \
       sidecar/src/main/webui/src/components/workbench.ts \
       sidecar/src/main/webui/src/app.ts
git commit -m "feat(#24): add memory management panel with process tree sidebar"
```

---

### Task 4: Visual verification with Playwright

**Files:** none (manual testing)

**Interfaces:**
- Consumes: running sidecar with all previous tasks committed

- [ ] **Step 1: Launch sidecar and navigate via Playwright**

```bash
pkill -f "quarkus-run.jar" 2>/dev/null
java -Dquarkus.http.port=0 -jar sidecar/target/quarkus-app/quarkus-run.jar &
```

Navigate Playwright to `http://localhost:{port}/#memory`

- [ ] **Step 2: Create a test terminal and verify it appears**

```bash
curl -s -X POST http://localhost:{port}/api/terminals \
  -H 'Content-Type: application/json' \
  -d '{"name":"mem-test","workingDir":"/tmp","repo":"test-repo","agent":{}}'
```

Take screenshot — table should show the terminal with memory, state badge, and action buttons.

- [ ] **Step 3: Test pause and verify memory drops**

Click Pause button for the test terminal. Take screenshot — state should change to PAUSED, memory to 0 MB.

- [ ] **Step 4: Test process tree sidebar**

Click the test terminal row. Side panel should appear showing the process tree with PIDs, commands, and memory per process.

- [ ] **Step 5: Test bulk selection**

Create a second terminal. Select both via checkboxes. Verify bulk action buttons become enabled and selection summary shows combined memory.

- [ ] **Step 6: Clean up test terminals**

```bash
curl -s -X DELETE http://localhost:{port}/api/terminals/mem-test
```

- [ ] **Step 7: Commit (if any fixes were needed)**

Only if visual testing revealed issues that needed code fixes.
