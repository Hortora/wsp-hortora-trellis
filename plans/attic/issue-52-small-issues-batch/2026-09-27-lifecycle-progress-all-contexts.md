# Lifecycle Progress for All Work Contexts — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #87 — Lifecycle step progress for all work contexts (standalone repos, single-repo slots)
**Issue group:** #87

**Goal:** Generalise lifecycle step progress from slot-scoped to context-scoped, so standalone repos get the same real-time SSE visibility as slots.

**Architecture:** Replace `slotId` with a sealed `WorkContext` interface (`SlotContext` / `RepoContext`) that gives compile-time exhaustiveness in Java and a string `contextId` at wire boundaries. Rename `SlotAgentCoordinator` → `LifecycleCoordinator`. Extract a shared `<lifecycle-progress>` Lit component for rendering.

**Tech Stack:** Java 21 (sealed interfaces, pattern matching), Quarkus 3.x, Lit (TypeScript), esbuild

## Global Constraints

- Java 21, sealed interfaces, pattern matching
- Package root: `io.hortora.trellis`
- `@JsonAlias("slotId")` on `OperationProgress.contextId` for backward compat with persisted JSON
- Keep plural step names (`stop-agents`) for both context types
- Keep synchronous coordinator methods for `LifecycleActionExecutor` (D7)

---

## Batch 1: Backend identity model and tracker generalisation

