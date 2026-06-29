# Review Fixes + MCP Enhancements Design

**Issues:** #106, #116, #102
**Date:** 2026-06-29
**Branch:** issue-106-review-fixes-mcp-enhancements

---

## 1. UUID v5 for Deterministic subjectId (#106 M1)

### Problem

`MergeDecisionObserver.handleMergeBatchPath()` (line 193) uses `UUID.nameUUIDFromBytes()` — Java's built-in UUID v3 (MD5). The comment says "UUID v5". Java stdlib has no UUID v5.

### Design

Create `DeterministicUuid` in `devtown-domain` (pure Java, no Quarkus dependency).

```java
package io.casehub.devtown.domain;

public final class DeterministicUuid {

    // v5 of DNS namespace + "casehub.io/devtown/merge-decision"
    public static final UUID MERGE_DECISION_NS = v5(
        UUID.fromString("6ba7b810-9dad-11d1-80b4-00c04fd430c8"),
        "casehub.io/devtown/merge-decision");

    public static UUID v5(UUID namespace, String name) { ... }
}
```

Implementation: SHA-1 hash of `namespace bytes || name bytes`, then set version=5 and variant=IETF per RFC 4122.

**Call site change** in `MergeDecisionObserver`:
```
// Before:
UUID subjectId = UUID.nameUUIDFromBytes(nameInput.getBytes(UTF_8));

// After:
UUID subjectId = DeterministicUuid.v5(
    DeterministicUuid.MERGE_DECISION_NS,
    event.caseId() + ":" + prNumber);
```

**Backward compatibility:** Not a concern. `subjectId` is generated fresh per entry. Existing entries keep their v3 UUIDs. No lookups cross the boundary — `findLatestBySubjectId` is keyed by `event.caseId()` (the case UUID, not the derived subjectId) in the PR review path.

---

## 2. Stale Binding Name (#106 M2)

### Problem

`ReviewOutcomeObserverTest.java:114` references `"merge"` as a binding name. The actual binding is `merge-direct` (pr-review.yaml:360). Renamed when `enqueue-for-merge` was added.

### Design

Change `"merge"` to `"merge-direct"` at line 114. The test behavior is unaffected — neither value is in `PLAN_ITEM_TO_CONTEXT_KEY`, so the observer filters both out as infrastructure bindings.

---

## 3. Structural `dequeue()` Fix + Test Isolation (#106 M3)

### Problem

`MergeQueueService.dequeue()` has a TOCTOU race. It reads queue state via `store.queued()` (returns QUEUED entries only) in one call, then writes via `store.dequeue()` in a separate `@Transactional` call. With `MIN_BATCH_SIZE=1` (the default), `enqueue()` immediately batches the entry (QUEUED → IN_BATCH), so `store.queued()` returns nothing and the WorkItem is never obsoleted.

**Evidence chain:**
1. `MIN_BATCH_SIZE` default = 1 (`MergeQueuePreferenceKeys:19`)
2. `enqueue()` is not `@Transactional` — each internal call runs in its own tx
3. `formBatchesTransactional()` finds the QUEUED entry, `1 < 1` is false → batches it
4. `DefaultBatchCompositionPolicy`: adaptiveMax=8 for trustScore=0.80, 1 < 8 → remaining entries flushed → one batch
5. Entry marked IN_BATCH, transaction commits
6. `dequeue()`: `store.queued()` skips IN_BATCH entries → target=null → obsoleteFromSystem never called

### Design

**A) Change `MergeQueueStore.dequeue()` return type.**

```java
// Before:
boolean dequeue(int prNumber, String repository);

// After:
Optional<QueueEntry> dequeue(int prNumber, String repository);
```

The JPA implementation already has the entity in hand when checking status. Return the full `QueueEntry` (including `workItemId`) instead of discarding it. The caller (`MergeQueueService.dequeue()`) gets the WorkItem ID from the same transaction that performed the status change.

