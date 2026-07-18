# Cross-Repo Coordinated Merge E2E Test Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #160 — cross-repo coordinated merge end-to-end integration test
**Issue group:** #160

**Goal:** Prove that a multi-repo change set merges atomically and that failure in one repo triggers automatic rollback of the other, exercised through the full CDI/engine lifecycle in a `@QuarkusTest`.

**Architecture:** Single `@QuarkusTest` class with 5 test methods. Two `@Alternative` inner classes stub `MergeClient` and `RevertClient` with programmable queued outcomes and call recording. Helper methods decompose the coordination into reusable phases. Each test exercises a different coordination scenario through real engine case lifecycle.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, Awaitility, AssertJ

## Global Constraints

- Use `ide_*` tools for all Java file creation and editing
- Use `CrossTenantCaseInstanceRepository.findByUuid(UUID)` for case queries (no tenancy param)
- Use sequential `CaseHubRuntime.signal(UUID, String, Object)` for context updates (batch `signal(UUID, Map)` throws `UnsupportedOperationException`)
- Use `CaseHubRuntime.eventLog(UUID, Set<CaseHubEventType>)` for filtered EventLog queries
- All `RepoChangeEntry` objects use `linesChanged: 10` to stay below `humanApprovalThreshold`
- Pre-seed all capability keys with PENDING values per PP-20260521-134c38 immediately after review case creation
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test single class: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest`

---

### Task 1: Test infrastructure + Scenario 1 (happy path)

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java`

**Interfaces:**
- Consumes: `CoordinatedChangeService.start(CoordinatedChangeRequest)` → `CoordinatedChangeOutcome`
- Consumes: `CaseHubRuntime.signal(UUID, String, Object)` → `CompletionStage<Void>`
- Consumes: `CaseHubRuntime.cancelCase(UUID)` → void
- Consumes: `CaseHubRuntime.eventLog(UUID, Set<CaseHubEventType>)` → `CompletionStage<List<CaseEventLogRecord>>`
- Consumes: `CrossTenantCaseInstanceRepository.findByUuid(UUID)` → `CaseInstance`
- Consumes: `CoordinatedChangeTracker.findByReviewCaseId(UUID)` → `Entry`
- Consumes: `WorkItemQueries.scanAll()` → `List<WorkItem>`
- Consumes: `MergeClient.merge(String, String, int, String)` → `MergeOutcome`
- Consumes: `RevertClient.revert(String, String, String, String, String)` → `RevertOutcome`

- [ ] **Step 1: Create test class with @Alternative stubs**

Use `ide_create_file` to create the test class with the two inner `@Alternative` classes and all injections.

