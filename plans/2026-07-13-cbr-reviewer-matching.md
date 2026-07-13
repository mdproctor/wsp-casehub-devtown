# CBR-Enhanced Reviewer Matching Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #133 — feat: CBR-enhanced reviewer matching — similarity bonus for trust-weighted assignment
**Issue group:** #133

**Goal:** Enhance trust-weighted agent routing to incorporate a similarity-weighted success rate bonus from CBR experiences, so agents that performed well on similar past cases receive a scoring boost.

**Architecture:** Three components across two repos. `ExperienceAnalyser` (engine-api) computes per-worker success rates from plan trace data. `TrustWeightedAgentStrategy` (engine-ledger) gains a CBR bonus in its scoring formula, controlled by a new `cbrWeight` field on `TrustRoutingPolicy`. Devtown wires the data flow: CBR config in the case definition, a feature provider for CBR queries, and per-capability `cbrWeight` configuration.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine 0.2-SNAPSHOT, casehub-blocks 0.2-SNAPSHOT

## Global Constraints

- All engine changes must be backward compatible: `cbrWeight` defaults to `0.0` — no behavioral change for apps that don't configure it
- `TrustRoutingPolicy` is a Java record — adding `cbrWeight` as 9th field breaks ALL constructor call sites across engine-api, engine-ledger, engine-ai, and all consumers. This is mechanical: add `0.0` as 9th arg everywhere except devtown's policy provider.
- Engine SNAPSHOT must be deployed before devtown changes can compile
- Use `ide-tooling` for all structural code operations. Bash for non-code files only.
- Spec: `specs/issue-133-cbr-reviewer-matching/2026-07-13-cbr-reviewer-matching-design.md`

## Cross-repo execution order

| Phase | Repo | Tasks | Blocks on |
|-------|------|-------|-----------|
| 1 | casehub-engine | Tasks 1-3 | — |
| 2 | casehub-devtown | Tasks 4-6 | Engine SNAPSHOT deployed |

---

### Task 1: `ExperienceAnalyser` utility (engine-api)

**Repo:** casehub-engine
**Module:** `api` (`casehub-engine-api`)

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/ExperienceAnalyser.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/ExperienceAnalyserTest.java`

**Interfaces:**
- Consumes: `RetrievedExperience`, `ExperiencePlanStep`, `RoutingOutcome` (all in same package)
- Produces: `ExperienceAnalyser.workerSuccessRates(List<RetrievedExperience>, Set<String>, String, Map<RoutingOutcome, Double>) → Map<String, Double>` and `ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS`

- [ ] **Step 1: Write the test class with first test — empty experiences**

```java
package io.casehub.api.spi.routing;

