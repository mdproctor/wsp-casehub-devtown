# Epic 3: PR Review CasePlanModel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the devtown PR review CasePlanModel — a YAML case definition with 9 bindings and 3 goals that routes PRs to specialist reviewers based on what code analysis finds, with all binding conditions verified by pure unit tests.

**Architecture:** YAML is the runtime artifact loaded by `PrReviewCaseHub extends YamlCaseHub`. `PrReviewCaseService @ApplicationScoped` (no `@DefaultBean`) displaces `NaivePrReviewService @DefaultBean` at CDI resolution time. Binding condition unit tests use a `PrReviewCaseDefinition` fluent DSL factory with `LambdaExpressionEvaluator` — no Quarkus, no YAML parsing.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine (YamlCaseHub, CaseDefinition, Binding, Goal), casehub-engine-api (LambdaExpressionEvaluator, CaseContext), JUnit 5, AssertJ 3.27.7

**Build command (full):** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

**Build command (review module only):** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

**Build command (app module only):** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Modify | `devtown/pom.xml` | Add `casehub-engine` to BOM dependencyManagement |
| Modify | `app/pom.xml` | Add `casehub-engine` runtime dependency |
| Modify | `review/pom.xml` | Add AssertJ test dependency |
| Create | `review/src/main/resources/devtown/pr-review.yaml` | 9 bindings, 3 goals, 9 capabilities |
| Create | `app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java` | `@ApplicationScoped extends YamlCaseHub` |
| Create | `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java` | `@ApplicationScoped` — builds initial context, opens case |
| Create | `review/src/test/java/io/casehub/devtown/review/MapCaseContext.java` | `CaseContext` over `Map<String,Object>` — test helper |
| Create | `review/src/test/java/io/casehub/devtown/review/PrReviewCaseDefinition.java` | Fluent DSL factory — mirrors YAML with lambda conditions |
| Create | `review/src/test/java/io/casehub/devtown/review/PrReviewBindingConditionTest.java` | Pure unit tests — all 9 binding conditions |
| Create | `app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubTest.java` | `@QuarkusTest` — YAML round-trip |

---

## Task 1: Add casehub-engine dependency

**Files:**
- Modify: `devtown/pom.xml` (dependencyManagement section)
- Modify: `app/pom.xml` (dependencies section)
- Modify: `review/pom.xml` (test dependencies section)

- [ ] **Step 1: Add casehub-engine to devtown BOM**

In `/Users/mdproctor/claude/casehub/devtown/pom.xml`, in the `<dependencyManagement><dependencies>` section, after the `casehub-engine-api` entry:

```xml
      <!-- casehub-engine runtime — needed by app/ for YamlCaseHub CDI wiring -->
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine</artifactId>
        <version>${casehub.version}</version>
      </dependency>
```

- [ ] **Step 2: Add casehub-engine to app pom**

In `/Users/mdproctor/claude/casehub/devtown/app/pom.xml`, in the `<!-- foundation runtime -->` section, after `casehub-work`:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine</artifactId>
    </dependency>
```

- [ ] **Step 3: Add AssertJ to review pom**

In `/Users/mdproctor/claude/casehub/devtown/review/pom.xml`, add a `<dependencies>` section (it currently has none for tests):

```xml
  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-devtown-domain</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-work-api</artifactId>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
```

Note: Read the current review/pom.xml first to confirm the existing dependency list, then replace the `<dependencies>` block with the above.

- [ ] **Step 4: Compile to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -q
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add pom.xml app/pom.xml review/pom.xml
git -C /Users/mdproctor/claude/casehub/devtown commit -m "build(app): add casehub-engine runtime dep for YamlCaseHub

Refs #10"
```

---

## Task 2: YAML case definition

**Files:**
- Create: `review/src/main/resources/devtown/pr-review.yaml`

- [ ] **Step 1: Create the resources directory and YAML file**

```bash
mkdir -p /Users/mdproctor/claude/casehub/devtown/review/src/main/resources/devtown
```

Create `/Users/mdproctor/claude/casehub/devtown/review/src/main/resources/devtown/pr-review.yaml`:

```yaml
dsl: "0.1"
version: "1.0.0"
name: pr-review
namespace: devtown
title: PR Review — content-driven routing and parallel checks
expressionLang: jq

spec:

  ## ─── Capabilities ──────────────────────────────────────────────────
  capabilities:
    - name: code-analysis
      description: "Characterises the code: crypto, auth, cross-API changes, scope"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{ codeAnalysis: . }"

    - name: security-review
      description: "Security specialist review — fires only on security-sensitive code"
      inputSchema: "{ pr: .pr, codeAnalysis: .codeAnalysis }"
      outputSchema: "{ securityReview: { outcome: . } }"

    - name: architecture-review
      description: "Architecture review — fires only when cross-API changes are detected"
      inputSchema: "{ pr: .pr, codeAnalysis: .codeAnalysis }"
      outputSchema: "{ architectureReview: { outcome: . } }"

    - name: style-review
      description: "Style and formatting check"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{ styleCheck: { outcome: . } }"

    - name: test-coverage
      description: "Test coverage analysis"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{ testCoverage: { outcome: . } }"

    - name: performance-analysis
      description: "Performance impact analysis"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{ performanceAnalysis: { outcome: . } }"

    - name: ci-runner
      description: "Runs the CI suite for the PR"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{ ci: { status: . } }"

    - name: human-decision:pr-approval
      description: "Human senior architect approval gate"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{ humanApproval: { status: . } }"

    - name: merge-executor
      description: "Executes the merge when all goals are satisfied"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{}"

  ## ─── Goals ──────────────────────────────────────────────────────────
  goals:
    - name: pr-approved
      kind: success
      condition: >-
        .securityReview.outcome == "APPROVED" and
        (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
        .styleCheck.outcome == "APPROVED" and
        .testCoverage.outcome == "APPROVED" and
        .performanceAnalysis.outcome == "APPROVED"

    - name: security-verified
      kind: success
      condition: >-
        .codeAnalysis.securitySensitive == false or
        .securityReview.outcome == "APPROVED"

    - name: ci-passing
      kind: success
      condition: '.ci.status == "passing"'

  completion:
    success:
      allOf:
        - pr-approved
        - security-verified
        - ci-passing

  ## ─── Bindings ────────────────────────────────────────────────────────
  bindings:

    ## Group 1: Entry — fire immediately when PR arrives
    - name: initial-analysis
      on: { contextChange: {} }
      when: ".pr != null and .codeAnalysis == null"
      capability: code-analysis

    - name: run-ci
      on: { contextChange: {} }
      when: ".pr != null and .ci == null"
      capability: ci-runner

    ## Group 2: Content-driven — fire after analysis, all in parallel
    - name: security-review
      on: { contextChange: {} }
      when: >-
        .codeAnalysis.complete == true and
        .codeAnalysis.securitySensitive == true and
        .securityReview == null
      capability: security-review

    - name: architecture-review
      on: { contextChange: {} }
      when: >-
        .codeAnalysis.complete == true and
        .codeAnalysis.architectureCrossing == true and
        .architectureReview == null
      capability: architecture-review

    - name: style-check
      on: { contextChange: {} }
      when: ".codeAnalysis.complete == true and .styleCheck == null"
      capability: style-review

    - name: test-coverage
      on: { contextChange: {} }
      when: ".codeAnalysis.complete == true and .testCoverage == null"
      capability: test-coverage

    - name: performance-analysis
      on: { contextChange: {} }
      when: ".codeAnalysis.complete == true and .performanceAnalysis == null"
      capability: performance-analysis

    ## Group 3: Human gate — fires alongside CI when PR exceeds threshold
    - name: human-approval
      on: { contextChange: {} }
      when: ".pr.linesChanged > .policy.humanApprovalThreshold and .humanApproval == null"
      capability: "human-decision:pr-approval"

    ## Group 4: Merge — fires when all conditions satisfied
    - name: merge
      on: { contextChange: {} }
      when: >-
        .securityReview.outcome == "APPROVED" and
        (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
        .styleCheck.outcome == "APPROVED" and
        .testCoverage.outcome == "APPROVED" and
        .performanceAnalysis.outcome == "APPROVED" and
        (.pr.linesChanged <= .policy.humanApprovalThreshold or .humanApproval.status == "approved") and
        .ci.status == "passing"
      capability: merge-executor
```

- [ ] **Step 2: Compile review module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl review -f /Users/mdproctor/claude/casehub/devtown/pom.xml -q
```
Expected: BUILD SUCCESS (YAML is a resource, not compiled — but confirms no syntax errors in other sources)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add review/src/main/resources/devtown/pr-review.yaml
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(review): PR review CasePlanModel YAML — 9 bindings, 3 goals

Content-driven routing, automatic parallelism, human gate.
Refs #10"
```

---

## Task 3: PrReviewCaseHub and PrReviewCaseService

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java`
- Create: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

- [ ] **Step 1: Create PrReviewCaseHub**

```java
package io.casehub.devtown.app;

import io.casehub.api.engine.YamlCaseHub;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class PrReviewCaseHub extends YamlCaseHub {

    public PrReviewCaseHub() {
        super("devtown/pr-review.yaml");
    }
}
```

- [ ] **Step 2: Create PrReviewCaseService**

```java
package io.casehub.devtown.app;