```java
package io.casehub.devtown.app;

import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.CaseStatus;
import io.casehub.api.model.event.CaseEventLogRecord;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.devtown.domain.CoordinatedChangeRequest;
import io.casehub.devtown.domain.MergeClient;
import io.casehub.devtown.domain.MergeOutcome;
import io.casehub.devtown.domain.RepoChangeEntry;
import io.casehub.devtown.domain.RevertClient;
import io.casehub.devtown.domain.RevertOutcome;
import io.casehub.devtown.review.CoordinatedChangeOutcome;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class CrossRepoCoordinatedMergeTest {

    @Inject CoordinatedChangeService coordinatedChangeService;
    @Inject CoordinatedChangeTracker tracker;
    @Inject PrReviewCaseHub prReviewCaseHub;
    @Inject CoordinatedChangeCaseHub coordinatedChangeCaseHub;
    @Inject CrossTenantCaseInstanceRepository caseInstanceRepository;
    @Inject CaseHubRuntime caseHubRuntime;
    @Inject WorkItemQueries workItemQueries;
    @Inject TestMergeClient testMergeClient;
    @Inject TestRevertClient testRevertClient;

    @BeforeEach
    void setUp() {
        testMergeClient.reset();
        testRevertClient.reset();
    }

    // ── @Alternative stubs ───────────────────────────────────────────

    @Alternative
    @Priority(1)
    @ApplicationScoped
    public static class TestMergeClient implements MergeClient {
        private final Queue<MergeOutcome> outcomes = new LinkedList<>();
        private final List<MergeCall> calls = Collections.synchronizedList(new ArrayList<>());

        public record MergeCall(String owner, String repo, int prNumber, String headSha) {}

        @Override
        public MergeOutcome merge(String owner, String repo, int prNumber, String headSha) {
            calls.add(new MergeCall(owner, repo, prNumber, headSha));
            MergeOutcome outcome = outcomes.poll();
            if (outcome == null) throw new IllegalStateException(
                "TestMergeClient: no outcome programmed for " + owner + "/" + repo + "#" + prNumber);
            return outcome;
        }

        public void enqueue(MergeOutcome... results) {
            Collections.addAll(outcomes, results);
        }

        public List<MergeCall> calls() { return List.copyOf(calls); }

        public void reset() {
            outcomes.clear();
            calls.clear();
        }
    }

    @Alternative
    @Priority(1)
    @ApplicationScoped
    public static class TestRevertClient implements RevertClient {
        private final Queue<RevertOutcome> outcomes = new LinkedList<>();
        private final List<RevertCall> calls = Collections.synchronizedList(new ArrayList<>());

        public record RevertCall(String owner, String repo, String targetBranch,
                                 String mergeSha, String commitMessage) {}

        @Override
        public RevertOutcome revert(String owner, String repo, String targetBranch,
                                    String mergeSha, String commitMessage) {
            calls.add(new RevertCall(owner, repo, targetBranch, mergeSha, commitMessage));
            RevertOutcome outcome = outcomes.poll();
            if (outcome == null) throw new IllegalStateException(
                "TestRevertClient: no outcome programmed for " + owner + "/" + repo);
            return outcome;
        }

        public void enqueue(RevertOutcome... results) {
            Collections.addAll(outcomes, results);
        }

        public List<RevertCall> calls() { return List.copyOf(calls); }

        public void reset() {
            outcomes.clear();
            calls.clear();
        }
    }

    // ── Helpers ──────────────────────────────────────────────────────

    private CoordinatedChangeRequest buildRequest(RepoChangeEntry... repos) {
        return new CoordinatedChangeRequest(List.of(repos));
    }

    private void preSeedCapabilityKeys(UUID reviewCaseId) {
        caseHubRuntime.signal(reviewCaseId, "codeAnalysis", java.util.Map.of("outcome", "PENDING"))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "styleCheck", java.util.Map.of("outcome", "PENDING"))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "testCoverage", java.util.Map.of("outcome", "PENDING"))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "performanceAnalysis", java.util.Map.of("outcome", "PENDING"))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "ci", java.util.Map.of("status", "PENDING"))
            .toCompletableFuture().join();
    }

    private void driveReviewToCompletion(UUID reviewCaseId) {
        caseHubRuntime.signal(reviewCaseId, "codeAnalysis",
            java.util.Map.of("complete", true, "securitySensitive", false, "architectureCrossing", false))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "styleCheck", java.util.Map.of("outcome", "APPROVED"))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "testCoverage", java.util.Map.of("outcome", "APPROVED"))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "performanceAnalysis", java.util.Map.of("outcome", "APPROVED"))
            .toCompletableFuture().join();
        caseHubRuntime.signal(reviewCaseId, "ci", java.util.Map.of("status", "passing"))
            .toCompletableFuture().join();
    }

    private void driveReviewToFault(UUID reviewCaseId) {
        caseHubRuntime.cancelCase(reviewCaseId);
    }

    private void awaitCaseStatus(UUID caseId, CaseStatus expected) {
        await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() -> {
            var instance = caseInstanceRepository.findByUuid(caseId);
            assertThat(instance).as("Case %s should exist", caseId).isNotNull();
            assertThat(instance.getState()).as("Case %s status", caseId).isEqualTo(expected);
        });
    }

    private void awaitCaseTerminal(UUID caseId) {
        await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() -> {
            var instance = caseInstanceRepository.findByUuid(caseId);
            assertThat(instance).isNotNull();
            assertThat(instance.getState().isTerminal())
                .as("Case %s should be terminal but is %s", caseId, instance.getState())
                .isTrue();
        });
    }

    private Object contextValue(UUID caseId, String path) {
        var instance = caseInstanceRepository.findByUuid(caseId);
        return instance.getCaseContext().getPath(path);
    }
}
```

