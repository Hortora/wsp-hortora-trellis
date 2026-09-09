# Plan Batch State Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #67 — Slot detail modal: show .plan batch state and progress
**Issue group:** #67

**Goal:** Parse `.plan` files from slot directories and display batch progress in the slot detail modal sidebar.

**Architecture:** Add `PlanProgress`, `PlanBatch`, `PlanItem` records to the scanner package. Extend `WorkspaceScanner.scanSlots()` to read `.plan` files alongside `.slot` files, attaching parsed plan state to `SlotInfo`. Render the batch tree in a new sidebar section in `slot-detail.ts`.

**Tech Stack:** Java 21 records, Quarkus 3.x, Lit (TypeScript), JUnit 5

## Global Constraints

- Java 21 — records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Frontend: Lit + esbuild, `casehub-dark` theme
- No new dependencies — parsing is regex-based, matching plan_manager.py format
- `.plan` files live at `slots/{N}/.plan` (sibling to `.slot`)

---

## Batch 1: Backend — Plan parsing and SlotInfo extension

### Task 1: Create PlanProgress, PlanBatch, PlanItem records and plan parser with tests

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/PlanItem.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/PlanBatch.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/PlanProgress.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/SlotInfo.java:6`
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java:68-101`
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java:118-121` (scanAttic)
- Modify: `sidecar/src/test/java/io/hortora/trellis/garden/ProvenanceEnricherTest.java:21-22`
- Test: `sidecar/src/test/java/io/hortora/trellis/scanner/WorkspaceScannerTest.java`

**Interfaces:**
- Produces: `PlanItem(String ref, String title, boolean done, boolean active)`
- Produces: `PlanBatch(String name, List<PlanItem> items)`
- Produces: `PlanProgress(List<PlanBatch> batches, String activeIssue, int completed, int total)`
- Produces: `SlotInfo.planProgress()` — nullable `PlanProgress`
- Produces: `WorkspaceScanner.parsePlanFile(Path planFile)` — returns `PlanProgress` or null

- [ ] **Step 1: Write failing tests for plan parsing — multi-batch plan**

Add to `WorkspaceScannerTest.java`:

```java
@Test
void scanParsesMultiBatchPlan() throws IOException {
    createSlot(5, """
            # Slot 5
            slug: issue-468-throttle

            ## Issue
            casehubio/parent#468

            ## Repos
            - engine
            """);
    Files.writeString(root.resolve("slots/5/.plan"), """
            # Work Plan — issue-468

            ## State
            state: active

            ## Queue
            - [ ] casehubio/parent#468 — Foundation Throttle (epic)
              ### Batch 1 — Shared contract
              - [x] casehubio/engine#1043 — Concurrency throttle
              - [ ] casehubio/engine#1044 — Watchdog bridge ← active
              ### Batch 2 — Session-level
              - [ ] casehubio/claudony#203 — Session throttle
            """);

    var model = scanner.scan(root);

    var slot = model.slots().getFirst();
    assertNotNull(slot.planProgress());
    var plan = slot.planProgress();
    assertEquals(2, plan.batches().size());
    assertEquals("Batch 1 — Shared contract", plan.batches().get(0).name());
    assertEquals(2, plan.batches().get(0).items().size());
    assertTrue(plan.batches().get(0).items().get(0).done());
    assertFalse(plan.batches().get(0).items().get(1).done());
    assertTrue(plan.batches().get(0).items().get(1).active());
    assertEquals("casehubio/engine#1044", plan.batches().get(0).items().get(1).ref());
    assertEquals("Watchdog bridge", plan.batches().get(0).items().get(1).title());
    assertEquals("Batch 2 — Session-level", plan.batches().get(1).name());
    assertEquals(1, plan.batches().get(1).items().size());
    assertEquals("casehubio/engine#1044", plan.activeIssue());
    assertEquals(1, plan.completed());
    assertEquals(3, plan.total());
}
```

- [ ] **Step 2: Write failing test — single-item plan (no batches)**

```java
@Test
void scanParsesPlanWithNoBatches() throws IOException {
    createSlot(3, """
            # Slot 3

            ## Issue
            org/repo#100

            ## Repos
            - engine
            """);
    Files.writeString(root.resolve("slots/3/.plan"), """
            # Work Plan — issue-100

            ## State
            state: active

            ## Queue
            - [ ] casehubio/engine#189 — Normative interop ← active
            """);

    var model = scanner.scan(root);

    var plan = model.slots().getFirst().planProgress();
    assertNotNull(plan);
    assertEquals(1, plan.batches().size());
    assertNull(plan.batches().getFirst().name());
    assertEquals(1, plan.batches().getFirst().items().size());
    assertTrue(plan.batches().getFirst().items().getFirst().active());
    assertEquals(0, plan.completed());
    assertEquals(1, plan.total());
}
```

- [ ] **Step 3: Write failing test — no .plan file**

```java
@Test
void scanReturnsNullPlanWhenNoPlanFile() throws IOException {
    createSlot(7, """
            # Slot 7

            ## Issue
            org/repo#50

            ## Repos
            - engine
            """);

    var model = scanner.scan(root);

    assertNull(model.slots().getFirst().planProgress());
}
```

- [ ] **Step 4: Write failing test — all items done**

```java
@Test
void scanParsesPlanWithAllItemsDone() throws IOException {
    createSlot(4, """
            # Slot 4

            ## Issue
            org/repo#30

            ## Repos
            - engine
            """);
    Files.writeString(root.resolve("slots/4/.plan"), """
            # Work Plan

            ## Queue
            - [x] org/repo#31 — First
            - [x] org/repo#32 — Second
            """);

    var model = scanner.scan(root);

    var plan = model.slots().getFirst().planProgress();
    assertNotNull(plan);
    assertNull(plan.activeIssue());
    assertEquals(2, plan.completed());
    assertEquals(2, plan.total());
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorkspaceScannerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation errors — `PlanProgress`, `PlanBatch`, `PlanItem` do not exist yet

- [ ] **Step 6: Create PlanItem record**

Create `sidecar/src/main/java/io/hortora/trellis/scanner/PlanItem.java`:

```java
package io.hortora.trellis.scanner;

public record PlanItem(String ref, String title, boolean done, boolean active) {}
```

- [ ] **Step 7: Create PlanBatch record**

Create `sidecar/src/main/java/io/hortora/trellis/scanner/PlanBatch.java`:

```java
package io.hortora.trellis.scanner;

import java.util.List;

public record PlanBatch(String name, List<PlanItem> items) {}
```

- [ ] **Step 8: Create PlanProgress record**

Create `sidecar/src/main/java/io/hortora/trellis/scanner/PlanProgress.java`:

```java
package io.hortora.trellis.scanner;

import java.util.List;

public record PlanProgress(List<PlanBatch> batches, String activeIssue, int completed, int total) {}
```

- [ ] **Step 9: Add planProgress field to SlotInfo**

Modify `SlotInfo.java` — add `PlanProgress planProgress` as the last parameter:

```java
public record SlotInfo(
        int number,
        Path path,
        String issue,
        SlotStatus status,
        boolean isEpic,
        List<String> repos,
        String slug,
        String title,
        String description,
        String whatToDo,
        List<Integer> covers,
        PlanProgress planProgress
) {}
```

- [ ] **Step 10: Fix all SlotInfo constructor call sites**

Three sites need the new `planProgress` parameter appended:

`WorkspaceScanner.java:250` (parseSlotFile return) — pass `null` for now:
```java
return new SlotInfo(number, slotDir, issue, status, isEpic, List.copyOf(repos), slug, title, description, whatToDo, List.copyOf(covers), null);
```

`WorkspaceScanner.java:120-121` (scanAttic) — pass `null`:
```java
slots.add(new SlotInfo(info.number(), info.path(), info.issue(),
        SlotStatus.ARCHIVED, info.isEpic(), info.repos(), info.slug(), info.title(), info.description(), info.whatToDo(), info.covers(), null));
```

`ProvenanceEnricherTest.java:21-22` — append `null`:
```java
var slot = new SlotInfo(3, Path.of("/tmp/slot-3"), "Hortora/trellis#14",
        SlotStatus.ACTIVE, false, List.of("trellis", "engine"), "issue-14-some-feature", "Some Feature", "Feature description", "What to do text", List.of(14), null);
```

- [ ] **Step 11: Implement parsePlanFile in WorkspaceScanner**

Add these patterns near the top of `WorkspaceScanner` (after existing patterns):

```java
private static final Pattern PLAN_ITEM_PATTERN = Pattern.compile(
        "^\\s*- \\[([ x])]\\s+([\\w-]+/[\\w-]+#\\d+)\\s+—\\s+(.+?)(?:\\s+←\\s+active)?\\s*$");
private static final Pattern PLAN_BATCH_PATTERN = Pattern.compile(
        "^\\s*###\\s+(.+?)(?:\\s+←\\s+current)?\\s*$");
```

Add the parser method:

```java
PlanProgress parsePlanFile(Path planFile) {
    if (!Files.isRegularFile(planFile)) return null;
    try {
        var lines = Files.readAllLines(planFile);
        var batches = new ArrayList<PlanBatch>();
        var currentItems = new ArrayList<PlanItem>();
        String currentBatchName = null;
        String activeIssue = null;
        int completed = 0;
        int total = 0;
        boolean inQueue = false;

        for (String line : lines) {
            String trimmed = line.trim();
            if (trimmed.equals("## Queue")) { inQueue = true; continue; }
            if (inQueue && trimmed.startsWith("## ")) break;
            if (!inQueue) continue;

            Matcher batchMatcher = PLAN_BATCH_PATTERN.matcher(trimmed);
            if (batchMatcher.matches()) {
                if (!currentItems.isEmpty() || currentBatchName != null) {
                    batches.add(new PlanBatch(currentBatchName, List.copyOf(currentItems)));
                    currentItems.clear();
                }
                currentBatchName = batchMatcher.group(1).trim();
                continue;
            }

            Matcher itemMatcher = PLAN_ITEM_PATTERN.matcher(trimmed);
            if (itemMatcher.matches()) {
                boolean done = "x".equals(itemMatcher.group(1));
                String ref = itemMatcher.group(2);
                String title = itemMatcher.group(3).replaceAll("\\(epic\\)", "").trim();
                boolean active = trimmed.contains("← active");
                currentItems.add(new PlanItem(ref, title, done, active));
                if (done) completed++;
                total++;
                if (active) activeIssue = ref;
            }
        }

        if (!currentItems.isEmpty() || currentBatchName != null) {
            batches.add(new PlanBatch(currentBatchName, List.copyOf(currentItems)));
        }

        if (total == 0) return null;
        return new PlanProgress(List.copyOf(batches), activeIssue, completed, total);
    } catch (IOException e) {
        LOG.warnf(e, "Failed to read plan file: %s", planFile);
        return null;
    }
}
```

- [ ] **Step 12: Wire parsePlanFile into scanSlots**

In `scanSlots()`, after `SlotInfo info = parseSlotFile(...)` succeeds (line ~93-94), add the plan parse:

```java
SlotInfo info = parseSlotFile(slotFile, slotDir, number);
if (info != null) {
    PlanProgress plan = parsePlanFile(slotDir.resolve(".plan"));
    if (plan != null) {
        info = new SlotInfo(info.number(), info.path(), info.issue(), info.status(),
                info.isEpic(), info.repos(), info.slug(), info.title(),
                info.description(), info.whatToDo(), info.covers(), plan);
    }
    slots.add(info);
}
```

- [ ] **Step 13: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorkspaceScannerTest`
Expected: all tests PASS including the 4 new plan tests

- [ ] **Step 14: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: all tests PASS (including ProvenanceEnricherTest with updated constructor)

- [ ] **Step 15: Commit**

```bash
git -C <PROJECT> add sidecar/src/main/java/io/hortora/trellis/scanner/PlanItem.java sidecar/src/main/java/io/hortora/trellis/scanner/PlanBatch.java sidecar/src/main/java/io/hortora/trellis/scanner/PlanProgress.java sidecar/src/main/java/io/hortora/trellis/scanner/SlotInfo.java sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java sidecar/src/test/java/io/hortora/trellis/scanner/WorkspaceScannerTest.java sidecar/src/test/java/io/hortora/trellis/garden/ProvenanceEnricherTest.java
git -C <PROJECT> commit -m "feat(#67): parse .plan files from slot directories into PlanProgress model Refs #67"
```

## Batch 2: Frontend — Plan tree rendering in slot detail sidebar

### Task 2: Render plan batch tree in slot-detail sidebar

**Files:**
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts:7-17` (add TS interfaces)
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts:62-123` (add CSS)
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts:243-306` (_renderSidebar, add _renderPlan)

**Interfaces:**
- Consumes: `PlanProgress`, `PlanBatch`, `PlanItem` from `/api/workspace` JSON response via `SlotInfo.planProgress`

- [ ] **Step 1: Add TypeScript interfaces**

Add after the existing `AgentSnapshot` interface (around line 43) in `slot-detail.ts`:

```typescript
interface PlanItem {
  ref: string;
  title: string;
  done: boolean;
  active: boolean;
}

interface PlanBatch {
  name: string | null;
  items: PlanItem[];
}

interface PlanProgress {
  batches: PlanBatch[];
  activeIssue: string | null;
  completed: number;
  total: number;
}
```

Add `planProgress?: PlanProgress` to the `SlotInfo` interface (after `covers`):

```typescript
interface SlotInfo {
  number: number;
  path: string;
  issue: string;
  status: string;
  isEpic: boolean;
  repos: string[];
  slug: string | null;
  title: string | null;
  covers: number[];
  planProgress?: PlanProgress;
}
```

- [ ] **Step 2: Add CSS for plan section**

Add inside the `static override styles` block:

```css
.plan-section { margin-bottom: 1.5rem; }
.plan-batch { margin-bottom: 0.75rem; }
.plan-batch-header {
  font-size: 0.7rem; color: #777; text-transform: uppercase;
  letter-spacing: 0.03em; margin-bottom: 0.3rem; margin-top: 0.5rem;
}
.plan-item {
  display: flex; align-items: baseline; gap: 0.4rem;
  font-size: 0.8rem; padding: 0.1rem 0;
}
.plan-item-done { color: #666; }
.plan-item-active { color: #e5e5e5; }
.plan-item-pending { color: #555; }
.plan-icon-done { color: #86efac; }
.plan-icon-active { color: #93c5fd; }
.plan-icon-pending { color: #555; }
.plan-ref { font-family: monospace; font-size: 0.7rem; flex-shrink: 0; }
.plan-title { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.plan-summary {
  font-size: 0.75rem; color: #888; padding-top: 0.5rem;
  margin-top: 0.5rem; border-top: 1px solid #333;
}
```

- [ ] **Step 3: Add _renderPlan method**

Add after `_renderSidebar()`:

```typescript
private _renderPlan() {
  const plan = this._slot?.planProgress;
  if (!plan) return nothing;

  const currentBatchIdx = plan.batches.findIndex(b =>
    b.items.some(i => i.active));
  const totalBatches = plan.batches.filter(b => b.items.length > 0).length;

  const shortRef = (ref: string) => {
    const parts = ref.split('/');
    return parts.length > 1 ? parts[parts.length - 1] : ref;
  };

  return html`
    <div class="sidebar-section plan-section">
      <h3>Plan</h3>
      ${plan.batches.map(batch => html`
        <div class="plan-batch">
          ${batch.name ? html`<div class="plan-batch-header">${batch.name}</div>` : nothing}
          ${batch.items.map(item => html`
            <div class="plan-item ${item.done ? 'plan-item-done' : item.active ? 'plan-item-active' : 'plan-item-pending'}">
              <span class="${item.done ? 'plan-icon-done' : item.active ? 'plan-icon-active' : 'plan-icon-pending'}">
                ${item.done ? '✓' : item.active ? '●' : '○'}
              </span>
              <span class="plan-ref">${shortRef(item.ref)}</span>
              <span class="plan-title">${item.title}</span>
            </div>
          `)}
        </div>
      `)}
      <div class="plan-summary">
        ${plan.completed}/${plan.total} done${totalBatches > 1
          ? ` · Batch ${currentBatchIdx >= 0 ? currentBatchIdx + 1 : totalBatches} of ${totalBatches}`
          : ''}
      </div>
    </div>
  `;
}
```

- [ ] **Step 4: Wire _renderPlan into _renderSidebar**

In `_renderSidebar()`, insert `${this._renderPlan()}` between the "Issues" section and the "Repos" section:

```typescript
// After the Issues sidebar-section closing </div>
${this._renderPlan()}
// Before the Repos sidebar-section
```

- [ ] **Step 5: Build and verify**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: build succeeds with no TypeScript errors

- [ ] **Step 6: Compile full project**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml compile`
Expected: compiles successfully (Quinoa picks up frontend build)

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add sidecar/src/main/webui/src/views/slot-detail.ts
git -C <PROJECT> commit -m "feat(#67): render plan batch tree in slot detail sidebar Refs #67"
```

## References

- [specs/issue-67-plan-batch-state/2026-09-07-plan-batch-state-design.md] — design spec
- [WorkspaceScanner.java:68] — scanSlots pattern
- [WorkspaceScanner.java:161] — parseSlotFile pattern
- [WorkspaceScannerTest.java] — existing test patterns and helpers
- [slot-detail.ts:243] — _renderSidebar structure
- [EpicInfo.java] — precedent for progress records
- [PlanState.java] — existing minimal plan model
- [plan_manager.py:263] — authoritative .plan format
- [slots/174/.plan] — real multi-batch .plan example
- [GitHub #67] — focal issue
