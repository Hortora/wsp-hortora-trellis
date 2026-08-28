# Decisions — #19 Work Intelligence

## D1: Both LLM model provider and UI panel

**Choice:** Full loop — WorkIntelligenceModelProvider for LLM consumption + UI panel for human visibility. Both in this issue.
**Alternatives:**
- LLM-first only — service + model provider, panel deferred. Solves "I forget things" via LLM proactivity, but no visual overview.
**Rationale:** The LLM is the primary consumer for proactive surfacing, but a panel gives the human a glanceable overview of the same data.
**Trade-offs:** More scope in one issue. Acceptable — the panel is a thin read-only view of the service output.
**Sources:** Issue #19 description, existing ModelProvider SPI pattern
**Exploration:** quick
**Status:** captured

## D2: Composable facet architecture — one per data source archetype

**Choice:** Build a plugin-style facet framework where each intelligence signal is a facet implementing a common interface. Ship one facet per archetype (temporal, state-transition, deferred-queue, cross-repo) to prove the framework across all source patterns. Facets are individually enable/disable.
**Alternatives:**
- Build 2 specific signals (stalled + unblocked) without a framework — faster but each new signal is bespoke work
- Build all 6 signals without a framework — lots of code, hard to extend
**Rationale:** The value is in making it trivial to add the next signal. Architecture over features. Each archetype exercises a different data source pattern, so the framework is proven across all of them.
**Trade-offs:** More upfront design for the facet interface. The individual facets are simpler because of it.
**Sources:** worklog.db (temporal), enrichment.db (state-transition), .plan files (deferred), GitHub API (cross-repo)
**Exploration:** quick
**Status:** captured

## D3: Scoped to trellis sidecar, GOAP via casehub-engine

**Choice:** Facet framework lives in trellis sidecar. Future GOAP-style work planning leverages casehub-engine (which already provides GOAP planning). No extraction to soredium yet.
**Alternatives:**
- Build in soredium as shared infrastructure — premature, interface isn't stable yet
**Rationale:** Pre-release, prove the pattern in one place. Trellis already has WorklogService, ModelProvider SPI, enrichment scripts. casehub-engine already has GOAP — no need to rebuild.
**Trade-offs:** Other tools can't consume facets until extraction. Acceptable for pre-release.
**Sources:** casehub-engine GOAP (RoutingStrategy, GoapDecompositionStrategy), issue #48 (claudony consolidation)
**Exploration:** quick
**Status:** captured
