# Binding Activation State + Dataset Enhancements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #170 — Binding activation state
**Issue group:** #170, #173

**Goal:** Capture and surface the blackboard state that triggered each binding activation, and adopt dataset refresh intervals / SSE streaming for near-real-time governance dashboards.

**Architecture:** Engine captures the changed layer content when a binding fires, stores it on PlanItemRecord, and surfaces it via the plan-items REST endpoint. New CDI events (PlanItemStateChangedEvent, CaseContextUpdatedEvent) enable an SSE stream per case. Devtown adopts refreshTime on polling datasets and platform preferences for operator-configurable intervals.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson, CDI events, JAX-RS SSE, TypeScript (pages-ui DSL)

## Global Constraints

- All casehubio artifacts at `0.2-SNAPSHOT`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Pre-release platform — breaking changes are acceptable
- IntelliJ MCP mandatory for all `.java`/`.ts` edits
- Engine workspace already open: use `project_path=/Users/mdproctor/claude/casehub/devtown` for devtown modules, `project_path=/Users/mdproctor/claude/casehub/engine/common` (or `/runtime`, `/rest`, `/planning`) for engine modules

---

### Task 1: Create PlanItemStateChangedEvent CDI event

**Files:**
- Create: `engine/common/src/main/java/io/casehub/engine/common/spi/event/PlanItemStateChangedEvent.java`
- Test: `engine/common/src/test/java/io/casehub/engine/common/spi/event/PlanItemStateChangedEventTest.java`

**Interfaces:**
- Consumes: `TaskStatus` from `io.casehub.engine.common.internal.model.TaskStatus`
- Produces: `PlanItemStateChangedEvent(UUID caseId, String planItemId, String bindingName, TaskStatus previousStatus, TaskStatus newStatus, String tenancyId)` — used by Tasks 4, 5, 6, 7, 9, 10

- [ ] **Step 1: Write the record**

```java
package io.casehub.engine.common.spi.event;

import io.casehub.engine.common.internal.model.TaskStatus;
import java.util.UUID;

public record PlanItemStateChangedEvent(
    UUID caseId,
    String planItemId,
    String bindingName,
    TaskStatus previousStatus,
    TaskStatus newStatus,
    String tenancyId) {}
```

Use `ide_create_file` at path `common/src/main/java/io/casehub/engine/common/spi/event/PlanItemStateChangedEvent.java` in the engine project.

- [ ] **Step 2: Write unit test**

```java
@Test
void recordFieldsAreAccessible() {
    var event = new PlanItemStateChangedEvent(
        UUID.randomUUID(), "pi-1", "security-review",
        TaskStatus.RUNNING, TaskStatus.COMPLETED, "tenant-1");
    assertEquals(TaskStatus.RUNNING, event.previousStatus());
    assertEquals(TaskStatus.COMPLETED, event.newStatus());
    assertEquals("security-review", event.bindingName());
}
```

- [ ] **Step 3: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl common -Dtest=PlanItemStateChangedEventTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 4: Commit**

```
feat(#170): add PlanItemStateChangedEvent CDI event record

Refs casehubio/devtown#170
```

---

### Task 2: Create CaseContextUpdatedEvent CDI event

**Files:**
- Create: `engine/common/src/main/java/io/casehub/engine/common/spi/event/CaseContextUpdatedEvent.java`

**Interfaces:**
- Produces: `CaseContextUpdatedEvent(UUID caseId, String changedLayer, String tenancyId)` — used by Tasks 9, 10

- [ ] **Step 1: Write the record**

```java
package io.casehub.engine.common.spi.event;

import java.util.UUID;

public record CaseContextUpdatedEvent(
    UUID caseId,
    String changedLayer,
    String tenancyId) {}
```

- [ ] **Step 2: Commit with Task 1**

Batch with Task 1 commit if not yet committed.

---

### Task 3: Add activationContext to WorkerScheduleEvent and HumanTaskScheduleEvent

**Files:**
- Modify: `engine/common/src/main/java/io/casehub/engine/common/internal/event/WorkerScheduleEvent.java`
- Modify: `engine/common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java`

**Interfaces:**
- Consumes: existing record fields
- Produces: `WorkerScheduleEvent.activationContext()` (JsonNode, nullable), `HumanTaskScheduleEvent.activationContext()` (JsonNode, nullable) — used by Tasks 4, 5

- [ ] **Step 1: Add activationContext field to WorkerScheduleEvent**

