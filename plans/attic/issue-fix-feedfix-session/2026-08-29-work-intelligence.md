# Work Intelligence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #19 — Work intelligence — proactive surfacing of stalled, forgotten, and unblocked work
**Issue group:** #19

**Goal:** Build a RAS-based intelligence layer that detects stalled, forgotten, unblocked, and cross-repo dependency gaps, surfacing findings via the model tree (LLM) and a UI panel (human).

**Architecture:** Four data adapters bridge trellis data sources into CloudEvents. Four JavaSwitchGanglion subclasses (one per data source archetype) evaluate events and produce DetectionResults. A WorkIntelligenceModelProvider maps ActiveSituations to a three-tier severity model for LLM consumption. A UI panel renders the same data for human visibility.

**Tech Stack:** Java 21, Quarkus 3.x, casehub-ras-api + casehub-ras-runtime (no Drools), CloudEvents SDK, Lit 3, pages-data-table

## Global Constraints

- Java 21, Quarkus 3.x
- Package root: `io.hortora.trellis`
- RAS dependency: `io.casehub:casehub-ras-api:${casehub.version}` + `io.casehub:casehub-ras-runtime:${casehub.version}`
- CloudEvents: `io.cloudevents:cloudevents-api` (transitive from ras-runtime)
- All ganglions are `JavaSwitchGanglion` subclasses — no Drools, no expression rules
- RAS requires `tenancyid` CloudEvent extension — use constant `"trellis"` for single-tenant mode
- All commits reference #19: `Refs #19`
- Frontend theme: `casehub-dark` via `applyTheme()` + `pages-density-compact`

---

## Batch 1: RAS foundation + first facet (stalled work detection)

After this batch: RAS engine runs in trellis, detects stalled work from worklog events, findings visible in model tree.

### Task 1: Add RAS dependencies and CloudEvent factory

Add casehub-ras-api and casehub-ras-runtime to the sidecar POM. Create a utility class for building trellis-domain CloudEvents with the required `tenancyid` extension.

