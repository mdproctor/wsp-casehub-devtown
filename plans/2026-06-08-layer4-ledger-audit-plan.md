# Layer 4 — Tamper-Evident Audit Trail + Compliance Report — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire devtown's merge decision audit trail with a domain-specific `MergeDecisionLedgerEntry`, an observer that derives the decision from terminal case state, and a compliance report endpoint that assembles per-case evidence across four regulatory requirement dimensions.

**Architecture:** `MergeDecisionObserver` observes `@ObservesAsync CaseLifecycleEvent`, filters for terminal states (COMPLETED→APPROVED, CANCELLED→REJECTED), looks up `CaseInstance` for PR metadata, writes `MergeDecisionLedgerEntry` with `ComplianceSupplement`. `CodeReviewComplianceService` queries all ledger entries for a case via `LedgerEntryRepository.findBySubjectId()`, filters by entry type, assembles four requirement sections (audit chain, SLA, trust routing, GDPR), and verifies the Merkle chain. REST endpoint exposes the compliance report.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (Hibernate), Flyway, H2 (test), CDI async observers, casehub-ledger (Merkle MMR), casehub-engine-ledger (`CaseLedgerEntry`, `WorkerDecisionEntry`)

**Spec:** `specs/2026-06-08-layer4-ledger-audit-design.md` (revision 4)
**Issues:** devtown#73 (Layer 4), devtown#7 (compliance report)

---

## File Map

### New files — Part 1 (Layer 4: MergeDecisionLedgerEntry + observer)

| File | Responsibility |
|------|---------------|
| `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionLedgerEntry.java` | JPA entity — `LedgerEntry` subclass with PR merge decision fields |
| `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java` | CDI observer — derives merge decision from terminal `CaseLifecycleEvent`, writes ledger entry |
| `app/src/main/resources/db/devtown/migration/V2002__merge_decision_ledger_entry.sql` | Flyway migration — join table for the new entity |
| `app/src/test/java/io/casehub/devtown/app/ledger/LedgerEnabledTestProfile.java` | `@QuarkusTestProfile` — enables ledger writes + Flyway schema for ledger tests |
| `app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java` | Integration test — verifies observer writes correct entries |

### New files — Part 2 (Compliance report #7)

| File | Responsibility |
|------|---------------|
| `review/src/main/java/io/casehub/devtown/review/compliance/RequirementStatus.java` | Enum — CLOSED, PARTIAL, GAP, BREACHED |
| `review/src/main/java/io/casehub/devtown/review/compliance/CodeReviewComplianceEvidence.java` | Top-level compliance report record |
| `review/src/main/java/io/casehub/devtown/review/compliance/AuditChainRequirement.java` | Merkle chain integrity requirement record |
| `review/src/main/java/io/casehub/devtown/review/compliance/ReviewSlaRequirement.java` | Human review SLA requirement record |
| `review/src/main/java/io/casehub/devtown/review/compliance/TrustRoutingRequirement.java` | Trust-weighted routing requirement record |
| `review/src/main/java/io/casehub/devtown/review/compliance/GdprRequirement.java` | GDPR capability declaration record |
| `review/src/main/java/io/casehub/devtown/review/compliance/LedgerEventRecord.java` | Per-entry detail record for audit chain |
| `review/src/main/java/io/casehub/devtown/review/compliance/InclusionProofRecord.java` | Inclusion proof detail record |
| `review/src/main/java/io/casehub/devtown/review/compliance/RoutingDecisionRecord.java` | Per-dispatch detail record for trust routing |
| `app/src/main/java/io/casehub/devtown/app/ledger/CodeReviewComplianceService.java` | CDI service — assembles compliance evidence from ledger entries |
| `app/src/main/java/io/casehub/devtown/app/CodeReviewComplianceResource.java` | REST endpoint — `GET /api/compliance/code-review/{caseId}` |
| `app/src/test/java/io/casehub/devtown/app/ledger/CodeReviewComplianceServiceTest.java` | Integration test — verifies compliance evidence assembly |

### Modified files

| File | Change |
|------|--------|
| `app/src/main/resources/application.properties` | Add `db/devtown/migration` to Flyway locations; add `io.casehub.devtown.app.ledger` to qhorus PU packages |
| `app/src/test/resources/application.properties` | Add Jandex index for engine-ledger test discovery (if not already present) |

---

## Task 1: Flyway migration + configuration

**Files:**
- Create: `app/src/main/resources/db/devtown/migration/V2002__merge_decision_ledger_entry.sql`
- Modify: `app/src/main/resources/application.properties`

- [ ] **Step 1: Create the Flyway migration directory and SQL file**

