# Evolution Apply-Stage Workers — Design Spec

**Issue:** casehubio/devtown#266
**Branch:** issue-266-evolution-apply-workers
**Date:** 2026-10-27

## Overview

DevTown's evolution conductor proposes operational improvements when capability area health drops below 0.6. The improvement lifecycle flows through 5 stages: analyze, propose (gated), review (gated), **apply**, observe. This issue delivers the apply-stage workers — the code that executes approved improvement proposals by modifying runtime preferences.

Five workers, one per evolution category:

| Worker | Category | What it tunes |
|--------|----------|---------------|
| `ReviewerCalibrationWorker` | `reviewer-calibration` | Trust thresholds, min observations, blend factors, quality floors |
| `RoutingAdjustmentWorker` | `routing-adjustment` | Blend factors, quality floor thresholds, fallback routing types |
| `SlaThresholdWorker` | `sla-tuning` | SLA durations per priority lane, escalation hours |
| `GateStrictnessWorker` | `gate-tightening` | Trust gate floor, human approval threshold, risk thresholds |
| `QueuePriorityWorker` | `queue-optimization` | Batch size, decay rate, failure rate window, risk threshold |

## Execution Model

Workers write to `PreferenceProvider` directly in the apply stage. No PRs, no code review pipeline — the conductor's gated propose and review stages provide human oversight before apply fires.

### Audit Trail

Every preference change is auditable through multiple channels:

1. **Case EventLog** — records every stage transition with the proposal, health score, and trigger
2. **Conductor inbox** — records who approved the propose and review gates, with reasoning
3. **Worker output** — each worker returns a structured `WorkerResult` containing `oldValues`, `newValues`, `rationale`, and `adjustmentMethod` (proportional or cbr-informed)
4. **CaseLedgerEntry** — tamper-evident Merkle record on case closure
5. **ImprovementOutcomeEventCapture** — feeds outcome to regression detector, CBR projector, signal projector

### Self-Improvement Template Compatibility

The engine's `self-improvement.yaml` routes operational improvements through: `introspect → implement → submit-pr → review → integrate → record-outcome`. Since preference writes don't produce PRs, the implement worker applies the change and emits the `improvement:pr-submitted` signal with status `MERGED`. Downstream bindings (review, integrate, record-outcome) resolve cleanly without engine template changes.

## Architecture

### Module Placement

All workers live in `app/src/main/java/io/casehub/devtown/app/evolution/worker/` alongside the existing evolution infrastructure. The `worker/` subdirectory keeps them visually distinct from the capability areas and proposal source.

```
app/src/main/java/io/casehub/devtown/app/evolution/
  worker/
    AdjustmentStrategy.java              -- strategy interface
    ProportionalAdjustmentStrategy.java  -- default: bounded proportional
    CbrInformedAdjustmentStrategy.java   -- auto-promotes when precedents >= min
    AdjustmentResult.java                -- record: parameter, oldValue, newValue, method
    ReviewerCalibrationWorker.java
    RoutingAdjustmentWorker.java
    SlaThresholdWorker.java
    GateStrictnessWorker.java
    QueuePriorityWorker.java
```

### Worker Structure

Each worker is an `@ApplicationScoped` CDI bean that provides a `WorkerFunction.Sync<Map, Map>`. Workers inject `PreferenceProvider` to read current values and write new ones, and the appropriate `CapabilityArea` to read current health scores.

```java
@ApplicationScoped
public class ReviewerCalibrationWorker {

  @Inject PreferenceProvider preferenceProvider;
  @Inject ReviewerTrustCapabilityArea capabilityArea;
  @Inject AdjustmentStrategy adjustmentStrategy;

  public WorkerFunction.Sync<Map, Map> createFunction() {
    return new WorkerFunction.Sync<>(Map.class, Map.class, (input, scope) -> {
      // 1. Read current preference values
      // 2. Read current health assessment
      // 3. Compute adjustment via strategy
      // 4. Write new values to PreferenceProvider
      // 5. Return WorkerResult with old/new values for audit
    });
  }
}
```

### Worker Registration

Workers register their functions with the engine's worker infrastructure. Each worker provides a function for the `improvement-implement` capability, scoped to its category. The engine's `WorkerFunctionProviderRegistry` discovers them via CDI.

