# CI Status Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Handle GitHub `check_suite` and `check_run` webhook events to resolve the `ci-passing` goal from real CI data, with dual-mode CI (external/dispatched), SHA guard, and multi-suite aggregation.

**Architecture:** Extends the existing hexagonal webhook adapter (`github/` module) with two new event handlers that dispatch through the `PrReviewApplicationService` port. CI mode is config-driven — external mode pre-seeds `{ status: "pending" }` to suppress the `run-ci` binding; dispatched mode leaves `.ci` null so the binding fires. SHA guard reads from the case context (authoritative), not the tracker (cache).

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson (DTO deserialization), Mockito (unit tests), AssertJ (assertions)

**Spec:** `specs/2026-06-21-ci-status-integration-design.md` (rev 3)

**Build:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

---

## File Structure

| File | Module | Action | Responsibility |
|------|--------|--------|---------------|
| `review/src/main/java/.../LifecycleResult.java` | review | Modify | Add `STALE_EVENT` value |
| `review/src/main/java/.../PrReviewApplicationService.java` | review | Modify | Add `signalCiStatus()` and `signalCheckRun()` port methods |
| `github/src/main/java/.../GitHubCheckSuiteEvent.java` | github | Create | DTO for `check_suite` webhook payload |
| `github/src/main/java/.../GitHubCheckRunEvent.java` | github | Create | DTO for `check_run` webhook payload |
| `github/src/main/java/.../GitHubWebhookResource.java` | github | Modify | Add `check_suite`/`check_run` dispatch arms |
| `app/src/main/java/.../mcp/CaseInfo.java` | app | Modify | Add `withHeadSha()` method |
| `app/src/main/java/.../mcp/PrReviewCaseTracker.java` | app | Modify | Add `updateHeadSha()` method |
| `app/src/main/java/.../PrReviewService.java` | app | Modify | No-op `signalCiStatus()` and `signalCheckRun()` |
| `app/src/main/java/.../QhorusPrReviewService.java` | app | Modify | No-op `signalCiStatus()` and `signalCheckRun()` |
| `app/src/main/java/.../PrReviewCaseService.java` | app | Modify | Implement CI signal, SHA guard, CI mode, startReview/revisePr changes |
| `github/src/test/java/.../GitHubCheckSuiteEventTest.java` | github | Create | DTO deserialization tests |
| `github/src/test/java/.../GitHubCheckRunEventTest.java` | github | Create | DTO deserialization tests |
| `github/src/test/java/.../GitHubWebhookResourceTest.java` | github | Modify | Extend RecordingService, add check_suite/check_run webhook tests |
| `app/src/test/java/.../PrReviewCaseServiceCiStatusTest.java` | app | Create | signalCiStatus, signalCheckRun, SHA guard, aggregation |
| `app/src/test/java/.../PrReviewCaseServiceLifecycleTest.java` | app | Modify | Add CI invalidation + tracker sync to revisePr tests, CI mode in startReview |
| `app/src/test/java/.../mcp/PrReviewCaseTrackerTest.java` | app | Modify | Add updateHeadSha test |

All paths relative to `/Users/mdproctor/claude/casehub/devtown/`.

---

### Task 1: Port interface + LifecycleResult

**Files:**
- Modify: `review/src/main/java/io/casehub/devtown/review/LifecycleResult.java`
- Modify: `review/src/main/java/io/casehub/devtown/review/PrReviewApplicationService.java`

- [ ] **Step 1: Add STALE_EVENT to LifecycleResult**

```java
public enum LifecycleResult {
    UPDATED,
    NO_ACTIVE_CASE,
    ALREADY_COMPLETED,
    ALREADY_ABANDONED,
    STALE_EVENT
}
```

- [ ] **Step 2: Add signalCiStatus and signalCheckRun to port interface**

```java
package io.casehub.devtown.review;

import java.time.Instant;

public interface PrReviewApplicationService {
    PrReviewOutcome startReview(PrPayload pr);
    LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged);
    LifecycleResult closePr(String repo, int prNumber, boolean merged);
    LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion);
    LifecycleResult signalCheckRun(String repo, int prNumber, String headSha, String checkName, String conclusion, Instant completedAt);
}
```

- [ ] **Step 3: Compile review module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml compile -pl review --batch-mode`

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add review/src/main/java/io/casehub/devtown/review/LifecycleResult.java review/src/main/java/io/casehub/devtown/review/PrReviewApplicationService.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(review): add signalCiStatus, signalCheckRun port methods and STALE_EVENT

Port interface for CI status integration. STALE_EVENT distinguishes
'no case found' from 'case exists but event is for an outdated commit.'

Refs #86"
```

---

