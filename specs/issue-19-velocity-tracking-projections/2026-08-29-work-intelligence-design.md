# Work Intelligence — Design Spec

**Issue:** Hortora/trellis#19
**Branch:** issue-19-velocity-tracking-projections
**Date:** 2026-08-29

## 1. Architecture Overview

Trellis work intelligence is a RAS-based detection system that evaluates worklog, enrichment, and cross-repo data to proactively surface stalled, forgotten, and unblocked work. The primary consumer is the LLM (via the model tree); a UI panel provides human visibility.

### Three layers

1. **Data adapters** — bridge trellis data sources into CloudEvents for the RAS engine. Event-driven by default (filesystem mtime watches, enrichment refresh triggers). Polling as fallback sweep only.

2. **Ganglion facets** — `JavaSwitchGanglion` subclasses, one per intelligence signal. Each evaluates CloudEvents and returns `DetectionResult` (DETECTED/WEAK/NOISE + confidence + evidence). Registered via `SituationDefinitionProvider` CDI beans. Four archetype facets ship initially, one per data source archetype.

3. **Presentation** — `WorkIntelligenceModelProvider` reads `ActiveSituation` records, maps confidence to severity tiers (INFO/ATTENTION/ACTION_NEEDED), assembles the `intelligence/` model subtree. REST endpoint serves the same data for the UI panel.

### Dependencies

- `casehub-ras-api` + `casehub-ras-runtime` — RAS framework (no Drools)
- Existing trellis services: `WorklogService`, enrichment scripts, `GenerationCounter`
- Existing SPI: `ModelProvider`

## 2. Data Adapters

Each adapter emits CloudEvents when its data source changes. Event-driven for local data; enrichment-refresh-driven for GitHub data (local Electron app has no webhook endpoint).

| Adapter | Source | CloudEvent type | Trigger |
|---------|--------|----------------|---------|
| `WorklogEventAdapter` | worklog.db `events` + `work_items` | `trellis.worklog.snapshot` | `GenerationCounter` mtime change |
| `EnrichmentAdapter` | enrichment.db (GitHub issue cache) | `trellis.enrichment.issue` | enrichment.db mtime change (triggered by `enrichment.py refresh` at work-start, or by fallback sweep) |
| `DeferredItemAdapter` | .plan files across workspaces | `trellis.deferred.item` | .plan file mtime change |
| `CrossRepoAdapter` | GitHub API via enrichment cache | `trellis.crossrepo.change` | Enrichment refresh (fallback: 15 min poll) |

Adapters reuse existing trellis services — `WorklogService` for worklog.db, enrichment cache for GitHub data. No new database connections.

A fallback sweep interval (`trellis.intelligence.poll-interval`, default 5 min) catches anything the event path missed. On-demand evaluation is supported — when `trellis_model` is called with the `intelligence` domain, adapters run immediately.

## 3. Ganglion Facets

Each facet is a `JavaSwitchGanglion` subclass — pure Java, no Drools. Registered via `SituationDefinitionProvider` CDI beans. Runtime enable/disable via RAS situation config.

### 3.1 StalledWorkGanglion (temporal archetype)

- **Listens for:** `trellis.worklog.snapshot`
- **Detects:** branches/issues where last event is > N days ago (configurable, default 7)
- **Confidence:** scales with staleness — 7 days = 0.4 (ATTENTION), 14+ days = 0.8 (ACTION_NEEDED)
- **Evidence:** branch name, issue number, last event type + timestamp, days idle

### 3.2 UnblockedWorkGanglion (state-transition archetype)

- **Listens for:** `trellis.enrichment.issue`
- **Detects:** issues with `blocked by #M` where #M is now CLOSED
- **Confidence:** 0.9 (ACTION_NEEDED) — an unblocked issue is always worth surfacing
- **Evidence:** unblocked issue number, blocker issue number + close date, how long it was blocked

### 3.3 ForgottenDeferralGanglion (deferred-queue archetype)

- **Listens for:** `trellis.deferred.item` + `trellis.enrichment.issue`
- **Detects:** deferred items whose reason references a blocker/condition that has been resolved
- **Confidence:** 0.7 (ATTENTION) if reason mentions a now-closed issue; 0.5 (ATTENTION) if deferral is > 14 days old with no re-evaluation
- **Evidence:** deferred title, original reason, blocker state, age

### 3.4 CrossRepoDependencyGanglion (cross-repo archetype)

- **Listens for:** `trellis.crossrepo.change`
- **Detects:** PRs merged in upstream repos that downstream repos haven't consumed
- **Confidence:** 0.5 (ATTENTION) for available-but-unconsumed; 0.8 (ACTION_NEEDED) if a trellis issue explicitly references the upstream change
- **Evidence:** upstream repo, PR number, downstream repo, related issue numbers

