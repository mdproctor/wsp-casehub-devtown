# Post-Merge Trust Feedback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** REST endpoint that records FLAGGED attestations against agents whose PR reviews missed issues later found in production incidents.

**Architecture:** Thin REST resource at `/api/incident-feedback` delegates to `IncidentFeedbackService` in `app/ledger/`. The service looks up PR → case → worker decision entries via the ledger, then writes `LedgerAttestation` records with idempotency checking via the tokenisation-aware `findAttestationsByAttestorIdAndCapabilityTag` query. Domain types (`IncidentSeverity`, `IncidentFeedback`, result records) are pure Java in `domain/`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger (LedgerAttestation, LedgerEntryRepository), casehub-engine-ledger (WorkerDecisionEntry, CaseLedgerEntryRepository), JPA/H2 for tests

**Spec:** `specs/2026-06-09-incident-feedback-design.md` (revision 4)

---

## File Map

| File | Action | Module | Responsibility |
|------|--------|--------|---------------|
| `domain/src/main/java/.../domain/IncidentSeverity.java` | Create | domain | Enum mapping severity → confidence |
| `domain/src/main/java/.../domain/IncidentFeedback.java` | Create | domain | Request payload record |
| `domain/src/main/java/.../domain/IncidentFeedbackResult.java` | Create | domain | Response record |
| `domain/src/main/java/.../domain/FlaggedAgent.java` | Create | domain | Agent attestation detail record |
| `domain/src/main/java/.../domain/ReviewDomain.java` | Modify | domain | Add `REVIEW_CAPABILITIES` set |
| `domain/src/test/java/.../domain/IncidentSeverityTest.java` | Create | domain | Confidence mapping tests |
| `domain/src/test/java/.../domain/ReviewDomainTest.java` | Modify | domain | Add REVIEW_CAPABILITIES tests |
| `app/src/main/java/.../app/ledger/MergeDecisionLedgerEntry.java` | Modify | app | Add @NamedQuery for repo+pr lookup |
| `app/src/main/java/.../app/ledger/IncidentFeedbackService.java` | Create | app | Lookup + attestation write + idempotency |
| `app/src/main/java/.../app/IncidentFeedbackResource.java` | Create | app | REST endpoint, validation, error mapping |
| `app/src/main/resources/db/devtown/migration/V2003__merge_decision_repo_pr_index.sql` | Create | app | Index for (repository, pr_number) |
| `app/src/test/java/.../app/ledger/IncidentFeedbackServiceTest.java` | Create | app | @QuarkusTest integration tests |

All paths abbreviated with `...` = `io/casehub/devtown`.

---

### Task 1: IncidentSeverity enum and IncidentFeedback record (domain/)

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/IncidentSeverity.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/IncidentFeedback.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/IncidentFeedbackResult.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/FlaggedAgent.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/IncidentSeverityTest.java`

- [ ] **Step 1: Write the IncidentSeverity test**

```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.assertj.core.api.Assertions.assertThat;

class IncidentSeverityTest {

    @ParameterizedTest
    @CsvSource({
        "LOW, 0.3",
        "MEDIUM, 0.5",
        "HIGH, 0.7",
        "CRITICAL, 0.9"
    })
    void confidenceMapping(IncidentSeverity severity, double expected) {
        assertThat(severity.confidence()).isEqualTo(expected);
    }

    @Test
    void allValuesHaveConfidence() {
        for (IncidentSeverity s : IncidentSeverity.values()) {
            assertThat(s.confidence()).isBetween(0.0, 1.0);
        }
    }