### Task 2: No-op implementations in PrReviewService and QhorusPrReviewService

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewService.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java`

- [ ] **Step 1: Add no-op implementations to PrReviewService**

Add after the existing `closePr()` method (after line 33):

```java
@Override
public LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion) {
    return LifecycleResult.NO_ACTIVE_CASE;
}

@Override
public LifecycleResult signalCheckRun(String repo, int prNumber, String headSha, String checkName, String conclusion, Instant completedAt) {
    return LifecycleResult.NO_ACTIVE_CASE;
}
```

Add `import java.time.Instant;` to the imports.

- [ ] **Step 2: Add no-op implementations to QhorusPrReviewService**

Add after the existing `closePr()` method (after line 139):

```java
@Override
public LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion) {
    return LifecycleResult.NO_ACTIVE_CASE;
}

@Override
public LifecycleResult signalCheckRun(String repo, int prNumber, String headSha, String checkName, String conclusion, Instant completedAt) {
    return LifecycleResult.NO_ACTIVE_CASE;
}
```

Add `import java.time.Instant;` to the imports.

- [ ] **Step 3: Compile app module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml compile -pl app --batch-mode`

Expected: BUILD SUCCESS (all implementations satisfy the port interface)

- [ ] **Step 4: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/PrReviewService.java app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): no-op signalCiStatus/signalCheckRun in PrReviewService, QhorusPrReviewService

Layer 1 @DefaultBean and Layer 3 @Priority(1) return NO_ACTIVE_CASE.
Only PrReviewCaseService @Priority(2) will have real implementation.

Refs #86"
```

---

### Task 3: CaseInfo.withHeadSha() and PrReviewCaseTracker.updateHeadSha()

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/CaseInfo.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/mcp/PrReviewCaseTrackerTest.java`

- [ ] **Step 1: Write the failing test for updateHeadSha**

Add to `PrReviewCaseTrackerTest.java`:

```java
@Test
void updateHeadSha_updatesPayloadSha_preservesOtherFields() {
    UUID caseId = UUID.randomUUID();
    var payload = new PrPayload("casehubio/devtown", 42, "oldsha", "main", 100, "octocat", List.of("src/Foo.java"));
    tracker.register(caseId, "tenant-1", payload);

    tracker.updateHeadSha(caseId, "newsha");

    CaseInfo updated = tracker.getCase(caseId);
    assertThat(updated.payload().headSha()).isEqualTo("newsha");
    assertThat(updated.payload().repo()).isEqualTo("casehubio/devtown");
    assertThat(updated.payload().prNumber()).isEqualTo(42);
    assertThat(updated.payload().baseRef()).isEqualTo("main");
    assertThat(updated.payload().linesChanged()).isEqualTo(100);
    assertThat(updated.payload().contributor()).isEqualTo("octocat");
    assertThat(updated.payload().changedPaths()).containsExactly("src/Foo.java");
    assertThat(updated.tenancyId()).isEqualTo("tenant-1");
    assertThat(updated.status()).isEqualTo(CaseTrackingStatus.RUNNING);
}
```

Add import for `CaseInfo` and `CaseTrackingStatus` if not already present.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.mcp.PrReviewCaseTrackerTest#updateHeadSha_updatesPayloadSha_preservesOtherFields" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — `updateHeadSha` method does not exist.

- [ ] **Step 3: Add withHeadSha to CaseInfo**

Add to `CaseInfo.java` after the existing `withStatus()` method:

```java
public CaseInfo withHeadSha(String newSha) {
    var updatedPayload = new PrPayload(
        payload.repo(), payload.prNumber(), newSha,
        payload.baseRef(), payload.linesChanged(),
        payload.contributor(), payload.changedPaths()
    );
    return new CaseInfo(caseId, tenancyId, updatedPayload, startedAt, lastEventAt, status);
}
```

- [ ] **Step 4: Add updateHeadSha to PrReviewCaseTracker**

Add to `PrReviewCaseTracker.java` after the existing `updateStatus()` method:

```java
public void updateHeadSha(UUID caseId, String newSha) {
    cases.computeIfPresent(caseId, (id, existing) -> existing.withHeadSha(newSha));
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.mcp.PrReviewCaseTrackerTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All PrReviewCaseTrackerTest tests pass.

- [ ] **Step 6: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/mcp/CaseInfo.java app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java app/src/test/java/io/casehub/devtown/app/mcp/PrReviewCaseTrackerTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): CaseInfo.withHeadSha, PrReviewCaseTracker.updateHeadSha

Tracker SHA sync — revisePr() will call updateHeadSha() alongside the
context signal so SHA guard reads current HEAD from either source.
Case context remains authoritative; tracker is a cache.

Refs #86"
```

---

