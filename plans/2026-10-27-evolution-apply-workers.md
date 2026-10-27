# Evolution Apply-Stage Workers Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #266 — feat: evolution apply-stage workers
**Issue group:** #266

**Goal:** Deliver 5 apply-stage workers that execute approved evolution proposals by writing runtime preferences, with proportional adjustment that auto-promotes to CBR-informed once precedents accumulate.

**Architecture:** Workers register via `YamlCaseHub.augment()` on `DevtownEvolutionCaseHub`. A single `improvement-implement` worker dispatches by category to 5 dedicated worker beans. Each worker reads current preferences via `PreferenceProvider`, computes adjustments via `AdjustmentStrategy` (domain-level pure computation), and writes new values. The adjustment strategy starts proportional and auto-promotes to CBR-informed when sufficient precedents exist.

**Tech Stack:** Java 21, Quarkus 3.39.3, CDI, casehub-platform PreferenceProvider, casehub-engine WorkerFunction SPI

## Global Constraints

- Java 21 source, compiled on Java 26 JVM
- All new domain types in `domain/src/main/java/io/casehub/devtown/domain/evolution/` (pure Java, zero framework)
- All new workers in `app/src/main/java/io/casehub/devtown/app/evolution/worker/` (CDI beans)
- Every commit references `Refs #266`
- `PreferenceWriter` SPI does not exist in the platform yet — workers define a local `PreferenceWriter` functional interface in `domain/`. Platform issue to be filed separately. When platform ships `PreferenceWriter`, devtown replaces the local interface.
- `self-improvement.yaml` submit-pr binding needs a `when` guard for `requiresPr != false` — engine issue to be filed separately. Until landed, the submit-pr binding fires but the worker output's `status: MERGED` causes downstream bindings to resolve correctly.
- Use `ide-tooling` for all code navigation and structural editing. Use `ide_insert_member` for new methods, `ide_replace_member` for modifications.

---

## Batch 1: Domain Foundation — Adjustment Strategy

### Task 1: AdjustmentBounds, AdjustmentResult, and ProportionalAdjustmentStrategy

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/AdjustmentBounds.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/AdjustmentResult.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/AdjustmentStrategy.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/ProportionalAdjustmentStrategy.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/evolution/ProportionalAdjustmentStrategyTest.java`

**Interfaces:**
- Consumes: nothing (pure domain, no external dependencies)
- Produces:
  - `AdjustmentBounds(double floor, double ceiling, double minStep, double maxStep, double stepFactor, Direction direction)` — record with `enum Direction { INCREASE_ON_LOW_HEALTH, DECREASE_ON_LOW_HEALTH }`
  - `AdjustmentResult(String parameter, double oldValue, double newValue, String method, int precedentCount, String rationale)` — record with `Map<String, Object> toMap()`
  - `AdjustmentStrategy.compute(String parameter, double currentValue, double healthScore, double targetHealth, AdjustmentBounds bounds, String category, UUID caseId) → AdjustmentResult`
  - `ProportionalAdjustmentStrategy implements AdjustmentStrategy` — proportional to health gap, bounded

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.devtown.domain.evolution;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class ProportionalAdjustmentStrategyTest {

  private final ProportionalAdjustmentStrategy strategy = new ProportionalAdjustmentStrategy();
  private final UUID caseId = UUID.randomUUID();

  @Test
  void adjustsProportionallyToHealthGap() {
    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.10, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.70, 0.45, 0.6, bounds,
        "reviewer-calibration", caseId);

    assertEquals("threshold", result.parameter());
    assertEquals(0.70, result.oldValue());
    assertTrue(result.newValue() > 0.70, "should increase when health is low");
    assertTrue(result.newValue() <= 0.80, "step capped by maxStep 0.10");
    assertEquals("proportional", result.method());
    assertEquals(0, result.precedentCount());
  }

  @Test
  void clampsToFloor() {
    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.50, 1.0,
        AdjustmentBounds.Direction.DECREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.35, 0.45, 0.6, bounds,
        "test", caseId);

    assertTrue(result.newValue() >= 0.30, "should not go below floor");
  }

  @Test
  void clampsToCeiling() {
    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.50, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.90, 0.10, 0.6, bounds,
        "test", caseId);

    assertTrue(result.newValue() <= 0.95, "should not exceed ceiling");
  }

  @Test
  void respectsMinStep() {
    var bounds = new AdjustmentBounds(0.30, 0.95, 0.05, 0.10, 0.1,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    // Small health gap * small stepFactor = tiny delta, but minStep enforces 0.05
    var result = strategy.compute("threshold", 0.70, 0.58, 0.6, bounds,
        "test", caseId);

    assertTrue(Math.abs(result.newValue() - result.oldValue()) >= 0.05,
        "adjustment should be at least minStep");
  }

  @Test
  void noAdjustmentWhenHealthAboveTarget() {
    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.10, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.70, 0.75, 0.6, bounds,
        "test", caseId);

    assertEquals(0.70, result.newValue(), "no change when health is above target");
  }

  @Test
  void decreasesWhenDirectionIsDecrease() {
    var bounds = new AdjustmentBounds(100, 2000, 10, 200, 1.0,
        AdjustmentBounds.Direction.DECREASE_ON_LOW_HEALTH);
    var result = strategy.compute("human-approval-threshold", 500, 0.45, 0.6,
        bounds, "gate-tightening", caseId);

    assertTrue(result.newValue() < 500, "should decrease when direction is DECREASE");
  }

  @Test
  void toMapProducesExpectedKeys() {
    var result = new AdjustmentResult("param", 0.5, 0.6, "proportional", 0, "test rationale");
    var map = result.toMap();

    assertEquals("param", map.get("parameter"));
    assertEquals(0.5, map.get("oldValue"));
    assertEquals(0.6, map.get("newValue"));
    assertEquals("proportional", map.get("method"));
    assertEquals(0, map.get("precedentCount"));
    assertEquals("test rationale", map.get("rationale"));
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ProportionalAdjustmentStrategyTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement AdjustmentBounds record**

```java
package io.casehub.devtown.domain.evolution;

public record AdjustmentBounds(
    double floor,
    double ceiling,
    double minStep,
    double maxStep,
    double stepFactor,
    Direction direction) {

  public enum Direction {
    INCREASE_ON_LOW_HEALTH,
    DECREASE_ON_LOW_HEALTH
  }
}
```

- [ ] **Step 4: Implement AdjustmentResult record**

```java
package io.casehub.devtown.domain.evolution;

import java.util.Map;

public record AdjustmentResult(
    String parameter,
    double oldValue,
    double newValue,
    String method,
    int precedentCount,
    String rationale) {

  public Map<String, Object> toMap() {
    return Map.of(
        "parameter", parameter,
        "oldValue", oldValue,
        "newValue", newValue,
        "method", method,
        "precedentCount", precedentCount,
        "rationale", rationale);
  }
}
```

- [ ] **Step 5: Implement AdjustmentStrategy interface**

```java
package io.casehub.devtown.domain.evolution;

import java.util.UUID;

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

- [ ] **Step 6: Implement ProportionalAdjustmentStrategy**

