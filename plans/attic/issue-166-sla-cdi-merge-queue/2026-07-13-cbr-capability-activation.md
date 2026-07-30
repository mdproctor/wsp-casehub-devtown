# CBR-Enhanced Capability Activation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #132 — feat: CBR-enhanced capability activation — fire review capabilities based on precedent
**Issue group:** #132

**Goal:** Activate conditional review capabilities (security-review, architecture-review) based on findings in similar past cases, even when static content analysis alone wouldn't trigger them.

**Architecture:** Pre-compute precedent activation decisions at memory recall time using a findings-based `PrecedentActivationPolicy`. Separate precedent-triggered bindings in the CasePlanModel dispatch capabilities with `activationSource: "precedent"` audit trail. Goal and merge binding conditions updated to gate on precedent-triggered reviews.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ

## Global Constraints

- `devtown-domain` is pure Java — no Quarkus, no engine API dependencies
- `review` module depends on `domain` + engine API (`casehub-api`)
- `app` module depends on everything — all CDI wiring lives here
- `Precedent` moves from `review/` to `domain/cbr/` — use `ide_move_file`
- Both Java DSL and YAML case definitions must stay in sync — `PrReviewCaseDefinitionEquivalenceTest` enforces this
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

### Task 1: CapabilityOutcome record + tests

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/cbr/CapabilityOutcome.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/cbr/CapabilityOutcomeTest.java`

**Interfaces:**
- Produces: `CapabilityOutcome(String outcome, String detail)` record with `hadFindings(): boolean`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain.cbr;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import static org.assertj.core.api.Assertions.assertThat;

class CapabilityOutcomeTest {

    @ParameterizedTest
    @CsvSource({
        "COMPLETED,,true",
        "COMPLETED,FINDINGS_PRESENT,true",
        "COMPLETED,flagged,true",
        "COMPLETED,needs-work,true",
        "COMPLETED,approved,false",
        "COMPLETED,passed,false",
        "COMPLETED,APPROVED,false",
        "COMPLETED,Passed,false",
        "FAILED,,false",
        "FAILED,approved,false",
        "FAILED,FINDINGS_PRESENT,false",
        "DECLINED,,false",
        "DECLINED,outside-scope,false"
    })
    void hadFindings(String outcome, String detail, boolean expected) {
        assertThat(new CapabilityOutcome(outcome, detail).hadFindings())
            .as("CapabilityOutcome(%s, %s)", outcome, detail)
            .isEqualTo(expected);
    }

    @Test
    void nullOutcomeIsNotFindings() {
        assertThat(new CapabilityOutcome(null, null).hadFindings()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=CapabilityOutcomeTest -Dsurefire.useFile=false`
Expected: compilation failure — `CapabilityOutcome` does not exist

- [ ] **Step 3: Write implementation**

Create `domain/src/main/java/io/casehub/devtown/domain/cbr/CapabilityOutcome.java`:

```java
package io.casehub.devtown.domain.cbr;

import java.util.Set;

public record CapabilityOutcome(String outcome, String detail) {

    private static final Set<String> SAFE_DETAILS = Set.of("approved", "passed");

    public boolean hadFindings() {
        return "COMPLETED".equals(outcome) &&
               (detail == null || !SAFE_DETAILS.contains(detail.toLowerCase()));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=CapabilityOutcomeTest -Dsurefire.useFile=false`
Expected: all tests pass

- [ ] **Step 5: Commit**

```
feat(#132): add CapabilityOutcome record with hadFindings() logic

Refs casehubio/devtown#132
```

---

### Task 2: Move Precedent + type change + caller updates

**Files:**
- Move: `review/src/main/java/io/casehub/devtown/review/Precedent.java` → `domain/src/main/java/io/casehub/devtown/domain/cbr/Precedent.java` (use `ide_move_file`)
- Modify: `domain/src/main/java/io/casehub/devtown/domain/cbr/Precedent.java` — change `capabilityOutcomes` type
- Modify: `app/src/main/java/io/casehub/devtown/app/DefaultCbrRetrievalService.java` — `enrichOutcomes()` returns `Map<String, CapabilityOutcome>`, `aggregateOutcome()` updated, `scoreCandidate()` updated
- Modify: `review/src/main/java/io/casehub/devtown/review/MemoryContext.java` — update `toContextMap()` serialization for `CapabilityOutcome`
- Modify: `app/src/test/java/io/casehub/devtown/app/DefaultCbrRetrievalServiceTest.java`
- Modify: `review/src/test/java/io/casehub/devtown/review/MemoryContextTest.java`