### Task 4: PrReviewCaseService — CI mode, startReview pre-seed, revisePr invalidation

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceLifecycleTest.java`

- [ ] **Step 1: Write failing tests for CI mode in startReview and revisePr**

Add to `PrReviewCaseServiceLifecycleTest.java`. The test setup needs updating first — add `ciMode` field injection and a `startCase` mock:

Add new tests after the existing ones:

```java
@Test
void revisePr_externalMode_resetsCiToPending() {
    service.ciMode = "external";
    service.revisePr("casehubio/devtown", 42, "newsha", 200);

    verify(caseHub).signal(eq(caseId), eq("ci"), eq(Map.of("status", "pending")));
}

@Test
void revisePr_dispatchedMode_nullsCi() {
    service.ciMode = "dispatched";
    service.revisePr("casehubio/devtown", 42, "newsha", 200);

    verify(caseHub).signal(eq(caseId), eq("ci"), isNull());
}

@Test
void revisePr_updatesTrackerHeadSha() {
    service.ciMode = "external";
    service.revisePr("casehubio/devtown", 42, "newsha", 200);

    assertThat(tracker.getCase(caseId).payload().headSha()).isEqualTo("newsha");
}

@Test
void revisePr_ciInvalidation_afterAnalysisInvalidation() {
    service.ciMode = "external";
    service.revisePr("casehubio/devtown", 42, "newsha", 200);

    InOrder inOrder = inOrder(caseHub);
    inOrder.verify(caseHub).signal(eq(caseId), eq("performanceAnalysis"), isNull());
    inOrder.verify(caseHub).signal(eq(caseId), eq("ci"), eq(Map.of("status", "pending")));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseServiceLifecycleTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — `ciMode` field does not exist, no ci signal in revisePr.

- [ ] **Step 3: Implement CI mode in PrReviewCaseService**

Add to PrReviewCaseService after the existing `@ConfigProperty` fields (after line 46):

```java
@ConfigProperty(name = "devtown.ci.mode", defaultValue = "external")
String ciMode;
```

Modify `startReview()` — insert after `initialContext.put("memory", ...)` (line 76) and before `caseHub.startCase()` (line 78):

```java
if ("external".equals(ciMode)) {
    initialContext.put("ci", Map.of("status", "pending"));
}
```

Modify `revisePr()` — insert after the last analysis invalidation (`performanceAnalysis`, line 102) and before the return:

```java
if ("external".equals(ciMode)) {
    caseHub.signal(caseId, "ci", Map.of("status", "pending"));
} else {
    caseHub.signal(caseId, "ci", null);
}
caseTracker.updateHeadSha(caseId, newHeadSha);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseServiceLifecycleTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All tests pass. Check that the existing `revisePr_nullsAllAnalysisFields` test still passes (it only checks the 6 analysis fields, not ci).

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): dual CI mode — external pre-seeds pending, dispatched allows binding

devtown.ci.mode config (default: external). External mode pre-seeds
ci={status:pending} in startReview and resets to pending in revisePr.
Dispatched mode omits ci (binding fires) and nulls on revise.
revisePr now also syncs tracker headSha.

Refs #86"
```

---

### Task 5: PrReviewCaseService — signalCiStatus with SHA guard and aggregation

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

- [ ] **Step 1: Write failing tests for signalCiStatus**

Create `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java`:

```java
package io.casehub.devtown.app;

import io.casehub.devtown.app.mcp.PrReviewCaseTracker;
import io.casehub.devtown.review.LifecycleResult;
import io.casehub.devtown.review.PrPayload;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class PrReviewCaseServiceCiStatusTest {

    private PrReviewCaseTracker tracker;
    private PrReviewCaseHub caseHub;
    private PrReviewCaseService service;
    private UUID caseId;

    @BeforeEach
    void setUp() {
        tracker = new PrReviewCaseTracker();
        caseId = UUID.randomUUID();

        var payload = new PrPayload("casehubio/devtown", 42, "sha123", "main", 100, "octocat", List.of());
        tracker.register(caseId, "default", payload);

        caseHub = mock(PrReviewCaseHub.class);
        when(caseHub.signal(any(), any(), any())).thenReturn(CompletableFuture.completedFuture(null));
        when(caseHub.query(eq(caseId), eq("pr.headSha"), eq(String.class)))
            .thenReturn(CompletableFuture.completedFuture("sha123"));

        service = new PrReviewCaseService();
        service.caseHub = caseHub;
        service.caseTracker = tracker;
        service.ciMode = "external";
    }

    @Test
    void signalCiStatus_noActiveCase_returnsNoActiveCase() {
        var result = service.signalCiStatus("casehubio/other", 99, "sha123", 1001, "success");
        assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
    }

    @Test
    void signalCiStatus_staleSha_returnsStaleEvent() {
        var result = service.signalCiStatus("casehubio/devtown", 42, "oldsha", 1001, "success");
        assertThat(result).isEqualTo(LifecycleResult.STALE_EVENT);
        verify(caseHub, never()).signal(eq(caseId), eq("ci.status"), any());
    }

    @Test
    void signalCiStatus_success_signalsPassingAndWritesSuite() {
        var result = service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "success");
        assertThat(result).isEqualTo(LifecycleResult.UPDATED);
        verify(caseHub).signal(eq(caseId), eq("ci.suites.1001"), any(Map.class));
        verify(caseHub).signal(caseId, "ci.status", "passing");
    }

    @Test
    void signalCiStatus_failure_signalsFailing() {
        service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "failure");
        verify(caseHub).signal(caseId, "ci.status", "failing");
    }

    @Test
    void signalCiStatus_secondSuiteSuccess_afterFirstSuccess_signalsPassing() {
        service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "success");
        service.signalCiStatus("casehubio/devtown", 42, "sha123", 1002, "success");
        verify(caseHub, times(2)).signal(caseId, "ci.status", "passing");
    }

    @Test
    void signalCiStatus_secondSuiteFailure_afterFirstSuccess_stickyFailing() {
        service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "success");
        service.signalCiStatus("casehubio/devtown", 42, "sha123", 1002, "failure");
        verify(caseHub).signal(caseId, "ci.status", "passing");
        verify(caseHub).signal(caseId, "ci.status", "failing");
    }

    @Test
    void signalCiStatus_successAfterFailure_stillFailing_stickyPolicy() {
        service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "failure");
        service.signalCiStatus("casehubio/devtown", 42, "sha123", 1002, "success");
        verify(caseHub, times(2)).signal(caseId, "ci.status", "failing");
        verify(caseHub, never()).signal(caseId, "ci.status", "passing");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseServiceCiStatusTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — `signalCiStatus` not implemented in PrReviewCaseService.

- [ ] **Step 3: Implement signalCiStatus in PrReviewCaseService**

Add a field to track suite results per case (after the existing injected fields):

```java
private final java.util.concurrent.ConcurrentHashMap<UUID, java.util.concurrent.ConcurrentHashMap<Long, String>> ciSuiteResults = new java.util.concurrent.ConcurrentHashMap<>();
```

Add the implementation after `closePr()`:

```java
@Override
public LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion) {
    var active = caseTracker.findActiveCaseByPr(repo, prNumber);
    if (active.isEmpty()) return LifecycleResult.NO_ACTIVE_CASE;

    UUID caseId = active.get().caseId();

    String currentSha = caseHub.query(caseId, "pr.headSha", String.class).toCompletableFuture().join();
    if (!headSha.equals(currentSha)) return LifecycleResult.STALE_EVENT;

    var suites = ciSuiteResults.computeIfAbsent(caseId, k -> new java.util.concurrent.ConcurrentHashMap<>());
    suites.put(suiteId, conclusion);

    caseHub.signal(caseId, "ci.suites." + suiteId, Map.of(
        "conclusion", conclusion,
        "completedAt", java.time.Instant.now().toString()
    ));

    boolean anyFailed = suites.values().stream().anyMatch(c -> !"success".equals(c));
    boolean allSuccess = suites.values().stream().allMatch("success"::equals);

    if (anyFailed) {
        caseHub.signal(caseId, "ci.status", "failing");
    } else if (allSuccess) {
        caseHub.signal(caseId, "ci.status", "passing");
    }

    return LifecycleResult.UPDATED;
}
```

Note: `ciSuiteResults` is cleared when `revisePr()` resets CI — add to the revisePr method (inside the ci invalidation block, before the signal):

```java
ciSuiteResults.remove(caseId);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseServiceCiStatusTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All 7 tests pass.

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): signalCiStatus — SHA guard, per-suite tracking, pessimistic aggregation