```java
package io.casehub.devtown.domain.evolution;

import java.util.UUID;

public class ProportionalAdjustmentStrategy implements AdjustmentStrategy {

  @Override
  public AdjustmentResult compute(
      String parameter, double currentValue, double healthScore,
      double targetHealth, AdjustmentBounds bounds, String category,
      UUID caseId) {

    double gap = targetHealth - healthScore;
    if (gap <= 0) {
      return new AdjustmentResult(parameter, currentValue, currentValue,
          "proportional", 0,
          String.format("Health %.2f >= target %.2f, no adjustment needed",
              healthScore, targetHealth));
    }

    double rawDelta = gap * bounds.stepFactor();

    double absDelta = Math.abs(rawDelta);
    absDelta = Math.max(absDelta, bounds.minStep());
    absDelta = Math.min(absDelta, bounds.maxStep());

    double newValue;
    if (bounds.direction() == AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH) {
      newValue = currentValue + absDelta;
    } else {
      newValue = currentValue - absDelta;
    }

    newValue = Math.max(newValue, bounds.floor());
    newValue = Math.min(newValue, bounds.ceiling());

    return new AdjustmentResult(parameter, currentValue, newValue,
        "proportional", 0,
        String.format("Health %.2f, gap %.2f, step %+.4f (bounded [%.2f, %.2f])",
            healthScore, gap, newValue - currentValue, bounds.floor(), bounds.ceiling()));
  }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ProportionalAdjustmentStrategyTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add domain/src/main/java/io/casehub/devtown/domain/evolution/ domain/src/test/java/io/casehub/devtown/domain/evolution/
git commit -m "feat(#266): adjustment strategy foundation — bounds, result, proportional strategy

Refs #266

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: OutcomeQuery and CbrInformedAdjustmentStrategy

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/OutcomeQuery.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/CbrInformedAdjustmentStrategy.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/EvolutionOutcomeRecord.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/evolution/CbrInformedAdjustmentStrategyTest.java`