- [ ] **Step 2: Verify test class compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl app`
Expected: BUILD SUCCESS

Use `ide_diagnostics` on the file to check for compile errors. If imports are wrong (e.g., `CaseStatus` package), fix them.

- [ ] **Step 3: Write Scenario 1 — happy path test method**

Use `ide_insert_member` to add the test method after the `contextValue` helper:

```java
@Test
void happyPath_allReviewsComplete_mergeSucceeds_parentCompleted() {
    // ── Setup ────────────────────────────────────────────────────
    testMergeClient.enqueue(
        new MergeOutcome.Success("sha-engine"),
        new MergeOutcome.Success("sha-platform"));

    var request = buildRequest(
        new RepoChangeEntry("casehubio", "engine", 42, "abc123", "main", "alice", List.of("src/Main.java"), 10),
        new RepoChangeEntry("casehubio", "platform", 99, "def456", "main", "bob", List.of("src/Config.java"), 10));

    // ── Phase 1: Initiation ──────────────────────────────────────
    CoordinatedChangeOutcome outcome = coordinatedChangeService.start(request);
    UUID parentCaseId = outcome.parentCaseId();
    UUID reviewA = outcome.reviewCaseIds().get("casehubio/engine");
    UUID reviewB = outcome.reviewCaseIds().get("casehubio/platform");

    assertThat(parentCaseId).isNotNull();
    assertThat(reviewA).isNotNull();
    assertThat(reviewB).isNotNull();
    assertThat(tracker.findByReviewCaseId(reviewA)).isNotNull();
    assertThat(tracker.findByReviewCaseId(reviewA).parentCaseId()).isEqualTo(parentCaseId);
    assertThat(tracker.findByReviewCaseId(reviewB).parentCaseId()).isEqualTo(parentCaseId);

    var parentInstance = caseInstanceRepository.findByUuid(parentCaseId);
    assertThat(parentInstance).isNotNull();
    assertThat(parentInstance.getCaseContext().getPath("reviewCases")).isNotNull();

    // ── Phase 2: Pre-seed + drive review cases to completion ─────
    preSeedCapabilityKeys(reviewA);
    preSeedCapabilityKeys(reviewB);
    driveReviewToCompletion(reviewA);
    driveReviewToCompletion(reviewB);

    awaitCaseStatus(reviewA, CaseStatus.COMPLETED);
    awaitCaseStatus(reviewB, CaseStatus.COMPLETED);

    // ── Phase 3: Coordination bridge ─────────────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() -> {
        assertThat(contextValue(parentCaseId, "completedReviews.casehubio/engine.status"))
            .isEqualTo("completed");
        assertThat(contextValue(parentCaseId, "completedReviews.casehubio/platform.status"))
            .isEqualTo("completed");
        assertThat(contextValue(parentCaseId, "allReviewsCompleted"))
            .isEqualTo(true);
    });

    // ── Phase 4: Worker execution ────────────────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "mergeResults")).isNotNull());

    assertThat(testMergeClient.calls()).hasSize(2);
    assertThat(testMergeClient.calls().get(0).repo()).isEqualTo("engine");
    assertThat(testMergeClient.calls().get(1).repo()).isEqualTo("platform");

    // ── Phase 5: Terminal state ──────────────────────────────────
    awaitCaseStatus(parentCaseId, CaseStatus.COMPLETED);

    var reviewAInstance = caseInstanceRepository.findByUuid(reviewA);
    assertThat(reviewAInstance.getCaseContext().getPath("coordinatedChange")).isEqualTo(true);

    // ── Phase 6: EventLog ────────────────────────────────────────
    var signals = caseHubRuntime.eventLog(parentCaseId,
        Set.of(CaseHubEventType.SIGNAL_RECEIVED)).toCompletableFuture().join();
    assertThat(signals).isNotEmpty();

    var goals = caseHubRuntime.eventLog(parentCaseId,
        Set.of(CaseHubEventType.GOAL_REACHED)).toCompletableFuture().join();
    assertThat(goals).isNotEmpty();
    assertThat(goals.stream().anyMatch(g ->
        g.payload() != null && g.payload().toString().contains("all-repos-merged"))).isTrue();

    var workerExecs = caseHubRuntime.eventLog(parentCaseId,
        Set.of(CaseHubEventType.WORKER_EXECUTION_COMPLETED)).toCompletableFuture().join();
    assertThat(workerExecs).isNotEmpty();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest#happyPath_allReviewsComplete_mergeSucceeds_parentCompleted`
Expected: PASS