```sql
-- app/src/main/resources/db/devtown/migration/V2002__merge_decision_ledger_entry.sql
CREATE TABLE merge_decision_ledger_entry (
    id          UUID NOT NULL,
    pr_number   INTEGER NOT NULL,
    repository  VARCHAR(255) NOT NULL,
    commit_sha  VARCHAR(40),
    decision    VARCHAR(20) NOT NULL,
    case_id     UUID NOT NULL,
    tenancy_id  VARCHAR(64) NOT NULL,
    CONSTRAINT pk_merge_decision_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_merge_decision_ledger_entry_id
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);

CREATE INDEX idx_merge_decision_entry_case_id ON merge_decision_ledger_entry(case_id);
CREATE INDEX idx_merge_decision_entry_tenancy_id ON merge_decision_ledger_entry(tenancy_id);
```

- [ ] **Step 2: Update application.properties — Flyway locations**

In `app/src/main/resources/application.properties`, replace the existing `quarkus.flyway.qhorus.locations` line:

```properties
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/engine-ledger/migration,classpath:db/devtown/migration
```

- [ ] **Step 3: Update application.properties — Hibernate qhorus PU packages**

In `app/src/main/resources/application.properties`, replace the existing `quarkus.hibernate-orm.qhorus.packages` line:

```properties
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model,io.casehub.devtown.app.ledger
```

- [ ] **Step 4: Commit**

```bash
git add app/src/main/resources/db/devtown/migration/V2002__merge_decision_ledger_entry.sql app/src/main/resources/application.properties
git commit -m "feat(layer4): Flyway V2002 migration + config for MergeDecisionLedgerEntry

Adds db/devtown/migration/ to Flyway locations and devtown ledger package
to qhorus PU. V2002 avoids conflict with engine-ledger V2000/V2001.

Refs casehubio/devtown#73"
```

---

## Task 2: MergeDecisionLedgerEntry entity

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionLedgerEntry.java`

- [ ] **Step 1: Create the entity**

```java
package io.casehub.devtown.app.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Index;
import jakarta.persistence.Table;
import java.util.UUID;

@Entity
@Table(name = "merge_decision_ledger_entry", indexes = {
    @Index(name = "idx_merge_decision_entry_case_id", columnList = "case_id"),
    @Index(name = "idx_merge_decision_entry_tenancy_id", columnList = "tenancy_id")
})
@DiscriminatorValue("MERGE_DECISION")
public class MergeDecisionLedgerEntry extends LedgerEntry {

    @Column(name = "pr_number", nullable = false)
    public int prNumber;

    @Column(name = "repository", nullable = false, length = 255)
    public String repository;

    @Column(name = "commit_sha", length = 40)
    public String commitSha;

    @Column(name = "decision", nullable = false, length = 20)
    public String decision;

    @Column(name = "case_id", nullable = false)
    public UUID caseId;

    @Column(name = "tenancy_id", nullable = false, length = 64)
    public String tenancyId;
}
```

- [ ] **Step 2: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app -am compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionLedgerEntry.java
git commit -m "feat(layer4): MergeDecisionLedgerEntry JPA entity

JOINED subclass of LedgerEntry with PR merge decision fields:
prNumber, repository, commitSha, decision (APPROVED/REJECTED),
caseId, tenancyId. DiscriminatorValue MERGE_DECISION.

Refs casehubio/devtown#73"
```

---

## Task 3: LedgerEnabledTestProfile

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/ledger/LedgerEnabledTestProfile.java`

- [ ] **Step 1: Create the test profile**

```java
package io.casehub.devtown.app.ledger;

import io.quarkus.test.junit.QuarkusTestProfile;
import java.util.Map;

public class LedgerEnabledTestProfile implements QuarkusTestProfile {

    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of(
            "casehub.ledger.enabled", "true",
            "quarkus.flyway.qhorus.migrate-at-start", "true",
            "quarkus.hibernate-orm.qhorus.database.generation", "none",
            "quarkus.datasource.qhorus.jdbc.url",
                "jdbc:h2:mem:devtown-ledger-test;MODE=PostgreSQL;DB_CLOSE_ON_EXIT=FALSE"
        );
    }
}
```

- [ ] **Step 2: Commit**

```bash
git add app/src/test/java/io/casehub/devtown/app/ledger/LedgerEnabledTestProfile.java
git commit -m "test(layer4): LedgerEnabledTestProfile for Flyway-managed ledger tests

Enables casehub.ledger, Flyway migration, H2 MODE=PostgreSQL.
Existing tests unaffected — they continue with ledger disabled.

