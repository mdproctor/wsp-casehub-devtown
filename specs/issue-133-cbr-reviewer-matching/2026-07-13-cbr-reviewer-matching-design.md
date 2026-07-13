# CBR-Enhanced Reviewer Matching — Design Spec

**Issue:** devtown#133
**Epic:** #129 (Epic 11: Case-Based Reasoning), Phase 2
**Date:** 2026-07-13
**Status:** Draft

## Problem

Trust-weighted reviewer routing selects agents based on accumulated trust scores and workload. When two agents are both QUALIFIED for a capability, the one with the higher trust-workload blend wins — regardless of how well each performed on *similar past cases*.

CBR retrieval already delivers per-worker, per-capability outcome data from similar cases via `AgentRoutingContext.experiences()` and `ExperiencePlanStep.planTrace()`. The trust routing strategy receives this data and ignores it.

## Solution

Enhance `TrustWeightedAgentStrategy` to incorporate a similarity-weighted success rate bonus from CBR experiences. When similar past cases exist and a `cbrWeight` is configured, agents that performed well on those cases receive a scoring boost. When no experiences exist or `cbrWeight` is zero, behavior is identical to today.

Three deliverables across two repos plus devtown integration:

1. **`ExperienceAnalyser`** (engine-api) — shared static utility for computing per-worker success rates from plan trace data
2. **Enhanced `TrustWeightedAgentStrategy`** (engine-ledger) — incorporates CBR bonus using the shared utility
3. **Devtown integration** — CBR config, feature extractor, policy configuration

## Architecture

### Repo placement rationale

The shared computation (`ExperienceAnalyser`) goes in **engine-api** where `RetrievedExperience` and `ExperiencePlanStep` live — co-located with the types it operates on.

The strategy enhancement stays in **engine-ledger** where `TrustWeightedAgentStrategy` already lives. The CBR bonus is data-driven, not a new strategy — when no experiences are available, the behavior is identical to the existing trust-weighted routing. No new class or CDI priority conflict.

`casehub-blocks` has an existing `CbrAgentRoutingStrategy` with an identical `analyseExperiences()` computation. That strategy serves a different contract (CBR-first routing with trust pre-filtering, named strategy activation). blocks#55 tracks refactoring it to use the shared `ExperienceAnalyser` once it lands. Not part of this implementation.

### Data flow

```
PR submitted
  → engine CbrRetrievalService retrieves similar cases (cbr: config in YAML)
  → RetrievedExperience objects with planTrace (per-worker, per-capability outcomes)
  → AgentRoutingContext.experiences() populated

Binding fires, capability routing needed
  → TrustWeightedAgentStrategy.select() called
  → TrustCandidateClassifier classifies candidates (BOOTSTRAP/QUALIFIED/BORDERLINE/EXCLUDED)
  → ExperienceAnalyser.workerSuccessRates() computes per-worker CBR scores
  → QUALIFIED candidates scored: trustBlend × (1 - cbrWeight) + cbrBonus × cbrWeight
  → BOOTSTRAP candidates scored: workloadScore (no CBR bonus)
  → classifier.decide() picks the winner
```

## Component 1: `ExperienceAnalyser` (engine-api)

**Package:** `io.casehub.api.spi.routing`
**Location:** alongside `RetrievedExperience`, `ExperiencePlanStep`, `RoutingOutcome`

```java
public final class ExperienceAnalyser {

    public static final Map<RoutingOutcome, Double> DEFAULT_OUTCOME_WEIGHTS = Map.of(
            RoutingOutcome.SUCCESS, 1.0,
            RoutingOutcome.FAILURE, 0.0,
            RoutingOutcome.GATE_REJECTED, 0.0,
            RoutingOutcome.GATE_EXPIRED, 0.0);

    public static Map<String, Double> workerSuccessRates(
            List<RetrievedExperience> experiences,
            Set<String> eligibleWorkerIds,
            String capabilityName,
            Map<RoutingOutcome, Double> outcomeWeights) { ... }
}
```

### Algorithm