### 3.5 Adding new facets

To add a new intelligence signal:
1. Create a `JavaSwitchGanglion` subclass implementing `evaluate()`
2. Create a `SituationDefinitionProvider` CDI bean registering the situation + ganglion
3. If the data source archetype exists, use the existing adapter. If not, add an adapter that emits the new CloudEvent type.

No framework changes, no registry updates, no configuration beyond the situation definition.

## 4. Presentation Layer

### 4.1 WorkIntelligenceModelProvider

Single `ModelProvider` implementation, domain `intelligence`. Reads all `ActiveSituation` records from the RAS engine. Maps each to a JSON finding.

Confidence → severity mapping (applied at the model provider, not in ganglions):
- `< 0.4` → INFO (LLM: mention if asked)
- `0.4 – 0.7` → ATTENTION (LLM: surface proactively in what-next/work)
- `> 0.7` → ACTION_NEEDED (LLM: lead with this)

Model tree output:
```json
{
  "intelligence": {
    "summary": { "actionNeeded": 2, "attention": 3, "info": 1 },
    "findings": [
      {
        "facet": "unblocked",
        "severity": "ACTION_NEEDED",
        "subject": "#19 — Work Intelligence",
        "summary": "Blocker #11 closed 2026-08-15. Issue unblocked for 13 days.",
        "suggestion": "Consider picking up #19 — its blocker is resolved.",
        "confidence": 0.9,
        "detectedAt": "2026-08-28T10:00:00Z",
        "evidence": { "blockerIssue": "11", "blockerClosedAt": "2026-08-15" }
      }
    ]
  }
}
```

### 4.2 REST endpoint

`GET /api/intelligence?root=...` — returns the same JSON as the model provider. Delegates to `WorkIntelligenceModelProvider`.

### 4.3 UI panel

`trellis-intelligence-panel` — read-only Lit component. Renders findings grouped by severity:
- ACTION_NEEDED at top with visual emphasis
- ATTENTION below
- INFO collapsed by default

Uses `pages-data-table` with `fromRows()` for the findings list (matches existing panel patterns — memory panel is the reference for tabular data). Registered as a dock-bar panel in `workbench-panels.ts`.

### 4.4 MCP integration

`trellis_model` already serves ModelProvider outputs. Adding the `intelligence` domain means `trellis_model domain=intelligence` returns findings — zero additional MCP wiring. The LLM sees findings in the model tree and can reference them in `what-next` recommendations and `work` routing.

## 5. Decisions

See [decisions.md](decisions.md) for the full decision log. Summary:

| # | Decision |
|---|----------|
| D1 | Full loop — model provider + UI panel |
| D2 | Composable facet architecture, one per data source archetype |
| D3 | Scoped to trellis, GOAP via casehub-engine for future work ordering |
| D4 | Facets as RAS JavaSwitchGanglion subclasses, CDI-discovered, runtime enable/disable |
| D5 | ActiveSituation as finding model, severity mapped from confidence thresholds |

## 6. Trade-offs Acknowledged

- **D1:** More scope (service + panel) in one issue. The panel is a thin view of service output.
- **D2:** More upfront design for the facet framework. Individual facets are simpler because of it.
- **D3:** Other tools can't consume facets until extraction to soredium. Acceptable for pre-release.
- **D4:** Takes dependency on casehub-ras. May need RAS extensions for worklog data input. Extensions go back to RAS.
- **D5:** Trellis depends on RAS's ActiveSituation model. If RAS evolves it, trellis adapts.

## 7. Future Extensions

- **Estimate calibration facet** — compare scale labels (S/M/L) to actual cycle times
- **Epic health facet** — detect epics with declining throughput
- **GOAP-style work ordering** — casehub-engine's GOAP planner reasons about issue dependencies as preconditions, sequences work optimally
- **Cross-tool intelligence** — extract facet framework to soredium for garden engine, CLI skills

## References

- [WorklogService.java] — existing worklog data access layer
- [WorklogModelProvider.java] — existing ModelProvider SPI pattern
- [casehub-ras JavaSwitchGanglion] — pure Java ganglion base class
- [casehub-ras SituationDefinitionProvider] — CDI situation registration
- [casehub-ras ActiveSituation] — situation model with confidence + evidence
- [casehub-engine GoapDecompositionStrategy] — future GOAP integration
- [enrichment.py] — GitHub issue cache and what-next scoring
- [GenerationCounter] — mtime-based change detection
- Hortora/trellis#19 — focal issue
- Hortora/trellis#11 — predecessor (Critical Path + Recommendations Engine)
- Hortora/trellis#2 — parent epic