SHA guard reads from case context via caseHub.query, not tracker.
Per-suite results tracked in ci.suites.<id> for observability.
Aggregation: failures sticky, passing only when all suites succeed.

Refs #86"
```

---

### Task 6: PrReviewCaseService — signalCheckRun

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

- [ ] **Step 1: Write failing tests for signalCheckRun**

Add to `PrReviewCaseServiceCiStatusTest.java`:

```java
@Test
void signalCheckRun_noActiveCase_returnsNoActiveCase() {
    var result = service.signalCheckRun("casehubio/other", 99, "sha123", "lint", "success", java.time.Instant.now());
    assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
}

@Test
void signalCheckRun_staleSha_returnsStaleEvent() {
    var result = service.signalCheckRun("casehubio/devtown", 42, "oldsha", "lint", "success", java.time.Instant.now());
    assertThat(result).isEqualTo(LifecycleResult.STALE_EVENT);
}

@Test
void signalCheckRun_validSha_signalsCheckResult() {
    var completedAt = java.time.Instant.parse("2026-06-21T12:00:00Z");
    var result = service.signalCheckRun("casehubio/devtown", 42, "sha123", "lint", "success", completedAt);
    assertThat(result).isEqualTo(LifecycleResult.UPDATED);
    verify(caseHub).signal(eq(caseId), eq("ci.checks.lint"), eq(Map.of(
        "conclusion", "success",
        "completedAt", completedAt.toString()
    )));
}

