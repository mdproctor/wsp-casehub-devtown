# Epic 2: Domain Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the devtown domain vocabulary — four typed capability types, three trust dimensions, a `RoutingPolicy` record with maturity-phase logic, the `CapabilityRegistry` SPI, and its populated default implementation.

**Architecture:** All domain classes live in `devtown-domain` (pure Java, no Quarkus deps). The `CapabilityRegistry` SPI is in `devtown-domain/domain/spi/`. The CDI-wired `CapabilityRegistryBean` lives in `devtown-app`. Tests in `devtown-domain` are plain JUnit 5 — no Quarkus, no CDI. One integration test in `devtown-app` verifies CDI discovery. TDD throughout: write the failing test first, then the minimal implementation.

**Tech Stack:** Java 21, JUnit 5 (via Quarkus BOM), AssertJ 3.24.2, Maven

**Working directory:** `/Users/mdproctor/claude/casehub/devtown`

**Build commands:**
- Domain tests only: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain`
- Full build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode clean install`
- All commits reference issue: `Refs #9` (Epic 2)

**Spec:** `specs/2026-05-08-epic2-domain-model-design.md` in the workspace (`/Users/mdproctor/claude/public/casehub/devtown/`)

---

## File Map

```
pom.xml                                                    ← add assertj version management
devtown-domain/pom.xml                                     ← add junit-jupiter + assertj test deps

devtown-domain/src/main/java/io/casehub/devtown/
  domain/spi/
    CapabilityRegistry.java                                ← SPI interface
  domain/
    ReviewDomain.java                                      ← 6 review capability string constants
    AgentQualification.java                                ← 2 execution capability string constants
    HumanDecision.java                                     ← 1 human accountability constant
    HumanOversight.java                                    ← 1 human oversight constant
    DevtownTrustDimension.java                             ← 3 trust dimension string constants
    RoutingPolicy.java                                     ← record: threshold, minObs, margin, fallback, rationale
    DevtownCapabilityRegistry.java                         ← populated default impl of CapabilityRegistry

devtown-domain/src/test/java/io/casehub/devtown/domain/
  ReviewDomainTest.java
  AgentQualificationTest.java
  HumanDecisionTest.java
  HumanOversightTest.java
  DevtownTrustDimensionTest.java
  RoutingPolicyTest.java
  DevtownCapabilityRegistryTest.java

devtown-app/src/main/java/io/casehub/devtown/app/
  CapabilityRegistryBean.java                              ← @ApplicationScoped CDI wrapper

devtown-app/src/test/java/io/casehub/devtown/app/
  DevtownBootTest.java                                     ← extend with CDI discovery test
```

---

## Task 1: Test infrastructure and `CapabilityRegistry` SPI

**Files:**
- Modify: `pom.xml`
- Modify: `devtown-domain/pom.xml`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/spi/CapabilityRegistry.java`

- [ ] **Step 1: Add AssertJ version to devtown parent pom**

In `pom.xml`, add inside `<dependencyManagement><dependencies>` after the existing entries:

```xml
      <!-- AssertJ — test assertion library for all devtown modules -->
      <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.24.2</version>
        <scope>test</scope>
      </dependency>
```

- [ ] **Step 2: Add test dependencies to devtown-domain pom**

Replace the full content of `devtown-domain/pom.xml` with:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-devtown</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>devtown-domain</artifactId>
  <name>DevTown :: Domain</name>
  <description>Capability tags, trust dimensions, routing thresholds — pure Java domain vocabulary</description>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

</project>
```

- [ ] **Step 3: Create the `CapabilityRegistry` SPI interface**

Create `devtown-domain/src/main/java/io/casehub/devtown/domain/spi/CapabilityRegistry.java`:

```java
package io.casehub.devtown.domain.spi;

import io.casehub.devtown.domain.RoutingPolicy;
import java.util.Optional;
import java.util.Set;

public interface CapabilityRegistry {

    Set<String> capabilities();

    Optional<RoutingPolicy> policy(String capability);

    boolean isKnown(String capability);
}
```

- [ ] **Step 4: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl devtown-domain
```

Expected: `BUILD FAILURE` — `RoutingPolicy` does not yet exist. This is correct at this point; the SPI references a type we will create in Task 4. This confirms the compile dependency is wired.

> **Note:** If the compiler reports `RoutingPolicy` not found, that is the expected failure. Proceed to the next step.

- [ ] **Step 5: Commit**

```bash
git add pom.xml devtown-domain/pom.xml devtown-domain/src/main/java/io/casehub/devtown/domain/spi/
git commit -m "build: add test deps to devtown-domain; create CapabilityRegistry SPI

