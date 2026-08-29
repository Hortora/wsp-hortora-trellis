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

## D4: Facets implemented as RAS JavaSwitchGanglion subclasses

**Choice:** Each intelligence facet is a `JavaSwitchGanglion` subclass discovered via `SituationDefinitionProvider` CDI beans. RAS handles detection dispatch, correlation, and situation lifecycle. Trellis adds `casehub-ras-api` + `casehub-ras-runtime` as dependencies. No Drools. Runtime enable/disable via RAS situation config.
**Alternatives:**
- Custom `IntelligenceFacet` interface + CDI discovery — works but reinvents what RAS already provides (detection, correlation, evidence tracking, situation lifecycle)
- Drools-based RAS rules — too heavy for this use case, adds DRL file management
- `@IfBuildProperty` gating — build-time only, can't toggle at runtime
**Rationale:** RAS is casehub platform infrastructure. Trellis should consume it, not duplicate it. `JavaSwitchGanglion` is pure Java with no Drools dependency — lightweight enough for trellis. Gets correlation, evidence, and situation lifecycle for free.
**Trade-offs:** Takes a dependency on casehub-ras. Acceptable — trellis already depends on casehub platform modules. May need to extend RAS if the worklog data input pattern (polling a DB vs receiving CloudEvents) isn't supported — extensions go back to RAS, not bespoke in trellis.
**Depends on:** D2 (facet architecture)
**Sources:** casehub-ras JavaSwitchGanglion, SituationDefinitionProvider, RasEngine
**Exploration:** deep-analysis
**Status:** revised (replaced custom IntelligenceFacet with RAS consumption)

## D5: ActiveSituation as the finding model, severity mapped from confidence

**Choice:** Use RAS `ActiveSituation` (correlationKey, confidence, evidence, timestamps, triggerCount) as the finding model. Trellis maps `DetectionSignal` + confidence thresholds to three LLM behaviour tiers at the ModelProvider layer: confidence < 0.4 → INFO (mention if asked), 0.4–0.7 → ATTENTION (surface proactively), > 0.7 → ACTION_NEEDED (lead with this).
**Alternatives:**
- Custom `Finding` record — works but duplicates RAS's situation model (confidence, evidence, timestamps all exist in ActiveSituation)
- Direct DetectionSignal exposure to LLM — NOISE/ANTI/WEAK/DETECTED is an internal vocabulary; the LLM needs behaviour guidance, not signal classification
**Rationale:** ActiveSituation carries richer data than the proposed Finding (correlation, trigger count, evidence map). The severity mapping is a thin presentation layer in the ModelProvider — one method, testable. Avoids a parallel type hierarchy.
**Trade-offs:** Trellis depends on RAS's situation model. If RAS changes ActiveSituation, trellis adapts. Acceptable — trellis is a platform consumer, not independent.
**Depends on:** D4 (RAS adoption)
**Sources:** casehub-ras ActiveSituation, DetectionSignal, DetectionResult
**Exploration:** deep-analysis
**Status:** revised (replaced custom Finding with RAS ActiveSituation)
