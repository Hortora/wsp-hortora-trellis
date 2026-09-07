## D1: Plan data model

**Choice:** Rich plan record with batch grouping — new `PlanProgress`, `PlanBatch`, `PlanItem` records in the scanner package. `WorkspaceScanner.scanSlots()` reads `.plan` alongside `.slot` and attaches to `SlotInfo`.
**Alternatives:**
- Reuse WorklogService.planPosition() — couples scanner to worklog concern, different file paths
- Flat JSON with no structured records — duplicates parsing in TypeScript, fragile
**Rationale:** Follows the established scanner pattern (EpicInfo, PauseEntry). Keeps parsing in one place. Frontend gets clean structured data. Batch-grouping record matches the `.plan` format directly.
**Trade-offs:** Adds ~3 records and ~60 lines of parsing to the scanner package.
**Sources:** WorkspaceScanner.java:68 (scanSlots pattern), PlanState.java (existing minimal model), EpicInfo.java (precedent for progress records), plan_manager.py (authoritative .plan format)
**Exploration:** quick
**Status:** captured

## D2: Frontend rendering location

**Choice:** New sidebar section with inline tree in `_renderSidebar()`, between "Issues" and "Repos". Batches as section headers, items as lists with done/active/pending states, compact progress summary at bottom.
**Alternatives:**
- Separate collapsible panel above terminal area — competes with terminal real estate
- Tooltip/flyout on issue badge — hidden, not discoverable
**Rationale:** Sidebar is the metadata pane — plan progress is slot metadata. Slots rarely have more than 5-8 items across 2-3 batches, so vertical space is not a concern. Consistent with existing sidebar pattern. No new components needed.
**Trade-offs:** Long queues could push terminals section down, but this is unlikely in practice.
**Sources:** slot-detail.ts:243 (_renderSidebar structure), slot 174/.plan (real-world batch example with 4 items across 3 batches)
**Exploration:** quick
**Status:** captured
