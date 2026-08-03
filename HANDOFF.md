# Handoff — trellis

## Last Session

Closed `issue-18-llm-coordinator-l3-isx` branch (4 commits squashed to 2). L3 autonomous execution landed on main:
- #18 LLM Coordinator L3 — three autonomy levels (MANUAL/OBSERVATION/AUTONOMOUS), CAS transitions, CountdownScheduler, rate limiter, restart recovery, frontend mode toggle

Code review caught two critical issues (both fixed before merge):
1. TOCTOU race on approve/reject/confirm/cancel — converted all transitions to CAS
2. autoExecute() blocking the countdown scheduler thread — offloaded to dedicated action executor

Blog entry: "When the Coordinator Stops Asking Permission" — covers CAS concurrency, countdown persistence, rate limiting as circuit breaker.

Garden entry: GE-20260803-0c6c56 (IntelliJ MCP ide_change_signature fails on record constructors).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #19 | Velocity Tracking + Projections | M | Med | Ready |
| #16 | Drafthouse Integration (B6c) | M | Med | Ready |
| #21 | End-to-end provenance test path | S | Low | Ready |
| #22 | Slot-agent pause/resume coordination | M | Med | Enhancement |

## Epic Progress

Batches 1-8 complete. #14, #15, #17, #18, #20 complete and landed. Remaining: #16, #19.

ISX (originally part of #18) was deferred — the design spec explicitly scoped it out. Will be raised as a separate issue when needed.

## References

- Design spec: `docs/specs/issue-18-llm-coordinator-l3-isx/2026-08-02-llm-coordinator-l3-design.md`
- Blog: `blog/2026-08-03-mdp03-when-the-coordinator-stops-asking-permission.md`
- Garden entry: GE-20260803-0c6c56 (ide_change_signature record constructor gotcha)
- Unrecovered specs from epic-2-post-mvp: process-isolation, artifact-browser, L2 coordinator (3 specs on closed workspace branch)
