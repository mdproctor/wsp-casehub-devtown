# SLA Calibration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #136 — feat: SLA calibration from similar past review assignments
**Issue group:** #136

**Goal:** Compute median review completion time from similar past cases and surface it as an advisory SLA estimate in case context.

**Architecture:** `SlaEstimator` (pure domain logic) computes median from `Precedent.completionTime`. Duration threaded through existing CBR retrieval from memory timestamps. Estimate written to initial case context at review start.

**Tech Stack:** Java 21, Quarkus 3.32.2

## Global Constraints

- `SlaEstimator` and `SlaEstimate` in `domain/sla/` — pure Java, no CDI
- `Precedent` field addition is a breaking change — all callers updated
- Duration = case-vector `createdAt` − latest outcome memory `createdAt`
- Advisory only — no binding reads `slaEstimate`
- `project_path` for all IntelliJ MCP calls: `/Users/mdproctor/claude/casehub/devtown`

---

### Task 1: Domain types — SlaEstimate + SlaEstimator

Create the pure domain types. No dependencies on existing code.

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/sla/SlaEstimate.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/sla/SlaEstimator.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/sla/SlaEstimateTest.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/sla/SlaEstimatorTest.java`

**Interfaces:**
- Consumes: `Precedent` (existing, read-only — `completionTime` field added in Task 2)
- Produces: `SlaEstimator.estimate(List<Precedent>) → Optional<SlaEstimate>`, `SlaEstimate.toContextMap() → Map<String, Object>`

- [ ] **Step 1: Write SlaEstimator and SlaEstimate tests**

`SlaEstimatorTest` — covers: empty list, all-null durations, single precedent, odd count median, even count (upper-middle), negative/zero filtered, min/max correct.

`SlaEstimateTest` — covers: `toContextMap()` keys and second conversions, sub-second truncation.

Note: These tests will initially reference `Precedent` with its current 5-arg constructor. They will need updating in Task 2 when `completionTime` is added — but the `SlaEstimator` logic itself is independent of how `completionTime` gets populated.

- [ ] **Step 2: Create SlaEstimate and SlaEstimator**

`SlaEstimate`: record with `median`, `precedentCount`, `min`, `max` (all `Duration`). `toContextMap()` converts to seconds.

`SlaEstimator`: static `estimate()` method — filters nulls/negatives/zeros, sorts, takes middle element for median.

- [ ] **Step 3: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest="SlaEstimatorTest,SlaEstimateTest"`

- [ ] **Step 4: Commit**

`feat(#136): add SlaEstimator and SlaEstimate domain types`

---

### Task 2: Enrich Precedent with completionTime + thread through retrieval

Add `Duration completionTime` to `Precedent`, refactor `DefaultCbrRetrievalService` to compute it from memory timestamps, update `MemoryContext.toContextMap()`.

**Files:**
- Modify: `domain/src/main/java/io/casehub/devtown/domain/cbr/Precedent.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/DefaultCbrRetrievalService.java`
- Modify: `review/src/main/java/io/casehub/devtown/review/MemoryContext.java`
- Modify: existing tests for `Precedent`, `DefaultCbrRetrievalService`, `MemoryContext`

**Interfaces:**
- Consumes: `Memory.createdAt()` (existing)
- Produces: `Precedent.completionTime()` (nullable `Duration`), updated `MemoryContext.toContextMap()` with `completionTimeSeconds`

- [ ] **Step 1: Update Precedent record**

Add `Duration completionTime` as 6th field. Use `ide_edit_member` on `Precedent`.

- [ ] **Step 2: Fix all compilation errors from Precedent change**

Find all callers creating `Precedent` instances — use `ide_find_references` on the constructor. Update each to pass `completionTime` (null for callers not yet computing it, real value in `scoreCandidate`).

- [ ] **Step 3: Add `EnrichmentResult` and refactor `enrichOutcomes()`**

In `DefaultCbrRetrievalService`: create inner record `EnrichmentResult(Map<String, CapabilityOutcome> outcomes, Instant latestOutcomeTime)`. Refactor `enrichOutcomes()` to return it. Update `scoreCandidate()` to use the result.

- [ ] **Step 4: Thread `startedAt` through `CandidateVector`**

Add `Instant startedAt` to `CandidateVector`. Pass `memory.createdAt()` in `toCandidateVector()`.

- [ ] **Step 5: Compute `completionTime` in `scoreCandidate()`**

Use `cv.startedAt()` and `enrichment.latestOutcomeTime()` to compute `Duration.between()`. Add negative duration warning log. Pass to `Precedent` constructor.

- [ ] **Step 6: Update `MemoryContext.toContextMap()` for completionTime**

Switch precedent serialization from `Map.of()` to `LinkedHashMap` to handle nullable `completionTimeSeconds`. Add `completionTimeSeconds` when non-null.

- [ ] **Step 7: Update existing tests**

Fix `DefaultCbrRetrievalServiceTest` — verify `completionTime` on returned precedents. Fix `MemoryContextTest` — verify `completionTimeSeconds` in serialized map. Fix `SlaEstimatorTest` from Task 1 — use 6-arg `Precedent` constructor.

- [ ] **Step 8: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain,review,app -Dtest="*Precedent*,*CbrRetrieval*,*MemoryContext*,*SlaEstimator*,*SlaEstimate*"`

- [ ] **Step 9: Commit**

`feat(#136): enrich Precedent with completionTime from memory timestamps`

---

### Task 3: Wire SLA estimate into PrReviewCaseService

Write estimate to case context at review start, invalidate on revise.

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`
- Test: `app/src/test/java/io/casehub/devtown/app/SlaCalibrationIntegrationTest.java`

**Interfaces:**
- Consumes: `SlaEstimator.estimate()`, `MemoryContext.precedents()`, `SlaEstimate.toContextMap()`
- Produces: `slaEstimate` key in case context

- [ ] **Step 1: Write integration test**

`SlaCalibrationIntegrationTest` (@QuarkusTest): verify `slaEstimate` appears in case context after `startReview()` with populated precedents. Verify absent when no precedents. Verify nulled after `revisePr()`.

- [ ] **Step 2: Wire in startReview()**

After `initialContext.put("memory", memoryContext.toContextMap())`, add:
```java
SlaEstimator.estimate(memoryContext.precedents()).ifPresent(estimate ->
    initialContext.put("slaEstimate", estimate.toContextMap()));
```

- [ ] **Step 3: Add invalidation in revisePr()**

After the existing `performanceAnalysis` invalidation, add:
```java
caseHub.signal(caseId, "slaEstimate", null);
```

- [ ] **Step 4: Run tests, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="SlaCalibrationIntegrationTest,PrReviewCaseServiceLifecycleTest"`

- [ ] **Step 5: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain,review,app`

- [ ] **Step 6: Commit**

`feat(#136): wire SLA calibration estimate into case context`

---

## Post-Implementation

After all 3 tasks: invoke `work-end` which handles code review, squash, push, and branch closure.
