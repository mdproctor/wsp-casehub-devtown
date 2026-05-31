# Layer 6 — Trust-Weighted Reviewer Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `TrustWeightedAgentStrategy` from `casehub-engine-ledger` into devtown with per-capability routing policies backed by YAML configuration, so reviewer selection uses trust scores rather than load alone.

**Architecture:** `casehub-engine-ledger` activates by classpath presence (`@Alternative @Priority(1)`). `DevtownTrustRoutingPolicyProvider` (`@ApplicationScoped` in `app/`) reads threshold/minObs/borderlineMargin from `DevtownCapabilityRegistry` (existing domain layer) and supplements with blendFactor + quality floors from YAML via `casehub-platform-config`. Trust scores accumulate via `WorkerDecisionEventCapture` (also in engine-ledger); all agents are BOOTSTRAP (availability routing) until Layer 4 attestations exist.

**Tech Stack:** Java 21, Quarkus 3.32.2, `casehub-engine-ledger`, `casehub-platform-config`, SnakeYAML (transitive via platform-config), JUnit 5, `@QuarkusTest`

**Spec:** `docs/specs/2026-05-30-layer6-trust-routing-design.md`  
**Issue:** devtown#57  
**Build:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

### Task 1: trust-maturity-model.md protocol

**Files:**
- Create: `../parent/docs/protocols/casehub/trust-maturity-model.md` (casehub-parent repo — NOT devtown)

- [ ] **Step 1: Write the protocol file**

```markdown
---
id: trust-maturity-model
title: Trust Routing Cold-Start and Maturity Model
scope: casehub
applies-to: any application using TrustWeightedAgentStrategy from casehub-engine-ledger
---

## Rule

Every application that activates trust-based routing MUST implement the four-phase maturity
model. Never block on missing trust data. Phase 0 is Gastown parity — availability routing.

## The Four Phases

| Phase | Condition | Routing |
|---|---|---|
| 0/1 BOOTSTRAP | `decisionCount < minimumObservations` OR no CAPABILITY score | Availability: `1/(1+runningJobs)` |
| 2 QUALIFIED | score ≥ threshold AND NOT borderline AND passes quality floors | Blended: `trust×blendFactor + workload×(1-blendFactor)` |
| 2a BORDERLINE | `Math.abs(score - threshold) <= borderlineMargin` | Score 0.0; if ALL non-bootstrap candidates are borderline → `EscalateToOversight` |
| 3 EXCLUDED | score < threshold (Phase 2b) OR quality floor failed (Phase 3) | Score 0.0; not escalated |

## Bootstrap > Borderline

A BOOTSTRAP candidate (positive availability score) always outscores a BORDERLINE candidate
(score 0.0). New agents with no decision history are preferred over established agents with
borderline trust. Operators who want to prevent unknown agents from executing sensitive
operations must configure the Qhorus trust gate (`casehub.qhorus.commitment.min-obligor-trust`).

## Quality Floors

`qualityFloors` maps dimension name → minimum required score. If a candidate's score for a
dimension is present in the ledger AND below the floor, it is EXCLUDED_PHASE3. If the
dimension data is absent, no penalty is applied — absence does not count as failure.

## Dimension Convention

All trust dimensions MUST be stored as higher = better (0.0–1.0). Dimensions with inverted
natural semantics (e.g. false-positive-rate) MUST be stored as their complement (precision =
1 - FPR). Storing a dimension as higher = worse makes quality floor logic semantically
inverted and silently wrong.

## Consumer Obligations

1. Declare a `RoutingPolicy` for every trust-sensitive capability in the domain registry.
2. Every capability with a routing policy MUST declare a `fallbackType` (the human oversight
   type to escalate to when the all-borderline pool condition is met).
3. Never hard-code trust thresholds — use YAML config via `casehub-platform-config` +
   `PreferenceKey` per field. See `DevtownTrustRoutingPolicyProvider` as the reference impl.

## Reference

- `TrustWeightedAgentStrategy` in `casehub-engine-ledger`
- `TrustCandidateClassifier` — classification and outcome decision logic
- `TrustRoutingPolicy` — the policy record (`threshold`, `minimumObservations`,
  `borderlineMargin`, `blendFactor`, `qualityFloors`)
- devtown#57 — first consumer; reference implementation of `TrustRoutingPolicyProvider`
```

