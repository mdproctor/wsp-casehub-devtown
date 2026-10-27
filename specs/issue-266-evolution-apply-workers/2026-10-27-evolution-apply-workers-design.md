# Evolution Apply-Stage Workers — Design Spec

**Issue:** casehubio/devtown#266
**Branch:** issue-266-evolution-apply-workers
**Date:** 2026-10-27

## Overview

DevTown's evolution conductor proposes operational improvements when capability area health drops below 0.6. The improvement lifecycle flows through 5 stages: analyze, propose (gated), review (gated), **apply**, observe. This issue delivers the apply-stage workers — the code that executes approved improvement proposals by modifying runtime preferences.

Five workers, one per evolution category:

| Worker | Category | What it tunes |
|--------|----------|---------------|
| `ReviewerCalibrationWorker` | `reviewer-calibration` | Trust thresholds, min observations, borderline margin (migrated from CapabilityRegistry) |
| `RoutingAdjustmentWorker` | `routing-adjustment` | Blend factors, quality floor thresholds, fallback routing types |
| `SlaThresholdWorker` | `sla-tuning` | SLA durations per priority lane, escalation hours |
| `GateStrictnessWorker` | `gate-tightening` | Trust gate floor, human approval threshold, risk thresholds |
| `QueuePriorityWorker` | `queue-optimization` | Batch size, decay rate, failure rate window, risk threshold |

## Execution Model

Workers read current preference values via `PreferenceProvider.resolve()` and write new values via `PreferenceWriter.write()`. No PRs, no code review pipeline — the conductor's gated propose and review stages provide human oversight before apply fires.

### Platform Prerequisite: PreferenceWriter SPI

`PreferenceProvider` is read-only (`resolve(SettingsScope) → Preferences`). There is no write API in the platform today. This spec requires a new `PreferenceWriter` SPI in `casehub-platform-api`:

```java
public interface PreferenceWriter {
    void write(SettingsScope scope, PreferenceKey<?> key, String value);
    void write(SettingsScope scope, PreferenceKey<?> key, String subKey, String value);
}
```

**Platform implementation:** A `JpaPreferenceWriter` in `casehub-platform-persistence-jpa-common` that persists `PreferenceEntry` records via `EntityManager`. The JPA entity and table (`platform_preference`) already exist with the correct schema (tenancy_id, scope, namespace, pref_name, sub_key, pref_value). The `PreferenceEntryRepository` (Spring JPA) already has `findByTenancyIdAndScopeAndNamespaceAndNameAndSubKey` — the Quarkus/Panache equivalent is a direct `EntityManager` query.

### Platform Prerequisite: LayeredPreferenceProvider

Devtown's runtime uses `ConfigFilePreferenceProvider` — a YAML-backed implementation that loads all preferences from config files at construction time into an immutable in-memory map. Its `resolve()` reads from this map and never queries JPA. Writes via `JpaPreferenceWriter` to the `platform_preference` table would be invisible to any consumer reading through `ConfigFilePreferenceProvider`.

This spec requires a `LayeredPreferenceProvider` that composes two sources:

1. **JPA layer (primary):** Reads from the `platform_preference` table for runtime-written overrides. Uses `EntityManager` queries equivalent to `SpringPreferenceProvider.resolve()` — which already implements JPA-backed preference resolution with scope hierarchy walking.
2. **YAML layer (fallback):** Reads from `ConfigFilePreferenceProvider` for defaults. Existing YAML config files (`trust-routing.yaml`, `trust-gate.yaml`, etc.) continue to provide operator-managed baseline values.

Resolution order: for each key, JPA value wins if present; otherwise YAML default applies. This preserves the existing YAML-based workflow while enabling runtime writes.

```java
public class LayeredPreferenceProvider implements PreferenceProvider {
    private final PreferenceProvider jpaProvider;   // reads from platform_preference table
    private final PreferenceProvider yamlProvider;   // reads from config files

    @Override
    public Preferences resolve(SettingsScope scope) {
        Preferences yamlPrefs = yamlProvider.resolve(scope);
        Preferences jpaPrefs = jpaProvider.resolve(scope);
        // JPA values override YAML defaults
        Map<String, Object> merged = new HashMap<>(yamlPrefs.asMap());
        merged.putAll(jpaPrefs.asMap());
        return new MapPreferences(merged);
    }
}
```

**Platform implementation:** `LayeredPreferenceProvider` in `casehub-platform-persistence-jpa-common`, alongside `JpaPreferenceWriter`. The JPA read path mirrors `SpringPreferenceProvider` (which already resolves preferences from JPA with scope hierarchy) but uses `EntityManager` (Quarkus/Panache) instead of Spring's `JpaRepository`. Devtown's CDI configuration switches from `ConfigFilePreferenceProvider` to `LayeredPreferenceProvider` as the `PreferenceProvider` bean.

**Cross-repo dependency:** This requires issues filed against `casehub-platform-api` (for the `PreferenceWriter` SPI) and `casehub-platform-persistence-jpa-common` (for `JpaPreferenceWriter` and `LayeredPreferenceProvider`). The devtown workers depend on both landing first.

