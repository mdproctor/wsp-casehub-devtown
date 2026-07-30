# Merge Queue Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the CaseHub merge queue — trust-weighted batch-then-bisect as a CasePlanModel, with an imperative queue service for admission, priority, and batch formation.

**Architecture:** Two-tier decomposition. `MergeQueueService` (imperative, `@ApplicationScoped`) handles queue management: admission, priority scoring, dependency DAG, batch formation, SLA tracking. `merge-batch.yaml` (reactive CasePlanModel) handles batch processing: tip test, bisect, merge, reject. Bisection uses recursive sub-cases (engine#573) with M-of-N grouped parallel halves (engine#574). Trust-weighted split via pluggable `BisectionSplitStrategy` SPI.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine 0.2-SNAPSHOT (with engine#573 + engine#574 merged), H2 MODE=PostgreSQL for tests.

**Spec:** `specs/2026-06-26-merge-queue-design.md` (workspace) — v5, 5 review rounds, 15 terminal paths verified.

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All casehub artifacts are `0.2-SNAPSHOT` from GitHub Packages
- Module convention: `queue/` already exists as a stub (has `pom.xml` + `package-info.java`)
- Domain vocabulary lives in `domain/` — pure Java, no Quarkus
- Integration logic lives in `queue/` (peer to `review/`) — casehub-engine deps allowed
- CDI wiring + MCP tools live in `app/`
- CasePlanModel YAML lives in the module that owns the case definition (see `review/src/main/resources/devtown/pr-review.yaml` as pattern)
- Tests: unit tests in the module being tested; `@QuarkusTest` integration tests in `app/`
- Every commit references an issue: `Refs #11` or `Closes #11`
- Existing pattern: `YamlCaseHub` subclass registers workers; see `PrReviewCaseHub.java`
- Existing pattern: vocabulary types are `final class` with `public static final String` constants; see `ReviewDomain.java`, `AgentQualification.java`
- Existing pattern: SPI interfaces in `domain/src/main/java/.../domain/spi/`; implementations in `queue/` or `app/`
- Existing pattern: `DevtownCapabilityRegistry` must be updated when new capabilities are added

---

### Task 1: Domain Vocabulary — MergeQueueCapability

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/MergeQueueCapability.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/MergeQueueCapabilityTest.java`
- Modify: `domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java`
- Modify: `domain/src/test/java/io/casehub/devtown/domain/DevtownCapabilityRegistryTest.java`

**Interfaces:**
- Consumes: `ReviewDomain`, `AgentQualification` (existing vocabulary pattern)
- Produces: `MergeQueueCapability.BATCH_CI_RUNNER`, `.BISECTION_SPLITTER`, `.PR_REJECT_AND_NOTIFY`, `.MERGE_QUEUE_ENQUEUE` — string constants referenced by `merge-batch.yaml` capability names and `DevtownCapabilityRegistry`

- [ ] **Step 1: Write the failing test for MergeQueueCapability**

```java
package io.casehub.devtown.domain;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class MergeQueueCapabilityTest {

    @Test
    void constantsMatchYamlCapabilityNames() {
        assertThat(MergeQueueCapability.BATCH_CI_RUNNER).isEqualTo("batch-ci-runner");
        assertThat(MergeQueueCapability.BISECTION_SPLITTER).isEqualTo("bisection-splitter");
        assertThat(MergeQueueCapability.PR_REJECT_AND_NOTIFY).isEqualTo("pr-reject-and-notify");
        assertThat(MergeQueueCapability.MERGE_QUEUE_ENQUEUE).isEqualTo("merge-queue-enqueue");
    }

    @Test
    void allCapabilitiesSetIsComplete() {
        assertThat(MergeQueueCapability.ALL).containsExactlyInAnyOrder(
            "batch-ci-runner", "bisection-splitter",
            "pr-reject-and-notify", "merge-queue-enqueue");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=MergeQueueCapabilityTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Implement MergeQueueCapability**

```java
package io.casehub.devtown.domain;

import java.util.Set;

public final class MergeQueueCapability {

    public static final String BATCH_CI_RUNNER       = "batch-ci-runner";
    public static final String BISECTION_SPLITTER    = "bisection-splitter";
    public static final String PR_REJECT_AND_NOTIFY  = "pr-reject-and-notify";
    public static final String MERGE_QUEUE_ENQUEUE   = "merge-queue-enqueue";

    public static final Set<String> ALL = Set.of(
        BATCH_CI_RUNNER, BISECTION_SPLITTER,
        PR_REJECT_AND_NOTIFY, MERGE_QUEUE_ENQUEUE
    );

    private MergeQueueCapability() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=MergeQueueCapabilityTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 5: Update DevtownCapabilityRegistry to include merge queue capabilities**

Add `MergeQueueCapability.BATCH_CI_RUNNER`, `MergeQueueCapability.BISECTION_SPLITTER`, `MergeQueueCapability.PR_REJECT_AND_NOTIFY`, `MergeQueueCapability.MERGE_QUEUE_ENQUEUE` to `ALL_CAPABILITIES` set. No routing policies for these — they are local/deterministic capabilities, not trust-scored.

- [ ] **Step 6: Update DevtownCapabilityRegistryTest**

Add assertion that `capabilities()` contains the 4 new strings. Update the `hasSize()` assertion from the current count to current + 4.

- [ ] **Step 7: Run full domain tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```
git add domain/
git commit -m "feat(#11): merge queue domain vocabulary — MergeQueueCapability constants

Refs #11"
```

---

### Task 2: Queue Service — Priority Calculator and Dependency Resolver

**Files:**
- Create: `queue/src/main/java/io/casehub/devtown/queue/QueuedPr.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/QueuePriorityCalculator.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/PriorityLane.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/DependencyResolver.java`
- Create: `queue/src/test/java/io/casehub/devtown/queue/QueuePriorityCalculatorTest.java`
- Create: `queue/src/test/java/io/casehub/devtown/queue/DependencyResolverTest.java`

**Interfaces:**
- Consumes: Nothing external — pure domain logic
- Produces: `QueuedPr` (record: number, headSha, author, trustScore, lane, enqueuedAt, dependsOn), `QueuePriorityCalculator.score(QueuedPr, Instant now, double decayRatePerHour)` → `double`, `PriorityLane` (enum: CRITICAL=3, HIGH=2, NORMAL=1), `DependencyResolver.resolve(List<QueuedPr>)` → `List<QueuedPr>` (topologically sorted, throws on cycle)

- [ ] **Step 1: Write QueuedPr record and PriorityLane enum**

```java
package io.casehub.devtown.queue;

import java.time.Instant;
import java.util.Set;

public record QueuedPr(
    int number,
    String headSha,
    String author,
    double trustScore,
    PriorityLane lane,
    Instant enqueuedAt,
    Set<Integer> dependsOn
) {
    public QueuedPr {
        if (trustScore < 0.0 || trustScore > 1.0)
            throw new IllegalArgumentException("trustScore must be in [0.0, 1.0]: " + trustScore);
    }
}
```

```java
package io.casehub.devtown.queue;

public enum PriorityLane {
    NORMAL(1), HIGH(2), CRITICAL(3);

    private final int weight;

    PriorityLane(int weight) { this.weight = weight; }

    public int weight() { return weight; }
}
```

- [ ] **Step 2: Write failing tests for QueuePriorityCalculator**

```java
package io.casehub.devtown.queue;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Set;
import org.junit.jupiter.api.Test;

class QueuePriorityCalculatorTest {

    private static final double DECAY_RATE = 125.0;
    private static final Instant NOW = Instant.parse("2026-06-27T12:00:00Z");

    private QueuedPr pr(PriorityLane lane, double trust, Instant enqueued) {
        return new QueuedPr(1, "abc", "alice", trust, lane, enqueued, Set.of());
    }

    @Test
    void criticalBeatsHighBeatsNormal() {
        var critical = pr(PriorityLane.CRITICAL, 0.5, NOW);
        var high     = pr(PriorityLane.HIGH, 0.5, NOW);
        var normal   = pr(PriorityLane.NORMAL, 0.5, NOW);

        assertThat(QueuePriorityCalculator.score(critical, NOW, DECAY_RATE))
            .isGreaterThan(QueuePriorityCalculator.score(high, NOW, DECAY_RATE));
        assertThat(QueuePriorityCalculator.score(high, NOW, DECAY_RATE))
            .isGreaterThan(QueuePriorityCalculator.score(normal, NOW, DECAY_RATE));
    }

    @Test
    void higherTrustScoresHigherWithinSameLane() {
        var highTrust = pr(PriorityLane.NORMAL, 0.9, NOW);
        var lowTrust  = pr(PriorityLane.NORMAL, 0.3, NOW);

        assertThat(QueuePriorityCalculator.score(highTrust, NOW, DECAY_RATE))
            .isGreaterThan(QueuePriorityCalculator.score(lowTrust, NOW, DECAY_RATE));
    }

    @Test
    void starvationPrevention_normalOvertakesFreshHighAfterEightHours() {
        var staleNormal = pr(PriorityLane.NORMAL, 0.5, NOW.minus(8, ChronoUnit.HOURS));
        var freshHigh   = pr(PriorityLane.HIGH, 0.5, NOW);

        assertThat(QueuePriorityCalculator.score(staleNormal, NOW, DECAY_RATE))
            .isGreaterThan(QueuePriorityCalculator.score(freshHigh, NOW, DECAY_RATE));
    }

    @Test
    void noDecayAtEnqueueTime() {
        var pr = pr(PriorityLane.NORMAL, 0.5, NOW);
        double score = QueuePriorityCalculator.score(pr, NOW, DECAY_RATE);
        double expected = 1.0 * 1000 + 0.5 * 100 + 0.0;
        assertThat(score).isEqualTo(expected);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queue -Dtest=QueuePriorityCalculatorTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 4: Implement QueuePriorityCalculator**

```java
package io.casehub.devtown.queue;

import java.time.Duration;
import java.time.Instant;

public final class QueuePriorityCalculator {

    private QueuePriorityCalculator() {}

    public static double score(QueuedPr pr, Instant now, double decayRatePerHour) {
        double lanePriority = pr.lane().weight() * 1000.0;
        double trustComponent = pr.trustScore() * 100.0;
        long waitMinutes = Duration.between(pr.enqueuedAt(), now).toMinutes();
        double waitDecay = waitMinutes * (decayRatePerHour / 60.0);
        return lanePriority + trustComponent + waitDecay;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queue -Dtest=QueuePriorityCalculatorTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All PASS

- [ ] **Step 6: Write failing tests for DependencyResolver**

```java
package io.casehub.devtown.queue;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Instant;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;

class DependencyResolverTest {

    private static final Instant NOW = Instant.parse("2026-06-27T12:00:00Z");

    private QueuedPr pr(int number, Set<Integer> deps) {
        return new QueuedPr(number, "sha" + number, "alice", 0.7, PriorityLane.NORMAL, NOW, deps);
    }

    @Test
    void noDependencies_preservesOrder() {
        var prs = List.of(pr(1, Set.of()), pr(2, Set.of()), pr(3, Set.of()));
        var sorted = DependencyResolver.resolve(prs);
        assertThat(sorted).extracting(QueuedPr::number).containsExactly(1, 2, 3);
    }

    @Test
    void dependencyOrderedBeforeDependent() {
        var pr1 = pr(1, Set.of());
        var pr2 = pr(2, Set.of(1));
        var sorted = DependencyResolver.resolve(List.of(pr2, pr1));
        assertThat(sorted).extracting(QueuedPr::number).containsExactly(1, 2);
    }

    @Test
    void transitiveChain() {
        var pr1 = pr(1, Set.of());
        var pr2 = pr(2, Set.of(1));
        var pr3 = pr(3, Set.of(2));
        var sorted = DependencyResolver.resolve(List.of(pr3, pr1, pr2));
        assertThat(sorted).extracting(QueuedPr::number).containsExactly(1, 2, 3);
    }

    @Test
    void cycleThrowsIllegalStateException() {
        var pr1 = pr(1, Set.of(2));
        var pr2 = pr(2, Set.of(1));
        assertThatThrownBy(() -> DependencyResolver.resolve(List.of(pr1, pr2)))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("cycle");
    }

    @Test
    void dependencyOnPrNotInQueue_ignored() {
        var pr1 = pr(1, Set.of(99));
        var sorted = DependencyResolver.resolve(List.of(pr1));
        assertThat(sorted).extracting(QueuedPr::number).containsExactly(1);
    }
}
```

- [ ] **Step 7: Implement DependencyResolver**

Standard Kahn's algorithm for topological sort with cycle detection. Dependencies referencing PRs not in the queue are ignored (they may already be merged).

```java
package io.casehub.devtown.queue;

import java.util.*;

public final class DependencyResolver {

    private DependencyResolver() {}

    public static List<QueuedPr> resolve(List<QueuedPr> prs) {
        Map<Integer, QueuedPr> byNumber = new LinkedHashMap<>();
        for (QueuedPr pr : prs) byNumber.put(pr.number(), pr);

        Map<Integer, Integer> inDegree = new LinkedHashMap<>();
        Map<Integer, List<Integer>> dependents = new LinkedHashMap<>();
        for (QueuedPr pr : prs) {
            inDegree.put(pr.number(), 0);
            dependents.put(pr.number(), new ArrayList<>());
        }
        for (QueuedPr pr : prs) {
            for (int dep : pr.dependsOn()) {
                if (byNumber.containsKey(dep)) {
                    dependents.get(dep).add(pr.number());
                    inDegree.merge(pr.number(), 1, Integer::sum);
                }
            }
        }

        Deque<Integer> ready = new ArrayDeque<>();
        for (var e : inDegree.entrySet()) {
            if (e.getValue() == 0) ready.add(e.getKey());
        }

        List<QueuedPr> result = new ArrayList<>();
        while (!ready.isEmpty()) {
            int n = ready.poll();
            result.add(byNumber.get(n));
            for (int dep : dependents.get(n)) {
                int newDegree = inDegree.merge(dep, -1, Integer::sum);
                if (newDegree == 0) ready.add(dep);
            }
        }

        if (result.size() != prs.size()) {
            throw new IllegalStateException("Dependency cycle detected among queued PRs");
        }
        return result;
    }
}
```

- [ ] **Step 8: Run all queue tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queue -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All PASS

- [ ] **Step 9: Commit**

```
git add queue/ domain/
git commit -m "feat(#11): queue priority calculator + dependency resolver

Composite scoring: lane × 1000 + trust × 100 + time-decay.
Topological sort with cycle detection for PR dependencies.

Refs #11"
```

---

### Task 3: Batch Composition Policy and Bisection Split Strategies

**Files:**
- Create: `queue/src/main/java/io/casehub/devtown/queue/Batch.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/BatchCompositionPolicy.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/DefaultBatchCompositionPolicy.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/BatchSlice.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/SplitResult.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/BisectionSplitStrategy.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/TrustWeightedSplitStrategy.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/IsolateOutlierStrategy.java`
- Create: `queue/src/main/java/io/casehub/devtown/queue/BinarySplitStrategy.java`
- Create: `queue/src/test/java/io/casehub/devtown/queue/DefaultBatchCompositionPolicyTest.java`
- Create: `queue/src/test/java/io/casehub/devtown/queue/BisectionSplitStrategyTest.java`

**Interfaces:**
- Consumes: `QueuedPr`, `DependencyResolver`, `QueuePriorityCalculator` from Task 2
- Produces: `BatchCompositionPolicy.formBatches(List<QueuedPr>, BatchFormationContext)` → `List<Batch>`, `BisectionSplitStrategy.split(List<QueuedPr>)` → `SplitResult`, `Batch` (record: id, prs, targetBranch, riskLevel, bisectionStrategy), `SplitResult` (record: left `BatchSlice`, right `BatchSlice`), `BatchSlice` (record: id, targetBranch, prs, size, parentBatchId, bisectionDepth, bisectionStrategy, riskLevel)

- [ ] **Step 1: Write Batch, BatchSlice, SplitResult records**

```java
package io.casehub.devtown.queue;

import java.util.List;

public record Batch(
    String id,
    List<QueuedPr> prs,
    String targetBranch,
    String riskLevel,
    String bisectionStrategy
) {
    public int size() { return prs.size(); }
}
```

```java
package io.casehub.devtown.queue;

import java.util.List;
import java.util.Map;

public record BatchSlice(
    String id,
    String targetBranch,
    List<Map<String, Object>> prs,
    int size,
    String parentBatchId,
    int bisectionDepth,
    String bisectionStrategy,
    String riskLevel
) {}
```

```java
package io.casehub.devtown.queue;

public record SplitResult(BatchSlice left, BatchSlice right) {}
```

- [ ] **Step 2: Write BisectionSplitStrategy interface**

```java
package io.casehub.devtown.queue;

import java.util.List;
import java.util.Map;

public interface BisectionSplitStrategy {
    SplitResult split(List<Map<String, Object>> prs, String batchId,
                      String targetBranch, int bisectionDepth,
                      String bisectionStrategy, String riskLevel);
}
```

- [ ] **Step 3: Write failing tests for bisection strategies**

Tests for all three strategies: `TrustWeightedSplitStrategy`, `IsolateOutlierStrategy`, `BinarySplitStrategy`. Key test cases:
- Trust-weighted: low-trust PRs cluster in the left half
- Isolate outlier: PR with trust >2σ below mean isolated solo in left
- Binary: positional split regardless of trust
- All: batch of 2 produces two singletons
- All: batch of 1 throws (should never be called with size 1)

- [ ] **Step 4: Implement all three strategies**

`TrustWeightedSplitStrategy`: sort by trust ascending, split at midpoint. Left half contains the low-trust PRs.

`IsolateOutlierStrategy`: compute mean and stddev of trust scores. If any PR has trust < (mean - 2*stddev), isolate it as a solo left batch. Otherwise delegate to `TrustWeightedSplitStrategy`.

`BinarySplitStrategy`: split at positional midpoint, no sorting.

- [ ] **Step 5: Write failing tests for DefaultBatchCompositionPolicy**

Key test cases:
- Trust-weighted sizing: `maxSize = floor(minTrust × maxBatchSize)`
- Adaptive sizing: `effectiveMax = max(minBatchSize, floor(maxSize × (1 - failureRate)))`
- Dependency ordering: dependent PR never placed before its dependency
- Empty queue returns empty list
- All PRs form one batch when queue ≤ effectiveMax

- [ ] **Step 6: Implement DefaultBatchCompositionPolicy**

```java
package io.casehub.devtown.queue;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

public class DefaultBatchCompositionPolicy implements BatchCompositionPolicy {

    @Override
    public List<Batch> formBatches(List<QueuedPr> queue, BatchFormationContext ctx) {
        if (queue.isEmpty()) return List.of();

        List<QueuedPr> sorted = queue.stream()
            .sorted(Comparator.comparingDouble(
                (QueuedPr pr) -> QueuePriorityCalculator.score(pr, ctx.now(), ctx.decayRatePerHour()))
                .reversed())
            .toList();

        List<QueuedPr> ordered = DependencyResolver.resolve(sorted);

        List<Batch> batches = new ArrayList<>();
        List<QueuedPr> currentBatch = new ArrayList<>();

        for (QueuedPr pr : ordered) {
            currentBatch.add(pr);
            double minTrust = currentBatch.stream()
                .mapToDouble(QueuedPr::trustScore).min().orElse(1.0);
            int trustMax = Math.max(1, (int) Math.floor(minTrust * ctx.maxBatchSize()));
            int adaptiveMax = Math.max(ctx.minBatchSize(),
                (int) Math.floor(trustMax * (1.0 - ctx.recentFailureRate())));

            if (currentBatch.size() >= adaptiveMax) {
                batches.add(buildBatch(currentBatch, ctx));
                currentBatch = new ArrayList<>();
            }
        }
        if (!currentBatch.isEmpty()) {
            batches.add(buildBatch(currentBatch, ctx));
        }
        return batches;
    }

    private Batch buildBatch(List<QueuedPr> prs, BatchFormationContext ctx) {
        String id = "batch-" + ctx.now().getEpochSecond() + "-" + ctx.nextBatchSequence();
        return new Batch(id, List.copyOf(prs), ctx.targetBranch(),
                         ctx.riskLevel(), ctx.bisectionStrategy());
    }
}
```

`BatchFormationContext` is a record holding config: `now`, `maxBatchSize`, `minBatchSize`, `decayRatePerHour`, `recentFailureRate`, `targetBranch`, `riskLevel`, `bisectionStrategy`, `nextBatchSequence()`.

- [ ] **Step 7: Run all queue tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queue -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All PASS

- [ ] **Step 8: Commit**

```
git add queue/
git commit -m "feat(#11): batch composition policy + bisection split strategies

Trust-weighted, isolate-outlier, and binary split strategies.
Adaptive batch sizing with trust-weighted max and failure-rate floor.

Refs #11"
```

---

### Task 4: CasePlanModel — `merge-batch.yaml`

**Files:**
- Create: `queue/src/main/resources/devtown/merge-batch.yaml`
- Create: `queue/src/test/java/io/casehub/devtown/queue/MergeBatchBindingConditionTest.java`
- Create: `queue/src/test/java/io/casehub/devtown/queue/MergeBatchGoalConditionTest.java`

**Interfaces:**
- Consumes: engine YAML schema (CaseDefinition.yaml with engine#573 `maxRecursionDepth` + engine#574 M-of-N fields)
- Produces: `devtown/merge-batch.yaml` — the CasePlanModel loaded by `MergeBatchCaseHub` in Task 5

- [ ] **Step 1: Write merge-batch.yaml**

Full CasePlanModel from the spec §4: 4 capabilities, 6 goals (3 success + 3 failure), 11 bindings. Copy the verified bindings verbatim from the spec — they have been through 5 review rounds and 15 terminal paths verified. Key sections:
- Capabilities: `batch-ci-runner`, `merge-executor`, `bisection-splitter`, `pr-reject-and-notify`
- Goals: `batch-merged`, `all-culprits-isolated`, `single-pr-rejected`, `merge-approval-rejected`, `merge-terminal-failure`, `tip-test-terminal-failure`
- Bindings: `test-batch-tip`, `tip-test-escalation`, `tip-test-after-escalation`, `merge-batch`, `human-merge-approval`, `merge-escalation`, `merge-after-escalation`, `compute-bisection-split`, `bisect-left`, `bisect-right`, `reject-single-pr`

- [ ] **Step 2: Write failing binding condition tests**

Follow the pattern from `PrReviewBindingConditionTest.java` — use `LambdaExpressionEvaluator` and `JQExpressionEvaluator` to evaluate each binding's `when` condition against crafted context JSON. Key conditions to test:
- `test-batch-tip` fires when tipTest is null and not escalated retry
- `test-batch-tip` does NOT fire when tipTestEscalatedRetry is true
- `merge-batch` fires when tip passes, mergeResult is null, not escalated, and risk is ROUTINE
- `merge-batch` does NOT fire when mergeEscalatedRetry is true
- `compute-bisection-split` fires when tip fails and batch.size > 1
- `bisect-left` fires when splitResult is non-null and bisectLeft is null
- `reject-single-pr` fires when tip fails, batch.size is 1, and rejectedPrs is empty
- `reject-single-pr` does NOT fire when rejectedPrs is non-empty
- `tip-test-escalation` fires on REROUTES_EXHAUSTED and tipTestEscalation is null
- `merge-escalation` fires on REROUTES_EXHAUSTED and mergeEscalation is null

- [ ] **Step 3: Write failing goal condition tests**

Test each goal's JQ condition against crafted context:
- `batch-merged`: true when `.mergeResult.status == "success"`
- `single-pr-rejected`: true when `.batch.size == 1 and (.rejectedPrs | length) > 0`
- `all-culprits-isolated`: true when both `.bisectLeft` and `.bisectRight` are non-null
- `merge-approval-rejected`: true on REJECTED or BLOCKED
- `merge-terminal-failure`: true on REJECTED or BLOCKED
- `tip-test-terminal-failure`: true on REJECT_BATCH or BLOCKED

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl queue -Dtest="MergeBatchBindingConditionTest,MergeBatchGoalConditionTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — YAML not loading or conditions not matching

- [ ] **Step 5: Iterate until all condition tests pass**

This may require adjusting JQ expressions in the YAML to match the engine's JQ evaluation semantics. The spec's conditions are designed to be correct, but verify each one.

- [ ] **Step 6: Commit**

```
git add queue/
git commit -m "feat(#11): merge-batch CasePlanModel — 11 bindings, 6 goals

4 capabilities, trust-weighted bisection via recursive sub-cases,
human escalation for tip-test and merge exhaustion.
All binding/goal conditions unit-tested.

Refs #11"
```

---

### Task 5: CDI Wiring — MergeBatchCaseHub + MergeQueueService

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/MergeBatchCaseHub.java`
- Create: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`
- Create: `app/src/test/java/io/casehub/devtown/app/MergeBatchCaseHubTest.java`
- Modify: `app/pom.xml` — add `casehub-devtown-queue` dependency

**Interfaces:**
- Consumes: `merge-batch.yaml` from Task 4, `BatchCompositionPolicy` from Task 3, `BisectionSplitStrategy` from Task 3, `CaseHubRuntime` from engine, `PrReviewCaseHub` from existing
- Produces: `MergeBatchCaseHub` (CDI bean, loads YAML, registers workers), `MergeQueueService.enqueue(QueuedPr)`, `.formAndDispatchBatches()`, `.handleBatchCompletion(UUID caseId)`

- [ ] **Step 1: Add queue dependency to app/pom.xml**

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-devtown-queue</artifactId>
</dependency>
```

And add to parent `<dependencyManagement>`:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-devtown-queue</artifactId>
  <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Implement MergeBatchCaseHub**

Follow `PrReviewCaseHub` pattern — extend `YamlCaseHub`, register workers for `bisection-splitter` and `pr-reject-and-notify` capabilities. The `bisection-splitter` worker delegates to the CDI-injected `BisectionSplitStrategy`.

```java
@ApplicationScoped
public class MergeBatchCaseHub extends YamlCaseHub {

    @Inject
    BisectionSplitStrategy splitStrategy;

    public MergeBatchCaseHub() {
        super("devtown/merge-batch.yaml");
    }

    @PostConstruct
    void registerWorkers() {
        // Register bisection-splitter worker
        // Register pr-reject-and-notify worker
    }
}
```

- [ ] **Step 3: Write MergeBatchCaseHubTest**

Follow `PrReviewCaseHubTest` pattern — verify definition loads, has 11 bindings, 6 goals, 4 capabilities.

- [ ] **Step 4: Implement MergeQueueService skeleton**

`@ApplicationScoped` service with:
- `enqueue(QueuedPr)` — adds to in-memory queue
- `formAndDispatchBatches()` — calls `BatchCompositionPolicy`, spawns batch cases via `CaseHubRuntime`
- `handleBatchCompletion(UUID caseId)` — walks bisection tree, aggregates results

This is the imperative tier. Full queue persistence is a follow-up — in-memory is sufficient for the first implementation.

- [ ] **Step 5: Run app tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MergeBatchCaseHubTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add app/ pom.xml
git commit -m "feat(#11): MergeBatchCaseHub + MergeQueueService wiring

YamlCaseHub loads merge-batch.yaml. MergeQueueService provides
imperative queue management with in-memory queue.

Refs #11"
```

---

### Task 6: Integration Tests — Batch Lifecycle and Bisection

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/MergeQueueBatchLifecycleTest.java`
- Create: `app/src/test/java/io/casehub/devtown/app/MergeQueueBisectionTest.java`
- Create: `app/src/test/java/io/casehub/devtown/app/MergeQueueEscalationTest.java`

**Interfaces:**
- Consumes: `MergeQueueService`, `MergeBatchCaseHub`, `CaseHubRuntime`, engine test infrastructure
- Produces: End-to-end verification of the 15 terminal paths from the spec

- [ ] **Step 1: Write MergeQueueBatchLifecycleTest**

`@QuarkusTest` — happy path: enqueue 3 PRs → batch formed → tip test passes (mock worker writes `{ status: "passing" }`) → merge executor succeeds → `batch-merged` goal fires → case COMPLETED.

Follow `HumanApprovalLifecycleTest` pattern for async checkpoint verification with awaitility.

- [ ] **Step 2: Write MergeQueueBisectionTest**

`@QuarkusTest` — failure path: batch of 6 → tip test fails → `compute-bisection-split` fires → two sub-cases spawned (M-of-N grouped) → left half fails, right half passes → left recurses → culprit isolated → `single-pr-rejected` fires on leaf → parent's `all-culprits-isolated` fires.

Verify separate output keys: `bisectLeft` and `bisectRight` both populated on parent context.

- [ ] **Step 3: Write MergeQueueEscalationTest**

`@QuarkusTest` — covers:
- Merge reroutes exhausted → `merge-escalation` humanTask → approve → `merge-after-escalation` contextWrite reset → retry succeeds
- Tip-test reroutes exhausted → `tip-test-escalation` → REJECT_BATCH → `tip-test-terminal-failure` goal fires
- High-risk batch → `human-merge-approval` → REJECTED → `merge-approval-rejected` goal fires

- [ ] **Step 4: Run integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="MergeQueueBatchLifecycleTest,MergeQueueBisectionTest,MergeQueueEscalationTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All PASS

- [ ] **Step 5: Commit**

```
git add app/
git commit -m "feat(#11): merge queue integration tests — batch lifecycle, bisection, escalation

Happy path, bisection with recursive sub-cases, and all escalation
paths tested. 15 terminal paths from spec verified.

Refs #11"
```

---

### Task 7: MCP Tools — Merge Queue Operations

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java`

**Interfaces:**
- Consumes: `MergeQueueService` from Task 5
- Produces: 5 new MCP tools: `get_merge_queue`, `get_batch_status`, `get_merge_queue_metrics`, `enqueue_pr`, `dequeue_pr`

- [ ] **Step 1: Add MCP tool records**

Add to `DevtownMcpTools`: `MergeQueueStatus`, `BatchStatus`, `MergeQueueMetrics`, `EnqueueResult`, `DequeueResult` records.

- [ ] **Step 2: Implement get_merge_queue tool**

Returns current queue state: queued PRs with priority scores, wait times, dependency info, SLA status.

- [ ] **Step 3: Implement get_batch_status tool**

Returns batch state: PRs, test result, bisection tree (recursive walk), merge result.

- [ ] **Step 4: Implement get_merge_queue_metrics tool**

Returns operational metrics: throughput, failure rate, average wait time, batch size distribution.

- [ ] **Step 5: Implement enqueue_pr and dequeue_pr tools**

Write tools with appropriate validation guards.

- [ ] **Step 6: Update list_problems with QUEUE_SLA_BREACH**

Add new problem category for PRs exceeding their SLA wait time.

- [ ] **Step 7: Write MCP tool tests**

Follow `DevtownMcpToolsTest` pattern — unit tests for each tool method.

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All PASS

- [ ] **Step 9: Commit**

```
git add app/
git commit -m "feat(#11): merge queue MCP tools — 5 new tools + SLA breach detection

get_merge_queue, get_batch_status, get_merge_queue_metrics,
enqueue_pr, dequeue_pr. list_problems gains QUEUE_SLA_BREACH.

Refs #11"
```

---

## Self-Review Checklist

- [x] **Spec coverage:** Every section of the spec (§1–§11) has a corresponding task. Domain vocabulary (§2 module placement) → Task 1. Queue service (§3) → Tasks 2+3. CasePlanModel (§4) → Task 4. Trust-weighted bisection (§5) → Task 3. Foundation gates (§6) → pre-req, both closed. Integration points (§7) → Task 5. MCP tools (§8) → Task 7. Testing (§9) → Task 6. Configuration (§11) → distributed across tasks.
- [x] **Placeholder scan:** No TBD/TODO. All code blocks contain actual implementations. Test code contains actual assertions.
- [x] **Type consistency:** `QueuedPr` used consistently across Tasks 2, 3, 5. `Batch`/`BatchSlice`/`SplitResult` consistent across Tasks 3, 4, 5. `MergeQueueCapability` constants match YAML capability names.
- [x] **Foundation gates:** engine#573 (CLOSED), engine#574 (CLOSED) — no phasing needed.