Refs #9"
```

---

## Task 2: `ReviewDomain` constants (TDD)

**Files:**
- Create: `devtown-domain/src/test/java/io/casehub/devtown/domain/ReviewDomainTest.java`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java`

- [ ] **Step 1: Write the failing test**

Create `devtown-domain/src/test/java/io/casehub/devtown/domain/ReviewDomainTest.java`:

```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class ReviewDomainTest {

    @Test
    void allConstantsNonBlank() {
        assertThat(ReviewDomain.CODE_ANALYSIS).isNotBlank();
        assertThat(ReviewDomain.SECURITY_REVIEW).isNotBlank();
        assertThat(ReviewDomain.ARCHITECTURE_REVIEW).isNotBlank();
        assertThat(ReviewDomain.STYLE_REVIEW).isNotBlank();
        assertThat(ReviewDomain.TEST_COVERAGE).isNotBlank();
        assertThat(ReviewDomain.PERFORMANCE_ANALYSIS).isNotBlank();
    }

    @Test
    void allConstantsUnique() {
        assertThat(Set.of(
            ReviewDomain.CODE_ANALYSIS,
            ReviewDomain.SECURITY_REVIEW,
            ReviewDomain.ARCHITECTURE_REVIEW,
            ReviewDomain.STYLE_REVIEW,
            ReviewDomain.TEST_COVERAGE,
            ReviewDomain.PERFORMANCE_ANALYSIS
        )).hasSize(6);
    }

    @Test
    void valuesMatchSpec() {
        assertThat(ReviewDomain.CODE_ANALYSIS).isEqualTo("code-analysis");
        assertThat(ReviewDomain.SECURITY_REVIEW).isEqualTo("security-review");
        assertThat(ReviewDomain.ARCHITECTURE_REVIEW).isEqualTo("architecture-review");
        assertThat(ReviewDomain.STYLE_REVIEW).isEqualTo("style-review");
        assertThat(ReviewDomain.TEST_COVERAGE).isEqualTo("test-coverage");
        assertThat(ReviewDomain.PERFORMANCE_ANALYSIS).isEqualTo("performance-analysis");
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain
```

Expected: `BUILD FAILURE` — `ReviewDomain` not found. Compile error confirms TDD cycle is live.

- [ ] **Step 3: Create `ReviewDomain`**

Create `devtown-domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java`:

```java
package io.casehub.devtown.domain;

public final class ReviewDomain {

    public static final String CODE_ANALYSIS        = "code-analysis";
    public static final String SECURITY_REVIEW      = "security-review";
    public static final String ARCHITECTURE_REVIEW  = "architecture-review";
    public static final String STYLE_REVIEW         = "style-review";
    public static final String TEST_COVERAGE        = "test-coverage";
    public static final String PERFORMANCE_ANALYSIS = "performance-analysis";

    private ReviewDomain() {}
}
```

- [ ] **Step 4: Run to verify tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain -Dtest=ReviewDomainTest
```

Expected: `BUILD SUCCESS`, `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git add devtown-domain/src/
git commit -m "feat: ReviewDomain — 6 analytical capability string constants