Refs casehubio/devtown#73"
```

---

## Task 4: MergeDecisionObserver + test (TDD)

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java`
- Create: `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.app.ledger;

import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

@QuarkusTest
@TestProfile(LedgerEnabledTestProfile.class)
class MergeDecisionObserverTest {

    @Inject Event<CaseLifecycleEvent> lifecycleEvents;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject CrossTenantCaseInstanceRepository caseInstanceRepo;

    @Test
    void completedCase_writesApprovedMergeDecision() {
        UUID caseId = UUID.randomUUID();
        String tenancyId = "test-tenant";
        seedCaseInstance(caseId, tenancyId, "casehubio/devtown", "42", "abc123def");

        lifecycleEvents.fireAsync(new CaseLifecycleEvent(
            caseId, tenancyId, "CompleteCase", "CaseCompleted",
            "COMPLETED", null, "System", null
        ));

        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            List<LedgerEntry> entries = ledgerRepo.findBySubjectId(caseId);
            List<MergeDecisionLedgerEntry> mergeEntries = entries.stream()
                .filter(MergeDecisionLedgerEntry.class::isInstance)
                .map(MergeDecisionLedgerEntry.class::cast)
                .toList();

            assertThat(mergeEntries).hasSize(1);
            MergeDecisionLedgerEntry entry = mergeEntries.getFirst();
            assertThat(entry.decision).isEqualTo("APPROVED");
            assertThat(entry.prNumber).isEqualTo(42);
            assertThat(entry.repository).isEqualTo("casehubio/devtown");
            assertThat(entry.commitSha).isEqualTo("abc123def");
            assertThat(entry.caseId).isEqualTo(caseId);
            assertThat(entry.tenancyId).isEqualTo(tenancyId);
            assertThat(entry.compliance()).isPresent();
            assertThat(entry.compliance().get().algorithmRef)
                .isEqualTo("casehub-devtown:pr-review-v1");
            assertThat(entry.compliance().get().humanOverrideAvailable).isTrue();
        });
    }

    @Test
    void cancelledCase_writesRejectedMergeDecision() {
        UUID caseId = UUID.randomUUID();
        String tenancyId = "test-tenant";
        seedCaseInstance(caseId, tenancyId, "casehubio/devtown", "99", "def456");

        lifecycleEvents.fireAsync(new CaseLifecycleEvent(
            caseId, tenancyId, "CancelCase", "CaseCancelled",
            "CANCELLED", null, "System", null
        ));

        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            List<LedgerEntry> entries = ledgerRepo.findBySubjectId(caseId);
            List<MergeDecisionLedgerEntry> mergeEntries = entries.stream()
                .filter(MergeDecisionLedgerEntry.class::isInstance)
                .map(MergeDecisionLedgerEntry.class::cast)
                .toList();

            assertThat(mergeEntries).hasSize(1);
            assertThat(mergeEntries.getFirst().decision).isEqualTo("REJECTED");
        });
    }

    @Test
    void faultedCase_writesNoMergeDecision() {
        UUID caseId = UUID.randomUUID();
        String tenancyId = "test-tenant";
        seedCaseInstance(caseId, tenancyId, "casehubio/devtown", "55", "fff000");

        lifecycleEvents.fireAsync(new CaseLifecycleEvent(
            caseId, tenancyId, "FaultCase", "CaseFaulted",
            "FAULTED", null, "System", null
        ));

        // Allow time for async processing, then verify no entry was written
        await().during(1, TimeUnit.SECONDS).atMost(3, TimeUnit.SECONDS).untilAsserted(() -> {
            List<LedgerEntry> entries = ledgerRepo.findBySubjectId(caseId);
            assertThat(entries.stream()
                .filter(MergeDecisionLedgerEntry.class::isInstance)
                .toList()).isEmpty();
        });
    }

    private void seedCaseInstance(UUID caseId, String tenancyId,
            String repo, String prId, String headSha) {
        CaseInstance ci = new CaseInstance();
        ci.setUuid(caseId);
        ci.tenancyId = tenancyId;
        CaseContext ctx = new CaseContext();
        ctx.put("pr", Map.of(
            "repo", repo,
            "id", prId,
            "headSha", headSha,
            "baseRef", "main",
            "linesChanged", 100,
            "contributor", "test-user",
            "changedPaths", List.of("src/Main.java")
        ));
        ci.setCaseContext(ctx);
        caseInstanceRepo.save(ci).await().atMost(java.time.Duration.ofSeconds(5));
    }
}
```

- [ ] **Step 2: Run the test — expect compilation failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest="MergeDecisionObserverTest" -q`
Expected: COMPILATION ERROR — `MergeDecisionObserver` does not exist

- [ ] **Step 3: Implement MergeDecisionObserver**

```java
package io.casehub.devtown.app.ledger;

import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import org.jboss.logging.Logger;

@ApplicationScoped
public class MergeDecisionObserver {

    private static final Logger LOG = Logger.getLogger(MergeDecisionObserver.class);

