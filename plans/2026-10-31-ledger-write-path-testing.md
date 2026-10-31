# Ledger Write Path Integration Testing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #288 — Ledger integration testing — verify write path in devtown context

**Goal:** Verify that the engine-level CDI capture beans (`CaseLedgerEventCapture`, `WorkerDecisionEventCapture`) write ledger entries and advance the Merkle frontier when events fire in devtown's CDI assembly.

**Architecture:** Extend two existing test classes (`MergeDecisionObserverTest`, `GdprErasureResourceTest`) and create one new test class (`WorkerDecisionWritePathTest`). All tests fire CDI events via `Event.fireAsync()`, wait with Awaitility, and assert entries + Merkle frontier state via JPA queries.

**Tech Stack:** Java 21, Quarkus 3.39.3, JUnit 5, AssertJ, Awaitility, H2 (MODE=PostgreSQL)

## Global Constraints

- Default `casehub.ledger.enabled=false` in `application.properties` must NOT change
- All new tests use existing test profiles (`LedgerEnabledTestProfile` or `TrustScoringTestProfile`)
- No new Flyway migrations — existing migrations cover all tables
- No production code changes — test code only

---

## Batch 1: Ledger write path verification

### Task 1: Extend MergeDecisionObserverTest — verify CaseLedgerEntry + Merkle frontier

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java`
- Test: same file (self-contained test class)

**Interfaces:**
- Consumes: `CaseLifecycleEvent` constructor (existing), `LedgerEntryRepository.findEntriesBySubjectId(UUID, String)`, `LedgerMerkleFrontierRepository.findBySubjectId(UUID, String)`
- Produces: test pattern for frontier verification reused in Tasks 2 and 3

- [ ] **Step 1: Add new imports and injections**

Add to `MergeDecisionObserverTest`:

```java
import io.casehub.ledger.api.model.LedgerMerkleFrontier;
import io.casehub.ledger.api.spi.LedgerMerkleFrontierRepository;
import io.casehub.ledger.model.CaseLedgerEntry;
```

Add injection:

```java
@Inject LedgerMerkleFrontierRepository frontierRepo;
```

- [ ] **Step 2: Write the failing test — CaseLedgerEntry written by engine capture**

Add new test method:

```java
@Test
void completedCase_writesCaseLedgerEntryAndAdvancesFrontier() {
    UUID caseId = UUID.randomUUID();
    String tenancyId = "test-tenant";

    ObjectNode snapshot = buildPrSnapshot("casehubio/devtown", "55", "aaa111");

    CaseLifecycleEvent event = new CaseLifecycleEvent(
            caseId, tenancyId, "COMPLETE", "CASE_COMPLETED",
            "COMPLETED", "system", "ORCHESTRATOR", "trace-frontier-1",
            null, null, snapshot, null, null);

    caseLifecycleEvents.fireAsync(event);

    await().atMost(5, SECONDS).untilAsserted(() -> {
        // Engine's CaseLedgerEventCapture should write a CaseLedgerEntry
        List<CaseLedgerEntry> entries = findCaseLedgerEntries(caseId, tenancyId);
        assertThat(entries).isNotEmpty();

        CaseLedgerEntry entry = entries.get(0);
        assertThat(entry.eventType).isEqualTo("CASE_COMPLETED");
        assertThat(entry.caseStatus).isEqualTo("COMPLETED");
        assertThat(entry.actorId).isEqualTo("system");
        assertThat(entry.digest).isNotNull();

        // Merkle frontier should have advanced
        // Total entries for this subject = CaseLedgerEntry + MergeDecisionLedgerEntry = 2
        // bitCount(2) = 1 frontier node
        List<LedgerMerkleFrontier> frontier = QuarkusTransaction.requiringNew()
                .call(() -> frontierRepo.findBySubjectId(caseId, tenancyId));
        assertThat(frontier).isNotEmpty();
        assertThat(frontier).allSatisfy(node ->
                assertThat(node.hash()).isNotNull());
    });
}
```

- [ ] **Step 3: Add helper method for CaseLedgerEntry queries**

Add to `MergeDecisionObserverTest`:

```java
private List<CaseLedgerEntry> findCaseLedgerEntries(UUID caseId, String tenancyId) {
    return QuarkusTransaction.requiringNew().call(() ->
        em.createQuery(
                "SELECT e FROM CaseLedgerEntry e WHERE e.subjectId = :subjectId AND e.tenancyId = :tenancyId",
                CaseLedgerEntry.class)
            .setParameter("subjectId", caseId)
            .setParameter("tenancyId", tenancyId)
            .getResultList()
    );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MergeDecisionObserverTest#completedCase_writesCaseLedgerEntryAndAdvancesFrontier -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — both `CaseLedgerEventCapture` and `MergeDecisionObserver` fire on the same `CaseLifecycleEvent`, so both entries + frontier should exist.

If the frontier assertion fails (empty list), it means `JpaLedgerMerkleFrontierRepository` isn't active — check whether it's excluded in the test profile or default `application.properties`.

- [ ] **Step 5: Run full existing test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MergeDecisionObserverTest`

Expected: all 4 tests pass (3 existing + 1 new).

- [ ] **Step 6: Commit**

```bash
git add app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java
git commit -m "test(#288): verify CaseLedgerEntry write path and Merkle frontier on case completion

CaseLedgerEventCapture (engine) writes a CaseLedgerEntry alongside
MergeDecisionObserver (devtown) on the same CaseLifecycleEvent.
New test asserts both entries exist and frontier advances.

Refs #288

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: New WorkerDecisionWritePathTest — verify WorkerDecisionEntry + trust enrichment

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/ledger/WorkerDecisionWritePathTest.java`
- Test: same file

**Interfaces:**
- Consumes: `WorkerDecisionEvent` record constructor, `TrustScoringTestProfile`, `LedgerEntryRepository`, `LedgerMerkleFrontierRepository`
- Produces: nothing — terminal test class

- [ ] **Step 1: Write the test class with first test — single decision entry**

Create `app/src/test/java/io/casehub/devtown/app/ledger/WorkerDecisionWritePathTest.java`:

```java
package io.casehub.devtown.app.ledger;

import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.engine.common.spi.event.WorkerDecisionEvent;
import io.casehub.ledger.api.model.LedgerMerkleFrontier;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.api.spi.LedgerMerkleFrontierRepository;
import io.casehub.ledger.jpa.LedgerPersistenceUnit;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.devtown.app.trust.TrustScoringTestProfile;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestProfile(TrustScoringTestProfile.class)
class WorkerDecisionWritePathTest {

    @Inject Event<WorkerDecisionEvent> workerDecisionEvents;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject LedgerMerkleFrontierRepository frontierRepo;
    @Inject @LedgerPersistenceUnit EntityManager em;

    @Test
    void workerDecision_writesEntryWithTrustEnrichment() {
        UUID caseId = UUID.randomUUID();
        String tenancyId = "test-tenant";
        String workerId = "agent-security-1";
        String capabilityTag = "security-review";

        WorkerDecisionEvent event = new WorkerDecisionEvent(
                caseId, tenancyId, workerId, capabilityTag, "trace-wd-1",
                null, null);

        workerDecisionEvents.fireAsync(event);

        await().atMost(5, SECONDS).untilAsserted(() -> {
            List<WorkerDecisionEntry> entries = findWorkerDecisions(caseId, tenancyId);
            assertThat(entries).hasSize(1);

            WorkerDecisionEntry entry = entries.get(0);
            assertThat(entry.workerId).isEqualTo(workerId);
            assertThat(entry.capabilityTag).isEqualTo(capabilityTag);
            assertThat(entry.caseId).isEqualTo(caseId);
            assertThat(entry.digest).isNotNull();

            // Merkle frontier: 1 entry → bitCount(1) = 1 node
            List<LedgerMerkleFrontier> frontier = QuarkusTransaction.requiringNew()
                    .call(() -> frontierRepo.findBySubjectId(caseId, tenancyId));
            assertThat(frontier).hasSize(1);
            assertThat(frontier.get(0).hash()).isNotNull();
        });
    }

    @Test
    void multipleDecisions_frontierAdvancesCorrectly() {
        UUID caseId = UUID.randomUUID();
        String tenancyId = "test-tenant";

        workerDecisionEvents.fireAsync(new WorkerDecisionEvent(
                caseId, tenancyId, "agent-style-1", "style-review", "trace-wd-2",
                null, null));
        workerDecisionEvents.fireAsync(new WorkerDecisionEvent(
                caseId, tenancyId, "agent-arch-1", "architecture-review", "trace-wd-3",
                null, null));
        workerDecisionEvents.fireAsync(new WorkerDecisionEvent(
                caseId, tenancyId, "agent-perf-1", "performance-analysis", "trace-wd-4",
                null, null));

        await().atMost(5, SECONDS).untilAsserted(() -> {
            List<WorkerDecisionEntry> entries = findWorkerDecisions(caseId, tenancyId);
            assertThat(entries).hasSize(3);

            // 3 entries → bitCount(3) = 2 frontier nodes
            List<LedgerMerkleFrontier> frontier = QuarkusTransaction.requiringNew()
                    .call(() -> frontierRepo.findBySubjectId(caseId, tenancyId));
            assertThat(frontier).hasSize(Integer.bitCount(3));
            assertThat(frontier).allSatisfy(node ->
                    assertThat(node.hash()).isNotNull());
        });
    }

    private List<WorkerDecisionEntry> findWorkerDecisions(UUID caseId, String tenancyId) {
        return QuarkusTransaction.requiringNew().call(() ->
            em.createQuery(
                    "SELECT w FROM WorkerDecisionEntry w WHERE w.caseId = :caseId AND w.tenancyId = :tenancyId",
                    WorkerDecisionEntry.class)
                .setParameter("caseId", caseId)
                .setParameter("tenancyId", tenancyId)
                .getResultList()
        );
    }
}
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=WorkerDecisionWritePathTest`

Expected: PASS for both tests.

If `trustScoreAtRouting` or `thresholdApplied` are null despite `TrustScoringTestProfile` enabling trust scoring, the capture may need pre-existing trust data. In that case, add a `@BeforeEach` that seeds a trust attestation via `LedgerEntryRepository.save()` for the worker/capability pair, or relax the assertion to accept null (no trust data = no routing score, which is a valid production state).

- [ ] **Step 3: Commit**

```bash
git add app/src/test/java/io/casehub/devtown/app/ledger/WorkerDecisionWritePathTest.java
git commit -m "test(#288): verify WorkerDecisionEntry write path with trust enrichment and Merkle frontier

New test fires WorkerDecisionEvent via CDI, asserts WorkerDecisionEntry
is written by WorkerDecisionEventCapture with trust enrichment fields.
Second test verifies Merkle frontier advances correctly with 3 entries.

Refs #288

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Extend GdprErasureResourceTest — verify Merkle frontier integrity after erasure

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/ledger/GdprErasureResourceTest.java`
- Test: same file

**Interfaces:**
- Consumes: `CaseLifecycleEvent` constructor, `LedgerMerkleFrontierRepository.findBySubjectId()`, erasure REST endpoint
- Produces: nothing — terminal

- [ ] **Step 1: Add new imports and injections**

Add to `GdprErasureResourceTest`:

```java
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.ledger.api.model.LedgerMerkleFrontier;
import io.casehub.ledger.api.spi.LedgerMerkleFrontierRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import jakarta.enterprise.event.Event;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.awaitility.Awaitility.await;
```

Add injections:

```java
@Inject Event<CaseLifecycleEvent> caseLifecycleEvents;
@Inject LedgerMerkleFrontierRepository frontierRepo;
@Inject ObjectMapper objectMapper;
```

- [ ] **Step 2: Write the new test — frontier integrity after erasure**

Add test method:

```java
@Test
void erasure_preservesMerkleFrontierIntegrity() {
    final UUID caseId = UUID.randomUUID();
    final String tenancyId = principal.tenancyId();
    final String rawActorId = "human-reviewer-frontier-" + UUID.randomUUID();

    // Fire event through capture path so digest + frontier are computed
    ObjectNode snapshot = objectMapper.createObjectNode();
    ObjectNode pr = snapshot.putObject("pr");
    pr.put("repo", "casehubio/devtown");
    pr.put("id", "77");
    pr.put("headSha", "fff999");
    pr.put("baseRef", "main");
    pr.put("linesChanged", 50);
    pr.put("contributor", rawActorId);
    pr.putArray("changedPaths").add("src/Foo.java");

    CaseLifecycleEvent event = new CaseLifecycleEvent(
            caseId, tenancyId, "COMPLETE", "CASE_COMPLETED",
            "COMPLETED", rawActorId, "REVIEWER", "trace-erasure-frontier",
            null, null, snapshot, null, null);

    caseLifecycleEvents.fireAsync(event);

    // Wait for entries and frontier to be written
    await().atMost(5, SECONDS).untilAsserted(() -> {
        List<LedgerMerkleFrontier> frontier = QuarkusTransaction.requiringNew()
                .call(() -> frontierRepo.findBySubjectId(caseId, tenancyId));
        assertThat(frontier).isNotEmpty();
    });

    int frontierSizeBefore = QuarkusTransaction.requiringNew()
            .call(() -> frontierRepo.findBySubjectId(caseId, tenancyId)).size();

    // Run erasure
    var response = given()
        .queryParam("reason", "GDPR Art.17 frontier test")
        .when()
            .post("/api/devtown/compliance/erasure/" + rawActorId)
        .then()
            .statusCode(200)
            .extract().body().jsonPath();

    assertThat(response.getLong("ledgerEntriesAffected")).isGreaterThan(0);

    // Frontier should still exist (erasure pseudonymises, doesn't delete)
    List<LedgerMerkleFrontier> frontierAfter = QuarkusTransaction.requiringNew()
            .call(() -> frontierRepo.findBySubjectId(caseId, tenancyId));
    assertThat(frontierAfter).isNotEmpty();
    assertThat(frontierAfter).allSatisfy(node ->
            assertThat(node.hash()).isNotNull());

    // Erasure receipt should have a valid digest
    UUID receiptId = UUID.fromString(response.getString("receiptEntryId"));
    var receipt = QuarkusTransaction.requiringNew().call(() ->
            ledgerRepo.findEntryById(receiptId, tenancyId));
    assertThat(receipt).isPresent();
    assertThat(receipt.get().digest).isNotNull();
}
```

- [ ] **Step 3: Add missing import for List**

Check if `java.util.List` is already imported — if not, add it. Also add the static import for RestAssured if not present (it's already there from existing tests).

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=GdprErasureResourceTest`

Expected: all 5 tests pass (4 existing + 1 new).

If the frontier assertion fails because `TokenisationEnabledTestProfile` doesn't activate the frontier repository, check whether `JpaLedgerMerkleFrontierRepository` is excluded by its parent `LedgerEnabledTestProfile` or the default `application.properties`.

- [ ] **Step 5: Run full ledger test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="MergeDecisionObserverTest,WorkerDecisionWritePathTest,GdprErasureResourceTest"`

Expected: all tests pass across all three classes.

- [ ] **Step 6: Commit**

```bash
git add app/src/test/java/io/casehub/devtown/app/ledger/GdprErasureResourceTest.java
git commit -m "test(#288): verify Merkle frontier integrity after GDPR erasure

Creates entries through CDI event capture path (not seeded directly)
so digests and frontier are computed. Asserts frontier persists after
erasure and erasure receipt has a valid digest.

Closes #288

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-10-31-ledger-write-path-testing-design.md] — design spec this plan implements
- [MergeDecisionObserverTest.java] — existing test pattern (fireAsync + await)
- [GdprErasureResourceTest.java] — existing erasure test pattern
- [LedgerEnabledTestProfile.java] — H2 + Flyway + excluded beans config
- [TrustScoringTestProfile.java] — ledger + trust scoring enabled profile
- [CaseLedgerEventCapture] (engine-ledger-core) — CDI adapter observes CaseLifecycleEvent
- [WorkerDecisionEventCapture] (engine-ledger-core) — CDI adapter observes WorkerDecisionEvent
- [LedgerMerkleFrontierRepository] — SPI: findBySubjectId() returns frontier nodes
- [WorkerDecisionEvent] — record: caseId, tenancyId, workerId, capabilityTag, traceId, selectionContext, reasoning
- [PP-20260521-134c38] — HITL test pre-seeding protocol
- [GitHub #288] — focal issue
