# LLM-Powered Reviewer Agents Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #203 — LLM-powered reviewer agents — augment stub implementations with real code analysis
**Issue group:** #203

**Goal:** Augment devtown's 6 reviewer agent stubs with LLM-powered implementations that produce structured `ReviewFinding` records from actual PR diffs, wired into both Layer 3 and Layer 5 dispatch paths.

**Architecture:** `ReviewerAgent` port interface gains `priority()` and `handle(ReviewContext)`. A `ReviewerAgentRegistry` indexes agents by capability, selecting the highest-priority per capability. LLM agents use `StructuredAgentInvoker` from casehub-blocks to invoke `AgentProvider`, batch per-file diffs, parse JSON findings. A separate `CodeAnalysisAgent` port handles PR classification. Function workers in `PrReviewCaseHub.augment()` bridge engine dispatch to the registry. Eidos descriptors declare agent identity with `modelRef` aliases resolved via `agent-config.yaml`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform AgentProvider, casehub-eidos, casehub-blocks StructuredAgentInvoker, Caffeine cache, casehub-platform simulation framework (testing)

## Global Constraints

- Java 21 source level on Java 26 JVM
- All new types in `domain/` are pure Java — no Quarkus/CDI deps
- All new types in `review/` may depend on platform-api and eidos-api (compile scope)
- CDI wiring only in `app/`
- No `@DefaultBean` on ReviewerAgent stubs — priority-based registry displacement
- `ReviewFinding.confidence` clamped to [0.0, 1.0] in compact constructor
- `ReviewFindings.findings` normalised null → empty list in compact constructor
- `FileDiff.patch` is nullable — filter with `hasReviewablePatch()` before batching
- Use `ide_insert_member` / `ide_replace_member` / `ide_edit_member` for code edits
- Use `ide_refactor_rename` for renames, `ide_move_file` for moves
- Use `Simulation.forTest()` for unit tests, `simulation.yaml` for integration tests

---

## Batch 1: Domain Model and Port Interfaces

