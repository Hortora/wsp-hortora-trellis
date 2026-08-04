# Repo Terminal Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD (create at work-start)

**Goal:** Give repos their own Claude agent terminal so they're first-class work targets alongside slots.

**Architecture:** Rebuild `repo-detail.ts` to mirror the slot-detail layout (main terminal area + metadata sidebar). Reuse existing `<trellis-terminal-panel>`, `<agent-status-badge>`, terminal REST API, and SSE push. No backend changes.

**Tech Stack:** Lit web components, xterm.js (existing), Quarkus REST + WebSocket (existing)

## Global Constraints

- Lit 3.x web components with `@customElement` decorators
- Terminal naming: `repo-{repoName}` for repo terminals
- All terminal I/O through Quarkus sidecar WebSocket (no node-pty)
- Follow existing slot-detail patterns for consistency

---

### Task 1: Rebuild repo-detail as terminal-hosting view

**Files:**
- Modify: `src/main/webui/src/views/repo-detail.ts` (complete rewrite)

**Interfaces:**
- Consumes: `GET /api/workspace/repo?root=...&repo=...` → `RepoData` (existing)
- Consumes: `GET /api/terminals` → `AgentSnapshot[]` (existing, same as slot-detail)
- Consumes: `POST /api/terminals` with `CreateTerminalRequest` (existing)
- Consumes: `POST /api/terminals/{name}/agent/{action}` (existing)
- Consumes: `GET /api/push?topics=agent:state` SSE (existing)
- Consumes: `<trellis-terminal-panel>` component (existing)
- Consumes: `<agent-status-badge>` component (existing)

- [ ] **Step 1: Add terminal and agent imports to repo-detail.ts**

Add the imports for terminal-panel and agent-status-badge at the top of the file, and add the `AgentProcess` and `AgentSnapshot` interfaces (same as slot-detail uses):

```typescript
import '../components/terminal-panel';
import '../components/agent-status-badge';
```

Add after `RepoData` interface:

```typescript
interface AgentProcess {
  pid: number;
  state: string;
  memoryBytes: number;
  startedAt: string | null;
  command: string | null;
}

interface AgentSnapshot {
  terminalName: string;
  terminal: {
    name: string;
    workingDir: string | null;
    slot: string | null;
    repo: string | null;
    issue: string | null;
  };
  process: AgentProcess | null;
  lastError: string | null;
}
```

- [ ] **Step 2: Add new state properties to TrellisRepoDetail**

Add these `@state()` fields alongside existing ones:

```typescript
@state() private _snapshot: AgentSnapshot | null = null;
@state() private _actionInProgress: string | null = null;
```

Add an EventSource field:

```typescript
private _eventSource: EventSource | null = null;
```

- [ ] **Step 3: Replace the styles block**

Replace the existing `static override styles` with a layout matching slot-detail — flex host, main area with toolbar and terminal, 280px sidebar:

