---
layout: post
title: "When Your Agent Learns to Read the Logbook"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [trellis]
tags: [worklog, sqlite, model-tree, mcp]
---

Trellis has always known what's on disk. The `FileWatcherService` scans the workspace, finds repos, slots, epics — a point-in-time snapshot of the physical layout. What it couldn't see was the history. When did work start on this branch? How many sessions deep is this slot? What issue is active in the plan queue? That lifecycle data lived in soredium's `worklog.db`, written by Python, read by nobody on the Java side.

The gap matters because the coordinating agent makes decisions. Should it pause a slot? Resume an old branch? Suggest the next issue? Those decisions need work history, not just filesystem state. A slot directory tells you it exists. The worklog tells you it was created three days ago, paused twice, and has four completed issues.

I wanted a clean bridge — one service that owns all access to `worklog.db`, following the same pattern we established for terminals: `TerminalRegistry` is the service, `TerminalResource` serves the frontend, `TerminalModelProvider` serves the MCP model tree. Three layers, one data source, no inline SQL scattered across resources.

The interesting design tension was consolidation. `BacklogResource` already queried `worklog.db` directly for enrichment and cache data — issue classifications, trajectory notes, the GitHub issue cache. The issue said "single access layer." I could have kept two readers with clear domain boundaries, but the maintenance cost of two schema-version checks, two availability guards, and two connection paths for the same SQLite file wasn't worth the architectural purity. `BacklogResource` became a thin delegate.

The freshness problem was subtler than expected. The worklog DB is external — soredium writes it, trellis only reads. Trellis can't increment its `GenerationCounter` on mutation because it doesn't control mutations. The natural instinct is `PRAGMA data_version` — SQLite's built-in change counter. We tried it; the decision review caught the flaw. `data_version` is per-connection. With connection-per-query from a `DataSource`, you get a fresh handle each time, and the value resets. Two connections opened a minute apart can report identical `data_version` even after a hundred writes in between. The fix is pedestrian: check the file modification time. One stat call, near-zero cost, and it actually works across processes.

The `WorklogModelProvider` adds a `worklog` branch to the model tree. `trellis_model()` now includes active work items, recent events, slot states, and plan queue position in its summary. `trellis_model(path="worklog/events")` returns the last 50 lifecycle events. The REST endpoints at `/api/worklog/` mirror the same subpaths but add query parameters — `?since=`, `?type=`, `?limit=` — for the frontend panels that will consume them next.

The worklog DB has eight tables across two schema versions. The service checks `PRAGMA user_version` on startup: below 2 means the schema is too old and the whole domain degrades gracefully; above 2 means soredium has evolved the schema and we continue best-effort. Individual subpaths catch their own exceptions — a missing V2 table takes down the backlog view, not the event stream.

What this opens up: the workspace panel can show session history alongside the physical layout. The coordinating agent can reason about work patterns — which slots are active, how long branches have been open, what the plan queue looks like — without shelling out to Python.
