# Review Fixes + MCP Enhancements Design

**Issues:** #106, #116, #102
**Date:** 2026-06-29
**Branch:** issue-106-review-fixes-mcp-enhancements

---

## 1. UUID v5 for Deterministic subjectId (#106 M1)

### Problem

`MergeDecisionObserver.handleMergeBatchPath()` (line 193) uses `UUID.nameUUIDFromBytes()` — Java's built-in UUID v3 (MD5). The comment says "UUID v5". Java stdlib has no UUID v5.

### Design

Create `DeterministicUuid` in `devtown-domain` (pure Java, no Quarkus dependency). The `v5()` algorithm is RFC 4122 standard and could live in `casehub-platform-api`; it stays in `devtown-domain` because devtown is the only current consumer. If another CaseHub module needs UUID v5, promote the algorithm to the platform and keep only the namespace constant in `devtown-domain`.

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

Two issues with the current `MergeQueueService.dequeue()`:

**A) Test failure: deterministic state sequencing.** With `MIN_BATCH_SIZE=1` (the default), `enqueue()` synchronously calls `formAndDispatchBatches()` → `formBatchesTransactional()`, which moves the entry from QUEUED to IN_BATCH in the same thread before any external caller can invoke `dequeue()`. This is not a race — it is deterministic state sequencing that causes the test to fail because the entry is never in QUEUED state when `dequeue()` runs.

**Evidence chain:**
1. `MIN_BATCH_SIZE` default = 1 (`MergeQueuePreferenceKeys:19`)
2. `enqueue()` is not `@Transactional` — each internal call runs in its own tx
3. `formBatchesTransactional()` finds the QUEUED entry, `1 < 1` is false → batches it
4. `DefaultBatchCompositionPolicy`: adaptiveMax=8 for trustScore=0.80, 1 < 8 → remaining entries flushed → one batch
5. Entry marked IN_BATCH, transaction commits
6. `dequeue()`: `store.queued()` skips IN_BATCH entries → target=null → obsoleteFromSystem never called

**B) Structural concern: `dequeue()` uses two separate operations.** The current `dequeue()` reads via `store.queued()` in one call to find the WorkItem ID, then writes via `store.dequeue()` in a separate `@Transactional` call. This two-step lookup-then-mutate pattern is fragile — the store already has the entity in hand during `dequeue()` and can return the full entry directly.

**IN_BATCH dequeue semantics:** `dequeue()` for IN_BATCH entries returns false by design. An entry that has been assigned to an active batch cannot be individually dequeued — that would break the batch's PR set. To remove a PR from an active batch, the batch must be completed or cancelled first.

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

**Test call sites affected:** The return type change from `boolean` to `Optional<QueueEntry>` breaks 5 existing test call sites:

| Test class | Pattern | Migration |
|---|---|---|
| `MergeQueuePersistenceTest` | `boolean removed = store.dequeue(...)` (×2) | `assertThat(store.dequeue(...)).isPresent()` / `.isEmpty()` |
| `JpaMergeQueueStoreTest` | `boolean dequeued = store.dequeue(...)` (×2) | `assertThat(store.dequeue(...)).isPresent()` / `.isEmpty()` |
| `MergeQueueIdempotencyTest` | `store.dequeue(...)` (discards return) | No change needed |

**B) Test isolation.**

The test tests enqueue/dequeue lifecycle, not batch formation. Batch formation has its own unit tests (`DefaultBatchCompositionPolicyTest`). Mixing the two is what makes it flaky.

Add to `app/src/test/resources/application.properties`:
```
casehub.platform.preferences.defaults."devtown.merge-queue.min-batch-size"=100
```

`ConfigFilePreferenceProvider` merges `casehub.platform.preferences.defaults` into resolved preferences with highest priority. `PreferenceKey.qualifiedName()` for `MIN_BATCH_SIZE` is `"devtown.merge-queue.min-batch-size"` (verified: namespace=`devtown.merge-queue`, name=`min-batch-size`). With `MIN_BATCH_SIZE=100`, `formBatchesTransactional()` skips batch formation for groups smaller than 100, so the entry stays QUEUED and `dequeue()` works correctly.