    @Test
    void exactlyFourValues() {
        assertThat(IncidentSeverity.values()).hasSize(4);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=IncidentSeverityTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `IncidentSeverity` class not found

- [ ] **Step 3: Implement IncidentSeverity**

```java
package io.casehub.devtown.domain;

public enum IncidentSeverity {

    LOW(0.3),
    MEDIUM(0.5),
    HIGH(0.7),
    CRITICAL(0.9);

    private final double confidence;

    IncidentSeverity(double confidence) {
        this.confidence = confidence;
    }

    public double confidence() {
        return confidence;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=IncidentSeverityTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (3 tests)

- [ ] **Step 5: Create IncidentFeedback record**

```java
package io.casehub.devtown.domain;

import java.util.UUID;

public record IncidentFeedback(
    String repository,
    int prNumber,
    String incidentId,
    IncidentSeverity severity,
    String description,
    String reviewCapability,
    UUID caseId
) {}
```

- [ ] **Step 6: Create IncidentFeedbackResult and FlaggedAgent records**

```java
package io.casehub.devtown.domain;

import java.util.List;
import java.util.UUID;

public record IncidentFeedbackResult(
    UUID caseId,
    int attestationsWritten,
    List<FlaggedAgent> flaggedAgents
) {}
```

```java
package io.casehub.devtown.domain;

import java.util.UUID;

public record FlaggedAgent(
    String agentId,
    String capabilityTag,
    UUID attestationId
) {}
```

- [ ] **Step 7: Compile domain module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl domain -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/IncidentSeverity.java domain/src/main/java/io/casehub/devtown/domain/IncidentFeedback.java domain/src/main/java/io/casehub/devtown/domain/IncidentFeedbackResult.java domain/src/main/java/io/casehub/devtown/domain/FlaggedAgent.java domain/src/test/java/io/casehub/devtown/domain/IncidentSeverityTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#5): domain types — IncidentSeverity, IncidentFeedback, result records

Refs casehubio/devtown#5"
```

---

### Task 2: ReviewDomain.REVIEW_CAPABILITIES validation set

**Files:**
- Modify: `domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java`
- Modify: `domain/src/test/java/io/casehub/devtown/domain/ReviewDomainTest.java`

- [ ] **Step 1: Write the test for REVIEW_CAPABILITIES**

Add to `ReviewDomainTest.java`:

```java
@Test
void reviewCapabilities_containsExactlySixAnalyticalCapabilities() {
    assertThat(ReviewDomain.REVIEW_CAPABILITIES).containsExactlyInAnyOrder(
        ReviewDomain.CODE_ANALYSIS,
        ReviewDomain.SECURITY_REVIEW,
        ReviewDomain.ARCHITECTURE_REVIEW,
        ReviewDomain.STYLE_REVIEW,
        ReviewDomain.TEST_COVERAGE,
        ReviewDomain.PERFORMANCE_ANALYSIS
    );
}

@Test
void reviewCapabilities_doesNotContainNonReviewCapabilities() {
    assertThat(ReviewDomain.REVIEW_CAPABILITIES)
        .doesNotContain(
            AgentQualification.CI_RUNNER,
            AgentQualification.MERGE_EXECUTOR,
            HumanDecision.PR_APPROVAL,
            HumanOversight.ROUTING_REVIEW
        );
}
```

Add necessary imports for `AgentQualification`, `HumanDecision`, `HumanOversight`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ReviewDomainTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `REVIEW_CAPABILITIES` field not found

- [ ] **Step 3: Add REVIEW_CAPABILITIES to ReviewDomain**

Add to `ReviewDomain.java`:

```java
import java.util.Set;

public static final Set<String> REVIEW_CAPABILITIES = Set.of(
    CODE_ANALYSIS, SECURITY_REVIEW, ARCHITECTURE_REVIEW,
    STYLE_REVIEW, TEST_COVERAGE, PERFORMANCE_ANALYSIS
);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ReviewDomainTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (all tests including the two new ones)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java domain/src/test/java/io/casehub/devtown/domain/ReviewDomainTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#5): ReviewDomain.REVIEW_CAPABILITIES validation set

Six analytical capabilities only — excludes CI_RUNNER, MERGE_EXECUTOR,
PR_APPROVAL, ROUTING_REVIEW. Incident feedback validates against this
set, not DevtownCapabilityRegistry.capabilities().

Refs casehubio/devtown#5"
```

---

### Task 3: MergeDecisionLedgerEntry @NamedQuery + V2003 migration

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionLedgerEntry.java`
- Create: `app/src/main/resources/db/devtown/migration/V2003__merge_decision_repo_pr_index.sql`

- [ ] **Step 1: Add the @NamedQuery to MergeDecisionLedgerEntry**

Add `@NamedQuery` annotation to `MergeDecisionLedgerEntry.java`. The entity currently has `@Entity`, `@Table`, and `@DiscriminatorValue`. Add:

```java
import jakarta.persistence.NamedQuery;

@Entity
@Table(name = "merge_decision_ledger_entry", indexes = {
    @Index(name = "idx_merge_decision_entry_case_id", columnList = "case_id"),
    @Index(name = "idx_merge_decision_entry_tenancy_id", columnList = "tenancy_id")
})
@DiscriminatorValue("MERGE_DECISION")
@NamedQuery(
    name = "MergeDecisionLedgerEntry.findApprovedByRepoAndPr",
    query = "SELECT m FROM MergeDecisionLedgerEntry m WHERE m.repository = :repo AND m.prNumber = :prNumber AND m.decision = 'APPROVED'"
)
public class MergeDecisionLedgerEntry extends LedgerEntry {
```

- [ ] **Step 2: Create V2003 migration**

```sql
CREATE INDEX idx_merge_decision_repo_pr ON merge_decision_ledger_entry(repository, pr_number);
```

Write to: `app/src/main/resources/db/devtown/migration/V2003__merge_decision_repo_pr_index.sql`

- [ ] **Step 3: Compile app module to verify annotations are correct**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -am -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionLedgerEntry.java app/src/main/resources/db/devtown/migration/V2003__merge_decision_repo_pr_index.sql
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#5): @NamedQuery for PR lookup + V2003 index migration

MergeDecisionLedgerEntry.findApprovedByRepoAndPr queries by
(repository, prNumber, decision=APPROVED). V2003 adds covering
index on (repository, pr_number).

Refs casehubio/devtown#5"
```

---

### Task 4: IncidentFeedbackService — happy path

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/ledger/IncidentFeedbackService.java`
- Create: `app/src/test/java/io/casehub/devtown/app/ledger/IncidentFeedbackServiceTest.java`

This task implements the core service with the happy path test. Subsequent tasks add error paths and edge cases.

- [ ] **Step 1: Write the happy path test**

```java
package io.casehub.devtown.app.ledger;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.devtown.domain.FlaggedAgent;
import io.casehub.devtown.domain.IncidentFeedback;
import io.casehub.devtown.domain.IncidentFeedbackResult;
import io.casehub.devtown.domain.IncidentSeverity;
import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestProfile(LedgerEnabledTestProfile.class)
class IncidentFeedbackServiceTest {

    private static final String TENANT = "test-tenant";
    private static final String REPO = "casehubio/devtown";

    @Inject IncidentFeedbackService service;
    @Inject LedgerEntryRepository ledgerRepo;

    @Test
    void happyPath_singleAgent_writesOneAttestation() {
        UUID caseId = UUID.randomUUID();
        Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

        seedMergeDecision(caseId, REPO, 42, "APPROVED", now);
        UUID wdeId = seedWorkerDecision(caseId, "claude:analyst@v1",
                ReviewDomain.SECURITY_REVIEW, now.plusMillis(100));

        IncidentFeedback feedback = new IncidentFeedback(
                REPO, 42, "INC-789", IncidentSeverity.HIGH,
                "Timing attack in crypto", ReviewDomain.SECURITY_REVIEW, null);

        IncidentFeedbackResult result = service.recordFeedback(feedback);

        assertThat(result.caseId()).isEqualTo(caseId);
        assertThat(result.attestationsWritten()).isEqualTo(1);
        assertThat(result.flaggedAgents()).hasSize(1);

        FlaggedAgent fa = result.flaggedAgents().get(0);
        assertThat(fa.agentId()).isEqualTo("claude:analyst@v1");
        assertThat(fa.capabilityTag()).isEqualTo(ReviewDomain.SECURITY_REVIEW);
        assertThat(fa.attestationId()).isNotNull();

        // Verify attestation fields in the ledger
        List<LedgerAttestation> attestations = findAttestationsForEntry(wdeId);
        assertThat(attestations).hasSize(1);

        LedgerAttestation att = attestations.get(0);
        assertThat(att.verdict).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(att.confidence).isEqualTo(0.7);
        assertThat(att.capabilityTag).isEqualTo(ReviewDomain.SECURITY_REVIEW);
        assertThat(att.trustDimension).isEqualTo("review-thoroughness");
        assertThat(att.attestorType).isEqualTo(ActorType.SYSTEM);
        assertThat(att.attestorRole).isEqualTo("INCIDENT_FEEDBACK");
        assertThat(att.dimensionScore).isNull();
        assertThat(att.evidence).startsWith("Incident INC-789:");
        assertThat(att.subjectId).isEqualTo(caseId);
    }

    // --- Test data seeding helpers ---

    private void seedMergeDecision(UUID caseId, String repo, int prNumber,
                                    String decision, Instant occurredAt) {
        QuarkusTransaction.requiringNew().run(() -> {
            MergeDecisionLedgerEntry mde = new MergeDecisionLedgerEntry();
            mde.subjectId = caseId;
            mde.caseId = caseId;
            mde.tenancyId = TENANT;
            mde.entryType = LedgerEntryType.EVENT;
            mde.prNumber = prNumber;
            mde.repository = repo;
            mde.decision = decision;
            mde.actorId = "system";
            mde.actorType = ActorType.SYSTEM;
            mde.actorRole = "ORCHESTRATOR";
            mde.occurredAt = occurredAt;
            ledgerRepo.save(mde);
        });
    }

    /** Returns the persisted WorkerDecisionEntry's id (auto-generated). */
    private UUID seedWorkerDecision(UUID caseId, String workerId,
                                     String capabilityTag, Instant occurredAt) {
        return QuarkusTransaction.requiringNew().call(() -> {
            WorkerDecisionEntry wde = new WorkerDecisionEntry();
            wde.subjectId = caseId;
            wde.caseId = caseId;
            wde.tenancyId = TENANT;
            wde.entryType = LedgerEntryType.EVENT;
            wde.workerId = workerId;
            wde.actorId = workerId;  // production pattern — NOT "system"
            wde.actorType = ActorType.SYSTEM;
            wde.actorRole = "WORKER";
            wde.capabilityTag = capabilityTag;
            wde.occurredAt = occurredAt;
            LedgerEntry saved = ledgerRepo.save(wde);
            return saved.id;
        });
    }

    private List<LedgerAttestation> findAttestationsForEntry(UUID entryId) {
        return QuarkusTransaction.requiringNew().call(
                () -> ledgerRepo.findAttestationsByEntryId(entryId));
    }
}
```

**Note on seeding:** `wde.actorId = workerId` matches production's `WorkerDecisionEventCapture` (line 52). The existing `CodeReviewComplianceServiceTest` uses `actorId = "system"` — that is a known test seeding inaccuracy documented in the spec. Our tests use the correct production pattern.

**Note on `ledgerRepo.save()` and `findAttestationsByEntryId()`:** These methods may require a tenancyId parameter depending on the snapshot version. If compilation fails with a missing-argument error, add `TENANT` as the second argument — the tenancy value comes from the seeded entries. The spec documents this: "tenancy comes from `MergeDecisionLedgerEntry.tenancyId`."

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=IncidentFeedbackServiceTest#happyPath_singleAgent_writesOneAttestation -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `IncidentFeedbackService` class not found

- [ ] **Step 3: Implement IncidentFeedbackService**

```java
package io.casehub.devtown.app.ledger;

import io.casehub.devtown.domain.DevtownTrustDimension;
import io.casehub.devtown.domain.FlaggedAgent;
import io.casehub.devtown.domain.IncidentFeedback;
import io.casehub.devtown.domain.IncidentFeedbackResult;
import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import jakarta.ws.rs.BadRequestException;
import jakarta.ws.rs.WebApplicationException;
import jakarta.ws.rs.core.Response;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class IncidentFeedbackService {

    static final String ATTESTOR_ID = "devtown:incident-feedback";
    static final String ATTESTOR_ROLE = "INCIDENT_FEEDBACK";

    @Inject LedgerEntryRepository ledgerRepo;
    @Inject LedgerConfig ledgerConfig;
    @Inject @LedgerPersistenceUnit EntityManager em;

    @Transactional
    public IncidentFeedbackResult recordFeedback(IncidentFeedback feedback) {
        if (!ledgerConfig.enabled()) {
            throw new WebApplicationException("Ledger is disabled", Response.Status.SERVICE_UNAVAILABLE);
        }

        if (!ReviewDomain.REVIEW_CAPABILITIES.contains(feedback.reviewCapability())) {
            throw new BadRequestException("Unknown review capability: " + feedback.reviewCapability());
        }

        UUID caseId = resolveCaseId(feedback);
        String tenancyId = resolveTenancyId(caseId, feedback);

        List<WorkerDecisionEntry> targets = findWorkerDecisions(caseId, feedback.reviewCapability());
        if (targets.isEmpty()) {
            throw new WebApplicationException(
                    "No agent performed " + feedback.reviewCapability() + " on case " + caseId,
                    Response.Status.NOT_FOUND);
        }

        List<LedgerAttestation> existingAttestations =
                ledgerRepo.findAttestationsByAttestorIdAndCapabilityTag(
                        ATTESTOR_ID, feedback.reviewCapability(), tenancyId);

        List<FlaggedAgent> flaggedAgents = new ArrayList<>();
        int newCount = 0;

        for (WorkerDecisionEntry wde : targets) {
            boolean alreadyRecorded = existingAttestations.stream()
                    .anyMatch(a -> wde.id.equals(a.ledgerEntryId)
                            && a.evidence != null
                            && a.evidence.startsWith("Incident " + feedback.incidentId() + ":"));

            if (alreadyRecorded) {
                LedgerAttestation existing = existingAttestations.stream()
                        .filter(a -> wde.id.equals(a.ledgerEntryId)
                                && a.evidence != null
                                && a.evidence.startsWith("Incident " + feedback.incidentId() + ":"))
                        .findFirst()
                        .orElseThrow();
                flaggedAgents.add(new FlaggedAgent(wde.workerId, wde.capabilityTag, existing.id));
            } else {
                LedgerAttestation attestation = buildAttestation(wde, feedback);
                ledgerRepo.saveAttestation(attestation, tenancyId);
                flaggedAgents.add(new FlaggedAgent(wde.workerId, wde.capabilityTag, attestation.id));
                newCount++;
            }
        }

        return new IncidentFeedbackResult(caseId, newCount, flaggedAgents);
    }

    private UUID resolveCaseId(IncidentFeedback feedback) {
        if (feedback.caseId() != null) {
            List<MergeDecisionLedgerEntry> decisions = em
                    .createNamedQuery("MergeDecisionLedgerEntry.findApprovedByRepoAndPr", MergeDecisionLedgerEntry.class)
                    .setParameter("repo", feedback.repository())
                    .setParameter("prNumber", feedback.prNumber())
                    .getResultList()
                    .stream()
                    .filter(m -> feedback.caseId().equals(m.caseId))
                    .toList();
            if (decisions.isEmpty()) {
                throw new WebApplicationException(
                        "No APPROVED merge decision for caseId " + feedback.caseId(),
                        Response.Status.NOT_FOUND);
            }
            return feedback.caseId();
        }

        List<MergeDecisionLedgerEntry> decisions = em
                .createNamedQuery("MergeDecisionLedgerEntry.findApprovedByRepoAndPr", MergeDecisionLedgerEntry.class)
                .setParameter("repo", feedback.repository())
                .setParameter("prNumber", feedback.prNumber())
                .getResultList();

        if (decisions.isEmpty()) {
            throw new WebApplicationException(
                    "No APPROVED merge decision for " + feedback.repository() + "#" + feedback.prNumber(),
                    Response.Status.NOT_FOUND);
        }
        if (decisions.size() > 1) {
            List<UUID> candidates = decisions.stream().map(m -> m.caseId).toList();
            throw new WebApplicationException(
                    Response.status(Response.Status.CONFLICT)
                            .entity(candidates)
                            .build());
        }
        return decisions.get(0).caseId;
    }

    private String resolveTenancyId(UUID caseId, IncidentFeedback feedback) {
        List<MergeDecisionLedgerEntry> decisions = em
                .createNamedQuery("MergeDecisionLedgerEntry.findApprovedByRepoAndPr", MergeDecisionLedgerEntry.class)
                .setParameter("repo", feedback.repository())
                .setParameter("prNumber", feedback.prNumber())
                .getResultList()
                .stream()
                .filter(m -> caseId.equals(m.caseId))
                .toList();
        if (decisions.isEmpty()) {
            throw new IllegalStateException("caseId resolved but no merge decision found — should not happen");
        }
        return decisions.get(0).tenancyId;
    }

    private List<WorkerDecisionEntry> findWorkerDecisions(UUID caseId, String capabilityTag) {
        List<LedgerEntry> entries = ledgerRepo.findBySubjectId(caseId);
        return entries.stream()
                .filter(WorkerDecisionEntry.class::isInstance)
                .map(WorkerDecisionEntry.class::cast)
                .filter(w -> capabilityTag.equals(w.capabilityTag))
                .toList();
    }

    private LedgerAttestation buildAttestation(WorkerDecisionEntry wde, IncidentFeedback feedback) {
        LedgerAttestation att = new LedgerAttestation();
        att.ledgerEntryId = wde.id;
        att.subjectId = wde.subjectId;
        att.attestorId = ATTESTOR_ID;
        att.attestorType = ActorType.SYSTEM;
        att.attestorRole = ATTESTOR_ROLE;
        att.verdict = AttestationVerdict.FLAGGED;
        att.capabilityTag = feedback.reviewCapability();
        att.trustDimension = DevtownTrustDimension.REVIEW_THOROUGHNESS;
        att.confidence = feedback.severity().confidence();
        att.dimensionScore = null;
        att.evidence = "Incident " + feedback.incidentId() + ": " + feedback.description();
        return att;
    }
}
```

**Implementation note:** `resolveTenancyId` re-queries the named query to extract tenancyId. An alternative is to cache the merge decision from `resolveCaseId`, but the duplication keeps the two methods independent and the query is cheap (indexed). During implementation, if you see an opportunity to extract and pass the merge decision object, refactor — but correctness first.

**Note on `ledgerRepo.findBySubjectId(caseId)` and `ledgerRepo.saveAttestation(attestation, tenancyId)`:** These may require a tenancyId parameter depending on the ledger snapshot version. If `findBySubjectId` requires two args, pass `tenancyId` as the second arg. The same applies to `findAttestationsByEntryId` in the test.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=IncidentFeedbackServiceTest#happyPath_singleAgent_writesOneAttestation -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/ledger/IncidentFeedbackService.java app/src/test/java/io/casehub/devtown/app/ledger/IncidentFeedbackServiceTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#5): IncidentFeedbackService — happy path with attestation write

Lookup: PR identity → MergeDecisionLedgerEntry → caseId →
WorkerDecisionEntry → FLAGGED LedgerAttestation. Idempotency via
findAttestationsByAttestorIdAndCapabilityTag (tokenisation-proof).

Refs casehubio/devtown#5"
```

---

### Task 5: Error paths and edge cases

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/ledger/IncidentFeedbackServiceTest.java`

Add remaining test cases. No production code changes expected — the service handles all these paths already.

- [ ] **Step 1: Add error and edge case tests**

Add to `IncidentFeedbackServiceTest.java`:

```java
@Test
void multipleAgents_flagsAll() {
    UUID caseId = UUID.randomUUID();
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    seedMergeDecision(caseId, REPO, 43, "APPROVED", now);
    seedWorkerDecision(caseId, "claude:analyst@v1", ReviewDomain.SECURITY_REVIEW, now.plusMillis(100));
    seedWorkerDecision(caseId, "claude:security@v2", ReviewDomain.SECURITY_REVIEW, now.plusMillis(200));

    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 43, "INC-790", IncidentSeverity.CRITICAL,
            "SQL injection", ReviewDomain.SECURITY_REVIEW, null);

    IncidentFeedbackResult result = service.recordFeedback(feedback);

    assertThat(result.attestationsWritten()).isEqualTo(2);
    assertThat(result.flaggedAgents()).hasSize(2);
    assertThat(result.flaggedAgents())
            .extracting(FlaggedAgent::agentId)
            .containsExactlyInAnyOrder("claude:analyst@v1", "claude:security@v2");
}

@Test
void unknownPr_throws404() {
    IncidentFeedback feedback = new IncidentFeedback(
            "casehubio/nonexistent", 999, "INC-X", IncidentSeverity.LOW,
            "irrelevant", ReviewDomain.CODE_ANALYSIS, null);

    assertThatThrownBy(() -> service.recordFeedback(feedback))
            .isInstanceOf(WebApplicationException.class)
            .extracting(e -> ((WebApplicationException) e).getResponse().getStatus())
            .isEqualTo(404);
}

@Test
void noAgentForCapability_throws404() {
    UUID caseId = UUID.randomUUID();
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    seedMergeDecision(caseId, REPO, 44, "APPROVED", now);
    seedWorkerDecision(caseId, "claude:analyst@v1", ReviewDomain.CODE_ANALYSIS, now.plusMillis(100));

    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 44, "INC-791", IncidentSeverity.HIGH,
            "missed security issue", ReviewDomain.SECURITY_REVIEW, null);

    assertThatThrownBy(() -> service.recordFeedback(feedback))
            .isInstanceOf(WebApplicationException.class)
            .extracting(e -> ((WebApplicationException) e).getResponse().getStatus())
            .isEqualTo(404);
}

@Test
void ambiguousPr_throws409() {
    UUID caseId1 = UUID.randomUUID();
    UUID caseId2 = UUID.randomUUID();
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    seedMergeDecision(caseId1, REPO, 45, "APPROVED", now);
    seedMergeDecision(caseId2, REPO, 45, "APPROVED", now.plusMillis(100));

    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 45, "INC-792", IncidentSeverity.MEDIUM,
            "ambiguous", ReviewDomain.CODE_ANALYSIS, null);