If it fails, read the failure output. Common issues:
- Import errors: fix with `ide_diagnostics` + `ide_optimize_imports`
- `CaseStatus` enum values may differ (e.g., `COMPLETED` vs `COMPLETE`): check with `ide_find_class` on `CaseStatus`
- Timing: increase `atMost` if async delivery is slow in test
- Pre-seeding race: if capability bindings fire before pre-seeding, add a short `Thread.sleep(200)` between `start()` and `preSeedCapabilityKeys()` (last resort)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "$(cat <<'EOF'
test(#160): cross-repo coordinated merge E2E — happy path

@QuarkusTest exercising full coordination lifecycle: start coordinated
change, drive review cases through engine to COMPLETED, observer bridges
completion to parent, merge worker executes, parent reaches COMPLETED.
Observable @Alternative stubs for MergeClient/RevertClient.

Refs #160

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Scenario 2 — review faults, parent terminates

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java`

**Interfaces:**
- Consumes: all helpers from Task 1
- Consumes: `driveReviewToFault(UUID)` (calls `cancelCase`)

- [ ] **Step 1: Write Scenario 2 test method**

Use `ide_insert_member` to add after the scenario 1 test:

```java
@Test
void faultPath_reviewFaults_parentTerminates_remainingStaysCompleted() {
    // No merge outcomes — merge should never be reached
    var request = buildRequest(
        new RepoChangeEntry("casehubio", "engine", 42, "abc123", "main", "alice", List.of("src/Main.java"), 10),
        new RepoChangeEntry("casehubio", "platform", 99, "def456", "main", "bob", List.of("src/Config.java"), 10));

    // ── Phase 1: Initiation ──────────────────────────────────────
    CoordinatedChangeOutcome outcome = coordinatedChangeService.start(request);
    UUID parentCaseId = outcome.parentCaseId();
    UUID reviewA = outcome.reviewCaseIds().get("casehubio/engine");
    UUID reviewB = outcome.reviewCaseIds().get("casehubio/platform");

    // ── Phase 2: Divergent lifecycle ─────────────────────────────
    preSeedCapabilityKeys(reviewA);
    preSeedCapabilityKeys(reviewB);
    driveReviewToCompletion(reviewA);
    awaitCaseStatus(reviewA, CaseStatus.COMPLETED);

    driveReviewToFault(reviewB);

    // ── Phase 3: Fault propagation ───────────────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() -> {
        assertThat(contextValue(parentCaseId, "reviewFaulted.repo"))
            .isEqualTo("casehubio/platform");
        assertThat(contextValue(parentCaseId, "reviewFaulted.reason"))
            .isEqualTo("CANCELLED");
    });

    // ── Phase 4: Terminal + cancel propagation ───────────────────
    awaitCaseTerminal(parentCaseId);

    // reviewA was already COMPLETED — cancelCase() throws on terminal cases,
    // caught by observer's try/catch. reviewA stays COMPLETED.
    var reviewAInstance = caseInstanceRepository.findByUuid(reviewA);
    assertThat(reviewAInstance.getState()).isEqualTo(CaseStatus.COMPLETED);

    assertThat(testMergeClient.calls()).isEmpty();

    // ── Phase 5: EventLog ────────────────────────────────────────
    var signals = caseHubRuntime.eventLog(parentCaseId,
        Set.of(CaseHubEventType.SIGNAL_RECEIVED)).toCompletableFuture().join();
    assertThat(signals.stream().anyMatch(s ->
        s.payload() != null && s.payload().toString().contains("reviewFaulted"))).isTrue();

    var goals = caseHubRuntime.eventLog(parentCaseId,
        Set.of(CaseHubEventType.GOAL_REACHED)).toCompletableFuture().join();
    assertThat(goals.stream().anyMatch(g ->
        g.payload() != null && g.payload().toString().contains("review-faulted"))).isTrue();
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest#faultPath_reviewFaults_parentTerminates_remainingStaysCompleted`
Expected: PASS

- [ ] **Step 3: Run all scenarios to confirm no cross-test interference**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest`
Expected: 2 tests PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "$(cat <<'EOF'
test(#160): cross-repo E2E — fault path with cancel propagation

Review fault triggers reviewFaulted signal to parent, parent terminates,
already-completed review stays COMPLETED (cancelCase throws on terminal).
Merge worker never called.

Refs #160

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Scenario 3 — rollback failure with human escalation

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java`

**Interfaces:**
- Consumes: all helpers from Task 1
- Consumes: `WorkItemQueries.scanAll()` → `List<WorkItem>` (WorkItem has `.title`, `.callerRef`, `.candidateGroups`)

- [ ] **Step 1: Write Scenario 3 test method**

Use `ide_insert_member` to add after scenario 2:

```java
@Test
void rollbackFailure_mergeConflict_humanEscalation() {
    testMergeClient.enqueue(
        new MergeOutcome.Success("sha-engine"),
        new MergeOutcome.Failure("merge conflict"));
    testRevertClient.enqueue(
        new RevertOutcome.MergeConflict(101, "branch protection prevents merge"));

    var request = buildRequest(
        new RepoChangeEntry("casehubio", "engine", 42, "abc123", "main", "alice", List.of("src/Main.java"), 10),
        new RepoChangeEntry("casehubio", "platform", 99, "def456", "main", "bob", List.of("src/Config.java"), 10),
        new RepoChangeEntry("casehubio", "work", 7, "ghi789", "main", "carol", List.of("src/Worker.java"), 10));

    // ── Phase 1: Initiation ──────────────────────────────────────
    CoordinatedChangeOutcome outcome = coordinatedChangeService.start(request);
    UUID parentCaseId = outcome.parentCaseId();
    UUID reviewA = outcome.reviewCaseIds().get("casehubio/engine");
    UUID reviewB = outcome.reviewCaseIds().get("casehubio/platform");
    UUID reviewC = outcome.reviewCaseIds().get("casehubio/work");

    // ── Phase 2-3: All reviews complete ──────────────────────────
    preSeedCapabilityKeys(reviewA);
    preSeedCapabilityKeys(reviewB);
    preSeedCapabilityKeys(reviewC);
    driveReviewToCompletion(reviewA);
    driveReviewToCompletion(reviewB);
    driveReviewToCompletion(reviewC);

    awaitCaseStatus(reviewA, CaseStatus.COMPLETED);
    awaitCaseStatus(reviewB, CaseStatus.COMPLETED);
    awaitCaseStatus(reviewC, CaseStatus.COMPLETED);

    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "allReviewsCompleted")).isEqualTo(true));

    // ── Phase 4a: Merge — stop on failure ────────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "mergeResults")).isNotNull());

    assertThat(testMergeClient.calls()).hasSize(2);
    assertThat(testMergeClient.calls().get(0).repo()).isEqualTo("engine");
    assertThat(testMergeClient.calls().get(1).repo()).isEqualTo("platform");

    // ── Phase 4b: Rollback ───────────────────────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "rollbackResults")).isNotNull());

    assertThat(testRevertClient.calls()).hasSize(1);
    assertThat(testRevertClient.calls().get(0).repo()).isEqualTo("engine");

    @SuppressWarnings("unchecked")
    var rollbackResults = (List<java.util.Map<String, Object>>)
        contextValue(parentCaseId, "rollbackResults");
    assertThat(rollbackResults).hasSize(1);
    assertThat(rollbackResults.get(0).get("status")).isEqualTo("conflict");

    // ── Phase 4c: Human escalation ───────────────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() -> {
        var items = workItemQueries.scanAll().stream()
            .filter(wi -> wi.callerRef != null && wi.callerRef.contains(parentCaseId.toString()))
            .toList();
        assertThat(items).as("human escalation WorkItem").isNotEmpty();
        assertThat(items.get(0).title)
            .isEqualTo("Coordinated rollback failed — manual revert required");
    });
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest#rollbackFailure_mergeConflict_humanEscalation`
Expected: PASS

- [ ] **Step 3: Run all scenarios**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest`
Expected: 3 tests PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "$(cat <<'EOF'
test(#160): cross-repo E2E — rollback failure + human escalation

3-repo change: merge stops on second repo failure, rollback worker
encounters merge conflict, rollback-human-escalation binding fires,
WorkItem created for human-oversight:general.

Refs #160

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Scenario 4 — out-of-order completion

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java`

**Interfaces:**
- Consumes: all helpers from Task 1

- [ ] **Step 1: Write Scenario 4 test method**

Use `ide_insert_member` to add after scenario 3:

```java
@Test
void outOfOrder_reverseCompletion_sameResult() {
    testMergeClient.enqueue(
        new MergeOutcome.Success("sha-engine"),
        new MergeOutcome.Success("sha-platform"));

    var request = buildRequest(
        new RepoChangeEntry("casehubio", "engine", 42, "abc123", "main", "alice", List.of("src/Main.java"), 10),
        new RepoChangeEntry("casehubio", "platform", 99, "def456", "main", "bob", List.of("src/Config.java"), 10));

    CoordinatedChangeOutcome outcome = coordinatedChangeService.start(request);
    UUID parentCaseId = outcome.parentCaseId();
    UUID reviewA = outcome.reviewCaseIds().get("casehubio/engine");
    UUID reviewB = outcome.reviewCaseIds().get("casehubio/platform");

    preSeedCapabilityKeys(reviewA);
    preSeedCapabilityKeys(reviewB);

    // ── Platform completes FIRST ─────────────────────────────────
    driveReviewToCompletion(reviewB);
    awaitCaseStatus(reviewB, CaseStatus.COMPLETED);

    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "completedReviews.casehubio/platform.status"))
            .isEqualTo("completed"));

    // allReviewsCompleted should NOT be true yet
    assertThat(contextValue(parentCaseId, "allReviewsCompleted")).isNull();

    // ── Engine completes SECOND ──────────────────────────────────
    driveReviewToCompletion(reviewA);
    awaitCaseStatus(reviewA, CaseStatus.COMPLETED);

    // ── Same result as happy path ────────────────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "allReviewsCompleted")).isEqualTo(true));

    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "mergeResults")).isNotNull());

    awaitCaseStatus(parentCaseId, CaseStatus.COMPLETED);
    assertThat(testMergeClient.calls()).hasSize(2);
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest#outOfOrder_reverseCompletion_sameResult`
Expected: PASS

- [ ] **Step 3: Run all scenarios**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest`
Expected: 4 tests PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "$(cat <<'EOF'
test(#160): cross-repo E2E — out-of-order review completion

Platform completes before engine — observer correctly counts
completions and only signals allReviewsCompleted when both are done.
Same terminal state as happy path.

Refs #160

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Scenario 5 — idempotent binding guard

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java`

**Interfaces:**
- Consumes: all helpers from Task 1

- [ ] **Step 1: Write Scenario 5 test method**

Use `ide_insert_member` to add after scenario 4:

```java
@Test
void idempotentGuard_extraContextChange_doesNotReFire() {
    testMergeClient.enqueue(
        new MergeOutcome.Success("sha-engine"),
        new MergeOutcome.Failure("merge conflict"));
    testRevertClient.enqueue(
        new RevertOutcome.Success(101, "revert-sha-engine"));

    var request = buildRequest(
        new RepoChangeEntry("casehubio", "engine", 42, "abc123", "main", "alice", List.of("src/Main.java"), 10),
        new RepoChangeEntry("casehubio", "platform", 99, "def456", "main", "bob", List.of("src/Config.java"), 10));

    CoordinatedChangeOutcome outcome = coordinatedChangeService.start(request);
    UUID parentCaseId = outcome.parentCaseId();
    UUID reviewA = outcome.reviewCaseIds().get("casehubio/engine");
    UUID reviewB = outcome.reviewCaseIds().get("casehubio/platform");

    preSeedCapabilityKeys(reviewA);
    preSeedCapabilityKeys(reviewB);
    driveReviewToCompletion(reviewA);
    driveReviewToCompletion(reviewB);

    // ── Wait for merge + rollback to complete ────────────────────
    await().atMost(10, SECONDS).pollInterval(100, TimeUnit.MILLISECONDS).untilAsserted(() ->
        assertThat(contextValue(parentCaseId, "rollbackResults")).isNotNull());

    int revertCallsBefore = testRevertClient.calls().size();
    assertThat(revertCallsBefore).isEqualTo(1);

    // ── Provoke re-evaluation with no-op signal ──────────────────
    caseHubRuntime.signal(parentCaseId, "probe", "idempotency-check")
        .toCompletableFuture().join();

    // Assert rollback binding does NOT re-fire — condition guard
    // (.rollbackResults == null) is now false
    await().during(2, SECONDS).atMost(3, SECONDS).pollInterval(200, TimeUnit.MILLISECONDS)
        .untilAsserted(() ->
            assertThat(testRevertClient.calls()).hasSize(revertCallsBefore));

    // ── Parent reaches terminal state ────────────────────────────
    awaitCaseTerminal(parentCaseId);
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest#idempotentGuard_extraContextChange_doesNotReFire`
Expected: PASS

- [ ] **Step 3: Run all 5 scenarios**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CrossRepoCoordinatedMergeTest`
Expected: 5 tests PASS

- [ ] **Step 4: Run full project build to confirm no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "$(cat <<'EOF'
test(#160): cross-repo E2E — idempotent rollback guard

After rollback completes, a no-op context signal re-evaluates bindings
but rollback-on-merge-failure does not re-fire because
.rollbackResults == null is now false. Confirms YAML condition guards
prevent duplicate worker dispatch.

Closes #160

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```