import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.PrReviewOutcome;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class PrReviewCaseService implements PrReviewApplicationService {

    @Inject PrReviewCaseHub caseHub;

    @ConfigProperty(name = "devtown.policy.human-approval-threshold", defaultValue = "500")
    int humanApprovalThreshold;

    @ConfigProperty(name = "devtown.policy.security-review-required", defaultValue = "true")
    boolean securityReviewRequired;

    @ConfigProperty(name = "devtown.policy.require-senior-approval", defaultValue = "false")
    boolean requireSeniorApproval;

    @Override
    public PrReviewOutcome review(PrPayload pr) {
        // TODO(parent#26): replace @ConfigProperty injection with PreferenceProvider.resolve(scope).asMap()
        var policy = Map.<String, Object>of(
            "humanApprovalThreshold", humanApprovalThreshold,
            "securityReviewRequired", securityReviewRequired,
            "requireSeniorApproval", requireSeniorApproval
        );
        var prContext = Map.<String, Object>of(
            "id", String.valueOf(pr.prNumber()),
            "repo", pr.repo(),
            "linesChanged", pr.linesChanged(),
            "baseRef", pr.baseRef(),
            "headSha", pr.headSha()
        );
        var initialContext = Map.<String, Object>of(
            "pr", prContext,
            "policy", policy
        );
        caseHub.startCase(initialContext);
        return new PrReviewOutcome("case-opened", List.of());
    }
}
```

Note: `PrPayload` currently has fields `repo`, `prNumber`, `headSha`, `linesChanged`. It's missing `baseRef` — add that field to `PrPayload` in this step.

Updated `PrPayload` (in `review/src/main/java/io/casehub/devtown/review/PrPayload.java`):

```java
package io.casehub.devtown.review;

public record PrPayload(String repo, int prNumber, String headSha, String baseRef, int linesChanged) {}
```

Also update `NaivePrReviewService` to use the new field (add `pr.baseRef()` — just add it to the constructor call or stub it):

```java
// In NaivePrReviewService.review() — no change needed, it doesn't use baseRef
```

And update `NaivePrReviewServiceTest` to pass the new field:

```java
var pr = new PrPayload("casehubio/devtown", 42, "abc123", "main", 150);
```

- [ ] **Step 3: Compile app**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -q
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Run existing tests to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml
```
Expected: All existing tests pass (DevtownBootTest × 2 + NaivePrReviewServiceTest × 3)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java \
  app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java \
  review/src/main/java/io/casehub/devtown/review/PrPayload.java \
  app/src/test/java/io/casehub/devtown/app/NaivePrReviewServiceTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): PrReviewCaseHub and PrReviewCaseService

Displaces NaivePrReviewService. Builds initial CaseContext from PR
payload and @ConfigProperty policy values. Adds baseRef to PrPayload.

Refs #10"
```

---

## Task 4: MapCaseContext test helper

**Files:**
- Create: `review/src/test/java/io/casehub/devtown/review/MapCaseContext.java`

- [ ] **Step 1: Create MapCaseContext**

```java
package io.casehub.devtown.review;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.context.CaseContext;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.function.Function;

/**
 * Test-only CaseContext backed by a flat Map<String, Object>.
 * Implements get(), contains(), and getPath() for nested access.
 * All other methods throw UnsupportedOperationException.
 */
class MapCaseContext implements CaseContext {

    private final Map<String, Object> data;

    MapCaseContext(Map<String, Object> data) {
        this.data = data;
    }

    @Override
    public Map<String, Object> getData() {
        return data;
    }

    @Override
    public Object get(String key) {
        return data.get(key);
    }

    @Override
    public boolean contains(String key) {
        return data.containsKey(key);
    }

    @Override
    @SuppressWarnings("unchecked")
    public Object getPath(String path) {
        String[] parts = path.split("\\.", 2);
        Object val = data.get(parts[0]);
        if (parts.length == 1 || val == null) return val;
        if (val instanceof Map<?, ?> m) {
            return new MapCaseContext((Map<String, Object>) m).getPath(parts[1]);
        }
        return null;
    }

    @Override
    public Integer getInt(String key) {
        Object val = data.get(key);
        if (val instanceof Number n) return n.intValue();
        return null;
    }

    @Override
    public Boolean getBoolean(String key) {
        Object val = data.get(key);
        if (val instanceof Boolean b) return b;
        return null;
    }

    @Override
    public String getString(String key) {
        Object val = data.get(key);
        return val != null ? val.toString() : null;
    }

