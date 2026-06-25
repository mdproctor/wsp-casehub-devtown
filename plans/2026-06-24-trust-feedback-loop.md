# Trust Feedback Loop — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove the trust feedback loop is closed end-to-end — SOUND attestations build trust, FLAGGED attestations degrade it, routing shifts automatically.

**Architecture:** Three deliverables on a single repo (devtown). The incident-feedback service, routing infrastructure, and attestation flow are all already shipped. This plan adds an MCP tool entry point, fixes a pre-existing equivalence test failure, writes the E2E closed-loop proof, and files follow-up issues.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, H2 (PostgreSQL mode) for tests, AssertJ, REST-assured

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All commits reference issue: `Refs casehubio/devtown#13`
- MCP tool annotations: `@Tool(name, description)`, `@ToolArg(name, description, required)`
- Test profile for ledger+trust: extends `LedgerEnabledTestProfile` pattern with trust scoring flags
- Existing patterns: `IncidentFeedbackServiceTest` for seeding, `DevtownMcpTools` for MCP tools
- `@TestSecurity(user = "devtown-admin", roles = {"devtown-admin"})` for REST endpoint tests

---

### Task 1: Fix equivalence test failure

**Files:**
- Modify: `review/src/main/java/io/casehub/devtown/review/PrReviewCaseDefinition.java:61`
- Test: `review/src/test/java/io/casehub/devtown/review/PrReviewCaseDefinitionEquivalenceTest.java` (existing)

**Interfaces:**
- Consumes: `Capability.builder()` from `casehub-engine-api`
- Produces: fixed `PrReviewCaseDefinition.build()` — YAML/DSL equivalence restored

- [ ] **Step 1: Run the failing test to confirm the exact failure**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml -pl review -Dtest=PrReviewCaseDefinitionEquivalenceTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`

Expected: FAIL at line 29 — `expected: null but was: "{}"`

The failure is `Capability.getOutputSchema()`: the YAML parser returns `null` for merge-executor's outputSchema (field omitted in YAML), but the DSL passes `"{}"` (empty JSON object string) at line 61 of `PrReviewCaseDefinition.java`.

- [ ] **Step 2: Fix the DSL — change merge-executor outputSchema from "{}" to null**

In `PrReviewCaseDefinition.java` line 61, change:

```java
var mergeExecutorCap  = cap(AgentQualification.MERGE_EXECUTOR, "{ pr: .pr }", "{}");
```

to:

```java
var mergeExecutorCap  = cap(AgentQualification.MERGE_EXECUTOR, "{ pr: .pr }", null);
```

- [ ] **Step 3: Run the test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml -pl review -Dtest=PrReviewCaseDefinitionEquivalenceTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`

Expected: PASS

- [ ] **Step 4: Run full review module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml -pl review --batch-mode`

Expected: All tests pass

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add review/src/main/java/io/casehub/devtown/review/PrReviewCaseDefinition.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "fix: align merge-executor outputSchema with YAML (null, not \"{}\")

Refs casehubio/devtown#13"
```

---

