# Slot Status Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #69 — Slot status field parse and display active-paused
**Issue group:** #69, #74

**Goal:** Enrich SlotStatus with all worklog.db lifecycle states and parse paused status from .slot files, so the dashboard accurately reflects slot lifecycle.

**Architecture:** Extend `SlotStatus` enum with 5 new values, teach `WorkspaceScanner` to parse `.slot` `## Status` section and merge worklog.db slot states by priority. Update dashboard badge colors for new states and add status filter pills.

**Tech Stack:** Java 21, TypeScript (Lit), SQLite (worklog.db via WorklogService)

## Global Constraints

- Java 21 records, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- Use `ide_insert_member` / `ide_replace_member` for Java edits
- WorkspaceScanner is `@ApplicationScoped` — CDI injection available

---

## Batch 1: Backend — enum + scanner enrichment

### Task 1: Extend SlotStatus and enrich WorkspaceScanner

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/SlotStatus.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java`
- Modify: `sidecar/src/test/java/io/hortora/trellis/scanner/WorkspaceScannerTest.java`

**Interfaces:**
- Consumes: `WorklogService.slotStatus(String familyRoot) → List<WorklogService.SlotInfo>` (existing)
- Produces: `SlotStatus` enum with values: `ACTIVE`, `PAUSED`, `PENDING`, `READY_TO_LAND`, `ARCHIVING`, `ARCHIVED`, `FAILED`, `PURGED`. `WorkspaceScanner.scan()` returns `WorkspaceModel` with enriched slot statuses.

- [ ] **Step 1: Write failing tests for paused status parsing and worklog merge**

Add to `WorkspaceScannerTest.java`:

```java
@Test
void parsesSlotWithPausedStatus(@TempDir Path root) throws IOException {
    var slotDir = root.resolve("slots/1");
    Files.createDirectories(slotDir);
    Files.writeString(slotDir.resolve(".slot"), """
        # Slot 1
        slug: test-slot
        title: Test slot

        ## Issue
        org/repo#1

        ## Status
        status: paused

        ## Repos
        - repo-a
        """);
    initGitRepo(root.resolve("repo-a"));
    var model = scanner.scan(root);
    var slot = model.slots().stream().filter(s -> s.number() == 1).findFirst().orElseThrow();
    assertEquals(SlotStatus.PAUSED, slot.status());
}

@Test
void defaultsToActiveWhenNoStatusSection(@TempDir Path root) throws IOException {
    var slotDir = root.resolve("slots/1");
    Files.createDirectories(slotDir);
    Files.writeString(slotDir.resolve(".slot"), """
        # Slot 1
        slug: test-slot
        title: Test slot

        ## Issue
        org/repo#1

        ## Repos
        - repo-a
        """);
    initGitRepo(root.resolve("repo-a"));
    var model = scanner.scan(root);
    var slot = model.slots().stream().filter(s -> s.number() == 1).findFirst().orElseThrow();
    assertEquals(SlotStatus.ACTIVE, slot.status());
}
```

Note: `WorkspaceScannerTest` is a `@QuarkusTest` — the scanner is `@Inject`ed. `initGitRepo` is a helper that creates a `.git/HEAD` file.

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorkspaceScannerTest`
Expected: FAIL — `PAUSED` not in `SlotStatus` enum

- [ ] **Step 3: Extend SlotStatus enum**

Replace `SlotStatus.java` contents:

```java
package io.hortora.trellis.scanner;

public enum SlotStatus {
    ACTIVE,
    PAUSED,
    PENDING,
    READY_TO_LAND,
    ARCHIVING,
    ARCHIVED,
    FAILED,
    PURGED
}
```

- [ ] **Step 4: Parse ## Status section in WorkspaceScanner.parseSlotFile()**

In `WorkspaceScanner.parseSlotFile()`, add parsing for `## Status` section.
In the `switch (currentSection)` block, add a case:

```java
case "## Status" -> {
    if (trimmed.startsWith("status:")) {
        String statusVal = trimmed.substring("status:".length()).trim();
        if ("paused".equalsIgnoreCase(statusVal)) {
            slotPaused = true;
        }
    }
}
```

Declare `boolean slotPaused = false;` alongside other local variables.

Update the status resolution at the end of `parseSlotFile()`:

```java
boolean readyToLand = Files.exists(slotDir.resolve(".phase-a-complete"));
SlotStatus status;
if (readyToLand) {
    status = SlotStatus.READY_TO_LAND;
} else if (slotPaused) {
    status = SlotStatus.PAUSED;
} else {
    status = SlotStatus.ACTIVE;
}
```

- [ ] **Step 5: Add WorklogService dependency and merge worklog states**

Add `@Inject WorklogService worklogService;` to `WorkspaceScanner`.

Add import: `import io.hortora.trellis.worklog.WorklogService;`

Add a new method to merge worklog states after scanning:

