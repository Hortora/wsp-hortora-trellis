# Handoff — trellis

## Last Session

Closed `epic-2-post-mvp` branch (48 commits squashed to 4). Three features landed on main:
- #14 Garden Service + Provenance — search, browse, provenance enrichment
- #20 Agent process lifecycle — monitoring, memory, start/stop/pause/resume/refresh, REST API, frontend
- #15 Artifact browser and workbench shell — dock bar replacing hash router, sidebar navigation, markdown viewer

Also: closed trellis#1 (workspace state log — superseded by soredium worklog). Filed soredium#157 (REST+MCP for worklog) and soredium#158 (individual issue tracking in worklog).

Fixed: 400 response body for invalid StartAgentRequest (moved validation from record constructor to explicit validate() method).

## What's Next

| # | Description | Scale | Complexity | Status |
|---|-------------|-------|------------|--------|
| #17 | LLM Coordinator L2 — Propose Actions | L | High | Worktree ready |
| #19 | Velocity Tracking + Projections | M | Med | Ready |
| #16 | Drafthouse Integration (B6c) | M | Med | Unblocked by #15 |
| #18 | LLM Coordinator L3 + ISX (B8) | L | High | Blocked by #17 |

## Epic Progress

Batches 1-6 (MVP) complete. #14, #15, #20 complete and landed. Remaining: #16, #17, #18, #19.

## References

- Worktree for #17: `/Users/mdproctor/claude/hortora/trellis/.claude/worktrees/issue-17-llm-coordinator-l2`
- Soredium worklog REST+MCP: Hortora/soredium#157
- Soredium issue tracking: Hortora/soredium#158
- Slot-agent coordination: Hortora/trellis#22
- Garden entry: GE-20260803-17fc03 (casehub-packages portal resolution gotcha)