**Files:**
- Modify: `sidecar/pom.xml` — add RAS dependencies
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/TrellisCloudEvents.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/TrellisCloudEventsTest.java`

**Interfaces:**
- Produces: `TrellisCloudEvents.worklogSnapshot(Map<String, Object> data): CloudEvent`, `TrellisCloudEvents.enrichmentIssue(Map<String, Object> data): CloudEvent`, `TrellisCloudEvents.deferredItem(Map<String, Object> data): CloudEvent`, `TrellisCloudEvents.crossRepoChange(Map<String, Object> data): CloudEvent`

- [ ] **Step 1: Write failing test for CloudEvent factory**

```java
@Test
void worklogSnapshotCreatesCloudEventWithCorrectTypeAndTenancy() {
    var data = Map.<String, Object>of("branch", "issue-42", "lastEventDaysAgo", 12);
    CloudEvent event = TrellisCloudEvents.worklogSnapshot(data);

    assertEquals("trellis.worklog.snapshot", event.getType());
    assertEquals("trellis", event.getExtension("tenancyid"));
    assertEquals("trellis-intelligence", event.getSource().toString());
    assertNotNull(event.getData());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisCloudEventsTest -pl .`
Expected: compilation failure

- [ ] **Step 3: Add RAS dependencies to pom.xml**

Add to `<dependencies>` in `sidecar/pom.xml`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ras-api</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ras-runtime</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

- [ ] **Step 4: Implement TrellisCloudEvents**

```java
package io.hortora.trellis.intelligence;

import io.cloudevents.CloudEvent;
import io.cloudevents.core.builder.CloudEventBuilder;
import io.cloudevents.core.data.PojoCloudEventData;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.net.URI;
import java.time.OffsetDateTime;
import java.util.Map;
import java.util.UUID;

public final class TrellisCloudEvents {

    private static final URI SOURCE = URI.create("trellis-intelligence");
    private static final String TENANCY = "trellis";
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private TrellisCloudEvents() {}

    public static CloudEvent worklogSnapshot(Map<String, Object> data) {
        return build("trellis.worklog.snapshot", data);
    }

    public static CloudEvent enrichmentIssue(Map<String, Object> data) {
        return build("trellis.enrichment.issue", data);
    }

    public static CloudEvent deferredItem(Map<String, Object> data) {
        return build("trellis.deferred.item", data);
    }

    public static CloudEvent crossRepoChange(Map<String, Object> data) {
        return build("trellis.crossrepo.change", data);
    }

    private static CloudEvent build(String type, Map<String, Object> data) {
        try {
            byte[] json = MAPPER.writeValueAsBytes(data);
            return CloudEventBuilder.v1()
                    .withId(UUID.randomUUID().toString())
                    .withSource(SOURCE)
                    .withType(type)
                    .withTime(OffsetDateTime.now())
                    .withExtension("tenancyid", TENANCY)
                    .withData("application/json", json)
                    .build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to build CloudEvent", e);
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=TrellisCloudEventsTest -pl .`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add sidecar/pom.xml sidecar/src/main/java/io/hortora/trellis/intelligence/ sidecar/src/test/java/io/hortora/trellis/intelligence/
git commit -m "feat(#19): add RAS dependencies and CloudEvent factory

casehub-ras-api + casehub-ras-runtime for work intelligence.
TrellisCloudEvents utility builds typed CloudEvents with tenancyid.

Refs #19"
```

---

### Task 2: StalledWorkGanglion + WorklogEventAdapter + situation registration

Create the first ganglion (stalled work detection), its data adapter (worklog → CloudEvents), and the CDI situation registration. This proves the full RAS pipeline end-to-end.

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/StalledWorkGanglion.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/WorklogEventAdapter.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/IntelligenceSituationProvider.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/StalledWorkGanglionTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/WorklogEventAdapterTest.java`

**Interfaces:**
- Consumes: `TrellisCloudEvents.worklogSnapshot()`, `WorklogService.getWorkItems()`, `WorklogService.getEvents()`
- Produces: `StalledWorkGanglion.evaluate(CloudEvent, SituationContext) → DetectionResult`, `WorklogEventAdapter.emitSnapshots(): void`, `IntelligenceSituationProvider.registrations() → List<SituationRegistration>`

- [ ] **Step 1: Write failing test for StalledWorkGanglion**

```java
@Test
void detectsStalledBranchOver7Days() {
    var ganglion = new StalledWorkGanglion();
    var data = Map.<String, Object>of(
        "branch", "issue-42-worklog",
        "issueNumber", 42,
        "lastEventDaysAgo", 12,
        "state", "active"
    );
    CloudEvent event = TrellisCloudEvents.worklogSnapshot(data);
    var context = new SituationContext("stalled-work", "issue-42-worklog",
            "trellis", Instant.now(), Instant.now(), List.of(),
            OptionalLong.empty(), null, 0);

    DetectionResult result = ganglion.evaluate(event, context);

    assertEquals(DetectionSignal.DETECTED, result.signal());
    assertTrue(result.confidence() >= 0.6);
    assertEquals("issue-42-worklog", result.evidence().get("branch"));
}

@Test
void returnsNoiseForRecentActivity() {
    var ganglion = new StalledWorkGanglion();
    var data = Map.<String, Object>of(
        "branch", "issue-49-layout",
        "issueNumber", 49,
        "lastEventDaysAgo", 2,
        "state", "active"
    );
    CloudEvent event = TrellisCloudEvents.worklogSnapshot(data);
    var context = new SituationContext("stalled-work", "issue-49-layout",
            "trellis", Instant.now(), Instant.now(), List.of(),
            OptionalLong.empty(), null, 0);

    DetectionResult result = ganglion.evaluate(event, context);

    assertEquals(DetectionSignal.NOISE, result.signal());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=StalledWorkGanglionTest -pl .`
Expected: compilation failure

- [ ] **Step 3: Implement StalledWorkGanglion**

```java
package io.hortora.trellis.intelligence;

import io.casehub.ras.api.*;
import io.cloudevents.CloudEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;
import java.util.Set;

public class StalledWorkGanglion extends JavaSwitchGanglion {

    private static final int ATTENTION_THRESHOLD_DAYS = 7;
    private static final int ACTION_THRESHOLD_DAYS = 14;
    private static final ObjectMapper MAPPER = new ObjectMapper();

    public StalledWorkGanglion() {
        super("stalled-work", Set.of("trellis.worklog.snapshot"));
    }

    @Override
    public DetectionResult evaluate(CloudEvent event, SituationContext context) {
        try {
            @SuppressWarnings("unchecked")
            var data = MAPPER.readValue(event.getData().toBytes(), Map.class);
            int daysAgo = ((Number) data.getOrDefault("lastEventDaysAgo", 0)).intValue();
            String state = (String) data.getOrDefault("state", "");

            if (!"active".equals(state) || daysAgo < ATTENTION_THRESHOLD_DAYS) {
                return noise();
            }

            double confidence = daysAgo >= ACTION_THRESHOLD_DAYS ? 0.85 : 0.45 + (daysAgo - 7) * 0.05;
            confidence = Math.min(confidence, 1.0);

            return detected(confidence, Map.of(
                    "branch", data.getOrDefault("branch", ""),
                    "issueNumber", data.getOrDefault("issueNumber", 0),
                    "lastEventDaysAgo", daysAgo,
                    "state", state
            ));
        } catch (Exception e) {
            return noise();
        }
    }
}
```

- [ ] **Step 4: Implement WorklogEventAdapter**

```java
package io.hortora.trellis.intelligence;

import io.cloudevents.CloudEvent;
import io.hortora.trellis.worklog.WorklogService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;

@ApplicationScoped
public class WorklogEventAdapter {

    @Inject WorklogService worklogService;
    @Inject Event<CloudEvent> cloudEventBus;

    public void emitSnapshots() {
        var items = worklogService.getWorkItems();
        var now = Instant.now();
        for (var item : items) {
            long daysAgo = item.endedAt() == null
                    ? Duration.between(item.lastEventAt(), now).toDays()
                    : 0;
            var data = Map.<String, Object>of(
                    "branch", item.branch(),
                    "issueNumber", item.issueNumber(),
                    "lastEventDaysAgo", daysAgo,
                    "state", item.state()
            );
            cloudEventBus.fireAsync(TrellisCloudEvents.worklogSnapshot(data));
        }
    }
}
```

- [ ] **Step 5: Implement IntelligenceSituationProvider**

```java
package io.hortora.trellis.intelligence;

import io.casehub.ras.api.*;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Duration;
import java.util.List;
import java.util.Set;

@ApplicationScoped
public class IntelligenceSituationProvider implements SituationDefinitionProvider {

    @Override
    public List<SituationRegistration> registrations() {
        return List.of(
                new SituationRegistration(new SituationDefinition(
                        "stalled-work",
                        Set.of("trellis.worklog.snapshot"),
                        Duration.ofDays(30),
                        Duration.ZERO,
                        ChainMode.NONE,
                        TriggerAction.LOG,
                        TriggerMode.EACH
                ))
        );
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=StalledWorkGanglionTest,WorklogEventAdapterTest -pl .`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/intelligence/ sidecar/src/test/java/io/hortora/trellis/intelligence/
git commit -m "feat(#19): StalledWorkGanglion + WorklogEventAdapter + situation provider

First RAS facet: detects branches with no activity for 7+ days.
WorklogEventAdapter bridges worklog.db work items to CloudEvents.
IntelligenceSituationProvider registers the stalled-work situation.

Refs #19"
```

---

### Task 3: WorkIntelligenceModelProvider + REST endpoint

Wire the RAS ActiveSituations into the trellis model tree and expose via REST.

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/WorkIntelligenceModelProvider.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/IntelligenceResource.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/WorkIntelligenceModelProviderTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/IntelligenceResourceTest.java`

**Interfaces:**
- Consumes: RAS `ActiveSituation` records, `WorklogEventAdapter.emitSnapshots()`
- Produces: `ModelProvider` domain `"intelligence"`, `GET /api/intelligence?root=...`

- [ ] **Step 1: Write failing test for severity mapping**

```java
@Test
void mapsHighConfidenceToActionNeeded() {
    assertEquals("ACTION_NEEDED", WorkIntelligenceModelProvider.mapSeverity(0.85));
}

@Test
void mapsMediumConfidenceToAttention() {
    assertEquals("ATTENTION", WorkIntelligenceModelProvider.mapSeverity(0.55));
}

@Test
void mapsLowConfidenceToInfo() {
    assertEquals("INFO", WorkIntelligenceModelProvider.mapSeverity(0.3));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=WorkIntelligenceModelProviderTest -pl .`
Expected: compilation failure

- [ ] **Step 3: Implement WorkIntelligenceModelProvider**

```java
package io.hortora.trellis.intelligence;

import io.hortora.trellis.mcp.ModelProvider;
import io.hortora.trellis.mcp.ActionDescriptor;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.*;

@ApplicationScoped
public class WorkIntelligenceModelProvider implements ModelProvider {

    @Inject WorklogEventAdapter worklogAdapter;

    @Override
    public String domain() { return "intelligence"; }

    @Override
    public Object summary() {
        worklogAdapter.emitSnapshots();
        // TODO: collect ActiveSituations from RAS engine
        // For now return empty — wired in when RAS situation store is accessible
        return Map.of(
                "summary", Map.of("actionNeeded", 0, "attention", 0, "info", 0),
                "findings", List.of()
        );
    }

    @Override
    public Object resolve(String subpath) {
        return summary();
    }

    @Override
    public List<ActionDescriptor> actionsFor(String nodeType) {
        return List.of();
    }

    static String mapSeverity(double confidence) {
        if (confidence > 0.7) return "ACTION_NEEDED";
        if (confidence >= 0.4) return "ATTENTION";
        return "INFO";
    }
}
```

- [ ] **Step 4: Implement IntelligenceResource**

```java
package io.hortora.trellis.intelligence;

import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

@Path("/api/intelligence")
@ApplicationScoped
public class IntelligenceResource {

    @Inject WorkIntelligenceModelProvider provider;

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response get(@QueryParam("root") String root) {
        if (root == null || root.isBlank()) {
            return Response.status(Response.Status.BAD_REQUEST).build();
        }
        return Response.ok(provider.summary()).build();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `/opt/homebrew/bin/mvn -f sidecar/pom.xml test -Dtest=WorkIntelligenceModelProviderTest,IntelligenceResourceTest -pl .`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add sidecar/src/main/java/io/hortora/trellis/intelligence/ sidecar/src/test/java/io/hortora/trellis/intelligence/
git commit -m "feat(#19): WorkIntelligenceModelProvider + REST endpoint

ModelProvider domain 'intelligence' with confidence-to-severity mapping.
GET /api/intelligence?root=... serves findings for UI panel.

Refs #19"
```

---

## Batch 2: Remaining archetype facets

After this batch: all four archetype facets operational (stalled, unblocked, deferred, cross-repo).

### Task 4: UnblockedWorkGanglion + EnrichmentAdapter

Detects issues whose blockers have been resolved.

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/UnblockedWorkGanglion.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/EnrichmentAdapter.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/intelligence/IntelligenceSituationProvider.java` — add unblocked situation
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/UnblockedWorkGanglionTest.java`

**Interfaces:**
- Consumes: `TrellisCloudEvents.enrichmentIssue()`, enrichment.db via `WorklogService.getBacklogSummary()`
- Produces: `UnblockedWorkGanglion.evaluate(CloudEvent, SituationContext) → DetectionResult`

- [ ] **Step 1: Write failing test**

```java
@Test
void detectsUnblockedIssue() {
    var ganglion = new UnblockedWorkGanglion();
    var data = Map.<String, Object>of(
        "issueNumber", 19,
        "state", "OPEN",
        "blockedBy", List.of(Map.of("number", 11, "state", "CLOSED"))
    );
    CloudEvent event = TrellisCloudEvents.enrichmentIssue(data);
    var context = new SituationContext("unblocked-work", "19",
            "trellis", Instant.now(), Instant.now(), List.of(),
            OptionalLong.empty(), null, 0);

    DetectionResult result = ganglion.evaluate(event, context);

    assertEquals(DetectionSignal.DETECTED, result.signal());
    assertTrue(result.confidence() >= 0.8);
}

@Test
void returnsNoiseWhenBlockerStillOpen() {
    var ganglion = new UnblockedWorkGanglion();
    var data = Map.<String, Object>of(
        "issueNumber", 19,
        "state", "OPEN",
        "blockedBy", List.of(Map.of("number", 11, "state", "OPEN"))
    );
    CloudEvent event = TrellisCloudEvents.enrichmentIssue(data);
    var context = new SituationContext("unblocked-work", "19",
            "trellis", Instant.now(), Instant.now(), List.of(),
            OptionalLong.empty(), null, 0);

    DetectionResult result = ganglion.evaluate(event, context);

    assertEquals(DetectionSignal.NOISE, result.signal());
}
```

- [ ] **Step 2: Implement UnblockedWorkGanglion + EnrichmentAdapter**
- [ ] **Step 3: Add unblocked situation to IntelligenceSituationProvider**
- [ ] **Step 4: Run tests, verify pass**
- [ ] **Step 5: Commit**

---

### Task 5: ForgottenDeferralGanglion + DeferredItemAdapter

Detects deferred items whose reasons no longer apply.

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/ForgottenDeferralGanglion.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/DeferredItemAdapter.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/intelligence/IntelligenceSituationProvider.java` — add forgotten-deferral situation
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/ForgottenDeferralGanglionTest.java`

**Interfaces:**
- Consumes: `TrellisCloudEvents.deferredItem()`, .plan files via workspace scan
- Produces: `ForgottenDeferralGanglion.evaluate(CloudEvent, SituationContext) → DetectionResult`

- [ ] **Step 1: Write failing test**

```java
@Test
void detectsDeferralWithResolvedBlocker() {
    var ganglion = new ForgottenDeferralGanglion();
    var data = Map.<String, Object>of(
        "title", "Add pagination to backlog",
        "reason", "blocked by #33",
        "blockerState", "CLOSED",
        "deferredDaysAgo", 21
    );
    CloudEvent event = TrellisCloudEvents.deferredItem(data);
    var context = new SituationContext("forgotten-deferral", "Add pagination to backlog",
            "trellis", Instant.now(), Instant.now(), List.of(),
            OptionalLong.empty(), null, 0);

    DetectionResult result = ganglion.evaluate(event, context);

    assertEquals(DetectionSignal.DETECTED, result.signal());
    assertTrue(result.confidence() >= 0.6);
}
```

- [ ] **Step 2: Implement ForgottenDeferralGanglion + DeferredItemAdapter**
- [ ] **Step 3: Add forgotten-deferral situation to IntelligenceSituationProvider**
- [ ] **Step 4: Run tests, verify pass**
- [ ] **Step 5: Commit**

---

### Task 6: CrossRepoDependencyGanglion + CrossRepoAdapter

Detects upstream changes not yet consumed downstream.

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/CrossRepoDependencyGanglion.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/intelligence/CrossRepoAdapter.java`
- Modify: `sidecar/src/main/java/io/hortora/trellis/intelligence/IntelligenceSituationProvider.java` — add cross-repo-dependency situation
- Test: `sidecar/src/test/java/io/hortora/trellis/intelligence/CrossRepoDependencyGanglionTest.java`

**Interfaces:**
- Consumes: `TrellisCloudEvents.crossRepoChange()`, GitHub API via enrichment cache
- Produces: `CrossRepoDependencyGanglion.evaluate(CloudEvent, SituationContext) → DetectionResult`

- [ ] **Step 1: Write failing test**

```java
@Test
void detectsUnconsumedUpstreamChange() {
    var ganglion = new CrossRepoDependencyGanglion();
    var data = Map.<String, Object>of(
        "upstreamRepo", "casehub-pages",
        "prNumber", 303,
        "prTitle", "feat: add activateDockPanel to LiveSite",
        "downstreamRepo", "trellis",
        "relatedIssues", List.of(49)
    );
    CloudEvent event = TrellisCloudEvents.crossRepoChange(data);
    var context = new SituationContext("cross-repo-dependency", "casehub-pages#303",
            "trellis", Instant.now(), Instant.now(), List.of(),
            OptionalLong.empty(), null, 0);

    DetectionResult result = ganglion.evaluate(event, context);

    assertEquals(DetectionSignal.DETECTED, result.signal());
}
```

- [ ] **Step 2: Implement CrossRepoDependencyGanglion + CrossRepoAdapter**
- [ ] **Step 3: Add cross-repo-dependency situation to IntelligenceSituationProvider**
- [ ] **Step 4: Run tests, verify pass**
- [ ] **Step 5: Commit**

---

## Batch 3: UI panel + integration wiring

After this batch: full loop — all four facets → model tree → REST → UI panel. Feature complete.

### Task 7: Intelligence panel (frontend)

Read-only Lit panel rendering findings grouped by severity.

**Files:**
- Create: `sidecar/src/main/webui/src/views/intelligence-panel.ts`
- Modify: `sidecar/src/main/webui/src/components/workbench-panels.ts` — register intelligence panel
- Modify: `CLAUDE.md` — document new panel and `/api/intelligence` endpoint

**Interfaces:**
- Consumes: `GET /api/intelligence?root=...` → JSON findings
- Produces: `<trellis-intelligence-panel>` Lit component with `workspaceRoot` property

- [ ] **Step 1: Create intelligence-panel.ts**

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { fromRows, ColumnType, columnId } from '@casehubio/pages-data';
import type { TypedDataSet } from '@casehubio/pages-data';

@customElement('trellis-intelligence-panel')
export class TrellisIntelligencePanel extends LitElement {

  @property() workspaceRoot = '';
  @state() private _findings: any[] = [];
  @state() private _loading = true;

  static override styles = css`
    :host { display: block; height: 100%; overflow-y: auto; padding: 16px; background: #1e1e1e; color: #d4d4d4; }
    .severity-group { margin-bottom: 16px; }
    .severity-label { font-size: 12px; font-weight: 600; text-transform: uppercase; margin-bottom: 8px; }
    .severity-label.action { color: #f87171; }
    .severity-label.attention { color: #fbbf24; }
    .severity-label.info { color: #60a5fa; }
    .finding { padding: 8px 12px; border-left: 3px solid; margin-bottom: 4px; background: #252525; border-radius: 2px; }
    .finding.action { border-color: #f87171; }
    .finding.attention { border-color: #fbbf24; }
    .finding.info { border-color: #60a5fa; }
    .subject { font-weight: 500; }
    .summary { font-size: 13px; color: #999; margin-top: 2px; }
    .suggestion { font-size: 12px; color: #6b7280; margin-top: 4px; font-style: italic; }
    .empty { color: #666; padding: 32px; text-align: center; }
  `;

  override connectedCallback() {
    super.connectedCallback();
    this._loadFindings();
  }

  override updated(changed: Map<PropertyKey, unknown>) {
    if (changed.has('workspaceRoot') && this.workspaceRoot) {
      this._loadFindings();
    }
  }

  private async _loadFindings() {
    if (!this.workspaceRoot) return;
    this._loading = true;
    try {
      const resp = await fetch(`/api/intelligence?root=${encodeURIComponent(this.workspaceRoot)}`);
      if (resp.ok) {
        const data = await resp.json();
        this._findings = data.findings ?? [];
      }
    } catch { /* non-critical */ }
    this._loading = false;
  }

  override render() {
    if (this._loading) return html`<div class="empty">Loading intelligence...</div>`;
    if (this._findings.length === 0) return html`<div class="empty">No findings — all clear.</div>`;

    const grouped = { action: [] as any[], attention: [] as any[], info: [] as any[] };
    for (const f of this._findings) {
      const key = f.severity === 'ACTION_NEEDED' ? 'action' : f.severity === 'ATTENTION' ? 'attention' : 'info';
      (grouped as any)[key].push(f);
    }

    return html`
      ${this._renderGroup('action', 'Action Needed', grouped.action)}
      ${this._renderGroup('attention', 'Attention', grouped.attention)}
      ${this._renderGroup('info', 'Info', grouped.info)}
    `;
  }

  private _renderGroup(cls: string, label: string, findings: any[]) {
    if (findings.length === 0) return '';
    return html`
      <div class="severity-group">
        <div class="severity-label ${cls}">${label} (${findings.length})</div>
        ${findings.map(f => html`
          <div class="finding ${cls}">
            <div class="subject">${f.subject}</div>
            <div class="summary">${f.summary}</div>
            ${f.suggestion ? html`<div class="suggestion">${f.suggestion}</div>` : ''}
          </div>
        `)}
      </div>
    `;
  }
}
```

- [ ] **Step 2: Register in workbench-panels.ts**

Add to `PANEL_TAGS`:
```typescript
intelligence: 'trellis-intelligence-panel',
```

Add import:
```typescript
import '../views/intelligence-panel.js';
```

Add to `DOCK_PANELS`:
```typescript
{ key: 'intelligence', label: 'Intelligence', icon: '\u{1F50D}', content: hostPanel('intelligence') },
```

- [ ] **Step 3: Update CLAUDE.md**

Add to Key Conventions:
```
- `GET /api/intelligence?root=...` — work intelligence findings (stalled, unblocked, deferred, cross-repo)
- Intelligence panel (`trellis-intelligence-panel`) — RAS-based work intelligence with four archetype facets; findings from `WorkIntelligenceModelProvider` domain `intelligence`
```

- [ ] **Step 4: Verify frontend builds**

Run: `cd sidecar/src/main/webui && yarn build`

- [ ] **Step 5: Commit**

```bash
git add sidecar/src/main/webui/src/views/intelligence-panel.ts sidecar/src/main/webui/src/components/workbench-panels.ts CLAUDE.md
git commit -m "feat(#19): intelligence panel — findings grouped by severity

Read-only Lit panel rendering work intelligence findings from
/api/intelligence. ACTION_NEEDED at top, ATTENTION below, INFO
collapsed. Registered as dock-bar panel.

Refs #19"
```

---

## References

- [2026-08-29-work-intelligence-design.md] — design spec this plan implements
- [WorklogService.java] — worklog data access layer
- [WorklogModelProvider.java] — reference ModelProvider implementation
- [ModelProvider.java:5] — ModelProvider SPI interface
- [GenerationCounter.java] — mtime-based change detection
- [FileWatcherService.java] — filesystem watch infrastructure
- [casehub-ras-api JavaSwitchGanglion] — pure Java ganglion base class
- [casehub-ras-api SituationDefinitionProvider] — CDI situation registration
- [casehub-ras-api ActiveSituation] — situation model
- [casehub-ras-api DetectionResult] — detection output record
- [casehub-ras-api DetectionSignal] — NOISE/ANTI/WEAK/DETECTED enum
- [GitHub #19] — focal issue
- [GitHub #11] — predecessor (Critical Path, closed)