**Interfaces:**
- Consumes: `CapabilityOutcome` from Task 1
- Produces: `Precedent(UUID, SimilarityScore, PrFeatureVector, String, Map<String, CapabilityOutcome>)` in `domain/cbr/`

- [ ] **Step 1: Move Precedent to domain/cbr/**

Use `ide_move_file` to move `review/src/main/java/io/casehub/devtown/review/Precedent.java` to `domain/src/main/java/io/casehub/devtown/domain/cbr/`. This updates all imports in `CbrRetrievalService`, `MemoryContext`, `DefaultCbrRetrievalService`, `CaseMemoryRecaller`, and tests automatically.

- [ ] **Step 2: Change Precedent record type**

Use `ide_edit_member` on `Precedent.java`, member `Precedent`:

```java
public record Precedent(
    UUID caseId,
    SimilarityScore similarity,
    PrFeatureVector vector,
    String outcome,
    Map<String, CapabilityOutcome> capabilityOutcomes
) {}
```

- [ ] **Step 3: Update DefaultCbrRetrievalService.enrichOutcomes()**

Use `ide_replace_member` on `DefaultCbrRetrievalService.java`, member `enrichOutcomes`:

```java
    try {
        List<Memory> outcomeFacts = store.query(
            MemoryQuery.forEntity(
                DevtownMemoryDomain.CONTRIBUTOR_PREFIX + contributor,
                DevtownMemoryDomain.SOFTWARE_REVIEW,
                tenantId)
            .withCaseId(caseId.toString())
            .withLimit(20));

        var outcomes = new LinkedHashMap<String, CapabilityOutcome>();
        for (var fact : outcomeFacts) {
            String capability = fact.attributes().get(DevtownMemoryKeys.CAPABILITY);
            String outcome = fact.attributes().get(MemoryAttributeKeys.OUTCOME);
            String detail = fact.attributes().get(DevtownMemoryKeys.OUTCOME_DETAIL);
            if (capability != null && outcome != null) {
                outcomes.put(capability, new CapabilityOutcome(outcome, detail));
            }
        }
        return outcomes;
    } catch (Exception e) {
        LOG.debugf(e, "Failed to enrich outcomes for case=%s", caseId);
        return Map.of();
    }
```

Also update the return type signature via `ide_edit_member`:

```java
private Map<String, CapabilityOutcome> enrichOutcomes(UUID caseId, String contributor, String tenantId) {
```

- [ ] **Step 4: Update DefaultCbrRetrievalService.aggregateOutcome()**

Use `ide_edit_member` on `DefaultCbrRetrievalService.java`, member `aggregateOutcome`:

```java
private String aggregateOutcome(Map<String, CapabilityOutcome> capabilityOutcomes) {
    boolean anyFailed = capabilityOutcomes.values().stream()
        .anyMatch(co -> "FAILED".equals(co.outcome()));
    if (anyFailed) return "failed";
    boolean anyFindings = capabilityOutcomes.values().stream()
        .anyMatch(CapabilityOutcome::hadFindings);
    return anyFindings ? "flagged" : "approved";
}
```

- [ ] **Step 5: Update DefaultCbrRetrievalService.scoreCandidate()**

Update the local variable type. Use `ide_replace_member` on `scoreCandidate`:

```java
    try {
        SimilarityScore score = metric.compute(query, cv.vector);

        Map<String, CapabilityOutcome> capabilityOutcomes = enrichOutcomes(cv.caseId, cv.contributor, tenantId);
        if (capabilityOutcomes.isEmpty()) return null;

        String aggregate = aggregateOutcome(capabilityOutcomes);
        return new Precedent(cv.caseId, score, cv.vector, aggregate, capabilityOutcomes);
    } catch (Exception e) {
        LOG.debugf(e, "Failed to score candidate case=%s", cv.caseId);
        return null;
    }
```

- [ ] **Step 6: Update MemoryContext.toContextMap() — capabilityOutcomes serialization**

Use `ide_replace_member` on `MemoryContext.java`, member `toContextMap`:

```java
    return Map.of(
        "contributorHistory", toEntryList(contributorHistory),
        "codeAreaHistory", toEntryList(codeAreaHistory),
        "precedents", precedents.stream().map(p -> Map.<String, Object>of(
            "caseId", p.caseId().toString(),
            "similarity", p.similarity().score(),
            "breakdown", p.similarity().breakdown(),
            "outcome", p.outcome(),
            "capabilityOutcomes", p.capabilityOutcomes().entrySet().stream()
                .collect(java.util.stream.Collectors.toMap(
                    Map.Entry::getKey,
                    e -> {
                        var co = e.getValue();
                        var m = new java.util.LinkedHashMap<String, String>();
                        m.put("outcome", co.outcome());
                        if (co.detail() != null) m.put("detail", co.detail());
                        return m;
                    }))
        )).toList()
    );
```

- [ ] **Step 7: Update DefaultCbrRetrievalServiceTest**

Update test expectations to use `CapabilityOutcome` instead of raw strings. The test constructs `Memory` objects with `OUTCOME` and `OUTCOME_DETAIL` attributes — verify `enrichOutcomes` returns `CapabilityOutcome` and `aggregateOutcome` uses `hadFindings()`.

- [ ] **Step 8: Update MemoryContextTest**

Update precedent construction in any tests that create `Precedent` objects to use `Map<String, CapabilityOutcome>` instead of `Map<String, String>`.

- [ ] **Step 9: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all tests pass, including `PrReviewCaseDefinitionEquivalenceTest`

- [ ] **Step 10: Commit**

```
refactor(#132): move Precedent to domain/cbr, enrich with CapabilityOutcome

Refs casehubio/devtown#132
```

---

### Task 3: PrecedentActivationPolicy + preference keys + tests

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/cbr/PrecedentActivationPolicy.java`
- Modify: `domain/src/main/java/io/casehub/devtown/domain/cbr/CbrPreferenceKeys.java` — two new keys
- Test: `domain/src/test/java/io/casehub/devtown/domain/cbr/PrecedentActivationPolicyTest.java`

**Interfaces:**
- Consumes: `Precedent` from domain/cbr (Task 2), `CapabilityOutcome.hadFindings()` from Task 1
- Produces: `PrecedentActivationPolicy.evaluate(List<Precedent>, int, double): Set<String>`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain.cbr;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class PrecedentActivationPolicyTest {

    private static final SimilarityScore SCORE = new SimilarityScore(0.8, Map.of());
    private static final PrFeatureVector VECTOR = new PrFeatureVector(
        "repo", 1, "dev", 100, Set.of(), Set.of(), Set.of(), false, false);

    private static final CapabilityOutcome FINDINGS = new CapabilityOutcome("COMPLETED", "FINDINGS_PRESENT");
    private static final CapabilityOutcome APPROVED = new CapabilityOutcome("COMPLETED", "approved");
    private static final CapabilityOutcome FAILED = new CapabilityOutcome("FAILED", null);

    private Precedent precedent(Map<String, CapabilityOutcome> outcomes) {
        return new Precedent(UUID.randomUUID(), SCORE, VECTOR, "flagged", outcomes);
    }

    @Test
    void emptyPrecedentsReturnsEmptySet() {
        assertThat(PrecedentActivationPolicy.evaluate(List.of(), 2, 0.4)).isEmpty();
    }

    @Test
    void belowMinFindingsNotActivated() {
        var precedents = List.of(
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED))
        );
        assertThat(PrecedentActivationPolicy.evaluate(precedents, 2, 0.4)).isEmpty();
    }

    @Test
    void belowMinFractionNotActivated() {
        var precedents = List.of(
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED))
        );
        // 2/10 = 0.2 < 0.4 min fraction
        assertThat(PrecedentActivationPolicy.evaluate(precedents, 2, 0.4)).isEmpty();
    }

    @Test
    void meetsBothThresholdsActivated() {
        var precedents = List.of(
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("security-review", APPROVED))
        );
        // 3/5 = 0.6 >= 0.4 and 3 >= 2
        assertThat(PrecedentActivationPolicy.evaluate(precedents, 2, 0.4))
            .containsExactly("security-review");
    }

    @Test
    void multipleCapabilitiesEvaluatedIndependently() {
        var precedents = List.of(
            precedent(Map.of("security-review", FINDINGS, "architecture-review", FINDINGS)),
            precedent(Map.of("security-review", FINDINGS, "architecture-review", APPROVED)),
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", APPROVED)),
            precedent(Map.of("architecture-review", APPROVED))
        );
        // security: 3/5 = 0.6 >= 0.4, 3 >= 2 → activated
        // architecture: 1/5 = 0.2, 1 < 2 → not activated
        assertThat(PrecedentActivationPolicy.evaluate(precedents, 2, 0.4))
            .containsExactly("security-review");
    }

    @Test
    void exactBoundaryMinFindingsActivated() {
        var precedents = List.of(
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", APPROVED))
        );
        // 2/3 = 0.67 >= 0.4 and 2 >= 2
        assertThat(PrecedentActivationPolicy.evaluate(precedents, 2, 0.4))
            .containsExactly("security-review");
    }

    @Test
    void failedOutcomeNotCountedAsFindings() {
        var precedents = List.of(
            precedent(Map.of("security-review", FAILED)),
            precedent(Map.of("security-review", FAILED)),
            precedent(Map.of("security-review", FAILED)),
            precedent(Map.of("security-review", FAILED)),
            precedent(Map.of("security-review", FAILED))
        );
        assertThat(PrecedentActivationPolicy.evaluate(precedents, 2, 0.4)).isEmpty();
    }

    @Test
    void capabilityAbsentFromPrecedentNotCounted() {
        var precedents = List.of(
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("security-review", FINDINGS)),
            precedent(Map.of("style-review", APPROVED)),
            precedent(Map.of("style-review", APPROVED)),
            precedent(Map.of("style-review", APPROVED))
        );
        // security: 2/5 = 0.4 >= 0.4 and 2 >= 2 → activated (absent = not counted as findings, but total is all precedents)
        assertThat(PrecedentActivationPolicy.evaluate(precedents, 2, 0.4))
            .containsExactly("security-review");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=PrecedentActivationPolicyTest -Dsurefire.useFile=false`
Expected: compilation failure — `PrecedentActivationPolicy` does not exist

- [ ] **Step 3: Write PrecedentActivationPolicy**

Create `domain/src/main/java/io/casehub/devtown/domain/cbr/PrecedentActivationPolicy.java`:

```java
package io.casehub.devtown.domain.cbr;

import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public final class PrecedentActivationPolicy {

    private PrecedentActivationPolicy() {}

    public static Set<String> evaluate(List<Precedent> precedents,
                                        int minFindings, double minFraction) {
        if (precedents.isEmpty()) return Set.of();
        Map<String, Long> counts = countFindings(precedents);
        Set<String> result = new LinkedHashSet<>();
        for (var entry : counts.entrySet()) {
            long count = entry.getValue();
            if (count >= minFindings &&
                (double) count / precedents.size() >= minFraction) {
                result.add(entry.getKey());
            }
        }
        return Set.copyOf(result);
    }

    private static Map<String, Long> countFindings(List<Precedent> precedents) {
        var counts = new HashMap<String, Long>();
        for (var precedent : precedents) {
            for (var entry : precedent.capabilityOutcomes().entrySet()) {
                if (entry.getValue().hadFindings()) {
                    counts.merge(entry.getKey(), 1L, Long::sum);
                }
            }
        }
        return counts;
    }
}
```

- [ ] **Step 4: Add preference keys to CbrPreferenceKeys**

Use `ide_insert_member` on `CbrPreferenceKeys.java`, after `GATE_SAME_REPO`:

```java
public static final PreferenceKey<IntPreference> PRECEDENT_ACTIVATION_MIN_FINDINGS =
    new PreferenceKey<>(NS, "precedent-activation-min-findings", IntPreference.of(2), IntPreference::parse);
public static final PreferenceKey<DoublePreference> PRECEDENT_ACTIVATION_MIN_FRACTION =
    new PreferenceKey<>(NS, "precedent-activation-min-fraction", DoublePreference.of(0.4), DoublePreference::parse);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=PrecedentActivationPolicyTest -Dsurefire.useFile=false`
Expected: all tests pass

- [ ] **Step 6: Commit**

```
feat(#132): add PrecedentActivationPolicy with threshold evaluation

Refs casehubio/devtown#132
```

---

### Task 4: MemoryContext + CaseMemoryRecaller — pre-computation + consolidation

**Files:**
- Modify: `review/src/main/java/io/casehub/devtown/review/MemoryContext.java` — add `precedentActivations` field, refactor `hasRisk()`, update `toContextMap()`, remove `SAFE_OUTCOMES`
- Modify: `app/src/main/java/io/casehub/devtown/app/CaseMemoryRecaller.java` — add `evaluateActivations()`, update `recall()`
- Modify: `review/src/test/java/io/casehub/devtown/review/MemoryContextTest.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/CaseMemoryRecallerTest.java`

**Interfaces:**
- Consumes: `PrecedentActivationPolicy.evaluate()` from Task 3, `CapabilityOutcome` from Task 1
- Produces: `MemoryContext` with `precedentActivations` field serialized to `memory.precedentActivations` in case context

- [ ] **Step 1: Write/update MemoryContext tests**

Add tests for the new `precedentActivations` field and `hasRisk()` delegation. Update the `EMPTY` constant assertion and `toContextMap()` tests.

- [ ] **Step 2: Update MemoryContext record**

Use `ide_edit_member` on `MemoryContext.java`, member `MemoryContext` (the record declaration):

```java
public record MemoryContext(
    List<Memory> contributorHistory,
    List<Memory> codeAreaHistory,
    List<Precedent> precedents,
    Set<String> precedentActivations
) {
    public static final MemoryContext EMPTY = new MemoryContext(List.of(), List.of(), List.of(), Set.of());
```

- [ ] **Step 3: Update toContextMap() — add precedentActivations**

Use `ide_replace_member` on `MemoryContext.java`, member `toContextMap`:

```java
    return Map.of(
        "contributorHistory", toEntryList(contributorHistory),
        "codeAreaHistory", toEntryList(codeAreaHistory),
        "precedents", precedents.stream().map(p -> Map.<String, Object>of(
            "caseId", p.caseId().toString(),
            "similarity", p.similarity().score(),
            "breakdown", p.similarity().breakdown(),
            "outcome", p.outcome(),
            "capabilityOutcomes", p.capabilityOutcomes().entrySet().stream()
                .collect(java.util.stream.Collectors.toMap(
                    Map.Entry::getKey,
                    e -> {
                        var co = e.getValue();
                        var m = new java.util.LinkedHashMap<String, String>();
                        m.put("outcome", co.outcome());
                        if (co.detail() != null) m.put("detail", co.detail());
                        return m;
                    }))
        )).toList(),
        "precedentActivations", List.copyOf(precedentActivations)
    );
```

- [ ] **Step 4: Refactor hasRisk() — delegate to CapabilityOutcome**

Use `ide_replace_member` on `MemoryContext.java`, member `hasRisk`:

```java
    return memories.stream().anyMatch(m -> {
        String outcome = m.attributes().get(MemoryAttributeKeys.OUTCOME);
        if (ReviewOutcome.FAILED.name().equals(outcome)) return true;
        String detail = m.attributes().get(DevtownMemoryKeys.OUTCOME_DETAIL);
        return new CapabilityOutcome(outcome, detail).hadFindings();
    });
```

Remove the `SAFE_OUTCOMES` constant via `ide_refactor_safe_delete`.

- [ ] **Step 5: Update CaseMemoryRecaller — add evaluateActivations()**

Use `ide_insert_member` on `CaseMemoryRecaller.java`, after `retrievePrecedents`:

```java
private Set<String> evaluateActivations(List<Precedent> precedents) {
    if (precedents.isEmpty()) return Set.of();
    Preferences cbrPrefs = preferenceProvider.resolve(
        SettingsScope.of("casehubio", "devtown", "cbr"));
    int minFindings = cbrPrefs.getOrDefault(
        CbrPreferenceKeys.PRECEDENT_ACTIVATION_MIN_FINDINGS).value();
    double minFraction = cbrPrefs.getOrDefault(
        CbrPreferenceKeys.PRECEDENT_ACTIVATION_MIN_FRACTION).value();
    return PrecedentActivationPolicy.evaluate(precedents, minFindings, minFraction);
}
```

- [ ] **Step 6: Update CaseMemoryRecaller.recall() — pass activations**

In the `recall()` method, update the return statement to compute and pass activations. Use `ide_replace_member` on `recall`:

Find the line:
```java
return new MemoryContext(contributorHistory, codeAreaHistory, precedents);
```

Replace with:
```java
Set<String> activations = evaluateActivations(precedents);
return new MemoryContext(contributorHistory, codeAreaHistory, precedents, activations);
```

Also update the catch block's `MemoryContext.EMPTY` — already correct (EMPTY now includes empty activations).

- [ ] **Step 7: Update tests**

Update `MemoryContextTest` — all `new MemoryContext(...)` calls need the 4th argument `Set.of()`. Update assertions for `toContextMap()` to verify `precedentActivations` key.

Update `CaseMemoryRecallerTest` — verify that `evaluateActivations()` is called with the correct preferences and returns the expected set.

- [ ] **Step 8: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all tests pass

- [ ] **Step 9: Commit**

```
feat(#132): pre-compute precedent activations at recall time

Refs casehubio/devtown#132
```

---

### Task 5: PrReviewCaseDefinition — goal/binding updates + precedent-triggered bindings

**Files:**
- Modify: `review/src/main/java/io/casehub/devtown/review/PrReviewCaseDefinition.java` — update 5 goal/binding conditions, add 2 new bindings, add `precedentActivates` helper
- Modify: `review/src/main/resources/devtown/pr-review.yaml` — update 4 goal/binding conditions, add 2 new bindings
- Modify: `review/src/test/java/io/casehub/devtown/review/PrReviewCaseDefinitionEquivalenceTest.java` — binding count assertion

**Interfaces:**
- Consumes: `memory.precedentActivations` from case context (set by Task 4)
- Produces: precedent-triggered `security-review` and `architecture-review` bindings with `activationSource: "precedent"` audit trail

- [ ] **Step 1: Add precedentActivates helper to PrReviewCaseDefinition**

Use `ide_insert_member` on `PrReviewCaseDefinition.java`, after `addEscalation`:

```java
@SuppressWarnings("unchecked")
private static boolean precedentActivates(CaseContext ctx, String capability) {
    var activations = (List<String>) ctx.getPath("memory.precedentActivations");
    return activations != null && activations.contains(capability);
}
```

Add import for `io.casehub.api.context.CaseContext` if not already present (it is — line 28).

- [ ] **Step 2: Update pr-approved goal condition (Java DSL)**

Use `ide_replace_member` on `PrReviewCaseDefinition.java` to update the `prApproved` goal builder. The condition changes from:

```java
(Boolean.FALSE.equals(ctx.getPath("codeAnalysis.securitySensitive")) ||
    "APPROVED".equals(ctx.getPath("securityReview.outcome")))
```

to:

```java
((Boolean.FALSE.equals(ctx.getPath("codeAnalysis.securitySensitive")) &&
    ctx.get("securityReview") == null) ||
    "APPROVED".equals(ctx.getPath("securityReview.outcome")))
```

Same pattern for the architecture condition. Full updated goal in the spec §Goal and merge binding condition updates.

- [ ] **Step 3: Update security-verified goal condition (Java DSL)**

Same pattern — add `&& ctx.get("securityReview") == null` to the first branch.

- [ ] **Step 4: Update enqueue-for-merge and merge-direct binding conditions (Java DSL)**

Both bindings have security and architecture conditions. Apply the same pattern:
- `(Boolean.FALSE.equals(...securitySensitive) && ctx.get("securityReview") == null) || "APPROVED".equals(...)`
- `(Boolean.FALSE.equals(...architectureCrossing) && ctx.get("architectureReview") == null) || "APPROVED".equals(...)`

- [ ] **Step 5: Add precedent-security-review binding (Java DSL)**

Use `ide_insert_member` on `PrReviewCaseDefinition.java`. Insert after the existing `performance-analysis` binding, before the human gate:

```java
def.getBindings().add(Binding.builder().name("precedent-security-review").on(trigger)
    .when(new LambdaExpressionEvaluator(ctx ->
        Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
        !Boolean.TRUE.equals(ctx.getPath("codeAnalysis.securitySensitive")) &&
        precedentActivates(ctx, ReviewDomain.SECURITY_REVIEW) &&
        ctx.get("securityReview") == null))
    .contextWrite(Map.of("securityReview", Map.of("activationSource", "precedent")))
    .capability(securityReviewCap)
    .conflictResolverStrategy(DEEP_MERGE)
    .outcomePolicy(REROUTE_POLICY)
    .build());
```

- [ ] **Step 6: Add precedent-architecture-review binding (Java DSL)**

```java
def.getBindings().add(Binding.builder().name("precedent-architecture-review").on(trigger)
    .when(new LambdaExpressionEvaluator(ctx ->
        Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
        !Boolean.TRUE.equals(ctx.getPath("codeAnalysis.architectureCrossing")) &&
        precedentActivates(ctx, ReviewDomain.ARCHITECTURE_REVIEW) &&
        ctx.get("architectureReview") == null))
    .contextWrite(Map.of("architectureReview", Map.of("activationSource", "precedent")))
    .capability(archReviewCap)
    .conflictResolverStrategy(DEEP_MERGE)
    .outcomePolicy(REROUTE_POLICY)
    .build());
```

- [ ] **Step 7: Update pr-review.yaml — goal conditions**

Update `pr-approved` goal condition:
```yaml
condition: >-
  .pr.status == "merged" or
  (((.codeAnalysis.securitySensitive == false and .securityReview == null) or
    .securityReview.outcome == "APPROVED") and
   ((.codeAnalysis.architectureCrossing == false and .architectureReview == null) or
    .architectureReview.outcome == "APPROVED") and
   .styleCheck.outcome == "APPROVED" and
   .testCoverage.outcome == "APPROVED" and
   .performanceAnalysis.outcome == "APPROVED")
```

Update `security-verified` goal condition:
```yaml
condition: >-
  .pr.status == "merged" or
  (.codeAnalysis.securitySensitive == false and .securityReview == null) or
  .securityReview.outcome == "APPROVED"
```

- [ ] **Step 8: Update pr-review.yaml — merge binding conditions**

Update `enqueue-for-merge` and `merge-direct` — same pattern on the security and architecture lines:
```yaml
((.codeAnalysis.securitySensitive == false and .securityReview == null) or .securityReview.outcome == "APPROVED") and
((.codeAnalysis.architectureCrossing == false and .architectureReview == null) or .architectureReview.outcome == "APPROVED") and
```

- [ ] **Step 9: Add precedent bindings to pr-review.yaml**

Insert after `performance-analysis` binding, before `human-approval`:

```yaml
- name: precedent-security-review
  on: { contextChange: {} }
  when: >-
    .codeAnalysis.complete == true and
    .codeAnalysis.securitySensitive == false and
    (.memory.precedentActivations // [] | any(. == "security-review")) and
    .securityReview == null
  contextWrite:
    securityReview:
      activationSource: precedent
  capability: security-review
  conflictResolverStrategy: DEEP_MERGE
  outcomePolicy:
    onDecline: REROUTE
    onFailure: REROUTE
    onExpired: REROUTE
    maxRerouteAttempts: 2

- name: precedent-architecture-review
  on: { contextChange: {} }
  when: >-
    .codeAnalysis.complete == true and
    .codeAnalysis.architectureCrossing == false and
    (.memory.precedentActivations // [] | any(. == "architecture-review")) and
    .architectureReview == null
  contextWrite:
    architectureReview:
      activationSource: precedent
  capability: architecture-review
  conflictResolverStrategy: DEEP_MERGE
  outcomePolicy:
    onDecline: REROUTE
    onFailure: REROUTE
    onExpired: REROUTE
    maxRerouteAttempts: 2
```

- [ ] **Step 10: Verify equivalence test passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest=PrReviewCaseDefinitionEquivalenceTest -Dsurefire.useFile=false`

The test checks `fromDsl.getBindings().hasSameSizeAs(fromYaml.getBindings())` — both should now have 2 additional bindings (matching count). It also checks names and target types match by position — ensure the new bindings appear in the same order in both DSL and YAML.

- [ ] **Step 11: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all tests pass

- [ ] **Step 12: Commit**

```
feat(#132): precedent-triggered capability bindings with goal gating

Refs casehubio/devtown#132
```

---

### Task 6: File out-of-scope issues + final verification

**Files:**
- No code changes

- [ ] **Step 1: File deferred issues under Epic #129**

Create 4 GitHub issues:
1. Per-capability precedent activation thresholds
2. Similarity-weighted evidence accumulation
3. `activationSource: "content-analysis"` on existing bindings
4. Precedent activation for currently-unconditional capabilities

```bash
gh issue create --repo casehubio/devtown --title "feat: per-capability precedent activation thresholds" --body "**Epic:** #129\n\nGlobal thresholds (minFindings=2, minFraction=0.4) work for v1. Per-capability thresholds (e.g., security: lower bar, architecture: higher) may improve activation precision.\n\nDeferred from #132." --label "scale: S" --label "complexity: Low"
```

(Repeat for each.)

- [ ] **Step 2: Full clean build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: all tests pass, zero compilation warnings in devtown modules

- [ ] **Step 3: Commit spec with resolved issue numbers**

Update the spec's Out of Scope section to replace `#TBD` with the filed issue numbers. Commit to workspace.

```
docs(#132): resolve out-of-scope issue references in spec
```