### Tenancy Context

`SettingsScope` requires a `tenancyId`. Workers obtain this from the improvement case's accumulated state — the evolution conductor sets `tenancyId` in the case context when creating the improvement case. Workers read it via `scope.accumulatedState().get("tenancyId")`.

### Audit Trail

Every preference change is auditable through multiple channels:

1. **Case EventLog** — records every stage transition with the proposal, health score, and trigger
2. **Conductor inbox** — records who approved the propose and review gates, with reasoning
3. **Worker output** — each worker returns a structured `WorkerResult` containing `oldValues`, `newValues`, `rationale`, and `adjustmentMethod` (proportional or cbr-informed)
4. **CaseLedgerEntry** — tamper-evident Merkle record on case closure
5. **ImprovementOutcomeEventCapture** — feeds outcome to regression detector, CBR projector, signal projector

### Self-Improvement Template Compatibility

The engine's `self-improvement.yaml` routes operational improvements through: `introspect → implement → submit-pr → review → integrate → record-outcome`.

Since preference writes don't produce PRs, the devtown implement worker must handle the template flow correctly:

1. **Worker output:** The implement worker writes `requiresPr: false`, `reviewOutcome: "approved"`, and `status: "MERGED"` to its output map.

2. **`submit-pr` binding gap (engine template change required):** The `submit-pr` binding fires unconditionally on-complete of `implement`. It must be conditioned:

   ```yaml
   - name: submit-pr
     trigger:
       type: on-complete
       source: implement
     when: "context.layer('WORKING').get('requiresPr') != false"
     capability: improvement-submit-pr
   ```

   This is a cross-repo change to `casehub-engine` (`self-improvement.yaml`). File an issue against the engine repo.

3. **Signal emission:** The implement worker emits the `improvement:pr-submitted` signal via `CaseHubRuntime.signal(caseId, "improvement:pr-submitted", true)`. The worker obtains the `CaseHubRuntime` reference through CDI injection (the `DevtownEvolutionCaseHub` bean, which extends `CaseHub`, exposes `signal(UUID, String, Object)` delegating to the injected `CaseHubRuntime`). The worker receives `caseId` from `WorkerScope.caseId()`.

   Concretely: `DevtownEvolutionCaseHub` is `@ApplicationScoped` and injected into the apply worker. After writing preferences, the worker calls `evolutionCaseHub.signal(scope.caseId(), "improvement:pr-submitted", true)`.

### Key Ownership — No Overlapping Parameters

ReviewerCalibrationWorker and RoutingAdjustmentWorker previously both touched blend factors and quality floors. This is resolved by strict key partitioning:

- **ReviewerCalibrationWorker** owns trust policy base fields: `threshold`, `minimum-observations`, `borderline-margin` (migrated from `DevtownCapabilityRegistry.POLICIES` to preferences). These are per the `harness-trust-policy-source-split` protocol — currently hardcoded in `CapabilityRegistry`, migrated to preferences to enable runtime tuning.

- **RoutingAdjustmentWorker** owns engine-specific tuning parameters: `blend-factor`, `quality-floor.*`, `cbr-weight`, `fallback-type`. These are already preference-backed in the trust-routing YAML.

No parameter is owned by more than one worker. `DevtownConflictStrategy` prevents concurrent improvements in the same category; the key partitioning prevents semantic conflicts across categories.

## Architecture

### Module Placement

Strategy interfaces and implementations (pure domain logic, no framework dependencies) live in `domain/`. Workers (CDI beans with framework annotations) live in `app/`.

```
domain/src/main/java/io/casehub/devtown/domain/evolution/
  AdjustmentStrategy.java              -- strategy interface
  ProportionalAdjustmentStrategy.java  -- default: bounded proportional
  CbrInformedAdjustmentStrategy.java   -- auto-promotes when precedents >= min
  OutcomeQuery.java                    -- functional interface for CBR data access
  AdjustmentResult.java                -- record: parameter, oldValue, newValue, method
  AdjustmentBounds.java                -- record: per-parameter safety bounds

app/src/main/java/io/casehub/devtown/app/evolution/worker/
  ReviewerCalibrationWorker.java
  RoutingAdjustmentWorker.java
  SlaThresholdWorker.java
  GateStrictnessWorker.java
  QueuePriorityWorker.java
```

This follows the three-tier module structure (ARC42STORIES §2): `devtown-domain` (pure Java, zero framework) / `devtown-app` (CDI wiring). `AdjustmentStrategy` and its implementations are pure computation — no CDI, no Quarkus, no JPA. They belong alongside `SlaEstimator` in the domain module.

### Worker Registration

Workers register via the established `YamlCaseHub.augment()` pattern. `DevtownEvolutionCaseHub` (already exists, loads `casehub/devtown/evolution.yaml`) overrides `augment()` to add the five workers:

```java
@ApplicationScoped
public class DevtownEvolutionCaseHub extends YamlCaseHub {

  @Inject ReviewerCalibrationWorker reviewerCalibration;
  @Inject RoutingAdjustmentWorker routingAdjustment;
  @Inject SlaThresholdWorker slaThreshold;
  @Inject GateStrictnessWorker gateStrictness;
  @Inject QueuePriorityWorker queuePriority;

  public DevtownEvolutionCaseHub() {
    super("casehub/devtown/evolution.yaml");
  }

  @Override
  protected void augment(CaseDefinition definition) {
    definition.getWorkers().add(Worker.builder()
        .name("evolution-apply")
        .capabilityName("improvement-implement")
        .function(new WorkerFunction.Sync<>(Map.class, Map.class, this::dispatch))
        .build());
  }

  WorkerResult<Map<String, Object>> dispatch(Map<String, Object> input, WorkerScope scope) {
    String category = (String) input.get("category");
    return switch (category) {
      case "reviewer-calibration" -> reviewerCalibration.apply(input, scope);
      case "routing-adjustment" -> routingAdjustment.apply(input, scope);
      case "sla-tuning" -> slaThreshold.apply(input, scope);
      case "gate-tightening" -> gateStrictness.apply(input, scope);
      case "queue-optimization" -> queuePriority.apply(input, scope);
      default -> WorkerResult.failed("Unknown category: " + category);
    };
  }
}
```

This follows the `augment()` pattern from `MergeBatchCaseHub` and `PrReviewCaseHub`, with one key difference: the dispatch function uses `new WorkerFunction.Sync<>(Map.class, Map.class, this::dispatch)` instead of the convenience `Builder.function(Function<Map, WorkerResult>)` overload. The convenience overload wraps the function as `(t, scope) -> fn.apply(t)` — discarding `WorkerScope`. Evolution workers need `scope.caseId()` (for signal emission) and `scope.accumulatedState()` (for tenancy context), so `WorkerScope` must be preserved. `MergeBatchCaseHub` uses the convenience overload correctly because its workers don't need `WorkerScope`.

### Worker Structure

Each worker is an `@ApplicationScoped` CDI bean. Workers inject `PreferenceProvider` to read current values, `PreferenceWriter` to write new values, and the appropriate `CapabilityArea` to read current health scores.

```java
@ApplicationScoped
public class ReviewerCalibrationWorker {

  @Inject PreferenceProvider preferenceProvider;
  @Inject PreferenceWriter preferenceWriter;
  @Inject ReviewerTrustCapabilityArea capabilityArea;
  @Inject AdjustmentStrategy adjustmentStrategy;

  public WorkerResult apply(Map<String, Object> input, WorkerScope scope) {
    String tenancyId = (String) scope.accumulatedState().get("tenancyId");
    String capability = (String) input.get("target");

    // 1. Read current preference values
    SettingsScope readScope = SettingsScope.of(tenancyId,
        Path.parse("casehubio/devtown/trust-routing/" + capability));
    Preferences prefs = preferenceProvider.resolve(readScope);

    TrustRoutingPolicyKeys keys = TrustRoutingPolicyKeys.create("casehubio.devtown.trust-routing");
    DoublePreference currentThreshold = prefs.getOrDefault(keys.threshold());

    // 2. Read current health assessment
    double healthScore = capabilityArea.health();

    // 3. Compute adjustment via strategy
    AdjustmentResult result = adjustmentStrategy.compute(
        "threshold", currentThreshold.value(), healthScore, 0.6,
        THRESHOLD_BOUNDS, "reviewer-calibration", /* caseId from input */);

    // 4. Write new value via PreferenceWriter
    SettingsScope writeScope = SettingsScope.of(tenancyId,
        Path.parse("casehubio/devtown/trust-routing/" + capability));
    preferenceWriter.write(writeScope, keys.threshold(),
        String.valueOf(result.newValue()));

    // 5. Return WorkerResult with old/new values for audit
    return WorkerResult.of(Map.of(
        "category", "reviewer-calibration",
        "adjustments", List.of(result.toMap()),
        "healthBefore", healthScore,
        "status", "MERGED",
        "reviewOutcome", "approved",
        "requiresPr", false));
  }
}
```

### Adjustment Strategy

`AdjustmentStrategy` is a pure domain interface in `domain/` with two implementations:

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
- Takes an `OutcomeQuery` functional interface as a constructor parameter (not a direct dependency on `ImprovementOutcomeStore`):
  ```java
  @FunctionalInterface
  public interface OutcomeQuery {
      List<ImprovementOutcome> findByCategory(String category, int limit);
  }
  ```
  The CDI producer in `app/` wires this to `outcomeStore::findByCategory`. This keeps the strategy as pure domain computation with no engine-runtime-core dependency — consistent with `ProportionalAdjustmentStrategy` and `SlaEstimator`. `ImprovementOutcome` (the record type) is in `casehub-engine-api`, which `domain/` can depend on; only `ImprovementOutcomeStore` (the runtime storage class) is in engine-runtime-core.
- Queries completed improvement outcomes in the same category via `OutcomeQuery`
- Reads CBR comparison data from `ImprovementOutcome.metadata()` (see CBR Metadata Contract below)
- Filters for outcomes with similar health gap (within configurable tolerance, e.g. +/- 0.1)
- If matching precedents >= configurable minimum (default 5): picks the adjustment that produced the best health improvement
- CBR recommendation still constrained by proportional bounds (safety guardrail)
- If insufficient precedents: delegates to `ProportionalAdjustmentStrategy`