@Test
void signalCheckRun_doesNotChangeCiStatus() {
    service.signalCheckRun("casehubio/devtown", 42, "sha123", "lint", "failure", java.time.Instant.now());
    verify(caseHub, never()).signal(eq(caseId), eq("ci.status"), any());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseServiceCiStatusTest#signalCheckRun*" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — `signalCheckRun` not implemented.

- [ ] **Step 3: Implement signalCheckRun in PrReviewCaseService**

Add after `signalCiStatus()`:

```java
@Override
public LifecycleResult signalCheckRun(String repo, int prNumber, String headSha, String checkName, String conclusion, java.time.Instant completedAt) {
    var active = caseTracker.findActiveCaseByPr(repo, prNumber);
    if (active.isEmpty()) return LifecycleResult.NO_ACTIVE_CASE;

    UUID caseId = active.get().caseId();

    String currentSha = caseHub.query(caseId, "pr.headSha", String.class).toCompletableFuture().join();
    if (!headSha.equals(currentSha)) return LifecycleResult.STALE_EVENT;

    caseHub.signal(caseId, "ci.checks." + checkName, Map.of(
        "conclusion", conclusion,
        "completedAt", completedAt.toString()
    ));

    return LifecycleResult.UPDATED;
}
```

- [ ] **Step 4: Run all CI status tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseServiceCiStatusTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All 11 tests pass.

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): signalCheckRun — per-job enrichment with SHA guard

Writes ci.checks.<name> with conclusion and completedAt.
Does not change ci.status — that is check_suite's responsibility.

Refs #86"
```

---

### Task 7: DTOs — GitHubCheckSuiteEvent and GitHubCheckRunEvent

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubCheckSuiteEvent.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubCheckRunEvent.java`
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubCheckSuiteEventTest.java`
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubCheckRunEventTest.java`

- [ ] **Step 1: Write failing tests for GitHubCheckSuiteEvent**

Create `github/src/test/java/io/casehub/devtown/github/GitHubCheckSuiteEventTest.java`:

```java
package io.casehub.devtown.github;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class GitHubCheckSuiteEventTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private static final String COMPLETED_EVENT = """
            {
              "action": "completed",
              "check_suite": {
                "id": 12345,
                "head_sha": "abc123",
                "status": "completed",
                "conclusion": "success",
                "pull_requests": [
                  { "number": 42, "head": { "sha": "abc123" } }
                ]
              },
              "repository": { "full_name": "casehubio/devtown" },
              "sender": { "login": "github-actions[bot]" }
            }
            """;

    @Test
    void parsesAction() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckSuiteEvent.class);
        assertThat(event.action()).isEqualTo("completed");
    }

    @Test
    void parsesSuiteId() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckSuiteEvent.class);
        assertThat(event.check_suite().id()).isEqualTo(12345);
    }

    @Test
    void parsesConclusion() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckSuiteEvent.class);
        assertThat(event.check_suite().conclusion()).isEqualTo("success");
    }

    @Test
    void parsesHeadSha() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckSuiteEvent.class);
        assertThat(event.check_suite().head_sha()).isEqualTo("abc123");
    }

    @Test
    void parsesPullRequests() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckSuiteEvent.class);
        assertThat(event.check_suite().pull_requests()).hasSize(1);
        assertThat(event.check_suite().pull_requests().get(0).number()).isEqualTo(42);
    }

    @Test
    void parsesRepository() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckSuiteEvent.class);
        assertThat(event.repository().full_name()).isEqualTo("casehubio/devtown");
    }

    @Test
    void ignoresUnknownFields() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckSuiteEvent.class);
        assertThat(event).isNotNull();
    }

    @Test
    void parsesEmptyPullRequests() throws Exception {
        var noPrs = COMPLETED_EVENT.replace(
            "\"pull_requests\": [\n                  { \"number\": 42, \"head\": { \"sha\": \"abc123\" } }\n                ]",
            "\"pull_requests\": []"
        );
        var event = MAPPER.readValue(noPrs, GitHubCheckSuiteEvent.class);
        assertThat(event.check_suite().pull_requests()).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dtest="io.casehub.devtown.github.GitHubCheckSuiteEventTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — class does not exist.

- [ ] **Step 3: Create GitHubCheckSuiteEvent DTO**

Create `github/src/main/java/io/casehub/devtown/github/GitHubCheckSuiteEvent.java`:

```java
package io.casehub.devtown.github;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubCheckSuiteEvent(
    String action,
    CheckSuite check_suite,
    Repository repository
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record CheckSuite(
        long id, String conclusion, String head_sha, String status,
        List<PullRequest> pull_requests
    ) {}
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record PullRequest(int number, Head head) {
        public record Head(String sha) {}
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Repository(String full_name) {}
}
```

- [ ] **Step 4: Run check_suite tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dtest="io.casehub.devtown.github.GitHubCheckSuiteEventTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All 8 tests pass.

- [ ] **Step 5: Write failing tests for GitHubCheckRunEvent**

Create `github/src/test/java/io/casehub/devtown/github/GitHubCheckRunEventTest.java`:

```java
package io.casehub.devtown.github;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class GitHubCheckRunEventTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private static final String COMPLETED_EVENT = """
            {
              "action": "completed",
              "check_run": {
                "name": "lint",
                "status": "completed",
                "conclusion": "success",
                "completed_at": "2026-06-21T12:00:00Z",
                "head_sha": "abc123",
                "pull_requests": [
                  { "number": 42, "head": { "sha": "abc123" } }
                ]
              },
              "repository": { "full_name": "casehubio/devtown" },
              "sender": { "login": "github-actions[bot]" }
            }
            """;

    @Test
    void parsesAction() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckRunEvent.class);
        assertThat(event.action()).isEqualTo("completed");
    }

    @Test
    void parsesCheckRunName() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckRunEvent.class);
        assertThat(event.check_run().name()).isEqualTo("lint");
    }

    @Test
    void parsesConclusion() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckRunEvent.class);
        assertThat(event.check_run().conclusion()).isEqualTo("success");
    }

    @Test
    void parsesCompletedAt() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckRunEvent.class);
        assertThat(event.check_run().completed_at()).isEqualTo("2026-06-21T12:00:00Z");
    }

    @Test
    void parsesHeadSha() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckRunEvent.class);
        assertThat(event.check_run().head_sha()).isEqualTo("abc123");
    }

    @Test
    void parsesPullRequests() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckRunEvent.class);
        assertThat(event.check_run().pull_requests()).hasSize(1);
        assertThat(event.check_run().pull_requests().get(0).number()).isEqualTo(42);
    }

    @Test
    void ignoresUnknownFields() throws Exception {
        var event = MAPPER.readValue(COMPLETED_EVENT, GitHubCheckRunEvent.class);
        assertThat(event).isNotNull();
    }
}
```

- [ ] **Step 6: Create GitHubCheckRunEvent DTO**

Create `github/src/main/java/io/casehub/devtown/github/GitHubCheckRunEvent.java`:

```java
package io.casehub.devtown.github;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubCheckRunEvent(
    String action,
    CheckRun check_run,
    Repository repository
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record CheckRun(
        String name, String status, String conclusion, String completed_at,
        String head_sha, List<PullRequest> pull_requests
    ) {}
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record PullRequest(int number, Head head) {
        public record Head(String sha) {}
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Repository(String full_name) {}
}
```

- [ ] **Step 7: Run all DTO tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dtest="io.casehub.devtown.github.GitHubCheckSuiteEventTest,io.casehub.devtown.github.GitHubCheckRunEventTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All 15 tests pass.

- [ ] **Step 8: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add github/src/main/java/io/casehub/devtown/github/GitHubCheckSuiteEvent.java github/src/main/java/io/casehub/devtown/github/GitHubCheckRunEvent.java github/src/test/java/io/casehub/devtown/github/GitHubCheckSuiteEventTest.java github/src/test/java/io/casehub/devtown/github/GitHubCheckRunEventTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(github): GitHubCheckSuiteEvent and GitHubCheckRunEvent DTOs

check_suite: id, conclusion, head_sha, pull_requests
check_run: name, conclusion, completed_at, head_sha, pull_requests

Refs #86"
```

---

### Task 8: GitHubWebhookResource — check_suite and check_run dispatch

**Files:**
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java`
- Modify: `github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java`

- [ ] **Step 1: Write failing tests for check_suite dispatch**

Extend `RecordingService` in `GitHubWebhookResourceTest.java` with the new port methods. Add fields:

```java
String lastCiRepo;
int lastCiPr;
String lastCiSha;
long lastCiSuiteId;
String lastCiConclusion;
String lastCheckRunName;
String lastCheckRunConclusion;
java.time.Instant lastCheckRunCompletedAt;
LifecycleResult signalCiResult = LifecycleResult.UPDATED;
LifecycleResult signalCheckRunResult = LifecycleResult.UPDATED;
```

Add implementations:

```java
@Override
public LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion) {
    lastCiRepo = repo;
    lastCiPr = prNumber;
    lastCiSha = headSha;
    lastCiSuiteId = suiteId;
    lastCiConclusion = conclusion;
    return signalCiResult;
}

