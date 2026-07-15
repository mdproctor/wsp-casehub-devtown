# SLA Calibration from Similar Past Review Assignments

**Issue:** devtown#136
**Date:** 2026-07-16
**Status:** Approved
**Parent:** Epic #129 (Epic 11: Case-Based Reasoning), Phase 4
**Blocked by:** #131 (CBR retrieval service) ✅ closed, engine#718 (PlanTrace priorities) ✅ fixed

---

## 1. Problem Statement

The SLA breach policy (`DefaultSlaBreachPolicy`) enforces configured SLA deadlines — but those deadlines are static per-priority-lane (CRITICAL 1h, HIGH 4h, NORMAL 8h). A PR touching 3 files in a well-tested module should not have the same SLA as a PR touching 40 files across 6 modules with no tests. The CBR infrastructure already finds similar past cases — those cases carry implicit timing data that can inform a realistic estimate.

This is advisory only — the estimate surfaces alongside the configured SLA, it does not override it.

---

## 2. Data Source

Both timestamps needed to compute duration are already in the memory store:

| Timestamp | Source | When written |
|-----------|--------|-------------|
| Review start | Case-vector memory `createdAt` | `FeatureVectorEmitter.emit()` in `PrReviewCaseService.startReview()` |
| Review end | Latest outcome memory `createdAt` | `CaseMemoryEmitter.onReviewCompleted()` on `ReviewCompletedEvent` |

**Duration** = latest outcome `createdAt` − case-vector `createdAt`.

No new data capture is required. The timestamps exist on every completed past case that has both a case-vector and at least one outcome memory.

---

## 3. Changes

### 3.1 Enrich Precedent with completion time

Add `Duration completionTime` to `Precedent`:

```java
// domain/src/main/java/io/casehub/devtown/domain/cbr/Precedent.java

public record Precedent(
        UUID caseId,
        SimilarityScore similarity,
        PrFeatureVector vector,
        String outcome,
        Map<String, CapabilityOutcome> capabilityOutcomes,
        Duration completionTime
) {}
```

`completionTime` is nullable — cases where no outcome memory exists (e.g., FAULTED before any capability completed) have `null` and are excluded from SLA estimation.

### 3.2 Thread start timestamp through retrieval

In `DefaultCbrRetrievalService`:

**`CandidateVector`** gains `Instant startedAt` — carried from the case-vector memory's `createdAt`:

```java
private record CandidateVector(UUID caseId, PrFeatureVector vector, String contributor, Instant startedAt) {}
```

**`toCandidateVector()`** passes `memory.createdAt()` through:

```java
return new CandidateVector(caseId, stored, stored.contributor(), memory.createdAt());
```

**`scoreCandidate()`** computes duration by finding the latest outcome memory's `createdAt` for the case:

```java
Instant latestOutcome = outcomeFacts.stream()
    .map(Memory::createdAt)
    .max(Instant::compareTo)
    .orElse(null);

Duration completionTime = (latestOutcome != null && cv.startedAt() != null)
    ? Duration.between(cv.startedAt(), latestOutcome)
    : null;
```

The outcome memories are already queried in `enrichOutcomes()` — the `latestOutcome` extraction uses the same `outcomeFacts` list. One structural change: `enrichOutcomes()` must return both the outcomes map AND the latest timestamp. Refactor to return a result record:

```java
private record EnrichmentResult(
    Map<String, CapabilityOutcome> outcomes,
    Instant latestOutcomeTime
) {}
```

### 3.3 SlaEstimator (new, domain/)

Pure domain logic — no CDI, no framework dependencies.

```java
// domain/src/main/java/io/casehub/devtown/domain/sla/SlaEstimator.java

public final class SlaEstimator {

    public static Optional<SlaEstimate> estimate(List<Precedent> precedents) {
        List<Duration> durations = precedents.stream()
            .map(Precedent::completionTime)
            .filter(Objects::nonNull)
            .filter(d -> !d.isNegative() && !d.isZero())
            .sorted()
            .toList();

        if (durations.isEmpty()) return Optional.empty();

        Duration median = durations.get(durations.size() / 2);
        Duration min = durations.getFirst();
        Duration max = durations.getLast();

        return Optional.of(new SlaEstimate(median, durations.size(), min, max));
    }

    private SlaEstimator() {}
}
```

### 3.4 SlaEstimate (new, domain/)

```java
// domain/src/main/java/io/casehub/devtown/domain/sla/SlaEstimate.java

public record SlaEstimate(
    Duration median,
    int precedentCount,
    Duration min,
    Duration max
) {
    public Map<String, Object> toContextMap() {
        return Map.of(
            "medianMinutes", median.toMinutes(),
            "precedentCount", precedentCount,
            "minMinutes", min.toMinutes(),
            "maxMinutes", max.toMinutes()
        );
    }
}
```

### 3.5 Wire into PrReviewCaseService.startReview()

After the existing `findSimilar()` call, compute the estimate and write to case context:

```java
List<Precedent> precedents = cbrRetrievalService.findSimilar(vector, repo, tenantId);

// SLA calibration — advisory estimate from similar past reviews
SlaEstimator.estimate(precedents).ifPresent(estimate ->
    caseHub.signal(caseId, "slaEstimate", estimate.toContextMap()));
```

The `slaEstimate` key in case context is readable by MCP tools (`get_case_detail`) and the governance workbench case detail view. No binding reads it — it is purely advisory.

---

## 4. Module Placement

| File | Module | Rationale |
|------|--------|-----------|
| `Precedent` (modified) | `domain/` | Existing domain record — add field |
| `SlaEstimator` | `domain/sla/` | Pure domain logic alongside `DefaultSlaBreachPolicy` |
| `SlaEstimate` | `domain/sla/` | Domain value object |
| `DefaultCbrRetrievalService` (modified) | `app/` | Thread timestamps, return enriched precedents |
| `PrReviewCaseService` (modified) | `app/` | Wire estimate into case start |

---

## 5. Testing Strategy

### 5.1 Unit Tests (domain/)

| Test | Covers |
|------|--------|
| `SlaEstimatorTest` | Empty precedents → empty Optional |
| | All null durations → empty Optional |
| | Single precedent → median equals that duration |
| | Odd count → middle element is median |
| | Even count → lower-middle element is median |
| | Negative/zero durations filtered out |
| | Min and max correct |
| `SlaEstimateTest` | `toContextMap()` produces correct keys and minute conversions |
| | Sub-minute durations round to 0 |

### 5.2 Unit Tests (app/)

| Test | Covers |
|------|--------|
| `DefaultCbrRetrievalServiceTest` (modified) | Precedents include `completionTime` computed from memory timestamps |
| | Missing outcome memories → null completionTime |
| | Case-vector with null createdAt → null completionTime |

### 5.3 Integration Tests (@QuarkusTest, app/)

| Test | Covers |
|------|--------|
| `SlaCalibrationIntegrationTest` | `startReview()` with similar past cases → `slaEstimate` in case context |
| | No similar cases → no `slaEstimate` in context |
| | Similar cases with no outcome memories → no `slaEstimate` |

---

## 6. Not in Scope

- Overriding configured SLA based on the estimate (advisory only)
- Per-capability duration breakdown (total only — can be added later)
- Governance workbench UI changes (case context is already displayed)
- Persisting the estimate to a database (case context is sufficient)

---

## 7. Revision History

- **v1 (2026-07-16):** Initial design. Duration computed from existing memory timestamps (case-vector start, latest outcome completion). SlaEstimator as pure domain logic. Advisory estimate in case context.