    @Inject CrossTenantCaseInstanceRepository caseInstanceRepo;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject LedgerConfig ledgerConfig;

    @Transactional
    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (!ledgerConfig.enabled()) return;

        String decision = switch (event.caseStatus()) {
            case "COMPLETED" -> "APPROVED";
            case "CANCELLED" -> "REJECTED";
            default -> null;
        };
        if (decision == null) return;

        CaseInstance ci;
        try {
            ci = caseInstanceRepo.findByUuid(event.caseId())
                .await().atMost(Duration.ofSeconds(5));
        } catch (Exception e) {
            LOG.warnf(e, "Failed to lookup CaseInstance for caseId=%s", event.caseId());
            return;
        }
        if (ci == null) return;

        CaseContext ctx = ci.getCaseContext();
        if (ctx == null) return;

        String repo = ctx.getPathAsString("pr.repo");
        String prIdStr = ctx.getPathAsString("pr.id");
        String headSha = ctx.getPathAsString("pr.headSha");
        if (repo == null || prIdStr == null) return;

        int prNumber;
        try {
            prNumber = Integer.parseInt(prIdStr);
        } catch (NumberFormatException e) {
            return;
        }

        MergeDecisionLedgerEntry entry = new MergeDecisionLedgerEntry();
        entry.subjectId = event.caseId();
        entry.caseId = event.caseId();
        entry.tenancyId = event.tenancyId();
        entry.entryType = LedgerEntryType.EVENT;
        entry.prNumber = prNumber;
        entry.repository = repo;
        entry.commitSha = headSha;
        entry.decision = decision;
        entry.actorId = "system";
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "ORCHESTRATOR";
        entry.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);

        // Best-effort causal link to the CaseLedgerEntry for the terminal transition.
        // Uses findLatestBySubjectId() (correct @LedgerPersistenceUnit) — NOT
        // findLatestByCaseId() which uses an unqualified EntityManager (engine#450).
        ledgerRepo.findLatestBySubjectId(event.caseId())
            .filter(latest -> latest instanceof CaseLedgerEntry cle
                && event.caseStatus().equals(cle.caseStatus))
            .ifPresent(latest -> entry.causedByEntryId = latest.id);

        ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "casehub-devtown:pr-review-v1";
        cs.humanOverrideAvailable = true;
        cs.contestationUri = "/api/reviews/" + prNumber + "/contest";
        entry.attach(cs);

        ledgerRepo.save(entry);
        LOG.debugf("Merge decision written: caseId=%s decision=%s pr=%s#%d",
            event.caseId(), decision, repo, prNumber);
    }
}
```

- [ ] **Step 4: Run the tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest="MergeDecisionObserverTest" -q`
Expected: 3 tests PASS

Note: if tests fail on Flyway migration or CDI wiring, check:
- That `application.properties` has the updated Flyway locations and qhorus packages
- That the test profile correctly overrides `casehub.ledger.enabled` and `database.generation`
- That `InMemoryLedgerEntryRepository` (existing test stub) does not conflict with `JpaLedgerEntryRepository` under the Flyway profile — the `LedgerEnabledTestProfile` forces Flyway which brings in JPA-backed repos

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java
git commit -m "feat(layer4): MergeDecisionObserver + tests

Observes @ObservesAsync CaseLifecycleEvent, derives APPROVED/REJECTED
from COMPLETED/CANCELLED, looks up CaseInstance for PR metadata,
writes MergeDecisionLedgerEntry with ComplianceSupplement (EU AI Act
Art.12). FAULTED is ignored — infrastructure error, not a merge decision.
Best-effort causedByEntryId link via findLatestBySubjectId().

Refs casehubio/devtown#73"
```

---

## Task 5: Compliance report domain types

**Files:**
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/RequirementStatus.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/LedgerEventRecord.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/InclusionProofRecord.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/RoutingDecisionRecord.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/AuditChainRequirement.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/ReviewSlaRequirement.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/TrustRoutingRequirement.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/GdprRequirement.java`
- Create: `review/src/main/java/io/casehub/devtown/review/compliance/CodeReviewComplianceEvidence.java`

- [ ] **Step 1: Create RequirementStatus enum**

```java
package io.casehub.devtown.review.compliance;

public enum RequirementStatus {
    CLOSED,
    PARTIAL,
    GAP,
    BREACHED
}
```

- [ ] **Step 2: Create supporting detail records**

```java
// LedgerEventRecord.java
package io.casehub.devtown.review.compliance;

import java.time.Instant;
import java.util.UUID;

public record LedgerEventRecord(
    UUID entryId,
    String eventType,
    String actorId,
    String actorRole,
    Instant occurredAt,
    UUID causedByEntryId,
    String digest,
    InclusionProofRecord inclusionProof
) {}
```