@Override
public LifecycleResult signalCheckRun(String repo, int prNumber, String headSha, String checkName, String conclusion, java.time.Instant completedAt) {
    lastCheckRunName = checkName;
    lastCheckRunConclusion = conclusion;
    lastCheckRunCompletedAt = completedAt;
    return signalCheckRunResult;
}
```

Add test methods:

```java
@Test
void checkSuite_completed_callsSignalCiStatus() {
    String body = checkSuiteEvent("completed", "success", 12345);
    resource.receive(body, "check_suite", sign(body), "delivery-1");
    assertThat(service.lastCiRepo).isEqualTo("casehubio/devtown");
    assertThat(service.lastCiPr).isEqualTo(42);
    assertThat(service.lastCiSha).isEqualTo("abc123");
    assertThat(service.lastCiSuiteId).isEqualTo(12345);
    assertThat(service.lastCiConclusion).isEqualTo("success");
}

@Test
void checkSuite_requested_returnsIgnored() {
    String body = checkSuiteEvent("requested", null, 12345);
    var response = resource.receive(body, "check_suite", sign(body), "delivery-1");
    assertThat(responseBody(response)).containsEntry("status", "ignored");
    assertThat(service.lastCiRepo).isNull();
}

@Test
void checkSuite_emptyPullRequests_returnsIgnoredNoPullRequests() {
    String body = checkSuiteEventNoPrs("completed", "success", 12345);
    var response = resource.receive(body, "check_suite", sign(body), "delivery-1");
    assertThat(responseBody(response)).containsEntry("status", "ignored");
    assertThat(responseBody(response)).containsEntry("reason", "no-pull-requests");
}

@Test
void checkSuite_staleEvent_returnsIgnoredStaleSha() {
    service.signalCiResult = LifecycleResult.STALE_EVENT;
    String body = checkSuiteEvent("completed", "success", 12345);
    var response = resource.receive(body, "check_suite", sign(body), "delivery-1");
    assertThat(responseBody(response)).containsEntry("status", "ignored");
    assertThat(responseBody(response)).containsEntry("reason", "stale-sha");
}

@Test
void checkRun_completed_callsSignalCheckRun() {
    String body = checkRunEvent("completed", "lint", "success");
    resource.receive(body, "check_run", sign(body), "delivery-1");
    assertThat(service.lastCheckRunName).isEqualTo("lint");
    assertThat(service.lastCheckRunConclusion).isEqualTo("success");
}

@Test
void checkRun_created_returnsIgnored() {
    String body = checkRunEvent("created", "lint", null);
    var response = resource.receive(body, "check_run", sign(body), "delivery-1");
    assertThat(responseBody(response)).containsEntry("status", "ignored");
    assertThat(service.lastCheckRunName).isNull();
}
```

Add JSON factory helpers:

```java
private String checkSuiteEvent(String action, String conclusion, long suiteId) {
    String conclusionField = conclusion != null ? "\"conclusion\":\"%s\",".formatted(conclusion) : "";
    return "{\"action\":\"%s\",\"check_suite\":{\"id\":%d,\"head_sha\":\"abc123\",\"status\":\"completed\",%s\"pull_requests\":[{\"number\":42,\"head\":{\"sha\":\"abc123\"}}]},\"repository\":{\"full_name\":\"casehubio/devtown\"}}".formatted(action, suiteId, conclusionField);
}

private String checkSuiteEventNoPrs(String action, String conclusion, long suiteId) {
    return "{\"action\":\"%s\",\"check_suite\":{\"id\":%d,\"head_sha\":\"abc123\",\"status\":\"completed\",\"conclusion\":\"%s\",\"pull_requests\":[]},\"repository\":{\"full_name\":\"casehubio/devtown\"}}".formatted(action, suiteId, conclusion);
}