    assertThatThrownBy(() -> service.recordFeedback(feedback))
            .isInstanceOf(WebApplicationException.class)
            .extracting(e -> ((WebApplicationException) e).getResponse().getStatus())
            .isEqualTo(409);
}

@Test
void disambiguatedWithCaseId_resolvesCorrectly() {
    UUID caseId1 = UUID.randomUUID();
    UUID caseId2 = UUID.randomUUID();
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    seedMergeDecision(caseId1, REPO, 46, "APPROVED", now);
    seedMergeDecision(caseId2, REPO, 46, "APPROVED", now.plusMillis(100));
    seedWorkerDecision(caseId2, "claude:analyst@v1", ReviewDomain.CODE_ANALYSIS, now.plusMillis(200));

    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 46, "INC-793", IncidentSeverity.LOW,
            "minor issue", ReviewDomain.CODE_ANALYSIS, caseId2);

    IncidentFeedbackResult result = service.recordFeedback(feedback);

    assertThat(result.caseId()).isEqualTo(caseId2);
    assertThat(result.attestationsWritten()).isEqualTo(1);
}

@Test
void caseIdProvided_noApprovedDecision_throws404() {
    UUID caseId = UUID.randomUUID();

    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 999, "INC-X", IncidentSeverity.LOW,
            "irrelevant", ReviewDomain.CODE_ANALYSIS, caseId);

    assertThatThrownBy(() -> service.recordFeedback(feedback))
            .isInstanceOf(WebApplicationException.class)
            .extracting(e -> ((WebApplicationException) e).getResponse().getStatus())
            .isEqualTo(404);
}