**`JpaMergeQueueStore` change:**
```java
@Transactional
public Optional<QueueEntry> dequeue(int prNumber, String repository) {
    QueuedPrEntity entity = em.find(QueuedPrEntity.class,
        new QueuedPrEntity.QueuedPrId(prNumber, repository));
    if (entity == null || !"QUEUED".equals(entity.status)) {
        return Optional.empty();
    }
    entity.status = QueueEntryStatus.DEQUEUED.name();
    em.merge(entity);
    return Optional.of(toQueueEntry(entity));
}
```

**`MergeQueueService.dequeue()` change:**
```java
public boolean dequeue(int prNumber, String repository) {
    Optional<QueueEntry> dequeued = store.dequeue(prNumber, repository);
    if (dequeued.isEmpty()) return false;

    UUID workItemId = dequeued.get().workItemId();
    if (workItemId != null && workItemServiceInstance.isResolvable()) {
        try {
            workItemServiceInstance.get().obsoleteFromSystem(
                workItemId, "system", "dequeued");
        } catch (Exception e) {
            LOG.warnf(e, "Failed to obsolete WorkItem %s for dequeued PR #%d",
                workItemId, prNumber);
        }
    }
    return true;
}
```

This eliminates the separate `store.queued()` lookup entirely.

**B) Test isolation.**

The test tests enqueue/dequeue lifecycle, not batch formation. Batch formation has its own unit tests (`DefaultBatchCompositionPolicyTest`). Mixing the two is what makes it flaky.

Add to `app/src/test/resources/application.properties`:
```
casehub.platform.preferences.defaults."devtown.merge-queue.min-batch-size"=100
```

`ConfigFilePreferenceProvider` merges `casehub.platform.preferences.defaults` into resolved preferences with highest priority. `PreferenceKey.qualifiedName()` for `MIN_BATCH_SIZE` is `"devtown.merge-queue.min-batch-size"` (verified: namespace=`devtown.merge-queue`, name=`min-batch-size`). With `MIN_BATCH_SIZE=100`, `formBatchesTransactional()` skips batch formation for groups smaller than 100, so the entry stays QUEUED and `dequeue()` works correctly.

No existing test depends on batch formation via `MergeQueueService.enqueue()` — verified by reading `MergeBatchCaseHubTest` (definition structure only), `DefaultBatchCompositionPolicyTest` (unit test, no CDI), `JpaMergeQueueStoreTest` (store-level only).

---

## 4. Surface Enqueue Idempotency (#116)

### Problem

Three callers discard or ignore the boolean return from `MergeQueueService.enqueue()`:

1. `DevtownMcpTools.enqueuePr()` — always returns `"ENQUEUED"`
2. `MergeQueueService.enqueue()` — LOG at line 117 fires unconditionally
3. `PrReviewMergeQueueAdapter.enqueue()` — discards boolean, always returns `"enqueued"`

### Design

**Site 1 — `MergeQueueService.enqueue()` LOG:**
```java
// Before (line 117-118):
LOG.infof("Enqueued PR #%d from %s (trust=%.2f, lane=%s)", ...);
if (inserted) { formAndDispatchBatches(); }

// After:
if (inserted) {
    LOG.infof("Enqueued PR #%d from %s (trust=%.2f, lane=%s)", ...);
    formAndDispatchBatches();
} else {
    LOG.debugf("Duplicate enqueue for PR #%d from %s — no-op",
        pr.number(), pr.repository());
}
```

**Site 2 — `DevtownMcpTools.enqueuePr()`:**
```java
// Before:
mergeQueueService.enqueue(pr);
return new EnqueueResult(prNumber, lane.name(), "ENQUEUED");

// After:
boolean inserted = mergeQueueService.enqueue(pr);
return new EnqueueResult(prNumber, lane.name(),
    inserted ? "ENQUEUED" : "ALREADY_QUEUED");
```

**Site 3 — `PrReviewMergeQueueAdapter.enqueue()`:**
```java
// Before:
mergeQueueService.enqueue(queuedPr);
return WorkerResult.of(Map.of("status", "enqueued", ...));

// After:
boolean inserted = mergeQueueService.enqueue(queuedPr);
return WorkerResult.of(Map.of(
    "status", inserted ? "enqueued" : "already-queued",
    "prNumber", prNumber, "repository", repository));
```

---

## 5. MCP Tool Enhancements (#102)

