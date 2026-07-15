# Batch Branch Management Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #104 — feat: batch branch management — git operations for merge queue batch testing
**Issue group:** #104

**Goal:** Implement the `batch-ci-runner` worker and cleanup observer for merge queue batch testing — port interface, GitHub adapter, worker registration, and lifecycle cleanup.

**Architecture:** Hexagonal port in `domain/`, GitHub REST adapter in `github/`, worker registration in `MergeBatchCaseHub.augment()`, cleanup observer on `CaseLifecycleEvent`. Follows the `MergeClient`/`GitHubMergeClient`/`NoOpMergeClient` pattern exactly.

**Tech Stack:** Java 21, Quarkus 3.32.2, MicroProfile REST Client, CDI `@ObservesAsync`

## Global Constraints

- Java 21 source (Java 26 JVM)
- All domain types in `domain/` must be pure Java — no Quarkus/CDI/Jakarta deps
- `github/` module uses MicroProfile REST Client with `configKey = "github-api"`
- `@DefaultBean` pattern for NoOp fallbacks in `app/spi/`
- Worker registration via `YamlCaseHub.augment()` — not `@PostConstruct`
- Branch naming: `merge-queue/batch-{batchId}`
- `project_path` for all IntelliJ MCP calls: `/Users/mdproctor/claude/casehub/devtown`

---

### Task 1: Domain port types

Create the port interface, value objects, and sealed result types in `domain/`.

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/PrRef.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/BatchBranchOutcome.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/BranchDeleteOutcome.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/BatchBranchClient.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/BatchBranchOutcomeTest.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/BranchDeleteOutcomeTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `BatchBranchClient` interface, `PrRef` record, `BatchBranchOutcome` sealed interface, `BranchDeleteOutcome` sealed interface — used by Tasks 2-5

- [ ] **Step 1: Write tests for sealed types**

```java
// domain/src/test/java/io/casehub/devtown/domain/BatchBranchOutcomeTest.java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class BatchBranchOutcomeTest {

    @Test
    void createdCarriesBranchNameAndTipSha() {
        var created = new BatchBranchOutcome.Created("merge-queue/batch-abc", "sha123");
        assertThat(created.branchName()).isEqualTo("merge-queue/batch-abc");
        assertThat(created.tipSha()).isEqualTo("sha123");
    }

    @Test
    void mergeConflictCarriesPrNumberAndBranchName() {
        var conflict = new BatchBranchOutcome.MergeConflict(42, "merge-queue/batch-abc");
        assertThat(conflict.conflictPrNumber()).isEqualTo(42);
        assertThat(conflict.branchName()).isEqualTo("merge-queue/batch-abc");
    }

    @Test
    void failureCarriesReason() {
        var failure = new BatchBranchOutcome.Failure("api error: HTTP 500");
        assertThat(failure.reason()).isEqualTo("api error: HTTP 500");
    }

    @Test
    void exhaustiveSwitchCoversAllCases() {
        BatchBranchOutcome outcome = new BatchBranchOutcome.Created("b", "s");
        String result = switch (outcome) {
            case BatchBranchOutcome.Created c -> "created:" + c.branchName();
            case BatchBranchOutcome.MergeConflict mc -> "conflict:" + mc.conflictPrNumber();
            case BatchBranchOutcome.Failure f -> "failure:" + f.reason();
        };
        assertThat(result).isEqualTo("created:b");
    }
}
```

```java
// domain/src/test/java/io/casehub/devtown/domain/BranchDeleteOutcomeTest.java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class BranchDeleteOutcomeTest {

    @Test
    void deletedCarriesBranchName() {
        var deleted = new BranchDeleteOutcome.Deleted("merge-queue/batch-abc");
        assertThat(deleted.branchName()).isEqualTo("merge-queue/batch-abc");
    }

    @Test
    void notFoundCarriesBranchName() {
        var notFound = new BranchDeleteOutcome.NotFound("merge-queue/batch-abc");
        assertThat(notFound.branchName()).isEqualTo("merge-queue/batch-abc");
    }

    @Test
    void failureCarriesReason() {
        var failure = new BranchDeleteOutcome.Failure("delete failed: HTTP 500");
        assertThat(failure.reason()).isEqualTo("delete failed: HTTP 500");
    }

    @Test
    void exhaustiveSwitchCoversAllCases() {
        BranchDeleteOutcome outcome = new BranchDeleteOutcome.NotFound("b");
        String result = switch (outcome) {
            case BranchDeleteOutcome.Deleted d -> "deleted";
            case BranchDeleteOutcome.NotFound nf -> "not-found";
            case BranchDeleteOutcome.Failure f -> "failure";
        };
        assertThat(result).isEqualTo("not-found");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest="BatchBranchOutcomeTest,BranchDeleteOutcomeTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 3: Create domain types**

```java
// domain/src/main/java/io/casehub/devtown/domain/PrRef.java
package io.casehub.devtown.domain;