No existing test depends on batch formation via `MergeQueueService.enqueue()` — verified exhaustively:

| Test class | Why unaffected |
|---|---|
| `MergeBatchCaseHubTest` | Tests case definition structure only, never calls `enqueue()` |
| `DefaultBatchCompositionPolicyTest` | Unit test — no CDI, no preference resolution |
| `JpaMergeQueueStoreTest` | Store-level only — calls `store.enqueue()` not `service.enqueue()` |
| `MergeQueuePersistenceTest` | Store-level only — calls `store.enqueue()` not `service.enqueue()` |
| `MergeQueueIdempotencyTest` | Store-level only — calls `store.enqueue()` not `service.enqueue()` |
| `ReviewOutcomeObserverTest` | Tests observer, not merge queue |
| `DevtownMcpToolsTest` | Uses mocked `MergeQueueService` |

The global override is intentional: test isolation from batch formation is a default that any test can override via `@TestProfile` if it specifically needs `MIN_BATCH_SIZE=1` behavior.

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
- Add `detectSlaBreaches()` to `MergeQueueService` — the service already owns the preference scope, so SLA detection belongs there, not in the MCP presentation layer
- `DevtownMcpTools.listProblems()` calls the service method; consolidate `now2` into the existing `now`

**New `MergeQueueService` method:**

```java
public record SlaBreach(QueuedPr pr, Duration waited, Duration sla) {}

public List<SlaBreach> detectSlaBreaches() {
    Preferences prefs = resolvePreferences();
    Instant now = Instant.now();
    List<SlaBreach> breaches = new ArrayList<>();
    for (QueueEntry entry : store.queued()) {
        QueuedPr pr = entry.pr();
        String slaDuration = prefs.getOrDefault(
            MergeQueuePreferenceKeys.slaKeyFor(pr.lane())).value();
        Duration sla = Duration.parse(slaDuration);
        Duration waited = Duration.between(pr.enqueuedAt(), now);
        if (waited.compareTo(sla) > 0) {
            breaches.add(new SlaBreach(pr, waited, sla));
        }
    }
    return breaches;
}
```

**`DevtownMcpTools.listProblems()` change:**

```java
// Before (flat SLA, uses now2):
Instant now2 = Instant.now();
for (QueuedPr pr : mergeQueueService.queuedPrs()) {
    long waitMinutes = Duration.between(pr.enqueuedAt(), now2).toMinutes();
    if (waitMinutes > queueSlaMinutes) { ... }
}

// After (delegates to service, removes now2):
for (var breach : mergeQueueService.detectSlaBreaches()) {
    problems.add(new Problem(
        "queue_sla_breach", "warning",
        String.format("PR #%d (%s) has waited %d min — exceeds %s SLA (%d min)",
            breach.pr().number(), breach.pr().lane(), breach.waited().toMinutes(),
            breach.pr().lane(), breach.sla().toMinutes()),
        null, breach.pr().author(), breach.pr().enqueuedAt()));
}
```

### 5.2 Expand `get_merge_queue_metrics`

**New store methods on `MergeQueueStore`:**

```java
List<BatchRecord> completedBatchesSince(Instant since);

/**
 * Aggregate failure rate across all repositories.
 *
 * <p>Overload of the existing per-repository method:
 * {@code recentBatchFailureRate(String repository, int window)}.
 * The existing method filters by {@code b.repository = :repo};
 * this overload omits the repository filter for cross-repo aggregate metrics.
 */
double recentBatchFailureRate(int window);
```

**New service methods on `MergeQueueService`:**

```java
public List<BatchRecord> completedBatches(Duration window) {
    return store.completedBatchesSince(Instant.now().minus(window));
}

public double aggregateFailureRate() {
    Preferences prefs = resolvePreferences();
    int window = prefs.getOrDefault(MergeQueuePreferenceKeys.FAILURE_RATE_WINDOW).value();
    return store.recentBatchFailureRate(window);
}
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

// Failure rate — delegates to service (owns preference resolution)
double failureRate = mergeQueueService.aggregateFailureRate();

// Batch size distribution
Map<Integer, Integer> batchSizeDist = new HashMap<>();
for (BatchRecord batch : completed) {
    batchSizeDist.merge(batch.prNumbers().size(), 1, Integer::sum);
}
```

