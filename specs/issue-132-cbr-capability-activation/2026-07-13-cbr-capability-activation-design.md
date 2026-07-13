# CBR-Enhanced Capability Activation — Design Spec

**Issue:** casehubio/devtown#132
**Date:** 2026-07-13
**Status:** Draft

## Problem

Capability activation in the PR review CasePlanModel is driven entirely by static content analysis. `security-review` fires when `codeAnalysis.securitySensitive == true`; `architecture-review` fires when `codeAnalysis.architectureCrossing == true`. If content analysis misses a subtle risk — one that isn't visible from the code diff alone — the capability never fires.

Precedent data from similar past cases is already available in the case context (`memory.precedents`), but no binding reads it. If 4/5 similar past PRs had security findings, the current system still won't activate security review unless the content analysis flags it.

## Solution

Add precedent-triggered bindings to the CasePlanModel that activate conditional capabilities when similar past cases had findings, even when content analysis alone wouldn't trigger them. Augments, not replaces, existing content-driven activation.

## Architecture

### Approach: Separate precedent-triggered bindings (not merged conditions)

Each conditional capability gets a dedicated precedent-triggered binding alongside its existing content-triggered binding. This is required because `contextWrite` on `Binding` is static — a merged condition (content OR precedent) cannot record *which* source triggered the activation. Separate bindings give explicit audit trail via `activationSource: "precedent"` in `contextWrite`.

### Data model enrichment: CapabilityOutcome

The activation signal is **findings-based**, not activation-based. "PRs like this tend to have security problems" is a stronger signal than "PRs like this tend to get security-reviewed." The latter is often redundant with content analysis.

Current `Precedent.capabilityOutcomes` is `Map<String, String>` with raw outcomes (COMPLETED/FAILED/DECLINED). COMPLETED doesn't distinguish "approved" from "findings present." The outcome detail exists in the memory store (`OUTCOME_DETAIL` attribute) but `DefaultCbrRetrievalService.enrichOutcomes()` doesn't retrieve it.

**New record:** `CapabilityOutcome` in `devtown-domain/cbr/`

```java
public record CapabilityOutcome(String outcome, String detail) {
    private static final Set<String> SAFE_DETAILS = Set.of("approved", "passed");

    public boolean hadFindings() {
        return "COMPLETED".equals(outcome) &&
               (detail == null || !SAFE_DETAILS.contains(detail.toLowerCase()));
    }
}
```

- COMPLETED + null detail → findings (no explicit approval = assume findings)
- COMPLETED + "approved"/"passed" → no findings
- COMPLETED + anything else → findings
- FAILED → not findings (operational failure, not domain finding)
- DECLINED → not findings (capability boundary)

**Change `Precedent.capabilityOutcomes`** from `Map<String, String>` to `Map<String, CapabilityOutcome>`.

#### Consolidation: MemoryContext.hasRisk()

`MemoryContext.hasRisk()` contains the same "is this a finding" logic as `CapabilityOutcome.hadFindings()`, operating on raw `Memory` attributes with its own `SAFE_OUTCOMES = Set.of("approved", "passed")`. This is the same business rule in two places — they will drift independently.

Refactor `hasRisk()` to delegate to `CapabilityOutcome.hadFindings()` for the COMPLETED-with-detail evaluation. The FAILED check stays separate — it is a different business rule (operational failure is a risk signal, not a domain finding):

```java
private static boolean hasRisk(List<Memory> memories) {
    return memories.stream().anyMatch(m -> {
        String outcome = m.attributes().get(MemoryAttributeKeys.OUTCOME);
        if (ReviewOutcome.FAILED.name().equals(outcome)) return true;
        String detail = m.attributes().get(DevtownMemoryKeys.OUTCOME_DETAIL);
        return new CapabilityOutcome(outcome, detail).hadFindings();
    });
}
```

Remove the now-unused `SAFE_OUTCOMES` constant from `MemoryContext`.

#### Serialization in toContextMap()

`CapabilityOutcome` serializes as `Map<String, String>` with keys `outcome` and `detail`:

```java
"capabilityOutcomes", p.capabilityOutcomes().entrySet().stream()
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        e -> {
            var co = e.getValue();
            var m = new LinkedHashMap<String, String>();
            m.put("outcome", co.outcome());
            if (co.detail() != null) m.put("detail", co.detail());
            return m;
        }))
```

Produces: `{"security-review": {"outcome": "COMPLETED", "detail": "approved"}, ...}`.

Null `detail` is omitted (not serialized as null) to keep the context lean.

### Move Precedent to domain/cbr/