@Test
void invalidCapability_throws400() {
    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 1, "INC-X", IncidentSeverity.LOW,
            "irrelevant", "made-up-capability", null);

    assertThatThrownBy(() -> service.recordFeedback(feedback))
            .isInstanceOf(BadRequestException.class);
}

@Test
void nonReviewCapability_throws400() {
    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 1, "INC-X", IncidentSeverity.LOW,
            "irrelevant", "ci-runner", null);

    assertThatThrownBy(() -> service.recordFeedback(feedback))
            .isInstanceOf(BadRequestException.class);
}

@Test
void idempotent_sameIncidentTwice_noNewAttestations() {
    UUID caseId = UUID.randomUUID();
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    seedMergeDecision(caseId, REPO, 47, "APPROVED", now);
    seedWorkerDecision(caseId, "claude:analyst@v1", ReviewDomain.CODE_ANALYSIS, now.plusMillis(100));

    IncidentFeedback feedback = new IncidentFeedback(
            REPO, 47, "INC-794", IncidentSeverity.HIGH,
            "same issue", ReviewDomain.CODE_ANALYSIS, null);

    IncidentFeedbackResult first = service.recordFeedback(feedback);
    assertThat(first.attestationsWritten()).isEqualTo(1);

    IncidentFeedbackResult second = service.recordFeedback(feedback);
    assertThat(second.attestationsWritten()).isEqualTo(0);
    assertThat(second.flaggedAgents()).hasSize(1);
    assertThat(second.flaggedAgents().get(0).attestationId())
            .isEqualTo(first.flaggedAgents().get(0).attestationId());
}