**Interfaces:**
- Consumes: `AdjustmentStrategy`, `AdjustmentBounds`, `AdjustmentResult`, `ProportionalAdjustmentStrategy` (Task 1)
- Produces:
  - `OutcomeQuery` — `@FunctionalInterface`, `List<EvolutionOutcomeRecord> findByCategory(String category, int limit)`
  - `EvolutionOutcomeRecord(String category, double healthBefore, double healthAfter, Map<String, Double> adjustments)` — domain record for CBR data
  - `CbrInformedAdjustmentStrategy implements AdjustmentStrategy` — delegates to proportional when insufficient precedents

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.devtown.domain.evolution;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class CbrInformedAdjustmentStrategyTest {

  private final UUID caseId = UUID.randomUUID();

  @Test
  void fallsBackToProportionalWhenNoPrecedents() {
    OutcomeQuery emptyQuery = (category, limit) -> List.of();
    var strategy = new CbrInformedAdjustmentStrategy(
        new ProportionalAdjustmentStrategy(), emptyQuery, 5, 0.1);

    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.10, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.70, 0.45, 0.6, bounds,
        "reviewer-calibration", caseId);

    assertEquals("proportional", result.method());
    assertEquals(0, result.precedentCount());
  }

  @Test
  void fallsBackWhenBelowMinPrecedents() {
    OutcomeQuery fewResults = (category, limit) -> List.of(
        new EvolutionOutcomeRecord("reviewer-calibration", 0.45, 0.72,
            Map.of("threshold", 0.75)),
        new EvolutionOutcomeRecord("reviewer-calibration", 0.42, 0.68,
            Map.of("threshold", 0.78)));
    var strategy = new CbrInformedAdjustmentStrategy(
        new ProportionalAdjustmentStrategy(), fewResults, 5, 0.1);

    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.10, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.70, 0.45, 0.6, bounds,
        "reviewer-calibration", caseId);

    assertEquals("proportional", result.method());
  }

  @Test
  void usesCbrWhenSufficientPrecedents() {
    OutcomeQuery query = (category, limit) -> List.of(
        new EvolutionOutcomeRecord("reviewer-calibration", 0.44, 0.71,
            Map.of("threshold", 0.76)),
        new EvolutionOutcomeRecord("reviewer-calibration", 0.46, 0.73,
            Map.of("threshold", 0.75)),
        new EvolutionOutcomeRecord("reviewer-calibration", 0.43, 0.65,
            Map.of("threshold", 0.80)),
        new EvolutionOutcomeRecord("reviewer-calibration", 0.47, 0.70,
            Map.of("threshold", 0.74)),
        new EvolutionOutcomeRecord("reviewer-calibration", 0.42, 0.69,
            Map.of("threshold", 0.77)));
    var strategy = new CbrInformedAdjustmentStrategy(
        new ProportionalAdjustmentStrategy(), query, 5, 0.1);

    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.10, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.70, 0.45, 0.6, bounds,
        "reviewer-calibration", caseId);

    assertEquals("cbr-informed", result.method());
    assertTrue(result.precedentCount() >= 5);
  }

  @Test
  void cbrConstrainedByBounds() {
    OutcomeQuery query = (category, limit) -> List.of(
        new EvolutionOutcomeRecord("test", 0.40, 0.80,
            Map.of("threshold", 0.99)),
        new EvolutionOutcomeRecord("test", 0.41, 0.78,
            Map.of("threshold", 0.98)),
        new EvolutionOutcomeRecord("test", 0.42, 0.76,
            Map.of("threshold", 0.97)),
        new EvolutionOutcomeRecord("test", 0.43, 0.75,
            Map.of("threshold", 0.96)),
        new EvolutionOutcomeRecord("test", 0.44, 0.74,
            Map.of("threshold", 0.95)));
    var strategy = new CbrInformedAdjustmentStrategy(
        new ProportionalAdjustmentStrategy(), query, 5, 0.1);

    var bounds = new AdjustmentBounds(0.30, 0.85, 0.01, 0.10, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.70, 0.45, 0.6, bounds,
        "test", caseId);

    assertTrue(result.newValue() <= 0.85, "CBR recommendation clamped to ceiling");
  }

  @Test
  void filtersOutPrecedentsWithoutHealthAfter() {
    OutcomeQuery query = (category, limit) -> List.of(
        new EvolutionOutcomeRecord("test", 0.45, Double.NaN,
            Map.of("threshold", 0.76)),
        new EvolutionOutcomeRecord("test", 0.44, Double.NaN,
            Map.of("threshold", 0.77)),
        new EvolutionOutcomeRecord("test", 0.46, Double.NaN,
            Map.of("threshold", 0.75)),
        new EvolutionOutcomeRecord("test", 0.43, Double.NaN,
            Map.of("threshold", 0.78)),
        new EvolutionOutcomeRecord("test", 0.42, Double.NaN,
            Map.of("threshold", 0.79)));
    var strategy = new CbrInformedAdjustmentStrategy(
        new ProportionalAdjustmentStrategy(), query, 5, 0.1);

    var bounds = new AdjustmentBounds(0.30, 0.95, 0.01, 0.10, 1.0,
        AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
    var result = strategy.compute("threshold", 0.70, 0.45, 0.6, bounds,
        "test", caseId);

    assertEquals("proportional", result.method(),
        "NaN healthAfter means cannot rank by effectiveness, fall back");
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=CbrInformedAdjustmentStrategyTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement OutcomeQuery and EvolutionOutcomeRecord**

```java
package io.casehub.devtown.domain.evolution;

import java.util.List;

@FunctionalInterface
public interface OutcomeQuery {
  List<EvolutionOutcomeRecord> findByCategory(String category, int limit);
}
```

```java
package io.casehub.devtown.domain.evolution;

import java.util.Map;

public record EvolutionOutcomeRecord(
    String category,
    double healthBefore,
    double healthAfter,
    Map<String, Double> adjustments) {}
```

- [ ] **Step 4: Implement CbrInformedAdjustmentStrategy**

```java
package io.casehub.devtown.domain.evolution;

import java.util.Comparator;
import java.util.List;
import java.util.UUID;

public class CbrInformedAdjustmentStrategy implements AdjustmentStrategy {

  private final ProportionalAdjustmentStrategy fallback;
  private final OutcomeQuery outcomeQuery;
  private final int minPrecedents;
  private final double healthGapTolerance;

  public CbrInformedAdjustmentStrategy(
      ProportionalAdjustmentStrategy fallback,
      OutcomeQuery outcomeQuery,
      int minPrecedents,
      double healthGapTolerance) {
    this.fallback = fallback;
    this.outcomeQuery = outcomeQuery;
    this.minPrecedents = minPrecedents;
    this.healthGapTolerance = healthGapTolerance;
  }

  @Override
  public AdjustmentResult compute(
      String parameter, double currentValue, double healthScore,
      double targetHealth, AdjustmentBounds bounds, String category,
      UUID caseId) {

    double gap = targetHealth - healthScore;
    if (gap <= 0) {
      return fallback.compute(parameter, currentValue, healthScore,
          targetHealth, bounds, category, caseId);
    }

    List<EvolutionOutcomeRecord> precedents = outcomeQuery.findByCategory(category, 50);

    List<EvolutionOutcomeRecord> similar = precedents.stream()
        .filter(p -> Math.abs((targetHealth - p.healthBefore()) - gap) <= healthGapTolerance)
        .filter(p -> !Double.isNaN(p.healthAfter()))
        .filter(p -> p.adjustments().containsKey(parameter))
        .toList();

    if (similar.size() < minPrecedents) {
      return fallback.compute(parameter, currentValue, healthScore,
          targetHealth, bounds, category, caseId);
    }

    EvolutionOutcomeRecord best = similar.stream()
        .max(Comparator.comparingDouble(p -> p.healthAfter() - p.healthBefore()))
        .orElseThrow();

    double cbrValue = best.adjustments().get(parameter);

    cbrValue = Math.max(cbrValue, bounds.floor());
    cbrValue = Math.min(cbrValue, bounds.ceiling());

    double delta = Math.abs(cbrValue - currentValue);
    if (delta > bounds.maxStep()) {
      if (bounds.direction() == AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH) {
        cbrValue = currentValue + bounds.maxStep();
      } else {
        cbrValue = currentValue - bounds.maxStep();
      }
      cbrValue = Math.max(cbrValue, bounds.floor());
      cbrValue = Math.min(cbrValue, bounds.ceiling());
    }

    return new AdjustmentResult(parameter, currentValue, cbrValue,
        "cbr-informed", similar.size(),
        String.format("CBR from %d precedents (best: health %.2f→%.2f), bounded [%.2f, %.2f]",
            similar.size(), best.healthBefore(), best.healthAfter(),
            bounds.floor(), bounds.ceiling()));
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=CbrInformedAdjustmentStrategyTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add domain/src/main/java/io/casehub/devtown/domain/evolution/ domain/src/test/java/io/casehub/devtown/domain/evolution/
git commit -m "feat(#266): CBR-informed adjustment strategy with outcome query

Refs #266

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Worker Registration and First Worker

### Task 3: PreferenceWriter interface, DevtownEvolutionCaseHub augment, and ReviewerCalibrationWorker

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/evolution/PreferenceWriter.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionCaseHub.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/ReviewerCalibrationWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/EvolutionWorkerConstants.java`
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/worker/ReviewerCalibrationWorkerTest.java`

**Interfaces:**
- Consumes: `AdjustmentStrategy`, `AdjustmentBounds`, `AdjustmentResult` (Task 1), `PreferenceProvider` (platform), `ReviewerTrustCapabilityArea` (existing), `WorkerFunction.Sync` / `WorkerResult` (engine worker API), `Worker.builder()` (engine worker API)
- Produces:
  - `PreferenceWriter` — `@FunctionalInterface`, `void write(String scope, String namespace, String key, String value)`
  - `EvolutionWorkerConstants` — shared constants: `TARGET_HEALTH = 0.6`, metadata key names
  - `ReviewerCalibrationWorker.apply(Map<String, Object> input, WorkerScope scope) → WorkerResult<Map<String, Object>>`
  - `DevtownEvolutionCaseHub.augment()` — registers single `improvement-implement` worker with category dispatch

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.ReviewerTrustCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class ReviewerCalibrationWorkerTest {

  private ReviewerCalibrationWorker worker;
  private PreferenceWriter preferenceWriter;
  private ReviewerTrustCapabilityArea capabilityArea;
  private WorkerScope scope;

  @BeforeEach
  void setUp() {
    preferenceWriter = mock(PreferenceWriter.class);
    capabilityArea = mock(ReviewerTrustCapabilityArea.class);
    var strategy = new ProportionalAdjustmentStrategy();

    worker = new ReviewerCalibrationWorker(preferenceWriter, capabilityArea, strategy);

    scope = mock(WorkerScope.class);
    when(scope.caseId()).thenReturn(UUID.randomUUID());
    when(scope.accumulatedState()).thenReturn(Map.of("tenancyId", "casehubio"));
  }

  @Test
  void adjustsThresholdWhenHealthIsLow() {
    when(capabilityArea.health()).thenReturn(0.45);

    Map<String, Object> input = Map.of(
        "category", "reviewer-calibration",
        "target", "security-review");

    WorkerResult<Map<String, Object>> result = worker.apply(input, scope);

    assertInstanceOf(WorkerOutcome.Success.class, result.outcome());
    @SuppressWarnings("unchecked")
    Map<String, Object> output = result.output();
    assertEquals("reviewer-calibration", output.get("category"));
    assertEquals("MERGED", output.get("status"));
    assertEquals(false, output.get("requiresPr"));

    @SuppressWarnings("unchecked")
    List<Map<String, Object>> adjustments =
        (List<Map<String, Object>>) output.get("adjustments");
    assertFalse(adjustments.isEmpty(), "should have at least one adjustment");

    verify(preferenceWriter, atLeastOnce()).write(anyString(), anyString(), anyString(), anyString());
  }

  @Test
  void declinesWhenHealthAboveTarget() {
    when(capabilityArea.health()).thenReturn(0.75);

    Map<String, Object> input = Map.of(
        "category", "reviewer-calibration",
        "target", "security-review");

    WorkerResult<Map<String, Object>> result = worker.apply(input, scope);

    assertInstanceOf(WorkerOutcome.Declined.class, result.outcome());
    verify(preferenceWriter, never()).write(anyString(), anyString(), anyString(), anyString());
  }

  @Test
  void outputIncludesHealthBefore() {
    when(capabilityArea.health()).thenReturn(0.45);

    Map<String, Object> input = Map.of(
        "category", "reviewer-calibration",
        "target", "security-review");

    WorkerResult<Map<String, Object>> result = worker.apply(input, scope);

    @SuppressWarnings("unchecked")
    Map<String, Object> output = result.output();
    assertEquals(0.45, output.get("healthBefore"));
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ReviewerCalibrationWorkerTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement PreferenceWriter interface**

```java
package io.casehub.devtown.domain.evolution;

@FunctionalInterface
public interface PreferenceWriter {
  void write(String scope, String namespace, String key, String value);
}
```

- [ ] **Step 4: Implement EvolutionWorkerConstants**

```java
package io.casehub.devtown.app.evolution.worker;

public final class EvolutionWorkerConstants {
  public static final double TARGET_HEALTH = 0.6;
  public static final String META_HEALTH_BEFORE = "evolution.healthBefore";
  public static final String META_HEALTH_AFTER = "evolution.healthAfter";
  public static final String META_ADJUSTMENTS = "evolution.adjustments";
  public static final String META_CATEGORY = "evolution.category";

  private EvolutionWorkerConstants() {}
}
```

- [ ] **Step 5: Implement ReviewerCalibrationWorker**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.ReviewerTrustCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static io.casehub.devtown.app.evolution.worker.EvolutionWorkerConstants.TARGET_HEALTH;

@ApplicationScoped
public class ReviewerCalibrationWorker {

  static final AdjustmentBounds THRESHOLD_BOUNDS = new AdjustmentBounds(
      0.30, 0.95, 0.01, 0.10, 1.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds MIN_OBS_BOUNDS = new AdjustmentBounds(
      3, 30, 1, 5, 10.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds BORDERLINE_BOUNDS = new AdjustmentBounds(
      0.02, 0.15, 0.005, 0.03, 0.5,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);

  private final PreferenceWriter preferenceWriter;
  private final ReviewerTrustCapabilityArea capabilityArea;
  private final AdjustmentStrategy strategy;

  @Inject
  public ReviewerCalibrationWorker(
      PreferenceWriter preferenceWriter,
      ReviewerTrustCapabilityArea capabilityArea,
      AdjustmentStrategy strategy) {
    this.preferenceWriter = preferenceWriter;
    this.capabilityArea = capabilityArea;
    this.strategy = strategy;
  }

  @SuppressWarnings("unchecked")
  public WorkerResult<Map<String, Object>> apply(
      Map<String, Object> input, WorkerScope scope) {

    double health = capabilityArea.health();
    if (health >= TARGET_HEALTH) {
      return WorkerResult.declined(
          String.format("Health %.2f >= target %.2f, no calibration needed",
              health, TARGET_HEALTH));
    }

    String capability = (String) input.get("target");
    String tenancyId = (String) scope.accumulatedState().get("tenancyId");
    String scopePath = "casehubio/devtown/trust-routing/" + capability;
    String namespace = "casehubio.devtown.trust-routing";

    List<Map<String, Object>> adjustments = new ArrayList<>();

    AdjustmentResult thresholdAdj = strategy.compute(
        "threshold", 0.70, health, TARGET_HEALTH, THRESHOLD_BOUNDS,
        "reviewer-calibration", scope.caseId());
    if (thresholdAdj.oldValue() != thresholdAdj.newValue()) {
      preferenceWriter.write(scopePath, namespace, "threshold",
          String.valueOf(thresholdAdj.newValue()));
      adjustments.add(thresholdAdj.toMap());
    }

    AdjustmentResult minObsAdj = strategy.compute(
        "minimum-observations", 10, health, TARGET_HEALTH, MIN_OBS_BOUNDS,
        "reviewer-calibration", scope.caseId());
    if (minObsAdj.oldValue() != minObsAdj.newValue()) {
      preferenceWriter.write(scopePath, namespace, "minimum-observations",
          String.valueOf(Math.round(minObsAdj.newValue())));
      adjustments.add(minObsAdj.toMap());
    }

    AdjustmentResult borderlineAdj = strategy.compute(
        "borderline-margin", 0.05, health, TARGET_HEALTH, BORDERLINE_BOUNDS,
        "reviewer-calibration", scope.caseId());
    if (borderlineAdj.oldValue() != borderlineAdj.newValue()) {
      preferenceWriter.write(scopePath, namespace, "borderline-margin",
          String.valueOf(borderlineAdj.newValue()));
      adjustments.add(borderlineAdj.toMap());
    }

    var output = new LinkedHashMap<String, Object>();
    output.put("category", "reviewer-calibration");
    output.put("target", capability);
    output.put("adjustments", adjustments);
    output.put("healthBefore", health);
    output.put("status", "MERGED");
    output.put("reviewOutcome", "approved");
    output.put("requiresPr", false);

    return WorkerResult.of(output);
  }
}
```

- [ ] **Step 6: Augment DevtownEvolutionCaseHub with worker dispatch**

Use `ide_replace_member` to replace the class body of `DevtownEvolutionCaseHub`:

```java
package io.casehub.devtown.app.evolution;

import io.casehub.api.engine.YamlCaseHub;
import io.casehub.api.model.CaseDefinition;
import io.casehub.devtown.app.evolution.worker.*;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Map;

@ApplicationScoped
public class DevtownEvolutionCaseHub extends YamlCaseHub {

  @Inject ReviewerCalibrationWorker reviewerCalibration;

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

  @SuppressWarnings("unchecked")
  WorkerResult<Map<String, Object>> dispatch(Map<String, Object> input, WorkerScope scope) {
    String category = (String) input.get("category");
    return switch (category) {
      case "reviewer-calibration" -> reviewerCalibration.apply(input, scope);
      default -> WorkerResult.failed("Unknown evolution category: " + category);
    };
  }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ReviewerCalibrationWorkerTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Verify full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add domain/src/main/java/io/casehub/devtown/domain/evolution/PreferenceWriter.java app/src/main/java/io/casehub/devtown/app/evolution/ app/src/test/java/io/casehub/devtown/app/evolution/worker/
git commit -m "feat(#266): ReviewerCalibrationWorker + augment dispatch on DevtownEvolutionCaseHub

Refs #266

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Remaining Workers

### Task 4: RoutingAdjustmentWorker and SlaThresholdWorker

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/RoutingAdjustmentWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/SlaThresholdWorker.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionCaseHub.java` (add dispatch cases)
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/worker/RoutingAdjustmentWorkerTest.java`
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/worker/SlaThresholdWorkerTest.java`

**Interfaces:**
- Consumes: `AdjustmentStrategy`, `AdjustmentBounds`, `PreferenceWriter` (Tasks 1-3), `ReviewQualityCapabilityArea` (existing), `SlaComplianceCapabilityArea` (existing), `WorkerScope` (engine API)
- Produces:
  - `RoutingAdjustmentWorker.apply(Map, WorkerScope) → WorkerResult` — tunes blend-factor, quality floors, cbr-weight
  - `SlaThresholdWorker.apply(Map, WorkerScope) → WorkerResult` — tunes SLA durations (with Duration ↔ minutes conversion and CRITICAL < HIGH < NORMAL ordering constraint)

- [ ] **Step 1: Write RoutingAdjustmentWorker tests**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.ReviewQualityCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class RoutingAdjustmentWorkerTest {

  private RoutingAdjustmentWorker worker;
  private PreferenceWriter preferenceWriter;
  private ReviewQualityCapabilityArea capabilityArea;
  private WorkerScope scope;

  @BeforeEach
  void setUp() {
    preferenceWriter = mock(PreferenceWriter.class);
    capabilityArea = mock(ReviewQualityCapabilityArea.class);
    worker = new RoutingAdjustmentWorker(preferenceWriter, capabilityArea,
        new ProportionalAdjustmentStrategy());
    scope = mock(WorkerScope.class);
    when(scope.caseId()).thenReturn(UUID.randomUUID());
    when(scope.accumulatedState()).thenReturn(Map.of("tenancyId", "casehubio"));
  }

  @Test
  void adjustsBlendFactorWhenHealthIsLow() {
    when(capabilityArea.health()).thenReturn(0.42);
    var result = worker.apply(
        Map.of("category", "routing-adjustment", "target", "security-review"), scope);
    assertInstanceOf(WorkerOutcome.Success.class, result.outcome());
    assertEquals("MERGED", result.output().get("status"));
    verify(preferenceWriter, atLeastOnce()).write(anyString(), anyString(), anyString(), anyString());
  }

  @Test
  void declinesWhenHealthAboveTarget() {
    when(capabilityArea.health()).thenReturn(0.72);
    var result = worker.apply(
        Map.of("category", "routing-adjustment", "target", "security-review"), scope);
    assertInstanceOf(WorkerOutcome.Declined.class, result.outcome());
  }
}
```

- [ ] **Step 2: Write SlaThresholdWorker tests**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.SlaComplianceCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerScope;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class SlaThresholdWorkerTest {

  private SlaThresholdWorker worker;
  private PreferenceWriter preferenceWriter;
  private SlaComplianceCapabilityArea capabilityArea;
  private WorkerScope scope;

  @BeforeEach
  void setUp() {
    preferenceWriter = mock(PreferenceWriter.class);
    capabilityArea = mock(SlaComplianceCapabilityArea.class);
    worker = new SlaThresholdWorker(preferenceWriter, capabilityArea,
        new ProportionalAdjustmentStrategy());
    scope = mock(WorkerScope.class);
    when(scope.caseId()).thenReturn(UUID.randomUUID());
    when(scope.accumulatedState()).thenReturn(Map.of("tenancyId", "casehubio"));
  }

  @Test
  void adjustsSlaWhenHealthIsLow() {
    when(capabilityArea.health()).thenReturn(0.40);
    var result = worker.apply(
        Map.of("category", "sla-tuning", "target", "sla-compliance"), scope);
    assertInstanceOf(WorkerOutcome.Success.class, result.outcome());
    assertEquals("MERGED", result.output().get("status"));
  }

  @Test
  void enforcesSlaOrdering() {
    when(capabilityArea.health()).thenReturn(0.30);
    var result = worker.apply(
        Map.of("category", "sla-tuning", "target", "sla-compliance"), scope);

    @SuppressWarnings("unchecked")
    List<Map<String, Object>> adjustments =
        (List<Map<String, Object>>) result.output().get("adjustments");

    // Extract SLA values from adjustments and verify CRITICAL < HIGH < NORMAL
    double critical = -1, high = -1, normal = -1;
    for (var adj : adjustments) {
      switch ((String) adj.get("parameter")) {
        case "sla.critical" -> critical = (double) adj.get("newValue");
        case "sla.high" -> high = (double) adj.get("newValue");
        case "sla.normal" -> normal = (double) adj.get("newValue");
      }
    }
    if (critical > 0 && high > 0) {
      assertTrue(critical < high, "CRITICAL must be less than HIGH");
    }
    if (high > 0 && normal > 0) {
      assertTrue(high < normal, "HIGH must be less than NORMAL");
    }
  }

  @Test
  void declinesWhenHealthAboveTarget() {
    when(capabilityArea.health()).thenReturn(0.72);
    var result = worker.apply(
        Map.of("category", "sla-tuning", "target", "sla-compliance"), scope);
    assertInstanceOf(WorkerOutcome.Declined.class, result.outcome());
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="RoutingAdjustmentWorkerTest,SlaThresholdWorkerTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL

- [ ] **Step 4: Implement RoutingAdjustmentWorker**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.ReviewQualityCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static io.casehub.devtown.app.evolution.worker.EvolutionWorkerConstants.TARGET_HEALTH;

@ApplicationScoped
public class RoutingAdjustmentWorker {

  static final AdjustmentBounds BLEND_FACTOR_BOUNDS = new AdjustmentBounds(
      0.20, 1.0, 0.02, 0.15, 1.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds QUALITY_FLOOR_BOUNDS = new AdjustmentBounds(
      0.30, 0.95, 0.02, 0.10, 1.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds CBR_WEIGHT_BOUNDS = new AdjustmentBounds(
      0.0, 0.5, 0.02, 0.10, 0.5,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);

  private final PreferenceWriter preferenceWriter;
  private final ReviewQualityCapabilityArea capabilityArea;
  private final AdjustmentStrategy strategy;

  @Inject
  public RoutingAdjustmentWorker(
      PreferenceWriter preferenceWriter,
      ReviewQualityCapabilityArea capabilityArea,
      AdjustmentStrategy strategy) {
    this.preferenceWriter = preferenceWriter;
    this.capabilityArea = capabilityArea;
    this.strategy = strategy;
  }

  @SuppressWarnings("unchecked")
  public WorkerResult<Map<String, Object>> apply(
      Map<String, Object> input, WorkerScope scope) {

    double health = capabilityArea.health();
    if (health >= TARGET_HEALTH) {
      return WorkerResult.declined(
          String.format("Health %.2f >= target %.2f", health, TARGET_HEALTH));
    }

    String capability = (String) input.get("target");
    String scopePath = "casehubio/devtown/trust-routing/" + capability;
    String namespace = "casehubio.devtown.trust-routing";

    List<Map<String, Object>> adjustments = new ArrayList<>();

    AdjustmentResult blendAdj = strategy.compute(
        "blend-factor", 0.60, health, TARGET_HEALTH, BLEND_FACTOR_BOUNDS,
        "routing-adjustment", scope.caseId());
    if (blendAdj.oldValue() != blendAdj.newValue()) {
      preferenceWriter.write(scopePath, namespace, "blend-factor",
          String.valueOf(blendAdj.newValue()));
      adjustments.add(blendAdj.toMap());
    }

    AdjustmentResult cbrAdj = strategy.compute(
        "cbr-weight", 0.10, health, TARGET_HEALTH, CBR_WEIGHT_BOUNDS,
        "routing-adjustment", scope.caseId());
    if (cbrAdj.oldValue() != cbrAdj.newValue()) {
      preferenceWriter.write(scopePath, namespace, "cbr-weight",
          String.valueOf(cbrAdj.newValue()));
      adjustments.add(cbrAdj.toMap());
    }

    var output = new LinkedHashMap<String, Object>();
    output.put("category", "routing-adjustment");
    output.put("target", capability);
    output.put("adjustments", adjustments);
    output.put("healthBefore", health);
    output.put("status", "MERGED");
    output.put("reviewOutcome", "approved");
    output.put("requiresPr", false);
    return WorkerResult.of(output);
  }
}
```

- [ ] **Step 5: Implement SlaThresholdWorker**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.SlaComplianceCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static io.casehub.devtown.app.evolution.worker.EvolutionWorkerConstants.TARGET_HEALTH;

@ApplicationScoped
public class SlaThresholdWorker {

  static final AdjustmentBounds SLA_CRITICAL_BOUNDS = new AdjustmentBounds(
      30, 240, 10, 60, 200.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds SLA_HIGH_BOUNDS = new AdjustmentBounds(
      60, 720, 15, 120, 400.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds SLA_NORMAL_BOUNDS = new AdjustmentBounds(
      120, 1440, 30, 240, 800.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds ESCALATION_BOUNDS = new AdjustmentBounds(
      2, 24, 1, 4, 10.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);

  private static final double MIN_CRITICAL_HIGH_GAP = 30;
  private static final double MIN_HIGH_NORMAL_GAP = 60;

  private final PreferenceWriter preferenceWriter;
  private final SlaComplianceCapabilityArea capabilityArea;
  private final AdjustmentStrategy strategy;

  @Inject
  public SlaThresholdWorker(
      PreferenceWriter preferenceWriter,
      SlaComplianceCapabilityArea capabilityArea,
      AdjustmentStrategy strategy) {
    this.preferenceWriter = preferenceWriter;
    this.capabilityArea = capabilityArea;
    this.strategy = strategy;
  }

  @SuppressWarnings("unchecked")
  public WorkerResult<Map<String, Object>> apply(
      Map<String, Object> input, WorkerScope scope) {

    double health = capabilityArea.health();
    if (health >= TARGET_HEALTH) {
      return WorkerResult.declined(
          String.format("Health %.2f >= target %.2f", health, TARGET_HEALTH));
    }

    List<Map<String, Object>> adjustments = new ArrayList<>();

    AdjustmentResult criticalAdj = strategy.compute(
        "sla.critical", 60, health, TARGET_HEALTH, SLA_CRITICAL_BOUNDS,
        "sla-tuning", scope.caseId());
    AdjustmentResult highAdj = strategy.compute(
        "sla.high", 240, health, TARGET_HEALTH, SLA_HIGH_BOUNDS,
        "sla-tuning", scope.caseId());
    AdjustmentResult normalAdj = strategy.compute(
        "sla.normal", 480, health, TARGET_HEALTH, SLA_NORMAL_BOUNDS,
        "sla-tuning", scope.caseId());

    double critical = criticalAdj.newValue();
    double high = highAdj.newValue();
    double normal = normalAdj.newValue();

    if (high < critical + MIN_CRITICAL_HIGH_GAP) {
      high = critical + MIN_CRITICAL_HIGH_GAP;
    }
    if (normal < high + MIN_HIGH_NORMAL_GAP) {
      normal = high + MIN_HIGH_NORMAL_GAP;
    }

    String mqScope = "casehubio/devtown/merge-queue";
    String mqNs = "devtown.merge-queue.sla";

    if (critical != criticalAdj.oldValue()) {
      preferenceWriter.write(mqScope, mqNs, "CRITICAL",
          minutesToDuration(critical));
      adjustments.add(withDuration(criticalAdj, critical));
    }
    if (high != highAdj.oldValue()) {
      preferenceWriter.write(mqScope, mqNs, "HIGH",
          minutesToDuration(high));
      adjustments.add(withDuration(highAdj, high));
    }
    if (normal != normalAdj.oldValue()) {
      preferenceWriter.write(mqScope, mqNs, "NORMAL",
          minutesToDuration(normal));
      adjustments.add(withDuration(normalAdj, normal));
    }

    AdjustmentResult escAdj = strategy.compute(
        "escalation-hours", 8, health, TARGET_HEALTH, ESCALATION_BOUNDS,
        "sla-tuning", scope.caseId());
    if (escAdj.oldValue() != escAdj.newValue()) {
      preferenceWriter.write("casehubio/devtown/sla", "devtown.sla",
          "escalation-hours", String.valueOf(Math.round(escAdj.newValue())));
      adjustments.add(escAdj.toMap());
    }

    var output = new LinkedHashMap<String, Object>();
    output.put("category", "sla-tuning");
    output.put("target", "sla-compliance");
    output.put("adjustments", adjustments);
    output.put("healthBefore", health);
    output.put("status", "MERGED");
    output.put("reviewOutcome", "approved");
    output.put("requiresPr", false);
    return WorkerResult.of(output);
  }

  private static String minutesToDuration(double minutes) {
    long totalMinutes = Math.round(minutes);
    long hours = totalMinutes / 60;
    long mins = totalMinutes % 60;
    if (mins == 0) return "PT" + hours + "H";
    if (hours == 0) return "PT" + mins + "M";
    return "PT" + hours + "H" + mins + "M";
  }

  private static Map<String, Object> withDuration(AdjustmentResult adj, double clampedValue) {
    var map = new LinkedHashMap<>(adj.toMap());
    map.put("newValue", clampedValue);
    map.put("duration", minutesToDuration(clampedValue));
    return map;
  }
}
```

- [ ] **Step 6: Add dispatch cases to DevtownEvolutionCaseHub**

Add `routing-adjustment` and `sla-tuning` cases to the dispatch switch and inject the new workers.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="RoutingAdjustmentWorkerTest,SlaThresholdWorkerTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/ app/src/test/java/io/casehub/devtown/app/evolution/worker/
git commit -m "feat(#266): RoutingAdjustmentWorker + SlaThresholdWorker with SLA ordering

Refs #266

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 5: GateStrictnessWorker and QueuePriorityWorker

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/GateStrictnessWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/QueuePriorityWorker.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionCaseHub.java` (add dispatch cases)
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/worker/GateStrictnessWorkerTest.java`
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/worker/QueuePriorityWorkerTest.java`

**Interfaces:**
- Consumes: `AdjustmentStrategy`, `AdjustmentBounds`, `PreferenceWriter` (Tasks 1-3), `CiReliabilityCapabilityArea` (existing), `MergeQueueHealthCapabilityArea` (existing)
- Produces:
  - `GateStrictnessWorker.apply(Map, WorkerScope) → WorkerResult` — tunes trust gate floor, human approval threshold, min approved reviews
  - `QueuePriorityWorker.apply(Map, WorkerScope) → WorkerResult` — tunes batch sizes (with MIN ≤ MAX constraint), decay rate, failure rate window

- [ ] **Step 1: Write GateStrictnessWorker tests**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.CiReliabilityCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerScope;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class GateStrictnessWorkerTest {

  private GateStrictnessWorker worker;
  private PreferenceWriter preferenceWriter;
  private CiReliabilityCapabilityArea capabilityArea;
  private WorkerScope scope;

  @BeforeEach
  void setUp() {
    preferenceWriter = mock(PreferenceWriter.class);
    capabilityArea = mock(CiReliabilityCapabilityArea.class);
    worker = new GateStrictnessWorker(preferenceWriter, capabilityArea,
        new ProportionalAdjustmentStrategy());
    scope = mock(WorkerScope.class);
    when(scope.caseId()).thenReturn(UUID.randomUUID());
    when(scope.accumulatedState()).thenReturn(Map.of("tenancyId", "casehubio"));
  }

  @Test
  void tightensGatesWhenHealthIsLow() {
    when(capabilityArea.health()).thenReturn(0.35);
    var result = worker.apply(
        Map.of("category", "gate-tightening", "target", "ci-reliability"), scope);
    assertInstanceOf(WorkerOutcome.Success.class, result.outcome());
    assertEquals("MERGED", result.output().get("status"));
    verify(preferenceWriter, atLeastOnce()).write(anyString(), anyString(), anyString(), anyString());
  }

  @Test
  void declinesWhenHealthAboveTarget() {
    when(capabilityArea.health()).thenReturn(0.75);
    var result = worker.apply(
        Map.of("category", "gate-tightening", "target", "ci-reliability"), scope);
    assertInstanceOf(WorkerOutcome.Declined.class, result.outcome());
  }
}
```

- [ ] **Step 2: Write QueuePriorityWorker tests**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.MergeQueueHealthCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerScope;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.*;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class QueuePriorityWorkerTest {

  private QueuePriorityWorker worker;
  private PreferenceWriter preferenceWriter;
  private MergeQueueHealthCapabilityArea capabilityArea;
  private WorkerScope scope;

  @BeforeEach
  void setUp() {
    preferenceWriter = mock(PreferenceWriter.class);
    capabilityArea = mock(MergeQueueHealthCapabilityArea.class);
    worker = new QueuePriorityWorker(preferenceWriter, capabilityArea,
        new ProportionalAdjustmentStrategy());
    scope = mock(WorkerScope.class);
    when(scope.caseId()).thenReturn(UUID.randomUUID());
    when(scope.accumulatedState()).thenReturn(Map.of("tenancyId", "casehubio"));
  }

  @Test
  void adjustsQueueParamsWhenHealthIsLow() {
    when(capabilityArea.health()).thenReturn(0.38);
    var result = worker.apply(
        Map.of("category", "queue-optimization", "target", "merge-queue-health"), scope);
    assertInstanceOf(WorkerOutcome.Success.class, result.outcome());
    assertEquals("MERGED", result.output().get("status"));
  }

  @Test
  void enforcesMinLessThanMaxBatchSize() {
    when(capabilityArea.health()).thenReturn(0.20);
    var result = worker.apply(
        Map.of("category", "queue-optimization", "target", "merge-queue-health"), scope);

    @SuppressWarnings("unchecked")
    List<Map<String, Object>> adjustments =
        (List<Map<String, Object>>) result.output().get("adjustments");

    double min = -1, max = -1;
    for (var adj : adjustments) {
      switch ((String) adj.get("parameter")) {
        case "min-batch-size" -> min = (double) adj.get("newValue");
        case "max-batch-size" -> max = (double) adj.get("newValue");
      }
    }
    if (min > 0 && max > 0) {
      assertTrue(min <= max, "MIN_BATCH_SIZE must be <= MAX_BATCH_SIZE");
    }
  }

  @Test
  void declinesWhenHealthAboveTarget() {
    when(capabilityArea.health()).thenReturn(0.72);
    var result = worker.apply(
        Map.of("category", "queue-optimization", "target", "merge-queue-health"), scope);
    assertInstanceOf(WorkerOutcome.Declined.class, result.outcome());
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="GateStrictnessWorkerTest,QueuePriorityWorkerTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL

- [ ] **Step 4: Implement GateStrictnessWorker**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.CiReliabilityCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static io.casehub.devtown.app.evolution.worker.EvolutionWorkerConstants.TARGET_HEALTH;

@ApplicationScoped
public class GateStrictnessWorker {

  static final AdjustmentBounds TRUST_GATE_BOUNDS = new AdjustmentBounds(
      0.10, 0.60, 0.02, 0.10, 1.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds HUMAN_APPROVAL_BOUNDS = new AdjustmentBounds(
      100, 2000, 25, 200, 1000.0,
      AdjustmentBounds.Direction.DECREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds MIN_REVIEWS_BOUNDS = new AdjustmentBounds(
      1, 3, 1, 1, 5.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);

  private final PreferenceWriter preferenceWriter;
  private final CiReliabilityCapabilityArea capabilityArea;
  private final AdjustmentStrategy strategy;

  @Inject
  public GateStrictnessWorker(
      PreferenceWriter preferenceWriter,
      CiReliabilityCapabilityArea capabilityArea,
      AdjustmentStrategy strategy) {
    this.preferenceWriter = preferenceWriter;
    this.capabilityArea = capabilityArea;
    this.strategy = strategy;
  }

  @SuppressWarnings("unchecked")
  public WorkerResult<Map<String, Object>> apply(
      Map<String, Object> input, WorkerScope scope) {

    double health = capabilityArea.health();
    if (health >= TARGET_HEALTH) {
      return WorkerResult.declined(
          String.format("Health %.2f >= target %.2f", health, TARGET_HEALTH));
    }

    List<Map<String, Object>> adjustments = new ArrayList<>();

    AdjustmentResult gateAdj = strategy.compute(
        "min-obligor-trust", 0.30, health, TARGET_HEALTH, TRUST_GATE_BOUNDS,
        "gate-tightening", scope.caseId());
    if (gateAdj.oldValue() != gateAdj.newValue()) {
      preferenceWriter.write("casehubio/devtown/trust-gate",
          "casehubio.devtown.trust-gate", "min-obligor-trust",
          String.valueOf(gateAdj.newValue()));
      adjustments.add(gateAdj.toMap());
    }

    AdjustmentResult approvalAdj = strategy.compute(
        "human-approval-threshold", 500, health, TARGET_HEALTH,
        HUMAN_APPROVAL_BOUNDS, "gate-tightening", scope.caseId());
    if (approvalAdj.oldValue() != approvalAdj.newValue()) {
      preferenceWriter.write("casehubio/devtown/pr-review",
          "devtown.pr-review", "human-approval-threshold",
          String.valueOf(Math.round(approvalAdj.newValue())));
      adjustments.add(approvalAdj.toMap());
    }

    AdjustmentResult reviewsAdj = strategy.compute(
        "min-approved-reviews", 1, health, TARGET_HEALTH,
        MIN_REVIEWS_BOUNDS, "gate-tightening", scope.caseId());
    if (reviewsAdj.oldValue() != reviewsAdj.newValue()) {
      preferenceWriter.write("casehubio/devtown/risk",
          "casehubio.devtown.risk", "merge-min-approved-reviews",
          String.valueOf(Math.round(reviewsAdj.newValue())));
      adjustments.add(reviewsAdj.toMap());
    }

    var output = new LinkedHashMap<String, Object>();
    output.put("category", "gate-tightening");
    output.put("target", "ci-reliability");
    output.put("adjustments", adjustments);
    output.put("healthBefore", health);
    output.put("status", "MERGED");
    output.put("reviewOutcome", "approved");
    output.put("requiresPr", false);
    return WorkerResult.of(output);
  }
}
```

- [ ] **Step 5: Implement QueuePriorityWorker**

```java
package io.casehub.devtown.app.evolution.worker;

import io.casehub.devtown.app.evolution.MergeQueueHealthCapabilityArea;
import io.casehub.devtown.domain.evolution.*;
import io.casehub.worker.api.WorkerResult;
import io.casehub.worker.api.WorkerScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static io.casehub.devtown.app.evolution.worker.EvolutionWorkerConstants.TARGET_HEALTH;

@ApplicationScoped
public class QueuePriorityWorker {

  static final AdjustmentBounds MAX_BATCH_BOUNDS = new AdjustmentBounds(
      2, 20, 1, 3, 10.0,
      AdjustmentBounds.Direction.DECREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds MIN_BATCH_BOUNDS = new AdjustmentBounds(
      1, 5, 1, 1, 3.0,
      AdjustmentBounds.Direction.DECREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds DECAY_RATE_BOUNDS = new AdjustmentBounds(
      50, 500, 10, 50, 200.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds FAILURE_WINDOW_BOUNDS = new AdjustmentBounds(
      5, 50, 2, 10, 30.0,
      AdjustmentBounds.Direction.INCREASE_ON_LOW_HEALTH);
  static final AdjustmentBounds HIGH_RISK_BOUNDS = new AdjustmentBounds(
      0.3, 0.8, 0.02, 0.10, 0.5,
      AdjustmentBounds.Direction.DECREASE_ON_LOW_HEALTH);

  private final PreferenceWriter preferenceWriter;
  private final MergeQueueHealthCapabilityArea capabilityArea;
  private final AdjustmentStrategy strategy;

  @Inject
  public QueuePriorityWorker(
      PreferenceWriter preferenceWriter,
      MergeQueueHealthCapabilityArea capabilityArea,
      AdjustmentStrategy strategy) {
    this.preferenceWriter = preferenceWriter;
    this.capabilityArea = capabilityArea;
    this.strategy = strategy;
  }

  @SuppressWarnings("unchecked")
  public WorkerResult<Map<String, Object>> apply(
      Map<String, Object> input, WorkerScope scope) {

    double health = capabilityArea.health();
    if (health >= TARGET_HEALTH) {
      return WorkerResult.declined(
          String.format("Health %.2f >= target %.2f", health, TARGET_HEALTH));
    }

    String mqScope = "casehubio/devtown/merge-queue";
    String mqNs = "devtown.merge-queue";

    List<Map<String, Object>> adjustments = new ArrayList<>();

    AdjustmentResult maxAdj = strategy.compute(
        "max-batch-size", 10, health, TARGET_HEALTH, MAX_BATCH_BOUNDS,
        "queue-optimization", scope.caseId());
    AdjustmentResult minAdj = strategy.compute(
        "min-batch-size", 1, health, TARGET_HEALTH, MIN_BATCH_BOUNDS,
        "queue-optimization", scope.caseId());

    double maxBatch = Math.round(maxAdj.newValue());
    double minBatch = Math.round(minAdj.newValue());
    if (minBatch > maxBatch) {
      minBatch = maxBatch;
    }

    if (maxBatch != maxAdj.oldValue()) {
      preferenceWriter.write(mqScope, mqNs, "max-batch-size",
          String.valueOf((int) maxBatch));
      adjustments.add(new AdjustmentResult("max-batch-size", maxAdj.oldValue(),
          maxBatch, maxAdj.method(), maxAdj.precedentCount(), maxAdj.rationale()).toMap());
    }
    if (minBatch != minAdj.oldValue()) {
      preferenceWriter.write(mqScope, mqNs, "min-batch-size",
          String.valueOf((int) minBatch));
      adjustments.add(new AdjustmentResult("min-batch-size", minAdj.oldValue(),
          minBatch, minAdj.method(), minAdj.precedentCount(), minAdj.rationale()).toMap());
    }

    AdjustmentResult decayAdj = strategy.compute(
        "decay-rate-per-hour", 125, health, TARGET_HEALTH, DECAY_RATE_BOUNDS,
        "queue-optimization", scope.caseId());
    if (decayAdj.oldValue() != decayAdj.newValue()) {
      preferenceWriter.write(mqScope, mqNs + ".priority", "decay-rate-per-hour",
          String.valueOf(Math.round(decayAdj.newValue())));
      adjustments.add(decayAdj.toMap());
    }

    AdjustmentResult windowAdj = strategy.compute(
        "failure-rate-window", 20, health, TARGET_HEALTH, FAILURE_WINDOW_BOUNDS,
        "queue-optimization", scope.caseId());
    if (windowAdj.oldValue() != windowAdj.newValue()) {
      preferenceWriter.write(mqScope, mqNs, "failure-rate-window",
          String.valueOf(Math.round(windowAdj.newValue())));
      adjustments.add(windowAdj.toMap());
    }

    AdjustmentResult riskAdj = strategy.compute(
        "high-risk-threshold", 0.5, health, TARGET_HEALTH, HIGH_RISK_BOUNDS,
        "queue-optimization", scope.caseId());
    if (riskAdj.oldValue() != riskAdj.newValue()) {
      preferenceWriter.write(mqScope, mqNs, "high-risk-threshold",
          String.valueOf(riskAdj.newValue()));
      adjustments.add(riskAdj.toMap());
    }

    var output = new LinkedHashMap<String, Object>();
    output.put("category", "queue-optimization");
    output.put("target", "merge-queue-health");
    output.put("adjustments", adjustments);
    output.put("healthBefore", health);
    output.put("status", "MERGED");
    output.put("reviewOutcome", "approved");
    output.put("requiresPr", false);
    return WorkerResult.of(output);
  }
}
```

- [ ] **Step 6: Complete DevtownEvolutionCaseHub dispatch**

Add all remaining cases to the dispatch switch and inject all 5 workers.

- [ ] **Step 7: Run all worker tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="ReviewerCalibrationWorkerTest,RoutingAdjustmentWorkerTest,SlaThresholdWorkerTest,GateStrictnessWorkerTest,QueuePriorityWorkerTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/ app/src/test/java/io/casehub/devtown/app/evolution/worker/
git commit -m "feat(#266): GateStrictnessWorker + QueuePriorityWorker — all 5 workers complete

Refs #266

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Cross-Repo Issues to File

After implementation, file these issues:

1. **casehub-platform-api:** Add `PreferenceWriter` SPI interface (blocking for production use)
2. **casehub-platform-persistence-jpa-common:** Add `JpaPreferenceWriter` and `LayeredPreferenceProvider`
3. **casehub-engine:** Add `when` guard to `submit-pr` binding in `self-improvement.yaml` for `requiresPr != false`
4. **casehub-engine:** Populate `ImprovementOutcome.metadata` from case context in `ImprovementOutcomeEventCapture`
5. **devtown:** Fix `RiskPreferenceKeys` key collisions (multiple constants sharing key name `"threshold"`)
6. **devtown:** Migrate `DevtownCapabilityRegistry.POLICIES` to PreferenceProvider (preference migration)
7. **devtown:** Migrate `DefaultBatchCompositionPolicy.HIGH_RISK_THRESHOLD` to PreferenceProvider

## References

- [2026-10-27-evolution-apply-workers-design.md] — design spec this plan implements
- [DevtownEvolutionCaseHub.java:7-12] — YamlCaseHub subclass, augment() extension point
- [MergeBatchCaseHub.java:37-55] — reference augment() pattern with Worker.builder()
- [WorkerFunction.java] — Sync<T,R> record for typed worker functions
- [WorkerResult.java] — result envelope with of(), declined(), failed()
- [DevtownProposalSource.java] — maps categories to improvement types and triggers
- [DevtownCategoryProvider.java] — 5 categories, 5 stages
- [self-improvement.yaml] — engine case template for improvement lifecycle
- [TrustRoutingPolicyKeys.java] — preference keys for trust routing
- [MergeQueuePreferenceKeys.java] — preference keys for queue config
- [DevtownCapabilityRegistry.java] — hardcoded routing policies (migration target)
- [PP-20260618-fc6a53] — failure cascade pattern protocol
- [PP-20260522-f08b62] — transactional loop control-flow exceptions protocol
- [#266] — focal issue