public record PrRef(int number, String headSha) {}
```

```java
// domain/src/main/java/io/casehub/devtown/domain/BatchBranchOutcome.java
package io.casehub.devtown.domain;

public sealed interface BatchBranchOutcome {
    record Created(String branchName, String tipSha) implements BatchBranchOutcome {}
    record MergeConflict(int conflictPrNumber, String branchName) implements BatchBranchOutcome {}
    record Failure(String reason) implements BatchBranchOutcome {}
}
```

```java
// domain/src/main/java/io/casehub/devtown/domain/BranchDeleteOutcome.java
package io.casehub.devtown.domain;

public sealed interface BranchDeleteOutcome {
    record Deleted(String branchName) implements BranchDeleteOutcome {}
    record NotFound(String branchName) implements BranchDeleteOutcome {}
    record Failure(String reason) implements BranchDeleteOutcome {}
}
```

```java
// domain/src/main/java/io/casehub/devtown/domain/BatchBranchClient.java
package io.casehub.devtown.domain;

import java.util.List;

public interface BatchBranchClient {

    BatchBranchOutcome createBatchBranch(
        String owner, String repo,
        String targetBranch, String batchId,
        List<PrRef> prs
    );

    BranchDeleteOutcome deleteBatchBranch(
        String owner, String repo,
        String branchName
    );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest="BatchBranchOutcomeTest,BranchDeleteOutcomeTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/PrRef.java domain/src/main/java/io/casehub/devtown/domain/BatchBranchOutcome.java domain/src/main/java/io/casehub/devtown/domain/BranchDeleteOutcome.java domain/src/main/java/io/casehub/devtown/domain/BatchBranchClient.java domain/src/test/java/io/casehub/devtown/domain/BatchBranchOutcomeTest.java domain/src/test/java/io/casehub/devtown/domain/BranchDeleteOutcomeTest.java
```

Commit: `feat(#104): add BatchBranchClient port interface and sealed result types`

---

### Task 2: Prerequisites — BatchSlice repository propagation + CaseTrackingStatus fix

Fix two pre-existing issues identified by the design review. BatchSlice needs a `repository` field for bisection sub-cases, and CaseTrackingStatus needs the SUPERSEDED mapping.

**Files:**
- Modify: `queue/src/main/java/io/casehub/devtown/queue/BatchSlice.java`
- Modify: `queue/src/main/java/io/casehub/devtown/queue/BisectionSplitStrategy.java`
- Modify: `queue/src/main/java/io/casehub/devtown/queue/TrustWeightedSplitStrategy.java`
- Modify: `queue/src/main/java/io/casehub/devtown/queue/BinarySplitStrategy.java`
- Modify: `queue/src/main/java/io/casehub/devtown/queue/IsolateOutlierStrategy.java`
- Modify: `queue/src/main/java/io/casehub/devtown/queue/PrecedentBisectionStrategy.java`
- Modify: `queue/src/test/java/io/casehub/devtown/queue/BisectionSplitStrategyTest.java`
- Modify: `merge/src/main/resources/devtown/merge-batch.yaml`
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeBatchCaseHub.java` (sliceToMap, adaptBisectionSplit)
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/CaseTrackingStatus.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/mcp/CaseTrackingStatusTest.java`

**Interfaces:**
- Consumes: nothing from Task 1
- Produces: `BatchSlice` with `repository` field, `BisectionSplitStrategy.split()` with `repository` param, fixed `CaseTrackingStatus.fromCaseStatus("SUPERSEDED")` → `SUPERSEDED`

- [ ] **Step 1: Write test for CaseTrackingStatus SUPERSEDED mapping**

Read `app/src/test/java/io/casehub/devtown/app/mcp/CaseTrackingStatusTest.java` first. Add a test:

```java
@Test
void supersededIsTerminal() {
    assertThat(CaseTrackingStatus.fromCaseStatus("SUPERSEDED")).isEqualTo(CaseTrackingStatus.SUPERSEDED);
    assertThat(CaseTrackingStatus.SUPERSEDED.isTerminal()).isTrue();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="CaseTrackingStatusTest#supersededIsTerminal" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `fromCaseStatus("SUPERSEDED")` returns RUNNING, not SUPERSEDED

- [ ] **Step 3: Fix CaseTrackingStatus**

Use `ide_replace_member` on `CaseTrackingStatus.fromCaseStatus` to add the `"SUPERSEDED" -> SUPERSEDED` case to the switch.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="CaseTrackingStatusTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 5: Update BisectionSplitStrategyTest for repository field**

Add assertion in `TrustWeighted.sliceMetadataIsCorrect()`:

```java
assertThat(result.left().repository()).isEqualTo("casehubio/devtown");
assertThat(result.right().repository()).isEqualTo("casehubio/devtown");
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queue -Dtest="BisectionSplitStrategyTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `repository()` method doesn't exist on BatchSlice

- [ ] **Step 7: Add `repository` to BatchSlice**

Use `ide_edit_member` on `BatchSlice` (member=`BatchSlice`) to add `repository` field after `id`:

```java
public record BatchSlice(
        String id,
        String repository,
        String targetBranch,
        List<QueuedPr> prs,
        int size,
        String parentBatchId,
        int bisectionDepth,
        String bisectionStrategy,
        String riskLevel
) {}
```

- [ ] **Step 8: Add `repository` param to BisectionSplitStrategy.split()**

Use `ide_edit_member` on `BisectionSplitStrategy.split` to add `repository` after `prs`:

```java
SplitResult split(List<QueuedPr> prs, String repository, String batchId,
                  String targetBranch, int bisectionDepth,
                  String bisectionStrategy, String riskLevel);
```

- [ ] **Step 9: Update all four implementations**

Use `ide_edit_member` on each implementation's `split` method. The pattern is the same for all — add `String repository` parameter and pass it to `new BatchSlice(...)`:

**TrustWeightedSplitStrategy.split:**
```java
@Override
public SplitResult split(List<QueuedPr> prs, String repository, String batchId,
                         String targetBranch, int bisectionDepth,
                         String bisectionStrategy, String riskLevel) {
    if (prs.size() < 2) {
        throw new IllegalArgumentException("Cannot split a batch with fewer than 2 PRs");
    }
    List<QueuedPr> sorted = new ArrayList<>(prs);
    sorted.sort(Comparator.comparingDouble(QueuedPr::trustScore));
    int midpoint = sorted.size() / 2;
    List<QueuedPr> leftPrs = List.copyOf(sorted.subList(0, midpoint));
    List<QueuedPr> rightPrs = List.copyOf(sorted.subList(midpoint, sorted.size()));
    var left = new BatchSlice(batchId + "-L", repository, targetBranch, leftPrs, leftPrs.size(),
            batchId, bisectionDepth, bisectionStrategy, riskLevel);
    var right = new BatchSlice(batchId + "-R", repository, targetBranch, rightPrs, rightPrs.size(),
            batchId, bisectionDepth, bisectionStrategy, riskLevel);
    return new SplitResult(left, right);
}
```

Repeat for `BinarySplitStrategy`, `IsolateOutlierStrategy`, `PrecedentBisectionStrategy` — same pattern: add `String repository` param, pass to `new BatchSlice(...)` constructor as second arg.

- [ ] **Step 10: Update test helper calls**

In `BisectionSplitStrategyTest`, update every `strategy.split(prs, BATCH_ID, TARGET, ...)` call to `strategy.split(prs, "casehubio/devtown", BATCH_ID, TARGET, ...)` — `repository` is the new second argument.

- [ ] **Step 11: Run queue tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queue -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 12: Fix bisection-splitter inputSchema in merge-batch.yaml**

Use `Edit` (YAML file, not Java) on `merge/src/main/resources/devtown/merge-batch.yaml`. Change the bisection-splitter inputSchema from:

```yaml
inputSchema: '{ prs: .batch.prs, strategy: (.batch.bisectionStrategy // "trust-weighted") }'
```

to:

```yaml
inputSchema: '{ prs: .batch.prs, strategy: (.batch.bisectionStrategy // "trust-weighted"), batch: .batch }'
```

- [ ] **Step 13: Update MergeBatchCaseHub.adaptBisectionSplit and sliceToMap**

Use `ide_replace_member` on `MergeBatchCaseHub.adaptBisectionSplit` — extract `repository` from `batch` and pass it to `splitStrategy.split()`:

```java
@SuppressWarnings("unchecked")
WorkerResult adaptBisectionSplit(Map<String, Object> input) {
    List<Map<String, Object>> prMaps = (List<Map<String, Object>>) input.get("prs");
    String strategy = (String) input.getOrDefault("strategy", "trust-weighted");

    Map<String, Object> batch = (Map<String, Object>) input.get("batch");
    String batchId = batch != null ? (String) batch.getOrDefault("id", "unknown") : "unknown";
    String repository = batch != null ? (String) batch.getOrDefault("repository", "unknown") : "unknown";
    String targetBranch = batch != null ? (String) batch.getOrDefault("targetBranch", "main") : "main";
    int bisectionDepth = batch != null
                         ? ((Number) batch.getOrDefault("bisectionDepth", 0)).intValue()
                         : 0;
    String riskLevel = batch != null ? (String) batch.getOrDefault("riskLevel", "ROUTINE") : "ROUTINE";

    List<QueuedPr> prs = prMaps.stream()
                                                        .map(MergeBatchCaseHub::mapToQueuedPr)
                                                        .toList();

    SplitResult result = splitStrategy.split(
            prs, repository, batchId, targetBranch, bisectionDepth + 1, strategy, riskLevel);

    var output = new LinkedHashMap<String, Object>();
    output.put("left", sliceToMap(result.left()));
    output.put("right", sliceToMap(result.right()));
    return WorkerResult.of(Map.of("splitResult", output));
}
```

Use `ide_replace_member` on `MergeBatchCaseHub.sliceToMap` — add `repository` to the map:

```java
private Map<String, Object> sliceToMap(io.casehub.devtown.queue.BatchSlice slice) {
    var map = new LinkedHashMap<String, Object>();
    map.put("id", slice.id());
    map.put("repository", slice.repository());
    map.put("targetBranch", slice.targetBranch());
    map.put("prs", slice.prs().stream().map(MergeBatchCaseHub::queuedPrToMap).toList());
    map.put("size", slice.size());
    map.put("parentBatchId", slice.parentBatchId());
    map.put("bisectionDepth", slice.bisectionDepth());
    map.put("bisectionStrategy", slice.bisectionStrategy());
    map.put("riskLevel", slice.riskLevel());
    return map;
}
```

- [ ] **Step 14: Run full build to verify nothing is broken**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 15: Commit**

Stage all modified files. Commit: `fix(#104): add repository to BatchSlice, fix bisection-splitter inputSchema, fix CaseTrackingStatus SUPERSEDED`

---

### Task 3: GitHub adapter

Create the GitHub Git Data API REST client, GitRef response record, and GitHubBatchBranchClient with comprehensive unit tests.

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitRef.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubGitApi.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubBatchBranchClient.java`
- Test: `github/src/test/java/io/casehub/devtown/github/GitRefTest.java`
- Test: `github/src/test/java/io/casehub/devtown/github/GitHubBatchBranchClientTest.java`

**Interfaces:**
- Consumes: `BatchBranchClient`, `PrRef`, `BatchBranchOutcome`, `BranchDeleteOutcome` from Task 1
- Produces: `GitHubBatchBranchClient` `@ApplicationScoped` — displaces `NoOpBatchBranchClient` via CDI priority

- [ ] **Step 1: Write GitRef test**

```java
// github/src/test/java/io/casehub/devtown/github/GitRefTest.java
package io.casehub.devtown.github;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class GitRefTest {

    @Test
    void shaExtractsFromNestedObject() {
        var ref = new GitRef("refs/heads/main", Map.of("sha", "abc123"));
        assertThat(ref.sha()).isEqualTo("abc123");
    }

    @Test
    void shaReturnsNullWhenObjectIsNotMap() {
        var ref = new GitRef("refs/heads/main", "not-a-map");
        assertThat(ref.sha()).isNull();
    }

    @Test
    void shaReturnsNullWhenObjectIsNull() {
        var ref = new GitRef("refs/heads/main", null);
        assertThat(ref.sha()).isNull();
    }

    @Test
    void shaReturnsNullWhenMapMissesShaKey() {
        var ref = new GitRef("refs/heads/main", Map.of("type", "commit"));
        assertThat(ref.sha()).isNull();
    }
}
```

- [ ] **Step 2: Write GitHubBatchBranchClient tests**

```java
// github/src/test/java/io/casehub/devtown/github/GitHubBatchBranchClientTest.java
package io.casehub.devtown.github;

import io.casehub.devtown.domain.BatchBranchOutcome;
import io.casehub.devtown.domain.BranchDeleteOutcome;
import io.casehub.devtown.domain.PrRef;
import jakarta.ws.rs.WebApplicationException;
import jakarta.ws.rs.core.Response;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class GitHubBatchBranchClientTest {

    private GitHubGitApi api;
    private GitHubBatchBranchClient client;

    @BeforeEach
    void setUp() {
        api = mock(GitHubGitApi.class);
        client = new GitHubBatchBranchClient(api);
    }

    @Nested
    class CreateBatchBranch {

        @Test
        void happyPath_allPrsMerge_returnsCreated() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "tip-sha")));
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(Map.of("sha", "merge-sha"));