@Test
void differentIncidents_sameEntry_bothRecorded() {
    UUID caseId = UUID.randomUUID();
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    seedMergeDecision(caseId, REPO, 48, "APPROVED", now);
    seedWorkerDecision(caseId, "claude:analyst@v1", ReviewDomain.SECURITY_REVIEW, now.plusMillis(100));

    IncidentFeedback first = new IncidentFeedback(
            REPO, 48, "INC-A", IncidentSeverity.HIGH,
            "first issue", ReviewDomain.SECURITY_REVIEW, null);
    IncidentFeedback second = new IncidentFeedback(
            REPO, 48, "INC-B", IncidentSeverity.CRITICAL,
            "second issue", ReviewDomain.SECURITY_REVIEW, null);

    IncidentFeedbackResult r1 = service.recordFeedback(first);
    IncidentFeedbackResult r2 = service.recordFeedback(second);

    assertThat(r1.attestationsWritten()).isEqualTo(1);
    assertThat(r2.attestationsWritten()).isEqualTo(1);
    assertThat(r1.flaggedAgents().get(0).attestationId())
            .isNotEqualTo(r2.flaggedAgents().get(0).attestationId());
}

@Test
void eachSeverity_correctConfidence() {
    for (IncidentSeverity severity : IncidentSeverity.values()) {
        UUID caseId = UUID.randomUUID();
        Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);
        int prNumber = 100 + severity.ordinal();

        seedMergeDecision(caseId, REPO, prNumber, "APPROVED", now);
        UUID wdeId = seedWorkerDecision(caseId, "agent-" + severity.name(),
                ReviewDomain.CODE_ANALYSIS, now.plusMillis(100));

        IncidentFeedback feedback = new IncidentFeedback(
                REPO, prNumber, "INC-SEV-" + severity.name(), severity,
                "severity test", ReviewDomain.CODE_ANALYSIS, null);

        service.recordFeedback(feedback);

        List<LedgerAttestation> attestations = findAttestationsForEntry(wdeId);
        assertThat(attestations).hasSize(1);
        assertThat(attestations.get(0).confidence).isEqualTo(severity.confidence());
    }
}
```

Add necessary imports: `assertThatThrownBy` from AssertJ, `BadRequestException` from JAX-RS.

- [ ] **Step 2: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=IncidentFeedbackServiceTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (all tests)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/ledger/IncidentFeedbackServiceTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test(#5): error paths, edge cases, idempotency, severity mapping

12 test cases: happy path, multi-agent, 404s (unknown PR, no agent,
invalid caseId), 409 ambiguity, caseId disambiguation, invalid/non-review
capability 400s, idempotency, different incidents same entry, severity
confidence mapping.

Refs casehubio/devtown#5"
```