Add `JsonNode activationContext` as the final field in the record. Update the compact constructor to handle null. Update the two convenience constructors to pass `null` for activationContext.

Use `ide_edit_member` with `member=WorkerScheduleEvent` (class declaration) to replace the entire record.

- [ ] **Step 2: Fix compilation — find all call sites**

Run `ide_find_references` on `WorkerScheduleEvent` to find all constructor calls. The 11-arg constructor in `CaseContextChangedEventHandler.scheduleWorker()` needs the new `activationContext` parameter (null for now — threaded in Task 5). Other convenience constructor calls should compile unchanged.

- [ ] **Step 3: Add activationContext field to HumanTaskScheduleEvent**

Add `JsonNode activationContext` as the final field. Use `ide_edit_member` with `member=HumanTaskScheduleEvent`.

- [ ] **Step 4: Fix compilation — find all HumanTaskScheduleEvent call sites**

Run `ide_find_references` on `HumanTaskScheduleEvent`. The constructor call in `CaseContextChangedEventHandler.publishHumanTaskSchedule()` needs the new parameter (null for now).

- [ ] **Step 5: Verify build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl common,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
feat(#170): add activationContext field to WorkerScheduleEvent and HumanTaskScheduleEvent

Refs casehubio/devtown#170
```

---

### Task 4: Add activationContext to PlanItemRecord and PlanItemSaveRequest

**Files:**
- Modify: `engine/common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java`
- Modify: `engine/common/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java`
- Modify: `engine/persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryPlanItemStore.java` (conversion)
- Test: existing PlanItemStore contract tests

**Interfaces:**
- Produces: `PlanItemRecord.activationContext()` (JsonNode, nullable), `PlanItemSaveRequest.activationContext()` (JsonNode, nullable) — used by Task 8

- [ ] **Step 1: Add activationContext to PlanItemRecord**

Add `JsonNode activationContext` as field 19. Update the `primitive()` factory method to accept and pass the new field, defaulting to null.

- [ ] **Step 2: Add activationContext to PlanItemSaveRequest**

Same pattern. Update `primitive()` factory.

- [ ] **Step 3: Fix all call sites**

Run `ide_find_references` on both `PlanItemRecord` and `PlanItemSaveRequest` to find all constructor/factory calls. Each needs the new parameter (null for now). The `InMemoryPlanItemStore` converts between them — update the conversion.

- [ ] **Step 4: Verify build across all engine modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 5: Verify devtown compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS (devtown uses PlanItemRecord via engine SNAPSHOT)

- [ ] **Step 6: Commit**

```
feat(#170): add activationContext field to PlanItemRecord and PlanItemSaveRequest

Refs casehubio/devtown#170
```

---

### Task 5: Thread activationContext through CaseContextChangedEventHandler

**Files:**
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Test: `engine/runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java`

**Interfaces:**
- Consumes: `WorkerScheduleEvent.activationContext()`, `HumanTaskScheduleEvent.activationContext()` (Task 3)
- Produces: activationContext threaded from `rules()` through the full dispatch chain

- [ ] **Step 1: Write test for activation context capture**

In `CaseContextChangedEventHandlerRoutingTest`, add a test that triggers a binding via context change and verifies the `WorkerScheduleEvent` published on the event bus carries a non-null `activationContext` matching the changed layer content.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — activationContext is null because it's not threaded yet.

- [ ] **Step 3: Capture snapshot in rules()**

In `rules()`, after binding evaluation succeeds and before dispatch, compute:
```java
JsonNode activationSnapshot = changedLayer != null
    && contextSnapshot.layer(changedLayer) != null
    ? contextSnapshot.layer(changedLayer).asJsonNode()
    : null;
```

Pass `activationSnapshot` through the method chain:
- `rules()` → add `JsonNode activationSnapshot` parameter (computed once, used for all eligible bindings in this evaluation)
- `publishByTarget()` → add `JsonNode activationSnapshot` parameter
- `publishWorkerSchedule()` → add `JsonNode activationSnapshot` parameter
- `publishHumanTaskSchedule()` → add `JsonNode activationSnapshot` parameter
- `scheduleWorker()` → include in `WorkerScheduleEvent` constructor call
- `publishHumanTaskSchedule()` → include in `HumanTaskScheduleEvent` constructor call

- [ ] **Step 4: Publish CaseContextUpdatedEvent**

Inject `Event<CaseContextUpdatedEvent>` into `CaseContextChangedEventHandler`. In `evaluateAndDispatch()`, after the null checks and before `rules()`:
```java
if (changedLayer != null) {
    caseContextUpdatedEvents.fireAsync(
        new CaseContextUpdatedEvent(caseInstance.getUuid(), changedLayer, caseInstance.tenancyId));
}
```

- [ ] **Step 5: Run test to verify it passes**

Expected: PASS

- [ ] **Step 6: Run full engine test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All tests pass

- [ ] **Step 7: Commit**

```
feat(#170): thread activationContext through binding dispatch chain

Captures changed layer content when a binding fires and threads it through
publishByTarget → scheduleWorker → WorkerScheduleEvent. Also publishes
CaseContextUpdatedEvent CDI event from evaluateAndDispatch.

Refs casehubio/devtown#170
```

---

### Task 6: Persist activationContext in WorkerScheduleEventHandler

**Files:**
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkerScheduleEventHandler.java`
- Test: add to existing test or write new test

**Interfaces:**
- Consumes: `WorkerScheduleEvent.activationContext()` (Task 3)
- Produces: `activationContext` in EventLog metadata

- [ ] **Step 1: Write test**

Test that when `WorkerScheduleEventHandler` processes a `WorkerScheduleEvent` with a non-null `activationContext`, the resulting EventLog entry's metadata includes the `activationContext` node.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Add activationContext to buildEventLog()**

Add `JsonNode activationContext` parameter to `buildEventLog()`. At the call site in `onWorkerScheduleEventHandler()`, pass `event.activationContext()`. In `buildEventLog()`:
```java
if (activationContext != null && !activationContext.isNull()) {
    metaNode.set("activationContext", activationContext);
}
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```
feat(#170): persist activationContext in WORKER_SCHEDULED event log metadata

Refs casehubio/devtown#170
```

---

### Task 7: Migrate PlanItemCompletionHandler to PlanItemStateChangedEvent

**Files:**
- Modify: `engine/planning/src/main/java/io/casehub/engine/planning/handler/PlanItemCompletionHandler.java`
- Modify: `engine/planning/src/test/java/io/casehub/engine/planning/handler/PlanItemCompletionHandlerTest.java`

**Interfaces:**
- Consumes: `PlanItemStateChangedEvent` (Task 1)
- Produces: fires `PlanItemStateChangedEvent` instead of `PlanItemCompletedEvent` on COMPLETED transitions

- [ ] **Step 1: Update PlanItemCompletionHandler**

Replace `Event<PlanItemCompletedEvent> planItemCompletedEvents` with `Event<PlanItemStateChangedEvent> planItemStateChangedEvents`. Update both `completePlanItemByBindingName()` and `completePlanItemByKey()` to fire:
```java
planItemStateChangedEvents.fireAsync(new PlanItemStateChangedEvent(
    caseId, item.getPlanItemId(), bindingName,
    TaskStatus.RUNNING, TaskStatus.COMPLETED, tenancyId));
```

Note: `previousStatus` is RUNNING (or DELEGATED) — read the actual previous status from the PlanItem before marking complete.

- [ ] **Step 2: Update PlanItemCompletionHandlerTest**

Migrate test assertions from `PlanItemCompletedEvent` to `PlanItemStateChangedEvent`.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl planning -f /Users/mdproctor/claude/casehub/engine/pom.xml`

- [ ] **Step 4: Commit**

```
feat(#170): migrate PlanItemCompletionHandler to PlanItemStateChangedEvent

Breaking: replaces PlanItemCompletedEvent with PlanItemStateChangedEvent.
Carries previousStatus/newStatus for transition awareness.

Refs casehubio/devtown#170
```

---

### Task 8: Migrate fault/reject handlers to PlanItemStateChangedEvent

**Files:**
- Modify: `engine/planning/src/main/java/io/casehub/engine/planning/handler/WorkerRetryExhaustionHandler.java`
- Modify: `engine/planning/src/main/java/io/casehub/engine/planning/handler/WorkerOutcomeResolvedHandler.java`
- Modify: `engine/planning/src/main/java/io/casehub/engine/planning/handler/ActionGateExpiredPlanItemHandler.java`
- Modify: `engine/planning/src/main/java/io/casehub/engine/planning/handler/ActionGateRejectedPlanItemHandler.java`
- Modify: `engine/planning/src/test/java/io/casehub/engine/planning/handler/WorkerRetryExhaustionHandlerTest.java`

**Interfaces:**
- Consumes: `PlanItemStateChangedEvent` (Task 1)
- Produces: fires `PlanItemStateChangedEvent` instead of `PlanItemFaultedEvent` on FAULTED transitions

- [ ] **Step 1: Migrate WorkerOutcomeResolvedHandler**

Replace `Event<PlanItemFaultedEvent>` with `Event<PlanItemStateChangedEvent>`. Update the `fireAsync` call with appropriate `previousStatus`/`newStatus`.

- [ ] **Step 2: Migrate ActionGateExpiredPlanItemHandler**

Same pattern.

- [ ] **Step 3: Migrate ActionGateRejectedPlanItemHandler**

Same pattern.

- [ ] **Step 4: Migrate WorkerRetryExhaustionHandler**

Same pattern. Update `WorkerRetryExhaustionHandlerTest`.

- [ ] **Step 5: Migrate MixedWorkersBlackboardTest.WorkerCompletionObserver**

Change observer from `PlanItemCompletedEvent` to `PlanItemStateChangedEvent`, filter on `newStatus == COMPLETED`.

- [ ] **Step 6: Run engine tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`

- [ ] **Step 7: Commit**

```
feat(#170): migrate fault/reject handlers to PlanItemStateChangedEvent

Consolidates PlanItemFaultedEvent usage across WorkerRetryExhaustionHandler,
WorkerOutcomeResolvedHandler, ActionGateExpired/RejectedPlanItemHandler.

Refs casehubio/devtown#170
```

---

### Task 9: Migrate devtown ReviewOutcomeObserver + delete old events

**Files:**
- Modify: `devtown/app/src/main/java/io/casehub/devtown/app/ReviewOutcomeObserver.java`
- Modify: `devtown/app/src/test/java/io/casehub/devtown/app/ReviewOutcomeObserverTest.java`
- Modify: `devtown/app/src/test/java/io/casehub/devtown/app/CaseMemoryIntegrationTest.java`
- Delete: `engine/common/src/main/java/io/casehub/engine/common/spi/event/PlanItemCompletedEvent.java`
- Delete: `engine/common/src/main/java/io/casehub/engine/common/spi/event/PlanItemFaultedEvent.java`
- Delete: `engine/common/src/main/java/io/casehub/engine/common/spi/event/PlanItemRejectedEvent.java`

**Interfaces:**
- Consumes: `PlanItemStateChangedEvent` (Task 1)

- [ ] **Step 1: Migrate ReviewOutcomeObserver**

Change `@ObservesAsync PlanItemCompletedEvent event` to `@ObservesAsync PlanItemStateChangedEvent event`. Add guard: `if (event.newStatus() != TaskStatus.COMPLETED) return;`. Update field access: `event.trackingKey()` → derive from `event.bindingName()` or `event.planItemId()`.

Note: `PlanItemCompletedEvent` had `trackingKey` (workerName for capability, childCaseId for subcase). `PlanItemStateChangedEvent` has `bindingName`. `ReviewOutcomeObserver` uses `trackingKey` to look up context keys — check if `bindingName` serves the same purpose or if additional mapping is needed.

- [ ] **Step 2: Migrate ReviewOutcomeObserverTest**

Replace `PlanItemCompletedEvent` constructor calls with `PlanItemStateChangedEvent`. The test fires events directly — update each `fireAsync(new PlanItemCompletedEvent(...))` call.

- [ ] **Step 3: Migrate CaseMemoryIntegrationTest**

Same pattern — update the injected event type and constructor calls.

- [ ] **Step 4: Delete old event classes**

Use `ide_refactor_safe_delete` on:
- `PlanItemCompletedEvent.java`
- `PlanItemFaultedEvent.java`
- `PlanItemRejectedEvent.java`

If safe delete reports remaining references (in docs/specs), use `force: true` — doc references are non-compilable.

- [ ] **Step 5: Run full build across engine + devtown**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

- [ ] **Step 6: Commit**

```
feat(#170): migrate ReviewOutcomeObserver and delete legacy plan item events

Breaking: removes PlanItemCompletedEvent, PlanItemFaultedEvent,
PlanItemRejectedEvent. All observers migrated to PlanItemStateChangedEvent.

Refs casehubio/devtown#170
```

---

### Task 10: Extend PlanItemResponse and CaseStreamResource SSE endpoint

**Files:**
- Modify: `engine/rest/src/main/java/io/casehub/engine/rest/dto/PlanItemResponse.java`
- Create: `engine/rest/src/main/java/io/casehub/engine/rest/CaseStreamResource.java`
- Test: `engine/rest/src/test/java/io/casehub/engine/rest/CaseInstancePlanItemsResourceTest.java`
- Create: `engine/rest/src/test/java/io/casehub/engine/rest/CaseStreamResourceTest.java`

**Interfaces:**
- Consumes: `PlanItemRecord.activationContext()` (Task 4), `PlanItemStateChangedEvent` (Task 1), `CaseContextUpdatedEvent` (Task 2)
- Produces: `PlanItemResponse.activationContext` (JsonNode), SSE endpoint at `/api/v1/cases/{caseId}/stream`

- [ ] **Step 1: Write test for PlanItemResponse.activationContext**

In `CaseInstancePlanItemsResourceTest`: start a case, signal context to trigger a binding (with activationContext populated from Task 5), GET `/api/v1/cases/{caseId}/plan-items`, assert `activationContext` field is present and non-null on the response.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — PlanItemResponse doesn't have activationContext field yet.

- [ ] **Step 3: Extend PlanItemResponse**

Add `JsonNode activationContext` as the final field. Update `from(PlanItemRecord)` to map `record.activationContext()` directly.

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write SSE endpoint test**

```java
@QuarkusTest
class CaseStreamResourceTest {
    @Test
    void sseStreamReceivesPlanItemStateChange() {
        // Start case, connect SSE to /api/v1/cases/{caseId}/stream
        // Trigger a plan item transition
        // Verify SSE event received with type "plan-item"
        // Use reconnectingEvery(Long.MAX_VALUE, MILLISECONDS) per GE-20260617-0c1498
    }
}
```

- [ ] **Step 6: Implement CaseStreamResource**

Create the SSE endpoint as specified in the design spec §B3. Key implementation details:
- `@Path("/api/v1/cases/{caseId}/stream")`
- `Map<UUID, Set<SseEventSink>>` for per-case sink tracking
- `@ObservesAsync PlanItemStateChangedEvent` and `@ObservesAsync CaseContextUpdatedEvent`
- `MAX_SINKS_PER_CASE = 32`
- Dead sink cleanup in `broadcast()`

- [ ] **Step 7: Run SSE test**

- [ ] **Step 8: Run full engine-rest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -f /Users/mdproctor/claude/casehub/engine/pom.xml`

- [ ] **Step 9: Commit**

```
feat(#170): extend PlanItemResponse with activationContext + SSE stream endpoint

PlanItemResponse maps activationContext directly from PlanItemRecord.
CaseStreamResource multiplexes plan-item and context-updated SSE events
per case.

Refs casehubio/devtown#170, casehubio/devtown#173
```

---

### Task 11: GovernancePreferencesResource

**Files:**
- Create: `devtown/app/src/main/java/io/casehub/devtown/app/governance/GovernancePreferencesResource.java`
- Create: `devtown/app/src/test/java/io/casehub/devtown/app/governance/GovernancePreferencesResourceTest.java`

**Interfaces:**
- Consumes: `PreferenceProvider` from `io.casehub.platform.api.preferences`
- Produces: `GET /api/governance/preferences` → `Map<String, String>` with keys `refresh.operational`, `refresh.metrics`, `refresh.caseDetail`

- [ ] **Step 1: Write test**

```java
@QuarkusTest
class GovernancePreferencesResourceTest {
    @Test
    void returnsDefaultPreferences() {
        given().when().get("/api/governance/preferences")
            .then().statusCode(200)
            .body("refresh.operational", is("10second"))
            .body("refresh.metrics", is("30second"))
            .body("refresh.caseDetail", is("5second"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement GovernancePreferencesResource**

As specified in the design spec §C3.

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Run devtown tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

- [ ] **Step 6: Commit**

```
feat(#173): add GovernancePreferencesResource for operator-configurable refresh intervals

Exposes dashboard refresh rates via platform PreferenceProvider with sensible
defaults: 10s operational, 30s metrics, 5s case detail.

Refs casehubio/devtown#173
```

---

### Task 12: Frontend — datasets.ts refactor + refreshTime + activation context column

**Files:**
- Modify: `devtown/app/src/main/webui/src/datasets.ts`
- Modify: `devtown/app/src/main/webui/src/views/reviews.ts`
- Modify: `devtown/app/src/main/webui/src/index.ts` (wire preferences)

**Interfaces:**
- Consumes: `GET /api/governance/preferences` (Task 11), `PlanItemResponse.activationContext` (Task 10)

- [ ] **Step 1: Refactor datasets.ts**

Keep the `rest()` helper. Make datasets a function that accepts preferences:

```typescript
import { bind, restSource } from "@casehubio/pages-ui";
import type { DataSetId } from "@casehubio/pages-data";

function rest(id: string, url: string, opts?: { dataPath?: string; expression?: string; refreshTime?: string }) {
  return bind(id, restSource(url, id as DataSetId, opts));
}

export function createDatasets(prefs: Record<string, string>) {
  const opRefresh = prefs["refresh.operational"] ?? undefined;
  const metRefresh = prefs["refresh.metrics"] ?? undefined;
  const caseRefresh = prefs["refresh.caseDetail"] ?? undefined;

  return [
    rest("queue-status", "/api/governance/queue-status", { dataPath: "reviews", refreshTime: opRefresh }),
    rest("problems", "/api/governance/problems?threshold_minutes=0", { dataPath: "items", refreshTime: opRefresh }),
    rest("merge-queue", "/api/governance/merge-queue", { dataPath: "queuedPrs", refreshTime: opRefresh }),
    rest("active-batches", "/api/governance/merge-queue", { dataPath: "activeBatches", refreshTime: opRefresh }),
    rest("triage", "/api/governance/triage", { dataPath: "items", refreshTime: opRefresh }),

    rest("system-health", "/api/governance/system-health", { expression: "[$]", refreshTime: metRefresh }),
    rest("merge-queue-metrics", "/api/governance/merge-queue/metrics", { expression: "[$]", refreshTime: metRefresh }),
    rest("reviewers", "/api/governance/reviewers", { dataPath: "items", refreshTime: metRefresh }),

    rest("recent-events", "/api/governance/recent-events?limit=100", { refreshTime: opRefresh }),

    rest("case-definitions", "/api/v1/case-definitions"),

    rest("plan-items", "/api/v1/cases/#{row.caseId}/plan-items", { refreshTime: caseRefresh }),
    rest("goal-status", "/api/v1/cases/#{row.caseId}/goals", { refreshTime: caseRefresh }),
    rest("case-context", "/api/v1/cases/#{row.caseId}/context", { refreshTime: caseRefresh }),
  ];
}
```

- [ ] **Step 2: Wire preferences in index.ts**

In `index.ts`, fetch preferences before constructing the site:
```typescript
const prefs = await fetch("/api/governance/preferences")
  .then(r => r.ok ? r.json() : {})
  .catch(() => ({}));

const app = page("DevTown", ...views, { datasets: createDatasets(prefs) });
```

- [ ] **Step 3: Add activationContext column to reviews.ts**

In the Plan Items section, add `col("activationContext")`:
```typescript
dataTable({
  lookup: lookup("plan-items", groupBy("planItemId",
    col("bindingName"), col("targetType"),
    col("status"), col("executorName"), col("createdAt"),
    col("activationContext"),
  )),
  sortable: true,
}),
```

- [ ] **Step 4: TypeScript type-check**

Run: `cd /Users/mdproctor/claude/casehub/devtown/app/src/main/webui && npm run typecheck`
Expected: no errors

- [ ] **Step 5: Build**

Run: `cd /Users/mdproctor/claude/casehub/devtown/app/src/main/webui && npm run build`
Expected: BUILD SUCCESS

- [ ] **Step 6: Full Maven build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

- [ ] **Step 7: Commit**

```
feat(#173): dataset refresh intervals and activation context column

Datasets now fetch preferences at load time for operator-configurable
refresh rates. Plan Items table includes activationContext column showing
the changed layer content that triggered each binding.

Refs casehubio/devtown#170, casehubio/devtown#173
```

---

## Verification Checklist

After all tasks:

- [ ] Engine full build green: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/engine/pom.xml`
- [ ] Devtown full build green: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
- [ ] `npm run typecheck` passes
- [ ] `npm run build` passes
- [ ] No `PlanItemCompletedEvent`, `PlanItemFaultedEvent`, or `PlanItemRejectedEvent` references remain in compilable code
- [ ] Visual: Plan Items table shows activationContext column in dev mode
- [ ] Visual: Operational datasets refresh at ~10s intervals