`Precedent` is currently in `review/` but depends only on domain types (`SimilarityScore`, `PrFeatureVector`, `UUID`, and now `CapabilityOutcome`). It belongs in `domain/cbr/` alongside the other CBR vocabulary types. Moving it lets `PrecedentActivationPolicy` work with typed `List<Precedent>` without introducing a dependency from `domain/` on `review/`.

Callers in `review/` (`CbrRetrievalService`, `MemoryContext`) and `app/` (`DefaultCbrRetrievalService`, `CaseMemoryRecaller`) update their imports. No semantic change — just the correct module placement.

### PrecedentActivationPolicy

Pure Java class in `devtown-domain/cbr/`. Stateless evaluation. Single typed method:

```java
public final class PrecedentActivationPolicy {

    public static Set<String> evaluate(
            List<Precedent> precedents,
            int minFindings, double minFraction) {
        if (precedents.isEmpty()) return Set.of();
        Set<String> result = new LinkedHashSet<>();
        Map<String, Long> findingsCounts = countFindings(precedents);
        for (var entry : findingsCounts.entrySet()) {
            long count = entry.getValue();
            if (count >= minFindings &&
                (double) count / precedents.size() >= minFraction) {
                result.add(entry.getKey());
            }
        }
        return Set.copyOf(result);
    }

    private static Map<String, Long> countFindings(List<Precedent> precedents) {
        // For each precedent, for each capability with hadFindings() == true,
        // increment the count for that capability.
    }
}
```

**No engine API dependency.** `devtown-domain` does not depend on `casehub-api`. Binding conditions consume the pre-computed result (see below).

### Pre-computation at recall time

Precedent data is immutable during the case lifecycle — it's set once at case creation via `CaseMemoryRecaller.recall()` and never changes (even on `revisePr()`, which invalidates analysis but preserves memory context). Evaluating activation thresholds once at recall time is semantically equivalent to evaluating on every context change, but architecturally simpler.

`CaseMemoryRecaller.recall()` evaluates `PrecedentActivationPolicy.evaluate()` after retrieving precedents and passes the result into `MemoryContext`:

```java
// In CaseMemoryRecaller.recall():
List<Precedent> precedents = retrievePrecedents(pr, tenantId);
Set<String> activations = evaluateActivations(precedents);
return new MemoryContext(contributorHistory, codeAreaHistory, precedents, activations);
```

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

`MemoryContext` gains a `Set<String> precedentActivations` field:

```java
public record MemoryContext(
    List<Memory> contributorHistory,
    List<Memory> codeAreaHistory,
    List<Precedent> precedents,
    Set<String> precedentActivations
) {
    public static final MemoryContext EMPTY =
        new MemoryContext(List.of(), List.of(), List.of(), Set.of());
    // ...
}
```

`toContextMap()` serializes it:

```java
"precedentActivations", List.copyOf(precedentActivations)
```

This eliminates the need for a serialized-form evaluation method in `PrecedentActivationPolicy`. The policy has one method (`evaluate`) that works on typed `List<Precedent>` — clean and testable. Binding conditions check the pre-computed result.

### Goal and merge binding condition updates

**Critical fix:** existing goal conditions and merge binding conditions use the pattern:

```
(securitySensitive == false) OR (securityReview.outcome == "APPROVED")
```

When `securitySensitive == false` (the precondition for the precedent binding), the first branch evaluates `true` immediately. The precedent-triggered security review runs but nothing gates on it — the case can complete and merge while the review is still running.

**Updated condition:**

```
(securitySensitive == false AND securityReview == null) OR (securityReview.outcome == "APPROVED")
```

"Security is not needed" only when no security review exists (neither content-triggered nor precedent-triggered). Once any binding creates `securityReview`, the case gates on its outcome.

**All affected sites — Java DSL:**

`pr-approved` goal:
```java
var prApproved = Goal.builder()
    .name("pr-approved").kind(GoalKind.SUCCESS)
    .condition(ctx ->
        ((Boolean.FALSE.equals(ctx.getPath("codeAnalysis.securitySensitive")) &&
            ctx.get("securityReview") == null) ||
            "APPROVED".equals(ctx.getPath("securityReview.outcome"))) &&
        ((Boolean.FALSE.equals(ctx.getPath("codeAnalysis.architectureCrossing")) &&
            ctx.get("architectureReview") == null) ||
            "APPROVED".equals(ctx.getPath("architectureReview.outcome"))) &&
        "APPROVED".equals(ctx.getPath("styleCheck.outcome")) &&
        "APPROVED".equals(ctx.getPath("testCoverage.outcome")) &&
        "APPROVED".equals(ctx.getPath("performanceAnalysis.outcome")))
    .build();
```