- [ ] **Step 2: Commit to casehub-parent**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/protocols/casehub/trust-maturity-model.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: add trust-maturity-model protocol (refs parent#116)"
```

---

### Task 2: Rename DevtownTrustDimension.FALSE_POSITIVE_RATE → PRECISION

**Files:**
- Modify: `domain/src/test/java/io/casehub/devtown/domain/DevtownTrustDimensionTest.java`
- Modify: `domain/src/main/java/io/casehub/devtown/domain/DevtownTrustDimension.java`

- [ ] **Step 1: Update the test first (it will fail to compile)**

Replace in `DevtownTrustDimensionTest.java`:
```java
// Replace these two lines:
assertThat(DevtownTrustDimension.FALSE_POSITIVE_RATE).isNotBlank();
// and:
assertThat(DevtownTrustDimension.FALSE_POSITIVE_RATE).isEqualTo("false-positive-rate");
// and in the allValues list:
DevtownTrustDimension.FALSE_POSITIVE_RATE,

// With:
assertThat(DevtownTrustDimension.PRECISION).isNotBlank();
// and:
assertThat(DevtownTrustDimension.PRECISION).isEqualTo("precision"); // precision = TP/(TP+FP); stored as higher = better, unlike raw FPR
// and in the allValues list:
DevtownTrustDimension.PRECISION,
```

- [ ] **Step 2: Verify test fails to compile (constant doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl domain 2>&1 | grep -E "error:|ERROR"
```
Expected: compilation error referencing `FALSE_POSITIVE_RATE` or `PRECISION`.

- [ ] **Step 3: Rename the constant in DevtownTrustDimension.java**

```java
public final class DevtownTrustDimension {
    public static final String REVIEW_THOROUGHNESS = "review-thoroughness";
    public static final String PRECISION           = "precision"; // TP/(TP+FP); higher = better
    public static final String SCOPE_CALIBRATION   = "scope-calibration";

    private DevtownTrustDimension() {}
}
```

- [ ] **Step 4: Verify test passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=DevtownTrustDimensionTest -q
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/DevtownTrustDimension.java domain/src/test/java/io/casehub/devtown/domain/DevtownTrustDimensionTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "refactor: rename TrustDimension.FALSE_POSITIVE_RATE to PRECISION (Refs #57)"
```

---

### Task 3: Deprecate RoutingPolicy.isBorderline()

**Files:**
- Modify: `domain/src/main/java/io/casehub/devtown/domain/RoutingPolicy.java`

- [ ] **Step 1: Add @Deprecated and explanatory comment**

```java
/**
 * @deprecated Never called by the routing path. The engine uses
 *   {@link io.casehub.api.spi.routing.TrustRoutingPolicy#isBorderline(double)},
 *   which has symmetric semantics (Math.abs). This implementation is one-sided
 *   (score >= threshold && score < threshold + margin) and will be removed.
 */
@Deprecated
public boolean isBorderline(double trustScore) {
    return borderlineMargin.isPresent() && threshold.isPresent()
        && trustScore >= threshold.getAsDouble()
        && trustScore < threshold.getAsDouble() + borderlineMargin.getAsDouble();
}
```

- [ ] **Step 2: Verify compile succeeds**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl domain -q
```
Expected: BUILD SUCCESS (deprecation warnings are fine; errors are not)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/RoutingPolicy.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "deprecate: RoutingPolicy.isBorderline() — dead code, wrong semantics (Refs #57)"
```

---

### Task 4: Create DoublePreference in domain/trust/

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/DoublePreference.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/trust/DoublePreferenceTest.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain.trust;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DoublePreferenceTest {

    @Test
    void ofPreservesValue() {
        assertThat(DoublePreference.of(0.70).value()).isEqualTo(0.70);
    }

    @Test
    void parseConvertsString() {
        assertThat(DoublePreference.parse("0.42").value()).isEqualTo(0.42);
    }

    @Test
    void parseRejectsNull() {
        assertThatThrownBy(() -> DoublePreference.parse(null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void parseRejectsNonNumeric() {
        assertThatThrownBy(() -> DoublePreference.parse("not-a-number"))
            .isInstanceOf(NumberFormatException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails (class doesn't exist)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=DoublePreferenceTest 2>&1 | tail -5
```
Expected: compilation error.

- [ ] **Step 3: Implement DoublePreference**

```java
package io.casehub.devtown.domain.trust;

import io.casehub.platform.api.preferences.SingleValuePreference;
import java.util.Objects;

public record DoublePreference(double value) implements SingleValuePreference {

    public static DoublePreference of(double value) {
        return new DoublePreference(value);
    }

    public static DoublePreference parse(String raw) {
        Objects.requireNonNull(raw, "raw must not be null");
        return new DoublePreference(Double.parseDouble(raw));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=DoublePreferenceTest -q
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/trust/DoublePreference.java domain/src/test/java/io/casehub/devtown/domain/trust/DoublePreferenceTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: DoublePreference in domain/trust — SingleValuePreference for double fields (Refs #57)"
```

---

### Task 5: Create TrustRoutingPolicyKeys

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/TrustRoutingPolicyKeys.java`

`TrustRoutingPolicyKeys` holds the `PreferenceKey` statics for the YAML-backed fields only
(blendFactor and quality floors). threshold/minObs/borderlineMargin come from the capability
registry, not from YAML.

`PreferenceKey.qualifiedName()` = `namespace + "." + name`. The YAML keys must match exactly.

`MapPreferences.get()` returns `null` when a key is absent (does NOT fall back to the
PreferenceKey's defaultValue). Floor keys use `DoublePreference.of(0.0)` as a non-null
constructor sentinel only — the provider checks `null` from `get()` independently.

- [ ] **Step 1: Create TrustRoutingPolicyKeys**

```java
package io.casehub.devtown.domain.trust;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.platform.api.preferences.PreferenceKey;

/**
 * PreferenceKey constants for trust routing YAML configuration.
 * Resolved at scope: casehubio/devtown/trust-routing/<capabilityName>
 *
 * threshold, minimumObservations, and borderlineMargin come from DevtownCapabilityRegistry —
 * these keys cover only the engine-specific fields not present in RoutingPolicy.
 */
public final class TrustRoutingPolicyKeys {

    /** Weight of trust score vs workload (0.0 = pure workload, 1.0 = pure trust). */
    public static final PreferenceKey<DoublePreference> BLEND_FACTOR =
        new PreferenceKey<>(
            "casehubio.devtown.trust-routing",
            "blend-factor",
            DoublePreference.of(TrustRoutingPolicy.DEFAULT.blendFactor()),
            DoublePreference::parse);

    /** Minimum review-thoroughness dimension score (higher = finds more real issues). 0.0 = no floor. */
    public static final PreferenceKey<DoublePreference> FLOOR_REVIEW_THOROUGHNESS =
        new PreferenceKey<>(
            "casehubio.devtown.trust-routing",
            "floor.review-thoroughness",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    /** Minimum precision dimension score (higher = fewer false positives). 0.0 = no floor. */
    public static final PreferenceKey<DoublePreference> FLOOR_PRECISION =
        new PreferenceKey<>(
            "casehubio.devtown.trust-routing",
            "floor.precision",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    /** Minimum scope-calibration dimension score (higher = better DECLINE accuracy). 0.0 = no floor. */
    public static final PreferenceKey<DoublePreference> FLOOR_SCOPE_CALIBRATION =
        new PreferenceKey<>(
            "casehubio.devtown.trust-routing",
            "floor.scope-calibration",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    private TrustRoutingPolicyKeys() {}
}
```

- [ ] **Step 2: Verify compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl domain -q
```
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/trust/TrustRoutingPolicyKeys.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: TrustRoutingPolicyKeys — PreferenceKey constants for trust routing YAML (Refs #57)"
```

---

### Task 6: Add Maven dependencies and YAML configuration

**Files:**
- Modify: `app/pom.xml`
- Create: `app/src/main/resources/casehub/devtown/trust-routing.yaml`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1: Add deps to app/pom.xml after the casehub-platform block (line ~73)**

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-ledger</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-config</artifactId>
    </dependency>
```

- [ ] **Step 2: Create trust-routing.yaml**

Create `app/src/main/resources/casehub/devtown/trust-routing.yaml`:

```yaml
# devtown trust routing — blendFactor and quality floors only.
# threshold / minimumObservations / borderlineMargin come from DevtownCapabilityRegistry.
# All dimension scores 0.0–1.0 where higher = better (industry-standard trust dimension convention).
entries:
  # security-review: trust dominates (0.70 blend); agents missing real issues are excluded
  - scope: casehubio/devtown/trust-routing/security-review
    casehubio.devtown.trust-routing.blend-factor: "0.70"
    casehubio.devtown.trust-routing.floor.review-thoroughness: "0.60"

  # architecture-review: trust dominates; agents missing structural issues are excluded
  - scope: casehubio/devtown/trust-routing/architecture-review
    casehubio.devtown.trust-routing.blend-factor: "0.70"
    casehubio.devtown.trust-routing.floor.review-thoroughness: "0.60"

  # style-review: baseline — workload/trust balanced; no quality floor (low-stakes)
  - scope: casehubio/devtown/trust-routing/style-review
    casehubio.devtown.trust-routing.blend-factor: "0.50"

  # merge-executor: trust heavily dominant (0.80 blend); agents with poor precision excluded
  # precision = TP/(TP+FP); floor 0.70 ensures merge executors rarely block valid merges
  - scope: casehubio/devtown/trust-routing/merge-executor
    casehubio.devtown.trust-routing.blend-factor: "0.80"
    casehubio.devtown.trust-routing.floor.precision: "0.70"
```

- [ ] **Step 3: Add to app/src/main/resources/application.properties**

Append to end of file:

```properties
# ── Trust routing (Layer 6) ─────────────────────────────────────────────────
casehub.platform.config.files=classpath:casehub/devtown/trust-routing.yaml
# Engine-ledger Flyway migrations (V2000 case_ledger_entry, V2001 worker_decision_entry)
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/engine-ledger/migration
# Jandex: engine-ledger lacks embedded index (mirrors engine/engine-common pattern)
%prod.quarkus.index-dependency.engine-ledger.group-id=io.casehub
%prod.quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

- [ ] **Step 4: Add Jandex to app/src/test/resources/application.properties**

Append to end of file:

```properties
# Jandex: engine-ledger — required for TrustWeightedAgentStrategy and WorkerDecisionEventCapture CDI discovery
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

- [ ] **Step 5: Verify app compiles with new deps**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -q
```
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/pom.xml app/src/main/resources/casehub/devtown/trust-routing.yaml app/src/main/resources/application.properties app/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/devtown commit -m "chore: add engine-ledger + platform-config deps and trust routing YAML (Refs #57)"
```

---

### Task 7: Implement DevtownTrustRoutingPolicyProvider (TDD)

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProvider.java`
- Create: `app/src/test/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProviderTest.java`

- [ ] **Step 1: Create stub provider (compiles, returns DEFAULT for all)**

```java
package io.casehub.devtown.app.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.devtown.domain.spi.CapabilityRegistry;
import io.casehub.platform.api.preferences.PreferenceProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class DevtownTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {

    private final PreferenceProvider preferenceProvider;
    private final CapabilityRegistry capabilityRegistry;

    @Inject
    public DevtownTrustRoutingPolicyProvider(
            PreferenceProvider preferenceProvider,
            CapabilityRegistry capabilityRegistry) {
        this.preferenceProvider = preferenceProvider;
        this.capabilityRegistry = capabilityRegistry;
    }

    @Override
    public TrustRoutingPolicy forCapability(String capabilityName) {
        return TrustRoutingPolicy.DEFAULT; // stub — will implement next
    }
}
```

- [ ] **Step 2: Write the failing unit test**

```java
package io.casehub.devtown.app.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.devtown.domain.DevtownCapabilityRegistry;
import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.domain.AgentQualification;
import io.casehub.platform.api.preferences.MapPreferences;
import io.casehub.platform.api.preferences.PreferenceProvider;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class DevtownTrustRoutingPolicyProviderTest {

    // Preference provider that returns empty preferences for all scopes
    private static final PreferenceProvider EMPTY = scope -> new MapPreferences(Map.of());

    // Preference provider that returns specific values (simulates YAML-loaded prefs)
    private static final PreferenceProvider POPULATED = scope ->
        new MapPreferences(Map.of(
            "casehubio.devtown.trust-routing.blend-factor", "0.42",
            "casehubio.devtown.trust-routing.floor.review-thoroughness", "0.88"
        ));

    private final DevtownCapabilityRegistry registry = new DevtownCapabilityRegistry();

    // === Fallback path: EMPTY prefs — values come from registry only ===

    @Test
    void securityReviewUsesRegistryThreshold() {
        var provider = new DevtownTrustRoutingPolicyProvider(EMPTY, registry);
        TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.threshold()).isEqualTo(0.70);
    }

    @Test
    void securityReviewUsesRegistryMinObservations() {
        var provider = new DevtownTrustRoutingPolicyProvider(EMPTY, registry);
        TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.minimumObservations()).isEqualTo(10);
    }

    @Test
    void securityReviewUsesRegistryBorderlineMargin() {
        var provider = new DevtownTrustRoutingPolicyProvider(EMPTY, registry);
        TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.borderlineMargin()).isEqualTo(0.05);
    }

    @Test
    void unknownCapabilityReturnsDefault() {
        var provider = new DevtownTrustRoutingPolicyProvider(EMPTY, registry);
        TrustRoutingPolicy policy = provider.forCapability("unknown-capability");
        assertThat(policy).isEqualTo(TrustRoutingPolicy.DEFAULT);
    }

    @Test
    void noFloorInEmptyPrefs() {
        var provider = new DevtownTrustRoutingPolicyProvider(EMPTY, registry);
        TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.qualityFloors()).isEmpty();
    }

    // === Parsing path: POPULATED prefs — verifies field assembly ===

    @Test
    void blendFactorParsedFromPrefs() {
        var provider = new DevtownTrustRoutingPolicyProvider(POPULATED, registry);
        TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.blendFactor()).isEqualTo(0.42);
    }

    @Test
    void floorParsedFromPrefsAndAddedToMap() {
        var provider = new DevtownTrustRoutingPolicyProvider(POPULATED, registry);
        TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.qualityFloors()).containsEntry("review-thoroughness", 0.88);
    }

    @Test
    void zeroFloorValueNotAddedToMap() {
        var provider = new DevtownTrustRoutingPolicyProvider(
            scope -> new MapPreferences(Map.of(
                "casehubio.devtown.trust-routing.floor.precision", "0.0"
            )),
            registry);
        TrustRoutingPolicy policy = provider.forCapability(AgentQualification.MERGE_EXECUTOR);
        assertThat(policy.qualityFloors()).doesNotContainKey("precision");
    }

    @Test
    void allSixCapabilitiesResolveWithoutException() {
        var provider = new DevtownTrustRoutingPolicyProvider(EMPTY, registry);
        assertThat(provider.forCapability(ReviewDomain.CODE_ANALYSIS)).isNotNull();
        assertThat(provider.forCapability(ReviewDomain.SECURITY_REVIEW)).isNotNull();
        assertThat(provider.forCapability(ReviewDomain.ARCHITECTURE_REVIEW)).isNotNull();
        assertThat(provider.forCapability(ReviewDomain.STYLE_REVIEW)).isNotNull();
        assertThat(provider.forCapability(ReviewDomain.TEST_COVERAGE)).isNotNull();
        assertThat(provider.forCapability(ReviewDomain.PERFORMANCE_ANALYSIS)).isNotNull();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail (stub returns DEFAULT)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownTrustRoutingPolicyProviderTest 2>&1 | grep -E "FAIL|ERROR|Tests run"
```
Expected: multiple failures — threshold/minObs/borderlineMargin/blendFactor assertions fail against DEFAULT values.

- [ ] **Step 4: Implement the full provider**

```java
package io.casehub.devtown.app.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.devtown.domain.DevtownTrustDimension;
import io.casehub.devtown.domain.RoutingPolicy;
import io.casehub.devtown.domain.spi.CapabilityRegistry;
import io.casehub.devtown.domain.trust.DoublePreference;
import io.casehub.devtown.domain.trust.TrustRoutingPolicyKeys;
import io.casehub.platform.api.preferences.PreferenceKey;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

/**
 * Devtown-specific {@link TrustRoutingPolicyProvider}.
 *
 * <p>Reads threshold/minimumObservations/borderlineMargin from {@link CapabilityRegistry}
 * (single source of truth in the domain layer). Reads blendFactor and quality floors from
 * YAML config via {@link PreferenceProvider} at scope
 * {@code casehubio/devtown/trust-routing/<capabilityName>}.
 *
 * <p>{@code @ApplicationScoped} (no {@code @DefaultBean}) displaces
 * {@code DefaultTrustRoutingPolicyProvider @DefaultBean} automatically.
 */
@ApplicationScoped
public class DevtownTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {

    private final PreferenceProvider preferenceProvider;
    private final CapabilityRegistry capabilityRegistry;

    @Inject
    public DevtownTrustRoutingPolicyProvider(
            final PreferenceProvider preferenceProvider,
            final CapabilityRegistry capabilityRegistry) {
        this.preferenceProvider = preferenceProvider;
        this.capabilityRegistry = capabilityRegistry;
    }

    @Override
    public TrustRoutingPolicy forCapability(final String capabilityName) {
        final Optional<RoutingPolicy> rp = capabilityRegistry.policy(capabilityName);
        if (rp.isEmpty()) {
            return TrustRoutingPolicy.DEFAULT;
        }

        final RoutingPolicy routingPolicy = rp.get();
        final Preferences prefs = preferenceProvider.resolve(
            SettingsScope.of("casehubio", "devtown", "trust-routing", capabilityName));

        final double threshold = routingPolicy.threshold()
            .orElse(TrustRoutingPolicy.DEFAULT.threshold());
        final int minimumObservations = routingPolicy.minimumObservations()
            .orElse(TrustRoutingPolicy.DEFAULT.minimumObservations());
        final double borderlineMargin = routingPolicy.borderlineMargin()
            .orElse(TrustRoutingPolicy.DEFAULT.borderlineMargin());

        final DoublePreference blendFactorPref = prefs.get(TrustRoutingPolicyKeys.BLEND_FACTOR);
        final double blendFactor = blendFactorPref != null
            ? blendFactorPref.value()
            : TrustRoutingPolicy.DEFAULT.blendFactor();

        final Map<String, Double> qualityFloors = new HashMap<>();
        addFloor(qualityFloors, prefs, TrustRoutingPolicyKeys.FLOOR_REVIEW_THOROUGHNESS,
            DevtownTrustDimension.REVIEW_THOROUGHNESS);
        addFloor(qualityFloors, prefs, TrustRoutingPolicyKeys.FLOOR_PRECISION,
            DevtownTrustDimension.PRECISION);
        addFloor(qualityFloors, prefs, TrustRoutingPolicyKeys.FLOOR_SCOPE_CALIBRATION,
            DevtownTrustDimension.SCOPE_CALIBRATION);

        return new TrustRoutingPolicy(threshold, minimumObservations, borderlineMargin,
            blendFactor, Map.copyOf(qualityFloors));
    }

    private static void addFloor(final Map<String, Double> floors, final Preferences prefs,
            final PreferenceKey<DoublePreference> key, final String dimension) {
        final DoublePreference value = prefs.get(key);
        if (value != null && value.value() > 0.0) {
            floors.put(dimension, value.value());
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownTrustRoutingPolicyProviderTest -q
```
Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProvider.java app/src/test/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProviderTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: DevtownTrustRoutingPolicyProvider — YAML-backed per-capability trust routing (Refs #57)"
```

---

### Task 8: Write and pass TrustRoutingActivationTest

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/TrustRoutingActivationTest.java`

This test verifies CDI wiring and proves the YAML file is actually loaded. The key assertion
is `style-review` threshold = 0.50, which differs from `TrustRoutingPolicy.DEFAULT.threshold()`
(0.70) — if YAML loading fails, the default fallback makes other assertions pass coincidentally.

- [ ] **Step 1: Write the test**

```java
package io.casehub.devtown.app;

import io.casehub.api.spi.routing.AgentRoutingStrategy;
import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.devtown.app.routing.DevtownTrustRoutingPolicyProvider;
import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.domain.AgentQualification;
import io.casehub.ledger.routing.TrustWeightedAgentStrategy;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class TrustRoutingActivationTest {

    @Inject
    AgentRoutingStrategy agentRoutingStrategy;

    @Inject
    TrustRoutingPolicyProvider policyProvider;

    @Test
    void trustWeightedStrategyActivated() {
        assertThat(agentRoutingStrategy).isInstanceOf(TrustWeightedAgentStrategy.class);
    }

    @Test
    void devtownProviderActivated() {
        assertThat(policyProvider).isInstanceOf(DevtownTrustRoutingPolicyProvider.class);
    }

    @Test
    void styleReviewThresholdProvesYamlLoaded() {
        // style-review threshold = 0.50 from registry; DEFAULT.threshold() = 0.70.
        // If YAML fails to load, blendFactor falls to DEFAULT but threshold still comes
        // from registry — this confirms registry reads are working.
        // The critical proof: if DevtownTrustRoutingPolicyProvider isn't wired, we'd get
        // DEFAULT (0.70). Getting 0.50 proves the provider is active.
        TrustRoutingPolicy policy = policyProvider.forCapability(ReviewDomain.STYLE_REVIEW);
        assertThat(policy.threshold()).isEqualTo(0.50);
    }

    @Test
    void architectureReviewThresholdDiffersFromDefault() {
        // architecture-review threshold = 0.65; DEFAULT = 0.70
        TrustRoutingPolicy policy = policyProvider.forCapability(ReviewDomain.ARCHITECTURE_REVIEW);
        assertThat(policy.threshold()).isEqualTo(0.65);
        assertThat(policy.threshold()).isNotEqualTo(TrustRoutingPolicy.DEFAULT.threshold());
    }

    @Test
    void securityReviewHasThoroughnessFloor() {
        TrustRoutingPolicy policy = policyProvider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.qualityFloors()).containsEntry("review-thoroughness", 0.60);
    }

    @Test
    void mergeExecutorHasPrecisionFloor() {
        TrustRoutingPolicy policy = policyProvider.forCapability(AgentQualification.MERGE_EXECUTOR);
        assertThat(policy.qualityFloors()).containsEntry("precision", 0.70);
    }

    @Test
    void securityReviewBlendFactorFromYaml() {
        // blend-factor for security-review = 0.70 (YAML); DEFAULT.blendFactor() = 0.60
        TrustRoutingPolicy policy = policyProvider.forCapability(ReviewDomain.SECURITY_REVIEW);
        assertThat(policy.blendFactor()).isEqualTo(0.70);
        assertThat(policy.blendFactor()).isNotEqualTo(TrustRoutingPolicy.DEFAULT.blendFactor());
    }
}
```

- [ ] **Step 2: Run the test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TrustRoutingActivationTest -q
```
Expected: BUILD SUCCESS, all tests pass.

If `TrustWeightedAgentStrategy` CDI injection fails: the engine-ledger Jandex entry is likely
missing or the `%prod.` prefix on the Jandex config is blocking test discovery. Check
`app/src/test/resources/application.properties` has the non-prefixed Jandex entry added in Task 6 Step 4.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/TrustRoutingActivationTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test: TrustRoutingActivationTest — verifies CDI wiring and YAML load (Refs #57)"
```

---

### Task 9: Full build verification and final commit

- [ ] **Step 1: Run the full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -20
```
Expected: BUILD SUCCESS, all modules green.

- [ ] **Step 2: Confirm test counts include new tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | grep "Tests run:" | tail -10
```
Expected: `DevtownTrustDimensionTest`, `DoublePreferenceTest`, `DevtownTrustRoutingPolicyProviderTest`, `TrustRoutingActivationTest` all appear with 0 failures.

- [ ] **Step 3: Push project branch**

```bash
git -C /Users/mdproctor/claude/casehub/devtown push --set-upstream origin issue-57-layer6-trust-routing
```
