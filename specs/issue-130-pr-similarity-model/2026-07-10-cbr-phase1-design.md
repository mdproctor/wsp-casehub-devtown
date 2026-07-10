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

The existing `CaseMemoryEmitter` fires on `ReviewCompletedEvent` — per capability review, multiple times per case. The feature vector describes the PR's structure (immutable once submitted), not a review outcome. Emitted once at case open time in the `PrReviewApplicationService` implementation's `startReview()`, before `caseHub.startCase()`.

Outcomes are already captured by existing contributor/reviewer/module facts (via `CaseMemoryEmitter` → `ReviewOutcomeObserver`). At retrieval time, the `Precedent` joins the feature vector with outcome facts for the same caseId.

### Why `CaseMemoryStore` not `CbrCaseMemoryStore` for Phase 1

The platform's `CbrCaseMemoryStore` (neocortex) provides structured CBR infrastructure with `FeatureVectorCbrCase`, `CbrQuery`, `CbrFeatureSchema`, and multiple implementations (`InMemory`, `Qdrant`, `Reranking`). This spec uses the general-purpose `CaseMemoryStore` instead because of four platform gaps:

1. **No set-valued feature type.** `FeatureField` supports `Categorical`, `Numeric`, and `Text`. Three of five similarity dimensions (`changedPaths`, `modules`, `languages`) require Jaccard on `Set<String>`. No existing field type handles this.
2. **No custom similarity passthrough.** `CbrSimilarityScorer.score()` accepts `LocalSimilarityFunction` overrides, but `InMemoryCbrCaseMemoryStore.retrieveSimilar()` passes `Map.of()`. Custom similarity functions cannot be injected per-query.
3. **`FeatureVectorCbrCase` requires non-blank `solution`.** The feature vector is stored at case-open time, before any review runs — there is no solution yet. The CBR case model assumes a complete problem→solution→outcome tuple.
4. **No per-dimension breakdown.** `ScoredCbrCase` returns a single `double score`. The spec requires per-dimension breakdown (`SimilarityScore.breakdown`) for ledger auditability.

**Platform extension needed:** `FeatureField.SetValued` with Jaccard as default similarity (a single sealed-interface case + `CbrSimilarityScorer` switch branch). When neocortex gains this, devtown migrates to `CbrCaseMemoryStore` — see §7.

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

**`PrFeatureVector.from(String repo, int prNumber, String contributor, int linesChanged, List<String> changedPaths)`** — static factory extracting features from primitive parameters (no `PrPayload` dependency — keeps `devtown-domain` independent of `review/`):
- `modules` — via existing `ModulePathNormalizer.normalize()`, converted to `Set`
- `languages` — file extension mapping:
  - `.java` → `java`, `.kt`/`.kts` → `kotlin`
  - `.ts`/`.tsx` → `typescript`, `.js`/`.jsx` → `javascript`
  - `.py` → `python`, `.go` → `go`, `.rs` → `rust`
  - `.rb` → `ruby`, `.scala` → `scala`
  - `.c`/`.cpp`/`.h` → `c`, `.cs` → `csharp`
  - `.swift` → `swift`, `.sh` → `shell`
  - `.xml`/`.yaml`/`.yml`/`.json`/`.properties` → `config`
  - No-extension files ignored (no-extension build files like `Makefile` are handled by `touchedConfigs`)
- `hasTests` — any path contains `/test/`, `/tests/`, `__tests__/`, or matches `*Test.java`, `*.test.ts`, `*.test.tsx`, `*.spec.ts`, `*.spec.tsx`, `*_test.go`, `*_test.py`, `test_*.py`
- `touchedConfigs` — any path matches `pom.xml`, `build.gradle`, `build.gradle.kts`, `settings.gradle`, `gradle.properties`, `*.properties`, `*.yaml`, `*.yml`, `*.json`, `package.json`, `tsconfig.json`, `requirements.txt`, `setup.py`, `setup.cfg`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `go.sum`, `Dockerfile`, `Makefile`, `Jenkinsfile`, `Rakefile`, `.eslintrc.*`, `.prettierrc`, `.github/*`

**`PrFeatureVector.toAttributes()`** — serialises to `Map<String, String>` for memory storage:
- Sets serialised as JSON arrays (e.g., `["src/main/Foo.java","src/test/FooTest.java"]`) — handles paths containing commas
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
| `change-size` | `1.0 - |a - b| / max(a, b)` (when both are 0: 1.0) | 1.0 | Similar scale → similar review dynamics |
| `contributor` | 1.0 if same, 0.0 otherwise | 0.5 | Relevant but shouldn't dominate; trust routing handles per-reviewer quality |

Final score: `Σ(weight × dimensionScore) / Σ(weight)`. Zero-weight dimensions excluded from both numerator and denominator. All weights zero → score 0.0.