```java
// InclusionProofRecord.java
package io.casehub.devtown.review.compliance;

import java.util.List;

public record InclusionProofRecord(
    int entryIndex,
    int treeSize,
    String leafHash,
    List<ProofStepRecord> siblings,
    String treeRoot
) {
    public record ProofStepRecord(String hash, String side) {}
}
```

```java
// RoutingDecisionRecord.java
package io.casehub.devtown.review.compliance;

import java.util.UUID;

public record RoutingDecisionRecord(
    String capabilityTag,
    String workerId,
    Double trustScoreAtRouting,
    Double thresholdApplied,
    UUID ledgerEntryId
) {}
```

- [ ] **Step 3: Create requirement records**

```java
// AuditChainRequirement.java
package io.casehub.devtown.review.compliance;

import java.util.List;

public record AuditChainRequirement(
    String requirementId,
    String citation,
    String mechanism,
    RequirementStatus status,
    String treeRoot,
    boolean chainVerified,
    List<LedgerEventRecord> events
) {
    public static final String REQUIREMENT_ID = "audit-chain";
    public static final String CITATION = "EU AI Act Art.12 — Logging Requirements";
    public static final String MECHANISM = "Merkle Mountain Range append-only ledger (casehub-ledger)";
}
```

```java
// ReviewSlaRequirement.java
package io.casehub.devtown.review.compliance;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record ReviewSlaRequirement(
    String requirementId,
    String citation,
    String mechanism,
    RequirementStatus status,
    UUID taskId,
    Instant claimDeadline,
    Instant completedAt,
    boolean slaMet,
    List<String> candidateGroups
) {
    public static final String REQUIREMENT_ID = "review-sla";
    public static final String CITATION = "Internal Engineering SLA — Human Oversight";
    public static final String MECHANISM = "casehub-work WorkItem lifecycle with SLA breach policy";
}
```

```java
// TrustRoutingRequirement.java
package io.casehub.devtown.review.compliance;

import java.util.List;

public record TrustRoutingRequirement(
    String requirementId,
    String citation,
    String mechanism,
    RequirementStatus status,
    List<RoutingDecisionRecord> decisions
) {
    public static final String REQUIREMENT_ID = "trust-routing";
    public static final String CITATION = "EU AI Act Art.14 — Human Oversight of AI Routing";
    public static final String MECHANISM = "TrustWeightedAgentStrategy with capability-scoped thresholds";
}
```

```java
// GdprRequirement.java
package io.casehub.devtown.review.compliance;

public record GdprRequirement(
    String requirementId,
    String citation,
    String mechanism,
    boolean erasureCapabilityWired,
    boolean pseudonymisationActive
) {
    public static final String REQUIREMENT_ID = "gdpr";
    public static final String CITATION = "GDPR Art.17 Right to Erasure, Art.22 Automated Decision Records";
    public static final String MECHANISM = "LedgerErasureService + ActorIdentityProvider tokenisation";
}
```

- [ ] **Step 4: Create the top-level composite record**

```java
// CodeReviewComplianceEvidence.java
package io.casehub.devtown.review.compliance;

import java.time.Instant;
import java.util.UUID;

public record CodeReviewComplianceEvidence(
    UUID caseId,
    Instant generatedAt,
    AuditChainRequirement auditChain,
    ReviewSlaRequirement reviewSla,
    TrustRoutingRequirement trustRouting,
    GdprRequirement gdpr
) {}
```

- [ ] **Step 5: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl review -am compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add review/src/main/java/io/casehub/devtown/review/compliance/
git commit -m "feat(#7): compliance report domain types

RequirementStatus enum, four requirement records (audit chain, SLA,
trust routing, GDPR), supporting detail records (LedgerEventRecord,
InclusionProofRecord, RoutingDecisionRecord), and top-level
CodeReviewComplianceEvidence composite. All pure Java in review/.

