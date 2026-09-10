# Handoff — trellis

## Last Session

Designed and implemented the Tamboui REPL for mechanical work lifecycle (#75). The core insight: the work lifecycle (start/pause/resume/end/next) doesn't need an LLM — it's state machine transitions. The REPL handles those mechanically; the LLM augments with sweeps, squashes, and reviews when opted into.

Built `repl/` Maven module — lightweight Java CLI (Tamboui 0.4.0, no Quarkus). YAML-defined command tree with four namespaces (work/git/project/llm), tab completion, GoalRunner for Q&A guided flows, SorediumBridge calling Python commands via JSON Lines subprocess protocol, SidecarClient for REST API + SSE agent state. 32 tests. Also built `cli/` JSON Lines wrapper in soredium. Landed as single squashed commit on main.

Filed #76 for ARIA compliance across all Trellis frontend components (zero `role`/`aria-*` attributes currently — blocks pages tutorial system integration).

## Immediate Next Step

Continue #75 — three pieces remain: SuggestionInput widget (Tamboui text+click composition), modal view integration (frontend split/tab in repo/slot detail), sidecar REPL terminal spawning. Blog entry on branch is partial — continuation session should extend it.

## Cross-Module

- Soredium `cli/` wrapper committed on `issue-353-write-marker-step` branch — needs merging to main.

## References

- Design spec: `docs/specs/issue-75-tamboui-repl/2026-09-10-tamboui-repl-design.md`
- Decisions: `docs/specs/issue-75-tamboui-repl/decisions.md`
- Blog: `blog/2026-09-10-mdp01-when-the-repl-stops-needing-permission.md`