---

### Task 6: IncidentFeedbackResource — REST endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/IncidentFeedbackResource.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/ledger/IncidentFeedbackServiceTest.java` (add REST tests)

- [ ] **Step 1: Write REST endpoint tests**

Add to `IncidentFeedbackServiceTest.java`:

```java
@Test
void restEndpoint_happyPath() {
    UUID caseId = UUID.randomUUID();
    Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    seedMergeDecision(caseId, REPO, 50, "APPROVED", now);
    seedWorkerDecision(caseId, "claude:analyst@v1", ReviewDomain.CODE_ANALYSIS, now.plusMillis(100));

    given()
        .contentType("application/json")
        .body("""
            {
              "repository": "casehubio/devtown",
              "prNumber": 50,
              "incidentId": "INC-REST-1",
              "severity": "HIGH",
              "description": "REST test",
              "reviewCapability": "code-analysis"
            }
            """)
    .when()
        .post("/api/incident-feedback")
    .then()
        .statusCode(200)
        .body("attestationsWritten", org.hamcrest.Matchers.equalTo(1))
        .body("flaggedAgents.size()", org.hamcrest.Matchers.equalTo(1))
        .body("flaggedAgents[0].agentId", org.hamcrest.Matchers.equalTo("claude:analyst@v1"));
}

@Test
void restEndpoint_unknownPr_returns404() {
    given()
        .contentType("application/json")
        .body("""
            {
              "repository": "casehubio/nonexistent",
              "prNumber": 999,
              "incidentId": "INC-REST-2",
              "severity": "LOW",
              "description": "not found",
              "reviewCapability": "code-analysis"
            }
            """)
    .when()
        .post("/api/incident-feedback")
    .then()
        .statusCode(404);
}

@Test
void restEndpoint_invalidCapability_returns400() {
    given()
        .contentType("application/json")
        .body("""
            {
              "repository": "casehubio/devtown",
              "prNumber": 1,
              "incidentId": "INC-REST-3",
              "severity": "LOW",
              "description": "invalid",
              "reviewCapability": "ci-runner"
            }
            """)
    .when()
        .post("/api/incident-feedback")
    .then()
        .statusCode(400);
}
```