### 5.1 Fix `list_problems` SLA Breach Detection

**Problem:** Uses `devtown.queue.sla-minutes` ConfigProperty (flat 120 min) while `enqueue()` uses per-lane SLA from `PreferenceProvider`. CRITICAL breaches detected 60 minutes late. HIGH and NORMAL produce false positives.

**Design:**
- Remove the `devtown.queue.sla-minutes` ConfigProperty from `DevtownMcpTools`
- Inject `PreferenceProvider` into `DevtownMcpTools`
- In `listProblems()`, resolve per-PR SLA:

```java
Preferences prefs = preferenceProvider.resolve(
    SettingsScope.of("casehubio", "devtown", "merge-queue"));

for (QueuedPr pr : mergeQueueService.queuedPrs()) {
    String slaDuration = prefs.getOrDefault(
        MergeQueuePreferenceKeys.slaKeyFor(pr.lane())).value();
    Duration sla = Duration.parse(slaDuration);
    Duration waited = Duration.between(pr.enqueuedAt(), now);
    if (waited.compareTo(sla) > 0) {
        problems.add(new Problem(
            "queue_sla_breach", "warning",
            String.format("PR #%d (%s) has waited %d min — exceeds %s SLA (%d min)",
                pr.number(), pr.lane(), waited.toMinutes(),
                pr.lane(), sla.toMinutes()),
            null, pr.author(), pr.enqueuedAt()));
    }
}
```

### 5.2 Expand `get_merge_queue_metrics`

**New store method:**

```java
// MergeQueueStore:
List<BatchRecord> completedBatchesSince(Instant since);

// Aggregate failure rate (no repository filter):
double recentBatchFailureRate(int window);
```

**New service method:**

```java
// MergeQueueService:
List<BatchRecord> completedBatches(Duration window);
```

**Expanded `MergeQueueMetrics` record:**

```java
public record MergeQueueMetrics(
    int queueDepth,
    int activeBatches,
    long oldestWaitMinutes,
    long avgWaitMinutes,            // NEW — mean wait time across queued PRs
    double avgTrustScore,
    Map<String, Integer> countsByLane,
    int throughput24h,              // NEW — completed batches in last 24h
    double failureRate,             // NEW — aggregate recent failure rate
    Map<Integer, Integer> batchSizeDistribution  // NEW — batch size → count
) {}
```

**`getMergeQueueMetrics()` implementation additions:**

```java
// Average wait
long avgWait = queued.isEmpty() ? 0 :
    (long) queued.stream()
        .mapToLong(pr -> Duration.between(pr.enqueuedAt(), now).toMinutes())
        .average().orElse(0);

// Throughput (24h)
List<BatchRecord> completed = mergeQueueService.completedBatches(Duration.ofHours(24));
int throughput24h = completed.size();

// Failure rate
Preferences prefs = preferenceProvider.resolve(
    SettingsScope.of("casehubio", "devtown", "merge-queue"));
int window = prefs.getOrDefault(MergeQueuePreferenceKeys.FAILURE_RATE_WINDOW).value();
double failureRate = mergeQueueService.aggregateFailureRate(window);

// Batch size distribution
Map<Integer, Integer> batchSizeDist = new HashMap<>();
for (BatchRecord batch : completed) {
    batchSizeDist.merge(batch.prNumbers().size(), 1, Integer::sum);
}
```

**JPA implementation for `completedBatchesSince`:**
```java
public List<BatchRecord> completedBatchesSince(Instant since) {
    return em.createQuery(
        "SELECT b FROM BatchEntity b WHERE b.completedAt IS NOT NULL " +
        "AND b.completedAt >= :since ORDER BY b.completedAt DESC",
        BatchEntity.class)
        .setParameter("since", since)
        .getResultList()
        .stream()
        .map(this::toBatchRecord)
        .toList();
}
```

---

## 6. Out of Scope

- GDPR erasure spec also uses `UUID.nameUUIDFromBytes()` — not live code, spec only. Will file an issue to track updating it when the erasure endpoint is implemented.
- `get_recent_events` merge queue event types (spec §8.2 line 565) — separate concern from metrics, not in #102.
