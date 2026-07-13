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

### Move Precedent to domain/cbr/

`Precedent` is currently in `review/` but depends only on domain types (`SimilarityScore`, `PrFeatureVector`, `UUID`, and now `CapabilityOutcome`). It belongs in `domain/cbr/` alongside the other CBR vocabulary types. Moving it lets `PrecedentActivationPolicy` work with typed `List<Precedent>` without introducing a dependency from `domain/` on `review/`.

Callers in `review/` (`CbrRetrievalService`, `MemoryContext`) and `app/` (`DefaultCbrRetrievalService`, `CaseMemoryRecaller`) update their imports. No semantic change — just the correct module placement.

### PrecedentActivationPolicy

Pure Java class in `devtown-domain/cbr/`. Stateless evaluation. Two methods:

**Typed method** — for direct use and unit testing:

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
}
```

**Serialized-form method** — for use inside binding conditions (reads from the case context's serialized `Map<String, Object>` representation, no `CaseContext` engine API dependency):

```java
public static boolean shouldActivateFromContext(
        List<Map<String, Object>> serializedPrecedents,
        String capability,
        int minFindings, double minFraction) {
    // Extracts capabilityOutcomes from each serialized precedent map,
    // constructs CapabilityOutcome to reuse hadFindings() logic,
    // applies threshold. Returns true if capability should activate.
}
```

Both methods use `CapabilityOutcome.hadFindings()` as the single source of truth for what constitutes a "finding."

**No engine API dependency.** `devtown-domain` does not depend on `casehub-api`. The binding condition lambda in `PrReviewCaseDefinition` (in `review/`, which does depend on the engine API) extracts `memory.precedents` from `CaseContext` as `List<Map<String, Object>>` and passes it to `shouldActivateFromContext`.

### Binding additions in PrReviewCaseDefinition

Two new bindings. Pattern for `precedent-security-review`:

```java
def.getBindings().add(Binding.builder().name("precedent-security-review").on(trigger)
    .when(new LambdaExpressionEvaluator(ctx ->
        Boolean.TRUE.equals(ctx.getPath("codeAnalysis.complete")) &&
        !Boolean.TRUE.equals(ctx.getPath("codeAnalysis.securitySensitive")) &&
        precedentActivates(ctx, ReviewDomain.SECURITY_REVIEW,
            precedentMinFindings, precedentMinFraction) &&
        ctx.get("securityReview") == null))
    .contextWrite(Map.of("securityReview", Map.of("activationSource", "precedent")))
    .capability(securityReviewCap)
    .conflictResolverStrategy(DEEP_MERGE)
    .outcomePolicy(REROUTE_POLICY)
    .build());

// Helper in PrReviewCaseDefinition (review module — has CaseContext access):
@SuppressWarnings("unchecked")
private static boolean precedentActivates(CaseContext ctx, String capability,
        int minFindings, double minFraction) {
    var precedents = (List<Map<String, Object>>) ctx.getPath("memory.precedents");
    return PrecedentActivationPolicy.shouldActivateFromContext(
        precedents, capability, minFindings, minFraction);
}
```

Same shape for `precedent-architecture-review` (checks `!codeAnalysis.architectureCrossing`, writes to `architectureReview`).

**Condition logic:**
1. `codeAnalysis.complete` — wait for content analysis first
2. `!codeAnalysis.securitySensitive` — only fire when content analysis didn't already trigger
3. `PrecedentActivationPolicy.shouldActivate(...)` — threshold check against precedent data
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

`PrReviewCaseDefinition.build()` gains two parameters (`precedentMinFindings`, `precedentMinFraction`) captured in binding condition closures. The caller resolves them from preferences at case definition build time.

## Change scope

| File | Module | Change |
|------|--------|--------|
| `CapabilityOutcome.java` | domain/cbr | New record |
| `PrecedentActivationPolicy.java` | domain/cbr | New class (typed + serialized-form methods) |
| `CbrPreferenceKeys.java` | domain/cbr | Two new preference keys |
| `Precedent.java` | review → domain/cbr | Move to domain + change `capabilityOutcomes` type |
| `MemoryContext.java` | review | Update `toContextMap()` serialization, update Precedent import |
| `CbrRetrievalService.java` | review | Update Precedent import |
| `PrReviewCaseDefinition.java` | review | Two new bindings, new build parameters |
| `DefaultCbrRetrievalService.java` | app | `enrichOutcomes()` retrieves OUTCOME_DETAIL |
| `DefaultCbrRetrievalService.java` | app | `aggregateOutcome()` works with CapabilityOutcome |

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

- `DefaultCbrRetrievalServiceTest` — enrichOutcomes returns CapabilityOutcome with detail
- `MemoryContextTest` — serialization with richer type
- Binding integration test — precedent binding fires when content analysis doesn't trigger but threshold met; doesn't fire when content analysis already triggers; `activationSource: "precedent"` in context

## End-to-end data flow

```
PR arrives
  → CaseMemoryRecaller.recall()
    → DefaultCbrRetrievalService.findSimilar()
      → enrichOutcomes() retrieves OUTCOME + OUTCOME_DETAIL
      → returns List<Precedent> with Map<String, CapabilityOutcome>
  → MemoryContext.toContextMap() serializes with outcome + detail
  → Case created with memory.precedents in initial context

Engine evaluates bindings on context change:
  → initial-analysis fires → produces codeAnalysis

  Path A (content-triggered):
    → codeAnalysis.securitySensitive == true
    → security-review binding fires (existing, unchanged)

  Path B (precedent-triggered):
    → codeAnalysis.securitySensitive == false
    → precedent-security-review evaluates memory.precedents
    → PrecedentActivationPolicy: 3/5 precedents had security findings → activate
    → contextWrite: {securityReview: {activationSource: "precedent"}}
    → dispatches security-review capability

  Path C (no activation):
    → codeAnalysis.securitySensitive == false
    → PrecedentActivationPolicy: 0/5 precedents had findings → skip
    → security-review never fires

EventLog captures which binding fired, including activation source.
```

## Garden entries referenced

- **GE-20260706-56a75c** — WorkerOutcomeResolvedEvent fires only for non-success outcomes. Relevant if extending outcome recording to include detail.
- **GE-20260710-31b535** — jsonschema2pojo enum fromValue() expects kebab-case. Relevant if adding CBR-related enums to YAML case definitions.

## Out of scope

- Per-capability precedent activation thresholds (global is sufficient for v1)
- Similarity-weighted evidence accumulation (count-based is sufficient for v1)
- Adding `activationSource: "content-analysis"` to existing bindings (minor symmetry enhancement)
- Precedent activation for currently-unconditional capabilities (they already fire)
