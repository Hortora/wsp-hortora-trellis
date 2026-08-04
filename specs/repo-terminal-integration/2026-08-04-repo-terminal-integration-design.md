# Repo Terminal Integration — Design Spec

**Date:** 2026-08-04
**Issue:** TBD (will be created at work-start)
**Status:** Approved

## Problem

Repos currently have a minimal detail view (name, branch, path, remote URL) with no ability to run Claude agents. All agent work requires creating a slot first, which is overhead for small tasks. Repos need to be first-class work targets with their own Claude agent terminals.

## Model

- **Repo (standalone):** one Claude agent terminal, working directory is the repo path
- **Slot:** one Claude agent terminal, all repos in the slot accessible through it (unchanged)

Terminal count is 1:1 with work targets. No multi-terminal tab groups needed for either.

## Design

### Backend — no changes required

The existing `POST /api/terminals` already accepts `{name, workingDir, repo}` without a `slot`. The WebSocket endpoint (`/ws/terminal/{id}/{cols}/{rows}`), tmux management (`TmuxManager`), agent lifecycle (`AgentProcessManager`), and SSE push (`agent:state`) all work regardless of whether a terminal belongs to a slot or a repo.

### Frontend — rebuild `repo-detail.ts`

Rebuild `trellis-repo-detail` from a simple info page to a terminal-hosting view that mirrors the slot detail layout.

#### Layout

Same flex structure as `trellis-slot-detail`:

```
┌─────────────────────────────────────────┬──────────┐
│ ← RepoName          [branch badge]      │ Status   │
├─────────────────────────────────────────┤ Branch   │
│                                         │ Path     │
│                                         │ Remote   │
│           Terminal Area                 │          │
│     (terminal-panel or empty state)     │ Terminal │
│                                         │ [badge]  │
│                                         │ [actions]│
│                                         │          │
└─────────────────────────────────────────┴──────────┘
```

- **Host:** `display: flex; height: 100%` (not `display: block` with padding)
- **Main area** (flex: 1): toolbar + terminal area
- **Sidebar** (280px): repo metadata + terminal/agent controls

#### States

**1. No terminal exists (empty state)**

Main area shows a centered prompt with a "Start Agent" button. Sidebar shows repo metadata only (branch, path, remote URL). No terminal section in sidebar.

On click "Start Agent":
- `POST /api/terminals` with body:
  ```json
  {
    "name": "repo-{repoName}",
    "workingDir": "{repo.path}",
    "repo": "{repoName}",
    "agent": {}
  }
  ```
- Transitions to the terminal-exists state

**2. Terminal exists**

Main area shows `<trellis-terminal-panel>` directly (not `<trellis-terminal-tab-group>` — single terminal, no tab chrome). Sidebar adds terminal section: agent status badge + action buttons matching slot detail (start/stop/pause/resume/refresh).

#### Data Loading

- `GET /api/workspace/repo?root=...&repo=...` — repo metadata (existing endpoint)
- `GET /api/terminals` → filter by `terminal.repo === repoName && terminal.slot === null` — find repo's terminal
- SSE: subscribe to `agent:state` topic via `/api/push` for live agent status updates

#### Terminal Naming Convention

- Repo terminals: `repo-{repoName}` (e.g., `repo-trellis`, `repo-engine`)
- Slot terminals: existing `slot-{N}` convention (unchanged)

### Components Reused (no changes)

- `<trellis-terminal-panel>` — xterm.js + WebSocket + fit/resize
- `<agent-status-badge>` — agent state display
- `TerminalResource` — REST API for terminal CRUD
- `TerminalWebSocket` — WebSocket bridge to tmux
- `AgentSubResource` — agent lifecycle actions
- `AgentProcessManager` — agent process management + SSE broadcast

## Not In Scope

- Shared component extraction between slot-detail and repo-detail (layout will change)
- IntelliJ-style dock for coordinator (future rework)
- Alternative navigation/layout for multi-window workflows (future)
- Work-start/work-end lifecycle changes for repos (separate refinement)