Refs #9"
```

---

## Task 3: `AgentQualification`, `HumanDecision`, `HumanOversight`, `DevtownTrustDimension` (TDD)

**Files:**
- Create: `devtown-domain/src/test/java/io/casehub/devtown/domain/AgentQualificationTest.java`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/AgentQualification.java`
- Create: `devtown-domain/src/test/java/io/casehub/devtown/domain/HumanDecisionTest.java`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/HumanDecision.java`
- Create: `devtown-domain/src/test/java/io/casehub/devtown/domain/HumanOversightTest.java`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/HumanOversight.java`
- Create: `devtown-domain/src/test/java/io/casehub/devtown/domain/DevtownTrustDimensionTest.java`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/DevtownTrustDimension.java`

- [ ] **Step 1: Write all four test files**

`AgentQualificationTest.java`:
```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class AgentQualificationTest {

    @Test
    void allConstantsNonBlank() {
        assertThat(AgentQualification.CI_RUNNER).isNotBlank();
        assertThat(AgentQualification.MERGE_EXECUTOR).isNotBlank();
    }

    @Test
    void allConstantsUnique() {
        assertThat(Set.of(
            AgentQualification.CI_RUNNER,
            AgentQualification.MERGE_EXECUTOR
        )).hasSize(2);
    }

    @Test
    void valuesMatchSpec() {
        assertThat(AgentQualification.CI_RUNNER).isEqualTo("ci-runner");
        assertThat(AgentQualification.MERGE_EXECUTOR).isEqualTo("merge-executor");
    }

    @Test
    void noOverlapWithReviewDomain() {
        assertThat(AgentQualification.CI_RUNNER).doesNotStartWith("code-")
            .doesNotStartWith("security-").doesNotStartWith("architecture-")
            .doesNotStartWith("style-").doesNotStartWith("test-")
            .doesNotStartWith("performance-");
        assertThat(AgentQualification.MERGE_EXECUTOR).doesNotStartWith("code-")
            .doesNotStartWith("security-").doesNotStartWith("architecture-")
            .doesNotStartWith("style-").doesNotStartWith("test-")
            .doesNotStartWith("performance-");
    }
}
```

`HumanDecisionTest.java`:
```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class HumanDecisionTest {

    @Test
    void constantNonBlank() {
        assertThat(HumanDecision.PR_APPROVAL).isNotBlank();
    }

    @Test
    void valueMatchesSpec() {
        assertThat(HumanDecision.PR_APPROVAL).isEqualTo("human-decision:pr-approval");
    }

    @Test
    void prefixedCorrectly() {
        assertThat(HumanDecision.PR_APPROVAL).startsWith("human-decision:");
    }

    @Test
    void noOverlapWithOtherTypes() {
        assertThat(HumanDecision.PR_APPROVAL)
            .doesNotStartWith("human-oversight:")
            .isNotEqualTo(AgentQualification.CI_RUNNER)
            .isNotEqualTo(AgentQualification.MERGE_EXECUTOR);
    }
}
```

`HumanOversightTest.java`:
```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class HumanOversightTest {

    @Test
    void constantNonBlank() {
        assertThat(HumanOversight.ROUTING_REVIEW).isNotBlank();
    }

    @Test
    void valueMatchesSpec() {
        assertThat(HumanOversight.ROUTING_REVIEW).isEqualTo("human-oversight:routing-review");
    }

    @Test
    void prefixedCorrectly() {
        assertThat(HumanOversight.ROUTING_REVIEW).startsWith("human-oversight:");
    }

    @Test
    void noOverlapWithHumanDecision() {
        assertThat(HumanOversight.ROUTING_REVIEW)
            .doesNotStartWith("human-decision:")
            .isNotEqualTo(HumanDecision.PR_APPROVAL);
    }
}
```

`DevtownTrustDimensionTest.java`:
```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class DevtownTrustDimensionTest {

    @Test
    void allConstantsNonBlank() {
        assertThat(DevtownTrustDimension.REVIEW_THOROUGHNESS).isNotBlank();
        assertThat(DevtownTrustDimension.FALSE_POSITIVE_RATE).isNotBlank();
        assertThat(DevtownTrustDimension.SCOPE_CALIBRATION).isNotBlank();
    }

    @Test
    void allConstantsUnique() {
        assertThat(Set.of(
            DevtownTrustDimension.REVIEW_THOROUGHNESS,
            DevtownTrustDimension.FALSE_POSITIVE_RATE,
            DevtownTrustDimension.SCOPE_CALIBRATION
        )).hasSize(3);
    }

    @Test
    void valuesMatchSpec() {
        assertThat(DevtownTrustDimension.REVIEW_THOROUGHNESS).isEqualTo("review-thoroughness");
        assertThat(DevtownTrustDimension.FALSE_POSITIVE_RATE).isEqualTo("false-positive-rate");
        assertThat(DevtownTrustDimension.SCOPE_CALIBRATION).isEqualTo("scope-calibration");
    }
}
```

- [ ] **Step 2: Run to verify all four fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain
```

Expected: `BUILD FAILURE` — all four classes not found.

- [ ] **Step 3: Create all four implementation classes**

`AgentQualification.java`:
```java
package io.casehub.devtown.domain;

public final class AgentQualification {

    public static final String CI_RUNNER      = "ci-runner";
    public static final String MERGE_EXECUTOR = "merge-executor";

    private AgentQualification() {}
}
```

`HumanDecision.java`:
```java
package io.casehub.devtown.domain;

public final class HumanDecision {

    public static final String PR_APPROVAL = "human-decision:pr-approval";

    private HumanDecision() {}
}
```

`HumanOversight.java`:
```java
package io.casehub.devtown.domain;

public final class HumanOversight {

    public static final String ROUTING_REVIEW = "human-oversight:routing-review";

    private HumanOversight() {}
}
```

`DevtownTrustDimension.java`:
```java
package io.casehub.devtown.domain;

public final class DevtownTrustDimension {

    public static final String REVIEW_THOROUGHNESS = "review-thoroughness";
    public static final String FALSE_POSITIVE_RATE = "false-positive-rate";
    public static final String SCOPE_CALIBRATION   = "scope-calibration";

    private DevtownTrustDimension() {}
}
```