The category is determined by the improvement case's context — the `ImprovementGoalFormationStrategy` sets the category when creating the improvement case, and the worker reads it from the input map.

### Adjustment Strategy

`AdjustmentStrategy` is an interface with two implementations:

```java
public interface AdjustmentStrategy {
  AdjustmentResult compute(
      String parameter,
      double currentValue,
      double healthScore,
      double targetHealth,
      AdjustmentBounds bounds,
      String category,
      UUID caseId);
}
```

**`ProportionalAdjustmentStrategy`** (default):
- Computes adjustment proportional to the health gap: `delta = (targetHealth - healthScore) * bounds.stepFactor()`
- Clamps result to `[bounds.floor(), bounds.ceiling()]`
- Step size clamped to `[bounds.minStep(), bounds.maxStep()]`
- Direction determined by parameter semantics (some parameters increase when health is low, others decrease)

**`CbrInformedAdjustmentStrategy`** (auto-promotes):
- Queries `EventLogRepository` for completed improvement cases in the same category
- Filters for cases with similar health gap (within configurable tolerance, e.g. +/- 0.1)
- If precedents >= configurable minimum (default 5): picks the adjustment that produced the best health improvement
- CBR recommendation still constrained by proportional bounds (safety guardrail)
- If insufficient precedents: delegates to `ProportionalAdjustmentStrategy`

**`AdjustmentResult`** record:
```java
public record AdjustmentResult(
    String parameter,
    double oldValue,
    double newValue,
    String method,       // "proportional" or "cbr-informed"
    int precedentCount,  // 0 for proportional
    String rationale) {}
```

### AdjustmentBounds

Per-parameter safety bounds, configured as constants in each worker:

```java
public record AdjustmentBounds(
    double floor,      // absolute minimum value
    double ceiling,    // absolute maximum value
    double minStep,    // minimum adjustment per cycle
    double maxStep,    // maximum adjustment per cycle
    double stepFactor, // multiplier for health-gap-proportional adjustment
    Direction direction) {} // INCREASE_ON_LOW_HEALTH or DECREASE_ON_LOW_HEALTH
```

## Worker Specifications

### 1. ReviewerCalibrationWorker

**Category:** `reviewer-calibration`
**Trigger:** `trust-drift` (ReviewerTrustCapabilityArea health < 0.6)
**Capability area:** `ReviewerTrustCapabilityArea`

**Parameters tuned:**

| Parameter | Preference key | Current default | Bounds | Direction |
|-----------|---------------|-----------------|--------|-----------|
| Trust threshold (per capability) | `routing.{capability}.threshold` | 0.50-0.80 | [0.30, 0.95] | INCREASE when low health (tighten) |
| Minimum observations | `routing.{capability}.min-observations` | 5-15 | [3, 30] | INCREASE when low health |
| Blend factor | `routing.{capability}.blend-factor` | 0.50-0.80 | [0.20, 1.0] | INCREASE when low health |
| Quality floor (per dimension) | `routing.{capability}.floor.{dimension}` | 0.60-0.70 | [0.30, 0.95] | INCREASE when low health |

**Migration required:** `DevtownCapabilityRegistry.POLICIES` threshold, minimumObservations, and borderlineMargin values must be migrated to PreferenceProvider with the hardcoded values as defaults.

**Preference scope path:** `casehubio/devtown/trust-routing/{capability}`

### 2. RoutingAdjustmentWorker

**Category:** `routing-adjustment`
**Trigger:** `review-quality-drift` (ReviewQualityCapabilityArea health < 0.6)
**Capability area:** `ReviewQualityCapabilityArea`

**Parameters tuned:**

| Parameter | Preference key | Current default | Bounds | Direction |
|-----------|---------------|-----------------|--------|-----------|
| Blend factor (per capability) | `routing.{capability}.blend-factor` | 0.50-0.80 | [0.20, 1.0] | Context-dependent |
| Quality floor (per dimension) | `routing.{capability}.floor.{dimension}` | 0.60-0.70 | [0.30, 0.95] | Context-dependent |
| Fallback type | `routing.{capability}.fallback-type` | varies | enum | N/A (discrete) |