This reuses the engine's existing `ImprovementOutcomeStore` (via the `OutcomeQuery` functional interface) rather than querying `EventLogRepository` directly. `ImprovementCbrProjector` already projects outcomes to the store; the strategy reads from it through the interface boundary.

**CBR Metadata Contract:**

`ImprovementOutcome` has `metadata: Map<String, String>` but `ImprovementOutcomeEventCapture` currently hardcodes it to `Map.of()`. Two changes are needed:

1. **Workers write health data to output:** Each worker includes `healthBefore` (captured at adjustment time) and adjustment details in its output map.

2. **Engine event capture passes metadata through (cross-repo change):** `ImprovementOutcomeEventCapture.onOutcome()` must read well-known keys from `improvementData` (the `_improvement` context in the case file snapshot) and populate `ImprovementOutcome.metadata` instead of hardcoding `Map.of()`. The conductor enriches `_improvement` with health/adjustment data from the apply stage output.

3. **`healthAfter` captured by the evolution conductor, not a template binding:** The engine's `self-improvement.yaml` template has no observe binding — the complete flow is `introspect → implement → submit-pr → review → integrate → record-outcome`. `DevtownCategoryProvider` declares an observe stage descriptor, but stage descriptors are metadata — the engine does not automatically create bindings from them. Rather than adding a template binding (which would require the engine to access harness-specific capability areas), `healthAfter` is captured by the evolution conductor's periodic health polling. After an improvement case completes for a category, the conductor records the category's health at the next poll cycle and updates the outcome metadata. This is a natural fit: the conductor already polls capability area health to detect drift — it simply records the post-adjustment value alongside the completed improvement case ID. The conductor spec (not yet created) will define the exact mechanism.

Well-known metadata keys (string constants defined in devtown `domain/`):

| Key | Type | Source |
|-----|------|--------|
| `evolution.healthBefore` | `Double` (as string) | Apply worker output |
| `evolution.healthAfter` | `Double` (as string) | Evolution conductor (next health poll after case completion) |
| `evolution.adjustments` | JSON array (as string) | Apply worker output |
| `evolution.category` | `String` | Case context |

The `CbrInformedAdjustmentStrategy` parses these from `ImprovementOutcome.metadata()` with fallback to empty/absent. Outcomes lacking `healthAfter` (before the conductor implements this capture) are used for category/health-gap matching but excluded from effectiveness ranking. CBR degrades gracefully: with `healthBefore` only, it can select precedents with similar starting conditions; without `healthAfter`, it cannot rank them by outcome quality and falls back to proportional.

**In-memory limitation:** `ImprovementOutcomeStore` uses `ConcurrentHashMap` with no persistence — CBR data is lost on application restart. The proportional fallback is safe (the strategy delegates to `ProportionalAdjustmentStrategy` when insufficient precedents), so this is a quality-of-learning limitation, not a correctness bug. For long-lived deployments, precedents accumulate naturally. For environments with frequent restarts, a persistent `ImprovementOutcomeStore` backing (JPA or file-based) is a follow-up item — file an issue against `casehub-engine` to track this.

**`AdjustmentResult`** record:
```java
public record AdjustmentResult(
    String parameter,
    double oldValue,
    double newValue,
    String method,       // "proportional" or "cbr-informed"
    int precedentCount,  // 0 for proportional
    String rationale) {

    public Map<String, Object> toMap() {
        return Map.of(
            "parameter", parameter, "oldValue", oldValue,
            "newValue", newValue, "method", method,
            "precedentCount", precedentCount, "rationale", rationale);
    }
}
```

### Duration and Discrete Type Handling

The `AdjustmentStrategy` interface operates on `double` values. Different parameter types are handled as follows:

**Duration parameters (SlaThresholdWorker):** SLA values are stored as `StringPreference` (e.g., `"PT1H"`). The worker converts Duration to minutes (`double`) for computation, then back to Duration ISO-8601 string for storage. Example: `PT1H` → `60.0` → compute → `75.0` → `PT1H15M`. The `AdjustmentResult.rationale` reports the Duration change (e.g., `"PT1H → PT1H15M"`) not the raw numeric delta.

**Integer-typed parameters (QueuePriorityWorker, ReviewerCalibrationWorker):** Several preference keys use `IntPreference` (e.g., `MAX_BATCH_SIZE`, `MIN_BATCH_SIZE`, `FAILURE_RATE_WINDOW`, `DECAY_RATE_PER_HOUR`, `minimumObservations`). Since `AdjustmentStrategy` returns `double`, workers with integer-typed parameters round `AdjustmentResult.newValue()` to the nearest integer (`Math.round()`) before writing via `PreferenceWriter.write()`. The audit rationale reports the rounded value.