`security-verified` goal:
```java
var securityVerified = Goal.builder()
    .name("security-verified").kind(GoalKind.SUCCESS)
    .condition(ctx ->
        (Boolean.FALSE.equals(ctx.getPath("codeAnalysis.securitySensitive")) &&
            ctx.get("securityReview") == null) ||
        "APPROVED".equals(ctx.getPath("securityReview.outcome")))
    .build();
```

`enqueue-for-merge` and `merge-direct` bindings: same pattern applied to the security and architecture conditions within their `when` lambdas.

**All affected sites — YAML:**

`pr-approved` goal:
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

`security-verified` goal:
```yaml
condition: >-
  .pr.status == "merged" or
  (.codeAnalysis.securitySensitive == false and .securityReview == null) or
  .securityReview.outcome == "APPROVED"
```

`enqueue-for-merge` and `merge-direct` bindings: same pattern applied to the security and architecture conditions within their `when` expressions.

**Total: 7 condition sites × 2 representations (Java DSL + YAML) = 14 updates.**

### Binding additions in PrReviewCaseDefinition

Two new bindings in both Java DSL and YAML. Pattern for `precedent-security-review`:

**Java DSL:**
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

// Helper — no threshold parameters, just list membership:
@SuppressWarnings("unchecked")
private static boolean precedentActivates(CaseContext ctx, String capability) {
    var activations = (List<String>) ctx.getPath("memory.precedentActivations");
    return activations != null && activations.contains(capability);
}
```

**YAML:**
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
```

Same shape for `precedent-architecture-review` (checks `!codeAnalysis.architectureCrossing`, writes to `architectureReview`, YAML uses `.memory.precedentActivations // [] | any(. == "architecture-review")`).

**Condition logic:**
1. `codeAnalysis.complete` — wait for content analysis first
2. `!codeAnalysis.securitySensitive` — only fire when content analysis didn't already trigger
3. `precedentActivates(...)` — check pre-computed activation result
4. `securityReview == null` — standard double-dispatch guard

**Audit trail:** `contextWrite` writes `activationSource: "precedent"` before capability dispatch. DEEP_MERGE preserves it when capability output merges on top. Final context: `{securityReview: {activationSource: "precedent", outcome: "APPROVED"}}`.

### Preference configuration

Two new keys in `CbrPreferenceKeys`:

| Key | Type | Default | Meaning |
|-----|------|---------|---------|
| `precedent-activation-min-findings` | int | 2 | Minimum precedents with findings in capability X |
| `precedent-activation-min-fraction` | double | 0.4 | Minimum fraction of precedents with findings |

Both must be satisfied (AND). Prevents false positives from thin data:
- 1/1 with findings → blocked by minFindings (only 1 data point)
- 2/100 with findings → blocked by minFraction (2%)

Defaults (2, 0.4): with 5 precedents, need 2+. With 3, need 2+ (67%). With 1, impossible — insufficient evidence.

Global thresholds. Per-capability thresholds can be added later without interface changes.

### Parameter passing

No new parameters for `PrReviewCaseDefinition.build()`. The threshold values are resolved from preferences at recall time in `CaseMemoryRecaller`, not at case definition build time. Binding conditions check the pre-computed `memory.precedentActivations` list — no threshold parameters in closures.

### DefaultCbrRetrievalService changes

`enrichOutcomes()` retrieves `OUTCOME_DETAIL` alongside `OUTCOME`:

```java
private Map<String, CapabilityOutcome> enrichOutcomes(UUID caseId, String contributor, String tenantId) {
    // ... existing query logic ...
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
}
```

`aggregateOutcome()` updated to work with `CapabilityOutcome`:

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

Key semantic improvement: old code checked `allApproved` against raw `"COMPLETED"` which doesn't distinguish approval from findings. New code uses `hadFindings()` — if any capability had findings, the aggregate is "flagged"; otherwise "approved".

## Change scope

| File | Module | Change |
|------|--------|--------|
| `CapabilityOutcome.java` | domain/cbr | New record |
| `PrecedentActivationPolicy.java` | domain/cbr | New class (typed `evaluate` method only) |
| `CbrPreferenceKeys.java` | domain/cbr | Two new preference keys |
| `Precedent.java` | review → domain/cbr | Move to domain + change `capabilityOutcomes` type |
| `MemoryContext.java` | review | Add `precedentActivations` field, update `toContextMap()` serialization, refactor `hasRisk()` to use `CapabilityOutcome`, remove `SAFE_OUTCOMES` constant |
| `CbrRetrievalService.java` | review | Update Precedent import |
| `PrReviewCaseDefinition.java` | review | Two new bindings, updated goal conditions and merge binding conditions (7 sites), new `precedentActivates` helper |
| `pr-review.yaml` | review/resources | Two new bindings, updated goal conditions and merge binding conditions (7 sites) |
| `DefaultCbrRetrievalService.java` | app | `enrichOutcomes()` retrieves OUTCOME_DETAIL, returns `Map<String, CapabilityOutcome>`. `aggregateOutcome()` uses `CapabilityOutcome` and `hadFindings()` |
| `CaseMemoryRecaller.java` | app | `evaluateActivations()` calls `PrecedentActivationPolicy.evaluate()` at recall time, passes result to `MemoryContext` |

