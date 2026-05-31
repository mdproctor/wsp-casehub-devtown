# Trust Gate Implementation Plan (devtown#58)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `DevtownObligorTrustPolicy` as the active `ObligorTrustPolicy` SPI bean, implementing a global trust floor with bootstrap exemption, backed by YAML config.

**Architecture:** `DevtownObligorTrustPolicy @ApplicationScoped` displaces `DefaultObligorTrustPolicy @DefaultBean` via CDI priority. It reads a global floor from `trust-gate.yaml` via `PreferenceProvider`, and uses `TrustGateService.currentScore()` returning `Optional.empty()` as the bootstrap detection signal. The gate is a safety net for agents with persistently poor recorded history — not merge-executor bootstrap protection (that is devtown#62).

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-qhorus-api (`ObligorTrustPolicy`, `ObligorTrustContext`), casehub-ledger (`TrustGateService`), casehub-platform-api (`PreferenceProvider`, `MapPreferences`), JUnit 5, AssertJ.

**Spec:** `specs/2026-05-31-trust-gate-design.md`

---

## File Map

| Action | Path | Responsibility |
|--------|------|---------------|
| Create | `domain/src/main/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeys.java` | PreferenceKey constant for YAML scope `casehubio/devtown/trust-gate` |
| Create | `domain/src/test/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeysTest.java` | Verify key scope, field name, default value |
| Create | `app/src/test/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicyTest.java` | 6 pure-Java unit tests for all branch paths |
| Create | `app/src/main/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicy.java` | @ApplicationScoped ObligorTrustPolicy implementation |
| Create | `app/src/main/resources/casehub/devtown/trust-gate.yaml` | YAML: global trust floor 0.30 |
| Modify | `app/src/main/resources/application.properties` | Add trust-gate.yaml to config files list |
| Create | `app/src/test/java/io/casehub/devtown/app/TrustGateWiringTest.java` | 4 @QuarkusTest cases: CDI wiring, bootstrap, below-threshold, global score assumption |

---

## Task 1: `TrustGatePreferenceKeys` constant

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeys.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeysTest.java`

- [ ] **Step 1: Write the failing test**

```java
// domain/src/test/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeysTest.java
package io.casehub.devtown.domain.trust;

import io.casehub.devtown.domain.preferences.DoublePreference;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TrustGatePreferenceKeysTest {

    @Test
    void minObligorTrustHasCorrectScope() {
        assertThat(TrustGatePreferenceKeys.MIN_OBLIGOR_TRUST.scope())
            .isEqualTo("casehubio.devtown.trust-gate");
    }

    @Test
    void minObligorTrustHasCorrectField() {
        assertThat(TrustGatePreferenceKeys.MIN_OBLIGOR_TRUST.field())
            .isEqualTo("min-obligor-trust");
    }

    @Test
    void minObligorTrustDefaultIsZero() {
        assertThat(TrustGatePreferenceKeys.MIN_OBLIGOR_TRUST.defaultValue())
            .isEqualTo(DoublePreference.of(0.0));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=TrustGatePreferenceKeysTest --batch-mode -q 2>&1 | tail -5
```

Expected: FAIL — `TrustGatePreferenceKeys` does not exist yet.

- [ ] **Step 3: Implement `TrustGatePreferenceKeys`**

```java
// domain/src/main/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeys.java
package io.casehub.devtown.domain.trust;

import io.casehub.devtown.domain.preferences.DoublePreference;
import io.casehub.platform.api.preferences.PreferenceKey;

public final class TrustGatePreferenceKeys {

    /**
     * Global trust floor for non-bootstrap agents. 0.0 = gate disabled.
     * Note: constructor default is unused at runtime — MapPreferences.get()
     * returns null for absent keys. See null guard in DevtownObligorTrustPolicy.
     */
    public static final PreferenceKey<DoublePreference> MIN_OBLIGOR_TRUST =
        new PreferenceKey<>(
            "casehubio.devtown.trust-gate",
            "min-obligor-trust",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    private TrustGatePreferenceKeys() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=TrustGatePreferenceKeysTest --batch-mode -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Run all domain tests to confirm no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain --batch-mode 2>&1 | grep -E "Tests run|BUILD" | tail -5
```

Expected: all tests pass, `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeys.java domain/src/test/java/io/casehub/devtown/domain/trust/TrustGatePreferenceKeysTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: TrustGatePreferenceKeys — YAML preference constant for trust gate floor (Refs #58)"
```

---

## Task 2: `DevtownObligorTrustPolicy` unit tests (write before implementation)

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicyTest.java`

**Background:** `TrustGateService` is a concrete class (not an interface). The test subclasses it and overrides `currentScore()` to control the returned score. A stub `ActorTrustScoreRepository` satisfies the `TrustGateService` constructor without ever being called. `PreferenceProvider` is created as a lambda using `MapPreferences(Map.of(key, value))` — the same pattern used in `DevtownTrustRoutingPolicyProviderTest`. The map key for `MIN_OBLIGOR_TRUST` is `"casehubio.devtown.trust-gate.min-obligor-trust"` (scope + "." + field).

- [ ] **Step 1: Write the failing tests**

```java
// app/src/test/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicyTest.java
package io.casehub.devtown.app.routing;

import io.casehub.ledger.runtime.model.ActorTrustScore;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.preferences.MapPreferences;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.qhorus.api.spi.ObligorTrustContext;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class DevtownObligorTrustPolicyTest {

    // --- Trust gate config key (scope.field format for MapPreferences lookup) ---
    private static final String GATE_KEY = "casehubio.devtown.trust-gate.min-obligor-trust";

    // --- Preference providers ---

    /** No YAML entries — prefs.get() returns null for all keys. */
    private static final PreferenceProvider EMPTY_PROVIDER =
        scope -> new MapPreferences(Map.of());

    /** Threshold = 0.0 (explicit gate disabled). */
    private static final PreferenceProvider GATE_DISABLED =
        scope -> new MapPreferences(Map.of(GATE_KEY, "0.0"));

    /** Threshold = 0.30. */
    private static final PreferenceProvider GATE_030 =
        scope -> new MapPreferences(Map.of(GATE_KEY, "0.30"));

    // --- TrustGateService stubs ---

    /**
     * Returns a TrustGateService whose currentScore(actorId) returns the given value.
     * The stub repository satisfies the constructor without being called.
     */
    private static TrustGateService gateWith(Optional<Double> score) {
        return new TrustGateService(stubRepo()) {
            @Override
            public Optional<Double> currentScore(String actorId) {
                return score;
            }
        };
    }

    private static ActorTrustScoreRepository stubRepo() {
        return new ActorTrustScoreRepository() {
            @Override public Optional<ActorTrustScore> findByActorId(String a) { return Optional.empty(); }
            @Override public Optional<ActorTrustScore> findCapabilityScore(String a, String c) { return Optional.empty(); }
            @Override public Optional<ActorTrustScore> findDimensionScore(String a, String d) { return Optional.empty(); }
            @Override public Optional<ActorTrustScore> findCapabilityDimension(String a, String c, String d) { return Optional.empty(); }
            @Override public List<ActorTrustScore> findCapabilityDimensions(String a, String c) { return List.of(); }
            @Override public List<ActorTrustScore> findByActorIdAndScoreType(String a, io.casehub.ledger.api.model.ActorTrustScore.ScoreType t) { return List.of(); }
            @Override public void upsert(String a, io.casehub.ledger.api.model.ActorTrustScore.ScoreType t, String c, String d, ActorType at, double s, int w, int o, double m, double v, int h, int tot, Instant i) {}
            @Override public void updateGlobalTrustScore(String a, double s) {}
            @Override public List<ActorTrustScore> findAll() { return List.of(); }
            @Override public List<ActorTrustScore> findAllByLastComputedAtAfter(Instant i) { return List.of(); }
        };
    }

    // --- Context helper ---

    private static ObligorTrustContext ctx(String agentId) {
        return new ObligorTrustContext(agentId, UUID.randomUUID(), "pr-review-1/work");
    }

    // === Tests ===

    /**
     * Null pref path: EMPTY_PROVIDER → prefs.get(MIN_OBLIGOR_TRUST) returns null
     * → null guard sets floor = 0.0 → gate disabled → permits even a low-scored agent.
     */
    @Test
    void permitsWhenThresholdPrefNull_nullGuardPath() {
        var policy = new DevtownObligorTrustPolicy(gateWith(Optional.of(0.05)), EMPTY_PROVIDER);
        assertThat(policy.permits(ctx("agent-1"))).isTrue();
    }

    /**
     * Explicit 0.0 path: GATE_DISABLED → floor = 0.0 → same branch as null guard
     * but via a parsed zero value rather than null.
     */
    @Test
    void permitsWhenThresholdExplicitZero_explicitDisabledPath() {
        var policy = new DevtownObligorTrustPolicy(gateWith(Optional.of(0.05)), GATE_DISABLED);
        assertThat(policy.permits(ctx("agent-1"))).isTrue();
    }

    /**
     * Bootstrap path: currentScore returns Optional.empty() (no ledger observations) →
     * gate always permits regardless of configured threshold.
     */
    @Test
    void permitsBootstrapAgent_noLedgerObservations() {
        var policy = new DevtownObligorTrustPolicy(gateWith(Optional.empty()), GATE_030);
        assertThat(policy.permits(ctx("new-agent"))).isTrue();
    }

    @Test
    void permitsWhenScoreAboveFloor() {
        var policy = new DevtownObligorTrustPolicy(gateWith(Optional.of(0.50)), GATE_030);
        assertThat(policy.permits(ctx("agent-1"))).isTrue();
    }

    @Test
    void permitsWhenScoreAtFloor() {
        var policy = new DevtownObligorTrustPolicy(gateWith(Optional.of(0.30)), GATE_030);
        assertThat(policy.permits(ctx("agent-1"))).isTrue();
    }

    @Test
    void deniesWhenScoreBelowFloor() {
        var policy = new DevtownObligorTrustPolicy(gateWith(Optional.of(0.20)), GATE_030);
        assertThat(policy.permits(ctx("agent-1"))).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to confirm they fail with "class not found"**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownObligorTrustPolicyTest --batch-mode 2>&1 | grep -E "ERROR|cannot find|BUILD" | head -10
```

Expected: compilation error — `DevtownObligorTrustPolicy` does not exist yet.

---

## Task 3: `DevtownObligorTrustPolicy` implementation

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicy.java`

**CDI displacement:** `@ApplicationScoped` (without `@DefaultBean`) takes CDI priority over `DefaultObligorTrustPolicy @DefaultBean` — no `@Alternative @Priority` needed.

**Null guard:** `prefs.get(key)` returns `null` for absent YAML keys (not the `PreferenceKey` constructor default). Step 2 of `permits()` must null-check before reading `.value()`.

- [ ] **Step 1: Implement `DevtownObligorTrustPolicy`**

```java
// app/src/main/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicy.java
package io.casehub.devtown.app.routing;

import io.casehub.devtown.domain.preferences.DoublePreference;
import io.casehub.devtown.domain.trust.TrustGatePreferenceKeys;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import io.casehub.qhorus.api.spi.ObligorTrustContext;
import io.casehub.qhorus.api.spi.ObligorTrustPolicy;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

/**
 * Devtown trust gate — global trust floor for non-bootstrap agents.
 *
 * <p>Bootstrap agents (no ledger observations — {@link TrustGateService#currentScore} returns
 * {@link java.util.Optional#empty()}) are always permitted. Only agents with a recorded score
 * below the configured floor are denied.
 *
 * <p>{@code @ApplicationScoped} (no {@code @DefaultBean}) displaces
 * {@code DefaultObligorTrustPolicy @DefaultBean} via CDI priority.
 */
@ApplicationScoped
public class DevtownObligorTrustPolicy implements ObligorTrustPolicy {

    private final TrustGateService trustGateService;
    private final PreferenceProvider preferenceProvider;

    @Inject
    public DevtownObligorTrustPolicy(TrustGateService trustGateService,
                                      PreferenceProvider preferenceProvider) {
        this.trustGateService = trustGateService;
        this.preferenceProvider = preferenceProvider;
    }

    @Override
    public boolean permits(ObligorTrustContext ctx) {
        Preferences prefs = preferenceProvider.resolve(
            SettingsScope.of("casehubio", "devtown", "trust-gate"));

        DoublePreference thresholdPref = prefs.get(TrustGatePreferenceKeys.MIN_OBLIGOR_TRUST);
        double floor = thresholdPref != null ? thresholdPref.value() : 0.0;

        if (floor <= 0.0) return true;          // gate disabled

        return trustGateService.currentScore(ctx.obligorId())
            .map(score -> score >= floor)
            .orElse(true);                       // bootstrap — no observations → permit
    }
}
```

- [ ] **Step 2: Run unit tests to verify all 6 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownObligorTrustPolicyTest --batch-mode 2>&1 | grep -E "Tests run|BUILD" | tail -5
```

Expected: `Tests run: 6, Failures: 0, Errors: 0`, `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicy.java \
  app/src/test/java/io/casehub/devtown/app/routing/DevtownObligorTrustPolicyTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: DevtownObligorTrustPolicy — global trust gate with bootstrap exemption (Refs #58)"
```

---

## Task 4: YAML config and `application.properties`

**Files:**
- Create: `app/src/main/resources/casehub/devtown/trust-gate.yaml`
- Modify: `app/src/main/resources/application.properties`

- [ ] **Step 1: Create `trust-gate.yaml`**

Create directory `app/src/main/resources/casehub/devtown/` if it doesn't exist (it does — `trust-routing.yaml` is already there).

```yaml
# app/src/main/resources/casehub/devtown/trust-gate.yaml
# Global trust gate — minimum trust floor for non-bootstrap agents.
# Bootstrap agents (no ledger observations) are always exempt.
# 0.30 is a permissive production floor — a safety net, not a quality filter.
# Routing (Layer 6) handles per-capability quality; the gate handles the global floor.
entries:
  - scope: casehubio/devtown/trust-gate
    casehubio.devtown.trust-gate.min-obligor-trust: "0.30"
```

- [ ] **Step 2: Add `trust-gate.yaml` to `application.properties`**

Find the existing line:
```
casehub.platform.config.files=classpath:casehub/devtown/trust-routing.yaml
```

Replace with:
```
casehub.platform.config.files=classpath:casehub/devtown/trust-routing.yaml,classpath:casehub/devtown/trust-gate.yaml
```

- [ ] **Step 3: Compile to verify no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app --batch-mode -q 2>&1 | tail -5
```

Expected: `BUILD SUCCESS` (no compilation errors).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/resources/casehub/devtown/trust-gate.yaml \
  app/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/devtown commit -m "config: trust-gate.yaml — global trust floor 0.30 via YAML (Refs #58)"
```

---

## Task 5: `TrustGateWiringTest` integration test

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/TrustGateWiringTest.java`

**Context for test setup:**
- `TrustGateService` is a CDI bean, already discoverable in `@QuarkusTest` (confirmed: `TrustWeightedAgentStrategy` injects it and `TrustRoutingActivationTest` passes).
- `ActorTrustScoreRepository` is the JPA repo for trust scores; its schema lives in the qhorus H2 in-memory datasource (test properties: `%test.quarkus.hibernate-orm.qhorus.database.generation=drop-and-create` creates all tables including `actor_trust_score`).
- `repository.updateGlobalTrustScore(actorId, score)` writes a global trust score row directly — use this to seed the below-threshold test case.
- Test case 4 calls `trustGateService.currentScore(agentId)` directly (inject `TrustGateService` as a separate field alongside the policy) — this is an `@ApplicationScoped` bean and resolves cleanly.
- Each test uses a unique agent ID to avoid interference between test cases.

- [ ] **Step 1: Write the integration test**

```java
// app/src/test/java/io/casehub/devtown/app/TrustGateWiringTest.java
package io.casehub.devtown.app;

import io.casehub.devtown.app.routing.DevtownObligorTrustPolicy;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.qhorus.api.spi.ObligorTrustContext;
import io.casehub.qhorus.api.spi.ObligorTrustPolicy;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class TrustGateWiringTest {

    @Inject ObligorTrustPolicy trustPolicy;
    @Inject TrustGateService trustGateService;
    @Inject ActorTrustScoreRepository trustScoreRepository;

    private static ObligorTrustContext ctx(String agentId) {
        return new ObligorTrustContext(agentId, UUID.randomUUID(), "pr-review-1/work");
    }

    /**
     * Case 1: CDI displacement — DevtownObligorTrustPolicy must be the active bean,
     * not DefaultObligorTrustPolicy.
     */
    @Test
    void devtownPolicyIsActiveBean() {
        assertThat(trustPolicy).isInstanceOf(DevtownObligorTrustPolicy.class);
    }

    /**
     * Case 2: Bootstrap path — agent with no trust score rows must be permitted.
     * Verifies Optional.empty() → permit semantics end-to-end.
     */
    @Test
    void permitsBootstrapAgent_noScoreRows() {
        String agentId = "bootstrap-agent-" + UUID.randomUUID();
        // No rows inserted — agent is genuinely unseen by the ledger
        assertThat(trustPolicy.permits(ctx(agentId))).isTrue();
    }

    /**
     * Case 3: Below-threshold non-bootstrap agent must be denied.
     * Seeds a global trust score of 0.15 (below the 0.30 YAML floor) and verifies rejection.
     */
    @Test
    void deniesNonBootstrapAgentBelowFloor() {
        String agentId = "low-trust-agent-" + UUID.randomUUID();
        trustScoreRepository.updateGlobalTrustScore(agentId, 0.15);
        assertThat(trustPolicy.permits(ctx(agentId))).isFalse();
    }

    /**
     * Case 4: Global score assumption check — verifies that TrustGateService.currentScore()
     * returns Optional.empty() for a completely unseen agent (no rows in ActorTrustScore).
     *
     * If TrustScoreJob only writes per-capability rows and never writes a global aggregate,
     * currentScore() would always return empty, making the gate permanently dormant.
     * This test confirms the empty-for-unseen-agent contract holds.
     * A separate end-to-end test (once Layer 4 attestation flow is wired) should verify
     * that currentScore() returns a non-empty value after attestations are processed.
     */
    @Test
    void currentScoreReturnsEmptyForUnseenAgent() {
        String agentId = "unseen-agent-" + UUID.randomUUID();
        Optional<Double> score = trustGateService.currentScore(agentId);
        assertThat(score).isEmpty();
    }
}
```

- [ ] **Step 2: Run the integration test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TrustGateWiringTest --batch-mode 2>&1 | grep -E "Tests run|ERROR|FAIL|BUILD" | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`, `BUILD SUCCESS`.

**If Case 3 fails** with `UnsatisfiedResolutionException` for `ActorTrustScoreRepository`: add the ledger indexing entry to `app/src/test/resources/application.properties`:
```properties
quarkus.index-dependency.casehub-ledger.group-id=io.casehub
quarkus.index-dependency.casehub-ledger.artifact-id=casehub-ledger
```

**If Case 3 fails** with `Table not found: ACTOR_TRUST_SCORE`: the qhorus PU packages already include `io.casehub.ledger.runtime.model`, so the entity should be discovered. If not, verify `quarkus.hibernate-orm.qhorus.packages` in `application.properties` includes `io.casehub.ledger.runtime.model`.

- [ ] **Step 3: Run the full app test suite to confirm no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode 2>&1 | grep -E "Tests run|BUILD" | tail -5
```

Expected: all tests pass (currently 35 pass; new tests add 4 integration + 6 unit = 45 total after Tasks 2–5), `BUILD SUCCESS`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/test/java/io/casehub/devtown/app/TrustGateWiringTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test: TrustGateWiringTest — CDI wiring, bootstrap, below-threshold, global score assumption (Refs #58)"
```

---

## Task 6: Final verification and issue close

- [ ] **Step 1: Run all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test --batch-mode 2>&1 | grep -E "Tests run|BUILD" | tail -10
```

Expected: all modules pass, `BUILD SUCCESS`.

- [ ] **Step 2: Close devtown#58**

```bash
gh issue close 58 --repo casehubio/devtown --comment "Implemented: DevtownObligorTrustPolicy @ApplicationScoped displaces DefaultObligorTrustPolicy. Global trust floor configured at 0.30 via trust-gate.yaml. Bootstrap agents (no ledger observations) always permitted. 10 tests: 6 unit + 4 integration."
```

- [ ] **Step 3: Final commit (if any uncommitted changes remain)**

```bash
git -C /Users/mdproctor/claude/casehub/devtown status
```

All changes should already be committed from Tasks 1–5.
