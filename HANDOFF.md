# Handoff — epic-2-post-mvp

## Last Session

Resumed from handover. Rebased epic-2-post-mvp (32 commits) onto latest main (7 new commits since last session). Clean rebase, no conflicts. No new implementation work this session.

## What's Left

- Manual browser test of agent status badges and lifecycle buttons (#20) · S · Low
- Filed Hortora/trellis#22 during design for slot-agent lifecycle coordination · tracked

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

Batches 1-6 (MVP) complete. Batch 7 (#14 Garden Service) complete. #20 (Process Isolation) complete. Remaining: #15, #16, #17, #18, #19, #1.

## References

- Spec: `wksp/specs/epic-2-post-mvp/2026-08-02-process-isolation-design.md`
- Plan: `wksp/plans/2026-08-03-process-isolation.md`
- Blog: `wksp/blog/2026-08-03-mdp01-when-your-agents-forget-theyre-mortal.md`
- Agent package: `io.hortora.trellis.agent` (AgentProcessManager, AgentSubResource, AgentState, AgentProcess, AgentSnapshot, StartAgentRequest, ProcessTreeWalker)
- Terminal package: `io.hortora.trellis.terminal` (TerminalInfo, TerminalRegistry — renamed from Session*)
- Slot-agent coordination: Hortora/trellis#22