### Task 1: ReviewFinding domain type + ReviewerOutcome breaking change

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/ReviewFinding.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/ReviewFindingTest.java`
- Modify: `review/src/main/java/io/casehub/devtown/review/ReviewerOutcome.java:8` — change `List<String>` to `List<ReviewFinding>`
- Modify: `review/src/main/java/io/casehub/devtown/review/PrReviewOutcome.java:6` — change `List<String>` to `List<ReviewFinding>`
- Modify: `app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java:78,97,106` — update types and message formatting
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/SecurityReviewAgent.java:20` — return `ReviewFinding` list
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/ArchitectureReviewAgent.java:18` — update return
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/TestCoverageReviewAgent.java:19` — update return
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/PerformanceAnalysisAgent.java:18` — update return
- Modify: `domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java` — add `FINDINGS_CAPABILITIES` constant

**Interfaces:**
- Produces: `ReviewFinding(Severity, String, String, LineRange, String, double)`, `ReviewFinding.Severity` enum, `ReviewFinding.LineRange(int, int)`, `ReviewDomain.FINDINGS_CAPABILITIES`

- [ ] **Step 1: Write ReviewFinding test**

```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ReviewFindingTest {

    @Test
    void confidenceClampedToUnitRange() {
        var finding = new ReviewFinding(
            ReviewFinding.Severity.HIGH, "injection", "src/Api.java",
            new ReviewFinding.LineRange(42, 45), "SQL injection", 1.5);
        assertEquals(1.0, finding.confidence());

        var low = new ReviewFinding(
            ReviewFinding.Severity.LOW, "naming", "src/Util.java",
            null, "inconsistent naming", -0.1);
        assertEquals(0.0, low.confidence());
    }

    @Test
    void lineRangeNullableForFileLevelFindings() {
        var finding = new ReviewFinding(
            ReviewFinding.Severity.INFO, "complexity", "src/Big.java",
            null, "file exceeds 500 lines", 0.7);
        assertNull(finding.lineRange());
    }

    @Test
    void normalConfidencePreserved() {
        var finding = new ReviewFinding(
            ReviewFinding.Severity.MEDIUM, "race-condition", "src/Cache.java",
            new ReviewFinding.LineRange(10, 20), "concurrent access", 0.85);
        assertEquals(0.85, finding.confidence(), 0.001);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ReviewFindingTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `ReviewFinding` does not exist

- [ ] **Step 3: Create ReviewFinding**

Use `ide_create_file` to create `domain/src/main/java/io/casehub/devtown/domain/ReviewFinding.java`:

```java
package io.casehub.devtown.domain;

import org.jspecify.annotations.Nullable;

public record ReviewFinding(
    Severity severity,
    String category,
    String filePath,
    @Nullable LineRange lineRange,
    String message,
    double confidence
) {
    public ReviewFinding {
        confidence = Math.max(0.0, Math.min(1.0, confidence));
    }

    public enum Severity { CRITICAL, HIGH, MEDIUM, LOW, INFO }
    public record LineRange(int startLine, int endLine) {}
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ReviewFindingTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 5: Add FINDINGS_CAPABILITIES to ReviewDomain**

Use `ide_insert_member` to add to `ReviewDomain`:

```java
public static final Set<String> FINDINGS_CAPABILITIES = Set.of(
    SECURITY_REVIEW, ARCHITECTURE_REVIEW,
    STYLE_REVIEW, TEST_COVERAGE, PERFORMANCE_ANALYSIS
);
```

- [ ] **Step 6: Update ReviewerOutcome.Completed**

Use `ide_edit_member` on `ReviewerOutcome` to change `Completed`:

```java
record Completed(List<ReviewFinding> findings) implements ReviewerOutcome {}
```

Add import: `import io.casehub.devtown.domain.ReviewFinding;`

- [ ] **Step 7: Update PrReviewOutcome**

Use `ide_edit_member` to change `PrReviewOutcome`:

```java
public record PrReviewOutcome(String verdict, List<ReviewFinding> findings, UUID caseId) {}
```

Add import: `import io.casehub.devtown.domain.ReviewFinding;`

- [ ] **Step 8: Update all stub agents**

Use `ide_replace_member` on each stub's `handle()` method to return `ReviewFinding` lists:

**SecurityReviewAgent:**
```java
return new ReviewerOutcome.Completed(List.of(
    new ReviewFinding(ReviewFinding.Severity.MEDIUM, "rate-limiting",
        "src/PaymentController.java", null,
        "rate-limiting absent on /payment", 0.8)));
```

**ArchitectureReviewAgent:**
```java
return new ReviewerOutcome.Declined("distributed transaction outside scope");
```
(No change needed — Declined has no type parameter change)

**TestCoverageReviewAgent:**
```java
return new ReviewerOutcome.Completed(List.of(
    new ReviewFinding(ReviewFinding.Severity.MEDIUM, "coverage",
        "src/PaymentService.java", null,
        "coverage 67%; payment path untested", 0.7)));
```

**PerformanceAnalysisAgent:**
```java
return new ReviewerOutcome.Failed("analysis timed out on large diff");
```
(No change needed — Failed has no type parameter change)

- [ ] **Step 9: Update QhorusPrReviewService**

Use `ide_edit_member` on `startReview`:
- Line 78: `List<ReviewFinding> allFindings = new ArrayList<>();`
- Line 97: Format findings for qhorus message:
  ```java
  String findingsText = completed.findings().stream()
      .map(f -> f.severity() + ": " + f.message())
      .collect(Collectors.joining("; "));
  ```
- Line 106: No change needed — `addAll` is type-compatible

- [ ] **Step 10: Run full build to verify ripple**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS — all existing tests pass with updated types

- [ ] **Step 11: Verify with ide_diagnostics**

Run: `ide_diagnostics` on all modified files to catch any remaining type errors.

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/ReviewFinding.java domain/src/test/java/io/casehub/devtown/domain/ReviewFindingTest.java review/src/main/java/io/casehub/devtown/review/ReviewerOutcome.java review/src/main/java/io/casehub/devtown/review/PrReviewOutcome.java app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java app/src/main/java/io/casehub/devtown/app/agents/ domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add ReviewFinding domain type, update ReviewerOutcome to structured findings Refs #203"
```

### Task 2: PrDiff model, PrDiffService SPI, ReviewContext, CodeAnalysisAgent, ReviewFindings

**Files:**
- Create: `review/src/main/java/io/casehub/devtown/review/PrDiff.java`
- Create: `review/src/main/java/io/casehub/devtown/review/PrDiffService.java`
- Create: `review/src/main/java/io/casehub/devtown/review/ReviewContext.java`
- Create: `review/src/main/java/io/casehub/devtown/review/CodeAnalysisAgent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/CodeAnalysisResult.java`
- Create: `review/src/main/java/io/casehub/devtown/review/ReviewFindings.java`
- Create: `review/src/test/java/io/casehub/devtown/review/PrDiffTest.java`
- Create: `review/src/test/java/io/casehub/devtown/review/ReviewFindingsTest.java`

**Interfaces:**
- Produces: `PrDiff(String, int, String, String, List<FileDiff>, boolean)`, `PrDiff.FileDiff(String, String, String, int, int)`, `PrDiffService.fetchDiff(String, int)`, `ReviewContext(PrPayload, PrDiff)`, `CodeAnalysisAgent.analyse(ReviewContext)`, `CodeAnalysisResult`, `ReviewFindings(List<ReviewFinding>)`

- [ ] **Step 1: Write PrDiff test**

```java
package io.casehub.devtown.review;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class PrDiffTest {

    @Test
    void fileDiffHasReviewablePatch_nonNullNonBlank() {
        var fd = new PrDiff.FileDiff("src/A.java", "modified", "+ new line", 1, 0);
        assertTrue(fd.hasReviewablePatch());
    }

    @Test
    void fileDiffHasReviewablePatch_nullPatch() {
        var fd = new PrDiff.FileDiff("image.png", "modified", null, 0, 0);
        assertFalse(fd.hasReviewablePatch());
    }

    @Test
    void fileDiffHasReviewablePatch_blankPatch() {
        var fd = new PrDiff.FileDiff("renamed.java", "renamed", "  ", 0, 0);
        assertFalse(fd.hasReviewablePatch());
    }
}
```

- [ ] **Step 2: Write ReviewFindings test**

```java
package io.casehub.devtown.review;

import io.casehub.devtown.domain.ReviewFinding;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class ReviewFindingsTest {

    @Test
    void nullFindingsNormalisedToEmptyList() {
        var rf = new ReviewFindings(null);
        assertNotNull(rf.findings());
        assertTrue(rf.findings().isEmpty());
    }

    @Test
    void findingsListIsImmutable() {
        var finding = new ReviewFinding(
            ReviewFinding.Severity.LOW, "naming", "src/A.java",
            null, "bad name", 0.5);
        var rf = new ReviewFindings(List.of(finding));
        assertThrows(UnsupportedOperationException.class,
            () -> rf.findings().add(finding));
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest="PrDiffTest,ReviewFindingsTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — types don't exist

- [ ] **Step 4: Create all port types**

Use `ide_create_file` for each:

**PrDiff.java:**
```java
package io.casehub.devtown.review;

import org.jspecify.annotations.Nullable;
import java.util.List;

public record PrDiff(
    String repo,
    int prNumber,
    String baseSha,
    String headSha,
    List<FileDiff> files,
    boolean truncated
) {
    public record FileDiff(
        String path,
        String status,
        @Nullable String patch,
        int additions,
        int deletions
    ) {
        public boolean hasReviewablePatch() {
            return patch != null && !patch.isBlank();
        }
    }
}
```

**PrDiffService.java:**
```java
package io.casehub.devtown.review;

public interface PrDiffService {
    PrDiff fetchDiff(String repo, int prNumber);
}
```

**ReviewContext.java:**
```java
package io.casehub.devtown.review;

public record ReviewContext(PrPayload pr, PrDiff diff) {}
```

**CodeAnalysisAgent.java:**
```java
package io.casehub.devtown.review;

public interface CodeAnalysisAgent {
    CodeAnalysisResult analyse(ReviewContext context);
    default int priority() { return 0; }
}
```

**CodeAnalysisResult.java:**
```java
package io.casehub.devtown.review;

import java.util.List;

public record CodeAnalysisResult(
    boolean complete,
    boolean securitySensitive,
    boolean architectureCrossing,
    String scope,
    List<String> flaggedFiles,
    List<String> crossingPoints
) {
    public CodeAnalysisResult {
        complete = true;
        flaggedFiles = flaggedFiles == null ? List.of() : List.copyOf(flaggedFiles);
        crossingPoints = crossingPoints == null ? List.of() : List.copyOf(crossingPoints);
    }
}
```

**ReviewFindings.java:**
```java
package io.casehub.devtown.review;

import io.casehub.devtown.domain.ReviewFinding;
import java.util.List;

public record ReviewFindings(List<ReviewFinding> findings) {
    public ReviewFindings {
        findings = findings == null ? List.of() : List.copyOf(findings);
    }
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest="PrDiffTest,ReviewFindingsTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Verify with ide_diagnostics**

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add review/src/main/java/io/casehub/devtown/review/PrDiff.java review/src/main/java/io/casehub/devtown/review/PrDiffService.java review/src/main/java/io/casehub/devtown/review/ReviewContext.java review/src/main/java/io/casehub/devtown/review/CodeAnalysisAgent.java review/src/main/java/io/casehub/devtown/review/CodeAnalysisResult.java review/src/main/java/io/casehub/devtown/review/ReviewFindings.java review/src/test/java/io/casehub/devtown/review/PrDiffTest.java review/src/test/java/io/casehub/devtown/review/ReviewFindingsTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add PrDiff model, PrDiffService SPI, ReviewContext, CodeAnalysisAgent, ReviewFindings Refs #203"
```

## Batch 2: Registry, Agent Infrastructure, and Stubs

### Task 3: ReviewerAgent interface change + ReviewerAgentRegistry + stub updates

**Files:**
- Modify: `review/src/main/java/io/casehub/devtown/review/ReviewerAgent.java` — add `priority()`, change `handle(PrPayload)` → `handle(ReviewContext)`
- Create: `review/src/main/java/io/casehub/devtown/review/ReviewerAgentRegistry.java`
- Create: `review/src/test/java/io/casehub/devtown/review/ReviewerAgentRegistryTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/SecurityReviewAgent.java` — update `handle()` signature
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/ArchitectureReviewAgent.java` — update `handle()` signature
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/TestCoverageReviewAgent.java` — update `handle()` signature
- Modify: `app/src/main/java/io/casehub/devtown/app/agents/PerformanceAnalysisAgent.java` — update `handle()` signature
- Create: `app/src/main/java/io/casehub/devtown/app/agents/StyleReviewAgent.java` — new stub
- Create: `app/src/main/java/io/casehub/devtown/app/agents/CodeAnalysisAgentStub.java` — new stub

**Interfaces:**
- Consumes: `ReviewContext`, `ReviewerOutcome`, `ReviewFinding`, `CodeAnalysisAgent`, `CodeAnalysisResult`, `ReviewDomain`
- Produces: `ReviewerAgentRegistry.forCapability(String)`, `ReviewerAgentRegistry.all()`, `ReviewerAgent.priority()`, `ReviewerAgent.handle(ReviewContext)`

- [ ] **Step 1: Write ReviewerAgentRegistry test**

```java
package io.casehub.devtown.review;

import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.domain.ReviewFinding;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class ReviewerAgentRegistryTest {

    @Test
    void highestPriorityWinsPerCapability() {
        ReviewerAgent stub = new ReviewerAgent() {
            @Override public String capability() { return ReviewDomain.SECURITY_REVIEW; }
            @Override public ReviewerOutcome handle(ReviewContext ctx) {
                return new ReviewerOutcome.Completed(List.of());
            }
        };

        ReviewerAgent llm = new ReviewerAgent() {
            @Override public String capability() { return ReviewDomain.SECURITY_REVIEW; }
            @Override public ReviewerOutcome handle(ReviewContext ctx) {
                return new ReviewerOutcome.Completed(List.of(
                    new ReviewFinding(ReviewFinding.Severity.HIGH, "injection",
                        "src/A.java", null, "found issue", 0.9)));
            }
            @Override public int priority() { return 1; }
        };

        var registry = ReviewerAgentRegistry.of(List.of(stub, llm));
        var resolved = registry.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertTrue(resolved.isPresent());
        assertEquals(1, resolved.get().priority());
    }

    @Test
    void missingCapabilityReturnsEmpty() {
        var registry = ReviewerAgentRegistry.of(List.of());
        assertTrue(registry.forCapability("nonexistent").isEmpty());
    }

    @Test
    void stubWinsWhenNoLlmAgent() {
        ReviewerAgent stub = new ReviewerAgent() {
            @Override public String capability() { return ReviewDomain.STYLE_REVIEW; }
            @Override public ReviewerOutcome handle(ReviewContext ctx) {
                return new ReviewerOutcome.Completed(List.of());
            }
        };

        var registry = ReviewerAgentRegistry.of(List.of(stub));
        var resolved = registry.forCapability(ReviewDomain.STYLE_REVIEW);
        assertTrue(resolved.isPresent());
        assertEquals(0, resolved.get().priority());
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest=ReviewerAgentRegistryTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL

- [ ] **Step 3: Update ReviewerAgent interface**

Use `ide_replace_member` on `ReviewerAgent`:

```java
public interface ReviewerAgent {
    String capability();
    ReviewerOutcome handle(ReviewContext context);
    default int priority() { return 0; }
}
```

- [ ] **Step 4: Create ReviewerAgentRegistry**

Use `ide_create_file`:

```java
package io.casehub.devtown.review;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

@ApplicationScoped
public class ReviewerAgentRegistry {

    private final Map<String, ReviewerAgent> agentsByCapability;

    @Inject
    public ReviewerAgentRegistry(Instance<ReviewerAgent> agents) {
        agentsByCapability = new HashMap<>();
        for (ReviewerAgent agent : agents) {
            agentsByCapability.merge(agent.capability(), agent,
                (existing, incoming) ->
                    incoming.priority() > existing.priority() ? incoming : existing);
        }
    }

    ReviewerAgentRegistry(Map<String, ReviewerAgent> agents) {
        this.agentsByCapability = new HashMap<>(agents);
    }

    public static ReviewerAgentRegistry of(List<ReviewerAgent> agents) {
        var map = new HashMap<String, ReviewerAgent>();
        for (ReviewerAgent agent : agents) {
            map.merge(agent.capability(), agent,
                (existing, incoming) ->
                    incoming.priority() > existing.priority() ? incoming : existing);
        }
        return new ReviewerAgentRegistry(map);
    }

    public Optional<ReviewerAgent> forCapability(String capability) {
        return Optional.ofNullable(agentsByCapability.get(capability));
    }

    public Collection<ReviewerAgent> all() {
        return agentsByCapability.values();
    }
}
```

- [ ] **Step 5: Run test — verify it passes**

- [ ] **Step 6: Update all existing stub `handle()` signatures**

Use `ide_replace_member` on each stub's `handle` method to change parameter from `PrPayload pr` to `ReviewContext context`:

Each stub replaces `handle(PrPayload pr)` with `handle(ReviewContext context)`. Body unchanged — stubs ignore the input.

- [ ] **Step 7: Create StyleReviewAgent stub**

Use `ide_create_file`:

```java
package io.casehub.devtown.app.agents;

import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.domain.ReviewFinding;
import io.casehub.devtown.review.ReviewContext;
import io.casehub.devtown.review.ReviewerAgent;
import io.casehub.devtown.review.ReviewerOutcome;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@ApplicationScoped
public class StyleReviewAgent implements ReviewerAgent {

    @Override
    public String capability() {
        return ReviewDomain.STYLE_REVIEW;
    }

    @Override
    public ReviewerOutcome handle(ReviewContext context) {
        return new ReviewerOutcome.Completed(List.of(
            new ReviewFinding(ReviewFinding.Severity.LOW, "naming",
                "src/Example.java", null,
                "inconsistent naming convention", 0.6)));
    }
}
```

- [ ] **Step 8: Create CodeAnalysisAgentStub**

Use `ide_create_file`:

```java
package io.casehub.devtown.app.agents;

import io.casehub.devtown.review.CodeAnalysisAgent;
import io.casehub.devtown.review.CodeAnalysisResult;
import io.casehub.devtown.review.ReviewContext;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@ApplicationScoped
public class CodeAnalysisAgentStub implements CodeAnalysisAgent {

    @Override
    public CodeAnalysisResult analyse(ReviewContext context) {
        return new CodeAnalysisResult(true, false, false,
            "unknown", List.of(), List.of());
    }
}
```

- [ ] **Step 9: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add review/src/main/java/io/casehub/devtown/review/ReviewerAgent.java review/src/main/java/io/casehub/devtown/review/ReviewerAgentRegistry.java review/src/test/java/io/casehub/devtown/review/ReviewerAgentRegistryTest.java app/src/main/java/io/casehub/devtown/app/agents/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add ReviewerAgentRegistry with priority-based displacement, update stubs Refs #203"
```

### Task 4: LlmAgentBase, LlmReviewerAgent, PrDiffCache

**Files:**
- Create: `review/src/main/java/io/casehub/devtown/review/LlmAgentBase.java`
- Create: `review/src/main/java/io/casehub/devtown/review/LlmReviewerAgent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/PrDiffCache.java`
- Create: `review/src/test/java/io/casehub/devtown/review/LlmReviewerAgentTest.java`
- Create: `review/src/test/java/io/casehub/devtown/review/PrDiffCacheTest.java`

**Interfaces:**
- Consumes: `ReviewerAgent`, `ReviewContext`, `ReviewFindings`, `ReviewFinding`, `PrDiff`, `PrDiffService`, `StructuredAgentInvoker`, `AgentProvider`, `AgentRegistry` (eidos)
- Produces: `LlmAgentBase.invokeWithRetry()`, `LlmAgentBase.resolveModelRef()`, `LlmReviewerAgent.handle(ReviewContext)`, `LlmReviewerAgent.batchFiles()`, `PrDiffCache.get(String, int, String)`

- [ ] **Step 1: Write LlmReviewerAgent test**

Test the core handle() logic with simulated AgentProvider responses using `Simulation.forTest()`:

```java
package io.casehub.devtown.review;

import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.domain.ReviewFinding;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.eidos.api.AgentRegistry;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

class LlmReviewerAgentTest {

    @Test
    void producesStructuredFindings() {
        var expectedJson = """
            {"findings": [{"severity": "HIGH", "category": "injection",
             "filePath": "src/Api.java",
             "lineRange": {"startLine": 42, "endLine": 45},
             "message": "SQL injection via string concat",
             "confidence": 0.92}]}""";

        // Create a test subclass with a mock AgentProvider
        var agent = createTestAgent(expectedJson);

        var diff = new PrDiff("org/repo", 42, "base", "head",
            List.of(new PrDiff.FileDiff("src/Api.java", "modified",
                "+ String sql = \"SELECT * FROM users WHERE id=\" + id;",
                1, 0)),
            false);
        var pr = new PrPayload("org/repo", 42, "head", "main",
            50, "alice", 1L, List.of("src/Api.java"));
        var context = new ReviewContext(pr, diff);

        var outcome = agent.handle(context);

        assertInstanceOf(ReviewerOutcome.Completed.class, outcome);
        var completed = (ReviewerOutcome.Completed) outcome;
        assertEquals(1, completed.findings().size());
        assertEquals(ReviewFinding.Severity.HIGH,
            completed.findings().get(0).severity());
        assertEquals("src/Api.java",
            completed.findings().get(0).filePath());
    }

    @Test
    void declinesWhenNoReviewableFiles() {
        var agent = createTestAgent("{}");

        var diff = new PrDiff("org/repo", 42, "base", "head",
            List.of(new PrDiff.FileDiff("image.png", "modified",
                null, 0, 0)),
            false);
        var pr = new PrPayload("org/repo", 42, "head", "main",
            50, "alice", 1L, List.of("image.png"));
        var context = new ReviewContext(pr, diff);

        var outcome = agent.handle(context);
        assertInstanceOf(ReviewerOutcome.Declined.class, outcome);
    }

    @Test
    void filtersHallucinatedFilePaths() {
        var json = """
            {"findings": [
              {"severity": "HIGH", "category": "injection",
               "filePath": "src/Api.java", "lineRange": null,
               "message": "real finding", "confidence": 0.9},
              {"severity": "LOW", "category": "naming",
               "filePath": "src/NotInDiff.java", "lineRange": null,
               "message": "hallucinated file", "confidence": 0.5}
            ]}""";

        var agent = createTestAgent(json);
        var diff = new PrDiff("org/repo", 42, "base", "head",
            List.of(new PrDiff.FileDiff("src/Api.java", "modified",
                "+ code", 1, 0)),
            false);
        var pr = new PrPayload("org/repo", 42, "head", "main",
            50, "alice", 1L, List.of("src/Api.java"));
        var context = new ReviewContext(pr, diff);

        var outcome = agent.handle(context);
        assertInstanceOf(ReviewerOutcome.Completed.class, outcome);
        var findings = ((ReviewerOutcome.Completed) outcome).findings();
        assertEquals(1, findings.size());
        assertEquals("src/Api.java", findings.get(0).filePath());
    }

    // Helper: creates a concrete LlmReviewerAgent subclass with mock AgentProvider
    private LlmReviewerAgent createTestAgent(String jsonResponse) {
        // Implementation: anonymous subclass with stubbed agentProvider()
        // that returns a Multi emitting TextDelta(jsonResponse) + InvocationComplete
        // agentRegistry() returns empty (no eidos descriptor = null modelRef = default backend)
        // ... (full implementation in the test file)
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

- [ ] **Step 3: Create LlmAgentBase**

Use `ide_create_file` — abstract base with retry, timeout, model resolution. Full implementation per spec §LlmAgentBase.

- [ ] **Step 4: Create LlmReviewerAgent**

Use `ide_create_file` — extends LlmAgentBase, implements ReviewerAgent. Full implementation per spec §LlmReviewerAgent with batching, file path validation, error isolation.

- [ ] **Step 5: Create PrDiffCache**

Use `ide_create_file`:

```java
package io.casehub.devtown.review;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Duration;

@ApplicationScoped
public class PrDiffCache {
    @Inject PrDiffService diffService;

    private final Cache<CacheKey, PrDiff> cache = Caffeine.newBuilder()
        .maximumSize(100)
        .expireAfterWrite(Duration.ofMinutes(10))
        .build();

    public PrDiff get(String repo, int prNumber, String headSha) {
        return cache.get(new CacheKey(repo, prNumber, headSha),
            k -> diffService.fetchDiff(k.repo(), k.prNumber()));
    }

    record CacheKey(String repo, int prNumber, String headSha) {}
}
```

- [ ] **Step 6: Run tests — verify they pass**

- [ ] **Step 7: Verify with ide_diagnostics, then commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add LlmAgentBase, LlmReviewerAgent, PrDiffCache Refs #203"
```

## Batch 3: GitHub Adapter

### Task 5: GitHubPrDiffClient + PrPayload.fromContextMap

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubPrDiffClient.java`
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubPrDiffClientTest.java`
- Modify: `review/src/main/java/io/casehub/devtown/review/PrPayload.java` — add `fromContextMap()` static factory

**Interfaces:**
- Consumes: `PrDiffService`, `PrDiff`, `PrDiff.FileDiff`, `GitHubPullRequestApi` (REST client)
- Produces: `GitHubPrDiffClient.fetchDiff(String, int)`, `PrPayload.fromContextMap(Map<String, Object>)`

- [ ] **Step 1: Write GitHubPrDiffClient test**

Test with simulation framework — REST client simulation generates decorator for `@RegisterRestClient` interface. Configure corpus fixture for `GET /repos/{owner}/{repo}/pulls/{number}/files`.

- [ ] **Step 2: Run test — verify it fails**

- [ ] **Step 3: Add diff files endpoint to GitHubPullRequestApi**

Use `ide_insert_member` to add `listPullRequestFiles` method to `GitHubPullRequestApi`:

```java
@GET
@Path("/repos/{owner}/{repo}/pulls/{pull_number}/files")
List<Map<String, Object>> listPullRequestFiles(
    @PathParam("owner") String owner,
    @PathParam("repo") String repo,
    @PathParam("pull_number") int pullNumber,
    @QueryParam("per_page") int perPage,
    @QueryParam("page") int page);
```

- [ ] **Step 4: Create GitHubPrDiffClient**

Full implementation with pagination (follow pages until response.size() < perPage), null-safe patch extraction, truncation detection (3000 file cap).

- [ ] **Step 5: Add PrPayload.fromContextMap()**

Use `ide_insert_member` on `PrPayload`:

```java
@SuppressWarnings("unchecked")
public static PrPayload fromContextMap(Map<String, Object> prMap) {
    return new PrPayload(
        (String) prMap.get("repo"),
        Integer.parseInt(String.valueOf(prMap.get("id"))),
        (String) prMap.get("headSha"),
        (String) prMap.get("baseRef"),
        ((Number) prMap.get("linesChanged")).intValue(),
        (String) prMap.get("contributor"),
        -1L,
        (List<String>) prMap.get("changedPaths"));
}
```

- [ ] **Step 6: Run tests — verify they pass**
- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add GitHubPrDiffClient with pagination, PrPayload.fromContextMap() Refs #203"
```

## Batch 4: LLM Agents and Engine Wiring

### Task 6: Eidos descriptors + agent-config + 6 LLM agent implementations

**Files:**
- Create: `app/src/main/resources/META-INF/eidos/descriptors.yaml`
- Create: `agent-config.yaml` (project root)
- Create: `app/src/main/java/io/casehub/devtown/app/agents/LlmCodeAnalysisAgent.java`
- Create: `app/src/main/java/io/casehub/devtown/app/agents/LlmSecurityReviewAgent.java`
- Create: `app/src/main/java/io/casehub/devtown/app/agents/LlmArchitectureReviewAgent.java`
- Create: `app/src/main/java/io/casehub/devtown/app/agents/LlmStyleReviewAgent.java`
- Create: `app/src/main/java/io/casehub/devtown/app/agents/LlmTestCoverageReviewAgent.java`
- Create: `app/src/main/java/io/casehub/devtown/app/agents/LlmPerformanceReviewAgent.java`
- Create: `app/src/test/java/io/casehub/devtown/app/agents/LlmSecurityReviewAgentTest.java`
- Modify: `app/pom.xml` — add casehub-eidos-api (compile), casehub-eidos (runtime) dependencies
- Modify: `review/pom.xml` — add casehub-eidos-api (compile) dependency

**Interfaces:**
- Consumes: `LlmReviewerAgent`, `LlmAgentBase`, `CodeAnalysisAgent`, `ReviewDomain`, `AgentProvider`, `AgentRegistry`, `StructuredAgentInvoker`
- Produces: 6 concrete LLM agent beans

- [ ] **Step 1: Add eidos dependencies to pom.xml files**
- [ ] **Step 2: Write LlmSecurityReviewAgent test** (using `Simulation.forTest()`)
- [ ] **Step 3: Run test — verify it fails**
- [ ] **Step 4: Create eidos descriptors.yaml** (per spec §Eidos integration)
- [ ] **Step 5: Create agent-config.yaml** (per spec §Agent config manifest)
- [ ] **Step 6: Create LlmCodeAnalysisAgent** — extends LlmAgentBase, implements CodeAnalysisAgent
- [ ] **Step 7: Create all 5 LlmReviewerAgent subclasses** — each is thin: system prompt + capability + agentId
- [ ] **Step 8: Run tests — verify they pass**
- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add 6 LLM-powered reviewer agents with eidos descriptors Refs #203"
```

### Task 7: PrReviewCaseHub function workers + QhorusPrReviewService update + pr-review.yaml

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java` — add function workers in `augment()`
- Modify: `app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java` — switch to registry + ReviewContext
- Modify: `review/src/main/resources/devtown/pr-review.yaml` — update output projections
- Create: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubReviewAdapterTest.java`

**Interfaces:**
- Consumes: `ReviewerAgentRegistry`, `CodeAnalysisAgent`, `PrDiffCache`, `PrPayload.fromContextMap()`, `Worker.builder()`, `WorkerResult`
- Produces: Function workers for all 6 review capabilities + code-analysis, registered in `augment()`

- [ ] **Step 1: Write adapter test** — `adaptReview()` with mock registry + simulated diff, per spec §Adapter bridge tests
- [ ] **Step 2: Run test — verify it fails**
- [ ] **Step 3: Add function workers to PrReviewCaseHub.augment()** — per spec §Review capability function workers
- [ ] **Step 4: Add adaptReview() and adaptCodeAnalysis() methods** — per spec
- [ ] **Step 5: Add buildContext() helper** — per spec
- [ ] **Step 6: Update QhorusPrReviewService** — switch from `Instance<ReviewerAgent>` to `ReviewerAgentRegistry`, single diff fetch, try-catch per agent
- [ ] **Step 7: Update pr-review.yaml output projections** — remove nested `{ outcome: . }` wrapping per spec table
- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: wire LLM agents into engine via function workers, update Layer 3 dispatch Refs #203"
```

## Batch 5: Integration Tests and Simulation Setup

### Task 8: Simulation-based integration tests

**Files:**
- Modify: `app/pom.xml` — add `casehub-platform-simulation-starter` (test scope)
- Create: `app/src/test/resources/simulation.yaml`
- Create: `app/src/test/resources/fixtures/llm-review-corpus.yaml`
- Create: `app/src/test/java/io/casehub/devtown/app/LlmReviewIntegrationTest.java`

**Interfaces:**
- Consumes: Everything from Batches 1-4

- [ ] **Step 1: Add simulation-starter dependency**
- [ ] **Step 2: Create simulation.yaml** with review-test profile
- [ ] **Step 3: Create corpus fixture** with sample LLM responses keyed by prompt content
- [ ] **Step 4: Write integration test** — full end-to-end: PR payload → case start → code-analysis → security-review → findings in context
- [ ] **Step 5: Run integration tests**
- [ ] **Step 6: Run full build — verify all tests pass**
- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add simulation-based integration tests for LLM reviewer agents Refs #203"
```

## References

- [2026-09-16-llm-reviewer-agents-design.md] — design spec this plan implements
- [review/src/main/java/.../ReviewerAgent.java] — port interface (modified)
- [review/src/main/java/.../ReviewerOutcome.java] — sealed outcome type (modified)
- [app/src/main/java/.../PrReviewCaseHub.java:24-30] — augment() method for function worker registration
- [app/src/main/java/.../QhorusPrReviewService.java] — Layer 3 dispatch (modified)
- [review/src/main/resources/devtown/pr-review.yaml] — CasePlanModel bindings (output projections modified)
- [casehub-blocks StructuredAgentInvoker] — blocks#287
- [casehub-platform consumer-guide.md §Agent infrastructure] — AgentProvider, manifest
- [casehub-eidos consumer-guide.md] — AgentDescriptor, AgentCapability
- [docs/protocols/casehub/failure-cascade-pattern.md] — 4-tier failure cascade
- [docs/protocols/casehub/alternative-extension-patterns.md] — @DefaultBean per-type semantics
- [GitHub #203] — focal issue