```java
private List<SlotInfo> mergeWorklogStatus(Path root, List<SlotInfo> scannedSlots) {
    var worklogSlots = worklogService.slotStatus(root.toString());
    if (worklogSlots.isEmpty()) return scannedSlots;

    var worklogByNumber = new java.util.HashMap<Integer, String>();
    for (var ws : worklogSlots) {
        worklogByNumber.put(ws.slotNumber(), ws.state());
    }

    return scannedSlots.stream().map(slot -> {
        String wlState = worklogByNumber.get(slot.number());
        if (wlState == null || slot.status() == SlotStatus.ARCHIVED) return slot;

        SlotStatus resolved = switch (wlState) {
            case "archiving" -> SlotStatus.ARCHIVING;
            case "failed" -> SlotStatus.FAILED;
            case "pending" -> SlotStatus.PENDING;
            case "purged" -> SlotStatus.PURGED;
            default -> slot.status();
        };
        if (resolved != slot.status()) {
            return new SlotInfo(slot.number(), slot.path(), slot.issue(),
                    resolved, slot.isEpic(), slot.repos(), slot.slug(),
                    slot.title(), slot.description(), slot.whatToDo(), slot.covers());
        }
        return slot;
    }).toList();
}
```

Call it in `scan()` after `scanSlots()`:

```java
var slots = mergeWorklogStatus(root, scanSlots(root));
```

Replace the existing `var slots = scanSlots(root);` line.

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorkspaceScannerTest`
Expected: all tests PASS

- [ ] **Step 7: Run full backend test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: no new failures (pre-existing failures acceptable)

- [ ] **Step 8: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/scanner/SlotStatus.java sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java sidecar/src/test/java/io/hortora/trellis/scanner/WorkspaceScannerTest.java
git commit -m "feat(#69): extend SlotStatus with worklog lifecycle states and paused parsing Refs #69 Refs #74"
```

---

## Batch 2: Frontend — badge colors + status filter

### Task 2: Dashboard badge colors and status filter pills

**Files:**
- Modify: `sidecar/src/main/webui/src/views/org-dashboard.ts`

**Interfaces:**
- Consumes: `SlotStatus` enum values from REST API (`/api/workspace?root=...` JSON)
- Produces: colored badges for all 8 statuses, filter pills for status

- [ ] **Step 1: Update STATUS_COLORS map**

In `org-dashboard.ts`, replace the `STATUS_COLORS` constant:

```typescript
const STATUS_COLORS: Record<string, string> = {
  ACTIVE: '#4ade80',
  PAUSED: '#f59e0b',
  PENDING: '#6b7280',
  READY_TO_LAND: '#facc15',
  ARCHIVING: '#3b82f6',
  ARCHIVED: '#6b7280',
  FAILED: '#ef4444',
  PURGED: '#4b5563',
};
```

- [ ] **Step 2: Update SlotInfo interface**

Update the `status` field type in the `SlotInfo` interface:

```typescript
status: 'ACTIVE' | 'PAUSED' | 'PENDING' | 'READY_TO_LAND' | 'ARCHIVING' | 'ARCHIVED' | 'FAILED' | 'PURGED';
```

- [ ] **Step 3: Add status filter state and pills**

Add a state field:

```typescript
@state() private _statusFilter: string | null = null;
```

In `_renderSlots()`, add filter pills after the archived toggle and filter slots by status:

```typescript
const statusPills = ['PAUSED', 'ARCHIVING', 'FAILED'] as const;
```

```html
${statusPills.map(s => html`
  <span class="pill ${this._statusFilter === s ? 'active' : ''}"
        @click=${() => { this._statusFilter = this._statusFilter === s ? null : s; }}>
    ${s.replace('_', ' ')}
  </span>
`)}
```

Apply status filter to the filtered slots list:

```typescript
let filtered = this._hideArchived ? slots.filter(s => s.status !== 'ARCHIVED') : slots;
if (this._statusFilter) {
  filtered = filtered.filter(s => s.status === this._statusFilter);
}
```

- [ ] **Step 4: Build frontend**

Run: `cd sidecar/src/main/webui && yarn build`
Expected: exit 0

- [ ] **Step 5: Run frontend tests**

Run: `cd sidecar/src/main/webui && yarn test --run`
Expected: no new failures

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/webui/src/views/org-dashboard.ts
git commit -m "feat(#69): dashboard badge colors and status filter for enriched slot states Refs #69 Refs #74"
```

---

## References

- [2026-09-07-slot-status-design.md] — design spec this plan implements
- [decisions.md] — D1-D3 design decisions
- [SlotStatus.java] — current 3-value enum
- [WorkspaceScanner.java:160-236] — parseSlotFile with filesystem-only status
- [WorklogService.java:191-214] — existing slotStatus() query
- [org-dashboard.ts:58-62] — current STATUS_COLORS map
- [GitHub #69] — parse active/paused from .slot files
- [GitHub #74] — add ARCHIVING state for landed-but-not-archived slots