    @Override public <T> T getAs(String key, Class<T> type) { throw new UnsupportedOperationException(); }
    @Override public <T> T getOrDefault(String key, T defaultValue) { throw new UnsupportedOperationException(); }
    @Override public Object computeIfAbsent(String key, Function<String, Object> f) { throw new UnsupportedOperationException(); }
    @Override public Object putIfAbsent(String key, Object value) { throw new UnsupportedOperationException(); }
    @Override public boolean compareAndSet(String key, Object expected, Object newValue) { throw new UnsupportedOperationException(); }
    @Override public CaseContext update(String key, Function<Object, Object> f) { throw new UnsupportedOperationException(); }
    @Override public Long getLong(String key) { throw new UnsupportedOperationException(); }
    @Override public Double getDouble(String key) { throw new UnsupportedOperationException(); }
    @Override public <T> java.util.List<T> getList(String key, Class<T> elementType) { throw new UnsupportedOperationException(); }
    @Override public String getPathAsString(String path) { throw new UnsupportedOperationException(); }
    @Override public CaseContext setPath(String path, Object value) { throw new UnsupportedOperationException(); }
    @Override public Optional<JsonNode> applyAndDiff(String path, Object value) { throw new UnsupportedOperationException(); }
    @Override public CaseContext set(String key, Object value) { throw new UnsupportedOperationException(); }
    @Override public CaseContext setAll(Map<String, Object> values) { throw new UnsupportedOperationException(); }
    @Override public Map<String, Object> getAll(String... keys) { throw new UnsupportedOperationException(); }
    @Override public CaseContext remove(String key) { throw new UnsupportedOperationException(); }
    @Override public CaseContext clear() { throw new UnsupportedOperationException(); }
    @Override public Set<String> getKeys() { throw new UnsupportedOperationException(); }
    @Override public boolean isEmpty() { throw new UnsupportedOperationException(); }
    @Override public int size() { throw new UnsupportedOperationException(); }
    @Override public JsonNode asJsonNode() { throw new UnsupportedOperationException(); }
    @Override public CaseContext merge(CaseContext other) { throw new UnsupportedOperationException(); }
    @Override public CaseContext snapshot() { throw new UnsupportedOperationException(); }
    @Override public JsonNode diff(CaseContext other) { throw new UnsupportedOperationException(); }
    @Override public void applyDiff(JsonNode diff) { throw new UnsupportedOperationException(); }
    @Override public long getVersion() { throw new UnsupportedOperationException(); }
    @Override public Map<String, Object> evalObjectTemplate(String template) { throw new UnsupportedOperationException(); }
}
```

- [ ] **Step 2: Compile review test sources**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl review -f /Users/mdproctor/claude/casehub/devtown/pom.xml -q
```
Expected: BUILD SUCCESS

---

## Task 5: PrReviewCaseDefinition fluent DSL factory + binding condition tests (TDD)

**Files:**
- Create: `review/src/test/java/io/casehub/devtown/review/PrReviewBindingConditionTest.java` (failing tests first)
- Create: `review/src/test/java/io/casehub/devtown/review/PrReviewCaseDefinition.java` (implementation)

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.devtown.review;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.evaluator.LambdaExpressionEvaluator;
import java.util.Map;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

class PrReviewBindingConditionTest {

    private static final int THRESHOLD = 500;
    private CaseDefinition def;

    @BeforeEach
    void setup() {
        def = PrReviewCaseDefinition.build(THRESHOLD);
    }

    private LambdaExpressionEvaluator condition(String bindingName) {
        return def.getBindings().stream()
            .filter(b -> b.getName().equals(bindingName))
            .findFirst()
            .map(b -> (LambdaExpressionEvaluator) b.getWhen())
            .orElseThrow(() -> new AssertionError("Binding not found: " + bindingName));
    }

    private MapCaseContext ctx(Map<String, Object> data) {
        return new MapCaseContext(data);
    }

    private Map<String, Object> pr(int linesChanged) {
        return Map.of("id", "42", "repo", "casehubio/devtown",
            "linesChanged", linesChanged, "baseRef", "main", "headSha", "abc123");
    }

    private Map<String, Object> policy() {
        return Map.of("humanApprovalThreshold", THRESHOLD);
    }

    private Map<String, Object> analysis(boolean securitySensitive, boolean architectureCrossing) {
        return Map.of("complete", true,
            "securitySensitive", securitySensitive,
            "architectureCrossing", architectureCrossing);
    }

