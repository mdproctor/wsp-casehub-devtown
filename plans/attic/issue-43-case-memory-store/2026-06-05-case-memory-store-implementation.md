# CaseMemoryStore Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate CaseMemoryStore into devtown PR review cases — emit structured facts on binding completion, recall prior context before case open.

**Architecture:** Two-layer emission: ReviewOutcomeObserver extracts structured data from engine events and fires a typed ReviewCompletedEvent; CaseMemoryEmitter receives it and calls CaseMemoryStore.storeAll(). Recall: CaseMemoryRecaller queries contributor and code-area history before case creation, enriching the initial context.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI @ObservesAsync, casehub-platform-api memory SPI, casehub-engine blackboard events

**Spec:** `specs/2026-06-05-case-memory-store-integration-design.md`

**Project repo:** `/Users/mdproctor/claude/casehub/devtown`

---

## File Structure

| Module | File | Action | Responsibility |
|---|---|---|---|
| domain | `domain/src/main/java/io/casehub/devtown/domain/memory/DevtownMemoryDomain.java` | Create | MemoryDomain constant |
| domain | `domain/src/main/java/io/casehub/devtown/domain/memory/DevtownMemoryKeys.java` | Create | Devtown-specific attribute key constants |
| domain | `domain/src/main/java/io/casehub/devtown/domain/memory/ReviewOutcome.java` | Create | COMPLETED/DECLINED/FAILED enum |
| domain | `domain/src/main/java/io/casehub/devtown/domain/memory/ModulePathNormalizer.java` | Create | changedPaths → module-level dedup |
| domain | `domain/src/test/java/io/casehub/devtown/domain/memory/ModulePathNormalizerTest.java` | Create | Normalization edge cases |
| domain | `domain/src/test/java/io/casehub/devtown/domain/memory/DevtownMemoryDomainTest.java` | Create | Constants + enum values |
| review | `review/src/main/java/io/casehub/devtown/review/PrPayload.java` | Modify | Add contributor, changedPaths |
| review | `review/src/main/java/io/casehub/devtown/review/ReviewCompletedEvent.java` | Create | Typed intermediate event |
| review | `review/src/main/java/io/casehub/devtown/review/MemoryContext.java` | Create | Recall result record |
| review | `review/src/test/java/io/casehub/devtown/review/MemoryContextTest.java` | Create | toContextMap, hasRiskSignals |
| review | `review/pom.xml` | Modify | Add casehub-platform-api dep |
| app | `app/src/main/java/io/casehub/devtown/app/CaseMemoryEmitter.java` | Create | Pure function: event → storeAll |
| app | `app/src/main/java/io/casehub/devtown/app/ReviewOutcomeObserver.java` | Create | Extraction: engine events → ReviewCompletedEvent |
| app | `app/src/main/java/io/casehub/devtown/app/CaseMemoryRecaller.java` | Create | Recall before case open |
| app | `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java` | Modify | Wire recall |
| app | `app/src/test/java/io/casehub/devtown/app/CaseMemoryEmitterTest.java` | Create | Plain unit test |
| app | `app/src/test/java/io/casehub/devtown/app/ReviewOutcomeObserverTest.java` | Create | @QuarkusTest |
| app | `app/src/test/java/io/casehub/devtown/app/CaseMemoryRecallerTest.java` | Create | @QuarkusTest |
| app | `app/src/test/java/io/casehub/devtown/app/CaseMemoryIntegrationTest.java` | Create | Full round-trip @QuarkusTest |
| app | Multiple existing test files | Modify | PrPayload constructor migration |

---