**Overlap with ReviewerCalibrationWorker:** Both touch blend factors and quality floors. `DevtownConflictStrategy` prevents concurrent execution in the same category. The workers' different triggers and capability area assessments ensure they fire for different reasons — calibration responds to trust drift (accuracy), routing responds to review quality drift (coverage/timeliness).

**Preference scope path:** `casehubio/devtown/trust-routing/{capability}`

### 3. SlaThresholdWorker

**Category:** `sla-tuning`
**Trigger:** `sla-drift` (SlaComplianceCapabilityArea health < 0.6)
**Capability area:** `SlaComplianceCapabilityArea`

**Parameters tuned:**

| Parameter | Preference key | Current default | Bounds | Direction |
|-----------|---------------|-----------------|--------|-----------|
| SLA critical | `merge-queue.sla.critical` | PT1H | [PT30M, PT4H] | INCREASE when low health (relax) |
| SLA high | `merge-queue.sla.high` | PT4H | [PT1H, PT12H] | INCREASE when low health |
| SLA normal | `merge-queue.sla.normal` | PT8H | [PT2H, PT24H] | INCREASE when low health |
| Escalation hours | `sla.escalation-hours` | 8 | [2, 24] | Context-dependent |
| Completion hours | `sla.completion-hours` | 24 | [8, 72] | INCREASE when low health |

**Note:** SLA values are `Duration` types, not doubles. The `AdjustmentStrategy` interface needs a Duration-aware overload, or SLA values are converted to minutes for computation and back to Duration for storage.

**Preference scope path:** `casehubio/devtown/merge-queue` and `casehubio/devtown/sla`

### 4. GateStrictnessWorker

**Category:** `gate-tightening`
**Trigger:** `ci-degradation` (CiReliabilityCapabilityArea health < 0.6)
**Capability area:** `CiReliabilityCapabilityArea`

**Parameters tuned:**

| Parameter | Preference key | Current default | Bounds | Direction |
|-----------|---------------|-----------------|--------|-----------|
| Trust gate floor | `trust-gate.min-obligor-trust` | 0.30 | [0.10, 0.60] | INCREASE when low health (tighten) |
| Human approval threshold (lines) | `pr-review.human-approval-threshold` | 500 | [100, 2000] | DECREASE when low health (review more) |
| Security severity threshold | `risk.security-severity-threshold` | HIGH | enum | N/A (discrete) |
| Min approved reviews | `risk.merge-min-approved-reviews` | 1 | [1, 3] | INCREASE when low health |

**Preference scope path:** `casehubio/devtown/trust-gate`, `casehubio/devtown/pr-review`, `casehubio/devtown/risk`

### 5. QueuePriorityWorker

**Category:** `queue-optimization`
**Trigger:** `queue-degradation` (MergeQueueHealthCapabilityArea health < 0.6)
**Capability area:** `MergeQueueHealthCapabilityArea`

**Parameters tuned:**

| Parameter | Preference key | Current default | Bounds | Direction |
|-----------|---------------|-----------------|--------|-----------|
| Max batch size | `merge-queue.max-batch-size` | 10 | [2, 20] | Context-dependent |
| Min batch size | `merge-queue.min-batch-size` | 1 | [1, 5] | Context-dependent |
| Decay rate per hour | `merge-queue.decay-rate-per-hour` | 125 | [50, 500] | Context-dependent |
| Failure rate window | `merge-queue.failure-rate-window` | 20 | [5, 50] | INCREASE when low health |
| High risk threshold | `queue.high-risk-threshold` | 0.5 | [0.3, 0.8] | DECREASE when low health (more conservative) |
| Failure rate alert threshold | `merge-queue.failure-rate-alert-threshold` | 0.5 | [0.2, 0.8] | DECREASE when low health |

**Migration required:** `DefaultBatchCompositionPolicy.HIGH_RISK_THRESHOLD` (hardcoded 0.5) must be migrated to PreferenceProvider.

**Preference scope path:** `casehubio/devtown/merge-queue`

## Preference Migration Pattern

For hardcoded values that need migration to PreferenceProvider:

1. Add a `PreferenceKeys` constant (or extend existing keys class) with the key name and default value matching the current hardcoded value
2. Update the consuming class to read from `PreferenceProvider` with the hardcoded value as the fallback default
3. The worker writes new values to the same preference key

