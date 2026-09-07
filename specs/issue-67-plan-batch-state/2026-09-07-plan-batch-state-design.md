# Plan Batch State in Slot Detail Modal

**Issue:** Hortora/trellis#67
**Date:** 2026-09-07

## Overview

Show `.plan` batch state and progress in the slot detail modal sidebar. When a slot has a `.plan` file, the sidebar displays the full queue tree: batches with titles, items with done/active/pending states, and an overall progress summary.

## Architecture

### Backend — Plan parsing in WorkspaceScanner

Three new records in the `scanner` package:

```java
record PlanItem(String ref, String title, boolean done, boolean active) {}

record PlanBatch(String name, List<PlanItem> items) {}

record PlanProgress(
    List<PlanBatch> batches,
    String activeIssue,
    int completed,
    int total
) {}
```

**`PlanItem`** — a single issue in the queue. `ref` is the full issue reference (e.g., `casehubio/engine#1043`). `done` maps to `[x]`, `active` maps to `← active` marker.

**`PlanBatch`** — a group of items under a `### Batch N — title` header. Plans without batches have a single unnamed batch containing all items.

**`PlanProgress`** — top-level plan state. `completed` and `total` count across all batches. `activeIssue` is the ref of the item marked `← active`.

### SlotInfo extension

Add an optional `PlanProgress planProgress` field to `SlotInfo`:

```java
public record SlotInfo(
    int number, Path path, String issue, SlotStatus status,
    boolean isEpic, List<String> repos, String slug, String title,
    String description, String whatToDo, List<Integer> covers,
    PlanProgress planProgress  // nullable — null when no .plan exists
) {}
```

### Parser — WorkspaceScanner.parsePlanFile()

New private method called from `scanSlots()` after `parseSlotFile()` succeeds. Reads `slots/{N}/.plan` if present.

Parsing rules (matching plan_manager.py format):
- Lines matching `- [x] <ref> — <title>` → done item
- Lines matching `- [ ] <ref> — <title>` → pending item
- `← active` suffix on any item → marks that item active
- `### Batch N — <title>` headers → batch boundaries
- Lines outside `## Queue` section → ignored
- Epic markers (`(epic)`) → stripped from title, not tracked (epics are parent containers in the queue, but the plan parser only extracts leaf items that appear as checkable lines)
- Indented items (children of epics) → treated as regular items (flattened for display)

Items appearing before any batch header go into a default unnamed batch. This handles plans with no batch structure (most common case — single-issue plans).

### FileWatcher integration

`.plan` files live at `slots/{N}/.plan`. The existing `PathDomainClassifier` maps `slots/` changes to the `SLOTS` domain, which triggers `workspace:slots` SSE broadcasts. The slot-detail component already subscribes to `workspace:slots` events. No new watcher wiring needed — `.plan` changes will trigger slot re-scans automatically.

### JSON serialization

Jackson serializes `SlotInfo` with a `planProgress` field. When null (no `.plan`), the field is omitted from JSON via `@JsonInclude(NON_NULL)` on the record or Quarkus default null-exclusion config. Frontend checks for presence.

## Frontend — Slot detail sidebar

### TypeScript interface

```typescript
interface PlanItem {
  ref: string;
  title: string;
  done: boolean;
  active: boolean;
}

interface PlanBatch {
  name: string | null;  // null for the default unnamed batch
  items: PlanItem[];
}

interface PlanProgress {
  batches: PlanBatch[];
  activeIssue: string | null;
  completed: number;
  total: number;
}
```

Add `planProgress?: PlanProgress` to the existing `SlotInfo` interface in `slot-detail.ts`.

### Rendering

New `_renderPlan()` method, called from `_renderSidebar()` between the "Issues" section and "Repos" section. Only rendered when `this._slot?.planProgress` exists.

Layout:
```
PLAN                          ← section header (h3, matches existing style)
Batch 1 — Shared contract     ← batch header (dimmer, smaller)
  ✓ engine#1043 throttle      ← done item (green check, strikethrough-ish dim)
  ✓ engine#1044 bridge        ← done item
Batch 2 — Session-level       ← batch header
  ● claudony#203 session      ← active item (blue dot, bright text)
Batch 3 — App wiring          ← batch header
  ○ devtown#185 resilience    ← pending item (dim circle, dim text)
──────────────────────────     ← thin separator
2/4 done · Batch 2 of 3       ← progress summary
```

Styling:
- Done items: muted text, `✓` in green (`#86efac`)
- Active item: bright text, `●` in blue (`#93c5fd`), subtle left-border highlight
- Pending items: dim text (`#666`), `○` outline
- Batch headers: `0.75rem`, uppercase, `#777`, with top margin between batches
- Progress summary: `0.75rem`, `#888`, separated by thin border-top
- Single-batch plans (no batch headers): items render flat, no batch title shown, progress summary still appears
- Issue refs truncated to `repo#N` for display (strip org prefix)

## Edge Cases

- **No `.plan` file**: `planProgress` is null, section not rendered. Most common for non-epic slots.
- **Empty `.plan`** (no queue items): `planProgress` is null (parser returns null when total is 0).
- **Plan with no batches**: Single unnamed batch, no batch headers rendered. Just a flat item list with progress summary.
- **Plan with only completed items**: All items show as done, progress shows "N/N done".
- **Plan with epic nesting**: Indented child items parsed as regular items. The tree is flattened for sidebar display — batch grouping provides enough structure without reproducing the full epic hierarchy.

## Testing

### Java unit test

`WorkspaceScannerTest` — test `parsePlanFile()` with:
- Multi-batch plan (like slot 174's format)
- Single-item plan (no batches)
- Plan with active marker
- Plan with all items done
- No `.plan` file (returns null)
- Malformed `.plan` (returns null gracefully)

### Frontend

No unit tests for rendering (Lit template). Verify visually via dev server — the slot detail modal is opened from the org dashboard by clicking a slot card.

## References

- `WorkspaceScanner.java:68` — `scanSlots()` pattern for reading per-slot files
- `EpicInfo.java` — precedent for progress tracking records in scanner package
- `PlanState.java` — existing minimal plan model (active, completed, total)
- `slot-detail.ts:243` — `_renderSidebar()` structure and styling conventions
- `slots/174/.plan` — real-world multi-batch plan example
- `plan_manager.py:263` — authoritative `.plan` format parser (batch regex, item regex, active marker)