Jaccard on empty sets: returns 1.0 (standard mathematical convention — identical sets, including both empty). Two PRs that both lack a feature are alike on that dimension.

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
- `entityId`: `"case-vector:" + repo + ":" + caseId`
- `domain`: `DevtownMemoryDomain.SOFTWARE_REVIEW`
- `tenantId`: from `CurrentPrincipal`
- `caseId`: the case UUID as string
- `text`: `"PR #%d in %s: %d lines, %d modules, %s"` (human-readable summary)
- `attributes`: all fields from `vector.toAttributes()` plus:
  - `DevtownMemoryKeys.ENTITY_TYPE = "case-vector"`
  - `DevtownMemoryKeys.PR_REPO = vector.repo()` — enables attribute-based scan filtering by repo (matches existing outcome fact convention from `CaseMemoryEmitter`)

**Emission point:** Production implementation of `PrReviewApplicationService.startReview()`, after `memoryRecaller.recall()`, before `caseHub.startCase()`. Sequence:

1. `caseId = UUID.randomUUID()` — generated before both emission and case start
2. `featureVectorEmitter.emit(caseId, tenantId, vector)` — stores the vector with the pre-generated caseId
3. `caseHub.startCase(caseId, caseType, initialContext)` — starts the case with the same ID

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

1. **Scan** — `CaseMemoryStore.scan(MemoryScanRequest)` with `tenantId`, `domain="software-review"`, `attributeKey="entity-type"`, `attributeValue="case-vector"`, paginated via `afterMemoryId`. Server-side attribute filtering returns only case-vector facts.
2. **Filter** — Client-side: `memory.attributes().get("pr-repo")` matches target `repo`. `memory.createdAt()` within time window (configurable, default 180 days).
3. **Score** — `PrFeatureVector.fromAttributes(memory.attributes())` for each stored vector. `SimilarityMetric.compute(query, stored)`.
4. **Rank** — sort by score descending, filter below minimum threshold, take top K.
5. **Enrich** — for each top-K result, query contributor outcome facts using the same `MemoryQuery` pattern as `CaseMemoryRecaller`:
   - Extract `contributor` from the stored `PrFeatureVector`
   - `MemoryQuery.forEntity("contributor:" + contributor, SOFTWARE_REVIEW, tenantId).withCaseId(caseId.toString())` — returns facts for that contributor in that case (one per capability review, following the `CaseMemoryEmitter` storage pattern)
   - Group returned facts by `DevtownMemoryKeys.CAPABILITY` attribute
   - Each capability's outcome from `MemoryAttributeKeys.OUTCOME` attribute
   - Aggregate case outcome: all approved → `"approved"`, any failed → `"failed"`, otherwise → `"flagged"`
   - If no outcome facts found for a caseId (case still in progress), exclude from results

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

`MemoryContext.toContextMap()` extended to include precedents:

```java
"precedents", precedents.stream().map(p -> Map.<String, Object>of(
    "caseId", p.caseId().toString(),
    "similarity", p.similarity().score(),
    "breakdown", p.similarity().breakdown(),
    "outcome", p.outcome(),
    "capabilityOutcomes", p.capabilityOutcomes()
)).toList()
```

Return type remains `Map<String, Object>` — nested maps/lists consistent with existing contributor/codeArea entries.

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
- Extraction from primitive parameters — correct modules, languages, hasTests, touchedConfigs
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
- Empty sets: Jaccard returns 1.0 (both lack the feature → identical on that dimension)

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

### Migration to `CbrCaseMemoryStore`

**Prerequisite:** Platform issue required in `casehubio/neocortex` for `FeatureField.SetValued` with Jaccard similarity — a single sealed-interface case + `CbrSimilarityScorer` switch branch. A devtown tracking issue must be filed to track this dependency before Phase 1 implementation begins.

When neocortex gains `FeatureField.SetValued` (with Jaccard as default similarity), devtown migrates:

1. **Storage:** `FeatureVectorEmitter` stores via `CbrCaseMemoryStore.store(FeatureVectorCbrCase)` instead of `CaseMemoryStore`. Schema registered with `CbrFeatureSchema` defining set-valued fields for `changedPaths`, `modules`, `languages`.
2. **Retrieval:** `DefaultCbrRetrievalService` calls `CbrCaseMemoryStore.retrieveSimilar(CbrQuery)` instead of manual scan+score. `CbrQuery` carries weights, topK, minSimilarity, notBefore — same parameters currently resolved from preferences.
3. **Scoring:** `WeightedJaccardSimilarity` logic moves into `LocalSimilarityFunction` overrides passed via `CbrQuery`. Per-dimension breakdown requires `ScoredCbrCase` extension or post-hoc recomputation.
4. **Production backend:** `QdrantCbrCaseMemoryStore` provides indexed retrieval, replacing the O(n) scan.

This migration is mechanical — the domain model (`PrFeatureVector`, `SimilarityScore`, `Precedent`) is unaffected. Only the storage and retrieval adapters in `app/` change.