```typescript
static override styles = css`
  :host { display: flex; height: 100%; font-family: system-ui, -apple-system, sans-serif; }

  .main { flex: 1; display: flex; flex-direction: column; min-width: 0; }

  .toolbar {
    display: flex; align-items: center; gap: 0.75rem; padding: 0.5rem 1rem;
    background: #1a1a1a; border-bottom: 1px solid #333; flex-shrink: 0;
  }
  .toolbar h2 { margin: 0; font-size: 1rem; font-weight: 600; }
  .toolbar .spacer { flex: 1; }

  .action-btn {
    padding: 0.3rem 0.75rem; border: 1px solid #444; border-radius: 4px;
    background: #2a2a2a; color: #ccc; cursor: pointer; font-size: 0.75rem;
    transition: background 0.15s;
  }
  .action-btn:hover { background: #333; }
  .action-btn:disabled { opacity: 0.4; cursor: not-allowed; }
  .action-btn.danger { border-color: #991b1b; color: #fca5a5; }
  .action-btn.danger:hover { background: #450a0a; }
  .action-btn.primary { border-color: #1d4ed8; color: #93c5fd; }
  .action-btn.primary:hover { background: #1e3a5f; }

  .terminal-area { flex: 1; min-height: 0; }

  .empty-state {
    flex: 1; display: flex; flex-direction: column; align-items: center;
    justify-content: center; gap: 1rem; color: #666;
  }
  .empty-state p { margin: 0; font-size: 0.9rem; }

  .start-btn {
    padding: 0.5rem 1.5rem; border: 1px solid #1d4ed8; border-radius: 6px;
    background: #1e3a5f; color: #93c5fd; cursor: pointer; font-size: 0.85rem;
    font-weight: 500; transition: background 0.15s;
  }
  .start-btn:hover { background: #1d4ed8; }
  .start-btn:disabled { opacity: 0.4; cursor: not-allowed; }

  .sidebar {
    width: 280px; background: #1e1e1e; border-left: 1px solid #333;
    padding: 1rem; overflow-y: auto; flex-shrink: 0;
  }
  .sidebar h3 {
    margin: 0 0 0.5rem; font-size: 0.85rem; font-weight: 600;
    color: #aaa; text-transform: uppercase; letter-spacing: 0.05em;
  }
  .sidebar-section { margin-bottom: 1.5rem; }

  .meta-item { font-size: 0.8rem; color: #999; margin-bottom: 0.3rem; }
  .meta-value { color: #ccc; font-family: monospace; }

  .badge {
    display: inline-flex; padding: 0.1rem 0.5rem; border-radius: 4px;
    font-size: 0.7rem; font-weight: 500;
  }
  .badge-branch { background: #1e3a5f; color: #93c5fd; }

  .remote-link { color: #60a5fa; font-size: 0.8rem; text-decoration: none; }
  .remote-link:hover { text-decoration: underline; }

  .error { color: #f87171; padding: 1rem; }
  .loading { color: #666; padding: 2rem; text-align: center; }
`;
```

- [ ] **Step 4: Add lifecycle methods for SSE and terminal loading**

Replace the `updated()` method and add `connectedCallback`, `disconnectedCallback`, terminal loading, and SSE subscription:

```typescript
override connectedCallback() {
  super.connectedCallback();
  this._loadRepo();
  this._loadTerminal();
  this._subscribeEvents();
}

override disconnectedCallback() {
  super.disconnectedCallback();
  this._eventSource?.close();
}

override updated(changed: Map<PropertyKey, unknown>) {
  if ((changed.has('repoName') || changed.has('workspaceRoot')) && this.repoName && this.workspaceRoot) {
    const key = `${this.workspaceRoot}:${this.repoName}`;
    if (key !== this._lastLoaded) {
      this._lastLoaded = key;
      this._loadRepo();
      this._loadTerminal();
    }
  }
}

private _subscribeEvents() {
  this._eventSource = new EventSource('/api/push?topics=agent:state');
  this._eventSource.addEventListener('agent:state', () => this._loadTerminal());
}

private async _loadTerminal() {
  try {
    const res = await fetch('/api/terminals');
    if (!res.ok) return;
    const all: AgentSnapshot[] = await res.json();
    this._snapshot = all.find(
      s => s.terminal.repo === this.repoName && !s.terminal.slot
    ) ?? null;
  } catch { /* ignore */ }
}
```

- [ ] **Step 5: Replace the render method**

Replace the existing `render()` with the two-panel layout:

```typescript
override render() {
  if (this._loading) return html`<div class="loading">Loading ${this.repoName}...</div>`;
  if (this._error) return html`<div class="error">${this._error}</div>`;
  if (!this._repo) return nothing;

  return html`
    <div class="main">
      ${this._renderToolbar()}
      ${this._snapshot
        ? html`<div class="terminal-area">
            <trellis-terminal-panel .sessionName=${this._snapshot.terminalName}></trellis-terminal-panel>
          </div>`
        : html`<div class="empty-state">
            <p>No agent running for this repo.</p>
            <button class="start-btn" ?disabled=${!!this._actionInProgress}
                    @click=${this._createTerminal}>Start Agent</button>
          </div>`
      }
    </div>
    ${this._renderSidebar()}
  `;
}
```

- [ ] **Step 6: Add toolbar and sidebar render methods**

Add the toolbar (back button + repo name + branch badge):

```typescript
private _renderToolbar() {
  const repo = this._repo!;
  return html`
    <div class="toolbar">
      <button class="action-btn" @click=${this._goBack} title="Back to workspace">←</button>
      <h2>${repo.name}</h2>
      <span class="badge badge-branch">${repo.branch}</span>
      <span class="spacer"></span>
    </div>
  `;
}
```

Add the sidebar with repo metadata and terminal controls:

```typescript
private _renderSidebar() {
  const repo = this._repo!;
  const gh = this._githubUrl();
  return html`
    <div class="sidebar">
      <div class="sidebar-section">
        <h3>Path</h3>
        <div class="meta-item"><span class="meta-value">${repo.path}</span></div>
      </div>

      ${gh ? html`
        <div class="sidebar-section">
          <h3>Remote</h3>
          <a class="remote-link" href=${gh} target="_blank">${gh}</a>
        </div>
      ` : repo.remoteUrl ? html`
        <div class="sidebar-section">
          <h3>Remote</h3>
          <div class="meta-item"><span class="meta-value">${repo.remoteUrl}</span></div>
        </div>
      ` : nothing}

      ${this._snapshot ? html`
        <div class="sidebar-section">
          <h3>Agent</h3>
          <div class="meta-item" style="display:flex;align-items:center;gap:0.4rem;margin-bottom:0.5rem">
            <agent-status-badge
              .state=${this._snapshot.process?.state ?? 'IDLE'}
              .memoryMb=${this._snapshot.process ? Math.round(this._snapshot.process.memoryBytes / (1024 * 1024)) : 0}
              .lastError=${this._snapshot.lastError}
            ></agent-status-badge>
          </div>
          <div style="display:flex;gap:0.3rem">
            ${this._renderAgentButtons()}
          </div>
        </div>
      ` : nothing}
    </div>
  `;
}
```

- [ ] **Step 7: Add agent action buttons and terminal creation**

Add the agent button renderer (mirrors slot-detail):

```typescript
private _renderAgentButtons() {
  if (!this._snapshot) return nothing;
  const state = this._snapshot.process?.state ?? 'IDLE';
  const disabled = !!this._actionInProgress;
  switch (state) {
    case 'RUNNING':
      return html`
        <button class="action-btn" ?disabled=${disabled}
                @click=${() => this._agentAction('refresh')}>refresh</button>
        <button class="action-btn" ?disabled=${disabled}
                @click=${() => this._agentAction('pause')}>pause</button>
        <button class="action-btn danger" ?disabled=${disabled}
                @click=${() => this._agentAction('stop')}>stop</button>
      `;
    case 'PAUSED':
    case 'PAUSED_BY_COORDINATOR':
      return html`
        <button class="action-btn primary" ?disabled=${disabled}
                @click=${() => this._agentAction('resume')}>resume</button>
      `;
    case 'IDLE':
      return html`
        <button class="action-btn primary" ?disabled=${disabled}
                @click=${() => this._agentAction('start')}>start</button>
      `;
    case 'STARTING':
      return html`<span class="meta-item">starting...</span>`;
    default:
      return nothing;
  }
}
```

Add terminal creation and agent action methods:

```typescript
private async _createTerminal() {
  if (!this._repo) return;
  this._actionInProgress = 'create';
  try {
    const res = await fetch('/api/terminals', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: `repo-${this.repoName}`,
        workingDir: this._repo.path,
        repo: this.repoName,
        agent: {},
      }),
    });
    if (!res.ok) {
      const body = await res.json().catch(() => null);
      this._error = body?.error ?? `Failed to create terminal: HTTP ${res.status}`;
      return;
    }
    await this._loadTerminal();
  } catch (e) {
    this._error = `Failed to create terminal: ${e}`;
  } finally {
    this._actionInProgress = null;
  }
}

private async _agentAction(action: string) {
  if (!this._snapshot) return;
  this._actionInProgress = action;
  try {
    const res = await fetch(`/api/terminals/${this._snapshot.terminalName}/agent/${action}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: action === 'start' ? '{}' : undefined,
    });
    if (!res.ok) {
      const body = await res.json().catch(() => null);
      this._error = body?.error ?? `${action} failed: HTTP ${res.status}`;
    }
    await this._loadTerminal();
  } catch (e) {
    this._error = `${action} failed: ${e}`;
  } finally {
    this._actionInProgress = null;
  }
}
```

- [ ] **Step 8: Build and verify**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: Build succeeds with no TypeScript errors.

- [ ] **Step 9: Manual test**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml package -DskipTests && npm start`

Test the following:
1. Navigate to a repo from the workspace dashboard — should show the new layout with empty state and "Start Agent" button
2. Click "Start Agent" — terminal should appear with Claude starting
3. Agent controls in sidebar (pause/resume/stop/refresh) should work
4. Back button returns to workspace dashboard
5. Re-entering the repo detail should reconnect to the existing terminal

- [ ] **Step 10: Commit**

```bash
git add sidecar/src/main/webui/src/views/repo-detail.ts
git commit -m "feat(#N): add Claude agent terminal to repo detail view

Rebuild repo-detail to mirror slot-detail layout: terminal area + metadata
sidebar. Repos are now first-class work targets with their own Claude agent.

Refs #N"
```
