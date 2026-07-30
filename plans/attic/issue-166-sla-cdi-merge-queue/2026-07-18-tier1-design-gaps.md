# Tier 1 Design Gaps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #164 — merge-failed goal terminates case before rollback-human-escalation can fire
**Issue group:** #164, #161, #163, #162

**Goal:** Fix four interconnected design gaps in the coordinated change lifecycle found by the E2E test (#160).

**Architecture:** Engine changes (#161, #163) go first — they add goal metadata to CaseLifecycleEvent and signal metadata to EventLog. Devtown changes (#164 YAML, observer, hydrator, tests) follow. All engine changes are additive (new record fields, new default method overload). Two repos: casehub-engine and casehub-devtown.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine API/SPI, JQ-style YAML goal conditions

## Global Constraints

- Engine API changes must be backward-compatible default methods (existing callers compile without changes)
- CaseLifecycleEvent is a record — new fields go at the end, factory methods get new overloads
- All YAML condition expressions use jq syntax
- Tests use Awaitility for async assertions, AssertJ for fluent assertions, Mockito for unit tests
- Engine repo: `/Users/mdproctor/claude/casehub/engine`
- Devtown repo: `/Users/mdproctor/claude/casehub/devtown`
- IntelliJ workspace: devtown + engine (already open)

---

### Task 1: Add goal metadata to CaseLifecycleEvent (engine — #161)

**Files:**
- Modify: `engine` `common/src/main/java/io/casehub/engine/common/spi/event/CaseLifecycleEvent.java`
- Modify: `engine` `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`
- Test: `engine` — existing tests must still compile

**Interfaces:**
- Produces: `CaseLifecycleEvent` record with two new fields: `satisfiedGoalName` (String, nullable), `satisfiedGoalKind` (String, nullable). New factory method `of(CaseInstance, String, String, String, String, String, String, String)` that accepts goal metadata.

- [ ] **Step 1: Add fields to CaseLifecycleEvent record**

Add `satisfiedGoalName` and `satisfiedGoalKind` as the last two fields of the record. Add a new factory method that accepts them. Update the existing 6-arg `of(CaseInstance, ...)` factory to delegate with nulls.

In `common/src/main/java/io/casehub/engine/common/spi/event/CaseLifecycleEvent.java`, use `ide_edit_member` to replace the record declaration:

```java
public record CaseLifecycleEvent(
    UUID caseId,
    String tenancyId,
    String commandType,
    String eventType,
    String caseStatus,
    String actorId,
    String actorRole,
    String traceId,
    String caseDefinitionName,
    String namespace,
    JsonNode contextSnapshot,
    String satisfiedGoalName,
    String satisfiedGoalKind) {

  public static CaseLifecycleEvent of(
      CaseInstance caseInstance,
      String commandType,
      String eventType,
      String actorId,
      String actorRole,
      String traceId) {
    return of(caseInstance, commandType, eventType, actorId, actorRole, traceId, null, null);
  }

  public static CaseLifecycleEvent of(
      CaseInstance caseInstance,
      String commandType,
      String eventType,
      String actorId,
      String actorRole,
      String traceId,
      String satisfiedGoalName,
      String satisfiedGoalKind) {
    CaseMetaModel mm = caseInstance.getCaseMetaModel();
    String defName = mm != null ? mm.getName() : null;
    String ns = mm != null ? mm.getNamespace() : null;
    JsonNode snapshot = null;
    if (caseInstance.getCaseContext() != null) {
      snapshot = caseInstance.getCaseContext().layer(ContextLayer.WORKING).asJsonNode();
    }
    return new CaseLifecycleEvent(
        caseInstance.getUuid(),
        caseInstance.tenancyId,
        commandType,
        eventType,
        caseInstance.getState() != null ? caseInstance.getState().name() : null,
        actorId,
        actorRole,
        traceId,
        defName,
        ns,
        snapshot,
        satisfiedGoalName,
        satisfiedGoalKind);
  }

  public static CaseLifecycleEvent of(
      UUID caseId,
      String tenancyId,
      String commandType,
      String eventType,
      String caseStatus,
      String actorId,
      String actorRole,
      String traceId) {
    return new CaseLifecycleEvent(
        caseId, tenancyId, commandType, eventType, caseStatus,
        actorId, actorRole, traceId, null, null, null, null, null);
  }
}
```

- [ ] **Step 2: Propagate goal metadata in CaseStatusChangedHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`, update the `lifecycleEvents.fireAsync()` call (around line 194) to use the new 8-arg factory:

Replace the existing `CaseLifecycleEvent.of(...)` call in the `.chain()` block:

```java
lifecycleEvents.fireAsync(
    CaseLifecycleEvent.of(
        caseInstance,
        resolveCommandType(newState),
        resolveEventType(newState),
        null,
        "System",
        traceId,
        event.satisfiedGoalName(),
        event.satisfiedGoalKind()))
```

- [ ] **Step 3: Fix all compile errors from new record fields**

Run `ide_build_project` on engine. Any call site constructing `CaseLifecycleEvent` directly (not through factory methods) needs the two extra null args. Check:
- Test files constructing `CaseLifecycleEvent` with `new CaseLifecycleEvent(...)` — add `, null, null` at the end
- Use `ide_find_references` on the `CaseLifecycleEvent` constructor to find all call sites

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,common,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run engine tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,common,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml -q`
Expected: All tests pass

- [ ] **Step 5: Commit engine changes**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/ common/ runtime/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#161): propagate satisfiedGoalName/Kind through CaseLifecycleEvent

CaseStatusChanged already carries goal metadata for EventLog and
CaseOutcomeObserver but did not propagate it to CaseLifecycleEvent.
External CDI observers (e.g. CoordinatedChangeObserver) can now
distinguish success-COMPLETED from failure-COMPLETED.

Refs casehubio/devtown#161"
```

---

### Task 2: Add signal metadata overload (engine — #163)

**Files:**
- Modify: `engine` `api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java`
- Modify: `engine` `common/src/main/java/io/casehub/engine/common/internal/event/SignalReceivedEvent.java`
- Modify: `engine` `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java`
- Modify: `engine` `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java`
- Modify: `engine` `runtime/src/main/java/io/casehub/engine/internal/engine/handler/SignalReceivedEventHandler.java`
- Test: `engine` — unit test for metadata in EventLog

**Interfaces:**
- Produces: `CaseHubRuntime.signal(UUID caseId, String path, Object value, Map<String, Object> signalMetadata)` default method. `SignalReceivedEvent` gains a `signalMetadata` field (nullable `Map<String, Object>`).

- [ ] **Step 1: Add signalMetadata field to SignalReceivedEvent**

In `common/src/main/java/io/casehub/engine/common/internal/event/SignalReceivedEvent.java`, add `Map<String, Object> signalMetadata` as the last field:

```java
public record SignalReceivedEvent(
    UUID caseId,
    String tenancyId,
    String path,
    Object value,
    String triggerChannelId,
    String triggerCorrelationId,
    Map<String, Object> signalMetadata) {
  // ... existing compact constructor unchanged ...

  public SignalReceivedEvent(UUID caseId, String tenancyId, String path, Object value) {
    this(caseId, tenancyId, path, value, null, null, null);
  }
}
```

Update the existing 6-arg canonical constructor calls — the 6-arg constructor `(caseId, tenancyId, path, value, triggerChannelId, triggerCorrelationId)` needs to become a convenience constructor delegating to the 7-arg canonical:

```java
public SignalReceivedEvent(UUID caseId, String tenancyId, String path, Object value,
    String triggerChannelId, String triggerCorrelationId) {
  this(caseId, tenancyId, path, value, triggerChannelId, triggerCorrelationId, null);
}
```

- [ ] **Step 2: Add signal overload to CaseHubRuntime**

In `api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java`, add after the existing `signal(UUID, String, Object)`:

```java
/**
 * Signals a case context update with caller-provided metadata that is stored
 * in the EventLog entry. Use for cross-case provenance linking.
 */
default CompletionStage<Void> signal(
    UUID caseId, String path, Object value, Map<String, Object> signalMetadata) {
  return signal(caseId, path, value);
}
```

- [ ] **Step 3: Implement in CaseHubReactor and CaseHubRuntimeImpl**

In `CaseHubReactor.java`, add a new `signal` overload:

```java
Uni<Void> signal(UUID caseId, String path, Object value, Map<String, Object> signalMetadata) {
  String tenancyId = requireInstance(caseId).tenancyId;
  return eventBus
      .<Void>request(
          SIGNAL_RECEIVED,
          new SignalReceivedEvent(caseId, tenancyId, path, value, null, null, signalMetadata))
      .replaceWithVoid();
}
```

In `CaseHubRuntimeImpl.java`, add:

```java
@Override
public CompletionStage<Void> signal(
    UUID caseId, String path, Object value, Map<String, Object> signalMetadata) {
  return reactor.signal(caseId, path, value, signalMetadata).subscribeAsCompletionStage();
}
```

- [ ] **Step 4: Merge signalMetadata into EventLog in SignalReceivedEventHandler**

In `SignalReceivedEventHandler.java`, update `buildSignalEventLog` to accept and merge metadata:

```java
private EventLog buildSignalEventLog(CaseInstance instance, JsonNode diff,
    Map<String, Object> signalMetadata) {
  EventLog eventLog = new EventLog();
  eventLog.setCaseId(instance.getUuid());
  eventLog.setEventType(CaseHubEventType.SIGNAL_RECEIVED);
  eventLog.setStreamType(EventStreamType.CASE);
  eventLog.setTimestamp(Instant.now());
  eventLog.setPayload(OBJECT_MAPPER.createObjectNode().set("patch", diff.deepCopy()));
  java.util.Map<String, String> metadata = new java.util.HashMap<>();
  metadata.put("origin", io.casehub.api.model.event.ExecutionOrigin.SIGNAL.name());
  ObjectNode metadataNode = OBJECT_MAPPER.valueToTree(metadata);
  if (signalMetadata != null) {
    OBJECT_MAPPER.valueToTree(signalMetadata).fields()
        .forEachRemaining(e -> metadataNode.set(e.getKey(), e.getValue()));
  }
  eventLog.setMetadata(metadataNode);
  return eventLog;
}
```

Update the call site in `applySignalUnderLock` to pass `event.signalMetadata()`:

```java
EventLog eventLog = buildSignalEventLog(instance, diff, event.signalMetadata());
```

- [ ] **Step 5: Build and test engine**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,common,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml -q`
Expected: BUILD SUCCESS

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,common,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml -q`
Expected: All tests pass

- [ ] **Step 6: Commit engine changes**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/ common/ runtime/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#163): signal() overload with caller-provided metadata for EventLog

Adds CaseHubRuntime.signal(UUID, String, Object, Map<String,Object>)
that merges caller metadata into the EventLog entry. Enables cross-case
provenance linking — callers pass causedByCaseId and causedByEvent.

Refs casehubio/devtown#163"
```

---

### Task 3: Install engine SNAPSHOT and rebuild devtown

**Files:**
- No source changes — dependency resolution only

- [ ] **Step 1: Install engine to local Maven repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/engine/pom.xml -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 2: Rebuild devtown to pick up new engine API**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f /Users/mdproctor/claude/casehub/devtown/pom.xml -q`

This will fail with compile errors in `CoordinatedChangeObserverTest` because `CaseLifecycleEvent` now has 13 fields and the test constructs it with 11. Fix in Task 4.

- [ ] **Step 3: Sync IntelliJ**

Run `ide_sync_files` on devtown and `ide_reload_project` so IntelliJ picks up the new engine API.

---

### Task 4: Fix CoordinatedChangeObserver — failure-goal detection (#161)

**Files:**
- Modify: `devtown` `app/src/main/java/io/casehub/devtown/app/CoordinatedChangeObserver.java`
- Modify: `devtown` `app/src/test/java/io/casehub/devtown/app/CoordinatedChangeObserverTest.java`

**Interfaces:**
- Consumes: `CaseLifecycleEvent.satisfiedGoalKind()` (from Task 1)
- Produces: Observer correctly routes failure-COMPLETED to `reviewFaulted` signal

- [ ] **Step 1: Write failing tests for failure-goal COMPLETED**

In `CoordinatedChangeObserverTest.java`, update the `lifecycleEvent` helper to accept goal metadata and add new tests:

```java
private CaseLifecycleEvent lifecycleEvent(UUID caseId, String status) {
    return new CaseLifecycleEvent(caseId, null, null, null, status,
        null, null, null, null, null, null, null, null);
}

private CaseLifecycleEvent lifecycleEvent(UUID caseId, String status,
    String goalName, String goalKind) {
    return new CaseLifecycleEvent(caseId, null, null, null, status,
        null, null, null, null, null, null, goalName, goalKind);
}
```

Add test:

```java
@Test
void failureGoalCompleted_signalsReviewFaulted() {
    observer.onCaseLifecycle(
        lifecycleEvent(reviewA, "COMPLETED", "review-abandoned", "failure"));
    verify(runtime).signal(eq(parentId), eq("reviewFaulted"), any(Map.class));
    verify(runtime, never()).signal(eq(parentId),
        eq("completedReviews.casehubio/engine"), any());
}
```

Update existing `reviewCompletion_signalsParentContext` to pass success goal:

```java
@Test
void reviewCompletion_signalsParentContext() {
    observer.onCaseLifecycle(
        lifecycleEvent(reviewA, "COMPLETED", "pr-approved", "success"));
    verify(runtime).signal(eq(parentId),
        eq("completedReviews.casehubio/engine"), any(Map.class));
}

@Test
void allReviewsComplete_signalsAllReviewsCompleted() {
    observer.onCaseLifecycle(
        lifecycleEvent(reviewA, "COMPLETED", "pr-approved", "success"));
    observer.onCaseLifecycle(
        lifecycleEvent(reviewB, "COMPLETED", "pr-approved", "success"));
    verify(runtime).signal(eq(parentId), eq("allReviewsCompleted"), eq(true));
}
```

- [ ] **Step 2: Run tests — expect failures**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -Dtest=CoordinatedChangeObserverTest -q`
Expected: `failureGoalCompleted_signalsReviewFaulted` FAILS (observer still treats all COMPLETED as success)

- [ ] **Step 3: Implement observer fix**

Replace the `onCaseLifecycle` method in `CoordinatedChangeObserver.java`:

```java
void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
    if (event.caseStatus() == null) return;
    var entry = tracker.findByReviewCaseId(event.caseId());
    if (entry == null) return;
    if (tracker.isParentTerminal(entry.parentCaseId())) return;

    String status = event.caseStatus();

    if ("COMPLETED".equals(status) && "failure".equals(event.satisfiedGoalKind())) {
        caseHubRuntime.signal(entry.parentCaseId(), "reviewFaulted",
            Map.of("repo", entry.repo(), "reason", event.satisfiedGoalName()));
    } else if ("COMPLETED".equals(status)) {
        if (tracker.markCompleted(entry.parentCaseId(), entry.repo())) {
            caseHubRuntime.signal(entry.parentCaseId(),
                "completedReviews." + entry.repo(),
                Map.of("status", "completed", "reviewCaseId", entry.reviewCaseId().toString()));
        }
        if (tracker.tryTransitionToAllCompleted(entry.parentCaseId())) {
            caseHubRuntime.signal(entry.parentCaseId(), "allReviewsCompleted", true);
        }
    } else if ("FAULTED".equals(status) || "CANCELLED".equals(status)) {
        caseHubRuntime.signal(entry.parentCaseId(), "reviewFaulted",
            Map.of("repo", entry.repo(), "reason", status));
    }
}
```

Remove the `TERMINAL_SUCCESS` and `TERMINAL_FAILURE` static sets — no longer needed.

- [ ] **Step 4: Run tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -Dtest=CoordinatedChangeObserverTest -q`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/CoordinatedChangeObserver.java app/src/test/java/io/casehub/devtown/app/CoordinatedChangeObserverTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "fix(#161): observer distinguishes failure-goal COMPLETED from success

Uses CaseLifecycleEvent.satisfiedGoalKind() to route failure-goal
COMPLETED to reviewFaulted signal instead of completedReviews.
Removes TERMINAL_SUCCESS/TERMINAL_FAILURE sets — status+goalKind
is the complete discriminator.

Closes #161"
```

---

### Task 5: Add signal provenance to observer (#163)

**Files:**
- Modify: `devtown` `app/src/main/java/io/casehub/devtown/app/CoordinatedChangeObserver.java`
- Modify: `devtown` `app/src/test/java/io/casehub/devtown/app/CoordinatedChangeObserverTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime.signal(UUID, String, Object, Map<String, Object>)` (from Task 2)

- [ ] **Step 1: Write failing test for provenance metadata**

Add test in `CoordinatedChangeObserverTest.java`:

```java
@Test
void signalCarriesProvenanceMetadata() {
    when(runtime.signal(any(UUID.class), anyString(), any(), any(Map.class)))
        .thenReturn(CompletableFuture.completedFuture(null));
    observer.onCaseLifecycle(
        lifecycleEvent(reviewA, "COMPLETED", "pr-approved", "success"));

    @SuppressWarnings("unchecked")
    var captor = ArgumentCaptor.forClass(Map.class);
    verify(runtime).signal(eq(parentId), eq("completedReviews.casehubio/engine"),
        any(), captor.capture());
    Map<String, Object> provenance = captor.getValue();
    assertThat(provenance).containsEntry("causedByCaseId", reviewA.toString());
}
```

Update setUp to also stub the 4-arg signal:

```java
when(runtime.signal(any(UUID.class), anyString(), any(), any(Map.class)))
    .thenReturn(CompletableFuture.completedFuture(null));
```

- [ ] **Step 2: Run test — expect failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -Dtest=CoordinatedChangeObserverTest#signalCarriesProvenanceMetadata -q`
Expected: FAIL — observer still uses 3-arg signal()

- [ ] **Step 3: Update observer to pass provenance on all signals**

Replace all `caseHubRuntime.signal(...)` calls in `onCaseLifecycle` to use the 4-arg overload with provenance:

```java
void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
    if (event.caseStatus() == null) return;
    var entry = tracker.findByReviewCaseId(event.caseId());
    if (entry == null) return;
    if (tracker.isParentTerminal(entry.parentCaseId())) return;

    String status = event.caseStatus();
    var provenance = Map.<String, Object>of(
        "causedByCaseId", entry.reviewCaseId().toString(),
        "causedByEvent", event.eventType() != null ? event.eventType() : status);

    if ("COMPLETED".equals(status) && "failure".equals(event.satisfiedGoalKind())) {
        caseHubRuntime.signal(entry.parentCaseId(), "reviewFaulted",
            Map.of("repo", entry.repo(), "reason", event.satisfiedGoalName()),
            provenance);
    } else if ("COMPLETED".equals(status)) {
        if (tracker.markCompleted(entry.parentCaseId(), entry.repo())) {
            caseHubRuntime.signal(entry.parentCaseId(),
                "completedReviews." + entry.repo(),
                Map.of("status", "completed", "reviewCaseId", entry.reviewCaseId().toString()),
                provenance);
        }
        if (tracker.tryTransitionToAllCompleted(entry.parentCaseId())) {
            caseHubRuntime.signal(entry.parentCaseId(), "allReviewsCompleted", true,
                provenance);
        }
    } else if ("FAULTED".equals(status) || "CANCELLED".equals(status)) {
        caseHubRuntime.signal(entry.parentCaseId(), "reviewFaulted",
            Map.of("repo", entry.repo(), "reason", status),
            provenance);
    }
}
```

- [ ] **Step 4: Run tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -Dtest=CoordinatedChangeObserverTest -q`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/CoordinatedChangeObserver.java app/src/test/java/io/casehub/devtown/app/CoordinatedChangeObserverTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#163): observer passes cross-case provenance on parent signals

Every signal to the parent case now carries causedByCaseId and
causedByEvent metadata, stored in the EventLog entry for
cross-case provenance linking.

Closes #163"
```

---

### Task 6: Fix merge-failed goal condition (#164)

**Files:**
- Modify: `devtown` `app/src/main/resources/casehub/devtown/coordinated-change.yaml`

**Interfaces:**
- Produces: `merge-failed` goal waits for rollback chain resolution before firing

- [ ] **Step 1: Update the goal condition**

In `coordinated-change.yaml`, replace the `merge-failed` goal (lines 31-35):

```yaml
    - name: merge-failed
      kind: failure
      condition: >-
        .mergeResults != null and
        (.mergeResults | any(.status == "failed")) and
        ((.rollbackResults != null and (.rollbackResults | all(.status == "success")))
          or .rollbackEscalation != null
          or .abandoned == true)
```

- [ ] **Step 2: Verify YAML is well-formed**

Run: `python3 -c "import yaml; yaml.safe_load(open('/Users/mdproctor/claude/casehub/devtown/app/src/main/resources/casehub/devtown/coordinated-change.yaml'))" && echo OK`
Expected: OK

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/resources/casehub/devtown/coordinated-change.yaml
git -C /Users/mdproctor/claude/casehub/devtown commit -m "fix(#164): merge-failed goal waits for rollback chain to resolve

The goal previously fired as soon as any merge failed, terminating
the case before rollback-human-escalation could fire. Now waits
for rollback to succeed, human escalation to complete, or explicit
abandonment.

Closes #164"
```

---

### Task 7: Implement TrackerHydrator restart recovery (#162)

**Files:**
- Modify: `devtown` `app/src/main/java/io/casehub/devtown/app/CoordinatedChangeTrackerHydrator.java`
- Modify: `devtown` `app/src/test/java/io/casehub/devtown/app/CoordinatedChangeTrackerHydratorTest.java`

**Interfaces:**
- Consumes: `CaseInstanceRepository.findByNamespaceAndName(String, String, String)`, `CurrentPrincipal.tenancyId()`

- [ ] **Step 1: Write failing tests**

Replace `CoordinatedChangeTrackerHydratorTest.java` with tests that verify startup hydration from CaseInstanceRepository:

```java
package io.casehub.devtown.app;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

import io.casehub.api.context.CaseContext;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.platform.api.identity.CurrentPrincipal;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;

class CoordinatedChangeTrackerHydratorTest {

    CoordinatedChangeTracker tracker;
    CaseInstanceRepository repository;
    CurrentPrincipal principal;
    CoordinatedChangeTrackerHydrator hydrator;

    @BeforeEach
    void setUp() {
        tracker = new CoordinatedChangeTracker();
        repository = mock(CaseInstanceRepository.class);
        principal = mock(CurrentPrincipal.class);
        when(principal.tenancyId()).thenReturn("test-tenant");
        hydrator = new CoordinatedChangeTrackerHydrator();
        hydrator.tracker = tracker;
        hydrator.caseInstanceRepository = repository;
        hydrator.principal = principal;
    }

    @Test
    void hydrate_rebuildsTrackerFromActiveCases() {
        UUID parentId = UUID.randomUUID();
        UUID reviewA = UUID.randomUUID();
        UUID reviewB = UUID.randomUUID();

        var instance = mockCaseInstance(parentId, CaseStatus.RUNNING,
            Map.of("reviewCases", Map.of(
                "casehubio/engine", reviewA.toString(),
                "casehubio/platform", reviewB.toString())));

        when(repository.findByNamespaceAndName("devtown", "coordinated-change", "test-tenant"))
            .thenReturn(List.of(instance));

        hydrator.hydrate();

        assertThat(tracker.findByReviewCaseId(reviewA)).isNotNull();
        assertThat(tracker.findByReviewCaseId(reviewA).repo()).isEqualTo("casehubio/engine");
        assertThat(tracker.findByReviewCaseId(reviewB)).isNotNull();
        assertThat(tracker.findByReviewCaseId(reviewB).repo()).isEqualTo("casehubio/platform");
    }

    @Test
    void hydrate_skipsTerminalCases() {
        UUID parentId = UUID.randomUUID();
        var instance = mockCaseInstance(parentId, CaseStatus.COMPLETED,
            Map.of("reviewCases", Map.of("casehubio/engine", UUID.randomUUID().toString())));

        when(repository.findByNamespaceAndName("devtown", "coordinated-change", "test-tenant"))
            .thenReturn(List.of(instance));

        hydrator.hydrate();

        assertThat(tracker.findReviewCaseIds(parentId)).isEmpty();
    }

    @Test
    void hydrate_noActiveCases_trackerStaysEmpty() {
        when(repository.findByNamespaceAndName("devtown", "coordinated-change", "test-tenant"))
            .thenReturn(List.of());

        hydrator.hydrate();

        // no assertions needed beyond no exception — tracker is empty
    }

    @Test
    void hydrate_marksAlreadyCompletedRepos() {
        UUID parentId = UUID.randomUUID();
        UUID reviewA = UUID.randomUUID();
        UUID reviewB = UUID.randomUUID();

        var instance = mockCaseInstance(parentId, CaseStatus.RUNNING,
            Map.of(
                "reviewCases", Map.of(
                    "casehubio/engine", reviewA.toString(),
                    "casehubio/platform", reviewB.toString()),
                "completedReviews", Map.of(
                    "casehubio/engine", Map.of("status", "completed"))));

        when(repository.findByNamespaceAndName("devtown", "coordinated-change", "test-tenant"))
            .thenReturn(List.of(instance));

        hydrator.hydrate();

        assertThat(tracker.tryTransitionToAllCompleted(parentId)).isFalse();
    }

    @SuppressWarnings("unchecked")
    private CaseInstance mockCaseInstance(UUID caseId, CaseStatus status,
        Map<String, Object> contextData) {
        var instance = mock(CaseInstance.class);
        when(instance.getUuid()).thenReturn(caseId);
        when(instance.getState()).thenReturn(status);
        var metaModel = mock(CaseMetaModel.class);
        when(metaModel.getNamespace()).thenReturn("devtown");
        when(metaModel.getName()).thenReturn("coordinated-change");
        when(instance.getCaseMetaModel()).thenReturn(metaModel);
        var context = mock(CaseContext.class);
        when(context.get(anyString())).thenAnswer(inv -> contextData.get(inv.getArgument(0)));
        when(instance.getCaseContext()).thenReturn(context);
        return instance;
    }
}
```

- [ ] **Step 2: Run tests — expect failures**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -Dtest=CoordinatedChangeTrackerHydratorTest -q`
Expected: Compile errors / test failures (hydrator doesn't implement hydrate() yet)

- [ ] **Step 3: Implement the hydrator**

Replace `CoordinatedChangeTrackerHydrator.java`:

```java
package io.casehub.devtown.app;

import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class CoordinatedChangeTrackerHydrator {

    private static final Logger LOG = Logger.getLogger(CoordinatedChangeTrackerHydrator.class);
    private static final Set<CaseStatus> TERMINAL = Set.of(
        CaseStatus.COMPLETED, CaseStatus.FAULTED, CaseStatus.CANCELLED);

    @Inject CoordinatedChangeTracker tracker;
    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject CurrentPrincipal principal;

    void onStartup(@Observes StartupEvent event) {
        hydrate();
    }

    @SuppressWarnings("unchecked")
    void hydrate() {
        String tenancyId = principal.tenancyId();
        var instances = caseInstanceRepository.findByNamespaceAndName(
            "devtown", "coordinated-change", tenancyId);
        int count = 0;

        for (CaseInstance instance : instances) {
            if (TERMINAL.contains(instance.getState())) continue;
            var ctx = instance.getCaseContext();
            if (ctx == null) continue;

            var reviewCases = (Map<String, String>) ctx.get("reviewCases");
            if (reviewCases == null) continue;

            for (var entry : reviewCases.entrySet()) {
                tracker.register(instance.getUuid(), entry.getKey(),
                    UUID.fromString(entry.getValue()));
            }

            var completedReviews = (Map<String, Object>) ctx.get("completedReviews");
            if (completedReviews != null) {
                for (String repo : completedReviews.keySet()) {
                    tracker.markCompleted(instance.getUuid(), repo);
                }
            }

            count++;
        }

        if (count > 0) {
            LOG.infof("Hydrated %d active coordinated changes from durable state", count);
        }
    }
}
```

- [ ] **Step 4: Run tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -Dtest=CoordinatedChangeTrackerHydratorTest -q`
Expected: All tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/CoordinatedChangeTrackerHydrator.java app/src/test/java/io/casehub/devtown/app/CoordinatedChangeTrackerHydratorTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#162): implement CoordinatedChangeTrackerHydrator restart recovery

Hydrates the in-memory tracker from CaseInstanceRepository on startup.
Queries active coordinated-change cases, extracts reviewCases map,
and marks already-completed repos. Follows the PrReviewCaseTracker-
Hydrator pattern.

Closes #162"
```

---

### Task 8: Update E2E test — failure-goal path and rollback escalation

**Files:**
- Modify: `devtown` `app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java`

**Interfaces:**
- Consumes: All fixes from Tasks 1-7

- [ ] **Step 1: Update driveReviewToFault to use failure goal**

Replace the `driveReviewToFault` helper to signal `pr.status = "closed"` instead of `cancelCase()`:

```java
private void driveReviewToFault(UUID reviewCaseId) {
    caseHubRuntime.signal(reviewCaseId, "pr.status", "closed")
        .toCompletableFuture().join();
}
```

- [ ] **Step 2: Update faultPath test — verify failure-COMPLETED produces reviewFaulted**

In `faultPath_reviewFaults_parentTerminates_remainingStaysCompleted`, update the assertion for the fault propagation phase. The sub-case now completes via `review-abandoned` (COMPLETED), not CANCELLED:

```java
// ── Phase 3: Fault propagation ───────────────────────────────
await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
    assertThat(contextValue(parentCaseId, "reviewFaulted.repo"))
            .isEqualTo("casehubio/platform");
    assertThat(contextValue(parentCaseId, "reviewFaulted.reason"))
            .isEqualTo("review-abandoned");
});
```

The `reason` changes from `"CANCELLED"` to `"review-abandoned"` (the goal name).

- [ ] **Step 3: Update rollbackFailure test — verify human escalation fires**

In `rollbackFailure_mergeConflict_parentTerminatesAfterRollback`, replace the Phase 5 section:

```java
// ── Phase 5: Human escalation — rollback conflict requires manual intervention ──
await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
    var workItems = workItemQueries.findActiveByGroup("human-oversight:general");
    assertThat(workItems).as("Human escalation WorkItem should be created").isNotEmpty();
    assertThat(workItems.get(0).getTitle()).contains("Coordinated rollback failed");
});

// ── Phase 6: Resolve human escalation ────────────────────────
var workItems = workItemQueries.findActiveByGroup("human-oversight:general");
caseHubRuntime.signal(parentCaseId, "rollbackEscalation",
    Map.of("outcome", "RESOLVED")).toCompletableFuture().join();

// ── Phase 7: Terminal — merge-failed goal fires after escalation ──
awaitCaseTerminal(parentCaseId);

var parentInstance = caseInstanceRepository.findByUuid(parentCaseId);
assertThat(parentInstance.getState()).isEqualTo(CaseStatus.COMPLETED);
```

Remove the old comment about the design gap (lines 348-352).

- [ ] **Step 4: Add provenance assertion to happyPath test**

In `happyPath_allReviewsComplete_mergeSucceeds_parentCompleted`, add EventLog provenance assertion in the Phase 5 section:

```java
// Verify cross-case provenance in EventLog
var signals = allEvents.stream()
    .filter(e -> e.eventType() == CaseHubEventType.SIGNAL_RECEIVED)
    .filter(e -> e.metadata() != null && e.metadata().has("causedByCaseId"))
    .toList();
assertThat(signals).as("Signals should carry cross-case provenance").isNotEmpty();
```

- [ ] **Step 5: Run full E2E test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -Dtest=CrossRepoCoordinatedMergeTest -q`
Expected: All 5 tests pass

- [ ] **Step 6: Run full devtown test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml -q`
Expected: BUILD SUCCESS — all tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test(#164): E2E tests exercise failure-goal path and rollback escalation

driveReviewToFault now signals pr.status=closed (triggering
review-abandoned failure goal) instead of cancelCase(). Rollback
failure test verifies human escalation WorkItem fires and case
terminates only after resolution. Provenance assertions verify
cross-case EventLog linking.

Refs #164, #161, #163"
```