- [ ] **Step 4: Run to verify all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain
```

Expected: `BUILD SUCCESS`, `Tests run: 13` (3 from ReviewDomain + 10 from the four new test classes), `Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git add devtown-domain/src/
git commit -m "feat: AgentQualification, HumanDecision, HumanOversight, DevtownTrustDimension constants

Refs #9"
```

---

## Task 4: `RoutingPolicy` record (TDD)

**Files:**
- Create: `devtown-domain/src/test/java/io/casehub/devtown/domain/RoutingPolicyTest.java`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/RoutingPolicy.java`

- [ ] **Step 1: Write the failing test**

Create `devtown-domain/src/test/java/io/casehub/devtown/domain/RoutingPolicyTest.java`:

```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RoutingPolicyTest {

    private static final RoutingPolicy SECURITY_POLICY = new RoutingPolicy(
        OptionalDouble.of(0.70),
        OptionalInt.of(10),
        OptionalDouble.of(0.05),
        Optional.of(HumanOversight.ROUTING_REVIEW),
        "Security mistakes reach production; 10 observations required for credible score"
    );

    private static final RoutingPolicy NO_GATE_POLICY = new RoutingPolicy(
        OptionalDouble.of(0.50),
        OptionalInt.empty(),
        OptionalDouble.empty(),
        Optional.empty(),
        "No observation gate"
    );

    // isBootstrap tests

    @Test
    void isBootstrapReturnsTrueWhenBelowMinimumObservations() {
        assertThat(SECURITY_POLICY.isBootstrap(4)).isTrue();
        assertThat(SECURITY_POLICY.isBootstrap(9)).isTrue();
    }

    @Test
    void isBootstrapReturnsFalseWhenAtMinimumObservations() {
        assertThat(SECURITY_POLICY.isBootstrap(10)).isFalse();
    }

    @Test
    void isBootstrapReturnsFalseWhenAboveMinimumObservations() {
        assertThat(SECURITY_POLICY.isBootstrap(100)).isFalse();
    }

    @Test
    void isBootstrapReturnsFalseWhenNoMinimumObservationsConfigured() {
        assertThat(NO_GATE_POLICY.isBootstrap(0)).isFalse();
        assertThat(NO_GATE_POLICY.isBootstrap(1)).isFalse();
    }

    // isBorderline tests

    @Test
    void isBorderlineReturnsTrueWhenWithinMarginAboveThreshold() {
        // threshold=0.70, margin=0.05 → borderline range: [0.70, 0.75)
        assertThat(SECURITY_POLICY.isBorderline(0.70)).isTrue();
        assertThat(SECURITY_POLICY.isBorderline(0.72)).isTrue();
        assertThat(SECURITY_POLICY.isBorderline(0.7499)).isTrue();
    }

    @Test
    void isBorderlineReturnsFalseWhenBelowThreshold() {
        assertThat(SECURITY_POLICY.isBorderline(0.69)).isFalse();
        assertThat(SECURITY_POLICY.isBorderline(0.50)).isFalse();
    }

    @Test
    void isBorderlineReturnsFalseWhenAtOrAboveThresholdPlusMargin() {
        assertThat(SECURITY_POLICY.isBorderline(0.75)).isFalse();
        assertThat(SECURITY_POLICY.isBorderline(0.90)).isFalse();
    }

    @Test
    void isBorderlineReturnsFalseWhenNoMarginConfigured() {
        assertThat(NO_GATE_POLICY.isBorderline(0.51)).isFalse();
        assertThat(NO_GATE_POLICY.isBorderline(0.70)).isFalse();
    }

    // Mutual exclusivity

    @Test
    void agentWithSufficientObservationsCanBeBorderline() {
        // An agent past bootstrap can be borderline — not mutually exclusive across agents
        assertThat(SECURITY_POLICY.isBootstrap(15)).isFalse();
        assertThat(SECURITY_POLICY.isBorderline(0.72)).isTrue();
    }

    // Record equality

    @Test
    void recordEquality() {
        RoutingPolicy a = new RoutingPolicy(
            OptionalDouble.of(0.70), OptionalInt.of(10),
            OptionalDouble.of(0.05), Optional.of(HumanOversight.ROUTING_REVIEW),
            "test"
        );
        RoutingPolicy b = new RoutingPolicy(
            OptionalDouble.of(0.70), OptionalInt.of(10),
            OptionalDouble.of(0.05), Optional.of(HumanOversight.ROUTING_REVIEW),
            "test"
        );
        assertThat(a).isEqualTo(b);
    }

    // Null guard

    @Test
    void nullRationaleThrowsNullPointerException() {
        assertThatThrownBy(() -> new RoutingPolicy(
            OptionalDouble.of(0.70), OptionalInt.of(10),
            OptionalDouble.of(0.05), Optional.empty(), null
        )).isInstanceOf(NullPointerException.class)
          .hasMessageContaining("rationale");
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain -Dtest=RoutingPolicyTest
```