For each experience where `similarityScore > 0.0`:
- For each `planTrace` step where `capabilityName` matches AND `workerName` is non-null AND `workerName` is in `eligibleWorkerIds`:
  - Parse `stepOutcome` to `RoutingOutcome` (skip unknown values silently)
  - `weightedSum += outcomeWeight × similarityScore`
  - `evidenceMass += similarityScore`
- Per-worker score = `weightedSum / evidenceMass` (omitted when `evidenceMass == 0`)

Returns `Map<workerId, score>` with scores in [0.0, 1.0]. Empty map when no experiences match.

### Default outcome weights

| RoutingOutcome | Weight | Rationale |
|---|---|---|
| SUCCESS | 1.0 | Full positive signal |
| FAILURE | 0.0 | No positive signal |
| GATE_REJECTED | 0.0 | Human rejected the agent's action |
| GATE_EXPIRED | 0.0 | No evidence of quality |

Matches blocks' `CbrOutcomeWeights.DEFAULT`. Consumers can override via the `outcomeWeights` parameter.

## Component 2: Enhanced `TrustWeightedAgentStrategy` (engine-ledger)

### TrustRoutingPolicy change

New field: `cbrWeight` (double, default `0.0`).

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
    double cbrWeight                          // ← new
) { ... }
```

Default `0.0` means no behavioral change for any app that doesn't configure it. `TrustRoutingPolicy.DEFAULT` gets `cbrWeight: 0.0`.

### TrustRoutingPolicyKeys change

New preference key: `cbr-weight` under the trust routing scope. `TrustRoutingPolicyResolver.resolve()` reads it alongside the existing keys.

### Scoring formula

```
BOOTSTRAP:
  finalScore = workloadScore                    // unchanged — no CBR bonus

QUALIFIED:
  trustBlend = trustScore × blendFactor + workload × (1 - blendFactor)

  if cbrWeight == 0.0 OR experiences empty OR no matching worker data:
      finalScore = trustBlend                   // identical to today

  else:
      cbrBonus = ExperienceAnalyser.workerSuccessRates(...).getOrDefault(workerId, 0.0)
      finalScore = trustBlend × (1 - cbrWeight) + cbrBonus × cbrWeight

BORDERLINE / EXCLUDED:
  finalScore = 0.0                              // unchanged — CBR cannot rescue
```

### Rationale string

When CBR bonus is active (cbrWeight > 0 AND bonus > 0):
```
"selected %s: trust %.2f, cbr_bonus %.2f, blended %.2f (threshold %.2f, cbrWeight %.2f)"
```

When CBR is inactive (cbrWeight == 0 or no experiences):
```
"selected %s: trust %.2f, blended %.2f (threshold %.2f)"    // unchanged from today
```

### Strategy constraints (from issue)

- Zero precedents = identical result to pure trust routing ✓ (cbrBonus defaults to 0.0)
- Bootstrap phase ignores similarity bonus ✓ (BOOTSTRAP branch unchanged)
- Bonus bounded — cannot override a below-threshold trust score ✓ (BORDERLINE/EXCLUDED score 0.0 regardless)
- Routing decision log includes precedent attribution ✓ (rationale string)

## Component 3: Devtown integration

### 3a. Engine-level CBR config (`pr-review.yaml`)

Add `cbr:` block to the case definition to enable `CbrRetrievalService` population of `AgentRoutingContext.experiences()`:

```yaml
cbr:
  cbrType: plan
  timing: CASE_LIFETIME
  topK: 5
  minSimilarity: 0.3
  featureExtractor:
    type: lambda