### Task 1: Create `WorkContext` sealed interface

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/WorkContext.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/WorkContextTest.java`

**Interfaces:**
- Produces: `WorkContext` sealed interface with `SlotContext(String slotId)`, `RepoContext(String repoName)`, `key()` → string, `parse(String contextId)` → `WorkContext`

- [ ] **Step 1: Write the failing test**

```java
package io.hortora.trellis.lifecycle;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class WorkContextTest {

    @Test
    void slotContextKeyFormat() {
        var ctx = new WorkContext.SlotContext("3");
        assertEquals("slot-3", ctx.key());
    }

    @Test
    void repoContextKeyFormat() {
        var ctx = new WorkContext.RepoContext("engine");
        assertEquals("repo-engine", ctx.key());
    }

    @Test
    void parseSlotContext() {
        var ctx = WorkContext.parse("slot-3");
        assertInstanceOf(WorkContext.SlotContext.class, ctx);
        assertEquals("3", ((WorkContext.SlotContext) ctx).slotId());
    }

    @Test
    void parseRepoContext() {
        var ctx = WorkContext.parse("repo-engine");
        assertInstanceOf(WorkContext.RepoContext.class, ctx);
        assertEquals("engine", ((WorkContext.RepoContext) ctx).repoName());
    }

    @Test
    void parseInvalidThrows() {
        assertThrows(IllegalArgumentException.class,
                () -> WorkContext.parse("unknown-ctx"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorkContextTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Write implementation**

```java
package io.hortora.trellis.lifecycle;

import com.fasterxml.jackson.annotation.JsonValue;

public sealed interface WorkContext {
    record SlotContext(String slotId) implements WorkContext {
        @Override public String key() { return "slot-" + slotId; }
    }
    record RepoContext(String repoName) implements WorkContext {
        @Override public String key() { return "repo-" + repoName; }
    }

    @JsonValue
    String key();

    static WorkContext parse(String contextId) {
        if (contextId.startsWith("slot-")) return new SlotContext(contextId.substring(5));
        if (contextId.startsWith("repo-")) return new RepoContext(contextId.substring(5));
        throw new IllegalArgumentException("Unknown context type: " + contextId);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=WorkContextTest`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git -C sidecar add src/main/java/io/hortora/trellis/lifecycle/WorkContext.java src/test/java/io/hortora/trellis/lifecycle/WorkContextTest.java
git commit -m "feat(#87): add WorkContext sealed interface Refs #87"
```

### Task 2: Generalise `OperationProgress` and `LifecycleOperationTracker`

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationProgress.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleOperationTracker.java`
- Modify: `sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleOperationTrackerTest.java`

**Interfaces:**
- Consumes: `WorkContext.key()` from Task 1
- Produces: `OperationProgress` with `contextId` field (was `slotId`), `LifecycleOperationTracker` methods accepting `contextId` strings

- [ ] **Step 1: Update existing tracker tests — rename `slotId` references to `contextId`**

In `LifecycleOperationTrackerTest.java`:
- Change `assertEquals("3", progress.slotId())` → `assertEquals("slot-3", progress.contextId())`
- Change all `tracker.startOperation("end", "3", ...)` → `tracker.startOperation("end", "slot-3", ...)`
- Change all `tracker.getActiveOperation("3")` → `tracker.getActiveOperation("slot-3")`

Add new test for repo context:

```java
@Test
void startOperationWithRepoContextId() {
    var opId = tracker.startOperation("end", "repo-engine",
            List.of("stop-agents", "rebase", "push", "stamp"));
    var progress = tracker.getProgress(opId);
    assertEquals("repo-engine", progress.contextId());
    assertEquals(OperationState.RUNNING, progress.state());
}

@Test
void concurrentOperationsOnDifferentContextTypes() {
    var op1 = tracker.startOperation("end", "slot-3", List.of("rebase"));
    var op2 = tracker.startOperation("pause", "repo-engine", List.of("commit-wip"));
    assertNotEquals(op1, op2);
    assertNotNull(tracker.getActiveOperation("slot-3"));
    assertNotNull(tracker.getActiveOperation("repo-engine"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleOperationTrackerTest`
Expected: FAIL — `slotId()` method no longer exists after next step

- [ ] **Step 3: Update `OperationProgress` record**

In `OperationProgress.java`:
- Rename `slotId` parameter to `contextId`
- Add `@com.fasterxml.jackson.annotation.JsonAlias("slotId")` on the `contextId` component for backward compat with persisted JSON
- Update all `withStep`, `withState`, `withFailed` methods to use `contextId`

In `LifecycleOperationTracker.java`:
- Rename `slotToOperation` → `contextToOperation`
- Update `startOperation` — parameter `slotId` → `contextId`, use `contextId` for map key
- Update `getActiveOperation` — parameter `slotId` → `contextId`
- Update `evictStale` — `op.slotId()` → `op.contextId()`
- Update `loadOnStartup` — `op.slotId()` → `op.contextId()`
- Update `broadcastStep` and `broadcastOperation` — `"slotId"` → `"contextId"` in payload maps

- [ ] **Step 4: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest=LifecycleOperationTrackerTest`
Expected: PASS (all existing + 2 new tests)

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationProgress.java sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleOperationTracker.java sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleOperationTrackerTest.java
git commit -m "feat(#87): generalise OperationProgress and tracker to contextId Refs #87"
```

## Batch 2: Coordinator rename and generalisation

### Task 3: Rename `SlotAgentCoordinator` → `LifecycleCoordinator` and generalise to `WorkContext`

**Files:**
- Rename: `SlotAgentCoordinator` → `LifecycleCoordinator` (use `ide_refactor_rename`)
- Rename: `SlotAgentCoordinatorTest` → `LifecycleCoordinatorTest` (use `ide_refactor_rename`)
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleCoordinator.java` (after rename)
- Modify: `sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleCoordinatorTest.java` (after rename)
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleManager.java` (parameter rename)

**Interfaces:**
- Consumes: `WorkContext` from Task 1, `LifecycleOperationTracker` with `contextId` from Task 2
- Produces: `LifecycleCoordinator` with `coordinatedEndAsync(WorkContext, Path)`, `coordinatedPauseAsync(WorkContext, Path)`, `coordinatedResumeAsync(WorkContext, Path)` and sync equivalents

- [ ] **Step 1: Rename class via IntelliJ**

Use `ide_refactor_rename` on `SlotAgentCoordinator` → `LifecycleCoordinator`. This updates all references (imports in `LifecycleResource`, `LifecycleActionExecutor`, `TrellisTools`, test file).

- [ ] **Step 2: Add `findTerminals` helper and update agent methods**

In `LifecycleCoordinator.java`:

Add the `findTerminals` method:

```java
private List<TerminalInfo> findTerminals(WorkContext ctx) {
    return switch (ctx) {
        case WorkContext.SlotContext s -> terminalRegistry.list().stream()
            .filter(t -> s.slotId().equals(t.slot())).toList();
        case WorkContext.RepoContext r -> terminalRegistry.list().stream()
            .filter(t -> r.repoName().equals(t.repo()) && t.slot() == null).toList();
    };
}
```

Rename agent methods and update them to use `findTerminals`:
- `stopAllSlotAgents(String slotId)` → `stopAllAgents(WorkContext ctx)` — calls `findTerminals(ctx)`
- `shutdownSlotAgents(String slotId)` → `shutdownAgents(WorkContext ctx)` — calls `findTerminals(ctx)`
- `resumeCoordinatorPausedAgents(String slotId)` → `resumePausedAgents(WorkContext ctx)` — calls `findTerminals(ctx)`

- [ ] **Step 3: Update async methods to accept `WorkContext`**

Change method signatures:
- `coordinatedEndAsync(String slotId, Path workspaceRoot)` → `coordinatedEndAsync(WorkContext context, Path workspaceRoot)`
- `coordinatedPauseAsync(String slotId, Path workspaceRoot)` → `coordinatedPauseAsync(WorkContext context, Path workspaceRoot)`
- `coordinatedResumeAsync(String slotId, Path workspaceRoot)` → `coordinatedResumeAsync(WorkContext context, Path workspaceRoot)`

Inside each async method:
- `asyncSlotLocks` → `asyncContextLocks`, keyed on `context.key()`
- `tracker.startOperation("end", slotId, ...)` → `tracker.startOperation("end", context.key(), ...)`
- Agent calls: `stopAllSlotAgents(slotId)` → `stopAllAgents(context)`
- `lifecycleManager.endSteps(slotId, ...)` → `lifecycleManager.endSteps(context.key(), ...)`

- [ ] **Step 4: Update sync methods to accept `WorkContext`**

Change signatures for `coordinatedEnd`, `coordinatedPause`, `coordinatedResume`:
- Parameter: `String slotId` → `WorkContext context`
- `slotLocks` → `contextLocks`, keyed on `context.key()`
- Agent calls: use `context` instead of `slotId`
- Manager calls: pass `context.key()` instead of `slotId`

- [ ] **Step 5: Update `LifecycleManager` parameter names**

In `LifecycleManager.java`, rename `slotId` parameter to `contextId` in:
- `end(String slotId, ...)` → `end(String contextId, ...)`
- `pause(String slotId, ...)` → `pause(String contextId, ...)`
- `resume(String slotId, ...)` → `resume(String contextId, ...)`
- `endSteps(String slotId, ...)` → `endSteps(String contextId, ...)`
- `pauseSteps(String slotId, ...)` → `pauseSteps(String contextId, ...)`
- `resumeSteps(String slotId, ...)` → `resumeSteps(String contextId, ...)`
- `slotMerge(String slotId, ...)` → `slotMerge(String contextId, ...)`

No logic change — just parameter rename for consistency.

- [ ] **Step 6: Update existing tests**

In `LifecycleCoordinatorTest.java` (already renamed):
- Update constructor and field references from `SlotAgentCoordinator` → `LifecycleCoordinator`
- Update `TerminalInfo` test data: `new TerminalInfo("t1", "/tmp", "slot-1", null, null, null)` — note `slot` field is "slot-1" but the coordinator will be called with `new WorkContext.SlotContext("slot-1")`
- Update sync method calls: `coordinator.coordinatedPause("slot-1", WORKSPACE)` → `coordinator.coordinatedPause(new WorkContext.SlotContext("slot-1"), WORKSPACE)`
- Update `lifecycleManager` verify calls: `verify(lifecycleManager).pause("slot-1", WORKSPACE)` → `verify(lifecycleManager).pause("slot-1", WORKSPACE)` (manager still gets string contextId via `context.key()`)

Add new test for repo context:

```java
@Test
void coordinatedPauseFindsTerminalByRepoWhenNoSlot() throws Exception {
    var repoTerm = new TerminalInfo("t1", "/tmp", null, "engine", null, null);
    var slotTerm = new TerminalInfo("t2", "/tmp", "slot-1", "engine", null, null);
    when(registry.list()).thenReturn(List.of(repoTerm, slotTerm));
    when(agentManager.getSnapshot("t1", repoTerm)).thenReturn(
            new AgentSnapshot("t1", repoTerm, runningAgent(), null));
    when(lifecycleManager.pause("repo-engine", WORKSPACE))
            .thenReturn(new OperationResult(true, 0, Map.of(), "", ""));

    coordinator.coordinatedPause(new WorkContext.RepoContext("engine"), WORKSPACE);

    verify(agentManager).gracefulShutdown("t1");
    verify(agentManager, never()).gracefulShutdown("t2");
    verify(lifecycleManager).pause("repo-engine", WORKSPACE);
}
```

- [ ] **Step 7: Run all lifecycle tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -pl . -Dtest="LifecycleCoordinatorTest,LifecycleOperationTrackerTest,WorkContextTest"`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add -A sidecar/src/
git commit -m "feat(#87): rename SlotAgentCoordinator to LifecycleCoordinator, generalise to WorkContext Refs #87"
```

### Task 4: Update `LifecycleResource`, `LifecycleActionExecutor`, and `TrellisTools`

**Files:**
- Modify: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleResource.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/coordinator/LifecycleActionExecutor.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java`
- Modify: `sidecar/src/test/java/io/hortora/trellis/coordinator/LifecycleActionExecutorTest.java`

**Interfaces:**
- Consumes: `WorkContext.parse()` from Task 1, `LifecycleCoordinator` from Task 3

- [ ] **Step 1: Update `LifecycleResource` endpoints**

Change `@PathParam("slotId")` → `@PathParam("contextId")` and add `WorkContext.parse()` at the boundary for `end`, `pause`, `resume` methods. Add `IllegalArgumentException` catch for HTTP 400.

Change `getOperationBySlot` query param from `@QueryParam("slot")` → `@QueryParam("context")`.

- [ ] **Step 2: Update `LifecycleActionExecutor`**

In `dispatch()`:
- `case "lifecycle.end"` → parse `p.get("slotId")` or `p.get("contextId")` (support both for backward compat) via `WorkContext.parse()`, pass typed context to `coordinator.coordinatedEnd(ctx, ...)`
- Same for `lifecycle.pause` and `lifecycle.resume`
- `case "slot.merge"` → `p.get("slotId")` stays as string (manager method)

- [ ] **Step 3: Update `TrellisTools.trellis_lifecycle`**

Change `(String) p.get("slotId")` → parse via `WorkContext.parse((String) p.get("contextId"))` for `end`, `pause`, `resume` operations. Pass typed context to coordinator.

- [ ] **Step 4: Update `LifecycleActionExecutorTest`**

Update test setup to pass `"contextId"` instead of `"slotId"` in action params. Update verify calls to use `WorkContext` matchers.

- [ ] **Step 5: Run full test suite**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleResource.java sidecar/src/main/java/io/hortora/trellis/coordinator/LifecycleActionExecutor.java sidecar/src/main/java/io/hortora/trellis/mcp/TrellisTools.java sidecar/src/test/java/io/hortora/trellis/coordinator/LifecycleActionExecutorTest.java
git commit -m "feat(#87): update REST, executor, and MCP tool to use WorkContext/contextId Refs #87"
```

## Batch 3: Frontend component extraction and repo-detail integration

### Task 5: Extract `<lifecycle-progress>` component from `slot-detail.ts`

**Files:**
- Create: `sidecar/src/main/webui/src/components/lifecycle-progress.ts`
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts`

**Interfaces:**
- Produces: `<lifecycle-progress>` custom element with `.operation` property, exported `StepProgress` and `OperationProgress` interfaces

- [ ] **Step 1: Create `lifecycle-progress.ts`**

Extract the `StepProgress` and `OperationProgress` interfaces and the `_renderLifecycle()` template from `slot-detail.ts` into a new pure rendering component:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';

export interface StepProgress {
  name: string;
  state: 'PENDING' | 'RUNNING' | 'DONE' | 'FAILED';
  summary: string;
  stdout: string | null;
  stderr: string | null;
}

export interface OperationProgress {
  operationId: string;
  operationType: string;
  contextId: string;
  state: 'RUNNING' | 'COMPLETED' | 'FAILED';
  steps: StepProgress[];
  errorMessage?: string;
}

@customElement('lifecycle-progress')
export class LifecycleProgress extends LitElement {
  @property({ type: Object }) operation: OperationProgress | null = null;
  @state() private _expandedStep: string | null = null;

  static override styles = css`
    /* Move lifecycle-specific CSS from slot-detail.ts here:
       .lifecycle-step, .lifecycle-step-active, .lifecycle-step-done,
       .lifecycle-step-failed, .lifecycle-step-pending,
       .lifecycle-icon-done, .lifecycle-icon-active, .lifecycle-icon-failed,
       .lifecycle-icon-pending, .lifecycle-spinner, .lifecycle-type-badge,
       .lifecycle-expand, .lifecycle-output */
  `;

  override render() {
    const op = this.operation;
    if (!op) return nothing;

    return html`
      <div class="sidebar-section">
        <h3>Lifecycle</h3>
        <span class="lifecycle-type-badge">${op.operationType}</span>
        ${op.steps.map(s => html`
          <div class="lifecycle-step ${s.state === 'RUNNING' ? 'lifecycle-step-active' : s.state === 'DONE' ? 'lifecycle-step-done' : s.state === 'FAILED' ? 'lifecycle-step-failed' : 'lifecycle-step-pending'}">
            <span class="${s.state === 'DONE' ? 'lifecycle-icon-done' : s.state === 'RUNNING' ? 'lifecycle-icon-active lifecycle-spinner' : s.state === 'FAILED' ? 'lifecycle-icon-failed' : 'lifecycle-icon-pending'}">
              ${s.state === 'DONE' ? '✓' : s.state === 'RUNNING' ? '●' : s.state === 'FAILED' ? '✗' : '○'}
            </span>
            <span>${s.name}</span>
          </div>
          ${(s.state === 'DONE' || s.state === 'FAILED') && (s.stdout || s.stderr) ? html`
            <button class="lifecycle-expand"
                    @click=${() => { this._expandedStep = this._expandedStep === s.name ? null : s.name; }}>
              ${this._expandedStep === s.name ? '▼' : '▶'} output
            </button>
            ${this._expandedStep === s.name ? html`
              <div class="lifecycle-output">${s.stdout || s.stderr || ''}</div>
            ` : nothing}
          ` : nothing}
          ${s.state === 'FAILED' && this._expandedStep !== s.name && (s.stderr || s.stdout) ? html`
            <div class="lifecycle-output" style="border-color:#991b1b">${s.stderr || s.stdout || ''}</div>
          ` : nothing}
        `)}
        ${op.errorMessage ? html`
          <div style="font-size:0.75rem;color:#f87171;margin-top:0.5rem">${op.errorMessage}</div>
        ` : nothing}
      </div>
    `;
  }
}
```

Copy the lifecycle CSS from `slot-detail.ts` styles into the component's `styles`.

- [ ] **Step 2: Update `slot-detail.ts` to use the component**

- Remove `StepProgress` and `OperationProgress` interfaces (import from `lifecycle-progress.ts`)
- Remove `_renderLifecycle()` method
- Remove `_expandedStep` state
- Import `'../components/lifecycle-progress.js'` and the interfaces
- Replace `${this._renderLifecycle()}` in `_renderSidebar()` with:
  ```typescript
  html`<lifecycle-progress .operation=${this._operation}></lifecycle-progress>`
  ```
- Remove lifecycle-specific CSS from `slot-detail.ts` styles (now in component)
- Update `_handleLifecycleEvent` — change `slotId` filter to `contextId`:
  `if (String(data.contextId) !== 'slot-' + String(this.slotNumber)) return;`
- Update `_lifecycleAction` URL paths: `\`/api/lifecycle/end/${this.slotNumber}\`` → `\`/api/lifecycle/end/slot-${this.slotNumber}\``
- Update `_end()` and `_pause()` to use the new URL format
- On `connectedCallback` refresh: `GET /api/lifecycle/operations?context=slot-${this.slotNumber}` (was `?slot=`)

- [ ] **Step 3: Build frontend to verify compilation**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: PASS — no TypeScript errors

- [ ] **Step 4: Commit**

```bash
git add sidecar/src/main/webui/src/components/lifecycle-progress.ts sidecar/src/main/webui/src/views/slot-detail.ts
git commit -m "feat(#87): extract lifecycle-progress component, update slot-detail Refs #87"
```

### Task 6: Add lifecycle progress to `repo-detail.ts`

**Files:**
- Modify: `sidecar/src/main/webui/src/views/repo-detail.ts`

**Interfaces:**
- Consumes: `<lifecycle-progress>` component and `OperationProgress` interface from Task 5, REST endpoints from Task 4

- [ ] **Step 1: Add lifecycle state and imports**

Import `OperationProgress` from `'../components/lifecycle-progress.js'`. Import the component side-effect.

Add state fields:

```typescript
@state() private _operation: OperationProgress | null = null;
```

- [ ] **Step 2: Add `lifecycle:progress` to SSE subscription**

In `_subscribeEvents()`, add `lifecycle:progress` to the EventSource topics URL. Add handling in the `onmessage` callback:

```typescript
} else if (data.topic === 'lifecycle:progress') {
  this._handleLifecycleEvent(data.payload ?? data);
}
```

- [ ] **Step 3: Add `_handleLifecycleEvent`**

```typescript
private _handleLifecycleEvent(data: Record<string, unknown>) {
  if (String(data.contextId) !== 'repo-' + this.repoName) return;
  if (data.step) {
    if (this._operation) {
      this._operation = {
        ...this._operation,
        state: data.operationState as OperationProgress['state'],
        steps: this._operation.steps.map(s =>
          s.name === data.step
            ? { ...s, state: data.state as StepProgress['state'], stdout: data.stdout as string | null, stderr: data.stderr as string | null }
            : s
        ),
      };
    }
  } else {
    if (this._operation) {
      this._operation = {
        ...this._operation,
        state: data.operationState as OperationProgress['state'],
        errorMessage: data.errorMessage as string | undefined,
      };
    }
    if (data.operationState === 'COMPLETED' || data.operationState === 'FAILED') {
      this._actionInProgress = null;
      this._loadRepo();
      this._loadTerminal();
      if (data.operationState === 'COMPLETED') {
        setTimeout(() => { this._operation = null; }, 10000);
      }
    }
  }
}
```

Import `StepProgress` from `'../components/lifecycle-progress.js'` as well.

- [ ] **Step 4: Add lifecycle buttons to `_renderToolbar()`**

After the branch badge, add pause and end buttons — only when repo is not on main:

```typescript
${repo.branch !== 'main' ? html`
  <button class="action-btn" ?disabled=${!!this._actionInProgress}
          @click=${() => this._lifecycleAction('pause')}>pause</button>
  <button class="action-btn danger" ?disabled=${!!this._actionInProgress}
          @click=${() => this._lifecycleAction('end')}>end</button>
` : nothing}
```

- [ ] **Step 5: Add `<lifecycle-progress>` to `_renderSidebar()`**

Add after the Path section (before the Remote section):

```typescript
html`<lifecycle-progress .operation=${this._operation}></lifecycle-progress>`
```

- [ ] **Step 6: Add `_lifecycleAction` method**

```typescript
private async _lifecycleAction(name: string) {
  this._actionInProgress = name;
  try {
    const res = await fetch(`/api/lifecycle/${name}/repo-${this.repoName}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ workspaceRoot: this.workspaceRoot }),
    });
    if (!res.ok) {
      const body = await res.json().catch(() => null);
      this._error = body?.error ?? `${name} failed: HTTP ${res.status}`;
      this._actionInProgress = null;
      return;
    }
    this._operation = await res.json() as OperationProgress;
  } catch (e) {
    this._error = `${name} failed: ${e}`;
    this._actionInProgress = null;
  }
}
```

- [ ] **Step 7: Add operation recovery on connectedCallback**

In `connectedCallback`, after `_loadRepo()`, add:

```typescript
if (this.repoName) {
  fetch(`/api/lifecycle/operations?context=repo-${this.repoName}`)
    .then(r => r.ok ? r.json() : null)
    .then(op => { if (op) this._operation = op; })
    .catch(() => {});
}
```

- [ ] **Step 8: Build frontend**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add sidecar/src/main/webui/src/views/repo-detail.ts
git commit -m "feat(#87): add lifecycle progress and actions to repo-detail Refs #87"
```