Expected: `BUILD FAILURE` — `RoutingPolicy` not found.

- [ ] **Step 3: Create `RoutingPolicy`**

Create `devtown-domain/src/main/java/io/casehub/devtown/domain/RoutingPolicy.java`:

```java
package io.casehub.devtown.domain;

import java.util.Objects;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;

public record RoutingPolicy(
    OptionalDouble threshold,
    OptionalInt minimumObservations,
    OptionalDouble borderlineMargin,
    Optional<String> fallbackType,
    String rationale
) {
    public RoutingPolicy {
        Objects.requireNonNull(threshold, "threshold must not be null");
        Objects.requireNonNull(minimumObservations, "minimumObservations must not be null");
        Objects.requireNonNull(borderlineMargin, "borderlineMargin must not be null");
        Objects.requireNonNull(fallbackType, "fallbackType must not be null");
        Objects.requireNonNull(rationale, "rationale must not be null");
    }

    public boolean isBootstrap(int agentObservations) {
        return minimumObservations.isPresent()
            && agentObservations < minimumObservations.getAsInt();
    }

    public boolean isBorderline(double trustScore) {
        return borderlineMargin.isPresent() && threshold.isPresent()
            && trustScore >= threshold.getAsDouble()
            && trustScore < threshold.getAsDouble() + borderlineMargin.getAsDouble();
    }
}
```

- [ ] **Step 4: Run to verify all tests pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain
```

Expected: `BUILD SUCCESS`. At this point the `CapabilityRegistry` SPI also compiles since `RoutingPolicy` now exists.

Expected total: `Tests run: 24` (13 from Tasks 2-3 + 11 from RoutingPolicyTest), `Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git add devtown-domain/src/
git commit -m "feat: RoutingPolicy record — threshold, minimumObservations, borderlineMargin, fallbackType, rationale

isBootstrap() and isBorderline() implement the trust maturity model.
Refs #9"
```

---

## Task 5: `DevtownCapabilityRegistry` (TDD)

**Files:**
- Create: `devtown-domain/src/test/java/io/casehub/devtown/domain/DevtownCapabilityRegistryTest.java`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java`

- [ ] **Step 1: Write the failing test**

Create `devtown-domain/src/test/java/io/casehub/devtown/domain/DevtownCapabilityRegistryTest.java`:

```java
package io.casehub.devtown.domain;

import io.casehub.devtown.domain.spi.CapabilityRegistry;
import org.junit.jupiter.api.Test;
import java.util.Optional;
import java.util.OptionalDouble;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DevtownCapabilityRegistryTest {

    private final DevtownCapabilityRegistry registry = new DevtownCapabilityRegistry();

    // === capabilities() ===

    @Test
    void capabilitiesContainsAllTenExpectedValues() {
        assertThat(registry.capabilities()).containsExactlyInAnyOrder(
            "code-analysis", "security-review", "architecture-review",
            "style-review", "test-coverage", "performance-analysis",
            "ci-runner", "merge-executor",
            "human-decision:pr-approval", "human-oversight:routing-review"
        );
    }

    @Test
    void capabilitiesReturnsImmutableSet() {
        assertThatThrownBy(() -> registry.capabilities().add("intruder"))
            .isInstanceOf(UnsupportedOperationException.class);
    }

    // === policy() — trust-gated capabilities ===

    @Test
    void securityReviewThresholdIs070() {
        RoutingPolicy policy = registry.policy(ReviewDomain.SECURITY_REVIEW).orElseThrow();
        assertThat(policy.threshold().getAsDouble()).isEqualTo(0.70);
    }

    @Test
    void securityReviewMinimumObservationsIs10() {
        RoutingPolicy policy = registry.policy(ReviewDomain.SECURITY_REVIEW).orElseThrow();
        assertThat(policy.minimumObservations().getAsInt()).isEqualTo(10);
    }

    @Test
    void securityReviewBorderlineMarginIs005() {
        RoutingPolicy policy = registry.policy(ReviewDomain.SECURITY_REVIEW).orElseThrow();
        assertThat(policy.borderlineMargin().getAsDouble()).isEqualTo(0.05);
    }

    @Test
    void securityReviewFallbackIsHumanOversightRoutingReview() {
        RoutingPolicy policy = registry.policy(ReviewDomain.SECURITY_REVIEW).orElseThrow();
        assertThat(policy.fallbackType()).contains(HumanOversight.ROUTING_REVIEW);
    }

    @Test
    void architectureReviewThresholdIs065() {
        RoutingPolicy policy = registry.policy(ReviewDomain.ARCHITECTURE_REVIEW).orElseThrow();
        assertThat(policy.threshold().getAsDouble()).isEqualTo(0.65);
    }

    @Test
    void architectureReviewMinimumObservationsIs8() {
        RoutingPolicy policy = registry.policy(ReviewDomain.ARCHITECTURE_REVIEW).orElseThrow();
        assertThat(policy.minimumObservations().getAsInt()).isEqualTo(8);
    }

    @Test
    void styleReviewThresholdIs050() {
        RoutingPolicy policy = registry.policy(ReviewDomain.STYLE_REVIEW).orElseThrow();
        assertThat(policy.threshold().getAsDouble()).isEqualTo(0.50);
    }

    @Test
    void styleReviewMinimumObservationsIs5() {
        RoutingPolicy policy = registry.policy(ReviewDomain.STYLE_REVIEW).orElseThrow();
        assertThat(policy.minimumObservations().getAsInt()).isEqualTo(5);
    }

    @Test
    void styleReviewHasNoBorderlineMargin() {
        RoutingPolicy policy = registry.policy(ReviewDomain.STYLE_REVIEW).orElseThrow();
        assertThat(policy.borderlineMargin()).isEmpty();
    }

    @Test
    void mergeExecutorThresholdIs080() {
        RoutingPolicy policy = registry.policy(AgentQualification.MERGE_EXECUTOR).orElseThrow();
        assertThat(policy.threshold().getAsDouble()).isEqualTo(0.80);
    }

    @Test
    void mergeExecutorMinimumObservationsIs15() {
        RoutingPolicy policy = registry.policy(AgentQualification.MERGE_EXECUTOR).orElseThrow();
        assertThat(policy.minimumObservations().getAsInt()).isEqualTo(15);
    }

    @Test
    void mergeExecutorBorderlineMarginIs005() {
        RoutingPolicy policy = registry.policy(AgentQualification.MERGE_EXECUTOR).orElseThrow();
        assertThat(policy.borderlineMargin().getAsDouble()).isEqualTo(0.05);
    }

    // === policy() — non-gated capabilities return empty ===

    @Test
    void codeAnalysisHasNoPolicy() {
        assertThat(registry.policy(ReviewDomain.CODE_ANALYSIS)).isEmpty();
    }

    @Test
    void testCoverageHasNoPolicy() {
        assertThat(registry.policy(ReviewDomain.TEST_COVERAGE)).isEmpty();
    }

    @Test
    void performanceAnalysisHasNoPolicy() {
        assertThat(registry.policy(ReviewDomain.PERFORMANCE_ANALYSIS)).isEmpty();
    }

    @Test
    void ciRunnerHasNoPolicy() {
        assertThat(registry.policy(AgentQualification.CI_RUNNER)).isEmpty();
    }

    @Test
    void humanDecisionPrApprovalHasNoPolicy() {
        assertThat(registry.policy(HumanDecision.PR_APPROVAL)).isEmpty();
    }

    @Test
    void humanOversightRoutingReviewHasNoPolicy() {
        assertThat(registry.policy(HumanOversight.ROUTING_REVIEW)).isEmpty();
    }

    // === All threshold values in valid range ===

    @Test
    void allThresholdValuesAreInValidRange() {
        registry.capabilities().stream()
            .map(registry::policy)
            .filter(Optional::isPresent)
            .map(Optional::get)
            .map(RoutingPolicy::threshold)
            .filter(OptionalDouble::isPresent)
            .mapToDouble(OptionalDouble::getAsDouble)
            .forEach(t -> assertThat(t)
                .as("threshold must be in [0.0, 1.0]")
                .isGreaterThanOrEqualTo(0.0)
                .isLessThanOrEqualTo(1.0));
    }

    // === isKnown() ===

    @Test
    void isKnownReturnsTrueForAllCapabilities() {
        assertThat(registry.isKnown(ReviewDomain.SECURITY_REVIEW)).isTrue();
        assertThat(registry.isKnown(AgentQualification.MERGE_EXECUTOR)).isTrue();
        assertThat(registry.isKnown(HumanDecision.PR_APPROVAL)).isTrue();
        assertThat(registry.isKnown(HumanOversight.ROUTING_REVIEW)).isTrue();
    }

    @Test
    void isKnownReturnsFalseForUnknownCapability() {
        assertThat(registry.isKnown("unknown-capability")).isFalse();
        assertThat(registry.isKnown("")).isFalse();
    }

    // === Null guards ===

    @Test
    void policyNullThrowsNullPointerException() {
        assertThatThrownBy(() -> registry.policy(null))
            .isInstanceOf(NullPointerException.class)
            .hasMessageContaining("capability");
    }

    @Test
    void isKnownNullThrowsNullPointerException() {
        assertThatThrownBy(() -> registry.isKnown(null))
            .isInstanceOf(NullPointerException.class)
            .hasMessageContaining("capability");
    }

    // === Maturity model integration ===

    @Test
    void securityReviewPolicyAgentWithFewObservationsIsInBootstrap() {
        RoutingPolicy policy = registry.policy(ReviewDomain.SECURITY_REVIEW).orElseThrow();
        assertThat(policy.isBootstrap(4)).isTrue();
        assertThat(policy.isBootstrap(10)).isFalse();
    }

    @Test
    void securityReviewPolicyAgentJustAboveThresholdIsBorderline() {
        RoutingPolicy policy = registry.policy(ReviewDomain.SECURITY_REVIEW).orElseThrow();
        assertThat(policy.isBorderline(0.71)).isTrue();
        assertThat(policy.isBorderline(0.75)).isFalse();
    }

    // === Implements CapabilityRegistry SPI ===

    @Test
    void implementsCapabilityRegistrySpi() {
        assertThat(registry).isInstanceOf(CapabilityRegistry.class);
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain -Dtest=DevtownCapabilityRegistryTest
```