### Task 2: Add `report_incident` MCP tool

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Test: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsIncidentTest.java` (new)

**Interfaces:**
- Consumes: `IncidentFeedbackService.recordFeedback(IncidentFeedback)` → `IncidentFeedbackResult`
- Consumes: `IncidentSeverity`, `IncidentFeedback` from `domain/`
- Produces: `reportIncident()` MCP tool method returning `IncidentFeedbackResult`

- [ ] **Step 1: Write the failing test**

Create `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsIncidentTest.java`:

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.devtown.app.ledger.IncidentFeedbackService;
import io.casehub.devtown.app.ledger.LedgerEnabledTestProfile;
import io.casehub.devtown.app.ledger.MergeDecisionLedgerEntry;
import io.casehub.devtown.domain.IncidentFeedbackResult;
import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestProfile(LedgerEnabledTestProfile.class)
@TestSecurity(user = "devtown-admin", roles = {"devtown-admin"})
class DevtownMcpToolsIncidentTest {

    private static final String TENANT = "test-tenant";

    @Inject DevtownMcpTools mcpTools;
    @Inject LedgerEntryRepository ledgerRepo;

    @Test
    void reportIncident_delegatesToService() {
        UUID caseId = UUID.randomUUID();
        Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);
        seedMergeDecision(caseId, "casehubio/devtown", 100, "APPROVED", now);
        seedWorkerDecision(caseId, "claude:sec@v1", ReviewDomain.SECURITY_REVIEW, now.plusMillis(100));

        IncidentFeedbackResult result = mcpTools.reportIncident(
                "casehubio/devtown", 100, "INC-MCP-1", "HIGH",
                "MCP test incident", ReviewDomain.SECURITY_REVIEW, null);

        assertThat(result.attestationsWritten()).isEqualTo(1);
        assertThat(result.flaggedAgents()).hasSize(1);
        assertThat(result.flaggedAgents().get(0).agentId()).isEqualTo("claude:sec@v1");
    }

    @Test
    void reportIncident_invalidSeverity_throwsIllegalArgument() {
        org.junit.jupiter.api.Assertions.assertThrows(IllegalArgumentException.class,
                () -> mcpTools.reportIncident(
                        "casehubio/devtown", 1, "INC-MCP-2", "INVALID",
                        "bad severity", ReviewDomain.SECURITY_REVIEW, null));
    }

    void seedMergeDecision(UUID caseId, String repo, int prNumber,
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
            ledgerRepo.save(mde, TENANT);
        });
    }

    UUID seedWorkerDecision(UUID caseId, String workerId,
                            String capabilityTag, Instant occurredAt) {
        return QuarkusTransaction.requiringNew().call(() -> {
            WorkerDecisionEntry wde = new WorkerDecisionEntry();
            wde.subjectId = caseId;
            wde.caseId = caseId;
            wde.tenancyId = TENANT;
            wde.entryType = LedgerEntryType.EVENT;
            wde.workerId = workerId;
            wde.actorId = workerId;
            wde.actorType = ActorType.SYSTEM;
            wde.actorRole = "WORKER";
            wde.capabilityTag = capabilityTag;
            wde.occurredAt = occurredAt;
            LedgerEntry saved = ledgerRepo.save(wde, TENANT);
            return saved.id;
        });
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml -pl app -Dtest=DevtownMcpToolsIncidentTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`

Expected: FAIL — `reportIncident` method does not exist on `DevtownMcpTools`

- [ ] **Step 3: Add `reportIncident` to DevtownMcpTools**

Add to `DevtownMcpTools.java` — inject `IncidentFeedbackService`, add the tool method. Place after the existing write tools section (after `forceCompleteCheck`):

```java
@Inject IncidentFeedbackService incidentFeedbackService;
```

And the method:

```java
@Tool(
    name = "report_incident",
    description = "Report a production incident against a merged PR — writes FLAGGED attestation against the reviewer's trust score"
)
public IncidentFeedbackResult reportIncident(
    @ToolArg(name = "repository", description = "GitHub repo slug (e.g. casehubio/devtown)") String repository,
    @ToolArg(name = "prNumber", description = "PR number") int prNumber,
    @ToolArg(name = "incidentId", description = "External incident tracker ID") String incidentId,
    @ToolArg(name = "severity", description = "LOW, MEDIUM, HIGH, or CRITICAL") String severity,
    @ToolArg(name = "description", description = "What went wrong") String description,
    @ToolArg(name = "reviewCapability", description = "Which capability missed the issue (e.g. security-review)") String reviewCapability,
    @ToolArg(name = "caseId", description = "Optional — disambiguate when multiple cases exist for the same PR", required = false) String caseId
) {
    IncidentSeverity sev = IncidentSeverity.valueOf(severity.toUpperCase());
    UUID parsedCaseId = caseId != null ? UUID.fromString(caseId) : null;
    IncidentFeedback feedback = new IncidentFeedback(
            repository, prNumber, incidentId, sev, description, reviewCapability, parsedCaseId);
    return incidentFeedbackService.recordFeedback(feedback);
}
```

Add imports:

```java
import io.casehub.devtown.app.ledger.IncidentFeedbackService;
import io.casehub.devtown.domain.IncidentFeedback;
import io.casehub.devtown.domain.IncidentSeverity;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml -pl app -Dtest=DevtownMcpToolsIncidentTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`

Expected: PASS

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsIncidentTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#13): report_incident MCP tool — dual entry point for incident feedback

Refs casehubio/devtown#13"
```

---

### Task 3: End-to-end closed-loop test

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/TrustFeedbackClosedLoopTest.java`
- Create: `app/src/test/java/io/casehub/devtown/app/TrustScoringTestProfile.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.save()`, `LedgerEntryRepository.saveAttestation()` — for seeding entries and SOUND attestations
- Consumes: `IncidentFeedbackService.recordFeedback()` — for FLAGGED attestation
- Consumes: `TrustGateService.currentScore(actorId, capabilityTag)` — for trust score assertions
- Consumes: `TrustWeightedAgentStrategy.select(AgentRoutingContext, List<AgentCandidate>)` — for routing assertions
- Consumes: `AgentCandidate(workerId, capabilities, runningJobs, health, agentDescriptor)`, `AgentRoutingContext(caseId, capabilityName, caseContext)`
- Consumes: `AgentAssignment.Assigned(workerId)` — for asserting winner
- Produces: `TrustFeedbackClosedLoopTest` — the done-when proof