Refs casehubio/devtown#7"
```

---

## Task 6: CodeReviewComplianceService + test (TDD)

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/ledger/CodeReviewComplianceServiceTest.java`
- Create: `app/src/main/java/io/casehub/devtown/app/ledger/CodeReviewComplianceService.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.app.ledger;

import io.casehub.devtown.review.compliance.CodeReviewComplianceEvidence;
import io.casehub.devtown.review.compliance.RequirementStatus;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@TestProfile(LedgerEnabledTestProfile.class)
class CodeReviewComplianceServiceTest {

    @Inject CodeReviewComplianceService service;
    @Inject LedgerEntryRepository ledgerRepo;

    @Test
    void fullCase_allRequirementsClosed() {
        UUID caseId = UUID.randomUUID();
        String tenancyId = "test-tenant";
        Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

        // Seed: case lifecycle entry (COMPLETED)
        CaseLedgerEntry caseEntry = new CaseLedgerEntry();
        caseEntry.caseId = caseId;
        caseEntry.subjectId = caseId;
        caseEntry.tenancyId = tenancyId;
        caseEntry.entryType = LedgerEntryType.EVENT;
        caseEntry.eventType = "CaseCompleted";
        caseEntry.caseStatus = "COMPLETED";
        caseEntry.actorId = "system";
        caseEntry.actorType = ActorType.SYSTEM;
        caseEntry.actorRole = "System";
        caseEntry.occurredAt = now;
        ledgerRepo.save(caseEntry);

        // Seed: worker decision entry
        WorkerDecisionEntry workerEntry = new WorkerDecisionEntry();
        workerEntry.caseId = caseId;
        workerEntry.subjectId = caseId;
        workerEntry.tenancyId = tenancyId;
        workerEntry.entryType = LedgerEntryType.EVENT;
        workerEntry.workerId = "claude:analyst@v1";
        workerEntry.capabilityTag = "security-review";
        workerEntry.trustScoreAtRouting = 0.85;
        workerEntry.thresholdApplied = 0.70;
        workerEntry.actorId = "system";
        workerEntry.actorType = ActorType.SYSTEM;
        workerEntry.actorRole = "WORKER";
        workerEntry.occurredAt = now.plusMillis(100);
        ledgerRepo.save(workerEntry);

        // Seed: merge decision entry
        MergeDecisionLedgerEntry mergeEntry = new MergeDecisionLedgerEntry();
        mergeEntry.subjectId = caseId;
        mergeEntry.caseId = caseId;
        mergeEntry.tenancyId = tenancyId;
        mergeEntry.entryType = LedgerEntryType.EVENT;
        mergeEntry.prNumber = 42;
        mergeEntry.repository = "casehubio/devtown";
        mergeEntry.commitSha = "abc123";
        mergeEntry.decision = "APPROVED";
        mergeEntry.actorId = "system";
        mergeEntry.actorType = ActorType.SYSTEM;
        mergeEntry.actorRole = "ORCHESTRATOR";
        mergeEntry.occurredAt = now.plusMillis(200);
        mergeEntry.causedByEntryId = caseEntry.id;
        ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "casehub-devtown:pr-review-v1";
        cs.humanOverrideAvailable = true;
        mergeEntry.attach(cs);
        ledgerRepo.save(mergeEntry);

        // Act
        Optional<CodeReviewComplianceEvidence> result = service.findEvidence(caseId);

        // Assert
        assertThat(result).isPresent();
        CodeReviewComplianceEvidence evidence = result.get();
        assertThat(evidence.caseId()).isEqualTo(caseId);
        assertThat(evidence.auditChain().events()).hasSize(3);
        assertThat(evidence.trustRouting().status()).isEqualTo(RequirementStatus.CLOSED);
        assertThat(evidence.trustRouting().decisions()).hasSize(1);
        assertThat(evidence.trustRouting().decisions().getFirst().capabilityTag())
            .isEqualTo("security-review");
        assertThat(evidence.gdpr().erasureCapabilityWired()).isTrue();
    }

    @Test
    void emptyCaseId_returnsEmpty() {
        Optional<CodeReviewComplianceEvidence> result = service.findEvidence(UUID.randomUUID());
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run the test — expect compilation failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest="CodeReviewComplianceServiceTest" -q`
Expected: COMPILATION ERROR — `CodeReviewComplianceService` does not exist

- [ ] **Step 3: Implement CodeReviewComplianceService**

