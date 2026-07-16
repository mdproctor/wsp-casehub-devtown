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

**Independence from SLA start mode:** This duration measures actual wall-clock time from case creation to final outcome, independent of the configured `SlaStartFrom` mode in the milestone schema. The calibration answers "how long did similar reviews actually take?" — not "how much SLA clock time elapsed." This remains valid regardless of future SLA start modes (`PREVIOUS_MILESTONE_COMPLETED`, `EVENT_OCCURRED`, etc.).

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

**Downstream effects of the Precedent change:**

- **`MemoryContext.toContextMap()`** — include `completionTimeSeconds` in the serialized precedent map (when non-null). Requires switching from `Map.of()` to a mutable map for nullable handling:
  ```java
  var m = new LinkedHashMap<String, Object>();
  m.put("caseId", p.caseId().toString());
  // ... existing fields ...
  if (p.completionTime() != null) {
      m.put("completionTimeSeconds", p.completionTime().toSeconds());
  }
  ```
- **`DevtownMcpTools.findSimilarCases()`** — returns `List<Precedent>` directly to MCP consumers. Adding `completionTime` changes the MCP tool's JSON response schema. This is desirable — governance agents benefit from per-precedent timing data. `null` values serialize as `"completionTime": null` in JSON for cases without outcomes.

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

Duration completionTime = null;
if (latestOutcome != null && cv.startedAt() != null) {
    Duration raw = Duration.between(cv.startedAt(), latestOutcome);
    if (raw.isNegative()) {
        LOG.warnf("Negative completion time for case=%s: start=%s outcome=%s — possible clock skew or async race",
                  cv.caseId(), cv.startedAt(), latestOutcome);
    } else {
        completionTime = raw;
    }
}
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

**Design note — unweighted median:** All precedents contribute equally to the estimate regardless of similarity score. This is intentional: (1) precedents already pass the configured minimum similarity threshold, so all are considered relevant matches; (2) median is inherently robust to outliers — a single marginal-similarity case cannot skew the estimate; (3) this is advisory, not binding — precision beyond "reasonable ballpark" adds complexity without value. Similarity-weighted estimation can be explored as a refinement if calibration accuracy becomes a concern.

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
            "medianSeconds", median.toSeconds(),
            "precedentCount", precedentCount,
            "minSeconds", min.toSeconds(),
            "maxSeconds", max.toSeconds()
        );
    }
}
```

### 3.5 Wire into PrReviewCaseService.startReview()

After the existing `memoryRecaller.recall(pr)` call, compute the estimate from the recalled precedents and include it in initial case context:

```java
var memoryContext = memoryRecaller.recall(pr);

// ... existing context building (pr, policy, memory, ci) ...
initialContext.put("memory", memoryContext.toContextMap());

// SLA calibration — advisory estimate from similar past reviews
SlaEstimator.estimate(memoryContext.precedents()).ifPresent(estimate ->
    initialContext.put("slaEstimate", estimate.toContextMap()));

UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().join();
```

No new dependencies required. The precedent list is already available from `memoryContext.precedents()`, populated by `CaseMemoryRecaller.recall()` via the existing CBR retrieval path (`cbrService.get().findSimilar()`). The estimate is included in initial case context, making it available from case creation to MCP tools (`get_case_detail`) and the governance workbench case detail view. No binding reads it — it is purely advisory.

**Invalidation on revise:** `PrReviewCaseService.revisePr()` must invalidate the SLA estimate alongside the existing capability analysis invalidations. The estimate was computed from CBR precedents matched against the original PR's feature vector — after a revise, `linesChanged` and `changedPaths` change, making the original match set (and therefore the estimate) stale. Add to the existing invalidation block:

```java
caseHub.signal(caseId, "slaEstimate", null);
```

Recomputation on revise is not in scope — it would require CBR recall in the revise path, which currently only updates metadata and invalidates stale results. The null-out is consistent with how every other content-dependent result is treated: absent is better than stale for advisory data.

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
| | Even count → upper-middle element is median (conservative: biases toward longer estimates) |
| | Negative/zero durations filtered out |
| | Min and max correct |
| `SlaEstimateTest` | `toContextMap()` produces correct keys and second conversions |
| | Sub-second durations truncate to 0 (acceptable for SLA advisory) |

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
| | `revisePr()` after `startReview()` → `slaEstimate` nulled out |

---

## 6. Not in Scope

The following items are deferred and tracked as GitHub issues:

- Overriding configured SLA based on the estimate (advisory only) — devtown#151
- Per-capability duration breakdown (total only — can be added later) — devtown#152
- Governance view showing estimated vs configured SLA side by side — devtown#153 (criterion 3 from #136; this spec addresses criteria 1 and 2)
- Persisting the estimate to a database (case context is sufficient) — devtown#154

---

## 7. Revision History

- **v1 (2026-07-16):** Initial design. Duration computed from existing memory timestamps (case-vector start, latest outcome completion). SlaEstimator as pure domain logic. Advisory estimate in case context.
- **v2 (2026-07-16):** Review round 1 fixes. Corrected §3.5 wiring to use `memoryContext.precedents()` instead of nonexistent `cbrRetrievalService` call. Fixed median test description (upper-middle, not lower-middle). Added negative duration warning at source. Changed toContextMap() from minutes to seconds. Documented Precedent downstream effects (MemoryContext serialization, MCP API). Added SlaStartFrom independence note. Added design rationale for unweighted median. Deferred scope items tracked as GitHub issues. Governance view criterion (#136 criterion 3) deferred to devtown#153.
- **v3 (2026-07-16):** Review round 2 fix. Added `slaEstimate` invalidation in `revisePr()` — consistent with existing capability analysis invalidation pattern. Stale advisory data is worse than absent.
