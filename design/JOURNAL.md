# Design Journal — issue-18-llm-coordinator-l3-isx

## §1 Autonomy Model — 2026-08-02

Three autonomy levels (MANUAL/OBSERVATION/AUTONOMOUS) with a hybrid
risk/override policy. Risk classification sets the default (LOW=autonomous,
HIGH=gated), per-action-type overrides let users promote or demote. Most
users need zero configuration.

Observation mode: server-side countdown, persisted for restart recovery.
Veto = REJECTED (existing terminal state — no new states needed).

## §2 ActionService Owns Autonomy — 2026-08-02

ActionService owns the autonomy decision, not CoordinatorService. Design
review surfaced god-class risk — extracted AutonomyResolver and
CountdownScheduler to keep responsibilities focused.

## §3 Review Findings — Countdown Path — 2026-08-02

Countdown calling approve() would send HIGH-risk actions to CONFIRMING
with nobody to confirm. Solution: autoExecute() bypasses the risk gate.
All transitions use CAS (WHERE status = ?) to prevent double-execution.