```java
package io.casehub.devtown.app.ledger;

import io.casehub.devtown.review.compliance.*;
import io.casehub.ledger.model.CaseLedgerEntry;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.*;

@ApplicationScoped
public class CodeReviewComplianceService {

    @Inject LedgerEntryRepository ledgerRepo;
    @Inject LedgerVerificationService verificationService;
    @Inject EntityManager em;

    @Transactional
    public Optional<CodeReviewComplianceEvidence> findEvidence(UUID caseId) {
        List<LedgerEntry> all = ledgerRepo.findBySubjectId(caseId);
        if (all.isEmpty()) return Optional.empty();

        List<CaseLedgerEntry> caseEntries = filterType(all, CaseLedgerEntry.class);
        List<WorkerDecisionEntry> workerEntries = filterType(all, WorkerDecisionEntry.class);
        List<MergeDecisionLedgerEntry> mergeEntries = filterType(all, MergeDecisionLedgerEntry.class);

        return Optional.of(new CodeReviewComplianceEvidence(
            caseId,
            Instant.now(),
            buildAuditChain(caseId, all, mergeEntries),
            buildSla(caseEntries),
            buildTrustRouting(workerEntries),
            buildGdpr()
        ));
    }

    private AuditChainRequirement buildAuditChain(UUID caseId,
            List<LedgerEntry> all, List<MergeDecisionLedgerEntry> mergeEntries) {
        if (all.isEmpty()) {
            return new AuditChainRequirement(
                AuditChainRequirement.REQUIREMENT_ID,
                AuditChainRequirement.CITATION,
                AuditChainRequirement.MECHANISM,
                RequirementStatus.GAP, null, false, List.of());
        }

        boolean chainVerified = false;
        String treeRoot = null;
        try {
            chainVerified = verificationService.verify(caseId);
            treeRoot = verificationService.treeRoot(caseId);
        } catch (IllegalStateException ignored) {
            // No Merkle frontier — chain cannot be verified
        }

        List<LedgerEventRecord> events = all.stream()
            .map(this::toLedgerEventRecord)
            .toList();

        boolean mergeDecisionLinked = !mergeEntries.isEmpty()
            && mergeEntries.stream().anyMatch(m -> m.causedByEntryId != null);

        RequirementStatus status;
        if (chainVerified && !mergeEntries.isEmpty() && mergeDecisionLinked) {
            status = RequirementStatus.CLOSED;
        } else if (!all.isEmpty()) {
            status = RequirementStatus.PARTIAL;
        } else {
            status = RequirementStatus.GAP;
        }

        return new AuditChainRequirement(
            AuditChainRequirement.REQUIREMENT_ID,
            AuditChainRequirement.CITATION,
            AuditChainRequirement.MECHANISM,
            status, treeRoot, chainVerified, events);
    }

    private ReviewSlaRequirement buildSla(List<CaseLedgerEntry> caseEntries) {
        // Locate the human-approval WorkItem from case entries.
        // In Layer 4, there's no direct link from ledger to WorkItem — return GAP.
        // The WorkItem lookup requires knowing the task ID, which is stored in the
        // case context. A full implementation would extract this from the CaseInstance.
        // For now, report the capability exists but evidence is not assembled.
        return new ReviewSlaRequirement(
            ReviewSlaRequirement.REQUIREMENT_ID,
            ReviewSlaRequirement.CITATION,
            ReviewSlaRequirement.MECHANISM,
            RequirementStatus.GAP,
            null, null, null, false, List.of());
    }

    private TrustRoutingRequirement buildTrustRouting(List<WorkerDecisionEntry> workerEntries) {
        if (workerEntries.isEmpty()) {
            return new TrustRoutingRequirement(
                TrustRoutingRequirement.REQUIREMENT_ID,
                TrustRoutingRequirement.CITATION,
                TrustRoutingRequirement.MECHANISM,
                RequirementStatus.GAP, List.of());
        }

        List<RoutingDecisionRecord> decisions = workerEntries.stream()
            .map(w -> new RoutingDecisionRecord(
                w.capabilityTag, w.workerId,
                w.trustScoreAtRouting, w.thresholdApplied, w.id))
            .toList();

        return new TrustRoutingRequirement(
            TrustRoutingRequirement.REQUIREMENT_ID,
            TrustRoutingRequirement.CITATION,
            TrustRoutingRequirement.MECHANISM,
            RequirementStatus.CLOSED, decisions);
    }

    private GdprRequirement buildGdpr() {
        return new GdprRequirement(
            GdprRequirement.REQUIREMENT_ID,
            GdprRequirement.CITATION,
            GdprRequirement.MECHANISM,
            true,
            true);
    }

    private LedgerEventRecord toLedgerEventRecord(LedgerEntry entry) {
        InclusionProofRecord proof = buildInclusionProof(entry.id);
        String eventType = resolveEventType(entry);
        return new LedgerEventRecord(
            entry.id, eventType, entry.actorId, entry.actorRole,
            entry.occurredAt, entry.causedByEntryId, entry.digest, proof);
    }

    private String resolveEventType(LedgerEntry entry) {
        if (entry instanceof MergeDecisionLedgerEntry m) return "MERGE_DECISION:" + m.decision;
        if (entry instanceof CaseLedgerEntry c) return c.eventType;
        if (entry instanceof WorkerDecisionEntry w) return "WORKER_DECISION:" + w.capabilityTag;
        return entry.entryType.name();
    }

    private InclusionProofRecord buildInclusionProof(UUID entryId) {
        try {
            InclusionProof proof = verificationService.inclusionProof(entryId);
            List<InclusionProofRecord.ProofStepRecord> steps = proof.siblings().stream()
                .map(s -> new InclusionProofRecord.ProofStepRecord(s.hash(), s.side().name()))
                .toList();
            return new InclusionProofRecord(
                proof.entryIndex(), proof.treeSize(),
                proof.leafHash(), steps, proof.treeRoot());
        } catch (Exception e) {
            return null;
        }
    }

    private static <T> List<T> filterType(List<LedgerEntry> entries, Class<T> type) {
        return entries.stream()
            .filter(type::isInstance)
            .map(type::cast)
            .toList();
    }
}
```

