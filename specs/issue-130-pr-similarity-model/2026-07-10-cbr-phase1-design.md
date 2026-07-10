# CBR Phase 1 — PR Similarity Model and Retrieval Service

> **Date:** 2026-07-10
> **Issues:** #130 (PR similarity model), #131 (CBR retrieval service)
> **Epic:** #129 (Epic 11: Case-Based Reasoning)
> **Branch:** issue-130-pr-similarity-model

---

## Overview

Adds the Retrieve step to devtown's degenerate CBR cycle (currently Retain+Reuse only — see GE-20260612-bd3b4d). Structured feature similarity (not embeddings) — deterministic, explainable, auditable. Every CBR-informed routing decision is traceable through the existing ledger.

Two deliverables:
1. **PR similarity model** (#130) — `PrFeatureVector` + `WeightedJaccardSimilarity` in `devtown-domain`. Pure Java.
2. **CBR retrieval service** (#131) — `CbrRetrievalService` in `review/`, `DefaultCbrRetrievalService` + `FeatureVectorEmitter` in `app/`. Integrates with `CaseMemoryStore` and `MemoryContext`.

---

## Design Decisions

### Feature vector as case-scoped memory fact

The feature vector is stored as a `case-vector:<caseId>` memory fact in `CaseMemoryStore`. Reconstruction from existing entity-scoped facts (contributor, reviewer, module) is impossible — raw `changedPaths` aren't preserved in any stored fact today. Storage cost is ~1-3KB per case, trivial at any realistic scale.

### Emission at case open, not review completion

`CaseMemoryEmitter` fires on `ReviewCompletedEvent` — per capability review, multiple times per case. The feature vector describes the PR's structure (immutable once submitted), not a review outcome. Emitted once at case open time in `PrReviewCaseService.startReview()`, before `caseHub.startCase()`.

Outcomes are already captured by existing contributor/reviewer/module facts. At retrieval time, the `Precedent` joins the feature vector with outcome facts for the same caseId.

### Weighted linear combination for similarity scoring

Each dimension produces a score in [0,1]. Final similarity is a weighted sum normalised by total weight. Simple, explainable, auditable. Hard gates deferred to #143.

---

## §1 Domain Model (`devtown-domain`)

All types in `io.casehub.devtown.domain.cbr`. Pure Java, no Quarkus dependencies.

### PrFeatureVector

```java
public record PrFeatureVector(
    String repo,
    int prNumber,
    String contributor,
    int linesChanged,
    Set<String> changedPaths,
    Set<String> modules,
    Set<String> languages,
    boolean hasTests,
    boolean touchedConfigs
)
```

**`PrFeatureVector.from(PrPayload)`** — static factory extracting features:
- `modules` — via existing `ModulePathNormalizer.normalize()`, converted to `Set`
- `languages` — file extension mapping (`.java` → `java`, `.ts`/`.tsx` → `typescript`, `.kt` → `kotlin`, `.py` → `python`, `.go` → `go`, `.rs` → `rust`, `.xml`/`.yaml`/`.yml`/`.json`/`.properties` → `config`). No-extension files ignored.
- `hasTests` — any path contains `/test/`, `/tests/`, or matches `*Test.java`, `*.test.ts`, `*.spec.ts`, `*_test.go`, `*_test.py`
- `touchedConfigs` — any path matches `pom.xml`, `build.gradle`, `build.gradle.kts`, `*.properties`, `*.yaml`, `*.yml`, `*.json`, `Dockerfile`, `.github/*`

**`PrFeatureVector.toAttributes()`** — serialises to `Map<String, String>` for memory storage:
- Sets serialised as comma-separated strings
- Booleans as `"true"`/`"false"`
- Integers as string values

**`PrFeatureVector.fromAttributes(Map<String, String>)`** — static factory reconstructing from stored attributes. Used by retrieval service.

### SimilarityScore

```java
public record SimilarityScore(
    double score,
    Map<String, Double> breakdown
) implements Comparable<SimilarityScore>
```

`score` is [0,1]. `breakdown` maps dimension name → individual score for ledger traceability. Natural ordering by `score` descending.

### SimilarityMetric

```java
public interface SimilarityMetric {
    SimilarityScore compute(PrFeatureVector a, PrFeatureVector b);
}
```

### WeightedJaccardSimilarity

Implements `SimilarityMetric`. Constructor takes 5 weights:

| Dimension | Computation | Default weight | Rationale |
|-----------|-------------|---------------|-----------|
| `file-paths` | Jaccard on `changedPaths` sets | 1.0 | Fine-grained overlap; meaningful but noisy for large PRs |
| `modules` | Jaccard on `modules` sets | 1.5 | Strongest structural signal; same modules → most relevant precedent |
| `languages` | Jaccard on `languages` sets | 0.5 | Coarse signal; most repos are dominated by one language |
| `change-size` | `1.0 - |a - b| / max(a, b)` | 1.0 | Similar scale → similar review dynamics |
| `contributor` | 1.0 if same, 0.0 otherwise | 0.5 | Relevant but shouldn't dominate; trust routing handles per-reviewer quality |

Final score: `Σ(weight × dimensionScore) / Σ(weight)`. Zero-weight dimensions excluded from both numerator and denominator. All weights zero → score 0.0.

Jaccard on empty sets: returns 0.0 (two empty sets have no overlap, not perfect similarity).

---

## §2 Feature Vector Storage

### FeatureVectorEmitter (`app/`)

```java
@ApplicationScoped
public class FeatureVectorEmitter {
    @Inject Instance<CaseMemoryStore> store;

    public void emit(UUID caseId, String tenantId, PrFeatureVector vector);
}
```

**Memory fact shape:**
- `entityId`: `"case-vector:" + caseId`
- `domain`: `DevtownMemoryDomain.SOFTWARE_REVIEW`
- `tenantId`: from `CurrentPrincipal`
- `caseId`: the case UUID as string
- `text`: `"PR #%d in %s: %d lines, %d modules, %s"` (human-readable summary)
- `attributes`: all fields from `vector.toAttributes()` plus `DevtownMemoryKeys.ENTITY_TYPE = "case-vector"`

**Emission point:** `PrReviewCaseService.startReview()`, after `memoryRecaller.recall()`, before `caseHub.startCase()`. The caseId is generated before emission (UUID.randomUUID or from caseHub).

**Fail-open:** emission failure is logged and swallowed. Case proceeds without stored vector.

### New DevtownMemoryKeys constants

```java
public static final String CHANGED_PATHS = "changed-paths";
public static final String MODULES = "modules";
public static final String LANGUAGES = "languages";
public static final String HAS_TESTS = "has-tests";
public static final String TOUCHED_CONFIGS = "touched-configs";
```

`ENTITY_TYPE` with value `"case-vector"` reuses the existing key.

### Entity ID prefix

New constant in `DevtownMemoryDomain`:
```java
public static final String CASE_VECTOR_PREFIX = "case-vector:";
```

---

## §3 Retrieval Service

### Precedent (`review/`)

```java
public record Precedent(
    UUID caseId,
    SimilarityScore similarity,
    PrFeatureVector vector,
    String outcome,
    Map<String, String> capabilityOutcomes
)
```

`outcome` is the aggregate case result: `"approved"`, `"flagged"`, `"failed"`. Derived from capability outcomes — all approved → approved, any failed → failed, otherwise flagged.

### CbrRetrievalService (`review/`)

Port interface — no Quarkus dependencies:

```java
public interface CbrRetrievalService {
    List<Precedent> findSimilar(PrFeatureVector query, String repo, String tenantId);
}
```

Returns precedents ranked by similarity score descending. Empty list if no precedents found or on failure.

### DefaultCbrRetrievalService (`app/`)

CDI implementation. Pipeline:

1. **Scan** — `MemoryScanRequest` with `tenantId`, `domain="software-review"`, `attributeKey="entity-type"`, `attributeValue="case-vector"`, paginated via `afterMemoryId`.
2. **Filter** — `pr-repo` attribute matches `repo`. `createdAt` within time window.
3. **Score** — `PrFeatureVector.fromAttributes(memory.attributes())` for each stored vector. `SimilarityMetric.compute(query, stored)`.
4. **Rank** — sort by score descending, filter below minimum threshold, take top K.
5. **Enrich** — for top-K case IDs, query outcome facts. Scan by caseId attribute across contributor/reviewer/module entity types. Derive per-capability outcomes and aggregate outcome.

Configurable parameters via `PreferenceProvider`:

| Parameter | Key | Default |
|-----------|-----|---------|
| K limit | `cbr.k-limit` | 5 |
| Min threshold | `cbr.min-threshold` | 0.3 |
| Time window | `cbr.time-window-days` | 180 |

`SettingsScope`: `casehubio/devtown/cbr`.

Similarity weights also resolved from preferences at construction/injection time, passed to `WeightedJaccardSimilarity`.

### Integration with MemoryContext

`MemoryContext` gains a third field:

```java
public record MemoryContext(
    List<Memory> contributorHistory,
    List<Memory> codeAreaHistory,
    List<Precedent> precedents
)
```

`MemoryContext.EMPTY` updated to include `List.of()` for precedents.

`CaseMemoryRecaller.recall()` gains `CbrRetrievalService` as a dependency:
- After existing contributor/module queries, extracts `PrFeatureVector.from(pr)`
- Calls `cbrRetrievalService.findSimilar(vector, pr.repo(), tenantId)`
- Passes results into `MemoryContext`
- CBR failure is independent of contributor/module recall — either can fail without affecting the other

`MemoryContext.toContextMap()` extended to include precedents as a serialised list.

`MemoryContext.hasRiskSignals()` extended: also returns true if any precedent has `outcome = "failed"`.

---

## §4 Preference Keys (`devtown-domain`)

New class `CbrPreferenceKeys` in `io.casehub.devtown.domain.cbr`:

```java
public final class CbrPreferenceKeys {
    // Similarity weights
    public static final DoublePreference WEIGHT_FILE_PATHS;      // 1.0
    public static final DoublePreference WEIGHT_MODULES;         // 1.5
    public static final DoublePreference WEIGHT_LANGUAGES;       // 0.5
    public static final DoublePreference WEIGHT_CHANGE_SIZE;     // 1.0
    public static final DoublePreference WEIGHT_CONTRIBUTOR;     // 0.5

    // Retrieval parameters
    public static final IntPreference K_LIMIT;                   // 5
    public static final DoublePreference MIN_THRESHOLD;          // 0.3
    public static final IntPreference TIME_WINDOW_DAYS;          // 180
}
```

`SettingsScope`: `casehubio/devtown/cbr` for all keys.

---

## §5 Testing

### Unit tests (`devtown-domain`)

**`PrFeatureVectorTest`:**
- Extraction from `PrPayload` — correct modules, languages, hasTests, touchedConfigs
- Empty file list → empty sets, hasTests=false, touchedConfigs=false
- Single-file PR → single module or `(root)`
- Language edge cases: `.tsx` → `typescript`, `.gradle.kts` → `kotlin`, no extension → ignored
- Attribute round-trip: `fromAttributes(vector.toAttributes())` equals original

**`WeightedJaccardSimilarityTest`:**
- Identical PRs → score 1.0
- Completely disjoint PRs → score 0.0
- Same contributor, different files → contributor dimension contributes, paths don't
- Single shared file in disjoint sets → small non-zero path Jaccard
- Change size ratio: 100/100 → 1.0, 100/200 → 0.5, 1/1000 → ~0.001
- Zero-weight dimension excluded from scoring
- All weights zero → score 0.0 (no division by zero)
- Breakdown map contains all 5 dimensions with correct individual scores
- Empty sets: Jaccard returns 0.0

### Integration tests (`app/`)

**`CbrIntegrationTest`** (`@QuarkusTest`):
- Emit 5 feature vectors via `FeatureVectorEmitter` with varying file overlap
- Submit a 6th PR → `CbrRetrievalService.findSimilar()` returns precedents ranked by similarity
- Most similar case is the one sharing the most file paths
- Cases below threshold excluded
- K limit respected
- Cases outside time window excluded
- Retrieval failure (store unavailable) → empty list (fail-open)

**`CbrMemoryContextIntegrationTest`** (`@QuarkusTest`):
- Full lifecycle: `startReview()` → feature vector stored → reviews complete → memory emitted → new `startReview()` → `MemoryContext.precedents()` populated
- Precedents appear in `initialContext` passed to `caseHub.startCase()`

---

## §6 Module Placement

| Type | Module | Package |
|------|--------|---------|
| `PrFeatureVector` | `devtown-domain` | `io.casehub.devtown.domain.cbr` |
| `SimilarityScore` | `devtown-domain` | `io.casehub.devtown.domain.cbr` |
| `SimilarityMetric` | `devtown-domain` | `io.casehub.devtown.domain.cbr` |
| `WeightedJaccardSimilarity` | `devtown-domain` | `io.casehub.devtown.domain.cbr` |
| `CbrPreferenceKeys` | `devtown-domain` | `io.casehub.devtown.domain.cbr` |
| `Precedent` | `review` | `io.casehub.devtown.review` |
| `CbrRetrievalService` | `review` | `io.casehub.devtown.review` |
| `DefaultCbrRetrievalService` | `app` | `io.casehub.devtown.app` |
| `FeatureVectorEmitter` | `app` | `io.casehub.devtown.app` |

Follows the three-tier rule: domain = pure Java, review = integration logic (ports), app = CDI wiring.

---

## §7 Deferred

- **#143** — Hard gates (minimum overlap filters before scoring)
- **#132** — CBR-enhanced capability activation (Phase 2 — consumes `Precedent`)
- **#133** — CBR-enhanced reviewer matching (Phase 2 — consumes `Precedent`)
- **#138** — Similarity weight refinement from outcome feedback (Phase 4 — adjusts `CbrPreferenceKeys` weights)