Example for `DevtownCapabilityRegistry.POLICIES` threshold:

```java
// Before: hardcoded in POLICIES map
RoutingPolicy.builder().threshold(0.70)...

// After: read from PreferenceProvider, hardcoded value as default
double threshold = preferenceProvider.get(
    Path.of("casehubio/devtown/trust-routing/security-review"),
    RoutingPreferenceKeys.THRESHOLD,
    0.70);
```

This is backward-compatible — existing behavior is preserved until a worker writes a new value.

## Worker Output Format

Every worker returns a consistent output structure for audit:

```json
{
  "category": "reviewer-calibration",
  "target": "reviewer-trust",
  "adjustments": [
    {
      "parameter": "routing.security-review.threshold",
      "oldValue": 0.70,
      "newValue": 0.75,
      "method": "proportional",
      "precedentCount": 0,
      "rationale": "Health 0.45, gap 0.15, step +0.05 (clamped to maxStep)"
    }
  ],
  "healthBefore": 0.45,
  "assessmentSummary": "3 of 8 recent reviews had false negatives in security-review",
  "status": "MERGED"
}
```

The `status: "MERGED"` field enables the self-improvement template's downstream bindings to resolve (D4).

## Testing Strategy

### Unit Tests

Each worker tested in isolation with mocked `PreferenceProvider` and `CapabilityArea`:
- Verify correct preference keys read and written
- Verify adjustment bounds enforced (floor, ceiling, minStep, maxStep)
- Verify output structure includes old/new values for audit
- Verify worker returns `WorkerResult.declined()` when health is already above target

### Strategy Tests

`ProportionalAdjustmentStrategy`:
- Health gap proportional scaling
- Bounds clamping (floor, ceiling, minStep, maxStep)
- Direction handling (increase vs decrease on low health)

`CbrInformedAdjustmentStrategy`:
- Falls back to proportional when precedents < minimum
- Selects best-outcome adjustment from precedents
- Constrains CBR recommendation within proportional bounds
- Handles empty and partial precedent sets

### Integration Tests

`@QuarkusTest` with real `PreferenceProvider`:
- Verify preference writes persist and are readable
- Verify workers compose with `DevtownProposalSource` — when health drops, proposals fire, and if the worker runs, the preference actually changes
- Verify `DevtownConflictStrategy` prevents concurrent improvements in the same category
- Verify `DevtownRegressionEvaluator` detects health regression after an adjustment

## References

- `DevtownProposalSource.java` — maps capability areas to improvement categories and triggers
- `DevtownCategoryProvider.java` — declares 5 categories and 5 stages (analyze, propose, review, apply, observe)
- `self-improvement.yaml` — engine case template for improvement lifecycle
- `WorkerFunction.java` / `WorkerResult.java` — worker API SPI
- `ImprovementOutcomeWorkerFunction.java` — engine's outcome recording pattern
- `DependencyUpdateWorker.java` / `LintFixWorker.java` — engine's introspection-only worker pattern
- `DevtownCapabilityRegistry.java` — hardcoded routing policies (migration target)
- `MergeQueuePreferenceKeys.java` — existing preference-based queue config
- `SlaPreferenceKeys.java` — existing preference-based SLA config
- `PrReviewPreferenceKeys.java` — existing preference-based review gate config
- `RiskPreferenceKeys.java` — existing preference-based risk config
- `DefaultBatchCompositionPolicy.java` — hardcoded HIGH_RISK_THRESHOLD (migration target)
- `trust-routing.yaml` — blend factors and quality floors
- `merge-queue.yaml` — queue configuration defaults
- `trust-gate.yaml` — global trust floor
- `SlaEstimator.java` — existing CBR-from-precedents pattern in devtown
- `ImprovementCbrProjector` (engine) — CBR projection from improvement outcomes
- `ImprovementOutcomeEventCapture` (engine) — outcome capture and feedback loop
- `DevtownConflictStrategy.java` — prevents concurrent improvements in same category
- `DevtownRegressionEvaluator.java` — detects health regression (10% delta threshold)
- `failure-cascade-pattern.md` (protocol PP-20260618-fc6a53) — failure recovery pattern
- `transactional-loop-control-flow-exceptions.md` (protocol PP-20260522-f08b62) — control-flow exception boundary rule
