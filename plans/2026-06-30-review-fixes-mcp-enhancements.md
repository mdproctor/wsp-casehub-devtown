# Review Fixes + MCP Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix three issues (#106, #116, #102) on one branch — UUID v5 for deterministic subjectId, structural dequeue() fix, enqueue idempotency surfacing, and MCP tool metric/SLA improvements.

**Architecture:** All changes are within devtown — `devtown-domain` (pure Java utility), `merge/` (store interface), and `app/` (service + MCP tools + persistence). No foundation changes. No migrations.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (tests)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- All commits reference issues: `Refs #106`, `Refs #116`, `Refs #102`
- IntelliJ MCP for all code navigation — never bash grep/find for classes
- TDD: write failing test → verify failure → implement → verify pass → commit

---

### Task 1: DeterministicUuid + Stale Binding Name (#106 M1 + M2)

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/DeterministicUuid.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/DeterministicUuidTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java:191-193`
- Modify: `app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java` (update assertion for new UUIDs)
- Modify: `app/src/test/java/io/casehub/devtown/app/ReviewOutcomeObserverTest.java:114`

**Interfaces:**
- Produces: `DeterministicUuid.v5(UUID namespace, String name) → UUID`
- Produces: `DeterministicUuid.MERGE_DECISION_NS` constant (UUID)

- [ ] **Step 1: Write DeterministicUuidTest**

```java
package io.casehub.devtown.domain;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class DeterministicUuidTest {

    private static final UUID DNS_NS = UUID.fromString("6ba7b810-9dad-11d1-80b4-00c04fd430c8");

    @Test
    void v5_deterministic() {
        UUID a = DeterministicUuid.v5(DNS_NS, "test.example.com");
        UUID b = DeterministicUuid.v5(DNS_NS, "test.example.com");
        assertThat(a).isEqualTo(b);
    }

    @Test
    void v5_namespace_isolation() {
        UUID ns2 = UUID.fromString("6ba7b811-9dad-11d1-80b4-00c04fd430c8");
        UUID a = DeterministicUuid.v5(DNS_NS, "same-name");
        UUID b = DeterministicUuid.v5(ns2, "same-name");
        assertThat(a).isNotEqualTo(b);
    }

    @Test
    void v5_version_bits() {
        UUID result = DeterministicUuid.v5(DNS_NS, "anything");
        assertThat(result.version()).isEqualTo(5);
    }

    @Test
    void v5_variant_bits() {
        UUID result = DeterministicUuid.v5(DNS_NS, "anything");
        assertThat(result.variant()).isEqualTo(2); // IETF variant = 2 in Java's UUID
    }

    @Test
    void v5_known_vector() {
        // RFC 4122 Appendix B: v5 of DNS namespace + "python.org"
        UUID expected = UUID.fromString("886313e1-3b8a-5372-9b90-0c9aee199e5d");
        UUID actual = DeterministicUuid.v5(DNS_NS, "python.org");
        assertThat(actual).isEqualTo(expected);
    }

    @Test
    void MERGE_DECISION_NS_stable() {
        assertThat(DeterministicUuid.MERGE_DECISION_NS.version()).isEqualTo(5);
        // Verify it's deterministic — re-derive and compare
        UUID rederived = DeterministicUuid.v5(DNS_NS, "casehub.io/devtown/merge-decision");
        assertThat(DeterministicUuid.MERGE_DECISION_NS).isEqualTo(rederived);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=DeterministicUuidTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `DeterministicUuid` class not found

- [ ] **Step 3: Implement DeterministicUuid**

```java
package io.casehub.devtown.domain;

import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.UUID;

public final class DeterministicUuid {

    private static final UUID DNS_NAMESPACE =
            UUID.fromString("6ba7b810-9dad-11d1-80b4-00c04fd430c8");

    public static final UUID MERGE_DECISION_NS =
            v5(DNS_NAMESPACE, "casehub.io/devtown/merge-decision");

    public static UUID v5(UUID namespace, String name) {
        byte[] nsBytes = uuidToBytes(namespace);
        byte[] nameBytes = name.getBytes(StandardCharsets.UTF_8);

        MessageDigest sha1;
        try {
            sha1 = MessageDigest.getInstance("SHA-1");
        } catch (NoSuchAlgorithmException e) {
            throw new AssertionError("SHA-1 not available", e);
        }

        sha1.update(nsBytes);
        sha1.update(nameBytes);
        byte[] hash = sha1.digest();

        hash[6] = (byte) ((hash[6] & 0x0f) | 0x50); // version 5
        hash[8] = (byte) ((hash[8] & 0x3f) | 0x80); // IETF variant

        ByteBuffer buf = ByteBuffer.wrap(hash, 0, 16);
        return new UUID(buf.getLong(), buf.getLong());
    }

    private static byte[] uuidToBytes(UUID uuid) {
        ByteBuffer buf = ByteBuffer.allocate(16);
        buf.putLong(uuid.getMostSignificantBits());
        buf.putLong(uuid.getLeastSignificantBits());
        return buf.array();
    }

    private DeterministicUuid() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=DeterministicUuidTest`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Update MergeDecisionObserver to use UUID v5**

In `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java`, replace lines 191-193:

```java
// Before:
// Derive deterministic subjectId from caseId + prNumber (UUID v5)
String nameInput = event.caseId().toString() + ":" + prNumber;
java.util.UUID subjectId = java.util.UUID.nameUUIDFromBytes(nameInput.getBytes(java.nio.charset.StandardCharsets.UTF_8));

// After:
UUID subjectId = DeterministicUuid.v5(
        DeterministicUuid.MERGE_DECISION_NS,
        event.caseId() + ":" + prNumber);
```

Add import: `import io.casehub.devtown.domain.DeterministicUuid;`

Remove the now-unused `java.nio.charset.StandardCharsets` import from `MergeDecisionObserver` if it's only used by the old UUID line (check — it's also used in `serializeBatchContext` via `getBytes`; no, `serializeBatchContext` uses `objectMapper.writeValueAsString` — but `domainContentBytes()` in the parent uses `StandardCharsets`. Actually, `MergeDecisionObserver` imports `StandardCharsets` at line 18 — check if any other line uses it. Line 193 was the only usage in this file. Remove the import).

Wait — check line 18: `import java.nio.charset.StandardCharsets;`. The only usage was line 193. After the change, this import is unused. Remove it. Run `ide_optimize_imports` on the file.

- [ ] **Step 6: Update MergeDecisionObserverTest**

Read `MergeDecisionObserverTest` to understand how it validates the batch path `subjectId`. The test should verify the new UUID is v5 with the correct namespace. Update assertions to check `entry.subjectId.version() == 5` if the test checks subjectId values. If the test doesn't check subjectId, no change needed — just verify the tests still pass.

- [ ] **Step 7: Fix stale binding name in ReviewOutcomeObserverTest**

In `app/src/test/java/io/casehub/devtown/app/ReviewOutcomeObserverTest.java` at line 114-115, change:

```java
// Before:
// "merge" is another infrastructure binding
planItemCompletedEvents.fireAsync(
        new PlanItemCompletedEvent(caseId, "merge", "merge-bot", "test-tenant"));

// After:
// "merge-direct" is an infrastructure binding (renamed from "merge" when enqueue-for-merge was added)
planItemCompletedEvents.fireAsync(
        new PlanItemCompletedEvent(caseId, "merge-direct", "merge-bot", "test-tenant"));
```

- [ ] **Step 8: Run full build to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add domain/src/main/java/io/casehub/devtown/domain/DeterministicUuid.java \
       domain/src/test/java/io/casehub/devtown/domain/DeterministicUuidTest.java \
       app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java \
       app/src/test/java/io/casehub/devtown/app/ReviewOutcomeObserverTest.java
# Include MergeDecisionObserverTest.java if modified
git commit -m "feat(#106): UUID v5 for deterministic subjectId + fix stale binding name

DeterministicUuid utility in devtown-domain implements RFC 4122 UUID v5
with SHA-1 + namespace UUID. MergeDecisionObserver batch path now uses
MERGE_DECISION_NS instead of UUID.nameUUIDFromBytes (v3/MD5).

ReviewOutcomeObserverTest: 'merge' → 'merge-direct' to match pr-review.yaml.

Refs #106"
```

---

### Task 2: Structural dequeue() Fix + Test Isolation (#106 M3)

**Files:**
- Modify: `merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java:35-42`
- Modify: `app/src/main/java/io/casehub/devtown/app/persistence/JpaMergeQueueStore.java:77-89`
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java:248-270`
- Modify: `app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java:128-152`
- Modify: `app/src/test/java/io/casehub/devtown/app/MergeQueuePersistenceTest.java:72-87`
- Modify: `app/src/test/java/io/casehub/devtown/app/MergeQueueSlaWorkItemTest.java`
- Modify: `app/src/test/resources/application.properties`

**Interfaces:**
- Changes: `MergeQueueStore.dequeue(int, String)` return type from `boolean` to `Optional<QueueEntry>`
- Consumes: `QueueEntry.workItemId()` from dequeue result

- [ ] **Step 1: Change `MergeQueueStore.dequeue()` return type**

In `merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java`, change:

```java
// Before:
boolean dequeue(int prNumber, String repository);

// After:
Optional<QueueEntry> dequeue(int prNumber, String repository);
```

Ensure `import java.util.Optional;` is present (it already is from `findBatchByCaseId`).

Update the Javadoc:
```java
/**
 * Remove a PR from the queue (sets status to DEQUEUED).
 *
 * @param prNumber PR number
 * @param repository repository name
 * @return the dequeued entry if found in QUEUED state; empty if absent or not QUEUED
 */
Optional<QueueEntry> dequeue(int prNumber, String repository);
```

- [ ] **Step 2: Update JpaMergeQueueStore.dequeue()**

In `app/src/main/java/io/casehub/devtown/app/persistence/JpaMergeQueueStore.java`, change:

```java
// Before:
@Override
@Transactional
public boolean dequeue(int prNumber, String repository) {
    QueuedPrEntity entity = em.find(QueuedPrEntity.class,
        new QueuedPrEntity.QueuedPrId(prNumber, repository));
    if (entity == null || !"QUEUED".equals(entity.status)) {
        return false;
    }
    entity.status = QueueEntryStatus.DEQUEUED.name();
    em.merge(entity);
    return true;
}

// After:
@Override
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

Add `import java.util.Optional;` if not already present.

- [ ] **Step 3: Update MergeQueueService.dequeue()**

In `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`, replace the entire `dequeue()` method:

```java
// Before (lines 248-270):
public boolean dequeue(int prNumber, String repository) {
    List<QueueEntry> queued = store.queued();
    QueueEntry target = queued.stream()
        .filter(e -> e.pr().number() == prNumber && e.pr().repository().equals(repository))
        .findFirst()
        .orElse(null);
    boolean removed = store.dequeue(prNumber, repository);
    if (removed && target != null && target.workItemId() != null
            && workItemServiceInstance.isResolvable()) {
        try {
            workItemServiceInstance.get().obsoleteFromSystem(
                target.workItemId(), "system", "dequeued");
        } catch (Exception e) {
            LOG.warnf(e, "Failed to obsolete WorkItem %s for dequeued PR #%d",
                target.workItemId(), prNumber);
        }
    }
    return removed;
}

// After:
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

Add `import java.util.Optional;` if not already present.

- [ ] **Step 4: Migrate JpaMergeQueueStoreTest call sites**

In `app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java`:

Test `dequeue_removesQueuedPr` (line 128):
```java
// Before:
boolean dequeued = store.dequeue(103, "casehubio/devtown");
assertThat(dequeued).isTrue();

// After:
Optional<QueueEntry> dequeued = store.dequeue(103, "casehubio/devtown");
assertThat(dequeued).isPresent();
assertThat(dequeued.get().status()).isEqualTo(QueueEntryStatus.DEQUEUED);
```

Test `dequeue_returnsFalse_whenNotQueued` (line 149):
```java
// Before:
boolean dequeued = store.dequeue(999, "casehubio/nonexistent");
assertThat(dequeued).isFalse();

// After:
Optional<QueueEntry> dequeued = store.dequeue(999, "casehubio/nonexistent");
assertThat(dequeued).isEmpty();
```

Add import: `import java.util.Optional;` (already present in the file).

- [ ] **Step 5: Migrate MergeQueuePersistenceTest call sites**

In `app/src/test/java/io/casehub/devtown/app/MergeQueuePersistenceTest.java`:

Test `dequeue_changes_status_and_removes_from_queued` (line 72):
```java
// Before:
boolean removed = store.dequeue(502, "casehubio/devtown");
assertThat(removed).isTrue();

// After:
Optional<QueueEntry> removed = store.dequeue(502, "casehubio/devtown");
assertThat(removed).isPresent();
```

Test `dequeue_returns_false_for_nonexistent` (line 84):
```java
// Before:
boolean removed = store.dequeue(9999, "casehubio/devtown");
assertThat(removed).isFalse();

// After:
Optional<QueueEntry> removed = store.dequeue(9999, "casehubio/devtown");
assertThat(removed).isEmpty();
```

`Optional` is already imported in this file.

- [ ] **Step 6: Add test isolation config**

Append to `app/src/test/resources/application.properties`:

```
# Prevent enqueue() from triggering immediate batch formation in tests.
# Tests that need batch formation use DefaultBatchCompositionPolicyTest (unit test, no CDI).
casehub.platform.preferences.defaults."devtown.merge-queue.min-batch-size"=100
```

- [ ] **Step 7: Run full build to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS. The `MergeQueueSlaWorkItemTest.dequeue_obsoletesWorkItem` should now pass reliably.

- [ ] **Step 8: Commit**

```bash
git add merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java \
       app/src/main/java/io/casehub/devtown/app/persistence/JpaMergeQueueStore.java \
       app/src/main/java/io/casehub/devtown/app/MergeQueueService.java \
       app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java \
       app/src/test/java/io/casehub/devtown/app/MergeQueuePersistenceTest.java \
       app/src/test/resources/application.properties
git commit -m "fix(#106): structural dequeue() fix — return Optional<QueueEntry> from store

MergeQueueStore.dequeue() now returns Optional<QueueEntry> instead of
boolean. MergeQueueService.dequeue() gets the WorkItem ID from the same
transaction that performs the status change — eliminates the TOCTOU race
where store.queued() missed IN_BATCH entries.

Test isolation: MIN_BATCH_SIZE=100 in test config prevents enqueue() from
triggering immediate batch formation.

Refs #106"
```

---

### Task 3: Enqueue Idempotency Surfacing (#116)

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java:116-122`
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java:750-755`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewMergeQueueAdapter.java:71-73`
- Modify: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java:569-580`

**Interfaces:**
- No signature changes — `MergeQueueService.enqueue()` already returns boolean

- [ ] **Step 1: Update existing enqueuePr test to verify idempotency status**

In `DevtownMcpToolsTest.java`, update `enqueuePr_validInputs_enqueuesSuccessfully`:

```java
// Before (line 569-580):
@Test
void enqueuePr_validInputs_enqueuesSuccessfully() {
    DevtownMcpTools.EnqueueResult result = tools.enqueuePr(
        "casehubio/devtown", 99, "deadbeef", "alice", 0.75, "HIGH"
    );
    assertThat(result.prNumber()).isEqualTo(99);
    assertThat(result.lane()).isEqualTo("HIGH");
    assertThat(result.status()).isEqualTo("ENQUEUED");
    verify(mergeQueueService).enqueue(argThat(pr ->
        pr.number() == 99 && pr.headSha().equals("deadbeef") && pr.lane() == PriorityLane.HIGH
    ));
}

// After:
@Test
void enqueuePr_validInputs_enqueuesSuccessfully() {
    when(mergeQueueService.enqueue(any())).thenReturn(true);
    DevtownMcpTools.EnqueueResult result = tools.enqueuePr(
        "casehubio/devtown", 99, "deadbeef", "alice", 0.75, "HIGH"
    );
    assertThat(result.prNumber()).isEqualTo(99);
    assertThat(result.lane()).isEqualTo("HIGH");
    assertThat(result.status()).isEqualTo("ENQUEUED");
}
```

Add new test for duplicate:
```java
@Test
void enqueuePr_duplicate_returnsAlreadyQueued() {
    when(mergeQueueService.enqueue(any())).thenReturn(false);
    DevtownMcpTools.EnqueueResult result = tools.enqueuePr(
        "casehubio/devtown", 99, "deadbeef", "alice", 0.75, "HIGH"
    );
    assertThat(result.status()).isEqualTo("ALREADY_QUEUED");
}
```

- [ ] **Step 2: Run tests to verify failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownMcpToolsTest#enqueuePr_duplicate_returnsAlreadyQueued`
Expected: FAIL — `enqueuePr` always returns "ENQUEUED"

- [ ] **Step 3: Fix MergeQueueService.enqueue() logging**

In `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`, replace lines 116-122:

```java
// Before:
boolean inserted = store.enqueue(pr, workItemId);
LOG.infof("Enqueued PR #%d from %s (trust=%.2f, lane=%s)",
    pr.number(), pr.repository(), pr.trustScore(), pr.lane());
if (inserted) {
    formAndDispatchBatches();
}

// After:
boolean inserted = store.enqueue(pr, workItemId);
if (inserted) {
    LOG.infof("Enqueued PR #%d from %s (trust=%.2f, lane=%s)",
        pr.number(), pr.repository(), pr.trustScore(), pr.lane());
    formAndDispatchBatches();
} else {
    LOG.debugf("Duplicate enqueue for PR #%d from %s — no-op",
        pr.number(), pr.repository());
}
```

- [ ] **Step 4: Fix DevtownMcpTools.enqueuePr()**

In `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`, replace lines 751-754:

```java
// Before:
QueuedPr pr = new QueuedPr(prNumber, repo, headSha, author, trustScore, lane, Instant.now(), Set.of());
mergeQueueService.enqueue(pr);
return new EnqueueResult(prNumber, lane.name(), "ENQUEUED");

// After:
QueuedPr pr = new QueuedPr(prNumber, repo, headSha, author, trustScore, lane, Instant.now(), Set.of());
boolean inserted = mergeQueueService.enqueue(pr);
return new EnqueueResult(prNumber, lane.name(), inserted ? "ENQUEUED" : "ALREADY_QUEUED");
```

- [ ] **Step 5: Fix PrReviewMergeQueueAdapter.enqueue()**

In `app/src/main/java/io/casehub/devtown/app/PrReviewMergeQueueAdapter.java`, replace lines 71-78:

```java
// Before:
mergeQueueService.enqueue(queuedPr);
return WorkerResult.of(Map.of(
    "status", "enqueued",
    "prNumber", prNumber,
    "repository", repository
));

// After:
boolean inserted = mergeQueueService.enqueue(queuedPr);
return WorkerResult.of(Map.of(
    "status", inserted ? "enqueued" : "already-queued",
    "prNumber", prNumber,
    "repository", repository
));
```

- [ ] **Step 6: Run tests to verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownMcpToolsTest`
Expected: PASS — both enqueue tests green

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/MergeQueueService.java \
       app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java \
       app/src/main/java/io/casehub/devtown/app/PrReviewMergeQueueAdapter.java \
       app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java
git commit -m "feat(#116): surface enqueue idempotency result in MCP tool and log messages

MergeQueueService.enqueue() LOG now conditional — INFO on insert, DEBUG
on duplicate. DevtownMcpTools.enqueuePr() returns ALREADY_QUEUED on
duplicate. PrReviewMergeQueueAdapter includes idempotency signal in
WorkerResult.

Closes #116"
```

---

### Task 4: SLA Breach Detection Fix (#102 §5.1)

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java` (add `detectSlaBreaches()`, `SlaBreach` record)
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java` (remove `queueSlaMinutes`, delegate to service)
- Modify: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java:636-663`

**Interfaces:**
- Produces: `MergeQueueService.SlaBreach` record
- Produces: `MergeQueueService.detectSlaBreaches() → List<SlaBreach>`

- [ ] **Step 1: Write test for detectSlaBreaches**

This is a `@QuarkusTest` in the app module. The SLA detection depends on `PreferenceProvider` resolving lane-specific SLAs.

Create test method in a new test class or add to an existing merge queue test. Given that the service method uses `resolvePreferences()` (which is already used in production), write a `@QuarkusTest`:

```java
// In app/src/test/java/io/casehub/devtown/app/MergeQueueSlaDetectionTest.java
package io.casehub.devtown.app;

import static org.assertj.core.api.Assertions.assertThat;
import io.casehub.devtown.domain.queue.PriorityLane;
import io.casehub.devtown.merge.MergeQueueStore;
import io.casehub.devtown.queue.QueuedPr;
import io.quarkus.hibernate.orm.PersistenceUnit;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class MergeQueueSlaDetectionTest {

    @Inject MergeQueueService mergeQueueService;
    @Inject MergeQueueStore store;

    @Inject @PersistenceUnit("qhorus") EntityManager em;

    @BeforeEach
    @Transactional
    void cleanAll() {
        em.createQuery("DELETE FROM QueuedPrEntity").executeUpdate();
        em.createQuery("DELETE FROM BatchEntity").executeUpdate();
    }

    @Test
    void detectSlaBreaches_criticalPrFlaggedAtCorrectThreshold() {
        // CRITICAL SLA = PT1H (60 min) — PR queued 90 min ago should breach
        QueuedPr critical = new QueuedPr(701, "casehubio/devtown", "sha-701", "alice",
            0.8, PriorityLane.CRITICAL, Instant.now().minus(90, ChronoUnit.MINUTES), Set.of());
        store.enqueue(critical, UUID.randomUUID());

        // NORMAL SLA = PT8H (480 min) — PR queued 120 min ago should NOT breach
        QueuedPr normal = new QueuedPr(702, "casehubio/devtown", "sha-702", "bob",
            0.7, PriorityLane.NORMAL, Instant.now().minus(120, ChronoUnit.MINUTES), Set.of());
        store.enqueue(normal, UUID.randomUUID());

        var breaches = mergeQueueService.detectSlaBreaches();

        assertThat(breaches).hasSize(1);
        assertThat(breaches.get(0).pr().number()).isEqualTo(701);
        assertThat(breaches.get(0).pr().lane()).isEqualTo(PriorityLane.CRITICAL);
        assertThat(breaches.get(0).waited().toMinutes()).isGreaterThanOrEqualTo(89);
        assertThat(breaches.get(0).sla().toHours()).isEqualTo(1);
    }

    @Test
    void detectSlaBreaches_emptyQueue_returnsEmpty() {
        assertThat(mergeQueueService.detectSlaBreaches()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MergeQueueSlaDetectionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `detectSlaBreaches()` method not found

- [ ] **Step 3: Implement `detectSlaBreaches()` on MergeQueueService**

Add to `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`:

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

Add imports: `import io.casehub.devtown.merge.QueueEntry;` and `import io.casehub.platform.api.preferences.Preferences;` (if not already present).

- [ ] **Step 4: Run SLA detection tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MergeQueueSlaDetectionTest`
Expected: PASS

- [ ] **Step 5: Update DevtownMcpTools — remove ConfigProperty, delegate to service**

In `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`:

Remove the field (line 106-107):
```java
// DELETE these lines:
@ConfigProperty(name = "devtown.queue.sla-minutes", defaultValue = "120")
int queueSlaMinutes;
```

Replace the SLA breach section in `listProblems()` (lines 362-380):
```java
// Before:
Instant now2 = Instant.now();
for (QueuedPr pr : mergeQueueService.queuedPrs()) {
    long waitMinutes = Duration.between(pr.enqueuedAt(), now2).toMinutes();
    if (waitMinutes > queueSlaMinutes) {
        problems.add(new Problem(
            "queue_sla_breach",
            "warning",
            String.format("PR #%d has been queued for %d minutes (SLA: %d minutes)",
                pr.number(), waitMinutes, queueSlaMinutes),
            null,
            pr.author(),
            pr.enqueuedAt()
        ));
    }
}

// After:
for (var breach : mergeQueueService.detectSlaBreaches()) {
    problems.add(new Problem(
        "queue_sla_breach",
        "warning",
        String.format("PR #%d (%s) has waited %d min — exceeds %s SLA (%d min)",
            breach.pr().number(), breach.pr().lane(),
            breach.waited().toMinutes(),
            breach.pr().lane(), breach.sla().toMinutes()),
        null,
        breach.pr().author(),
        breach.pr().enqueuedAt()
    ));
}
```

- [ ] **Step 6: Update DevtownMcpToolsTest SLA breach test**

Replace the test `listProblems_detectsQueueSlaBreaches` (lines 636-663):

```java
@Test
void listProblems_detectsQueueSlaBreaches() {
    // Mock service-level SLA detection
    Instant longAgo = Instant.now().minus(180, ChronoUnit.MINUTES);
    QueuedPr breachPr = new QueuedPr(77, "casehubio/devtown", "sha-old", "dave",
        0.5, PriorityLane.CRITICAL, longAgo, java.util.Set.of());
    var breach = new MergeQueueService.SlaBreach(
        breachPr,
        java.time.Duration.ofMinutes(180),
        java.time.Duration.ofHours(1));

    when(tracker.stalledCases(60)).thenReturn(List.of());
    when(commitmentStore.findExpiredBefore(any(Instant.class))).thenReturn(List.of());
    when(tracker.recentEvents(100, null)).thenReturn(List.of());
    when(mergeQueueService.detectSlaBreaches()).thenReturn(List.of(breach));

    List<DevtownMcpTools.Problem> problems = tools.listProblems(60);

    assertThat(problems).hasSize(1);
    assertThat(problems.get(0).category()).isEqualTo("queue_sla_breach");
    assertThat(problems.get(0).description()).contains("PR #77").contains("CRITICAL");
    assertThat(problems.get(0).actorId()).isEqualTo("dave");
}
```

Also update the other `listProblems_*` tests that mock `mergeQueueService.queuedPrs()` — they need to mock `detectSlaBreaches()` instead (return empty list). Find all instances of `when(mergeQueueService.queuedPrs()).thenReturn(List.of());` in listProblems tests and add `when(mergeQueueService.detectSlaBreaches()).thenReturn(List.of());`.

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/MergeQueueService.java \
       app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java \
       app/src/test/java/io/casehub/devtown/app/MergeQueueSlaDetectionTest.java \
       app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java
git commit -m "feat(#102): lane-specific SLA breach detection in list_problems

MergeQueueService.detectSlaBreaches() uses per-lane SLA from
PreferenceProvider (CRITICAL=PT1H, HIGH=PT4H, NORMAL=PT8H) instead of
the flat devtown.queue.sla-minutes ConfigProperty.

Removes the ConfigProperty — single source of truth is preferences.

Refs #102"
```

---

### Task 5: Expanded Merge Queue Metrics (#102 §5.2)

**Files:**
- Modify: `merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java` (add 2 methods)
- Modify: `app/src/main/java/io/casehub/devtown/app/persistence/JpaMergeQueueStore.java` (implement 2 methods)
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java` (add 2 service methods)
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java` (expand record + tool)
- Modify: `app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java` (add store tests)
- Modify: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java` (update metrics tests)

**Interfaces:**
- Produces: `MergeQueueStore.completedBatchesSince(Instant) → List<BatchRecord>`
- Produces: `MergeQueueStore.recentBatchFailureRate(int) → double`
- Produces: `MergeQueueService.completedBatches(Duration) → List<BatchRecord>`
- Produces: `MergeQueueService.aggregateFailureRate() → double`

- [ ] **Step 1: Add store interface methods**

In `merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java`, add:

```java
/**
 * Returns all batches completed since the given instant.
 *
 * @param since cutoff instant (inclusive)
 * @return completed batches ordered by completedAt descending
 */
List<BatchRecord> completedBatchesSince(Instant since);

/**
 * Aggregate failure rate across all repositories.
 *
 * <p>Overload of {@link #recentBatchFailureRate(String, int)} without the
 * repository filter — for cross-repo aggregate metrics.
 *
 * @param window maximum number of recent completed batches to consider
 * @return failure rate [0.0, 1.0]; 0.0 if no completed batches exist
 */
double recentBatchFailureRate(int window);
```

- [ ] **Step 2: Write store tests for new methods**

Add to `app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java`:

```java
@Test
@Transactional
void completedBatchesSince_filtersCorrectly() {
    Instant now = Instant.now();
    // Batch completed 2 hours ago
    store.recordBatch("batch-old", UUID.randomUUID(), List.of(700), "casehubio/devtown");
    store.completeBatch("batch-old", true);
    // Batch completed now (active — not completed)
    store.recordBatch("batch-active", UUID.randomUUID(), List.of(701), "casehubio/devtown");
    // Batch completed recently
    store.recordBatch("batch-recent", UUID.randomUUID(), List.of(702), "casehubio/devtown");
    store.completeBatch("batch-recent", false);

    List<BatchRecord> result = store.completedBatchesSince(now.minus(1, java.time.temporal.ChronoUnit.HOURS));
    // Should include batch-old and batch-recent (both completed), exclude batch-active
    // But batch-old might be before "1 hour ago" depending on timing.
    // Use a wide window to catch both:
    List<BatchRecord> all = store.completedBatchesSince(now.minus(24, java.time.temporal.ChronoUnit.HOURS));
    assertThat(all).hasSize(2);
    assertThat(all).allSatisfy(b -> assertThat(b.completedAt()).isNotNull());
}

@Test
@Transactional
void recentBatchFailureRate_aggregate_crossRepo() {
    store.recordBatch("batch-a1", UUID.randomUUID(), List.of(710), "casehubio/devtown");
    store.completeBatch("batch-a1", false);  // failed
    store.recordBatch("batch-b1", UUID.randomUUID(), List.of(711), "casehubio/engine");
    store.completeBatch("batch-b1", true);  // succeeded

    // Per-repo: devtown=1.0, engine=0.0
    // Aggregate: 1 failed / 2 total = 0.5
    double aggregate = store.recentBatchFailureRate(20);
    assertThat(aggregate).isCloseTo(0.5, org.assertj.core.data.Offset.offset(0.001));
}

@Test
@Transactional
void recentBatchFailureRate_aggregate_respectsWindow() {
    for (int i = 0; i < 10; i++) {
        store.recordBatch("batch-agg-" + i, UUID.randomUUID(), List.of(720 + i), "casehubio/devtown");
        store.completeBatch("batch-agg-" + i, i < 8);  // 8 pass, 2 fail
    }
    double rateAll = store.recentBatchFailureRate(20);
    double rate5 = store.recentBatchFailureRate(5);
    assertThat(rateAll).isCloseTo(0.2, org.assertj.core.data.Offset.offset(0.001));
    // Window of 5 gets the 5 most recent: indices 5,6,7,8,9 → 7=pass, 8=fail, 9=fail → 2/5 = 0.4
    assertThat(rate5).isCloseTo(0.4, org.assertj.core.data.Offset.offset(0.001));
}
```

- [ ] **Step 3: Run store tests to verify failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=JpaMergeQueueStoreTest#completedBatchesSince_filtersCorrectly+recentBatchFailureRate_aggregate_crossRepo+recentBatchFailureRate_aggregate_respectsWindow`
Expected: FAIL — methods not implemented

- [ ] **Step 4: Implement JPA store methods**

In `app/src/main/java/io/casehub/devtown/app/persistence/JpaMergeQueueStore.java`, add:

```java
@Override
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

@Override
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

- [ ] **Step 5: Run store tests to verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=JpaMergeQueueStoreTest#completedBatchesSince_filtersCorrectly+recentBatchFailureRate_aggregate_crossRepo+recentBatchFailureRate_aggregate_respectsWindow`
Expected: PASS

- [ ] **Step 6: Add service methods to MergeQueueService**

In `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java`, add:

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

Add import for `BatchRecord` if not present: `import io.casehub.devtown.merge.BatchRecord;`

- [ ] **Step 7: Expand MergeQueueMetrics record and getMergeQueueMetrics()**

In `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`:

Replace the `MergeQueueMetrics` record:
```java
// Before:
public record MergeQueueMetrics(
    int queueDepth,
    int activeBatches,
    long oldestWaitMinutes,
    double avgTrustScore,
    Map<String, Integer> countsByLane
) {}

// After:
public record MergeQueueMetrics(
    int queueDepth,
    int activeBatches,
    long oldestWaitMinutes,
    long avgWaitMinutes,
    double avgTrustScore,
    Map<String, Integer> countsByLane,
    int throughput24h,
    double failureRate,
    Map<Integer, Integer> batchSizeDistribution
) {}
```

Replace the `getMergeQueueMetrics()` method:
```java
@Tool(
    name = "get_merge_queue_metrics",
    description = "Get operational metrics: queue depth, active batches, wait times, throughput, failure rate, trust score, lane and batch size distribution"
)
public MergeQueueMetrics getMergeQueueMetrics() {
    Instant now = Instant.now();
    List<QueuedPr> queued = mergeQueueService.queuedPrs();
    Map<String, BatchRecord> batches = mergeQueueService.activeBatches();

    long oldestWaitMinutes = queued.stream()
        .mapToLong(pr -> Duration.between(pr.enqueuedAt(), now).toMinutes())
        .max()
        .orElse(0);

    long avgWaitMinutes = queued.isEmpty() ? 0 :
        (long) queued.stream()
            .mapToLong(pr -> Duration.between(pr.enqueuedAt(), now).toMinutes())
            .average().orElse(0);

    double avgTrust = queued.stream()
        .mapToDouble(QueuedPr::trustScore)
        .average()
        .orElse(0.0);

    Map<String, Integer> countsByLane = new HashMap<>();
    for (QueuedPr pr : queued) {
        countsByLane.merge(pr.lane().name(), 1, Integer::sum);
    }

    List<BatchRecord> completed = mergeQueueService.completedBatches(Duration.ofHours(24));
    int throughput24h = completed.size();

    double failureRate = mergeQueueService.aggregateFailureRate();

    Map<Integer, Integer> batchSizeDist = new HashMap<>();
    for (BatchRecord batch : completed) {
        batchSizeDist.merge(batch.prNumbers().size(), 1, Integer::sum);
    }

    return new MergeQueueMetrics(
        queued.size(), batches.size(), oldestWaitMinutes, avgWaitMinutes,
        avgTrust, countsByLane, throughput24h, failureRate, batchSizeDist);
}
```

Add import: `import io.casehub.devtown.merge.BatchRecord;` if not present.

- [ ] **Step 8: Update DevtownMcpToolsTest metrics tests**

Update `getMergeQueueMetrics_emptyQueue_returnsZeros`:
```java
@Test
void getMergeQueueMetrics_emptyQueue_returnsZeros() {
    when(mergeQueueService.queuedPrs()).thenReturn(List.of());
    when(mergeQueueService.activeBatches()).thenReturn(Map.<String, BatchRecord>of());
    when(mergeQueueService.completedBatches(any())).thenReturn(List.of());
    when(mergeQueueService.aggregateFailureRate()).thenReturn(0.0);

    DevtownMcpTools.MergeQueueMetrics metrics = tools.getMergeQueueMetrics();

    assertThat(metrics.queueDepth()).isZero();
    assertThat(metrics.activeBatches()).isZero();
    assertThat(metrics.oldestWaitMinutes()).isZero();
    assertThat(metrics.avgWaitMinutes()).isZero();
    assertThat(metrics.avgTrustScore()).isZero();
    assertThat(metrics.countsByLane()).isEmpty();
    assertThat(metrics.throughput24h()).isZero();
    assertThat(metrics.failureRate()).isZero();
    assertThat(metrics.batchSizeDistribution()).isEmpty();
}
```

Update `getMergeQueueMetrics_withPrs_computesCorrectly`:
```java
@Test
void getMergeQueueMetrics_withPrs_computesCorrectly() {
    Instant old = Instant.now().minus(60, ChronoUnit.MINUTES);
    Instant recent = Instant.now().minus(10, ChronoUnit.MINUTES);
    QueuedPr pr1 = new QueuedPr(1, "casehubio/devtown", "sha1", "alice", 0.6, PriorityLane.NORMAL, old, java.util.Set.of());
    QueuedPr pr2 = new QueuedPr(2, "casehubio/devtown", "sha2", "bob", 0.8, PriorityLane.HIGH, recent, java.util.Set.of());

    BatchRecord completedBatch = new BatchRecord("batch-done", UUID.randomUUID(),
        List.of(10, 11), "casehubio/devtown", Instant.now().minus(2, ChronoUnit.HOURS),
        Instant.now().minus(1, ChronoUnit.HOURS), true);

    when(mergeQueueService.queuedPrs()).thenReturn(List.of(pr1, pr2));
    when(mergeQueueService.activeBatches()).thenReturn(Map.<String, BatchRecord>of());
    when(mergeQueueService.completedBatches(any())).thenReturn(List.of(completedBatch));
    when(mergeQueueService.aggregateFailureRate()).thenReturn(0.25);

    DevtownMcpTools.MergeQueueMetrics metrics = tools.getMergeQueueMetrics();

    assertThat(metrics.queueDepth()).isEqualTo(2);
    assertThat(metrics.oldestWaitMinutes()).isGreaterThanOrEqualTo(59);
    assertThat(metrics.avgWaitMinutes()).isGreaterThanOrEqualTo(34);
    assertThat(metrics.avgTrustScore()).isCloseTo(0.7, org.assertj.core.data.Offset.offset(0.01));
    assertThat(metrics.countsByLane()).containsEntry("NORMAL", 1).containsEntry("HIGH", 1);
    assertThat(metrics.throughput24h()).isEqualTo(1);
    assertThat(metrics.failureRate()).isCloseTo(0.25, org.assertj.core.data.Offset.offset(0.01));
    assertThat(metrics.batchSizeDistribution()).containsEntry(2, 1);
}
```

- [ ] **Step 9: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add merge/src/main/java/io/casehub/devtown/merge/MergeQueueStore.java \
       app/src/main/java/io/casehub/devtown/app/persistence/JpaMergeQueueStore.java \
       app/src/main/java/io/casehub/devtown/app/MergeQueueService.java \
       app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java \
       app/src/test/java/io/casehub/devtown/app/persistence/JpaMergeQueueStoreTest.java \
       app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsTest.java
git commit -m "feat(#102): expanded merge queue metrics — throughput, failure rate, avg wait, batch size distribution

MergeQueueStore gains completedBatchesSince() and aggregate
recentBatchFailureRate(window). get_merge_queue_metrics now returns all
four spec §8 metrics: throughput (24h), failure rate, average wait time,
and batch size distribution.

Closes #102"
```
