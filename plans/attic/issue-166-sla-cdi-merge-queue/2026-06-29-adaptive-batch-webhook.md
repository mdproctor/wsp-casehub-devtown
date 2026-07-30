# Adaptive Batch Sizing + Webhook Merge Queue Admission

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire adaptive batch sizing from batch outcome history (#103) and add webhook-based merge queue admission (#101).

**Architecture:** Task 1 evolves batch records from ephemeral to lifecycle-tracked — the SPI break (`deleteBatch` → `completeBatch`) forces all callers to record outcomes. Task 2 adds the third merge queue admission path via a hexagonal port in `merge/`, implemented by `MergeQueueService`, consumed by `GitHubWebhookResource`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (test), Flyway, AssertJ, JUnit 5

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Flyway: V101 is the next devtown domain migration (V100 is the only existing one; V2000+ is ledger subclass range)
- Entity `@Table.indexes` must mirror Flyway indexes (V100 pattern)
- JPQL not native SQL for new queries (avoids H2 UUID gotcha GE-20260629-500611)
- `Boolean.FALSE.equals()` for nullable Boolean comparison (never auto-unbox)
- IntelliJ MCP for all code navigation — never bash grep/find for classes
- All commits reference issues: `Refs #103` or `Refs #101`

---

### Task 1: Batch Lifecycle Evolution (#103)

**Files:**
- Modify: `merge/src/main/java/io/casehub/devtown/merge/BatchRecord.java`
- Modify: `merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java`
- Modify: `merge/src/test/java/io/casehub/devtown/merge/BatchRecordTest.java`
- Create: `app/src/main/resources/db/devtown/migration/V101__batch_outcome_columns.sql`
- Modify: `app/src/main/java/io/casehub/devtown/app/persistence/ActiveBatchEntity.java` → rename to `BatchEntity.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/persistence/JpaMergeQueueStore.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/MergeQueuePersistenceTest.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/MergeQueueMultiRepoBatchTest.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/MergeQueueIdempotencyTest.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/MergeQueueBatchLifecycleTest.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java`

**Interfaces:**
- Produces: `BatchRecord(String, UUID, List<Integer>, String, Instant, Instant, Boolean)` — 7-arg constructor
- Produces: `BatchRecord.isActive()` — returns `completedAt == null`
- Produces: `MergeQueueStore.completeBatch(String batchId, boolean succeeded)` — replaces `deleteBatch`
- Produces: `MergeQueueStore.recentBatchFailureRate(String repository, int window)` — returns double [0.0, 1.0]
- Produces: `MergeQueueStore.enqueue(QueuedPr, UUID)` — returns `boolean` (true = inserted, false = duplicate no-op)
- Produces: `MergeQueueService.enqueue(QueuedPr)` — returns `boolean` (propagates store result)

#### Step 1: Write BatchRecord unit tests

- [ ] In `merge/src/test/java/io/casehub/devtown/merge/BatchRecordTest.java`, add tests for `isActive()` and update existing constructor calls to 7-arg:

```java
@Test
void isActive_returnsTrue_whenCompletedAtNull() {
    BatchRecord record = new BatchRecord(
        "batch-1", UUID.randomUUID(), List.of(1), "repo", Instant.now(), null, null);
    assertThat(record.isActive()).isTrue();
}

@Test
void isActive_returnsFalse_whenCompletedAtSet() {
    BatchRecord record = new BatchRecord(
        "batch-1", UUID.randomUUID(), List.of(1), "repo", Instant.now(), Instant.now(), true);
    assertThat(record.isActive()).isFalse();
}
```

Update the two existing tests: add `, null, null` to the 5-arg constructors (lines 21 and 38).

- [ ] Run tests to verify they fail (BatchRecord has 5-arg constructor, no `isActive()`).

```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl merge -Dtest=BatchRecordTest -f pom.xml
```

Expected: FAIL — constructor mismatch.

#### Step 2: Evolve BatchRecord

- [ ] In `merge/src/main/java/io/casehub/devtown/merge/BatchRecord.java`:

```java
package io.casehub.devtown.merge;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

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

- [ ] Run BatchRecordTest — expect PASS.

#### Step 3: Update MergeQueueStore SPI

- [ ] In `merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java`:

Replace `deleteBatch`:
```java
// REMOVE:
void deleteBatch(String batchId);

// ADD:
void completeBatch(String batchId, boolean succeeded);
double recentBatchFailureRate(String repository, int window);
```

Change `enqueue` return type from `void` to `boolean`:
```java
// BEFORE:
void enqueue(QueuedPr pr, UUID workItemId);

// AFTER:
boolean enqueue(QueuedPr pr, UUID workItemId);
```

Update the Javadoc for `enqueue` to document the boolean return:
```java
/**
 * Enqueue a PR with its associated SLA WorkItem ID.
 *
 * <p>Idempotent: if an entry for {@code (pr.number(), pr.repository())} already exists
 * in state {@code QUEUED} or {@code IN_BATCH}, this is a no-op and returns {@code false}.
 *
 * @param pr the PR to enqueue
 * @param workItemId the SLA WorkItem tracking queue wait time
 * @return true if a new entry was created; false if already queued (no-op)
 */
boolean enqueue(QueuedPr pr, UUID workItemId);
```

- [ ] Verify merge module compiles (it will — SPI is an interface, no implementation here):

```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl merge -f pom.xml
```

#### Step 4: Flyway migration

- [ ] Create `app/src/main/resources/db/devtown/migration/V101__batch_outcome_columns.sql`:

```sql
ALTER TABLE merge_queue_batch ADD COLUMN completed_at TIMESTAMP;
ALTER TABLE merge_queue_batch ADD COLUMN succeeded BOOLEAN;
CREATE INDEX idx_merge_queue_batch_repo_completed
    ON merge_queue_batch(repository, completed_at);
```

#### Step 5: Rename entity ActiveBatchEntity → BatchEntity

- [ ] Use IntelliJ MCP `ide_refactor_rename` to rename the class:

```
ide_refactor_rename file=app/src/main/java/io/casehub/devtown/app/persistence/ActiveBatchEntity.java
    line=20 column=14 newName=BatchEntity
```

This renames the class and updates all Java references (JpaMergeQueueStore, imports, JPQL strings in the same file). Verify the `@Table(name = "merge_queue_batch")` annotation is unchanged.

- [ ] Add the two new fields and the new index annotation to `BatchEntity.java`:

```java
@Entity
@Table(name = "merge_queue_batch", indexes = {
    @Index(name = "idx_merge_queue_batch_case_id", columnList = "case_id"),
    @Index(name = "idx_merge_queue_batch_repo_completed", columnList = "repository, completed_at")
})
public class BatchEntity {
    // ... existing fields ...

    @Column(name = "completed_at")
    public Instant completedAt;

    @Column(name = "succeeded")
    public Boolean succeeded;
}
```

- [ ] Fix JPQL entity name references in test cleanup methods (runtime-caught, not compiler-caught):

  - `MergeQueuePersistenceTest.java` line 50: `"DELETE FROM ActiveBatchEntity"` → `"DELETE FROM BatchEntity"`
  - `MergeQueueMultiRepoBatchTest.java` line 50: `"DELETE FROM ActiveBatchEntity"` → `"DELETE FROM BatchEntity"`
  - `MergeQueueIdempotencyTest.java` line 41: `"DELETE FROM ActiveBatchEntity"` → `"DELETE FROM BatchEntity"`

Use IntelliJ `ide_search_text` with query `ActiveBatchEntity` to verify no references remain.

#### Step 6: Write JpaMergeQueueStore tests

- [ ] In `app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java`, add these tests:

```java
@Test
@Transactional
void completeBatch_setsOutcome() {
    UUID caseId = UUID.randomUUID();
    store.recordBatch("batch-complete-1", caseId, List.of(200), "casehubio/devtown");

    store.completeBatch("batch-complete-1", true);

    Optional<BatchRecord> found = store.findBatchByCaseId(caseId);
    assertThat(found).isPresent();
    assertThat(found.get().completedAt()).isNotNull();
    assertThat(found.get().succeeded()).isTrue();
    assertThat(found.get().isActive()).isFalse();
}

@Test
@Transactional
void completeBatch_idempotent_preservesOriginalValues() {
    UUID caseId = UUID.randomUUID();
    store.recordBatch("batch-complete-2", caseId, List.of(201), "casehubio/devtown");
    store.completeBatch("batch-complete-2", true);

    Instant firstCompletedAt = store.findBatchByCaseId(caseId).get().completedAt();

    store.completeBatch("batch-complete-2", false);

    BatchRecord after = store.findBatchByCaseId(caseId).get();
    assertThat(after.completedAt()).isEqualTo(firstCompletedAt);
    assertThat(after.succeeded()).isTrue();
}

@Test
@Transactional
void activeBatches_excludesCompleted() {
    store.recordBatch("batch-active", UUID.randomUUID(), List.of(202), "casehubio/devtown");
    store.recordBatch("batch-done", UUID.randomUUID(), List.of(203), "casehubio/devtown");
    store.completeBatch("batch-done", true);

    Map<String, BatchRecord> active = store.activeBatches();
    assertThat(active).hasSize(1);
    assertThat(active).containsKey("batch-active");
}

@Test
@Transactional
void findBatchByCaseId_returnsCompletedBatch() {
    UUID caseId = UUID.randomUUID();
    store.recordBatch("batch-find-completed", caseId, List.of(204), "casehubio/devtown");
    store.completeBatch("batch-find-completed", false);

    Optional<BatchRecord> found = store.findBatchByCaseId(caseId);
    assertThat(found).isPresent();
    assertThat(found.get().succeeded()).isFalse();
}

@Test
@Transactional
void recentBatchFailureRate_returnsZero_withNoHistory() {
    double rate = store.recentBatchFailureRate("casehubio/devtown", 20);
    assertThat(rate).isEqualTo(0.0);
}

@Test
@Transactional
void recentBatchFailureRate_computesCorrectly() {
    for (int i = 0; i < 5; i++) {
        store.recordBatch("batch-rate-" + i, UUID.randomUUID(), List.of(300 + i), "casehubio/devtown");
        store.completeBatch("batch-rate-" + i, i < 3);
    }

    double rate = store.recentBatchFailureRate("casehubio/devtown", 20);
    assertThat(rate).isCloseTo(0.4, org.assertj.core.data.Offset.offset(0.001));
}

@Test
@Transactional
void recentBatchFailureRate_respectsWindow() {
    for (int i = 0; i < 10; i++) {
        store.recordBatch("batch-window-" + i, UUID.randomUUID(), List.of(400 + i), "casehubio/devtown");
        store.completeBatch("batch-window-" + i, i < 8);
    }

    double rateAll = store.recentBatchFailureRate("casehubio/devtown", 20);
    double rateWindow5 = store.recentBatchFailureRate("casehubio/devtown", 5);
    assertThat(rateAll).isCloseTo(0.2, org.assertj.core.data.Offset.offset(0.001));
    assertThat(rateWindow5).isCloseTo(0.4, org.assertj.core.data.Offset.offset(0.001));
}

@Test
@Transactional
void recentBatchFailureRate_isPerRepository() {
    store.recordBatch("batch-repoA", UUID.randomUUID(), List.of(500), "casehubio/devtown");
    store.completeBatch("batch-repoA", false);

    store.recordBatch("batch-repoB", UUID.randomUUID(), List.of(501), "casehubio/engine");
    store.completeBatch("batch-repoB", true);

    assertThat(store.recentBatchFailureRate("casehubio/devtown", 20)).isEqualTo(1.0);
    assertThat(store.recentBatchFailureRate("casehubio/engine", 20)).isEqualTo(0.0);
}

@Test
@Transactional
void enqueue_returnsTrue_whenNewEntry() {
    QueuedPr pr = new QueuedPr(600, "casehubio/devtown", "abc", "author", 0.5,
        PriorityLane.NORMAL, Instant.now(), Set.of());
    boolean result = store.enqueue(pr, UUID.randomUUID());
    assertThat(result).isTrue();
}

@Test
@Transactional
void enqueue_returnsFalse_whenDuplicate() {
    QueuedPr pr = new QueuedPr(601, "casehubio/devtown", "abc", "author", 0.5,
        PriorityLane.NORMAL, Instant.now(), Set.of());
    store.enqueue(pr, UUID.randomUUID());
    boolean result = store.enqueue(pr, UUID.randomUUID());
    assertThat(result).isFalse();
}
```

- [ ] Run tests — expect FAIL (methods not yet implemented).

#### Step 7: Implement JpaMergeQueueStore changes

- [ ] In `JpaMergeQueueStore.java`:

**Change `enqueue` return type** from `void` to `boolean`. Return `false` in the existing idempotency early-return (line 49), return `true` after the persist/merge at the end.

**Remove `deleteBatch` method.** Replace with `completeBatch`:

```java
@Override
@Transactional
public void completeBatch(String batchId, boolean succeeded) {
    BatchEntity entity = em.find(BatchEntity.class, batchId);
    if (entity == null || entity.completedAt != null) return;
    entity.completedAt = Instant.now();
    entity.succeeded = succeeded;
    em.merge(entity);
}
```

**Add `recentBatchFailureRate`:**

```java
@Override
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

**Update `activeBatches`:**

```java
@Override
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

**Update `toBatchRecord`:**

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

- [ ] Run `JpaMergeQueueStoreTest` — expect all new and existing tests PASS.

#### Step 8: Update MergeQueueService

- [ ] In `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`:

**Change `enqueue(QueuedPr)` return type** from `void` to `boolean`. Propagate the store result:
```java
// Line ~97: change from store.enqueue(pr, workItemId); to:
boolean inserted = store.enqueue(pr, workItemId);
// ... existing logging ...
if (inserted) {
    formAndDispatchBatches();
}
return inserted;
```

**Update `handleBatchCompletion`:**

After `BatchRecord batch = batchOpt.get();` (line 185), add idempotency guard:
```java
if (!batch.isActive()) {
    LOG.debugf("Batch %s already completed — idempotent no-op", batch.batchId());
    return;
}
```

Replace `store.deleteBatch(batch.batchId())` (currently last line before final log) with:
```java
store.completeBatch(batch.batchId(), batchSucceeded);
```

**Wire failure rate in `formBatchesTransactional`:**

Inside the per-repository loop (after line 286 where `bisectionStrategy` is read), add:
```java
int failureRateWindow = prefs.getOrDefault(MergeQueuePreferenceKeys.FAILURE_RATE_WINDOW).value();
```

Replace `0.0, // recentFailureRate — wired in devtown#103` (line 316) with:
```java
store.recentBatchFailureRate(repository, failureRateWindow),
```

- [ ] Fix mechanical constructor breakage in `DevtownMcpToolsTest.java` — find all `BatchRecord` constructor calls and add `, null, null` for `completedAt` and `succeeded`.

Use IntelliJ `ide_find_references` on `BatchRecord` constructor to find all call sites in test files.

- [ ] Run full build to verify everything compiles and all tests pass:

```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml
```

- [ ] Commit:

```
feat(#103): adaptive batch sizing — batch lifecycle evolution + failure rate wiring

Evolve batch records from ephemeral (deleted on completion) to lifecycle-tracked.
Wire recentBatchFailureRate into batch formation via FAILURE_RATE_WINDOW preference.

- BatchRecord gains completedAt/succeeded fields + isActive()
- MergeQueueStore: deleteBatch → completeBatch, add recentBatchFailureRate
- MergeQueueStore.enqueue returns boolean (idempotency signal)
- Flyway V101 adds completed_at/succeeded columns + composite index
- ActiveBatchEntity → BatchEntity rename
- MergeQueueService: idempotent handleBatchCompletion, failure rate wiring

Refs #103
```

---

### Task 2: Webhook Merge Queue Admission (#101)

**Files:**
- Create: `merge/src/main/java/io/casehub/devtown/merge/MergeQueueAdmissionPort.java`
- Create: `merge/src/main/java/io/casehub/devtown/merge/AdmissionResult.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`
- Modify: `github/pom.xml`
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubPullRequestEvent.java`
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java`
- Modify: `github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java`
- Modify: `github/src/test/java/io/casehub/devtown/github/GitHubPullRequestEventTest.java`

**Interfaces:**
- Consumes: `MergeQueueService.enqueue(QueuedPr)` returns `boolean` (from Task 1)
- Produces: `MergeQueueAdmissionPort.admit(int, String, String, String)` → `AdmissionResult`
- Produces: `AdmissionResult.ENQUEUED`, `AdmissionResult.ALREADY_QUEUED`
- Produces: `GitHubPullRequestEvent.label()` → nullable `Label`
- Produces: `GitHubWebhookResource.handleLabeled(GitHubPullRequestEvent)` → `Response`

#### Step 1: Create port interface and result enum

- [ ] Create `merge/src/main/java/io/casehub/devtown/merge/AdmissionResult.java`:

```java
package io.casehub.devtown.merge;

public enum AdmissionResult {
    ENQUEUED,
    ALREADY_QUEUED
}
```

- [ ] Create `merge/src/main/java/io/casehub/devtown/merge/MergeQueueAdmissionPort.java`:

```java
package io.casehub.devtown.merge;

public interface MergeQueueAdmissionPort {
    AdmissionResult admit(int prNumber, String repository, String headSha, String author);
}
```

- [ ] Verify merge module compiles:

```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl merge -f pom.xml
```

#### Step 2: Write webhook handler tests

- [ ] In `github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java`:

Add a `RecordingAdmissionPort` inner class alongside `RecordingService`:

```java
static class RecordingAdmissionPort implements MergeQueueAdmissionPort {
    int lastPrNumber;
    String lastRepo;
    String lastHeadSha;
    String lastAuthor;
    boolean called;
    AdmissionResult result = AdmissionResult.ENQUEUED;

    @Override
    public AdmissionResult admit(int prNumber, String repository, String headSha, String author) {
        called = true;
        lastPrNumber = prNumber;
        lastRepo = repository;
        lastHeadSha = headSha;
        lastAuthor = author;
        return result;
    }
}
```

Add field and wire in `setUp()`:
```java
private RecordingAdmissionPort admissionPort;

@BeforeEach
void setUp() {
    service = new RecordingService();
    admissionPort = new RecordingAdmissionPort();
    resource = new GitHubWebhookResource();
    resource.service = service;
    resource.admissionPort = admissionPort;
    resource.webhookSecret = SECRET;
}
```

Add test helper for labeled events:
```java
private String labeledEvent(String labelName, boolean draft) {
    String labelField = labelName != null
        ? ",\"label\":{\"name\":\"%s\"}".formatted(labelName)
        : "";
    return "{\"action\":\"labeled\",\"number\":42,\"pull_request\":{\"head\":{\"sha\":\"abc\"},\"base\":{\"ref\":\"main\"},\"user\":{\"login\":\"octocat\"},\"draft\":%s,\"merged\":false,\"additions\":10,\"deletions\":5,\"changed_files\":1},\"repository\":{\"full_name\":\"casehubio/devtown\"}%s}".formatted(draft, labelField);
}
```

Add tests:
```java
@Test
void labeled_mergeReady_callsAdmit() {
    String body = labeledEvent("merge-ready", false);
    var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(responseBody(response)).containsEntry("action", "merge-queue-enqueued");
    assertThat(admissionPort.called).isTrue();
    assertThat(admissionPort.lastPrNumber).isEqualTo(42);
    assertThat(admissionPort.lastRepo).isEqualTo("casehubio/devtown");
    assertThat(admissionPort.lastHeadSha).isEqualTo("abc");
    assertThat(admissionPort.lastAuthor).isEqualTo("octocat");
}

@Test
void labeled_mergeReady_alreadyQueued_returnsAlreadyQueued() {
    admissionPort.result = AdmissionResult.ALREADY_QUEUED;
    String body = labeledEvent("merge-ready", false);
    var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(responseBody(response)).containsEntry("action", "already-queued");
}

@Test
void labeled_otherLabel_doesNotCallAdmit() {
    String body = labeledEvent("bug", false);
    var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(responseBody(response)).containsEntry("status", "ignored");
    assertThat(responseBody(response)).containsEntry("reason", "not-merge-ready");
    assertThat(admissionPort.called).isFalse();
}

@Test
void labeled_nullLabel_doesNotCallAdmit() {
    String body = labeledEvent(null, false);
    var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(responseBody(response)).containsEntry("status", "ignored");
    assertThat(admissionPort.called).isFalse();
}

@Test
void labeled_draft_doesNotCallAdmit() {
    String body = labeledEvent("merge-ready", true);
    var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(responseBody(response)).containsEntry("reason", "draft");
    assertThat(admissionPort.called).isFalse();
}
```

Update the existing `unknownAction_returns200Ignored` test to use a truly unknown action (since `labeled` is now handled):
```java
@Test
void unknownAction_returns200Ignored() {
    String body = prEvent("review_requested", false, false);
    var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
    assertThat(response.getStatus()).isEqualTo(200);
    assertThat(responseBody(response)).containsEntry("status", "ignored");
}
```

- [ ] Run tests — expect FAIL (no `admissionPort` field, no `handleLabeled`, no `Label` record).

#### Step 3: Add Label to GitHubPullRequestEvent

- [ ] In `github/src/main/java/io/casehub/devtown/github/GitHubPullRequestEvent.java`:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubPullRequestEvent(
    String action,
    int number,
    PullRequest pull_request,
    Repository repository,
    Label label
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record PullRequest(
        Head head, Base base, User user,
        boolean draft, boolean merged,
        int additions, int deletions, int changed_files
    ) {
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Head(String sha) {}
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Base(String ref) {}
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record User(String login) {}
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Repository(String full_name) {}
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Label(String name) {}
}
```

- [ ] In `github/src/test/java/io/casehub/devtown/github/GitHubPullRequestEventTest.java`, add:

```java
@Test
void parsesLabeledEvent() throws Exception {
    String json = "{\"action\":\"labeled\",\"number\":42,\"label\":{\"name\":\"merge-ready\"},\"pull_request\":{\"head\":{\"sha\":\"abc\"},\"base\":{\"ref\":\"main\"},\"user\":{\"login\":\"octocat\"},\"draft\":false,\"merged\":false,\"additions\":10,\"deletions\":5,\"changed_files\":1},\"repository\":{\"full_name\":\"casehubio/devtown\"}}";
    var event = new com.fasterxml.jackson.databind.ObjectMapper().readValue(json, GitHubPullRequestEvent.class);
    assertThat(event.label()).isNotNull();
    assertThat(event.label().name()).isEqualTo("merge-ready");
}

@Test
void parsesOpenedEvent_labelIsNull() throws Exception {
    String json = "{\"action\":\"opened\",\"number\":42,\"pull_request\":{\"head\":{\"sha\":\"abc\"},\"base\":{\"ref\":\"main\"},\"user\":{\"login\":\"octocat\"},\"draft\":false,\"merged\":false,\"additions\":10,\"deletions\":5,\"changed_files\":1},\"repository\":{\"full_name\":\"casehubio/devtown\"}}";
    var event = new com.fasterxml.jackson.databind.ObjectMapper().readValue(json, GitHubPullRequestEvent.class);
    assertThat(event.label()).isNull();
}
```

#### Step 4: Add dependency and implement webhook handler

- [ ] In `github/pom.xml`, add dependency on `devtown-merge`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-devtown-merge</artifactId>
</dependency>
```

- [ ] In `github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java`:

Add import and field:
```java
import io.casehub.devtown.merge.MergeQueueAdmissionPort;

@Inject
MergeQueueAdmissionPort admissionPort;
```

Add `labeled` case to the `handlePullRequest` switch:
```java
case "labeled"          -> handleLabeled(event);
```

Add handler method:
```java
private Response handleLabeled(GitHubPullRequestEvent event) {
    if (event.label() == null || !"merge-ready".equals(event.label().name())) {
        return ok(Map.of("status", "ignored", "action", "labeled",
                         "reason", "not-merge-ready"));
    }
    if (event.pull_request().draft()) {
        return ok(Map.of("status", "ignored", "reason", "draft"));
    }

    var result = admissionPort.admit(
        event.number(),
        event.repository().full_name(),
        event.pull_request().head().sha(),
        event.pull_request().user().login()
    );

    String action = switch (result) {
        case ENQUEUED -> "merge-queue-enqueued";
        case ALREADY_QUEUED -> "already-queued";
    };
    return ok(Map.of("status", "accepted", "action", action));
}
```

#### Step 5: Implement MergeQueueService.admit()

- [ ] In `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`:

Add `implements MergeQueueAdmissionPort` to class declaration:
```java
public class MergeQueueService implements MergeQueueAdmissionPort {
```

Add imports:
```java
import io.casehub.devtown.merge.AdmissionResult;
import io.casehub.devtown.merge.MergeQueueAdmissionPort;
```

Add method:
```java
@Override
public AdmissionResult admit(int prNumber, String repository, String headSha, String author) {
    QueuedPr pr = new QueuedPr(prNumber, repository, headSha, author,
        0.5, PriorityLane.NORMAL, Instant.now(), Set.of());
    boolean inserted = enqueue(pr);
    return inserted ? AdmissionResult.ENQUEUED : AdmissionResult.ALREADY_QUEUED;
}
```

- [ ] Run `GitHubWebhookResourceTest` — expect all tests PASS.

- [ ] Run `GitHubPullRequestEventTest` — expect PASS.

- [ ] Run full build:

```
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml
```

- [ ] Commit:

```
feat(#101): webhook merge queue admission — labeled event → enqueue

Add MergeQueueAdmissionPort hexagonal port in merge/, implemented by
MergeQueueService. GitHubWebhookResource handles pull_request.labeled
with merge-ready label as the third merge queue admission path.

- MergeQueueAdmissionPort + AdmissionResult in merge/
- MergeQueueService implements admit() with neutral defaults
- GitHubPullRequestEvent gains Label record
- GitHubWebhookResource.handleLabeled routes merge-ready labels
- github/ adds devtown-merge dependency

Refs #101
```
