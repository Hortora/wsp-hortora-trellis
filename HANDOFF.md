# Handoff — epic-2-post-mvp

## Last Session

Built process isolation + session lifecycle (trellis#20). Renamed Session→Terminal throughout codebase. Added AgentProcessManager with hybrid process discovery (tmux pane_current_command introspection + process tree RSS summation), lifecycle operations (start/stop/pause/resume/refresh with tree-kill and shell escaping), AgentSubResource REST API nested under /api/terminals/{name}/agent/, and frontend agent-status-badge component with contextual lifecycle buttons. All 213 tests green. Design spec went through standard adversarial review (59 issues, all resolved). Blog entry written.

## What's Left

- Push epic-2-post-mvp branch to remote (7 commits for #20) · XS · Low
- Push workspace commits to remote (spec, plan, blog) · XS · Low
- Manual browser test of agent status badges and lifecycle buttons · S · Low
- Filed Hortora/trellis#22 during design for slot↔agent lifecycle coordination · tracked

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #15 | Artifact Browser (B6b) | M | Low | Ready |
| #17 | LLM Coordinator L2 — Propose Actions | L | High | Ready |
| #19 | Velocity Tracking + Projections | M | Med | Ready |
| #1 | Workspace State Log | M | Med | Ready |
| #16 | Drafthouse Integration (B6c) | M | Med | Blocked by #15 |
| #18 | LLM Coordinator L3 + ISX (B8) | L | High | Blocked by #17 |

## Epic Progress

Batches 1–6 (MVP) complete. Batch 7 (#14 Garden Service) complete. #20 (Process Isolation) complete. Remaining: #15, #16, #17, #18, #19, #1.

## References

- Spec: `wksp/specs/epic-2-post-mvp/2026-08-02-process-isolation-design.md`
- Plan: `wksp/plans/2026-08-03-process-isolation.md`
- Blog: `wksp/blog/2026-08-03-mdp01-when-your-agents-forget-theyre-mortal.md`
- Agent package: `io.hortora.trellis.agent` (AgentProcessManager, AgentSubResource, AgentState, AgentProcess, AgentSnapshot, StartAgentRequest, ProcessTreeWalker)
- Terminal package: `io.hortora.trellis.terminal` (TerminalInfo, TerminalRegistry — renamed from Session*)
- Claudony reference: `io.casehub.claudony.server.expiry.StatusAwareExpiryPolicy` (discovery pattern source)
- Slot↔agent coordination: Hortora/trellis#22