**Discrete/enum parameters (GateStrictnessWorker, RoutingAdjustmentWorker):** `security-severity-threshold` (enum: HIGH/MEDIUM/LOW) and `fallback-type` are not tuned through the `AdjustmentStrategy`. They are adjusted by direct worker logic based on health score thresholds — e.g., if health < 0.4, tighten from HIGH to MEDIUM. The `AdjustmentStrategy` pattern is designed for continuous numeric parameters; forcing enum transitions through it would be artificial.

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

| Parameter | PreferenceKey source | Namespace / Name | Current source | Bounds | Direction |
|-----------|---------------------|------------------|----------------|--------|-----------|
| Trust threshold (per capability) | `TrustRoutingPolicyKeys.threshold()` | `casehubio.devtown.trust-routing` / `threshold` | `DevtownCapabilityRegistry.POLICIES` | [0.30, 0.95] | INCREASE when low health (tighten) |
| Minimum observations | `TrustRoutingPolicyKeys.minimumObservations()` | `casehubio.devtown.trust-routing` / `minimum-observations` | `DevtownCapabilityRegistry.POLICIES` | [3, 30] | INCREASE when low health |
| Borderline margin | `TrustRoutingPolicyKeys.borderlineMargin()` | `casehubio.devtown.trust-routing` / `borderline-margin` | `DevtownCapabilityRegistry.POLICIES` | [0.02, 0.15] | Context-dependent |

**Migration required:** `DevtownCapabilityRegistry.POLICIES` threshold, minimumObservations, and borderlineMargin values must be migrated to `PreferenceProvider`. The migration requires:
1. `TrustRoutingPolicyKeys` already defines `PreferenceKey` constants for all three fields (`threshold()`, `minimumObservations()`, `borderlineMargin()`) with defaults of 0.0/0.
2. Update `DevtownTrustRoutingPolicyProvider.forCapability()` to read these from preferences (via the already-resolved `prefs`) instead of from `RoutingPolicy` record. The current code reads `routingPolicy.threshold().orElse(TrustRoutingPolicy.DEFAULT.threshold())` — replace with `prefs.getOrDefault(KEYS.threshold()).value()`.
3. Seed the preference YAML (`trust-routing.yaml`) with the current `POLICIES` values as defaults for each capability scope.
4. The `harness-trust-policy-source-split` protocol must be updated: with evolution workers, these base fields ARE tunable and belong in preferences. The protocol's split (CapabilityRegistry for base fields, YAML for engine-specific) was designed when values were static.
5. Remove the deprecated `RoutingPolicy.isBorderline()` method — it is unused by the routing path (confirmed by `@Deprecated` annotation and its Javadoc). The engine uses `TrustRoutingPolicy.isBorderline(double)` with symmetric semantics.

**Note on `RoutingPolicy` type changes:** After migration, `DevtownCapabilityRegistry.POLICIES` entries may retain threshold/minimumObservations/borderlineMargin as compile-time defaults (wrapped in `OptionalDouble.empty()` / `OptionalInt.empty()`), or the `RoutingPolicy` record can be simplified. The preference system provides the defaults; the registry provides only non-preference fields (fallbackType, rationale).

**Preference scope path:** `casehubio/devtown/trust-routing/{capability}`

### 2. RoutingAdjustmentWorker

**Category:** `routing-adjustment`
**Trigger:** `review-quality-drift` (ReviewQualityCapabilityArea health < 0.6)
**Capability area:** `ReviewQualityCapabilityArea`

**Parameters tuned:**

| Parameter | PreferenceKey source | Namespace / Name | Current source | Bounds | Direction |
|-----------|---------------------|------------------|----------------|--------|-----------|
| Blend factor (per capability) | `TrustRoutingPolicyKeys.blendFactor()` | `casehubio.devtown.trust-routing` / `blend-factor` | trust-routing.yaml | [0.20, 1.0] | Context-dependent |
| Quality floor (per dimension) | `TrustRoutingPolicyKeys.allFloorKeys()` | `casehubio.devtown.trust-routing` / `floor.{dimension}` | trust-routing.yaml | [0.30, 0.95] | Context-dependent |
| CBR weight | `TrustRoutingPolicyKeys.cbrWeight()` | `casehubio.devtown.trust-routing` / `cbr-weight` | `DevtownTrustRoutingPolicyProvider.CBR_WEIGHT_DEFAULTS` | [0.0, 0.5] | Context-dependent |
| Fallback type | *(direct worker logic)* | `RoutingPolicy.fallbackType()` | `DevtownCapabilityRegistry.POLICIES` | enum | N/A (discrete) |

**No overlap with ReviewerCalibrationWorker:** Strict key partitioning — calibration owns base policy fields (threshold, minimumObservations, borderlineMargin); routing owns engine-specific tuning parameters (blend-factor, quality-floor, cbr-weight, fallback-type).

**Preference scope path:** `casehubio/devtown/trust-routing/{capability}`

### 3. SlaThresholdWorker

**Category:** `sla-tuning`
**Trigger:** `sla-drift` (SlaComplianceCapabilityArea health < 0.6)
**Capability area:** `SlaComplianceCapabilityArea`

**Parameters tuned:**