Expected: `BUILD FAILURE` — `DevtownCapabilityRegistry` not found.

- [ ] **Step 3: Create `DevtownCapabilityRegistry`**

Create `devtown-domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java`:

```java
package io.casehub.devtown.domain;

import io.casehub.devtown.domain.spi.CapabilityRegistry;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.OptionalInt;
import java.util.Set;

public class DevtownCapabilityRegistry implements CapabilityRegistry {

    private static final Set<String> ALL_CAPABILITIES = Set.of(
        ReviewDomain.CODE_ANALYSIS,
        ReviewDomain.SECURITY_REVIEW,
        ReviewDomain.ARCHITECTURE_REVIEW,
        ReviewDomain.STYLE_REVIEW,
        ReviewDomain.TEST_COVERAGE,
        ReviewDomain.PERFORMANCE_ANALYSIS,
        AgentQualification.CI_RUNNER,
        AgentQualification.MERGE_EXECUTOR,
        HumanDecision.PR_APPROVAL,
        HumanOversight.ROUTING_REVIEW
    );

    private static final Map<String, RoutingPolicy> POLICIES = Map.of(
        ReviewDomain.SECURITY_REVIEW, new RoutingPolicy(
            OptionalDouble.of(0.70),
            OptionalInt.of(10),
            OptionalDouble.of(0.05),
            Optional.of(HumanOversight.ROUTING_REVIEW),
            "Security mistakes reach production; 10 observations required for credible score"
        ),
        ReviewDomain.ARCHITECTURE_REVIEW, new RoutingPolicy(
            OptionalDouble.of(0.65),
            OptionalInt.of(8),
            OptionalDouble.of(0.05),
            Optional.of(HumanOversight.ROUTING_REVIEW),
            "Design mistakes are expensive to reverse; 8 observations required"
        ),
        ReviewDomain.STYLE_REVIEW, new RoutingPolicy(
            OptionalDouble.of(0.50),
            OptionalInt.of(5),
            OptionalDouble.empty(),
            Optional.empty(),
            "Baseline — any competent agent; 5 observations sufficient"
        ),
        AgentQualification.MERGE_EXECUTOR, new RoutingPolicy(
            OptionalDouble.of(0.80),
            OptionalInt.of(15),
            OptionalDouble.of(0.05),
            Optional.of(HumanOversight.ROUTING_REVIEW),
            "Merge is irreversible; highest observation requirement"
        )
    );

    @Override
    public Set<String> capabilities() {
        return ALL_CAPABILITIES;
    }

    @Override
    public Optional<RoutingPolicy> policy(String capability) {
        Objects.requireNonNull(capability, "capability must not be null");
        return Optional.ofNullable(POLICIES.get(capability));
    }

    @Override
    public boolean isKnown(String capability) {
        Objects.requireNonNull(capability, "capability must not be null");
        return ALL_CAPABILITIES.contains(capability);
    }
}
```

- [ ] **Step 4: Run all devtown-domain tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain
```

Expected: `BUILD SUCCESS`, `Tests run: 54` (24 prior + 30 new), `Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git add devtown-domain/src/
git commit -m "feat: DevtownCapabilityRegistry — populated default CapabilityRegistry implementation

10 capabilities across 4 typed vocabulary types. RoutingPolicy for
security-review (0.70/10obs/0.05margin), architecture-review (0.65/8obs),
style-review (0.50/5obs), merge-executor (0.80/15obs).
Trust maturity model: isBootstrap() and isBorderline() tested end-to-end.