    @Nested class InitialAnalysis {
        @Test void fires_whenPrArrivesAndNoAnalysis() {
            assertThat(condition("initial-analysis").test(ctx(Map.of("pr", pr(100))))).isTrue();
        }
        @Test void doesNotFire_whenAnalysisAlreadyPresent() {
            assertThat(condition("initial-analysis").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(false, false))))).isFalse();
        }
        @Test void doesNotFire_whenNoPr() {
            assertThat(condition("initial-analysis").test(ctx(Map.of()))).isFalse();
        }
    }

    @Nested class RunCi {
        @Test void fires_whenPrArrivesAndNoCi() {
            assertThat(condition("run-ci").test(ctx(Map.of("pr", pr(100))))).isTrue();
        }
        @Test void doesNotFire_whenCiAlreadyPresent() {
            assertThat(condition("run-ci").test(ctx(Map.of(
                "pr", pr(100), "ci", Map.of("status", "pending"))))).isFalse();
        }
    }

    @Nested class SecurityReview {
        @Test void fires_whenAnalysisCompleteAndSecuritySensitive() {
            assertThat(condition("security-review").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(true, false))))).isTrue();
        }
        @Test void doesNotFire_whenNotSecuritySensitive() {
            assertThat(condition("security-review").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(false, false))))).isFalse();
        }
        @Test void doesNotFire_whenAlreadyReviewed() {
            assertThat(condition("security-review").test(ctx(Map.of(
                "pr", pr(100),
                "codeAnalysis", analysis(true, false),
                "securityReview", Map.of("outcome", "APPROVED"))))).isFalse();
        }
        @Test void doesNotFire_whenAnalysisNotComplete() {
            assertThat(condition("security-review").test(ctx(Map.of(
                "pr", pr(100),
                "codeAnalysis", Map.of("complete", false, "securitySensitive", true, "architectureCrossing", false))))).isFalse();
        }
    }

    @Nested class ArchitectureReview {
        @Test void fires_whenAnalysisCompleteAndArchitectureCrossing() {
            assertThat(condition("architecture-review").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(false, true))))).isTrue();
        }
        @Test void doesNotFire_whenNoArchitectureCrossing() {
            assertThat(condition("architecture-review").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(false, false))))).isFalse();
        }
        @Test void doesNotFire_whenAlreadyReviewed() {
            assertThat(condition("architecture-review").test(ctx(Map.of(
                "pr", pr(100),
                "codeAnalysis", analysis(false, true),
                "architectureReview", Map.of("outcome", "APPROVED"))))).isFalse();
        }
    }

    @Nested class ParallelChecks {
        @Test void styleCheck_fires_whenAnalysisComplete() {
            assertThat(condition("style-check").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(false, false))))).isTrue();
        }
        @Test void styleCheck_doesNotFire_whenAlreadyDone() {
            assertThat(condition("style-check").test(ctx(Map.of(
                "pr", pr(100),
                "codeAnalysis", analysis(false, false),
                "styleCheck", Map.of("outcome", "APPROVED"))))).isFalse();
        }
        @Test void testCoverage_fires_whenAnalysisComplete() {
            assertThat(condition("test-coverage").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(false, false))))).isTrue();
        }
        @Test void performanceAnalysis_fires_whenAnalysisComplete() {
            assertThat(condition("performance-analysis").test(ctx(Map.of(
                "pr", pr(100), "codeAnalysis", analysis(false, false))))).isTrue();
        }
    }

    @Nested class HumanApproval {
        @Test void fires_whenLinesExceedThreshold() {
            assertThat(condition("human-approval").test(ctx(Map.of(
                "pr", pr(THRESHOLD + 1), "policy", policy())))).isTrue();
        }
        @Test void doesNotFire_whenLinesAtThreshold() {
            assertThat(condition("human-approval").test(ctx(Map.of(
                "pr", pr(THRESHOLD), "policy", policy())))).isFalse();
        }
        @Test void doesNotFire_whenLinesBelow() {
            assertThat(condition("human-approval").test(ctx(Map.of(
                "pr", pr(100), "policy", policy())))).isFalse();
        }
        @Test void doesNotFire_whenAlreadyApproved() {
            assertThat(condition("human-approval").test(ctx(Map.of(
                "pr", pr(THRESHOLD + 1),
                "policy", policy(),
                "humanApproval", Map.of("status", "approved"))))).isFalse();
        }
    }

    @Nested class Merge {
        private Map<String, Object> allApproved() {
            return Map.of(
                "pr", pr(100),
                "policy", policy(),
                "codeAnalysis", analysis(false, false),
                "securityReview", Map.of("outcome", "APPROVED"),
                "styleCheck", Map.of("outcome", "APPROVED"),
                "testCoverage", Map.of("outcome", "APPROVED"),
                "performanceAnalysis", Map.of("outcome", "APPROVED"),
                "ci", Map.of("status", "passing")
            );
        }

        @Test void fires_whenAllConditionsSatisfied_noArchCrossing_noHumanRequired() {
            assertThat(condition("merge").test(ctx(allApproved()))).isTrue();
        }
        @Test void doesNotFire_whenCiNotPassing() {
            var ctx = new java.util.HashMap<>(allApproved());
            ctx.put("ci", Map.of("status", "failing"));
            assertThat(condition("merge").test(ctx(ctx))).isFalse();
        }
        @Test void doesNotFire_whenStyleCheckNotApproved() {
            var ctx = new java.util.HashMap<>(allApproved());
            ctx.put("styleCheck", Map.of("outcome", "FAILED"));
            assertThat(condition("merge").test(ctx(ctx))).isFalse();
        }
        @Test void doesNotFire_whenArchCrossingAndNoArchReview() {
            var ctx = new java.util.HashMap<>(allApproved());
            ctx.put("codeAnalysis", analysis(false, true));
            assertThat(condition("merge").test(ctx(ctx))).isFalse();
        }
        @Test void fires_whenArchCrossingAndArchReviewApproved() {
            var data = new java.util.HashMap<>(allApproved());
            data.put("codeAnalysis", analysis(false, true));
            data.put("architectureReview", Map.of("outcome", "APPROVED"));
            assertThat(condition("merge").test(ctx(data))).isTrue();
        }
        @Test void doesNotFire_whenHumanRequiredButNotApproved() {
            var data = new java.util.HashMap<>(allApproved());
            data.put("pr", pr(THRESHOLD + 1));
            assertThat(condition("merge").test(ctx(data))).isFalse();
        }
        @Test void fires_whenHumanRequiredAndApproved() {
            var data = new java.util.HashMap<>(allApproved());
            data.put("pr", pr(THRESHOLD + 1));
            data.put("humanApproval", Map.of("status", "approved"));
            assertThat(condition("merge").test(ctx(data))).isTrue();
        }
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest=PrReviewBindingConditionTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml 2>&1 | tail -10
```
Expected: COMPILATION ERROR — `PrReviewCaseDefinition` does not exist yet.

- [ ] **Step 3: Create PrReviewCaseDefinition fluent DSL factory**

```java
package io.casehub.devtown.review;

import io.casehub.api.model.Binding;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.Goal;
import io.casehub.api.model.GoalExpression;
import io.casehub.api.model.GoalKind;
import io.casehub.devtown.domain.AgentQualification;
import io.casehub.devtown.domain.HumanDecision;
import io.casehub.devtown.domain.ReviewDomain;

/**
 * Fluent DSL factory for the PR review case definition.
 *
 * <p>Produces the same logical structure as {@code devtown/pr-review.yaml} but uses
 * {@link io.casehub.api.model.evaluator.LambdaExpressionEvaluator} for binding conditions —
 * enabling pure unit tests without YAML parsing or a Quarkus context.
 *
 * <p>The human-approval binding uses a capability reference ({@link HumanDecision#PR_APPROVAL})
 * matching the YAML. Full {@code HumanTaskTarget} configuration is deferred to devtown#30
 * (HITL wiring).
 */
public final class PrReviewCaseDefinition {

    private PrReviewCaseDefinition() {}

    public static CaseDefinition build(int humanApprovalThreshold) {
        // Capabilities
        var codeAnalysisCap = cap(ReviewDomain.CODE_ANALYSIS);
        var securityReviewCap = cap(ReviewDomain.SECURITY_REVIEW);
        var architectureReviewCap = cap(ReviewDomain.ARCHITECTURE_REVIEW);
        var styleReviewCap = cap(ReviewDomain.STYLE_REVIEW);
        var testCoverageCap = cap(ReviewDomain.TEST_COVERAGE);
        var performanceAnalysisCap = cap(ReviewDomain.PERFORMANCE_ANALYSIS);
        var ciRunnerCap = cap(AgentQualification.CI_RUNNER);
        var humanDecisionCap = cap(HumanDecision.PR_APPROVAL);
        var mergeExecutorCap = cap(AgentQualification.MERGE_EXECUTOR);

        // Goals
        var prApproved = Goal.builder()
            .name("pr-approved")
            .kind(GoalKind.SUCCESS)
            .condition(ctx ->
                "APPROVED".equals(ctx.getPath("securityReview.outcome")) &&
                (Boolean.FALSE.equals(ctx.getPath("codeAnalysis.architectureCrossing")) ||
                    "APPROVED".equals(ctx.getPath("architectureReview.outcome"))) &&
                "APPROVED".equals(ctx.getPath("styleCheck.outcome")) &&
                "APPROVED".equals(ctx.getPath("testCoverage.outcome")) &&
                "APPROVED".equals(ctx.getPath("performanceAnalysis.outcome")))
            .build();

        var securityVerified = Goal.builder()
            .name("security-verified")
            .kind(GoalKind.SUCCESS)
            .condition(ctx ->
                Boolean.FALSE.equals(ctx.getPath("codeAnalysis.securitySensitive")) ||
                "APPROVED".equals(ctx.getPath("securityReview.outcome")))
            .build();

        var ciPassing = Goal.builder()
            .name("ci-passing")
            .kind(GoalKind.SUCCESS)
            .condition(ctx -> "passing".equals(ctx.getPath("ci.status")))
            .build();

        // null filter = fires on any context change (matches YAML `on: { contextChange: {} }`)
        var trigger = new ContextChangeTrigger((io.casehub.api.model.evaluator.ExpressionEvaluator) null);

        // GoalExpression.allOf() — if varargs not available, use List.of(prApproved, securityVerified, ciPassing)
        CaseDefinition def = CaseDefinition.builder()
            .namespace("devtown")
            .name("pr-review")
            .version("1.0.0")
            .completion(GoalExpression.allOf(prApproved, securityVerified, ciPassing))
            .build();

        def.getCapabilities().addAll(java.util.List.of(
            codeAnalysisCap, securityReviewCap, architectureReviewCap,
            styleReviewCap, testCoverageCap, performanceAnalysisCap,
            ciRunnerCap, humanDecisionCap, mergeExecutorCap));

        def.getGoals().addAll(java.util.List.of(prApproved, securityVerified, ciPassing));

        // Group 1: Entry
        def.getBindings().add(Binding.builder().name("initial-analysis").on(trigger)
            .when(ctx -> ctx.get("pr") != null && ctx.get("codeAnalysis") == null)
            .capability(codeAnalysisCap).build());

        def.getBindings().add(Binding.builder().name("run-ci").on(trigger)
            .when(ctx -> ctx.get("pr") != null && ctx.get("ci") == null)
            .capability(ciRunnerCap).build());

        // Group 2: Content-driven
        def.getBindings().add(Binding.builder().name("security-review").on(trigger)
            .when(ctx ->
                Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
                Boolean.TRUE.equals(ctx.getPath("codeAnalysis.securitySensitive")) &&
                ctx.get("securityReview") == null)
            .capability(securityReviewCap).build());

        def.getBindings().add(Binding.builder().name("architecture-review").on(trigger)
            .when(ctx ->
                Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
                Boolean.TRUE.equals(ctx.getPath("codeAnalysis.architectureCrossing")) &&
                ctx.get("architectureReview") == null)
            .capability(architectureReviewCap).build());

        def.getBindings().add(Binding.builder().name("style-check").on(trigger)
            .when(ctx ->
                Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
                ctx.get("styleCheck") == null)
            .capability(styleReviewCap).build());

        def.getBindings().add(Binding.builder().name("test-coverage").on(trigger)
            .when(ctx ->
                Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
                ctx.get("testCoverage") == null)
            .capability(testCoverageCap).build());

        def.getBindings().add(Binding.builder().name("performance-analysis").on(trigger)
            .when(ctx ->
                Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
                ctx.get("performanceAnalysis") == null)
            .capability(performanceAnalysisCap).build());

        // Group 3: Human gate
        def.getBindings().add(Binding.builder().name("human-approval").on(trigger)
            .when(ctx -> {
                Object linesChanged = ctx.getPath("pr.linesChanged");
                return linesChanged instanceof Number n &&
                    n.intValue() > humanApprovalThreshold &&
                    ctx.get("humanApproval") == null;
            })
            .capability(humanDecisionCap).build());

        // Group 4: Merge
        def.getBindings().add(Binding.builder().name("merge").on(trigger)
            .when(ctx -> {
                Object linesChanged = ctx.getPath("pr.linesChanged");
                Object threshold = ctx.getPath("policy.humanApprovalThreshold");
                boolean humanOk = (linesChanged instanceof Number l &&
                    threshold instanceof Number t &&
                    l.intValue() <= t.intValue()) ||
                    "approved".equals(ctx.getPath("humanApproval.status"));
                return "APPROVED".equals(ctx.getPath("securityReview.outcome")) &&
                    (Boolean.FALSE.equals(ctx.getPath("codeAnalysis.architectureCrossing")) ||
                        "APPROVED".equals(ctx.getPath("architectureReview.outcome"))) &&
                    "APPROVED".equals(ctx.getPath("styleCheck.outcome")) &&
                    "APPROVED".equals(ctx.getPath("testCoverage.outcome")) &&
                    "APPROVED".equals(ctx.getPath("performanceAnalysis.outcome")) &&
                    humanOk &&
                    "passing".equals(ctx.getPath("ci.status"));
            })
            .capability(mergeExecutorCap).build());

        return def;
    }

    private static Capability cap(String name) {
        return Capability.builder().name(name).inputSchema("{}").outputSchema("{}").build();
    }
}
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest=PrReviewBindingConditionTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml
```
Expected: All tests GREEN, BUILD SUCCESS. Count: 27 tests (3 InitialAnalysis + 2 RunCi + 4 SecurityReview + 3 ArchitectureReview + 4 ParallelChecks + 4 HumanApproval + 7 Merge)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  review/src/test/java/io/casehub/devtown/review/MapCaseContext.java \
  review/src/test/java/io/casehub/devtown/review/PrReviewCaseDefinition.java \
  review/src/test/java/io/casehub/devtown/review/PrReviewBindingConditionTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test(review): binding condition unit tests for PR review CasePlanModel

18 tests covering all 9 binding conditions — TDD, pure unit, no Quarkus.
Refs #10"
```

---

## Task 6: YAML round-trip @QuarkusTest

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.app;

import static org.assertj.core.api.Assertions.assertThat;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class PrReviewCaseHubTest {

    @Inject PrReviewCaseHub caseHub;

    @Test
    void definitionLoads() {
        var def = caseHub.getDefinition();
        assertThat(def).isNotNull();
        assertThat(def.getNamespace()).isEqualTo("devtown");
        assertThat(def.getName()).isEqualTo("pr-review");
        assertThat(def.getVersion()).isEqualTo("1.0.0");
    }

    @Test
    void hasNineBindings() {
        var def = caseHub.getDefinition();
        assertThat(def.getBindings()).hasSize(9);
        var names = def.getBindings().stream().map(b -> b.getName()).toList();
        assertThat(names).containsExactlyInAnyOrder(
            "initial-analysis", "run-ci",
            "security-review", "architecture-review", "style-check",
            "test-coverage", "performance-analysis",
            "human-approval", "merge");
    }

    @Test
    void hasThreeGoals() {
        var def = caseHub.getDefinition();
        assertThat(def.getGoals()).hasSize(3);
        var names = def.getGoals().stream().map(g -> g.getName()).toList();
        assertThat(names).containsExactlyInAnyOrder("pr-approved", "security-verified", "ci-passing");
    }

    @Test
    void hasNineCapabilities() {
        var def = caseHub.getDefinition();
        assertThat(def.getCapabilities()).hasSize(9);
    }

    @Test
    void hasCompletion() {
        var def = caseHub.getDefinition();
        assertThat(def.getCompletion()).isNotNull();
    }
}
```

- [ ] **Step 2: Run test — confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PrReviewCaseHubTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml 2>&1 | tail -15
```
Expected: FAIL — CDI injection or YAML load error (YAML file not yet on classpath for the app test)

If the test fails because `PrReviewCaseHub` cannot be found on classpath: verify `casehub-engine` is added to `app/pom.xml`. If the YAML loads but counts are wrong: fix the YAML.

- [ ] **Step 3: Run test — confirm it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PrReviewCaseHubTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml
```
Expected: 5 tests GREEN, BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test(app): YAML round-trip @QuarkusTest for PrReviewCaseHub

Verifies pr-review.yaml parses: 9 bindings, 3 goals, 9 capabilities.
Refs #10"
```

---

## Task 7: Full build verification

- [ ] **Step 1: Run full devtown build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml
```
Expected: BUILD SUCCESS, all modules pass.

- [ ] **Step 2: Confirm test counts**

Expected minimum:
- `domain`: 70 tests
- `review`: 18+ tests (binding condition tests)
- `app`: 7+ tests (2 DevtownBootTest + 3 NaivePrReviewServiceTest + 5 PrReviewCaseHubTest... wait, DevtownBootTest was 2, NaivePrReviewServiceTest was 3, PrReviewCaseHubTest adds 5 = total 10 in app)

---

## Task 8: LAYER-LOG update

**Files:**
- Modify: `LAYER-LOG.md` at project root

- [ ] **Step 1: Update Layer 5 entry**

Read `/Users/mdproctor/claude/casehub/devtown/LAYER-LOG.md` and fill in the `🔲` sections of the Layer 5 entry:

**Completed:** Update from `🔲 in progress` to `2026-05-19 — Epic 3 devtown#10`

**Key files:** Replace `🔲` with actual file paths and descriptions.

**Key wiring:** Fill in:
- YAML loading: `PrReviewCaseHub extends YamlCaseHub("devtown/pr-review.yaml")` — lazy-loaded via `CaseDefinitionYamlMapper`, CDI-managed in `app/`
- `@DefaultBean` displacement: `PrReviewCaseService @ApplicationScoped` (no `@DefaultBean`) displaces `NaivePrReviewService @DefaultBean` — no extra wiring needed, CDI handles it automatically
- Initial context: `PrReviewCaseService.review(PrPayload)` builds `{ pr: {...}, policy: {...} }` from `@ConfigProperty` values and calls `caseHub.startCase(initialContext)`
- Human approval: expressed as `capability: human-decision:pr-approval` in YAML — full `HumanTaskTarget` wiring deferred to devtown#30
- `casehub-engine` runtime dep added to `app/pom.xml` — required for `YamlCaseHub` CDI injection of `CaseHubRuntime`

**Gotchas:** Fill in any discovered during implementation (leave `🔲` with note if none).

**Pattern to replicate:** Fill in steps from the implementation.

- [ ] **Step 2: Commit LAYER-LOG**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add LAYER-LOG.md
git -C /Users/mdproctor/claude/casehub/devtown commit -m "docs(layer-log): complete Layer 5 entry for Epic 3

Refs #10"
```