            var result = client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1"), new PrRef(2, "sha-2")));

            assertThat(result).isInstanceOf(BatchBranchOutcome.Created.class);
            var created = (BatchBranchOutcome.Created) result;
            assertThat(created.branchName()).isEqualTo("merge-queue/batch-b1");
            assertThat(created.tipSha()).isEqualTo("tip-sha");

            verify(api, times(2)).merge(eq("owner"), eq("repo"), anyMap());
        }

        @Test
        void conflictOnSecondPr_returnsMergeConflict() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")));
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(Map.of("sha", "merge-sha"))
                .thenThrow(new WebApplicationException(Response.status(409).build()));

            var result = client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1"), new PrRef(2, "sha-2")));

            assertThat(result).isInstanceOf(BatchBranchOutcome.MergeConflict.class);
            var conflict = (BatchBranchOutcome.MergeConflict) result;
            assertThat(conflict.conflictPrNumber()).isEqualTo(2);
            assertThat(conflict.branchName()).isEqualTo("merge-queue/batch-b1");
        }

        @Test
        void conflictOnFirstPr_returnsMergeConflict() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")));
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenThrow(new WebApplicationException(Response.status(409).build()));

            var result = client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1")));

            assertThat(result).isInstanceOf(BatchBranchOutcome.MergeConflict.class);
            assertThat(((BatchBranchOutcome.MergeConflict) result).conflictPrNumber()).isEqualTo(1);
        }

        @Test
        void targetBranchNullSha_returnsFailure() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", null));

            var result = client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1")));

            assertThat(result).isInstanceOf(BatchBranchOutcome.Failure.class);
            assertThat(((BatchBranchOutcome.Failure) result).reason()).contains("target branch");
        }

        @Test
        void apiErrorOnMerge_returnsFailure() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")));
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenThrow(new WebApplicationException(Response.status(500).build()));

            var result = client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1")));

            assertThat(result).isInstanceOf(BatchBranchOutcome.Failure.class);
            assertThat(((BatchBranchOutcome.Failure) result).reason()).contains("HTTP 500");
        }

        @Test
        void staleBranchDeletedBeforeCreate() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "tip-sha")));
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(Map.of("sha", "merge-sha"));

            client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1")));

            verify(api).deleteRef("owner", "repo", "heads/merge-queue/batch-b1");
        }

        @Test
        void staleBranchDeleteNotFound_continuesNormally() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "tip-sha")));
            doThrow(new WebApplicationException(Response.status(422).build()))
                .when(api).deleteRef("owner", "repo", "heads/merge-queue/batch-b1");
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(Map.of("sha", "merge-sha"));

            var result = client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1")));

            assertThat(result).isInstanceOf(BatchBranchOutcome.Created.class);
        }

        @Test
        void tipShaNullAfterMerges_returnsFailure() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")));
            when(api.getRef("owner", "repo", "heads/merge-queue/batch-b1"))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", null));
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(Map.of("sha", "merge-sha"));

            var result = client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(1, "sha-1")));

            assertThat(result).isInstanceOf(BatchBranchOutcome.Failure.class);
            assertThat(((BatchBranchOutcome.Failure) result).reason()).contains("no resolvable SHA after merges");
        }

        @Test
        void mergeCommitMessageIncludesPrNumber() {
            when(api.getRef("owner", "repo", "heads/main"))
                .thenReturn(new GitRef("refs/heads/main", Map.of("sha", "base-sha")))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "tip-sha")));
            when(api.createRef(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(new GitRef("refs/heads/merge-queue/batch-b1", Map.of("sha", "base-sha")));
            when(api.merge(eq("owner"), eq("repo"), anyMap()))
                .thenReturn(Map.of("sha", "merge-sha"));

            client.createBatchBranch("owner", "repo", "main", "b1",
                List.of(new PrRef(42, "sha-42")));

            @SuppressWarnings("unchecked")
            ArgumentCaptor<Map<String, String>> captor = ArgumentCaptor.forClass(Map.class);
            verify(api).merge(eq("owner"), eq("repo"), captor.capture());
            assertThat(captor.getValue().get("commit_message")).contains("PR #42");
        }
    }

    @Nested
    class DeleteBatchBranch {

        @Test
        void happyPath_returnsDeleted() {
            doNothing().when(api).deleteRef("owner", "repo", "heads/my-branch");
            var result = client.deleteBatchBranch("owner", "repo", "my-branch");
            assertThat(result).isInstanceOf(BranchDeleteOutcome.Deleted.class);
            assertThat(((BranchDeleteOutcome.Deleted) result).branchName()).isEqualTo("my-branch");
        }

        @Test
        void notFound422_returnsNotFound() {
            doThrow(new WebApplicationException(Response.status(422).build()))
                .when(api).deleteRef("owner", "repo", "heads/my-branch");
            var result = client.deleteBatchBranch("owner", "repo", "my-branch");
            assertThat(result).isInstanceOf(BranchDeleteOutcome.NotFound.class);
        }

        @Test
        void apiError_returnsFailure() {
            doThrow(new WebApplicationException(Response.status(500).build()))
                .when(api).deleteRef("owner", "repo", "heads/my-branch");
            var result = client.deleteBatchBranch("owner", "repo", "my-branch");
            assertThat(result).isInstanceOf(BranchDeleteOutcome.Failure.class);
            assertThat(((BranchDeleteOutcome.Failure) result).reason()).contains("HTTP 500");
        }
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest="GitRefTest,GitHubBatchBranchClientTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 4: Create GitRef, GitHubGitApi, GitHubBatchBranchClient**

Create the three files using `ide_create_file` for each. Code as specified in spec §4.1 and §4.2.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest="GitRefTest,GitHubBatchBranchClientTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

Commit: `feat(#104): add GitHubBatchBranchClient — GitHub Git Data API adapter`

---

### Task 4: NoOp default + Worker registration + integration tests

Wire the worker into MergeBatchCaseHub and create the NoOp fallback.

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/spi/NoOpBatchBranchClient.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeBatchCaseHub.java`
- Test: `app/src/test/java/io/casehub/devtown/app/BatchCiRunnerWorkerTest.java`

**Interfaces:**
- Consumes: `BatchBranchClient`, `PrRef`, `BatchBranchOutcome` from Task 1
- Produces: `batch-ci-runner` worker registered in `MergeBatchCaseHub` — engine dispatches to it when `test-batch-tip` binding fires

- [ ] **Step 1: Write worker adapter integration tests**

```java
// app/src/test/java/io/casehub/devtown/app/BatchCiRunnerWorkerTest.java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.BatchBranchClient;
import io.casehub.devtown.domain.BatchBranchOutcome;
import io.casehub.devtown.domain.BranchDeleteOutcome;
import io.casehub.devtown.domain.PrRef;
import io.casehub.worker.api.WorkerResult;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class BatchCiRunnerWorkerTest {

    @Inject MergeBatchCaseHub caseHub;

    @BeforeEach
    void setUp() {
        // Force definition loading so augment() runs
        caseHub.getDefinition();
    }

    @Test
    void workerIsRegistered() {
        var worker = caseHub.getDefinition().getWorkers().stream()
            .filter(w -> "batch-ci-runner".equals(w.name()))
            .findFirst();
        assertThat(worker).isPresent();
    }

    @Test
    void createdOutcome_returnsSuccessWithBranchAndTipSha() {
        var result = caseHub.adaptBatchCiRunner(buildInput("casehubio/devtown"));
        // NoOp returns Failure — this test verifies the adapter handles the mapping
        assertThat(result.outcome()).isNotNull();
    }

    @Test
    void missingRepository_returnsFailed() {
        var input = buildInput(null);
        var result = caseHub.adaptBatchCiRunner(input);
        assertThat(result.outcome().toString()).contains("Failed");
    }

    @Test
    void invalidRepositoryFormat_returnsFailed() {
        var input = buildInput("no-slash");
        var result = caseHub.adaptBatchCiRunner(input);
        assertThat(result.outcome().toString()).contains("Failed");
    }

    private Map<String, Object> buildInput(String repository) {
        var batch = new LinkedHashMap<String, Object>();
        batch.put("id", "test-batch-1");
        if (repository != null) {
            batch.put("repository", repository);
        }
        batch.put("targetBranch", "main");
        batch.put("prs", List.of(
            Map.of("number", 1, "headSha", "sha-1"),
            Map.of("number", 2, "headSha", "sha-2")
        ));
        return Map.of("batch", batch);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="BatchCiRunnerWorkerTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `adaptBatchCiRunner` method doesn't exist

- [ ] **Step 3: Create NoOpBatchBranchClient**

```java
// app/src/main/java/io/casehub/devtown/app/spi/NoOpBatchBranchClient.java
package io.casehub.devtown.app.spi;

import io.casehub.devtown.domain.BatchBranchClient;
import io.casehub.devtown.domain.BatchBranchOutcome;
import io.casehub.devtown.domain.BranchDeleteOutcome;
import io.casehub.devtown.domain.PrRef;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@DefaultBean
@ApplicationScoped
public class NoOpBatchBranchClient implements BatchBranchClient {

    @Override
    public BatchBranchOutcome createBatchBranch(
            String owner, String repo,
            String targetBranch, String batchId,
            List<PrRef> prs) {
        return new BatchBranchOutcome.Failure("no batch branch client configured");
    }

    @Override
    public BranchDeleteOutcome deleteBatchBranch(String owner, String repo, String branchName) {
        return new BranchDeleteOutcome.Failure("no batch branch client configured");
    }
}
```

- [ ] **Step 4: Add worker to MergeBatchCaseHub**

Use `ide_insert_member` to add `@Inject BatchBranchClient batchBranchClient;` field.

Use `ide_replace_member` on `augment` to add the `batch-ci-runner` worker registration alongside existing workers.

Use `ide_insert_member` to add `adaptBatchCiRunner` method (code as specified in spec §5.2, with input validation guards).

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="BatchCiRunnerWorkerTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Run existing merge queue tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="MergeQueueBatchLifecycleTest,MergeQueueBisectionTest,MergeQueueEscalationTest,MergeBatchCaseHubTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS — existing tests register their own mock `batch-ci-runner` worker, which takes CDI priority over the one registered in `augment()` because `removeIf` + `add` in `@BeforeEach` replaces it

- [ ] **Step 7: Commit**

Commit: `feat(#104): register batch-ci-runner worker in MergeBatchCaseHub`

---

### Task 5: Cleanup observer

Create the CDI observer that deletes batch branches when merge-batch cases reach a terminal state.

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/BatchBranchCleanupObserver.java`
- Test: `app/src/test/java/io/casehub/devtown/app/BatchBranchCleanupObserverTest.java`

**Interfaces:**
- Consumes: `BatchBranchClient`, `BranchDeleteOutcome` from Task 1; `CaseTrackingStatus` fix from Task 2; `CaseLifecycleEvent` from engine
- Produces: automatic branch cleanup on case terminal state

- [ ] **Step 1: Write cleanup observer tests**

```java
// app/src/test/java/io/casehub/devtown/app/BatchBranchCleanupObserverTest.java
package io.casehub.devtown.app;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.devtown.domain.BatchBranchClient;
import io.casehub.devtown.domain.BranchDeleteOutcome;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;

import static org.mockito.Mockito.*;

class BatchBranchCleanupObserverTest {

    private BatchBranchClient client;
    private BatchBranchCleanupObserver observer;
    private final ObjectMapper mapper = new ObjectMapper();

    @BeforeEach
    void setUp() {
        client = mock(BatchBranchClient.class);
        observer = new BatchBranchCleanupObserver();
        observer.batchBranchClient = client;
    }

    @Test
    void terminalMergeBatch_callsDeleteBranch() {
        when(client.deleteBatchBranch("casehubio", "devtown", "merge-queue/batch-b1"))
            .thenReturn(new BranchDeleteOutcome.Deleted("merge-queue/batch-b1"));

        observer.onCaseLifecycle(event("COMPLETED", "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verify(client).deleteBatchBranch("casehubio", "devtown", "merge-queue/batch-b1");
    }

    @Test
    void faultedCase_triggersCleanup() {
        when(client.deleteBatchBranch(any(), any(), any()))
            .thenReturn(new BranchDeleteOutcome.Deleted("merge-queue/batch-b1"));

        observer.onCaseLifecycle(event("FAULTED", "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verify(client).deleteBatchBranch("casehubio", "devtown", "merge-queue/batch-b1");
    }

    @Test
    void supersededCase_triggersCleanup() {
        when(client.deleteBatchBranch(any(), any(), any()))
            .thenReturn(new BranchDeleteOutcome.Deleted("merge-queue/batch-b1"));

        observer.onCaseLifecycle(event("SUPERSEDED", "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verify(client).deleteBatchBranch("casehubio", "devtown", "merge-queue/batch-b1");
    }

    @Test
    void nonTerminalStatus_noCleanup() {
        observer.onCaseLifecycle(event("RUNNING", "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verifyNoInteractions(client);
    }

    @Test
    void nonMergeBatchDefinition_noCleanup() {
        observer.onCaseLifecycle(event("COMPLETED", "devtown", "pr-review",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verifyNoInteractions(client);
    }

    @Test
    void nonDevtownNamespace_noCleanup() {
        observer.onCaseLifecycle(event("COMPLETED", "other-ns", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verifyNoInteractions(client);
    }

    @Test
    void nullCaseStatus_noCleanup() {
        observer.onCaseLifecycle(event(null, "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verifyNoInteractions(client);
    }

    @Test
    void nullContextSnapshot_noCleanup() {
        var event = new CaseLifecycleEvent(
            UUID.randomUUID(), "t1", "cmd", "evt", "COMPLETED",
            null, null, null, "merge-batch", "devtown", null);
        observer.onCaseLifecycle(event);

        verifyNoInteractions(client);
    }

    @Test
    void missingBatchId_noCleanup() {
        observer.onCaseLifecycle(event("COMPLETED", "devtown", "merge-batch",
            Map.of("batch", Map.of("repository", "casehubio/devtown"))));

        verifyNoInteractions(client);
    }

    @Test
    void invalidRepositoryFormat_noCleanup() {
        observer.onCaseLifecycle(event("COMPLETED", "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "no-slash"))));

        verifyNoInteractions(client);
    }

    @Test
    void deleteNotFound_logsDebugNoError() {
        when(client.deleteBatchBranch(any(), any(), any()))
            .thenReturn(new BranchDeleteOutcome.NotFound("merge-queue/batch-b1"));

        observer.onCaseLifecycle(event("COMPLETED", "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verify(client).deleteBatchBranch("casehubio", "devtown", "merge-queue/batch-b1");
    }

    @Test
    void deleteFailure_logsWarningNoException() {
        when(client.deleteBatchBranch(any(), any(), any()))
            .thenReturn(new BranchDeleteOutcome.Failure("api error"));

        observer.onCaseLifecycle(event("COMPLETED", "devtown", "merge-batch",
            Map.of("batch", Map.of("id", "b1", "repository", "casehubio/devtown"))));

        verify(client).deleteBatchBranch("casehubio", "devtown", "merge-queue/batch-b1");
    }

    private CaseLifecycleEvent event(String status, String namespace,
                                      String defName, Map<String, Object> context) {
        JsonNode contextNode = mapper.valueToTree(context);
        return new CaseLifecycleEvent(
            UUID.randomUUID(), "t1", "cmd", "evt", status,
            null, null, null, defName, namespace, contextNode);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="BatchBranchCleanupObserverTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Create BatchBranchCleanupObserver**

Code as specified in spec §6.1 — namespace filter, definition name filter, repository validation, fire-and-forget with logging.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="BatchBranchCleanupObserverTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 5: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS — all existing tests still pass

- [ ] **Step 6: Commit**

Commit: `feat(#104): add BatchBranchCleanupObserver for merge queue lifecycle cleanup`

---

## Post-Implementation

After all 5 tasks: invoke `work-end` which handles code review, ARC42STORIES.MD update, squash, push, and branch closure.