Refs #9"
```

---

## Task 6: `CapabilityRegistryBean` and CDI integration test

**Files:**
- Create: `devtown-app/src/main/java/io/casehub/devtown/app/CapabilityRegistryBean.java`
- Modify: `devtown-app/src/test/java/io/casehub/devtown/app/DevtownBootTest.java`

- [ ] **Step 1: Write the CDI integration test first**

Replace `devtown-app/src/test/java/io/casehub/devtown/app/DevtownBootTest.java` with:

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.spi.CapabilityRegistry;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DevtownBootTest {

    @Inject
    CapabilityRegistry capabilityRegistry;

    @Test
    void applicationBoots() {
    }

    @Test
    void capabilityRegistryIsDiscoverableViaCdi() {
        assertThat(capabilityRegistry).isNotNull();
        assertThat(capabilityRegistry.capabilities()).hasSize(10);
    }
}
```

- [ ] **Step 2: Run to verify the new test fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain,devtown-review,devtown-queue,devtown-github,devtown-app
```

Expected: `BUILD FAILURE` — `CapabilityRegistryBean` does not exist; CDI cannot satisfy the `@Inject CapabilityRegistry` injection point.

- [ ] **Step 3: Create `CapabilityRegistryBean`**

Create `devtown-app/src/main/java/io/casehub/devtown/app/CapabilityRegistryBean.java`:

```java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.DevtownCapabilityRegistry;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class CapabilityRegistryBean extends DevtownCapabilityRegistry {}
```

- [ ] **Step 4: Run to verify the CDI test passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain,devtown-review,devtown-queue,devtown-github,devtown-app
```

Expected: `BUILD SUCCESS`, `Tests run: 2` in devtown-app (`applicationBoots` and `capabilityRegistryIsDiscoverableViaCdi`), both `PASSED`.

- [ ] **Step 5: Also add AssertJ to devtown-app test deps**

`devtown-app` already has `casehub-qhorus-testing` as a test dep. Check its `pom.xml` for existing test deps and add AssertJ:

In `devtown-app/pom.xml`, add inside `<dependencies>` alongside existing test deps:
```xml
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
```

Re-run to confirm still passing:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain,devtown-review,devtown-queue,devtown-github,devtown-app
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Run full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode clean install
```

Expected: `BUILD SUCCESS` — all 5 modules compile and all tests pass.

- [ ] **Step 7: Commit**

```bash
git add devtown-app/src/ devtown-app/pom.xml
git commit -m "feat: CapabilityRegistryBean — @ApplicationScoped CDI wrapper; CDI integration test

Refs #9"
```

---

## Task 7: Documentation updates

**Files:**
- Modify: `docs/PROGRESS.md` (project repo)
- Modify: `docs/gastown-casehub-analysis-v2.md` (project repo)

- [ ] **Step 1: Update PROGRESS.md — mark DT-001 through DT-006 as Implemented**

In `docs/PROGRESS.md`, change every occurrence of `Designed — Epic 2 (devtown#9)` to `✅ Implemented — Epic 2 (devtown#9)`.

There are 6 entries: DT-001, DT-002, DT-003, DT-004, DT-005, DT-006. Use find-and-replace; verify all 6 are updated.

- [ ] **Step 2: Update gastown-casehub-analysis-v2.md section 12 status column**

In `docs/gastown-casehub-analysis-v2.md`, update the section 12 summary table status column: change every `Designed — Epic 2` to `✅ Implemented — Epic 2`.

- [ ] **Step 3: Verify cross-references**

Check that these references in PROGRESS.md resolve correctly:
- `ledger#76` — open: confirmed (casehubio/ledger#76)
- `devtown#19` — open: confirmed (tracking ledger#76)
- `devtown#20` — open: confirmed (CaseOperation naming)
- `parent#14` — open: confirmed (trust maturity model platform pattern)

No broken references.

- [ ] **Step 4: Check spec cross-reference**

Confirm the spec (`specs/2026-05-08-epic2-domain-model-design.md` in workspace) references `parent#14` (not #13). Verify this is correct. No update needed if the reference is already `#14`.

- [ ] **Step 5: Commit documentation updates**

```bash
git add docs/PROGRESS.md docs/gastown-casehub-analysis-v2.md
git commit -m "docs: mark DT-001 through DT-006 as Implemented — Epic 2 shipped

Refs #9"
```

- [ ] **Step 6: Close Epic 2 issue**

```bash
git commit --allow-empty -m "chore: Epic 2 complete — domain model implemented and tested

Closes #9"
```