| Parameter | PreferenceKey constant | Namespace / Name | Current default | Bounds (minutes) | Direction |
|-----------|----------------------|------------------|-----------------|------------------|-----------|
| SLA critical | `MergeQueuePreferenceKeys.SLA_CRITICAL` | `devtown.merge-queue.sla` / `CRITICAL` | PT1H (60) | [30, 240] | INCREASE when low health (relax) |
| SLA high | `MergeQueuePreferenceKeys.SLA_HIGH` | `devtown.merge-queue.sla` / `HIGH` | PT4H (240) | [60, 720] | INCREASE when low health |
| SLA normal | `MergeQueuePreferenceKeys.SLA_NORMAL` | `devtown.merge-queue.sla` / `NORMAL` | PT8H (480) | [120, 1440] | INCREASE when low health |
| Escalation hours | `SlaPreferenceKeys.ESCALATION_HOURS` | `devtown.sla` / `escalation-hours` | 8 | [2, 24] | Context-dependent |
| Completion hours | `SlaPreferenceKeys.COMPLETION_HOURS` | `devtown.sla` / `completion-hours` | 24 | [8, 72] | INCREASE when low health |

**Duration handling:** SLA values are stored as `StringPreference` (ISO-8601: `"PT1H"`, `"PT4H"`, etc.). The worker parses to `Duration`, converts to minutes for `AdjustmentStrategy` computation, then formats back to ISO-8601 string for `PreferenceWriter.write()`. The `AdjustmentResult.rationale` reports Duration changes in human-readable form.

**SLA ordering constraint:** After computing all three lane adjustments independently, the worker enforces `CRITICAL < HIGH < NORMAL` with minimum separations: CRITICAL must be at least 30 minutes less than HIGH, and HIGH at least 60 minutes less than NORMAL. If an independent adjustment violates ordering, it is clamped to maintain the invariant. Example: if CRITICAL adjusts to 240min and HIGH adjusts to 200min, HIGH is clamped up to `240 + 30 = 270min`. The clamping is logged in the adjustment rationale so the audit trail shows both the computed and clamped values.

**Preference scope paths:** `casehubio/devtown/merge-queue` (SLA keys) and `casehubio/devtown/sla` (escalation/completion)

### 4. GateStrictnessWorker

**Category:** `gate-tightening`
**Trigger:** `ci-degradation` (CiReliabilityCapabilityArea health < 0.6)
**Capability area:** `CiReliabilityCapabilityArea`

**Parameters tuned:**

| Parameter | PreferenceKey constant | Namespace / Name | Current default | Bounds | Direction |
|-----------|----------------------|------------------|-----------------|--------|-----------|
| Trust gate floor | *(new key needed)* | `casehubio.devtown.trust-gate` / `min-obligor-trust` | 0.30 | [0.10, 0.60] | INCREASE when low health (tighten) |
| Human approval threshold (lines) | `PrReviewPreferenceKeys.HUMAN_APPROVAL_THRESHOLD` | `devtown.pr-review` / `human-approval-threshold` | 500 | [100, 2000] | DECREASE when low health (review more) |
| Min approved reviews | *(see note)* | `casehubio.devtown.risk` / *(needs dedup)* | 1 | [1, 3] | INCREASE when low health |
| Security severity threshold | *(direct worker logic, see note)* | `casehubio.devtown.risk` / *(needs dedup)* | HIGH | enum | N/A (discrete) |

**Note — RiskPreferenceKeys key collision:** `RiskPreferenceKeys` has a pre-existing bug: `MERGE_MIN_APPROVED_REVIEWS`, `SECURITY_SEVERITY_THRESHOLD`, `ISSUE_CLOSE_COMMENT_THRESHOLD`, `DEPENDENCY_USAGE_THRESHOLD`, and `DEPLOY_MODULE_THRESHOLD` all share the key name `"threshold"` in namespace `"casehubio.devtown.risk"`. Similarly, `EXPIRES_IN_MINUTES_REVERSIBLE` and `EXPIRES_IN_MINUTES_IRREVERSIBLE` share `"expiresInMinutes"`. These collisions must be fixed before workers can read/write these keys distinctly. File an issue against devtown to fix `RiskPreferenceKeys` with unique key names per constant.

**Trust gate key:** The trust-gate YAML uses key `casehubio.devtown.trust-gate.min-obligor-trust` in scope `casehubio/devtown/trust-gate`. A new `TrustGatePreferenceKeys` constant is needed (does not exist today).

**Preference scope paths:** `casehubio/devtown/trust-gate`, `casehubio/devtown/pr-review`, `casehubio/devtown/risk`

### 5. QueuePriorityWorker

**Category:** `queue-optimization`
**Trigger:** `queue-degradation` (MergeQueueHealthCapabilityArea health < 0.6)
**Capability area:** `MergeQueueHealthCapabilityArea`

**Parameters tuned:**

