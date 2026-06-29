# Adaptive Batch Sizing Feedback Loop

**Issue:** devtown#103
**Date:** 2026-06-29
**Status:** Approved

## Problem

`BatchFormationContext.recentFailureRate` is hardcoded to `0.0` in `MergeQueueService.formBatchesTransactional()` (line 316). The adaptive batch sizing formula in `DefaultBatchCompositionPolicy` already uses this value:

```
adaptiveMax = max(minBatchSize, floor(trustMax × (1.0 - recentFailureRate)))
```

But it has no effect because the rate is always zero. The `FAILURE_RATE_WINDOW` preference key exists (default 20) but is unused.

The root cause: `ActiveBatchEntity` records are deleted on batch completion (`store.deleteBatch()`), so batch outcome history is lost. There is no data source to compute the failure rate from.

## Design

### Approach: Evolve batch records to track full lifecycle

Instead of deleting batch records on completion, mark them completed with an outcome. This retains audit history and provides the data source for failure rate computation. Retention/expunge policies can be added later.

### Schema (Flyway `V101__batch_outcome_columns.sql`)

Path: `app/src/main/resources/db/devtown/migration/V101__batch_outcome_columns.sql`

V101 is collision-free across all Flyway-scanned locations (`db/qhorus/migration`, `db/ledger/migration`, `db/engine-ledger/migration`, `db/devtown/migration`). V100 is the only existing devtown domain migration; V2002–V2003 are ledger subclass joins in the V2000+ range.

```sql
ALTER TABLE merge_queue_batch ADD COLUMN completed_at TIMESTAMP;
ALTER TABLE merge_queue_batch ADD COLUMN succeeded BOOLEAN;
CREATE INDEX idx_merge_queue_batch_repo_completed
    ON merge_queue_batch(repository, completed_at);
```

Existing active batch rows keep `completed_at = NULL, succeeded = NULL`.

### Entity rename: `ActiveBatchEntity` → `BatchEntity`

Same `@Table(name = "merge_queue_batch")`. Updated `@Table.indexes` to include the new composite index (mirrors V101 migration, consistent with V100 pattern):

```java
@Table(name = "merge_queue_batch", indexes = {
    @Index(name = "idx_merge_queue_batch_case_id", columnList = "case_id"),
    @Index(name = "idx_merge_queue_batch_repo_completed", columnList = "repository, completed_at")
})
```

Two new nullable fields:

```java
@Column(name = "completed_at")
public Instant completedAt;

@Column(name = "succeeded")
public Boolean succeeded;
```

`Boolean` (boxed) because active batches have no outcome.

### Domain record: `BatchRecord`

```java
public record BatchRecord(
    String batchId,
    UUID caseId,
    List<Integer> prNumbers,
    String repository,
    Instant dispatchedAt,
    Instant completedAt,
    Boolean succeeded
) {
    public boolean isActive() { return completedAt == null; }
}
```

### SPI changes (`MergeQueueStore`)

Replace:
```java
void deleteBatch(String batchId);
```

With:
```java
void completeBatch(String batchId, boolean succeeded);
double recentBatchFailureRate(String repository, int window);
```

- `completeBatch`: sets `completedAt = now()` and `succeeded`. Idempotent — already-completed batches are a no-op.
- `recentBatchFailureRate`: returns `failedCount / totalCount` over the last `window` completed batches for the repository. Returns `0.0` if no completed batches exist (cold start matches current behavior).
- `activeBatches()`: semantics unchanged — now filters to `completedAt IS NULL`.
- `findBatchByCaseId`: returns both active and completed batches.

### JPA implementation (`JpaMergeQueueStore`)

`completeBatch`:
```java
@Transactional
public void completeBatch(String batchId, boolean succeeded) {
    BatchEntity entity = em.find(BatchEntity.class, batchId);
    if (entity == null || entity.completedAt != null) return;
    entity.completedAt = Instant.now();
    entity.succeeded = succeeded;
    em.merge(entity);
}
```

`recentBatchFailureRate` — JPQL (avoids H2 native query UUID gotcha GE-20260629-500611):
```java
public double recentBatchFailureRate(String repository, int window) {
    List<BatchEntity> recent = em.createQuery(
        "SELECT b FROM BatchEntity b WHERE b.repository = :repo " +
        "AND b.completedAt IS NOT NULL ORDER BY b.completedAt DESC",
        BatchEntity.class)
        .setParameter("repo", repository)
        .setMaxResults(window)
        .getResultList();

    if (recent.isEmpty()) return 0.0;
    long failed = recent.stream().filter(b -> Boolean.FALSE.equals(b.succeeded)).count();
    return (double) failed / recent.size();
}
```