## Batch 4: CLAUDE.md update and full verification

### Task 7: Update CLAUDE.md conventions and run full build

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- None — documentation only

- [ ] **Step 1: Update CLAUDE.md**

Update the `## Key Conventions` section:
- Change `POST /api/lifecycle/{end|pause|resume}/{slotId}` → `POST /api/lifecycle/{end|pause|resume}/{contextId}`
- Change `GET /api/lifecycle/operations?slot={slotId}` → `GET /api/lifecycle/operations?context={contextId}`
- Change `SlotAgentCoordinator` references → `LifecycleCoordinator`
- Update the `lifecycle:progress` SSE topic description to mention both slot-detail and repo-detail
- Add note about `WorkContext` sealed interface

- [ ] **Step 2: Run full backend build and tests**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml clean test`
Expected: PASS — all tests green

- [ ] **Step 3: Run full frontend build**

Run: `yarn --cwd sidecar/src/main/webui build`
Expected: PASS — no errors

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md conventions for contextId and LifecycleCoordinator Refs #87"
```

## References

- [2026-09-27-lifecycle-progress-all-contexts-design.md] — design spec this plan implements
- `OperationProgress.java:6-41` — current record with `slotId`
- `LifecycleOperationTracker.java:25-229` — operation tracking, SSE broadcast
- `SlotAgentCoordinator.java:21-253` — current coordinator (to be renamed)
- `LifecycleManager.java:16-277` — script execution
- `LifecycleResource.java:20-156` — REST endpoints
- `LifecycleActionExecutor.java:18-76` — LLM coordinator path
- `TrellisTools.java:279-315` — MCP tool
- `slot-detail.ts:460-493` — current `_renderLifecycle()`
- `repo-detail.ts:37-464` — current repo detail (no lifecycle)
- `LifecycleOperationTrackerTest.java:14-50` — existing tracker tests
- `SlotAgentCoordinatorTest.java:21-80` — existing coordinator tests
- GitHub #87 — focal issue
- GitHub #68 — predecessor (slot-scoped implementation)