```

`timing: CASE_LIFETIME` retrieves once at case open and caches — PR features don't change during a review.

### 3b. `DevtownRoutingFeatureExtractor`

**Package:** `io.casehub.devtown.app.routing`
**Implements:** `RoutingFeatureExtractor` (blocks SPI, available via engine-api)

Extracts PR feature vector fields from the case context and maps them to `FeatureValue` entries for CBR queries:

| Case context field | Feature key | FeatureValue type |
|---|---|---|
| `.pr.repo` | `repo` | string |
| `.pr.linesChanged` | `lines-changed` | numeric |
| `.pr.changedPaths` | `changed-paths` | set |
| derived from `.pr.changedPaths` via `ModulePathNormalizer` | `modules` | set |
| derived from `.pr.changedPaths` via extension mapping | `languages` | set |
| derived from `.pr.changedPaths` via test path pattern | `has-tests` | boolean |
| derived from `.pr.changedPaths` via config pattern | `touched-configs` | boolean |

The feature extractor applies the same derivation logic as `PrFeatureVector.from()` — modules, languages, hasTests, and touchedConfigs are computed from changed paths, not stored directly in the case context.

`@ApplicationScoped` — displaces the `@DefaultBean` `TextOnlyFeatureExtractor`.

### 3c. Policy configuration

`DevtownTrustRoutingPolicyProvider.forCapability()` adds `cbrWeight` to the returned `TrustRoutingPolicy`:

| Capability | cbrWeight | Rationale |
|---|---|---|
| `security-review` | 0.2 | CBR signal valuable — similar PRs reveal reviewer strengths |
| `architecture-review` | 0.2 | Same reasoning |
| `style-review` | 0.2 | Same reasoning |
| `test-coverage` | 0.2 | Same reasoning |
| `performance-analysis` | 0.2 | Same reasoning |
| `code-analysis` | 0.0 | Analysis is deterministic — agent quality less variable |
| `merge-executor` | 0.0 | Irreversible — pure trust, no CBR boost |
| `ci-runner` | 0.0 | Deterministic execution |

`cbrWeight` reads from preferences (scope: `casehubio.devtown.trust-routing.<capability>.cbr-weight`) so operators can tune per-capability without code changes.

## Component 4: Deferred — blocks#55

Refactor `CbrAgentRoutingStrategy.analyseExperiences()` to delegate to the shared `ExperienceAnalyser`. No behavioral change. Blocked on engine-api shipping the utility. Tracked at https://github.com/casehubio/blocks/issues/55.

## Testing strategy

### engine-api (`ExperienceAnalyser`)

| Test | What it verifies |
|---|---|
| Empty experiences list | Returns empty map |
| Experiences with no matching capability | Returns empty map |
| Experiences with no matching worker | Returns empty map |
| Single experience, single matching step (SUCCESS) | Score = 1.0 × similarity |
| Single experience, single matching step (FAILURE) | Score = 0.0 |
| Multiple experiences, varying similarity scores | Correctly weighted average |
| Unknown `stepOutcome` string | Skipped, not counted |
| Custom outcome weights | Overrides applied correctly |
| Zero similarity score | Experience skipped |
| Multiple workers in same experience | Independent scores per worker |

### engine-ledger (`TrustWeightedAgentStrategy` enhancement)

| Test | What it verifies |
|---|---|
| `cbrWeight = 0.0` (all existing tests) | Identical to pre-change behavior — regression suite |
| `cbrWeight = 0.2`, empty experiences | Identical to pure trust routing |
| `cbrWeight = 0.2`, one agent with CBR bonus | Score = trust × 0.8 + cbrBonus × 0.2 |
| Agent with lower trust but higher CBR bonus wins | Issue acceptance criterion |
| Zero precedents = same result as pure trust | Issue constraint |
| BOOTSTRAP candidates get no CBR bonus | Issue constraint |
| BORDERLINE agent not rescued by CBR | Issue constraint |
| Rationale includes `cbr_bonus` when active | Transparent attribution |
| Rationale unchanged when CBR inactive | No noise |

### devtown

| Test | What it verifies |
|---|---|
| `DevtownRoutingFeatureExtractor` maps PR fields correctly | Feature extraction |
| `DevtownTrustRoutingPolicyProvider` returns `cbrWeight = 0.2` for review caps | Policy config |
| `DevtownTrustRoutingPolicyProvider` returns `cbrWeight = 0.0` for merge-executor | Policy config |
| Integration: agent selection shifts with CBR data | End-to-end acceptance |

## Cross-repo dependency order

1. **engine-api** — `ExperienceAnalyser` utility (no deps on other changes)
2. **engine-ledger** — `TrustRoutingPolicy.cbrWeight` + strategy enhancement (depends on 1)
3. **devtown** — feature extractor, policy config, YAML config (depends on 2)
4. **blocks#55** — deferred refactoring (depends on 1, independent of 2-3)