Add import: `import static io.restassured.RestAssured.given;`

- [ ] **Step 2: Run REST tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=IncidentFeedbackServiceTest#restEndpoint_happyPath -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — 404 (no resource at `/api/incident-feedback`)

- [ ] **Step 3: Implement IncidentFeedbackResource**

```java
package io.casehub.devtown.app;

import io.casehub.devtown.app.ledger.IncidentFeedbackService;
import io.casehub.devtown.domain.IncidentFeedback;
import io.casehub.devtown.domain.IncidentFeedbackResult;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/api/incident-feedback")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
@RolesAllowed("admin")
@ApplicationScoped
public class IncidentFeedbackResource {

    @Inject
    IncidentFeedbackService service;

    @POST
    public IncidentFeedbackResult recordFeedback(IncidentFeedback feedback) {
        return service.recordFeedback(feedback);
    }
}
```

- [ ] **Step 4: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=IncidentFeedbackServiceTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (all tests including REST endpoint tests)

**Note on `@RolesAllowed`:** This is inert without `casehub-platform-oidc` on classpath. The REST tests will NOT be blocked by the annotation. If tests fail with 403, the test profile needs `quarkus.http.auth.proactive=false` — check `MemoryAdminResource` tests for the pattern.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/IncidentFeedbackResource.java app/src/test/java/io/casehub/devtown/app/ledger/IncidentFeedbackServiceTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#5): IncidentFeedbackResource — POST /api/incident-feedback

Thin REST resource with @RolesAllowed(\"admin\"). REST endpoint tests
for happy path, 404, and 400.

Refs casehubio/devtown#5"
```

---

### Task 7: Full build and final verification

**Files:** None — verification only

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS with all tests passing

- [ ] **Step 2: Verify test count**

Check the Surefire report for `IncidentFeedbackServiceTest` — should show 15 tests (1 happy path + 11 error/edge + 3 REST endpoint).
Check `IncidentSeverityTest` — 3 tests.
Check `ReviewDomainTest` — original tests + 2 new.

- [ ] **Step 3: Commit any fixes needed from full build**

If the full build reveals issues (e.g. tenancy parameter mismatches, import conflicts), fix them and commit with a descriptive message referencing #5.