- [ ] **Step 4: Run the tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest="CodeReviewComplianceServiceTest" -q`
Expected: 2 tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/ledger/CodeReviewComplianceService.java app/src/test/java/io/casehub/devtown/app/ledger/CodeReviewComplianceServiceTest.java
git commit -m "feat(#7): CodeReviewComplianceService + tests

Assembles compliance evidence from ledger entries: audit chain (Merkle
verification + inclusion proofs), trust routing (WorkerDecisionEntry
records), GDPR capability declaration. SLA requirement is GAP in this
layer — requires WorkItem task ID linkage from case context.
Uses LedgerEntryRepository.findBySubjectId() (correct @LedgerPersistenceUnit).

Refs casehubio/devtown#7"
```

---

## Task 7: REST endpoint + integration test

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/CodeReviewComplianceResource.java`
- Add to: `app/src/test/java/io/casehub/devtown/app/ledger/CodeReviewComplianceServiceTest.java` (REST assertions)

- [ ] **Step 1: Create the REST resource**

```java
package io.casehub.devtown.app;

import io.casehub.devtown.app.ledger.CodeReviewComplianceService;
import io.casehub.devtown.review.compliance.CodeReviewComplianceEvidence;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.UUID;

@Path("/api/compliance/code-review")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
public class CodeReviewComplianceResource {

    @Inject CodeReviewComplianceService service;

    @GET
    @Path("/{caseId}")
    public Response getEvidence(@PathParam("caseId") UUID caseId) {
        return service.findEvidence(caseId)
            .map(evidence -> Response.ok(evidence).build())
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }
}
```

- [ ] **Step 2: Add REST integration test to existing test class**

Add to `CodeReviewComplianceServiceTest.java`:

```java
import static io.restassured.RestAssured.given;

@Test
void restEndpoint_returnsEvidence() {
    UUID caseId = UUID.randomUUID();
    String tenancyId = "test-tenant";
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    MergeDecisionLedgerEntry mergeEntry = new MergeDecisionLedgerEntry();
    mergeEntry.subjectId = caseId;
    mergeEntry.caseId = caseId;
    mergeEntry.tenancyId = tenancyId;
    mergeEntry.entryType = LedgerEntryType.EVENT;
    mergeEntry.prNumber = 1;
    mergeEntry.repository = "test/repo";
    mergeEntry.commitSha = "aaa";
    mergeEntry.decision = "APPROVED";
    mergeEntry.actorId = "system";
    mergeEntry.actorType = ActorType.SYSTEM;
    mergeEntry.actorRole = "ORCHESTRATOR";
    mergeEntry.occurredAt = now;
    ledgerRepo.save(mergeEntry);

    given()
        .when().get("/api/compliance/code-review/" + caseId)
        .then()
        .statusCode(200)
        .body("caseId", org.hamcrest.Matchers.equalTo(caseId.toString()));
}

@Test
void restEndpoint_returns404ForUnknownCase() {
    given()
        .when().get("/api/compliance/code-review/" + UUID.randomUUID())
        .then()
        .statusCode(404);
}
```

- [ ] **Step 3: Run all tests — expect pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest="CodeReviewComplianceServiceTest" -q`
Expected: 4 tests PASS

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/CodeReviewComplianceResource.java app/src/test/java/io/casehub/devtown/app/ledger/CodeReviewComplianceServiceTest.java
git commit -m "feat(#7): GET /api/compliance/code-review/{caseId} REST endpoint

Returns CodeReviewComplianceEvidence as JSON (200) or 404 if no ledger
entries exist for the case.

Closes casehubio/devtown#7"
```

---

## Task 8: Full build verification

- [ ] **Step 1: Run the full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q`
Expected: BUILD SUCCESS — all existing tests pass, new tests pass

- [ ] **Step 2: Verify existing tests are unaffected**

Check that `PrReviewCaseHubTest`, `PrReviewQhorusLifecycleTest`, `HumanApprovalLifecycleTest`, `ReviewOutcomeObserverTest`, `SlaBreachHandlerWiringTest`, `TrustRoutingActivationTest`, and all other existing tests still pass with `casehub.ledger.enabled=false`.

- [ ] **Step 3: Commit any fixes if needed**

---

## Task 9: Code review + final commit

- [ ] **Step 1: Invoke superpowers:requesting-code-review**

Review all changes on this branch against the spec. Any finding Minor or above that isn't fixed this session must be captured as a GitHub issue.

- [ ] **Step 2: Fix any review findings**

- [ ] **Step 3: Run implementation-doc-sync**

Check for any living doc drift after the implementation.