- [ ] **Step 1: Create the test profile**

Create `app/src/test/java/io/casehub/devtown/app/TrustScoringTestProfile.java`:

```java
package io.casehub.devtown.app;

import io.quarkus.test.junit.QuarkusTestProfile;
import java.util.Map;

public class TrustScoringTestProfile implements QuarkusTestProfile {

    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of(
            "casehub.ledger.enabled", "true",
            "casehub.ledger.trust-score.enabled", "true",
            "casehub.ledger.trust-score.incremental.enabled", "true",
            "casehub.ledger.trust-score.materialization.enabled", "true",
            "quarkus.flyway.qhorus.migrate-at-start", "true",
            "quarkus.hibernate-orm.qhorus.database.generation", "none",
            "quarkus.datasource.qhorus.jdbc.url",
                "jdbc:h2:mem:devtown-trust-test;MODE=PostgreSQL;DB_CLOSE_ON_EXIT=FALSE",
            "quarkus.flyway.qhorus.locations",
                "classpath:db/ledger/migration,classpath:db/engine-ledger/migration,classpath:db/devtown/migration,classpath:db/devtown-test/migration"
        );
    }
}
```

- [ ] **Step 2: Write the E2E test**

Create `app/src/test/java/io/casehub/devtown/app/TrustFeedbackClosedLoopTest.java`:

```java
package io.casehub.devtown.app;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.spi.routing.AgentAssignment;
import io.casehub.api.spi.routing.AgentCandidate;
import io.casehub.api.spi.routing.AgentRoutingContext;
import io.casehub.api.spi.routing.AgentRoutingStrategy;
import io.casehub.devtown.app.ledger.IncidentFeedbackService;
import io.casehub.devtown.app.ledger.MergeDecisionLedgerEntry;
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
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.OptionalDouble;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestProfile(TrustScoringTestProfile.class)
@TestSecurity(user = "devtown-admin", roles = {"devtown-admin"})
class TrustFeedbackClosedLoopTest {

    private static final String TENANT = "test-tenant";
    private static final String AGENT_ALPHA = "claude:alpha@v1";
    private static final String AGENT_BETA = "claude:beta@v1";
    private static final String CAPABILITY = ReviewDomain.SECURITY_REVIEW;
    private static final int MIN_OBSERVATIONS = 10;

    @Inject AgentRoutingStrategy routingStrategy;
    @Inject TrustGateService trustGateService;
    @Inject IncidentFeedbackService incidentFeedbackService;
    @Inject LedgerEntryRepository ledgerRepo;

    @Test
    void closedLoop_soundAttestationsBuildTrust_flaggedDegradesTrust_routingShifts() {
        // Phase 1 — Build trust from SOUND attestations for both agents
        UUID alphaCase = UUID.randomUUID();
        UUID betaCase = UUID.randomUUID();

        buildTrustFromAttestations(AGENT_ALPHA, alphaCase, MIN_OBSERVATIONS + 2);
        buildTrustFromAttestations(AGENT_BETA, betaCase, MIN_OBSERVATIONS + 2);

        // Both agents should now have materialized trust scores
        OptionalDouble alphaScore = trustGateService.currentScore(AGENT_ALPHA, CAPABILITY);
        OptionalDouble betaScore = trustGateService.currentScore(AGENT_BETA, CAPABILITY);
        assertThat(alphaScore).as("Alpha must have a capability score after %d SOUND attestations", MIN_OBSERVATIONS + 2).isPresent();
        assertThat(betaScore).as("Beta must have a capability score after %d SOUND attestations", MIN_OBSERVATIONS + 2).isPresent();

        double alphaScoreBefore = alphaScore.getAsDouble();
        double betaScoreBefore = betaScore.getAsDouble();

        // Phase 2 — Degrade agent-alpha via incident feedback
        UUID incidentCaseId = UUID.randomUUID();
        seedMergeDecisionAndWorkerDecision(incidentCaseId, AGENT_ALPHA, 42);

        IncidentFeedbackResult result = incidentFeedbackService.recordFeedback(
                new IncidentFeedback("casehubio/devtown", 42, "INC-LOOP-1",
                        IncidentSeverity.CRITICAL, "Security flaw in production", CAPABILITY, null));

        assertThat(result.attestationsWritten()).isEqualTo(1);
        assertThat(result.flaggedAgents()).hasSize(1);
        assertThat(result.flaggedAgents().get(0).agentId()).isEqualTo(AGENT_ALPHA);

        // IncrementalTrustUpdateObserver fires synchronously after saveAttestation()
        // (AFTER_SUCCESS transaction phase) — trust score should already be updated
        OptionalDouble alphaScoreAfter = trustGateService.currentScore(AGENT_ALPHA, CAPABILITY);
        assertThat(alphaScoreAfter).isPresent();
        assertThat(alphaScoreAfter.getAsDouble())
                .as("Alpha's trust score must decrease after CRITICAL FLAGGED attestation")
                .isLessThan(alphaScoreBefore);

        // Beta's score should be unchanged
        OptionalDouble betaScoreAfter = trustGateService.currentScore(AGENT_BETA, CAPABILITY);
        assertThat(betaScoreAfter).isPresent();
        assertThat(betaScoreAfter.getAsDouble()).isEqualTo(betaScoreBefore);

        // Phase 3 — Prove routing shift
        AgentRoutingContext ctx = new AgentRoutingContext(UUID.randomUUID(), CAPABILITY, null);
        List<AgentCandidate> candidates = List.of(
                new AgentCandidate(AGENT_ALPHA, Set.of(CAPABILITY), 0, null, null),
                new AgentCandidate(AGENT_BETA, Set.of(CAPABILITY), 0, null, null));

        AgentAssignment assignment = routingStrategy.select(ctx, candidates).await().indefinitely();
        assertThat(assignment).isInstanceOf(AgentAssignment.Assigned.class);
        assertThat(((AgentAssignment.Assigned) assignment).workerId())
                .as("Beta should be selected — alpha's degraded score shifts routing")
                .isEqualTo(AGENT_BETA);
    }

    private void buildTrustFromAttestations(String agentId, UUID caseId, int count) {
        for (int i = 0; i < count; i++) {
            Instant ts = Instant.now().truncatedTo(ChronoUnit.MILLIS).plusMillis(i);

            UUID entryId = QuarkusTransaction.requiringNew().call(() -> {
                WorkerDecisionEntry wde = new WorkerDecisionEntry();
                wde.subjectId = caseId;
                wde.caseId = caseId;
                wde.tenancyId = TENANT;
                wde.entryType = LedgerEntryType.EVENT;
                wde.workerId = agentId;
                wde.actorId = agentId;
                wde.actorType = ActorType.SYSTEM;
                wde.actorRole = "WORKER";
                wde.capabilityTag = CAPABILITY;
                wde.occurredAt = ts;
                LedgerEntry saved = ledgerRepo.save(wde, TENANT);
                return saved.id;
            });

            // Write SOUND attestation — triggers IncrementalTrustUpdateObserver
            QuarkusTransaction.requiringNew().run(() -> {
                LedgerAttestation att = new LedgerAttestation();
                att.ledgerEntryId = entryId;
                att.subjectId = caseId;
                att.attestorId = agentId;
                att.attestorType = ActorType.AGENT;
                att.verdict = AttestationVerdict.SOUND;
                att.confidence = 0.7;
                att.capabilityTag = CAPABILITY;
                att.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);
                ledgerRepo.saveAttestation(att, TENANT);
            });
        }
    }

    private void seedMergeDecisionAndWorkerDecision(UUID caseId, String agentId, int prNumber) {
        Instant now = Instant.now().truncatedTo(ChronoUnit.MILLIS);

        QuarkusTransaction.requiringNew().run(() -> {
            MergeDecisionLedgerEntry mde = new MergeDecisionLedgerEntry();
            mde.subjectId = caseId;
            mde.caseId = caseId;
            mde.tenancyId = TENANT;
            mde.entryType = LedgerEntryType.EVENT;
            mde.prNumber = prNumber;
            mde.repository = "casehubio/devtown";
            mde.decision = "APPROVED";
            mde.actorId = "system";
            mde.actorType = ActorType.SYSTEM;
            mde.actorRole = "ORCHESTRATOR";
            mde.occurredAt = now;
            ledgerRepo.save(mde, TENANT);
        });

        QuarkusTransaction.requiringNew().run(() -> {
            WorkerDecisionEntry wde = new WorkerDecisionEntry();
            wde.subjectId = caseId;
            wde.caseId = caseId;
            wde.tenancyId = TENANT;
            wde.entryType = LedgerEntryType.EVENT;
            wde.workerId = agentId;
            wde.actorId = agentId;
            wde.actorType = ActorType.SYSTEM;
            wde.actorRole = "WORKER";
            wde.capabilityTag = CAPABILITY;
            wde.occurredAt = now.plusMillis(100);
            ledgerRepo.save(wde, TENANT);
        });
    }
}
```