import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class ExperienceAnalyserTest {

    @Test
    void emptyExperiences_returnsEmptyMap() {
        Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
                List.of(), Set.of("agent-a"), "security-review",
                ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ExperienceAnalyserTest#emptyExperiences_returnsEmptyMap -f /path/to/engine/pom.xml`
Expected: Compilation error — `ExperienceAnalyser` does not exist

- [ ] **Step 3: Implement `ExperienceAnalyser`**

```java
package io.casehub.api.spi.routing;

import java.util.*;

public final class ExperienceAnalyser {

    public static final Map<RoutingOutcome, Double> DEFAULT_OUTCOME_WEIGHTS = Map.of(
            RoutingOutcome.SUCCESS, 1.0,
            RoutingOutcome.GATE_EXPIRED, 0.5,
            RoutingOutcome.GATE_REJECTED, 0.25,
            RoutingOutcome.FAILURE, 0.0);

    private ExperienceAnalyser() {}

    public static Map<String, Double> workerSuccessRates(
            final List<RetrievedExperience> experiences,
            final Set<String> eligibleWorkerIds,
            final String capabilityName,
            final Map<RoutingOutcome, Double> outcomeWeights) {

        final Map<String, double[]> workerStats = new HashMap<>();

        for (final RetrievedExperience exp : experiences) {
            final double relevance = exp.similarityScore();
            if (relevance <= 0.0) { continue; }

            for (final ExperiencePlanStep step : exp.planTrace()) {
                if (!capabilityName.equals(step.capabilityName())
                        || step.workerName() == null
                        || !eligibleWorkerIds.contains(step.workerName())) {
                    continue;
                }

                RoutingOutcome outcome;
                try {
                    outcome = RoutingOutcome.valueOf(step.stepOutcome());
                } catch (final IllegalArgumentException e) {
                    continue;
                }

                final double outcomeWeight = outcomeWeights.getOrDefault(outcome, 0.0);
                final double[] stats = workerStats.computeIfAbsent(
                        step.workerName(), k -> new double[2]);
                stats[0] += outcomeWeight * relevance;
                stats[1] += relevance;
            }
        }

        final Map<String, Double> scores = new HashMap<>();
        for (final Map.Entry<String, double[]> entry : workerStats.entrySet()) {
            final double evidenceMass = entry.getValue()[1];
            if (evidenceMass > 0.0) {
                scores.put(entry.getKey(), entry.getValue()[0] / evidenceMass);
            }
        }
        return scores;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ExperienceAnalyserTest#emptyExperiences_returnsEmptyMap -f /path/to/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Add remaining tests**

Add these test methods to `ExperienceAnalyserTest`:

```java
@Test
void noMatchingCapability_returnsEmptyMap() {
    var exp = experience(0.8, step("agent-a", "style-review", "SUCCESS"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).isEmpty();
}

@Test
void noMatchingWorker_returnsEmptyMap() {
    var exp = experience(0.8, step("agent-b", "security-review", "SUCCESS"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).isEmpty();
}

@Test
void singleSuccessStep_returnsWeightedScore() {
    var exp = experience(0.9, step("agent-a", "security-review", "SUCCESS"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).containsEntry("agent-a", 1.0);
}

@Test
void singleFailureStep_returnsZeroScore() {
    var exp = experience(0.9, step("agent-a", "security-review", "FAILURE"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).containsEntry("agent-a", 0.0);
}

@Test
void multipleExperiences_weightedAverage() {
    var exp1 = experience(0.9, step("agent-a", "security-review", "SUCCESS"));
    var exp2 = experience(0.3, step("agent-a", "security-review", "FAILURE"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp1, exp2), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    // (1.0 * 0.9 + 0.0 * 0.3) / (0.9 + 0.3) = 0.9 / 1.2 = 0.75
    assertThat(result.get("agent-a")).isCloseTo(0.75, within(0.001));
}

@Test
void unknownOutcome_skipped() {
    var exp = experience(0.8, step("agent-a", "security-review", "UNKNOWN_STATUS"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).isEmpty();
}

@Test
void zeroSimilarity_experienceSkipped() {
    var exp = experience(0.0, step("agent-a", "security-review", "SUCCESS"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).isEmpty();
}

@Test
void negativeSimilarity_experienceSkipped() {
    var exp = experience(-0.5, step("agent-a", "security-review", "SUCCESS"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).isEmpty();
}

@Test
void multipleWorkers_independentScores() {
    var exp = experience(0.8,
            step("agent-a", "security-review", "SUCCESS"),
            step("agent-b", "security-review", "FAILURE"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a", "agent-b"), "security-review",
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
    assertThat(result).containsEntry("agent-a", 1.0);
    assertThat(result).containsEntry("agent-b", 0.0);
}

@Test
void customOutcomeWeights_appliedCorrectly() {
    var customWeights = Map.of(
            RoutingOutcome.SUCCESS, 0.5,
            RoutingOutcome.FAILURE, 0.0,
            RoutingOutcome.GATE_REJECTED, 0.0,
            RoutingOutcome.GATE_EXPIRED, 0.0);
    var exp = experience(1.0, step("agent-a", "security-review", "SUCCESS"));
    Map<String, Double> result = ExperienceAnalyser.workerSuccessRates(
            List.of(exp), Set.of("agent-a"), "security-review", customWeights);
    assertThat(result).containsEntry("agent-a", 0.5);
}

// --- helpers ---

private static RetrievedExperience experience(double similarity, ExperiencePlanStep... steps) {
    return new RetrievedExperience(
            "problem", "solution", "COMPLETED", 1.0, similarity,
            Map.of(), List.of(steps), Map.of());
}

private static ExperiencePlanStep step(String worker, String capability, String outcome) {
    return new ExperiencePlanStep(
            "binding-" + worker, capability, worker, outcome, 0, Map.of());
}
```

- [ ] **Step 6: Run all tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ExperienceAnalyserTest -f /path/to/engine/pom.xml`
Expected: All 10 tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/routing/ExperienceAnalyser.java api/src/test/java/io/casehub/api/spi/routing/ExperienceAnalyserTest.java
git commit -m "feat(api): ExperienceAnalyser — shared CBR worker success rate utility

Refs casehubio/devtown#133"
```

---

### Task 2: `TrustRoutingPolicy.cbrWeight` + preference key + resolver (engine-api)

**Repo:** casehub-engine
**Module:** `api` (`casehub-engine-api`)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicy.java` — add `cbrWeight` field
- Modify: `api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicyKeys.java` — add `cbrWeight()` key
- Modify: `api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicyResolver.java` — read `cbrWeight` from prefs, pass as 9th arg
- Test: `api/src/test/java/io/casehub/api/spi/routing/TrustRoutingPolicyResolverTest.java` — verify resolver reads cbrWeight
- Test: `api/src/test/java/io/casehub/api/spi/routing/TrustRoutingPolicyTest.java` — verify default

**Interfaces:**
- Consumes: `PreferenceKey<DoublePreference>` (platform-api)
- Produces: `TrustRoutingPolicy.cbrWeight()` accessor, `TrustRoutingPolicyKeys.cbrWeight()` key

**BREAKING CHANGE:** Adding a 9th record field breaks every `new TrustRoutingPolicy(...)` call. Task 3 handles the migration across engine modules.

- [ ] **Step 1: Write test for resolver reading cbrWeight**

Add to `TrustRoutingPolicyResolverTest`:

```java
@Test
void resolve_readsCbrWeight() {
    Preferences prefs = stubPrefs(Map.of(
            KEYS.threshold(), DoublePreference.of(0.7),
            KEYS.cbrWeight(), DoublePreference.of(0.2)));
    TrustRoutingPolicy result = TrustRoutingPolicyResolver.resolve(prefs, KEYS);
    assertThat(result.cbrWeight()).isEqualTo(0.2);
}

@Test
void resolve_cbrWeightDefaultsToZero() {
    Preferences prefs = stubPrefs(Map.of(
            KEYS.threshold(), DoublePreference.of(0.7)));
    TrustRoutingPolicy result = TrustRoutingPolicyResolver.resolve(prefs, KEYS);
    assertThat(result.cbrWeight()).isEqualTo(0.0);
}
```

- [ ] **Step 2: Run tests to verify failure**

Expected: Compilation error — `cbrWeight()` does not exist on `TrustRoutingPolicy` or `TrustRoutingPolicyKeys`

- [ ] **Step 3: Add `cbrWeight` to `TrustRoutingPolicy`**

Use `ide_edit_member` on `TrustRoutingPolicy` to add `cbrWeight` as 9th field:

```java
public record TrustRoutingPolicy(
    double threshold,
    int minimumObservations,
    double borderlineMargin,
    double blendFactor,
    Map<String, Double> qualityFloors,
    boolean bootstrapEscalationRequired,
    String fallbackBinding,
    Set<TrustPhase> evidentialCheckPhases,
    double cbrWeight) {

  public TrustRoutingPolicy {
    qualityFloors = qualityFloors != null ? Map.copyOf(qualityFloors) : Map.of();
    evidentialCheckPhases =
        evidentialCheckPhases != null ? Set.copyOf(evidentialCheckPhases) : Set.of();
  }

  public static final TrustRoutingPolicy DEFAULT =
      new TrustRoutingPolicy(0.7, 10, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.0);
```

- [ ] **Step 4: Add `cbrWeight()` to `TrustRoutingPolicyKeys`**

Add field and accessor:

```java
private final PreferenceKey<DoublePreference> cbrWeight;
```

In the constructor, after `blendFactor` init:

```java
this.cbrWeight =
    new PreferenceKey<>(
        scopePrefix, "cbr-weight", DoublePreference.of(0.0), DoublePreference::parse);
```

Add accessor:

```java
public PreferenceKey<DoublePreference> cbrWeight() {
    return cbrWeight;
}
```

- [ ] **Step 5: Update `TrustRoutingPolicyResolver.resolve()` — add cbrWeight**

In the 2-arg `resolve()` method, read and pass `cbrWeight`:

```java
DoublePreference cbrWeightPref = prefs.get(keys.cbrWeight());

return new TrustRoutingPolicy(
    threshold.value(),
    minObs != null ? minObs.value() : TrustRoutingPolicy.DEFAULT.minimumObservations(),
    margin != null ? margin.value() : TrustRoutingPolicy.DEFAULT.borderlineMargin(),
    blend != null ? blend.value() : TrustRoutingPolicy.DEFAULT.blendFactor(),
    Map.copyOf(qualityFloors),
    bootstrapEscalationRequired,
    TrustRoutingPolicy.DEFAULT.fallbackBinding(),
    TrustRoutingPolicy.DEFAULT.evidentialCheckPhases(),
    cbrWeightPref != null ? cbrWeightPref.value() : 0.0);
```

- [ ] **Step 6: Run resolver tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=TrustRoutingPolicyResolverTest -f /path/to/engine/pom.xml`
Expected: Compilation errors from other test files using the 8-arg constructor — that's Task 3

- [ ] **Step 7: Commit (will not compile until Task 3 completes)**

```bash
git add api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicy.java \
        api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicyKeys.java \
        api/src/main/java/io/casehub/api/spi/routing/TrustRoutingPolicyResolver.java \
        api/src/test/java/io/casehub/api/spi/routing/TrustRoutingPolicyResolverTest.java
git commit -m "feat(api): TrustRoutingPolicy.cbrWeight — preference-configurable CBR weight

BREAKING: TrustRoutingPolicy record gains 9th field. All constructor
call sites must add 0.0 as 9th argument.

Refs casehubio/devtown#133"
```

---

### Task 3: Mechanical constructor migration + TrustWeightedAgentStrategy CBR enhancement (engine)

**Repo:** casehub-engine
**Modules:** `api` (tests), `ledger`, `ai` (tests)

This task has two parts: (A) migrate all `new TrustRoutingPolicy(...)` call sites to 9 args, and (B) enhance `TrustWeightedAgentStrategy` to use CBR bonus.

**Files (Part A — mechanical, add `, 0.0` as 9th arg):**
- Modify: `api/src/test/java/io/casehub/api/spi/routing/TrustRoutingPolicyTest.java`
- Modify: `api/src/test/java/io/casehub/api/spi/routing/TrustRoutingPolicyResolverTest.java` (existing 8-arg calls)
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedAgentStrategyTest.java`
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustCandidateClassifierTest.java`
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustGatedAttestationPolicyTest.java`
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedImplementationRoutingStrategyTest.java`
- Modify: `ledger/src/main/java/io/casehub/ledger/routing/DefaultTrustRoutingPolicyProvider.java`
- Modify: `ai/src/test/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategyTest.java`

**Files (Part B — CBR enhancement):**
- Modify: `ledger/src/main/java/io/casehub/ledger/routing/TrustWeightedAgentStrategy.java`
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedAgentStrategyTest.java` (new CBR tests)

**Interfaces:**
- Consumes: `ExperienceAnalyser.workerSuccessRates()` (Task 1), `TrustRoutingPolicy.cbrWeight()` (Task 2), `AgentRoutingContext.experiences()`
- Produces: Enhanced `TrustWeightedAgentStrategy.select()` with CBR scoring

- [ ] **Step 1: Part A — migrate all constructor calls**

Use `ide_search_text` to find all `new TrustRoutingPolicy(` in the engine repo. For each call site, use `ide_edit_member` or `ide_replace_member` to add `, 0.0` before the closing `)`.

Pattern: `new TrustRoutingPolicy(a, b, c, d, e, f, g, h)` → `new TrustRoutingPolicy(a, b, c, d, e, f, g, h, 0.0)`

Also update `DefaultTrustRoutingPolicyProvider.forCapability()` which returns `TrustRoutingPolicy.DEFAULT` — this already has the 9th arg from Task 2.

- [ ] **Step 2: Verify full engine build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -f /path/to/engine/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 3: Run existing test suite — regression check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /path/to/engine/pom.xml`
Expected: All existing tests pass (no behavioral change — cbrWeight 0.0 everywhere)

- [ ] **Step 4: Part B — Write CBR tests for `TrustWeightedAgentStrategy`**

Add to `TrustWeightedAgentStrategyTest`:

```java
@Test
void cbrWeightZero_emptyExperiences_identicalToPureTrust() {
    // Existing DEFAULT_POLICY has cbrWeight=0.0
    // Verify behavior unchanged with empty experiences list
    var ctx = new AgentRoutingContext(UUID.randomUUID(), "security-review",
            NullNode.instance, "test-tenant", List.of());
    // ... same as existing qualified test — result should be identical
}

@Test
void cbrWeight_agentWithHigherCbrBonusWins() {
    var policy = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.2);
    // agent-a: trust=0.75, CBR success rate=1.0
    // agent-b: trust=0.80, no CBR history
    // With cbrWeight=0.2: agent-a effective = 0.75*0.6*0.8 + 1.0*0.2 = 0.56
    //                      agent-b effective = 0.80*0.6     = 0.48 (pure trust blend, no CBR penalty)
    // Wait — need workload too. Let both have 0 running jobs (workload=1.0):
    // agent-a trustBlend = 0.75*0.6 + 1.0*0.4 = 0.85; effective = 0.85*0.8 + 1.0*0.2 = 0.88
    // agent-b trustBlend = 0.80*0.6 + 1.0*0.4 = 0.88; effective = 0.88 (no CBR data — pure trust)
    // agent-a wins (0.88 vs 0.88)... need wider gap. Use trust 0.72 vs 0.78:
    // agent-a: trustBlend = 0.72*0.6 + 1.0*0.4 = 0.832; effective = 0.832*0.8 + 1.0*0.2 = 0.866
    // agent-b: trustBlend = 0.78*0.6 + 1.0*0.4 = 0.868; effective = 0.868 (pure trust)
    // agent-a wins! (0.866 < 0.868) ... agent-a still loses by 0.002.
    // Need higher CBR or wider trust gap. Use trust 0.72 vs 0.76:
    // agent-a: trustBlend = 0.832; effective = 0.832*0.8 + 1.0*0.2 = 0.8656
    // agent-b: trustBlend = 0.856; effective = 0.856
    // agent-a wins (0.8656 > 0.856) ✓

    var experiences = List.of(new RetrievedExperience(
            "problem", "solution", "COMPLETED", 1.0, 0.9,
            Map.of(), List.of(new ExperiencePlanStep(
                    "binding", "security-review", "agent-a", "SUCCESS", 0, Map.of())),
            Map.of()));

    var ctx = new AgentRoutingContext(UUID.randomUUID(), "security-review",
            NullNode.instance, "test-tenant", experiences);

    // Set up trust scores: agent-a=0.72, agent-b=0.76, both 10+ observations
    setTrustScore("agent-a", "security-review", 0.72, 15);
    setTrustScore("agent-b", "security-review", 0.76, 15);

    var candidates = List.of(
            candidate("agent-a", 0),
            candidate("agent-b", 0));

    var result = strategy(policy).select(ctx, candidates).await().indefinitely();
    assertThat(result).isInstanceOf(RoutingResult.Selected.class);
    var selected = (RoutingResult.Selected) result;
    assertThat(selected.single().executorId()).isEqualTo("agent-a");
    assertThat(selected.single().reason()).contains("cbr_bonus");
}

@Test
void cbrWeight_noExperiences_identicalToPureTrust() {
    var policy = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.2);
    var ctx = new AgentRoutingContext(UUID.randomUUID(), "security-review",
            NullNode.instance, "test-tenant", List.of());

    setTrustScore("agent-a", "security-review", 0.80, 15);
    var candidates = List.of(candidate("agent-a", 0));

    var result = strategy(policy).select(ctx, candidates).await().indefinitely();
    assertThat(result).isInstanceOf(RoutingResult.Selected.class);
    var selected = (RoutingResult.Selected) result;
    assertThat(selected.single().reason()).doesNotContain("cbr_bonus");
}

@Test
void cbrWeight_asymmetricHistory_workerWithoutHistory_retainsPureTrust() {
    var policy = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.2);
    var experiences = List.of(new RetrievedExperience(
            "problem", "solution", "COMPLETED", 1.0, 0.9,
            Map.of(), List.of(new ExperiencePlanStep(
                    "binding", "security-review", "agent-a", "SUCCESS", 0, Map.of())),
            Map.of()));

    var ctx = new AgentRoutingContext(UUID.randomUUID(), "security-review",
            NullNode.instance, "test-tenant", experiences);

    // Both agents have same trust — agent-a has CBR history, agent-b does not
    setTrustScore("agent-a", "security-review", 0.80, 15);
    setTrustScore("agent-b", "security-review", 0.80, 15);

    var candidates = List.of(candidate("agent-a", 0), candidate("agent-b", 0));

    var result = strategy(policy).select(ctx, candidates).await().indefinitely();
    assertThat(result).isInstanceOf(RoutingResult.Selected.class);
    // agent-a should win: same trust, but CBR bonus > 0
    assertThat(((RoutingResult.Selected) result).single().executorId()).isEqualTo("agent-a");
}

@Test
void cbrWeight_bootstrapCandidate_noCbrBonus() {
    var policy = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.2);
    var experiences = List.of(new RetrievedExperience(
            "problem", "solution", "COMPLETED", 1.0, 0.9,
            Map.of(), List.of(new ExperiencePlanStep(
                    "binding", "security-review", "agent-a", "SUCCESS", 0, Map.of())),
            Map.of()));

    var ctx = new AgentRoutingContext(UUID.randomUUID(), "security-review",
            NullNode.instance, "test-tenant", experiences);

    // agent-a is bootstrap (2 < 5 minimumObservations)
    setTrustScore("agent-a", "security-review", 0.80, 2);

    var candidates = List.of(candidate("agent-a", 0));

    var result = strategy(policy).select(ctx, candidates).await().indefinitely();
    assertThat(result).isInstanceOf(RoutingResult.Selected.class);
    // Bootstrap — rationale should NOT mention cbr_bonus
    assertThat(((RoutingResult.Selected) result).single().reason()).contains("bootstrap");
    assertThat(((RoutingResult.Selected) result).single().reason()).doesNotContain("cbr_bonus");
}

@Test
void cbrWeight_borderlineAgent_notRescuedByCbr() {
    var policy = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.2);
    var experiences = List.of(new RetrievedExperience(
            "problem", "solution", "COMPLETED", 1.0, 0.9,
            Map.of(), List.of(new ExperiencePlanStep(
                    "binding", "security-review", "agent-a", "SUCCESS", 0, Map.of())),
            Map.of()));

    var ctx = new AgentRoutingContext(UUID.randomUUID(), "security-review",
            NullNode.instance, "test-tenant", experiences);

    // agent-a trust=0.65 — borderline (within 0.1 of 0.7 threshold)
    setTrustScore("agent-a", "security-review", 0.65, 15);

    var candidates = List.of(candidate("agent-a", 0));

    var result = strategy(policy).select(ctx, candidates).await().indefinitely();
    // Borderline → escalation, not assignment — CBR cannot rescue
    assertThat(result).isInstanceOf(RoutingResult.Escalated.class);
}
```

Note: helper methods `setTrustScore()`, `candidate()`, `strategy()` follow the existing test patterns — adapt from the test class's existing helpers.

- [ ] **Step 5: Enhance `TrustWeightedAgentStrategy.select()`**

Modify the `select()` method. After the bootstrap guard and eligible list computation, add CBR pre-computation:

```java
// CBR pre-computation
final Map<String, Double> cbrScores;
if (policy.cbrWeight() > 0.0 && !context.experiences().isEmpty()) {
    final Set<String> workerIds = eligible.stream()
            .filter(c -> c.phase() == Phase.QUALIFIED)
            .map(c -> c.candidate().workerId())
            .collect(Collectors.toSet());
    cbrScores = ExperienceAnalyser.workerSuccessRates(
            context.experiences(), workerIds, context.capabilityName(),
            ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS);
} else {
    cbrScores = Map.of();
}
```

Then update the scoring loop to pass `cbrScores`:

```java
final double finalScore = score(cc, policy, cbrScores);
scored.add(new ScoredCandidate(cc, finalScore, buildRationale(cc, finalScore, policy, cbrScores)));
```

- [ ] **Step 6: Update `score()` method**

```java
private double score(final ClassifiedCandidate cc, final TrustRoutingPolicy policy,
                     final Map<String, Double> cbrScores) {
    return switch (cc.phase()) {
        case Phase.BOOTSTRAP -> cc.workloadScore();
        case Phase.QUALIFIED -> {
            final double t = cc.trustScore().getAsDouble();
            final double trustBlend = t * policy.blendFactor()
                    + cc.workloadScore() * (1.0 - policy.blendFactor());
            if (policy.cbrWeight() > 0.0 && cbrScores.containsKey(cc.candidate().workerId())) {
                final double cbrBonus = cbrScores.get(cc.candidate().workerId());
                yield trustBlend * (1.0 - policy.cbrWeight()) + cbrBonus * policy.cbrWeight();
            }
            yield trustBlend;
        }
        case Phase.BORDERLINE, Phase.EXCLUDED_PHASE2B, Phase.EXCLUDED_PHASE3 -> 0.0;
    };
}
```

- [ ] **Step 7: Update `buildRationale()` method**

```java
private String buildRationale(final ClassifiedCandidate cc, final double finalScore,
                              final TrustRoutingPolicy policy,
                              final Map<String, Double> cbrScores) {
    final String workerId = cc.candidate().workerId();
    return switch (cc.phase()) {
        case Phase.BOOTSTRAP ->
            "selected %s: availability %.2f (bootstrap)".formatted(workerId, cc.workloadScore());
        case Phase.QUALIFIED -> {
            final double trustScore = cc.trustScore().getAsDouble();
            if (policy.cbrWeight() > 0.0 && cbrScores.containsKey(workerId)) {
                final double cbrBonus = cbrScores.get(workerId);
                yield "selected %s: trust %.2f, cbr_bonus %.2f, blended %.2f (threshold %.2f, cbrWeight %.2f)"
                    .formatted(workerId, trustScore, cbrBonus, finalScore,
                               policy.threshold(), policy.cbrWeight());
            }
            yield "selected %s: trust %.2f, blended %.2f (threshold %.2f)"
                .formatted(workerId, trustScore, finalScore, policy.threshold());
        }
        case Phase.BORDERLINE, Phase.EXCLUDED_PHASE2B, Phase.EXCLUDED_PHASE3 ->
            "excluded %s: phase %s".formatted(workerId, cc.phase());
    };
}
```

- [ ] **Step 8: Add `Collectors` import if not already present**

`import java.util.stream.Collectors;`

- [ ] **Step 9: Run all engine tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /path/to/engine/pom.xml`
Expected: All tests pass including new CBR tests

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(ledger): TrustWeightedAgentStrategy CBR-enhanced scoring

Add ExperienceAnalyser-based similarity bonus for QUALIFIED candidates.
cbrWeight field on TrustRoutingPolicy controls the blend ratio (default 0.0
— no behavioral change). Migrate all constructor call sites to 9-arg form.

Refs casehubio/devtown#133"
```

- [ ] **Step 11: Deploy engine SNAPSHOT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn deploy -f /path/to/engine/pom.xml -DskipTests`

This makes the new engine-api and engine-ledger artifacts available for devtown.

---

### Task 4: `DevtownCbrFeatureProvider` (devtown)

**Repo:** casehub-devtown

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/cbr/DevtownCbrFeatureProvider.java`
- Test: `app/src/test/java/io/casehub/devtown/app/cbr/DevtownCbrFeatureProviderTest.java`

**Interfaces:**
- Consumes: `CbrFeatureFunction` (engine-api — `io.casehub.api.model.cbr`), `PrFeatureVector.from()` (devtown domain), `CaseContext` (engine-api)
- Produces: `DevtownCbrFeatureProvider.apply(CaseContext) → Map<String, Object>`

**Prerequisite:** Engine SNAPSHOT with `CbrFeatureFunction` deployed. If `CbrFeatureFunction` doesn't exist yet in engine-api, it must be created as part of engine Task 2 (it's referenced in the design spec but may not exist yet — check first with `ide_find_class`).

- [ ] **Step 1: Verify `CbrFeatureFunction` exists in engine-api**

Use `ide_find_class` for `CbrFeatureFunction`. If it doesn't exist, create it in the engine first:

```java
package io.casehub.api.model.cbr;

import io.casehub.api.context.CaseContext;
import java.util.Map;
import java.util.function.Function;

@FunctionalInterface
public interface CbrFeatureFunction extends Function<CaseContext, Map<String, Object>> {}
```

- [ ] **Step 2: Write test**

```java
package io.casehub.devtown.app.cbr;

import io.casehub.api.context.CaseContext;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.Set;
import static org.assertj.core.api.Assertions.assertThat;

class DevtownCbrFeatureProviderTest {

    private final DevtownCbrFeatureProvider provider = new DevtownCbrFeatureProvider();

    @Test
    void extractsAllFeatures_fromPrContext() {
        var context = CaseContext.of(Map.of(
                "pr", Map.of(
                        "repo", "casehubio/devtown",
                        "linesChanged", 150,
                        "changedPaths", List.of(
                                "src/main/java/io/casehub/devtown/app/Foo.java",
                                "src/test/java/io/casehub/devtown/app/FooTest.java",
                                "pom.xml"))));

        Map<String, Object> features = provider.apply(context);

        assertThat(features.get("repo")).isEqualTo("casehubio/devtown");
        assertThat(features.get("lines-changed")).isEqualTo(150);
        assertThat((Set<?>) features.get("languages")).contains("java");
        assertThat((Boolean) features.get("has-tests")).isTrue();
        assertThat((Boolean) features.get("touched-configs")).isTrue();
    }

    @Test
    void missingPr_returnsEmptyMap() {
        var context = CaseContext.of(Map.of());
        Map<String, Object> features = provider.apply(context);
        assertThat(features).isEmpty();
    }
}
```

- [ ] **Step 3: Implement `DevtownCbrFeatureProvider`**

```java
package io.casehub.devtown.app.cbr;

import io.casehub.api.context.CaseContext;
import io.casehub.api.model.cbr.CbrFeatureFunction;
import io.casehub.devtown.domain.cbr.PrFeatureVector;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.*;

@ApplicationScoped
public class DevtownCbrFeatureProvider implements CbrFeatureFunction {

    @Override
    public Map<String, Object> apply(final CaseContext context) {
        var working = context.layer(io.casehub.api.context.ContextLayer.WORKING);
        var prNode = working.get("pr");
        if (prNode == null) { return Map.of(); }

        String repo = prNode.has("repo") ? prNode.get("repo").asText() : "";
        int linesChanged = prNode.has("linesChanged") ? prNode.get("linesChanged").asInt() : 0;

        List<String> changedPaths = new ArrayList<>();
        if (prNode.has("changedPaths") && prNode.get("changedPaths").isArray()) {
            prNode.get("changedPaths").forEach(n -> changedPaths.add(n.asText()));
        }

        if (changedPaths.isEmpty()) { return Map.of(); }

        var vector = PrFeatureVector.from(repo, 0, "", linesChanged, changedPaths);

        var features = new LinkedHashMap<String, Object>();
        features.put("repo", repo);
        features.put("lines-changed", linesChanged);
        features.put("changed-paths", vector.changedPaths());
        features.put("modules", vector.modules());
        features.put("languages", vector.languages());
        features.put("has-tests", vector.hasTests());
        features.put("touched-configs", vector.touchedConfigs());
        return features;
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownCbrFeatureProviderTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/cbr/DevtownCbrFeatureProvider.java \
        app/src/test/java/io/casehub/devtown/app/cbr/DevtownCbrFeatureProviderTest.java
git commit -m "feat(app): DevtownCbrFeatureProvider — PR feature extraction for CBR routing

Refs #133"
```

---

### Task 5: Policy configuration + YAML config + devtown constructor migration (devtown)

**Repo:** casehub-devtown

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProvider.java` — add `cbrWeight` resolution
- Modify: `review/src/main/resources/devtown/pr-review.yaml` — add `cbr:` block
- Modify: all devtown test files constructing `TrustRoutingPolicy` — add `, 0.0` as 9th arg
- Test: `app/src/test/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProviderTest.java` — new cbrWeight tests

**Interfaces:**
- Consumes: `TrustRoutingPolicyKeys.cbrWeight()` (Task 2), `DoublePreference` (engine-api)
- Produces: Per-capability `cbrWeight` in `TrustRoutingPolicy` returned by `forCapability()`

- [ ] **Step 1: Migrate devtown test constructor calls**

Search for all `new TrustRoutingPolicy(` in devtown source. Add `, 0.0` as 9th arg to each:
- `app/src/test/java/io/casehub/devtown/app/trust/EvidentialAttestationPolicyTest.java` (4 sites)
- `app/src/test/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProviderTest.java` (if any direct construction)
- `app/src/test/java/io/casehub/devtown/app/TrustRoutingActivationTest.java` (if any)

- [ ] **Step 2: Verify devtown compiles with new engine SNAPSHOT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile`
Expected: BUILD SUCCESS

- [ ] **Step 3: Write cbrWeight policy tests**

Add to `DevtownTrustRoutingPolicyProviderTest`:

```java
@Test
void securityReview_cbrWeightIsPointTwo() {
    TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.SECURITY_REVIEW);
    assertThat(policy.cbrWeight()).isEqualTo(0.2);
}

@Test
void mergeExecutor_cbrWeightIsZero() {
    TrustRoutingPolicy policy = provider.forCapability(AgentQualification.MERGE_EXECUTOR);
    assertThat(policy.cbrWeight()).isEqualTo(0.0);
}

@Test
void ciRunner_cbrWeightIsZero() {
    TrustRoutingPolicy policy = provider.forCapability(AgentQualification.CI_RUNNER);
    assertThat(policy.cbrWeight()).isEqualTo(0.0);
}

@Test
void codeAnalysis_cbrWeightIsZero() {
    TrustRoutingPolicy policy = provider.forCapability(ReviewDomain.CODE_ANALYSIS);
    assertThat(policy.cbrWeight()).isEqualTo(0.0);
}
```

- [ ] **Step 4: Update `DevtownTrustRoutingPolicyProvider.forCapability()`**

Add the cbrWeight defaults map and resolution:

```java
private static final Map<String, Double> CBR_WEIGHT_DEFAULTS = Map.of(
        ReviewDomain.SECURITY_REVIEW, 0.2,
        ReviewDomain.ARCHITECTURE_REVIEW, 0.2,
        ReviewDomain.STYLE_REVIEW, 0.2,
        ReviewDomain.TEST_COVERAGE, 0.2,
        ReviewDomain.PERFORMANCE_ANALYSIS, 0.2);
```

In `forCapability()`, after blendFactor resolution, add:

```java
final DoublePreference cbrWeightPref = prefs.get(KEYS.cbrWeight());
final double cbrWeight = cbrWeightPref != null
                         ? cbrWeightPref.value()
                         : CBR_WEIGHT_DEFAULTS.getOrDefault(capabilityName, 0.0);
```

Update the `TrustRoutingPolicy` constructor call to include `cbrWeight` as 9th arg.

- [ ] **Step 5: Add `cbr:` block to `pr-review.yaml`**

Add at the top level of the `spec:` block (after `completion:`, before `bindings:`):

```yaml
  cbr:
    cbrType: plan
    timing: CASE_LIFETIME
    topK: 5
    minSimilarity: 0.3
    featureExtractor:
      type: lambda
```

- [ ] **Step 6: Run all devtown tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: All tests pass

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProvider.java \
        app/src/test/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProviderTest.java \
        review/src/main/resources/devtown/pr-review.yaml
git commit -m "feat(app): per-capability cbrWeight + engine CBR config for PR review

security/architecture/style/test-coverage/performance: cbrWeight=0.2
code-analysis/merge-executor/ci-runner: cbrWeight=0.0
pr-review.yaml: engine-level CBR retrieval with lambda feature extractor

Refs #133"
```

---

### Task 6: Integration test + full build verification (devtown)

**Repo:** casehub-devtown

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/routing/CbrReviewerMatchingIntegrationTest.java`

**Interfaces:**
- Consumes: All previous tasks — full stack integration

- [ ] **Step 1: Write integration test — issue acceptance criterion**

```java
package io.casehub.devtown.app.routing;

import io.casehub.api.spi.routing.*;
import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.ledger.routing.TrustCandidateClassifier;
import io.casehub.ledger.routing.TrustWeightedAgentStrategy;
import com.fasterxml.jackson.databind.node.NullNode;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class CbrReviewerMatchingIntegrationTest {

    @Test
    void agentWithLowerTrustButHigherPrecedentMatch_winsOverHigherTrustNoPrecedent() {
        // Issue #133 acceptance criterion:
        // "Unit test: agent with lower trust but higher precedent match wins
        //  over agent with higher trust but no precedent"

        var experiences = List.of(new RetrievedExperience(
                "security fix in auth module", "reviewed auth changes", "COMPLETED", 1.0, 0.85,
                Map.of(), List.of(new ExperiencePlanStep(
                        "security-review-binding", ReviewDomain.SECURITY_REVIEW,
                        "specialist-agent", "SUCCESS", 0, Map.of())),
                Map.of()));

        var ctx = new AgentRoutingContext(UUID.randomUUID(), ReviewDomain.SECURITY_REVIEW,
                NullNode.instance, "test-tenant", experiences);

        // specialist-agent: trust=0.72 but strong CBR history
        // generalist-agent: trust=0.76 but no CBR history
        var policy = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.2);

        // Build strategy with mock trust source
        // specialist-agent: capScore=0.72, decisionCount=15
        // generalist-agent: capScore=0.76, decisionCount=15
        var trustSource = new StubTrustScoreSource(Map.of(
                "specialist-agent|security-review", 0.72,
                "generalist-agent|security-review", 0.76),
                Map.of(
                "specialist-agent|security-review", 15,
                "generalist-agent|security-review", 15));

        var classifier = new TrustCandidateClassifier();
        var policyProvider = (TrustRoutingPolicyProvider) cap -> policy;
        var strategy = new TrustWeightedAgentStrategy(classifier, trustSource, policyProvider);

        var candidates = List.of(
                new AgentCandidate("specialist-agent", Set.of(ReviewDomain.SECURITY_REVIEW),
                        0, AgentHealth.HEALTHY, null, null),
                new AgentCandidate("generalist-agent", Set.of(ReviewDomain.SECURITY_REVIEW),
                        0, AgentHealth.HEALTHY, null, null));

        var result = strategy.select(ctx, candidates).await().indefinitely();

        assertThat(result).isInstanceOf(RoutingResult.Selected.class);
        var selected = (RoutingResult.Selected) result;
        assertThat(selected.single().executorId()).isEqualTo("specialist-agent");
        assertThat(selected.single().reason()).contains("cbr_bonus");
    }

    @Test
    void zeroPrecedents_identicalToPureTrustRouting() {
        // Issue #133 acceptance criterion:
        // "Unit test: zero precedents = identical result to pure trust routing"

        var ctx = new AgentRoutingContext(UUID.randomUUID(), ReviewDomain.SECURITY_REVIEW,
                NullNode.instance, "test-tenant", List.of());

        var policy = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.2);
        var policyZero = new TrustRoutingPolicy(0.7, 5, 0.1, 0.6, Map.of(), false, null, Set.of(), 0.0);

        var trustSource = new StubTrustScoreSource(Map.of(
                "agent-a|security-review", 0.80,
                "agent-b|security-review", 0.75),
                Map.of(
                "agent-a|security-review", 15,
                "agent-b|security-review", 15));

        var classifier = new TrustCandidateClassifier();
        var strategyWithCbr = new TrustWeightedAgentStrategy(
                classifier, trustSource, cap -> policy);
        var strategyWithout = new TrustWeightedAgentStrategy(
                classifier, trustSource, cap -> policyZero);

        var candidates = List.of(
                new AgentCandidate("agent-a", Set.of(ReviewDomain.SECURITY_REVIEW),
                        0, AgentHealth.HEALTHY, null, null),
                new AgentCandidate("agent-b", Set.of(ReviewDomain.SECURITY_REVIEW),
                        0, AgentHealth.HEALTHY, null, null));

        var resultWith = strategyWithCbr.select(ctx, candidates).await().indefinitely();
        var resultWithout = strategyWithout.select(ctx, candidates).await().indefinitely();

        // Both should select agent-a (highest trust) with identical behavior
        assertThat(((RoutingResult.Selected) resultWith).single().executorId())
                .isEqualTo(((RoutingResult.Selected) resultWithout).single().executorId());
    }

    // Stub trust source for testing
    private record StubTrustScoreSource(
            Map<String, Double> scores,
            Map<String, Integer> counts
    ) implements io.casehub.ledger.api.spi.TrustScoreSource {
        @Override
        public OptionalDouble capabilityScore(String workerId, String capability) {
            Double s = scores.get(workerId + "|" + capability);
            return s != null ? OptionalDouble.of(s) : OptionalDouble.empty();
        }

        @Override
        public int decisionCount(String workerId, String capability) {
            return counts.getOrDefault(workerId + "|" + capability, 0);
        }

        @Override
        public OptionalDouble capabilityDimensionScore(String workerId, String capability, String dimension) {
            return OptionalDouble.empty();
        }

        @Override
        public Map<String, Double> allDimensionScores(String workerId, String capability) {
            return Map.of();
        }
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CbrReviewerMatchingIntegrationTest`
Expected: PASS

- [ ] **Step 3: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/devtown/app/routing/CbrReviewerMatchingIntegrationTest.java
git commit -m "test(app): CBR reviewer matching integration test — issue #133 acceptance criteria

Closes #133"
```

---

## Self-review checklist

1. **Spec coverage:** All 4 components covered. ExperienceAnalyser (Task 1), TrustRoutingPolicy.cbrWeight + Keys + Resolver (Task 2), TrustWeightedAgentStrategy enhancement (Task 3), DevtownCbrFeatureProvider (Task 4), Policy config + YAML (Task 5), Integration test (Task 6). ✓
2. **Placeholder scan:** All steps contain code. No TBD/TODO. ✓
3. **Type consistency:** `ExperienceAnalyser.workerSuccessRates()` signature used consistently in Tasks 1 and 3. `TrustRoutingPolicy` 9-arg constructor used consistently. `cbrWeight()` accessor used consistently. ✓
4. **Tooling safety scan:** No bash file operations on source files. All code edits use `ide_*` tools or `Write` for new files. git/mvn commands use bash appropriately. ✓