private String checkRunEvent(String action, String name, String conclusion) {
    String conclusionField = conclusion != null ? "\"conclusion\":\"%s\",".formatted(conclusion) : "";
    return "{\"action\":\"%s\",\"check_run\":{\"name\":\"%s\",\"status\":\"completed\",%s\"completed_at\":\"2026-06-21T12:00:00Z\",\"head_sha\":\"abc123\",\"pull_requests\":[{\"number\":42,\"head\":{\"sha\":\"abc123\"}}]},\"repository\":{\"full_name\":\"casehubio/devtown\"}}".formatted(action, name, conclusionField);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dtest="io.casehub.devtown.github.GitHubWebhookResourceTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: Compilation error — `RecordingService` doesn't implement the new port methods (until Step 1 code is added). Then FAIL — no `check_suite`/`check_run` handling in the resource.

- [ ] **Step 3: Implement check_suite and check_run dispatch in GitHubWebhookResource**

Add to the event type switch in `receive()` (after the `"pull_request"` case, line 49):

```java
case "check_suite" -> handleCheckSuite(body);
case "check_run" -> handleCheckRun(body);
```

Add the handler methods before `lifecycleAction()`:

```java
private Response handleCheckSuite(String body) throws Exception {
    var event = MAPPER.readValue(body, GitHubCheckSuiteEvent.class);

    if (!"completed".equals(event.action())) {
        return ok(Map.of("status", "ignored", "action", event.action()));
    }

    if (event.check_suite().pull_requests() == null || event.check_suite().pull_requests().isEmpty()) {
        return ok(Map.of("status", "ignored", "reason", "no-pull-requests"));
    }

    String repo = event.repository().full_name();
    String headSha = event.check_suite().head_sha();
    long suiteId = event.check_suite().id();
    String conclusion = event.check_suite().conclusion();

    LifecycleResult lastResult = LifecycleResult.UPDATED;
    for (var pr : event.check_suite().pull_requests()) {
        lastResult = service.signalCiStatus(repo, pr.number(), headSha, suiteId, conclusion);
    }
    return ok(Map.of("status", "accepted", "action", lifecycleAction("ci-status-updated", lastResult)));
}

private Response handleCheckRun(String body) throws Exception {
    var event = MAPPER.readValue(body, GitHubCheckRunEvent.class);

    if (!"completed".equals(event.action())) {
        return ok(Map.of("status", "ignored", "action", event.action()));
    }

    if (event.check_run().pull_requests() == null || event.check_run().pull_requests().isEmpty()) {
        return ok(Map.of("status", "ignored", "reason", "no-pull-requests"));
    }

    String repo = event.repository().full_name();
    String headSha = event.check_run().head_sha();
    String checkName = event.check_run().name();
    String conclusion = event.check_run().conclusion();
    java.time.Instant completedAt = event.check_run().completed_at() != null
        ? java.time.Instant.parse(event.check_run().completed_at()) : java.time.Instant.now();

    LifecycleResult lastResult = LifecycleResult.UPDATED;
    for (var pr : event.check_run().pull_requests()) {
        lastResult = service.signalCheckRun(repo, pr.number(), headSha, checkName, conclusion, completedAt);
    }
    return ok(Map.of("status", "accepted", "action", lifecycleAction("check-run-recorded", lastResult)));
}
```

Update `lifecycleAction()` to handle `STALE_EVENT`:

```java
private static String lifecycleAction(String normalAction, LifecycleResult result) {
    return switch (result) {
        case UPDATED -> normalAction;
        case NO_ACTIVE_CASE -> "no-active-case";
        case ALREADY_COMPLETED -> "already-completed";
        case ALREADY_ABANDONED -> "already-abandoned";
        case STALE_EVENT -> "stale-sha";
    };
}
```

And update the response mapping: when result is `STALE_EVENT`, return `"ignored"` status instead of `"accepted"`:

Replace the return in `handleCheckSuite` and `handleCheckRun`:
```java
if (lastResult == LifecycleResult.STALE_EVENT) {
    return ok(Map.of("status", "ignored", "reason", "stale-sha"));
}
return ok(Map.of("status", "accepted", "action", lifecycleAction("ci-status-updated", lastResult)));
```

(Same pattern for check_run handler, with `"check-run-recorded"` action.)

- [ ] **Step 4: Run all webhook tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All tests pass (existing pull_request tests + new check_suite/check_run tests).

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(github): check_suite and check_run webhook dispatch

check_suite:completed → signalCiStatus (CI goal resolution)
check_run:completed → signalCheckRun (per-job enrichment)
Other actions ignored. Empty pull_requests returns no-pull-requests.
STALE_EVENT mapped to ignored/stale-sha response.

Refs #86"
```

---

### Task 9: Full integration test + follow-up issue

**Files:**
- None new — run full test suite

- [ ] **Step 1: Run the full app test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: Same 5 pre-existing failures from #88. No new failures.

- [ ] **Step 2: Run the full github module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All tests pass.

- [ ] **Step 3: File follow-up issue for GitHub REST API aggregation**

```bash
gh issue create --repo casehubio/devtown --title "feat: GitHub REST API combined-status check for multi-suite CI aggregation" --body "## Context

devtown#86 implements webhook-only CI aggregation with pessimistic sticky-failure policy. This has two known limitations:

1. **Aggregation gap:** We don't know the total expected suite count without the REST API. A 'passing' status may be premature if additional workflows haven't reported.
2. **Atomicity race:** Two concurrent check_suite webhooks can both read zero existing suites and both signal 'passing.'

## What to do

Query the GitHub Checks API combined status for the HEAD SHA before signaling ci.status = 'passing'. This eliminates both the aggregation gap and the atomicity race.

Requires: GitHub REST client (REST client dependency, GitHub App or PAT auth).

Refs #86"
```

- [ ] **Step 4: Commit**

No code changes — just record the follow-up issue number in a final commit message or in the spec.