| Parameter | PreferenceKey constant | Namespace / Name | Current default | Bounds | Direction |
|-----------|----------------------|------------------|-----------------|--------|-----------|
| Max batch size | `MergeQueuePreferenceKeys.MAX_BATCH_SIZE` | `devtown.merge-queue` / `max-batch-size` | 10 | [2, 20] | Context-dependent |
| Min batch size | `MergeQueuePreferenceKeys.MIN_BATCH_SIZE` | `devtown.merge-queue` / `min-batch-size` | 1 | [1, 5] | Context-dependent |
| Decay rate per hour | `MergeQueuePreferenceKeys.DECAY_RATE_PER_HOUR` | `devtown.merge-queue.priority` / `decay-rate-per-hour` | 125 | [50, 500] | Context-dependent |
| Failure rate window | `MergeQueuePreferenceKeys.FAILURE_RATE_WINDOW` | `devtown.merge-queue` / `failure-rate-window` | 20 | [5, 50] | INCREASE when low health |
| High risk threshold | *(new key needed)* | `devtown.merge-queue` / `high-risk-threshold` | 0.5 | [0.3, 0.8] | DECREASE when low health (more conservative) |
| Failure rate alert threshold | `MergeQueuePreferenceKeys.FAILURE_RATE_ALERT_THRESHOLD` | `devtown.merge-queue` / `failure-rate-alert-threshold` | 0.5 | [0.2, 0.8] | DECREASE when low health |

**Migration required:** `DefaultBatchCompositionPolicy.HIGH_RISK_THRESHOLD` (hardcoded `0.5` in `queue/`) must be migrated to a new `MergeQueuePreferenceKeys.HIGH_RISK_THRESHOLD` constant and read from preferences, with the hardcoded value as the default.

**Batch size ordering constraint:** After computing adjustments for both `MIN_BATCH_SIZE` and `MAX_BATCH_SIZE` independently, the worker enforces `MIN_BATCH_SIZE ≤ MAX_BATCH_SIZE`. Their bounds overlap (MIN ceiling 5 > MAX floor 2), so independent adjustments can produce MIN > MAX — an impossible constraint that would prevent `MergeQueueService.formBatchesTransactional()` from forming any batches. If violated, MIN is clamped down to MAX. Clamping is logged in the adjustment rationale.

**Preference scope path:** `casehubio/devtown/merge-queue`

## Preference Migration Pattern

For hardcoded values that need migration to PreferenceProvider:

1. Add a `PreferenceKey` constant with namespace, name, and default value matching the current hardcoded value
2. Update the consuming class to read from `PreferenceProvider` with the hardcoded value as the fallback default
3. The worker writes new values via `PreferenceWriter` to the same preference key

Example for `DevtownCapabilityRegistry.POLICIES` threshold:

```java
// Before: hardcoded in POLICIES map, read by DevtownTrustRoutingPolicyProvider
double threshold = routingPolicy.threshold()
    .orElse(TrustRoutingPolicy.DEFAULT.threshold());

// After: read from Preferences (TrustRoutingPolicyKeys already defines the key)
Preferences prefs = preferenceProvider.resolve(
    SettingsScope.of("casehubio",
        Path.parse("casehubio/devtown/trust-routing/" + capabilityName)));
double threshold = prefs.getOrDefault(KEYS.threshold()).value();
```

This is backward-compatible — existing behavior is preserved until a worker writes a new value. The preference defaults in `TrustRoutingPolicyKeys` need updating: currently `DoublePreference.of(0.0)` / `IntPreference.of(0)` — should be set to the `DevtownCapabilityRegistry.POLICIES` values or left as 0 with the YAML seed providing the actual defaults.

## Worker Output Format

Every worker returns a consistent output structure for audit:

```json
{
  "category": "reviewer-calibration",
  "target": "reviewer-trust",
  "adjustments": [
    {
      "parameter": "casehubio.devtown.trust-routing.threshold",
      "oldValue": 0.70,
      "newValue": 0.75,
      "method": "proportional",
      "precedentCount": 0,
      "rationale": "Health 0.45, gap 0.15, step +0.05 (clamped to maxStep)"
    }
  ],
  "healthBefore": 0.45,
  "assessmentSummary": "3 of 8 recent reviews had false negatives in security-review",
  "status": "MERGED",
  "reviewOutcome": "approved",
  "requiresPr": false
}
```

The `status`, `reviewOutcome`, and `requiresPr` fields enable the self-improvement template's downstream bindings to resolve correctly.

## Cross-Repo Dependencies

| Repo | Change | Blocking? |
|------|--------|-----------|
| `casehub-platform-api` | Add `PreferenceWriter` SPI interface | Yes — workers cannot write without it |
| `casehub-platform-persistence-jpa-common` | Add `JpaPreferenceWriter` implementation | Yes |
| `casehub-platform-persistence-jpa-common` | Add `LayeredPreferenceProvider` (JPA + YAML composition) | Yes — without it, writes are invisible to readers |
| `casehub-engine` | Add `when` condition to `submit-pr` binding in `self-improvement.yaml` | Yes — without it, preference-only improvements trigger a PR submission worker |
| `casehub-engine` | Update `ImprovementOutcomeEventCapture` to populate `metadata` from case context | No — CBR degrades gracefully (proportional fallback) |
| `casehub-engine` | Persistent backing for `ImprovementOutcomeStore` | No — follow-up for CBR learning quality |
| `casehub-engine` | Ensure `ImprovementOutcomeStore` is injectable in harness apps | No — may already work via CDI |