`activeBatches`:
```java
public Map<String, BatchRecord> activeBatches() {
    List<BatchEntity> entities = em.createQuery(
        "SELECT b FROM BatchEntity b WHERE b.completedAt IS NULL",
        BatchEntity.class)
        .getResultList();

    Map<String, BatchRecord> result = new HashMap<>();
    for (BatchEntity entity : entities) {
        result.put(entity.batchId, toBatchRecord(entity));
    }
    return result;
}
```

`toBatchRecord` — updated for new fields:
```java
private BatchRecord toBatchRecord(BatchEntity entity) {
    return new BatchRecord(
        entity.batchId,
        entity.caseId,
        deserializePrNumbers(entity.prNumbers),
        entity.repository,
        entity.dispatchedAt,
        entity.completedAt,
        entity.succeeded
    );
}
```

### Service changes (`MergeQueueService`)

**`handleBatchCompletion`:**

1. Idempotency guard:
```java
BatchRecord batch = batchOpt.get();
if (!batch.isActive()) {
    LOG.debugf("Batch %s already completed — idempotent no-op", batch.batchId());
    return;
}
```

2. Replace `store.deleteBatch(batch.batchId())` with `store.completeBatch(batch.batchId(), batchSucceeded)`.

**`formBatchesTransactional`:**

Replace hardcoded `0.0` (line 316) with:
```java
int failureRateWindow = prefs.getOrDefault(MergeQueuePreferenceKeys.FAILURE_RATE_WINDOW).value();
double recentFailureRate = store.recentBatchFailureRate(repository, failureRateWindow);
```

Inside the per-repository loop — failure rate is per-repository.

### Bisection and failure rate semantics

The failure rate measures root batch outcomes only. When bisection triggers, the root batch is marked `succeeded=false`. Sub-cases created by bisection have no batch records (`findBatchByCaseId` returns empty for sub-case lifecycle events) and are not counted.

This is intentionally conservative. Bisection is a recovery mechanism, not a normal operating mode — each bisection run requires log₂(batchSize) additional CI cycles. The formula's purpose is to find the batch size that avoids triggering bisection, not just to avoid unrecoverable failures. The system self-corrects: smaller batches have lower failure rates, which raises `adaptiveMax` back toward `maxBatchSize`. The equilibrium batch size balances throughput against bisection avoidance.

## Tests

**Unit (merge module — `BatchRecordTest`):**
- `isActive()`: null completedAt → true, non-null → false

**Integration (app module — `JpaMergeQueueStoreTest`):**
- `completeBatch` sets fields correctly
- `completeBatch` idempotent — second call preserves original values
- `activeBatches` excludes completed batches
- `findBatchByCaseId` returns completed batches
- `recentBatchFailureRate` returns 0.0 with no history
- `recentBatchFailureRate` computes correctly (e.g., 2/5 = 0.4)
- `recentBatchFailureRate` respects window size
- `recentBatchFailureRate` is per-repository

**Mechanical test updates (constructor breakage — compiler-caught):**
- `BatchRecordTest`: update 2 existing constructor calls from 5-arg to 7-arg (add `null, null` for active batches)
- `DevtownMcpToolsTest`: update `BatchRecord` constructor calls in `getMergeQueue_withQueuedPrsAndBatches_returnsCorrectState`, `getBatchStatus_knownBatch_returnsDetail`, and metrics tests (add `null, null`)

**Mechanical test updates (JPQL entity name — runtime-caught):**
- `MergeQueuePersistenceTest` (line 50): `@BeforeEach` cleanup `"DELETE FROM ActiveBatchEntity"` → `"DELETE FROM BatchEntity"`
- `MergeQueueMultiRepoBatchTest` (line 50): `@BeforeEach` cleanup `"DELETE FROM ActiveBatchEntity"` → `"DELETE FROM BatchEntity"`
- `MergeQueueIdempotencyTest` (line 41): `@BeforeEach` cleanup `"DELETE FROM ActiveBatchEntity"` → `"DELETE FROM BatchEntity"`

**Integration (app module — existing test updates):**
- `MergeQueueBatchLifecycleTest`: batch completion assertions replace deletion assertions; idempotency test
- End-to-end wiring: failed batch → smaller next batch via reduced `adaptiveMax`

## Not in scope

- Retention/expunge policies for completed batch records (devtown#107)
- Dashboard visibility of failure rate trends (devtown#108)
- Alerting on sustained high failure rates (devtown#109)