- [ ] **Step 3: Run the test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/devtown/pom.xml -pl app -Dtest=TrustFeedbackClosedLoopTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`

Expected: PASS — the full loop works: SOUND attestations build trust scores, FLAGGED attestation degrades one agent's score, routing shifts to the unflagged agent.

If the test fails, diagnose:
- Trust scores not materialized → check that `TrustScoringTestProfile` config flags are all active
- `AttestationRecordedEvent` not firing → verify `JpaLedgerEntryRepository` (not `NoOpLedgerEntryRepository`) is active via `selected-alternatives`
- Routing doesn't shift → verify `TrustWeightedAgentStrategy` (not `LeastLoadedAgentStrategy`) is active — check CDI priority chain
- `MaterializedTrustScoreSource` returning stale values → it reads from `JpaActorTrustScoreRepository` on every call (no cache), so this indicates `IncrementalTrustUpdateObserver` didn't write

- [ ] **Step 4: Run the full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml --batch-mode`

Expected: BUILD SUCCESS — all modules pass

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/test/java/io/casehub/devtown/app/TrustFeedbackClosedLoopTest.java app/src/test/java/io/casehub/devtown/app/TrustScoringTestProfile.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "test(#13): E2E closed-loop proof — SOUND builds trust, FLAGGED degrades, routing shifts

Proves the done-when: two agents compete, incident degrades the
responsible agent's trust score, routing shifts automatically.
Trust is built from attestations, not seeded scores.

Refs casehubio/devtown#13"
```

---

### Task 4: File issues

**Files:** None — GitHub issues only

- [ ] **Step 1: File qhorus issue — capabilityTag in CommitmentContext**

```
gh issue create --repo casehubio/qhorus --title "feat: add capabilityTag to CommitmentContext for trust-gated attestation" --label "enhancement" --body "$(cat <<'BODY'
LedgerWriteService.writeAttestation() extracts the capability tag from the COMMAND's JSON content via extractCapabilityTag(commandEntry.content), but this happens after calling attestationPolicy.attestationFor(). A trust-gated attestation policy needs the capability tag to call TrustScoreSource.capabilityScore() instead of globalScore().

Fix: move extractCapabilityTag() before the attestationFor() call and add capabilityTag as a field in CommitmentContext.

This unblocks capability-scoped trust-gated attestation verification in devtown (devtown#13 follow-up) and any other harness that needs per-capability trust thresholds on attestation verdicts.

Refs casehubio/devtown#13
BODY
)"
```

- [ ] **Step 2: File devtown issue — TrustGatedAttestationPolicy**

```
gh issue create --repo casehubio/devtown --title "feat: TrustGatedAttestationPolicy — capability-scoped evidential verification" --label "enhancement,epic" --body "$(cat <<'BODY'
Once qhorus ships capabilityTag in CommitmentContext, implement a CommitmentAttestationPolicy override that:
- Uses capabilityScore() (not globalScore()) to gate DONE→SOUND attestations
- High-trust agents get presumption of honesty
- Low-trust agents trigger EvidentialChecker.checkObligation()
- Per-capability thresholds derived from DevtownCapabilityRegistry routing policies

Blocked on: qhorus capabilityTag in CommitmentContext issue.

Design context: specs/2026-06-23-trust-feedback-loop-design.md

Refs casehubio/devtown#13
BODY
)"
```

- [ ] **Step 3: File devtown issue — Trust visibility UI (brainstorm-first)**

```
gh issue create --repo casehubio/devtown --title "feat: trust visibility UI — reviewer scores, routing history, incidents" --label "enhancement" --body "$(cat <<'BODY'
Trust scores by capability, routing history, and incident reports need a UI surface. Data APIs are available: TrustExportService (ledger), TrustGateService query methods, IncidentFeedbackResource.

Brainstorm-first — casehub-pages is new and UI strategy is evolving. This issue captures the requirement, not a solution. Run brainstorming before implementation.

Refs casehubio/devtown#13
BODY
)"
```

- [ ] **Step 4: Commit — no code changes, just confirm issues are filed**

Verify all three issues are created by listing them:

```
gh issue list --repo casehubio/qhorus --limit 3 --search "capabilityTag CommitmentContext"
gh issue list --repo casehubio/devtown --limit 3 --search "TrustGatedAttestationPolicy"
gh issue list --repo casehubio/devtown --limit 3 --search "trust visibility UI"
```