**JPA implementations in `JpaMergeQueueStore`:**

```java
// completedBatchesSince — no result limit; batch throughput is inherently
// bounded by the merge queue's sequential-per-repo processing model
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

// Aggregate overload — same as existing recentBatchFailureRate(String, int)
// but without the repository filter
public double recentBatchFailureRate(int window) {
    List<BatchEntity> recent = em.createQuery(
        "SELECT b FROM BatchEntity b WHERE b.completedAt IS NOT NULL " +
        "ORDER BY b.completedAt DESC",
        BatchEntity.class)
        .setMaxResults(window)
        .getResultList();

    if (recent.isEmpty()) return 0.0;
    long failed = recent.stream().filter(b -> Boolean.FALSE.equals(b.succeeded)).count();
    return (double) failed / recent.size();
}
```

---

## 6. Out of Scope

- GDPR erasure spec also uses `UUID.nameUUIDFromBytes()` — not live code, spec only. Filed as [devtown#1](https://github.com/mdproctor/wsp-casehub-devtown/issues/1).
- `get_recent_events` merge queue event types — separate concern from metrics, not in #102. Filed as [devtown#2](https://github.com/mdproctor/wsp-casehub-devtown/issues/2).

---

## 7. Test Plan

### §1 DeterministicUuid

| Test | Verifies |
|---|---|
| `v5_deterministic` | Same namespace+name → same UUID |
| `v5_namespace_isolation` | Different namespace → different UUID for same name |
| `v5_version_bits` | UUID version nibble = 5 |
| `v5_variant_bits` | Variant bits = IETF (10xx) |
| `v5_known_vector` | Match RFC 4122 Appendix B test vector for DNS namespace |
| `MERGE_DECISION_NS_stable` | Namespace constant is itself a valid v5 UUID derived from DNS namespace |

### §2 Stale binding name

Existing test — change `"merge"` → `"merge-direct"` at `ReviewOutcomeObserverTest:114`. No new test needed; the test already verifies the observer filters infrastructure bindings.

### §3 dequeue() return type + test isolation

| Test | Verifies |
|---|---|
| Update `MergeQueuePersistenceTest` | `store.dequeue()` returns `Optional<QueueEntry>` when QUEUED; `.isEmpty()` when absent |
| Update `JpaMergeQueueStoreTest` | Same as above at store level |
| `dequeue_inBatch_returnsEmpty` | IN_BATCH entry → `Optional.empty()` (by-design behavior) |
| `MergeQueueService_dequeue_obsoletesWorkItem` | Service-level: dequeue returns true and obsoletes the WorkItem from the returned entry |

### §4 Enqueue idempotency

| Test | Verifies |
|---|---|
| `enqueuePr_returnsAlreadyQueued` | `DevtownMcpTools.enqueuePr()` returns `ALREADY_QUEUED` on duplicate |
| `enqueue_log_conditional` | INFO log fires only on insert; DEBUG fires on duplicate |
| `adapter_returnsAlreadyQueued` | `PrReviewMergeQueueAdapter.enqueue()` returns `"already-queued"` status on duplicate |

### §5.1 SLA breach detection

| Test | Verifies |
|---|---|
| `detectSlaBreaches_perLane` | CRITICAL flagged at 61 min, NORMAL not flagged at 121 min |
| `detectSlaBreaches_empty` | No queued PRs → empty list |
| `listProblems_includesSlaBreaches` | MCP tool delegates to service and formats correctly |

### §5.2 Expanded metrics

| Test | Verifies |
|---|---|
| `completedBatchesSince_filtersCorrectly` | Only batches completed after the `since` instant are returned |
| `recentBatchFailureRate_aggregate` | Aggregate overload computes across all repositories |
| `recentBatchFailureRate_aggregate_respectsWindow` | Window limits the number of batches considered |
| `aggregateFailureRate_delegatesToStore` | Service method resolves preferences and delegates |
| `getMergeQueueMetrics_expanded` | All four new fields populated correctly |