## Testing Strategy

### Unit Tests

Each worker tested in isolation with mocked `PreferenceProvider`, `PreferenceWriter`, and `CapabilityArea`:
- Verify correct preference keys read (using actual `PreferenceKey` constants)
- Verify `PreferenceWriter.write()` called with correct scope, key, and value
- Verify adjustment bounds enforced (floor, ceiling, minStep, maxStep)
- Verify output structure includes old/new values for audit
- Verify worker returns `WorkerResult.declined()` when health is already above target
- Verify Duration ↔ minutes conversion for SLA parameters
- Verify discrete parameter logic (enum transitions based on health thresholds)

### Strategy Tests

`ProportionalAdjustmentStrategy` (in `domain/` test sources):
- Health gap proportional scaling
- Bounds clamping (floor, ceiling, minStep, maxStep)
- Direction handling (increase vs decrease on low health)

`CbrInformedAdjustmentStrategy` (in `domain/` test sources):
- Falls back to proportional when precedents < minimum
- Selects best-outcome adjustment from precedents
- Constrains CBR recommendation within proportional bounds
- Handles empty and partial precedent sets

### Integration Tests

`@QuarkusTest` with real `PreferenceProvider` and `PreferenceWriter`:
- Verify preference writes persist and are readable via `PreferenceProvider.resolve()`
- Verify workers compose with `DevtownProposalSource` — when health drops, proposals fire, and if the worker runs, the preference actually changes
- Verify `DevtownConflictStrategy` prevents concurrent improvements in the same category
- Verify `DevtownRegressionEvaluator` detects health regression after an adjustment

## References

- `DevtownEvolutionCaseHub.java` — YamlCaseHub for evolution cases, worker registration via `augment()`
- `DevtownProposalSource.java` — maps capability areas to improvement categories and triggers
- `DevtownCategoryProvider.java` — declares 5 categories and 5 stages (analyze, propose, review, apply, observe)
- `self-improvement.yaml` — engine case template for improvement lifecycle
- `WorkerFunction.java` / `WorkerResult.java` — worker API SPI
- `Worker.java` / `Worker.Builder` — worker registration API (capabilities, function)
- `YamlCaseHub.augment(CaseDefinition)` — established pattern for registering CDI-injected workers
- `MergeBatchCaseHub.java` — reference implementation of `augment()` pattern
- `PrReviewCaseHub.java` — reference implementation of `augment()` pattern with CDI injection
- `ImprovementOutcomeStore.java` — engine CBR storage, `findByCategory()` for precedent retrieval
- `ImprovementCbrProjector.java` — engine CBR projection from improvement outcomes
- `ImprovementOutcomeEventCapture` (engine) — outcome capture and feedback loop
- `TrustRoutingPolicyKeys.java` — preference keys for trust routing parameters (threshold, blend-factor, floors, etc.)
- `MergeQueuePreferenceKeys.java` — existing preference-based queue config
- `SlaPreferenceKeys.java` — existing preference-based SLA config
- `PrReviewPreferenceKeys.java` — existing preference-based review gate config
- `RiskPreferenceKeys.java` — existing preference-based risk config (has key collision bug)
- `DevtownCapabilityRegistry.java` — hardcoded routing policies (migration target)
- `DevtownTrustRoutingPolicyProvider.java` — current consumer of trust-routing preferences
- `DefaultBatchCompositionPolicy.java` — hardcoded HIGH_RISK_THRESHOLD (migration target)
- `PreferenceProvider.java` — read-only preference SPI (platform-api)
- `PreferenceWriter.java` — write SPI (to be added to platform-api)
- `LayeredPreferenceProvider.java` — JPA + YAML composed provider (to be added to platform-persistence-jpa-common)
- `SpringPreferenceProvider.java` — existing JPA-backed provider (Spring) — reference for Quarkus JPA read implementation
- `ConfigFilePreferenceProvider.java` — existing YAML-backed provider (devtown's current provider — to be replaced by LayeredPreferenceProvider)
- `PreferenceEntry.java` — JPA entity for preference persistence
- `WorkerFunction.Sync` — record preserving WorkerScope in dispatch (vs convenience overload that discards it)
- `SettingsScope.java` — scope record (tenancyId, path, effectiveAt)
- `SlaEstimator.java` — existing CBR-from-precedents pattern in devtown (different data model)
- `DevtownConflictStrategy.java` — prevents concurrent improvements in same category
- `DevtownRegressionEvaluator.java` — detects health regression (10% delta threshold)
- `CaseHub.signal(UUID, String, Object)` — case-level signal emission via CaseHubRuntime
- `harness-trust-policy-source-split.md` (protocol PP-20260531-300481) — trust policy field ownership (needs update)
- `worker-function-execution-model.md` (protocol PP-20260531-worker-func-exec) — worker function authoring requirements
- `failure-cascade-pattern.md` (protocol PP-20260618-fc6a53) — failure recovery pattern
- `transactional-loop-control-flow-exceptions.md` (protocol PP-20260522-f08b62) — control-flow exception boundary rule
