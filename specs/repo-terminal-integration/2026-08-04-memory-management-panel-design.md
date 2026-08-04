# Memory Management Panel — Design Spec

## Summary

A new dock-bar panel for monitoring and controlling terminal/agent memory usage.
Shows all terminals in a sortable table with process tree memory totals,
individual and bulk pause/resume/terminate controls, and a side panel for
per-process tree inspection.

## Motivation

Trellis manages many concurrent Claude agents across slots and repos. Each agent
spawns child processes (MCP servers, LSPs, tools) that accumulate memory. Without
visibility, agents silently consume gigabytes. The memory panel gives operators a
single view to identify, diagnose, and reclaim memory.

## Panel Registration

Add `memory` to `PANELS` and `DOCK_PANELS` in `workbench.ts`:
- Icon: a gauge/meter icon
- Label: "Memory"
- Tag: `trellis-memory-panel`
- Hash route: `#memory`

No additional context required — the panel queries `/api/terminals` directly.

## Table Layout

### Columns

| Column | Source | Notes |
|--------|--------|-------|
| ☐ (checkbox) | UI state | Select for bulk actions |
| Type | `terminal.slot ? 'slot' : 'repo'` | Badge: green for slot, blue for repo |
| Slot | `terminal.slot` | Dash if null |
| Repo | `terminal.repo` | Terminal name fallback |
| State | `process.state` | Render via `agent-status-badge` |
| Memory | `process.memoryBytes` | Formatted as MB/GB, process tree total |
| Actions | — | Pause/Resume + Terminate buttons |

### Sorting

- Default: memory descending (heaviest first)
- All columns sortable by header click
- Paused/idle terminals (0 memory) sink to bottom naturally

### Selection and Bulk Actions

**Header row:**
- Select-all checkbox: toggles all visible rows
- Partial selection shows indeterminate state on select-all
- Bulk buttons (greyed out until 1+ selected):
  - **Pause Selected** — calls `POST /api/terminals/{name}/agent/pause` for each
  - **Resume Selected** — calls `POST /api/terminals/{name}/agent/resume` for each
  - **Terminate Selected** — calls `DELETE /api/terminals/{name}` for each
- Buttons operate sequentially to avoid overloading (not parallel)

**Summary line (visible when items selected):**
- `Selected: 1.0 GB (3 of 5 terminals)`
- Shows total memory of selected terminals — the amount you'd free by pausing

**Total line (always visible):**
- `Total: 1.8 GB across 5 terminals`

## Side Panel — Process Tree Detail

Clicking a table row opens a side panel showing the process tree for that
terminal. Layout follows the existing sidebar pattern (280px, right side).

### Content

- Terminal name and state at top
- Process tree, indented by parent/child:
  ```
  PID 12340 — claude --resume          524 MB
    PID 12345 — node playwright-mcp     45 MB
    PID 12350 — node intellij-mcp       37 MB
  ```
- Total at bottom

### Data Source

New endpoint: `GET /api/terminals/{name}/agent/tree`

Response:
```json
{
  "rootPid": 12340,
  "totalBytes": 608174080,
  "processes": [
    {"pid": 12340, "ppid": 12300, "rssBytes": 524288000, "command": "claude --resume"},
    {"pid": 12345, "ppid": 12340, "rssBytes": 47185920, "command": "node playwright-mcp"},
    {"pid": 12350, "ppid": 12340, "rssBytes": 36700160, "command": "node intellij-mcp"}
  ]
}
```

Returns empty `processes` array if agent is not running.

## Backend Changes

### ProcessTreeWalker — expose per-process entries

Currently `ProcessTree` is `record ProcessTree(long claudePid, long totalRssBytes, List<Long> allPids)`.

Add a new record for individual entries:
```java
record ProcessEntry(long pid, long ppid, long rssBytes, String command) {}
```

Extend `ProcessTree`:
```java
record ProcessTree(long claudePid, long totalRssBytes, List<Long> allPids, List<ProcessEntry> entries) {}
```

The `fromPsOutput` parser already has all this data — it just discards the
per-process details. Retain them instead.

### New endpoint on AgentSubResource

```
GET /api/terminals/{name}/agent/tree
```

Calls `ProcessTreeWalker.walk(pid)` and returns the process entries. Returns
404 if terminal not found, empty tree if agent not running.

### No other backend changes

Existing endpoints cover all lifecycle operations:
- `POST /agent/pause` — tree-kills agent + all children
- `POST /agent/resume` — restarts Claude in existing tmux session
- `POST /agent/stop` — stops agent
- `DELETE /api/terminals/{name}` — destroys terminal entirely

## Pause / Resume / Terminate Semantics

| Action | API Call | Effect | Memory freed |
|--------|----------|--------|-------------|
| Pause | `POST /agent/pause` | SIGTERM → 5s → tree-SIGKILL. Tmux stays. | Agent + all children |
| Resume | `POST /agent/resume` | Restarts Claude in existing tmux. Claude respawns MCP servers. | None (adds) |
| Terminate | `DELETE /api/terminals/{name}` | Kills everything, destroys tmux session. | All |

Pause kills the entire process tree via `treeKill()` — MCP servers, LSPs, and
all children are terminated. No option to keep MCP servers alive because:
1. Trellis's `treeKill` already handles this correctly (children-first kill order)
2. Claude respawns fresh MCP servers on resume — it doesn't reconnect to old ones
3. Orphaned MCP servers are pure waste (known Claude Code issue)

## Live Updates

Subscribe to `EventSource('/api/push?topics=agent:state')`. On each event,
re-fetch `GET /api/terminals` and reconcile the table. Memory numbers update
as `AgentProcessManager` polls process state.

Side panel auto-refreshes its process tree on the same event cadence.

## Component Structure

```
trellis-memory-panel (Lit element, registered in workbench dock bar)
├── header: total memory, bulk action buttons
├── table: sortable, selectable rows
│   └── each row: type badge, slot, repo, state badge, memory, action buttons
├── footer: selection summary
└── side panel (conditional, on row click)
    └── process tree detail
```

Single file: `sidecar/src/main/webui/src/components/memory-panel.ts`

Reuses `agent-status-badge` for state display. No new shared components needed.

## Out of Scope

- Memory history / time-series charts
- Auto-pause on memory threshold
- Per-process kill from side panel (visibility only)
- MCP server preservation during pause (not useful — Claude respawns them)