## Testing

### Unit tests (domain, no Quarkus)

**`CapabilityOutcomeTest`** — `hadFindings()` truth table:
- COMPLETED + null → true
- COMPLETED + "approved" → false
- COMPLETED + "passed" → false
- COMPLETED + "FINDINGS_PRESENT" → true
- COMPLETED + "flagged" → true
- FAILED + any → false
- DECLINED + any → false

**`PrecedentActivationPolicyTest`** — threshold logic:
- Empty precedents → empty set
- Below minFindings → not activated
- Below minFraction → not activated
- Meets both → activated
- Multiple capabilities evaluated independently
- Exact boundary values (count == minFindings, fraction == minFraction)
- Capability absent from a precedent's outcomes → doesn't count toward that capability

### Integration / wiring updates

- `DefaultCbrRetrievalServiceTest` — enrichOutcomes returns CapabilityOutcome with detail; aggregateOutcome uses hadFindings for flagged/approved distinction
- `MemoryContextTest` — serialization with richer type; `precedentActivations` field serialized; `hasRisk()` delegates to CapabilityOutcome
- `CaseMemoryRecallerTest` — `evaluateActivations()` calls policy with threshold preferences; result passed into MemoryContext; empty precedents produce empty activations
- `PrReviewCaseDefinitionEquivalenceTest` — verify two new bindings exist in both YAML and DSL; verify updated goal/binding count matches
- Binding integration test — precedent binding fires when content analysis doesn't trigger but activation is pre-computed; doesn't fire when content analysis already triggers; `activationSource: "precedent"` in context
- Goal integration test — `pr-approved` and `security-verified` goals gate on `securityReview.outcome == "APPROVED"` when `securityReview` context key exists (even with `securitySensitive == false`); goals pass immediately when no `securityReview` key exists and `securitySensitive == false`

## End-to-end data flow

```
PR arrives
  → CaseMemoryRecaller.recall()
    → DefaultCbrRetrievalService.findSimilar()
      → enrichOutcomes() retrieves OUTCOME + OUTCOME_DETAIL
      → returns List<Precedent> with Map<String, CapabilityOutcome>
    → evaluateActivations(precedents)
      → PrecedentActivationPolicy.evaluate(precedents, minFindings, minFraction)
      → returns Set<String> e.g. {"security-review"}
    → MemoryContext(contributorHistory, codeAreaHistory, precedents, activations)
  → MemoryContext.toContextMap() serializes:
      memory.precedents (with outcome + detail)
      memory.precedentActivations: ["security-review"]
  → Case created with memory in initial context

Engine evaluates bindings on context change:
  → initial-analysis fires → produces codeAnalysis

  Path A (content-triggered):
    → codeAnalysis.securitySensitive == true
    → security-review binding fires (existing, unchanged)
    → Goal condition: securitySensitive==true → first branch false → gates on outcome

  Path B (precedent-triggered):
    → codeAnalysis.securitySensitive == false
    → precedent-security-review checks memory.precedentActivations
    → "security-review" present → binding fires
    → contextWrite: {securityReview: {activationSource: "precedent"}}
    → dispatches security-review capability
    → Goal condition: securitySensitive==false but securityReview!=null
      → first branch false → gates on securityReview.outcome == "APPROVED"

  Path C (no activation):
    → codeAnalysis.securitySensitive == false
    → "security-review" NOT in memory.precedentActivations → skip
    → securityReview remains null
    → Goal condition: securitySensitive==false and securityReview==null
      → first branch true → goal satisfied immediately

EventLog captures which binding fired, including activation source.
```

## Garden entries referenced

- **GE-20260706-56a75c** — WorkerOutcomeResolvedEvent fires only for non-success outcomes. Relevant if extending outcome recording to include detail.
- **GE-20260710-31b535** — jsonschema2pojo enum fromValue() expects kebab-case. Relevant if adding CBR-related enums to YAML case definitions.

## Out of scope

- Per-capability precedent activation thresholds (global is sufficient for v1) — casehubio/devtown#146
- Similarity-weighted evidence accumulation (count-based is sufficient for v1) — casehubio/devtown#147
- Adding `activationSource: "content-analysis"` to existing bindings (minor symmetry enhancement) — casehubio/devtown#148
- Precedent activation for currently-unconditional capabilities (they already fire) — casehubio/devtown#149