### Task 1: Domain Model — DevtownMemoryDomain, DevtownMemoryKeys, ReviewOutcome, ModulePathNormalizer

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/memory/DevtownMemoryDomain.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/memory/DevtownMemoryKeys.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/memory/ReviewOutcome.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/memory/ModulePathNormalizer.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/memory/ModulePathNormalizerTest.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/memory/DevtownMemoryDomainTest.java`

- [ ] **Step 1: Write ModulePathNormalizerTest**

```java
package io.casehub.devtown.domain.memory;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class ModulePathNormalizerTest {

    @Test
    void javaMainFile_extractsModuleName() {
        assertThat(ModulePathNormalizer.normalize(List.of(
            "app/src/main/java/io/casehub/devtown/app/Foo.java"
        ))).containsExactly("app");
    }

    @Test
    void javaTestFile_extractsModuleName() {
        assertThat(ModulePathNormalizer.normalize(List.of(
            "app/src/test/java/io/casehub/devtown/app/FooTest.java"
        ))).containsExactly("app");
    }

    @Test
    void mainAndTest_sameModule_deduplicates() {
        assertThat(ModulePathNormalizer.normalize(List.of(
            "app/src/main/java/io/casehub/devtown/app/Foo.java",
            "app/src/test/java/io/casehub/devtown/app/FooTest.java"
        ))).containsExactly("app");
    }

    @Test
    void multipleModules_returnsDistinct() {
        assertThat(ModulePathNormalizer.normalize(List.of(
            "app/src/main/java/io/casehub/Foo.java",
            "domain/src/main/java/io/casehub/Bar.java"
        ))).containsExactlyInAnyOrder("app", "domain");
    }

    @Test
    void multipleFilesInSameModule_deduplicates() {
        assertThat(ModulePathNormalizer.normalize(List.of(
            "app/src/main/java/io/casehub/Foo.java",
            "app/src/main/java/io/casehub/Bar.java"
        ))).containsExactly("app");
    }

    @Test
    void rootLevelFile_pomXml_returnsRoot() {
        assertThat(ModulePathNormalizer.normalize(List.of("pom.xml")))
            .containsExactly("(root)");
    }

    @Test
    void rootLevelFile_readme_returnsRoot() {
        assertThat(ModulePathNormalizer.normalize(List.of("README.md")))
            .containsExactly("(root)");
    }

    @Test
    void configFile_noSrcBoundary_returnsRoot() {
        assertThat(ModulePathNormalizer.normalize(List.of("config/application.yaml")))
            .containsExactly("(root)");
    }

    @Test
    void githubWorkflow_returnsRoot() {
        assertThat(ModulePathNormalizer.normalize(List.of(".github/workflows/ci.yml")))
            .containsExactly("(root)");
    }

    @Test
    void mixedRootAndModule_returnsBoth() {
        assertThat(ModulePathNormalizer.normalize(List.of(
            "pom.xml",
            "app/src/main/java/io/casehub/Foo.java"
        ))).containsExactlyInAnyOrder("(root)", "app");
    }

    @Test
    void emptyList_returnsEmpty() {
        assertThat(ModulePathNormalizer.normalize(List.of())).isEmpty();
    }

    @Test
    void nestedModulePath_extractsFirstSegmentBeforeSrc() {
        assertThat(ModulePathNormalizer.normalize(List.of(
            "review/src/main/java/io/casehub/devtown/review/PrPayload.java"
        ))).containsExactly("review");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ModulePathNormalizerTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: Compilation failure — `ModulePathNormalizer` does not exist

- [ ] **Step 3: Write ModulePathNormalizer**

```java
package io.casehub.devtown.domain.memory;

import java.util.List;

public final class ModulePathNormalizer {

    public static final String ROOT = "(root)";

    public static List<String> normalize(List<String> changedPaths) {
        return changedPaths.stream()
            .map(ModulePathNormalizer::toModule)
            .distinct()
            .toList();
    }

    private static String toModule(String path) {
        int srcIdx = path.indexOf("/src/");
        if (srcIdx <= 0) return ROOT;
        return path.substring(0, srcIdx);
    }

    private ModulePathNormalizer() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ModulePathNormalizerTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All 12 tests PASS

- [ ] **Step 5: Write DevtownMemoryDomainTest**

```java
package io.casehub.devtown.domain.memory;

import io.casehub.platform.api.memory.MemoryDomain;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class DevtownMemoryDomainTest {

    @Test
    void domainConstantHasCorrectName() {
        assertThat(DevtownMemoryDomain.SOFTWARE_REVIEW)
            .isEqualTo(new MemoryDomain("software-review"));
    }

    @Test
    void reviewOutcomeValues() {
        assertThat(ReviewOutcome.values()).containsExactly(
            ReviewOutcome.COMPLETED, ReviewOutcome.DECLINED, ReviewOutcome.FAILED);
    }

    @Test
    void devtownMemoryKeysAreKebabCase() {
        assertThat(DevtownMemoryKeys.CAPABILITY).matches("[a-z-]+");
        assertThat(DevtownMemoryKeys.PR_NUMBER).matches("[a-z-]+");
        assertThat(DevtownMemoryKeys.PR_REPO).matches("[a-z-]+");
        assertThat(DevtownMemoryKeys.LINES_CHANGED).matches("[a-z-]+");
        assertThat(DevtownMemoryKeys.ENTITY_TYPE).matches("[a-z-]+");
        assertThat(DevtownMemoryKeys.OUTCOME_DETAIL).matches("[a-z-]+");
    }

    @Test
    void devtownMemoryKeysMatchSpec() {
        assertThat(DevtownMemoryKeys.CAPABILITY).isEqualTo("capability");
        assertThat(DevtownMemoryKeys.PR_NUMBER).isEqualTo("pr-number");
        assertThat(DevtownMemoryKeys.PR_REPO).isEqualTo("pr-repo");
        assertThat(DevtownMemoryKeys.LINES_CHANGED).isEqualTo("lines-changed");
        assertThat(DevtownMemoryKeys.ENTITY_TYPE).isEqualTo("entity-type");
        assertThat(DevtownMemoryKeys.OUTCOME_DETAIL).isEqualTo("outcome-detail");
    }
}
```

- [ ] **Step 6: Write DevtownMemoryDomain, DevtownMemoryKeys, ReviewOutcome**

```java
// DevtownMemoryDomain.java
package io.casehub.devtown.domain.memory;

import io.casehub.platform.api.memory.MemoryDomain;

public final class DevtownMemoryDomain {
    public static final MemoryDomain SOFTWARE_REVIEW = new MemoryDomain("software-review");
    private DevtownMemoryDomain() {}
}
```

```java
// DevtownMemoryKeys.java
package io.casehub.devtown.domain.memory;

public final class DevtownMemoryKeys {
    public static final String CAPABILITY = "capability";
    public static final String PR_NUMBER = "pr-number";
    public static final String PR_REPO = "pr-repo";
    public static final String LINES_CHANGED = "lines-changed";
    public static final String ENTITY_TYPE = "entity-type";
    public static final String OUTCOME_DETAIL = "outcome-detail";
    private DevtownMemoryKeys() {}
}
```

```java
// ReviewOutcome.java
package io.casehub.devtown.domain.memory;

public enum ReviewOutcome {
    COMPLETED, DECLINED, FAILED
}
```

- [ ] **Step 7: Run all domain tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All domain tests PASS (existing + new)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#43): add memory domain model — DevtownMemoryDomain, DevtownMemoryKeys, ReviewOutcome, ModulePathNormalizer

Refs #43"
```

---

### Task 2: PrPayload Enhancement — add contributor and changedPaths

**Files:**
- Modify: `review/src/main/java/io/casehub/devtown/review/PrPayload.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/PrReviewServiceTest.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/PrReviewQhorusLifecycleTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

The PrPayload record gets two new fields. This is a compile-breaking change — every constructor call site must supply the new values. The migration is mechanical.

- [ ] **Step 1: Modify PrPayload**

Change `review/src/main/java/io/casehub/devtown/review/PrPayload.java`:

```java
package io.casehub.devtown.review;

import java.util.List;

public record PrPayload(
    String repo,
    int prNumber,
    String headSha,
    String baseRef,
    int linesChanged,
    String contributor,
    List<String> changedPaths
) {}
```

- [ ] **Step 2: Fix all PrPayload constructor call sites**

Every `new PrPayload(repo, prNumber, headSha, baseRef, linesChanged)` becomes `new PrPayload(repo, prNumber, headSha, baseRef, linesChanged, "test-contributor", List.of())` in tests. In production code, the values will come from the REST payload.

Files to fix — add `, "test-contributor", List.of()` to every existing 5-arg constructor:
- `PrReviewServiceTest.java` — 3 sites
- `PrReviewQhorusLifecycleTest.java` — 21 sites

For `PrReviewCaseService.java`, add the new fields to the prContext map:

```java
var prContext = Map.<String, Object>of(
    "id", String.valueOf(pr.prNumber()),
    "repo", pr.repo(),
    "linesChanged", pr.linesChanged(),
    "baseRef", pr.baseRef(),
    "headSha", pr.headSha(),
    "contributor", pr.contributor(),
    "changedPaths", pr.changedPaths()
);
```

The HumanApprovalLifecycleTest.java and SlaBreachLifecycleTest.java don't construct PrPayload — they build case context maps directly. No changes needed.

- [ ] **Step 3: Build all modules to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS — all constructor sites updated

- [ ] **Step 4: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All 59 existing tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add review/src/main/java/io/casehub/devtown/review/PrPayload.java app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/test/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#43): enhance PrPayload with contributor and changedPaths

Breaking change — all constructor call sites updated. Tests use
default values (test-contributor, empty list).

Refs #43"
```

---

### Task 3: ReviewCompletedEvent + MemoryContext in review module

**Files:**
- Modify: `review/pom.xml` (add casehub-platform-api explicit dep)
- Create: `review/src/main/java/io/casehub/devtown/review/ReviewCompletedEvent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/MemoryContext.java`
- Create: `review/src/test/java/io/casehub/devtown/review/MemoryContextTest.java`

- [ ] **Step 1: Add casehub-platform-api to review pom.xml**

Add to `review/pom.xml` `<dependencies>`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-api</artifactId>
</dependency>
```

- [ ] **Step 2: Write MemoryContextTest**

```java
package io.casehub.devtown.review;

import io.casehub.devtown.domain.memory.DevtownMemoryKeys;
import io.casehub.devtown.domain.memory.ReviewOutcome;
import io.casehub.platform.api.memory.Memory;
import io.casehub.platform.api.memory.MemoryAttributeKeys;
import io.casehub.platform.api.memory.MemoryDomain;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class MemoryContextTest {

    private static final MemoryDomain DOMAIN = new MemoryDomain("software-review");

    private Memory memory(String outcome, String outcomeDetail) {
        return new Memory("m1", "contributor:mdproctor", DOMAIN, "t1", "case1",
            "Some review text.", Map.of(
                MemoryAttributeKeys.OUTCOME, outcome,
                DevtownMemoryKeys.OUTCOME_DETAIL, outcomeDetail,
                DevtownMemoryKeys.CAPABILITY, "security-review"
            ), Instant.parse("2026-06-01T10:00:00Z"));
    }

    @Test
    void empty_hasNoRiskSignals() {
        assertThat(MemoryContext.EMPTY.hasRiskSignals()).isFalse();
    }

    @Test
    void empty_toContextMap_hasEmptyLists() {
        var map = MemoryContext.EMPTY.toContextMap();
        assertThat((List<?>) map.get("contributorHistory")).isEmpty();
        assertThat((List<?>) map.get("codeAreaHistory")).isEmpty();
    }

    @Test
    void completedApproved_noRiskSignal() {
        var ctx = new MemoryContext(
            List.of(memory(ReviewOutcome.COMPLETED.name(), "APPROVED")),
            List.of());
        assertThat(ctx.hasRiskSignals()).isFalse();
    }

    @Test
    void completedPassed_noRiskSignal() {
        var ctx = new MemoryContext(
            List.of(memory(ReviewOutcome.COMPLETED.name(), "passed")),
            List.of());
        assertThat(ctx.hasRiskSignals()).isFalse();
    }

    @Test
    void completedApprovedLowerCase_noRiskSignal() {
        var ctx = new MemoryContext(
            List.of(memory(ReviewOutcome.COMPLETED.name(), "approved")),
            List.of());
        assertThat(ctx.hasRiskSignals()).isFalse();
    }

    @Test
    void completedWithFindings_isRiskSignal() {
        var ctx = new MemoryContext(
            List.of(memory(ReviewOutcome.COMPLETED.name(), "FINDINGS_PRESENT")),
            List.of());
        assertThat(ctx.hasRiskSignals()).isTrue();
    }

    @Test
    void failed_isRiskSignal() {
        var ctx = new MemoryContext(
            List.of(memory(ReviewOutcome.FAILED.name(), "timeout")),
            List.of());
        assertThat(ctx.hasRiskSignals()).isTrue();
    }

    @Test
    void declined_notRiskSignal() {
        var ctx = new MemoryContext(
            List.of(memory(ReviewOutcome.DECLINED.name(), "out-of-scope")),
            List.of());
        assertThat(ctx.hasRiskSignals()).isFalse();
    }

    @Test
    void toContextMap_extractsFieldsCorrectly() {
        var m = memory(ReviewOutcome.COMPLETED.name(), "APPROVED");
        var ctx = new MemoryContext(List.of(m), List.of());
        var map = ctx.toContextMap();

        @SuppressWarnings("unchecked")
        var history = (List<Map<String, Object>>) map.get("contributorHistory");
        assertThat(history).hasSize(1);
        var entry = history.get(0);
        assertThat(entry.get("text")).isEqualTo("Some review text.");
        assertThat(entry.get("outcome")).isEqualTo("COMPLETED");
        assertThat(entry.get("capability")).isEqualTo("security-review");
        assertThat(entry.get("createdAt")).isEqualTo("2026-06-01T10:00:00Z");
    }

    @Test
    void codeAreaHistory_appearsInContextMap() {
        var m = memory(ReviewOutcome.COMPLETED.name(), "APPROVED");
        var ctx = new MemoryContext(List.of(), List.of(m));
        var map = ctx.toContextMap();

        @SuppressWarnings("unchecked")
        var history = (List<Map<String, Object>>) map.get("codeAreaHistory");
        assertThat(history).hasSize(1);
    }
}
```

- [ ] **Step 3: Write ReviewCompletedEvent and MemoryContext**

```java
// ReviewCompletedEvent.java
package io.casehub.devtown.review;

import io.casehub.devtown.domain.memory.ReviewOutcome;
import java.util.UUID;

public record ReviewCompletedEvent(
    UUID caseId,
    String tenantId,
    String capability,
    String reviewerId,
    ReviewOutcome outcome,
    String outcomeDetail,
    PrPayload pr
) {}
```

```java
// MemoryContext.java
package io.casehub.devtown.review;

import io.casehub.devtown.domain.memory.DevtownMemoryKeys;
import io.casehub.devtown.domain.memory.ReviewOutcome;
import io.casehub.platform.api.memory.Memory;
import io.casehub.platform.api.memory.MemoryAttributeKeys;
import java.util.List;
import java.util.Map;
import java.util.Set;

public record MemoryContext(
    List<Memory> contributorHistory,
    List<Memory> codeAreaHistory
) {
    public static final MemoryContext EMPTY = new MemoryContext(List.of(), List.of());

    private static final Set<String> SAFE_OUTCOMES = Set.of("APPROVED", "passed", "approved");

    public Map<String, Object> toContextMap() {
        return Map.of(
            "contributorHistory", toEntryList(contributorHistory),
            "codeAreaHistory", toEntryList(codeAreaHistory)
        );
    }

    public boolean hasRiskSignals() {
        return hasRisk(contributorHistory) || hasRisk(codeAreaHistory);
    }

    private static boolean hasRisk(List<Memory> memories) {
        return memories.stream().anyMatch(m -> {
            String outcome = m.attributes().get(MemoryAttributeKeys.OUTCOME);
            if (ReviewOutcome.FAILED.name().equals(outcome)) return true;
            if (ReviewOutcome.COMPLETED.name().equals(outcome)) {
                String detail = m.attributes().get(DevtownMemoryKeys.OUTCOME_DETAIL);
                return detail != null && !SAFE_OUTCOMES.contains(detail);
            }
            return false;
        });
    }

    private static List<Map<String, Object>> toEntryList(List<Memory> memories) {
        return memories.stream().map(m -> Map.<String, Object>of(
            "text", m.text(),
            "outcome", m.attributes().getOrDefault(MemoryAttributeKeys.OUTCOME, ""),
            "capability", m.attributes().getOrDefault(DevtownMemoryKeys.CAPABILITY, ""),
            "createdAt", m.createdAt().toString()
        )).toList();
    }
}
```

- [ ] **Step 4: Run review module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain,review -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All review + domain tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add review/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#43): add ReviewCompletedEvent and MemoryContext

ReviewCompletedEvent carries typed review outcome data.
MemoryContext wraps recall results with toContextMap() and
hasRiskSignals() (fail-closed: unknown outcome details = risk).

Refs #43"
```

---

### Task 4: CaseMemoryEmitter — pure function unit test

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/CaseMemoryEmitter.java`
- Create: `app/src/test/java/io/casehub/devtown/app/CaseMemoryEmitterTest.java`

This is the component that converts `ReviewCompletedEvent` into `List<MemoryInput>` and calls `storeAll()`. It's a pure function with no engine dependencies — fully unit testable.

- [ ] **Step 1: Write CaseMemoryEmitterTest**

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.memory.DevtownMemoryDomain;
import io.casehub.devtown.domain.memory.DevtownMemoryKeys;
import io.casehub.devtown.domain.memory.ReviewOutcome;
import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.ReviewCompletedEvent;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryAttributeKeys;
import io.casehub.platform.api.memory.MemoryInput;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class CaseMemoryEmitterTest {

    private final List<List<MemoryInput>> storedBatches = new ArrayList<>();
    private CaseMemoryEmitter emitter;

    @BeforeEach
    void setUp() {
        storedBatches.clear();
        CaseMemoryStore mockStore = mock(CaseMemoryStore.class);
        when(mockStore.storeAll(any())).thenAnswer(inv -> {
            @SuppressWarnings("unchecked")
            List<MemoryInput> inputs = inv.getArgument(0);
            storedBatches.add(new ArrayList<>(inputs));
            return inputs.stream().map(i -> "id-" + storedBatches.size()).toList();
        });

        @SuppressWarnings("unchecked")
        Instance<CaseMemoryStore> instance = mock(Instance.class);
        when(instance.isResolvable()).thenReturn(true);
        when(instance.get()).thenReturn(mockStore);

        emitter = new CaseMemoryEmitter();
        emitter.store = instance;
    }

    private ReviewCompletedEvent event(List<String> changedPaths) {
        return new ReviewCompletedEvent(
            UUID.randomUUID(),
            "tenant-1",
            "security-review",
            "security-agent-1",
            ReviewOutcome.COMPLETED,
            "APPROVED",
            new PrPayload("casehubio/devtown", 45, "sha1", "main", 342,
                "mdproctor", changedPaths)
        );
    }

    @Test
    void emits_contributorReviewerAndModuleFacts() {
        emitter.onReviewCompleted(event(List.of(
            "app/src/main/java/io/casehub/Foo.java")));

        assertThat(storedBatches).hasSize(1);
        var batch = storedBatches.get(0);
        assertThat(batch).hasSize(3); // contributor + reviewer + 1 module

        var entityTypes = batch.stream()
            .map(i -> i.attributes().get(DevtownMemoryKeys.ENTITY_TYPE))
            .toList();
        assertThat(entityTypes).containsExactlyInAnyOrder(
            "contributor", "reviewer", "code-area");
    }

    @Test
    void contributorFact_hasCorrectEntityId() {
        emitter.onReviewCompleted(event(List.of(
            "app/src/main/java/io/casehub/Foo.java")));
        var contributor = storedBatches.get(0).stream()
            .filter(i -> "contributor".equals(i.attributes().get(DevtownMemoryKeys.ENTITY_TYPE)))
            .findFirst().orElseThrow();
        assertThat(contributor.entityId()).isEqualTo("contributor:mdproctor");
    }

    @Test
    void reviewerFact_hasCorrectEntityId() {
        emitter.onReviewCompleted(event(List.of(
            "app/src/main/java/io/casehub/Foo.java")));
        var reviewer = storedBatches.get(0).stream()
            .filter(i -> "reviewer".equals(i.attributes().get(DevtownMemoryKeys.ENTITY_TYPE)))
            .findFirst().orElseThrow();
        assertThat(reviewer.entityId()).isEqualTo("reviewer:security-agent-1");
    }

    @Test
    void codeAreaFact_hasModuleLevelEntityId() {
        emitter.onReviewCompleted(event(List.of(
            "app/src/main/java/io/casehub/Foo.java")));
        var area = storedBatches.get(0).stream()
            .filter(i -> "code-area".equals(i.attributes().get(DevtownMemoryKeys.ENTITY_TYPE)))
            .findFirst().orElseThrow();
        assertThat(area.entityId()).isEqualTo("module:casehubio/devtown/app");
    }

    @Test
    void allFacts_haveDomainAndTenant() {
        emitter.onReviewCompleted(event(List.of("app/src/main/java/Foo.java")));
        storedBatches.get(0).forEach(input -> {
            assertThat(input.domain()).isEqualTo(DevtownMemoryDomain.SOFTWARE_REVIEW);
            assertThat(input.tenantId()).isEqualTo("tenant-1");
        });
    }

    @Test
    void allFacts_haveCaseId() {
        var evt = event(List.of("app/src/main/java/Foo.java"));
        emitter.onReviewCompleted(evt);
        storedBatches.get(0).forEach(input ->
            assertThat(input.caseId()).isEqualTo(evt.caseId().toString()));
    }

    @Test
    void allFacts_usePlatformReservedKeys() {
        emitter.onReviewCompleted(event(List.of("app/src/main/java/Foo.java")));
        storedBatches.get(0).forEach(input -> {
            assertThat(input.attributes()).containsKey(MemoryAttributeKeys.OUTCOME);
            assertThat(input.attributes()).containsKey(MemoryAttributeKeys.ACTOR_ID);
            assertThat(input.attributes().get(MemoryAttributeKeys.OUTCOME))
                .isEqualTo("COMPLETED");
        });
    }

    @Test
    void multipleModules_deduplicatedInBatch() {
        emitter.onReviewCompleted(event(List.of(
            "app/src/main/java/Foo.java",
            "app/src/main/java/Bar.java",
            "domain/src/main/java/Baz.java"
        )));
        var areas = storedBatches.get(0).stream()
            .filter(i -> "code-area".equals(i.attributes().get(DevtownMemoryKeys.ENTITY_TYPE)))
            .toList();
        assertThat(areas).hasSize(2); // app + domain, not 3
    }

    @Test
    void textIsNaturalLanguage_notStructuredLog() {
        emitter.onReviewCompleted(event(List.of("app/src/main/java/Foo.java")));
        storedBatches.get(0).forEach(input ->
            assertThat(input.text())
                .doesNotContain("=")
                .doesNotContain("{")
                .contains(" "));
    }

    @Test
    void codeAreaText_doesNotContainContributorLogin() {
        emitter.onReviewCompleted(event(List.of("app/src/main/java/Foo.java")));
        var area = storedBatches.get(0).stream()
            .filter(i -> "code-area".equals(i.attributes().get(DevtownMemoryKeys.ENTITY_TYPE)))
            .findFirst().orElseThrow();
        assertThat(area.text()).doesNotContain("mdproctor");
    }

    @Test
    void storeFailure_swallowed() {
        CaseMemoryStore failingStore = mock(CaseMemoryStore.class);
        when(failingStore.storeAll(any())).thenThrow(new RuntimeException("store down"));
        @SuppressWarnings("unchecked")
        Instance<CaseMemoryStore> instance = mock(Instance.class);
        when(instance.isResolvable()).thenReturn(true);
        when(instance.get()).thenReturn(failingStore);
        emitter.store = instance;

        // Should not throw
        emitter.onReviewCompleted(event(List.of("app/src/main/java/Foo.java")));
    }

    @Test
    void storeNotResolvable_skips() {
        @SuppressWarnings("unchecked")
        Instance<CaseMemoryStore> instance = mock(Instance.class);
        when(instance.isResolvable()).thenReturn(false);
        emitter.store = instance;

        emitter.onReviewCompleted(event(List.of("app/src/main/java/Foo.java")));
        assertThat(storedBatches).isEmpty();
    }
}
```

- [ ] **Step 2: Add mockito-core test dependency to app pom.xml**

Check if mockito is already available. If not, add to `app/pom.xml`:

```xml
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-core</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 3: Write CaseMemoryEmitter**

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.memory.DevtownMemoryDomain;
import io.casehub.devtown.domain.memory.DevtownMemoryKeys;
import io.casehub.devtown.domain.memory.ModulePathNormalizer;
import io.casehub.devtown.review.ReviewCompletedEvent;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryAttributeKeys;
import io.casehub.platform.api.memory.MemoryInput;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import org.jboss.logging.Logger;

@ApplicationScoped
public class CaseMemoryEmitter {

    private static final Logger LOG = Logger.getLogger(CaseMemoryEmitter.class);

    @Inject Instance<CaseMemoryStore> store;

    void onReviewCompleted(@ObservesAsync ReviewCompletedEvent event) {
        if (!store.isResolvable()) return;
        var s = store.get();
        try {
            s.storeAll(buildFacts(event));
        } catch (Exception e) {
            LOG.warnf(e, "Memory emission failed for case=%s capability=%s",
                event.caseId(), event.capability());
        } finally {
            store.destroy(s);
        }
    }

    private List<MemoryInput> buildFacts(ReviewCompletedEvent event) {
        var facts = new ArrayList<MemoryInput>();
        var pr = event.pr();
        String caseId = event.caseId().toString();

        facts.add(new MemoryInput(
            "contributor:" + pr.contributor(),
            DevtownMemoryDomain.SOFTWARE_REVIEW,
            event.tenantId(), caseId,
            contributorText(event),
            sharedAttributes(event, "contributor", pr.contributor())));

        facts.add(new MemoryInput(
            "reviewer:" + event.reviewerId(),
            DevtownMemoryDomain.SOFTWARE_REVIEW,
            event.tenantId(), caseId,
            reviewerText(event),
            sharedAttributes(event, "reviewer", event.reviewerId())));

        for (String module : ModulePathNormalizer.normalize(pr.changedPaths())) {
            facts.add(new MemoryInput(
                "module:" + pr.repo() + "/" + module,
                DevtownMemoryDomain.SOFTWARE_REVIEW,
                event.tenantId(), caseId,
                codeAreaText(event, module),
                sharedAttributes(event, "code-area", event.reviewerId())));
        }
        return facts;
    }

    private Map<String, String> sharedAttributes(ReviewCompletedEvent event,
                                                  String entityType, String actorId) {
        var attrs = new LinkedHashMap<String, String>();
        attrs.put(MemoryAttributeKeys.ACTOR_ID, actorId);
        attrs.put(MemoryAttributeKeys.ACTOR_ROLE, "contributor".equals(entityType) ? "contributor" : "reviewer");
        attrs.put(MemoryAttributeKeys.OUTCOME, event.outcome().name());
        attrs.put(DevtownMemoryKeys.CAPABILITY, event.capability());
        attrs.put(DevtownMemoryKeys.PR_NUMBER, String.valueOf(event.pr().prNumber()));
        attrs.put(DevtownMemoryKeys.PR_REPO, event.pr().repo());
        attrs.put(DevtownMemoryKeys.LINES_CHANGED, String.valueOf(event.pr().linesChanged()));
        attrs.put(DevtownMemoryKeys.ENTITY_TYPE, entityType);
        if (event.outcomeDetail() != null) {
            attrs.put(DevtownMemoryKeys.OUTCOME_DETAIL, event.outcomeDetail());
        }
        return attrs;
    }

    private String contributorText(ReviewCompletedEvent event) {
        String capName = event.capability().replace("-", " ");
        String detail = outcomePhrase(event);
        return String.format("%s of a %d-line pull request by %s in %s %s.",
            capitalize(capName), event.pr().linesChanged(), event.pr().contributor(),
            event.pr().repo(), detail);
    }

    private String reviewerText(ReviewCompletedEvent event) {
        String detail = outcomePhrase(event);
        return String.format("%s completed a %s of pull request #%d in %s. %s.",
            event.reviewerId(), event.capability().replace("-", " "),
            event.pr().prNumber(), event.pr().repo(), capitalize(detail.trim()));
    }

    private String codeAreaText(ReviewCompletedEvent event, String module) {
        String detail = outcomePhrase(event);
        return String.format("The %s module in %s received a %s on pull request #%d. %s.",
            module, event.pr().repo(), event.capability().replace("-", " "),
            event.pr().prNumber(), capitalize(detail.trim()));
    }

    private String outcomePhrase(ReviewCompletedEvent event) {
        if (event.outcomeDetail() == null) return "outcome " + event.outcome().name().toLowerCase();
        String d = event.outcomeDetail();
        if ("APPROVED".equalsIgnoreCase(d) || "passed".equalsIgnoreCase(d)) return "no issues found";
        return "found issues requiring attention";
    }

    private static String capitalize(String s) {
        if (s == null || s.isEmpty()) return s;
        return Character.toUpperCase(s.charAt(0)) + s.substring(1);
    }
}
```

- [ ] **Step 4: Run CaseMemoryEmitterTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CaseMemoryEmitterTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All 11 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/CaseMemoryEmitter.java app/src/test/java/io/casehub/devtown/app/CaseMemoryEmitterTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#43): add CaseMemoryEmitter — pure function from ReviewCompletedEvent to storeAll

Natural language text for semantic embedding. Platform reserved keys
(OUTCOME, ACTOR_ID) + devtown-specific keys. storeAll batching.
Instance<CaseMemoryStore> pattern. Failure swallowed.

Refs #43"
```

---

### Task 5: ReviewOutcomeObserver — extraction component

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/ReviewOutcomeObserver.java`
- Create: `app/src/test/java/io/casehub/devtown/app/ReviewOutcomeObserverTest.java`

This is the hardest component — the single place that touches engine internals. It observes `PlanItemCompletedEvent`, extracts structured data from the case context, and fires `ReviewCompletedEvent`.

- [ ] **Step 1: Write ReviewOutcomeObserverTest**

A `@QuarkusTest` because it needs the engine running (CaseInstance lookup). The test starts a case, completes a binding via case context signal, and verifies the ReviewCompletedEvent is fired with correct extracted data.

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.memory.ReviewOutcome;
import io.casehub.devtown.review.ReviewCompletedEvent;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.CopyOnWriteArrayList;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@QuarkusTest
class ReviewOutcomeObserverTest {

    @Inject PrReviewCaseHub caseHub;
    @Inject CrossTenantCaseInstanceRepository caseInstanceRepository;
    @Inject ReviewCompletedEventCapture capture;

    private static final Map<String, Object> INITIAL_CTX = Map.of(
        "pr", Map.of("id", "42", "repo", "casehubio/devtown", "linesChanged", 100,
                      "baseRef", "main", "headSha", "abc123",
                      "contributor", "testuser", "changedPaths", java.util.List.of("app/src/main/java/Foo.java")),
        "policy", Map.of("humanApprovalThreshold", 500,
                         "securityReviewRequired", false, "requireSeniorApproval", false),
        "codeAnalysis", Map.of("complete", true, "securitySensitive", false, "architectureCrossing", false),
        "styleCheck", Map.of("outcome", "PENDING"),
        "testCoverage", Map.of("outcome", "PENDING"),
        "performanceAnalysis", Map.of("outcome", "PENDING"),
        "ci", Map.of("status", "passing"));

    @Test
    void planItemCompleted_firesReviewCompletedEvent() throws Exception {
        var caseId = caseHub.startCase(INITIAL_CTX).toCompletableFuture().get(5, SECONDS);

        // Signal a binding completion that writes to case context
        caseHub.signal(caseId, "styleCheck", Map.of("outcome", "APPROVED"));

        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() ->
            assertThat(capture.events.stream()
                .filter(e -> e.caseId().equals(caseId) && "style-review".equals(e.capability()))
                .toList()).isNotEmpty());

        var event = capture.events.stream()
            .filter(e -> e.caseId().equals(caseId) && "style-review".equals(e.capability()))
            .findFirst().orElseThrow();

        assertThat(event.outcome()).isEqualTo(ReviewOutcome.COMPLETED);
        assertThat(event.outcomeDetail()).isEqualTo("APPROVED");
        assertThat(event.tenantId()).isNotNull();
        assertThat(event.pr().contributor()).isEqualTo("testuser");
    }

    @Test
    void infrastructureBinding_doesNotFireEvent() throws Exception {
        var caseId = caseHub.startCase(INITIAL_CTX).toCompletableFuture().get(5, SECONDS);

        // initial-analysis and run-ci are infrastructure — should not fire ReviewCompletedEvent
        long countBefore = capture.events.stream()
            .filter(e -> e.caseId().equals(caseId))
            .count();

        // Wait a moment to confirm no event fires for infrastructure bindings
        Thread.sleep(500);

        // Any ReviewCompletedEvents for this case should only be for review bindings
        capture.events.stream()
            .filter(e -> e.caseId().equals(caseId))
            .forEach(e -> assertThat(e.capability())
                .isNotIn("code-analysis", "ci-runner", "merge-executor"));
    }

    @ApplicationScoped
    public static class ReviewCompletedEventCapture {
        final CopyOnWriteArrayList<ReviewCompletedEvent> events = new CopyOnWriteArrayList<>();

        void onEvent(@ObservesAsync ReviewCompletedEvent event) {
            events.add(event);
        }
    }
}
```

Note: This test depends on the engine's PlanItemCompletedEvent actually firing when a case context signal resolves a binding. The exact assertion may need adjustment during implementation based on how the engine's blackboard dispatches plan items for the style-check binding. If the binding fires via a ScheduleHandler that then marks the PlanItem as completed, the PlanItemCompletedEvent will follow. The test verifies the end-to-end chain.

- [ ] **Step 2: Write ReviewOutcomeObserver**

```java
package io.casehub.devtown.app;

import io.casehub.api.context.CaseContext;
import io.casehub.blackboard.event.PlanItemCompletedEvent;
import io.casehub.devtown.domain.memory.ReviewOutcome;
import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.ReviewCompletedEvent;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import org.jboss.logging.Logger;

@ApplicationScoped
public class ReviewOutcomeObserver {

    private static final Logger LOG = Logger.getLogger(ReviewOutcomeObserver.class);

    private static final Map<String, String> PLAN_ITEM_TO_CONTEXT_KEY = Map.of(
        "security-review",       "securityReview.outcome",
        "architecture-review",   "architectureReview.outcome",
        "style-check",           "styleCheck.outcome",
        "test-coverage",         "testCoverage.outcome",
        "performance-analysis",  "performanceAnalysis.outcome",
        "human-approval",        "humanApproval.status"
    );

    @Inject CrossTenantCaseInstanceRepository caseRepo;
    @Inject Event<ReviewCompletedEvent> reviewCompletedEvents;

    void onPlanItemCompleted(@ObservesAsync PlanItemCompletedEvent event) {
        String contextKeyPath = PLAN_ITEM_TO_CONTEXT_KEY.get(event.planItemId());
        if (contextKeyPath == null) return;

        try {
            var instance = caseRepo.findByUuid(event.caseId())
                .await().atMost(Duration.ofSeconds(5));
            if (instance == null) {
                LOG.warnf("CaseInstance not found for caseId=%s — skipping memory emission",
                    event.caseId());
                return;
            }

            CaseContext ctx = instance.getCaseContext();

            String outcomeDetail = ctx.getPathAsString(contextKeyPath);
            PrPayload pr = extractPrPayload(ctx);
            if (pr == null) {
                LOG.warnf("PR metadata missing in case context for caseId=%s", event.caseId());
                return;
            }

            String capability = resolveCapability(event.planItemId());
            String reviewerId = event.trackingKey() != null ? event.trackingKey() : "unknown";

            reviewCompletedEvents.fireAsync(new ReviewCompletedEvent(
                event.caseId(),
                instance.getTenancyId(),
                capability,
                reviewerId,
                ReviewOutcome.COMPLETED,
                outcomeDetail,
                pr
            ));
        } catch (Exception e) {
            LOG.warnf(e, "ReviewOutcomeObserver failed for caseId=%s planItemId=%s",
                event.caseId(), event.planItemId());
        }
    }

    @SuppressWarnings("unchecked")
    private PrPayload extractPrPayload(CaseContext ctx) {
        Object prObj = ctx.get("pr");
        if (!(prObj instanceof Map<?, ?> prMap)) return null;

        try {
            String repo = (String) prMap.get("repo");
            int prNumber = toInt(prMap.get("id"));
            String headSha = (String) prMap.get("headSha");
            String baseRef = (String) prMap.get("baseRef");
            int linesChanged = toInt(prMap.get("linesChanged"));
            String contributor = (String) prMap.get("contributor");
            List<String> changedPaths = prMap.get("changedPaths") instanceof List<?> list
                ? list.stream().map(Object::toString).toList()
                : List.of();

            if (repo == null || headSha == null || baseRef == null || contributor == null) return null;
            return new PrPayload(repo, prNumber, headSha, baseRef, linesChanged,
                contributor, changedPaths);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to extract PrPayload from case context");
            return null;
        }
    }

    private static String resolveCapability(String planItemId) {
        return switch (planItemId) {
            case "style-check" -> "style-review";
            case "human-approval" -> "human-decision:pr-approval";
            default -> planItemId;
        };
    }

    private static int toInt(Object o) {
        if (o instanceof Number n) return n.intValue();
        if (o instanceof String s) return Integer.parseInt(s);
        return 0;
    }
}
```

- [ ] **Step 3: Run ReviewOutcomeObserverTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ReviewOutcomeObserverTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: Tests PASS (or adjust assertions based on actual engine PlanItem behavior)

- [ ] **Step 4: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/ReviewOutcomeObserver.java app/src/test/java/io/casehub/devtown/app/ReviewOutcomeObserverTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#43): add ReviewOutcomeObserver — extraction from engine events

Observes PlanItemCompletedEvent, extracts reviewer outcome from case
context via hard-coded planItemId→contextKey map, fires typed
ReviewCompletedEvent. CrossTenantCaseInstanceRepository used (accepted
tech debt — follow-up: add tenantId to PlanItemCompletedEvent).

Refs #43"
```

---

### Task 6: CaseMemoryRecaller — recall before case open

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/CaseMemoryRecaller.java`
- Create: `app/src/test/java/io/casehub/devtown/app/CaseMemoryRecallerTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

- [ ] **Step 1: Write CaseMemoryRecallerTest**

A `@QuarkusTest` because it uses `CaseMemoryStore` and `CurrentPrincipal`.

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.memory.DevtownMemoryDomain;
import io.casehub.devtown.domain.memory.DevtownMemoryKeys;
import io.casehub.devtown.domain.memory.ReviewOutcome;
import io.casehub.devtown.review.MemoryContext;
import io.casehub.devtown.review.PrPayload;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryAttributeKeys;
import io.casehub.platform.api.memory.MemoryInput;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class CaseMemoryRecallerTest {

    @Inject CaseMemoryRecaller recaller;
    @Inject CaseMemoryStore store;

    private PrPayload prPayload() {
        return new PrPayload("casehubio/devtown", 99, "sha", "main", 200,
            "testuser", List.of("app/src/main/java/Foo.java"));
    }

    @Test
    void recall_emptyStore_returnsEmpty() {
        var result = recaller.recall(prPayload());
        assertThat(result.contributorHistory()).isEmpty();
        assertThat(result.codeAreaHistory()).isEmpty();
    }

    @Test
    void recall_withContributorFacts_returnsThem() {
        store.store(new MemoryInput(
            "contributor:testuser",
            DevtownMemoryDomain.SOFTWARE_REVIEW,
            "default", null,
            "Security review of PR by testuser found issues.",
            Map.of(MemoryAttributeKeys.OUTCOME, ReviewOutcome.COMPLETED.name(),
                   DevtownMemoryKeys.CAPABILITY, "security-review",
                   DevtownMemoryKeys.ENTITY_TYPE, "contributor")));

        var result = recaller.recall(prPayload());
        assertThat(result.contributorHistory()).isNotEmpty();
        assertThat(result.contributorHistory().get(0).entityId())
            .isEqualTo("contributor:testuser");
    }

    @Test
    void recall_withCodeAreaFacts_returnsThem() {
        store.store(new MemoryInput(
            "module:casehubio/devtown/app",
            DevtownMemoryDomain.SOFTWARE_REVIEW,
            "default", null,
            "The app module received a security review.",
            Map.of(MemoryAttributeKeys.OUTCOME, ReviewOutcome.COMPLETED.name(),
                   DevtownMemoryKeys.CAPABILITY, "security-review",
                   DevtownMemoryKeys.ENTITY_TYPE, "code-area")));

        var result = recaller.recall(prPayload());
        assertThat(result.codeAreaHistory()).isNotEmpty();
    }
}
```

Note: The test depends on having a `CaseMemoryStore` bean that actually stores things. If only the no-op is on the classpath, this test would always get empty results. A test-scoped in-memory implementation or the `memory-inmem` module dependency may be needed. Adjust during implementation based on what's available in test classpath.

- [ ] **Step 2: Write CaseMemoryRecaller**

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.memory.DevtownMemoryDomain;
import io.casehub.devtown.domain.memory.ModulePathNormalizer;
import io.casehub.devtown.review.MemoryContext;
import io.casehub.devtown.review.PrPayload;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.Memory;
import io.casehub.platform.api.memory.MemoryOrder;
import io.casehub.platform.api.memory.MemoryQuery;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import org.jboss.logging.Logger;

@ApplicationScoped
public class CaseMemoryRecaller {

    private static final Logger LOG = Logger.getLogger(CaseMemoryRecaller.class);

    @Inject Instance<CaseMemoryStore> store;
    @Inject CurrentPrincipal principal;

    public MemoryContext recall(PrPayload pr) {
        if (!store.isResolvable()) return MemoryContext.EMPTY;
        var s = store.get();
        try {
            String tenantId = principal.tenancyId();
            Instant since = Instant.now().minus(Duration.ofDays(90));

            List<Memory> contributorHistory = s.query(
                MemoryQuery.forEntity(
                    "contributor:" + pr.contributor(),
                    DevtownMemoryDomain.SOFTWARE_REVIEW, tenantId)
                .withLimit(10)
                .withSince(since)
                .withOrder(MemoryOrder.CHRONOLOGICAL));

            List<String> moduleIds = ModulePathNormalizer.normalize(pr.changedPaths()).stream()
                .map(m -> "module:" + pr.repo() + "/" + m)
                .limit(MemoryQuery.MAX_ENTITY_IDS)
                .toList();

            List<Memory> codeAreaHistory = moduleIds.isEmpty()
                ? List.of()
                : s.query(
                    MemoryQuery.forEntities(moduleIds,
                        DevtownMemoryDomain.SOFTWARE_REVIEW, tenantId)
                    .withLimit(15)
                    .withSince(since)
                    .withOrder(MemoryOrder.RELEVANCE)
                    .withQuestion("review history for "
                        + String.join(", ", ModulePathNormalizer.normalize(pr.changedPaths()))
                        + " in " + pr.repo()));

            return new MemoryContext(contributorHistory, codeAreaHistory);
        } catch (Exception e) {
            LOG.warnf(e, "Memory recall failed for contributor=%s — proceeding without memory",
                pr.contributor());
            return MemoryContext.EMPTY;
        } finally {
            store.destroy(s);
        }
    }
}
```

- [ ] **Step 3: Wire recall into PrReviewCaseService**

Modify `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`:

```java
@Inject CaseMemoryRecaller memoryRecaller;

@Override
public PrReviewOutcome review(PrPayload pr) {
    var policy = Map.<String, Object>of(
        "humanApprovalThreshold", humanApprovalThreshold,
        "securityReviewRequired", securityReviewRequired,
        "requireSeniorApproval", requireSeniorApproval);
    var prContext = Map.<String, Object>of(
        "id", String.valueOf(pr.prNumber()),
        "repo", pr.repo(),
        "linesChanged", pr.linesChanged(),
        "baseRef", pr.baseRef(),
        "headSha", pr.headSha(),
        "contributor", pr.contributor(),
        "changedPaths", pr.changedPaths());

    var memoryContext = memoryRecaller.recall(pr);

    var initialContext = new java.util.HashMap<String, Object>();
    initialContext.put("pr", prContext);
    initialContext.put("policy", policy);
    initialContext.put("memory", memoryContext.toContextMap());

    caseHub.startCase(initialContext);
    return new PrReviewOutcome(VERDICT_CASE_OPENED, List.of());
}
```

Note: `Map.of()` doesn't allow null values, and `memoryContext.toContextMap()` won't be null, so `HashMap` is used for the outer map to allow flexibility. Alternatively, keep `Map.of` if all values are guaranteed non-null.

- [ ] **Step 4: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/CaseMemoryRecaller.java app/src/test/java/io/casehub/devtown/app/CaseMemoryRecallerTest.java app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#43): add CaseMemoryRecaller and wire into PrReviewCaseService

Queries contributor history (10 facts, 90 days, chronological) and
code area history (15 facts, 90 days, relevance-ranked) before case
open. Memory context injected as initial case context key.
Fail-open: query failure returns MemoryContext.EMPTY.

Refs #43"
```

---

### Task 7: Integration Test — full round-trip

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/CaseMemoryIntegrationTest.java`

This test verifies the complete emission → recall chain: start a case, a binding completes, the observer extracts and fires ReviewCompletedEvent, the emitter stores facts, a second case for the same contributor recalls the stored facts.

- [ ] **Step 1: Write CaseMemoryIntegrationTest**

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.memory.DevtownMemoryDomain;
import io.casehub.devtown.review.MemoryContext;
import io.casehub.devtown.review.PrPayload;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryOrder;
import io.casehub.platform.api.memory.MemoryQuery;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@QuarkusTest
class CaseMemoryIntegrationTest {

    @Inject PrReviewCaseHub caseHub;
    @Inject CaseMemoryStore store;
    @Inject CaseMemoryRecaller recaller;

    @Test
    void fullRoundTrip_emitThenRecall() throws Exception {
        // ── Phase 1: Start case with a binding that will complete ──────
        var initialCtx = Map.<String, Object>of(
            "pr", Map.of("id", "900", "repo", "casehubio/devtown", "linesChanged", 100,
                          "baseRef", "main", "headSha", "intsha",
                          "contributor", "roundtrip-user",
                          "changedPaths", List.of("app/src/main/java/Foo.java")),
            "policy", Map.of("humanApprovalThreshold", 500,
                             "securityReviewRequired", false, "requireSeniorApproval", false),
            "codeAnalysis", Map.of("complete", true, "securitySensitive", false,
                                    "architectureCrossing", false),
            "styleCheck", Map.of("outcome", "PENDING"),
            "testCoverage", Map.of("outcome", "PENDING"),
            "performanceAnalysis", Map.of("outcome", "PENDING"),
            "ci", Map.of("status", "passing"));

        var caseId = caseHub.startCase(initialCtx).toCompletableFuture().get(5, SECONDS);

        // ── Phase 2: Complete a review binding ────────────────────────
        caseHub.signal(caseId, "styleCheck", Map.of("outcome", "APPROVED"));

        // Wait for the async emission chain to complete
        await().atMost(10, SECONDS).pollInterval(200, MILLISECONDS).untilAsserted(() -> {
            var memories = store.query(
                MemoryQuery.forEntity("contributor:roundtrip-user",
                    DevtownMemoryDomain.SOFTWARE_REVIEW, "default")
                .withLimit(5)
                .withOrder(MemoryOrder.CHRONOLOGICAL));
            assertThat(memories).as("contributor memories from emission").isNotEmpty();
        });

        // ── Phase 3: Recall for a second PR by the same contributor ──
        var pr2 = new PrPayload("casehubio/devtown", 901, "sha2", "main", 200,
            "roundtrip-user", List.of("app/src/main/java/Bar.java"));
        MemoryContext recalled = recaller.recall(pr2);

        assertThat(recalled.contributorHistory())
            .as("recall returns facts from Phase 2 emission")
            .isNotEmpty();
        assertThat(recalled.contributorHistory().get(0).entityId())
            .isEqualTo("contributor:roundtrip-user");
    }
}
```

Note: This test depends on the full async chain working end-to-end. If the engine's PlanItemCompletedEvent doesn't fire for context signal-driven binding completions, the test will need adjustment — potentially signaling in a way that triggers the PlanItem completion chain. Adjust based on actual engine behavior discovered in Task 5.

- [ ] **Step 2: Run the integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CaseMemoryIntegrationTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: All tests PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/CaseMemoryIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test(#43): add CaseMemoryIntegrationTest — full round-trip emission and recall

Start case → binding completes → observer extracts → emitter stores →
second case recalls contributor history from first case.

Refs #43"
```

---

### Task 8: File Follow-up Issues

Create the follow-up GitHub issues documented in the spec.

- [ ] **Step 1: File engine issues**

```bash
gh issue create --repo casehubio/engine --title "feat: promote PlanItemCompletedEvent to engine-common-spi" \
  --body "PlanItemCompletedEvent is in io.casehub.blackboard.event — not public SPI. devtown#43 (CaseMemoryStore) observes it from an application module. Promote to engine-common-spi as a stable extension point."

gh issue create --repo casehubio/engine --title "feat: add tenantId to PlanItemCompletedEvent" \
  --body "devtown#43 uses CrossTenantCaseInstanceRepository to look up CaseInstance in an async observer because tenantId is unavailable. Adding tenantId to PlanItemCompletedEvent eliminates the cross-tenant lookup. Refs devtown#43."

gh issue create --repo casehubio/engine --title "feat: add CDI event for PlanItem fault/rejection" \
  --body "PlanItemCompletedEvent only fires on COMPLETED. Faulted/rejected PlanItems emit no CDI event. Needed for FAILED outcome emission in devtown CaseMemoryStore (devtown#43). Refs devtown#43."
```

- [ ] **Step 2: File qhorus issue**

```bash
gh issue create --repo casehubio/qhorus --title "feat: add CDI event for commitment DECLINED" \
  --body "devtown#43 needs to observe DECLINED outcomes for CaseMemoryStore emission. Currently no CDI event is fired when a commitment transitions to DECLINED. Refs devtown#43."
```

- [ ] **Step 3: File devtown follow-up issues**

```bash
gh issue create --repo casehubio/devtown --title "feat: make memory recall limits configurable via PreferenceKey" \
  --body "Currently hard-coded: contributor limit=10, code area limit=15, time window=90 days. Make configurable via PreferenceKey definitions. Low priority — current defaults are reasonable. Refs #43."

gh issue create --repo casehubio/devtown --title "feat: GDPR erasure endpoint for contributor memory" \
  --body "CaseMemoryStore.eraseEntity() is the mechanism for GDPR Art.17 deletion. Need an admin endpoint or scheduled job to handle requests. Documented in devtown#43 spec. Refs #43."
```

- [ ] **Step 4: Commit (no code changes — just document that issues were filed)**

No commit needed — issues are in GitHub, not in the repo.

---

## Execution Notes

**Task dependencies:** Sequential — each task builds on the previous.

**Test infrastructure:** The `CaseMemoryRecallerTest` and `CaseMemoryIntegrationTest` need a `CaseMemoryStore` that actually stores facts (not the no-op). Options:
1. Add `casehub-platform-memory-inmem` as a test-scoped dependency in `app/pom.xml`
2. Create a minimal test-scoped `@ApplicationScoped @Alternative` in-memory store in test sources

Evaluate during Task 6 implementation. The `InMemoryMemoryStore` from `memory-inmem` uses `@Alternative @Priority(10)` and requires `CurrentPrincipal` — which works in `@QuarkusTest` request scope (the recaller test runs in a REST context). For the emitter's async path, the no-op store is fine since `CaseMemoryEmitterTest` is a plain unit test with mocks.

**PlanItemCompletedEvent binding:** The hardest integration question is whether the engine fires `PlanItemCompletedEvent` when a case context signal resolves a binding's `when` condition and the binding's ScheduleHandler completes. If not, the `ReviewOutcomeObserverTest` will fail and the observation strategy needs adjustment. Investigate during Task 5 implementation.
