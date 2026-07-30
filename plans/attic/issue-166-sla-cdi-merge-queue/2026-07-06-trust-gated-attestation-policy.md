# Trust-Gated Attestation Policy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #97 — feat: TrustGatedAttestationPolicy — capability-scoped evidential verification
**Issue group:** #97

**Goal:** Implement a trust-gated attestation policy in engine-ledger that modulates attestation confidence based on the agent's capability trust score, then prove it end-to-end in devtown.

**Architecture:** One new class (`TrustGatedAttestationPolicy`) in `casehub-engine-ledger`'s `routing` package, implementing `CommitmentAttestationPolicy` from qhorus-api. Displaces the `@DefaultBean` `StoredCommitmentAttestationPolicy` via `@Alternative @Priority(1)`. Devtown picks it up transitively and proves activation in an integration test.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mockito (engine unit tests), AssertJ, `@QuarkusTest` (devtown integration)

## Global Constraints

- engine-ledger already has Jandex configured — no build plugin changes needed
- `@Alternative @Priority(1)` activation pattern (not `@DefaultBean` — this class is not a subclass of a library `@Alternative`, it competes with a `@DefaultBean`)
- Phase classification order must mirror `TrustCandidateClassifier.classifyOne()`: bootstrap → borderline → passesThresholdCheck → below_threshold
- `CapabilityTag.GLOBAL = "*"` (from `io.casehub.ledger.api.model.CapabilityTag`) — fallback trigger
- `MIN_CONFIDENCE_FLOOR = 0.05` — guards against `OutcomeRecord` rejecting `confidence ≤ 0.0`
- Protocol PP-20260530-8725fa does NOT apply — `TrustGatedAttestationPolicy` is not a subclass of a library `@Alternative`; it implements `CommitmentAttestationPolicy` (a pure SPI interface)
- No Flyway migrations — no persistence in this feature
- **Ordering constraint:** engine-ledger changes (Tasks 1–3) must be done last, after devtown work is ready. Pause before executing engine tasks and wait for user go-ahead.

---

## File Structure

### engine-ledger (casehub-engine repo)

| Action | File | Responsibility |
|--------|------|----------------|
| Modify | `ledger/pom.xml` | Add `casehub-qhorus-api` compile dependency |
| Create | `ledger/src/main/java/io/casehub/ledger/routing/TrustGatedAttestationPolicy.java` | The policy — implements `CommitmentAttestationPolicy`, injects `TrustScoreSource` + `TrustRoutingPolicyProvider` |
| Create | `ledger/src/test/java/io/casehub/ledger/routing/TrustGatedAttestationPolicyTest.java` | Mock-based unit tests covering all phase/verdict/fallback combinations |

### devtown

| Action | File | Responsibility |
|--------|------|----------------|
| Create | `app/src/test/java/io/casehub/devtown/app/trust/TrustGatedAttestationPolicyActivationTest.java` | `@QuarkusTest` proving CDI displacement and confidence modulation |

---

### Task 1: File engine issue and update devtown#97

**Files:**
- None (GitHub API only)

**Interfaces:**
- Produces: engine issue number for cross-referencing

- [ ] **Step 1: File engine issue**

```bash
gh issue create --repo casehubio/engine \
  --title "feat: TrustGatedAttestationPolicy — capability-scoped attestation confidence" \
  --body "Add TrustGatedAttestationPolicy to engine-ledger's routing package.

Implements CommitmentAttestationPolicy (qhorus-api) with trust-score-modulated confidence:
- BOOTSTRAP: base confidence (0.7)
- QUALIFIED: boosted proportionally to distance above threshold
- BORDERLINE: base confidence (uncertainty zone)
- BELOW_THRESHOLD: scaled down by capability score, floored at 0.05

Displaces StoredCommitmentAttestationPolicy (@DefaultBean) via @Alternative @Priority(1).
Activates automatically when casehub-engine-ledger is on the classpath.

Adds casehub-qhorus-api as a compile dependency to engine-ledger (narrow coupling: CommitmentAttestationPolicy SPI, CommitmentContext record, MessageType enum).

Design spec: casehubio/devtown specs/2026-07-06-trust-gated-attestation-policy-design.md
Refs casehubio/devtown#97" \
  --label enhancement
```

- [ ] **Step 2: Update devtown#97 to reference engine issue**

```bash
gh issue comment 97 --repo casehubio/devtown \
  --body "Engine-side implementation tracked at casehubio/engine#<N>.
Devtown scope reduced to: integration test proving CDI activation and confidence modulation.
Both blockers resolved: qhorus#307 (capabilityTag in CommitmentContext) ✅, ledger#157 (FLAGGED in capability scores) ✅."
```

---

### Task 2: devtown integration test (TDD — write before engine implementation)

This test will fail until engine-ledger publishes the SNAPSHOT with `TrustGatedAttestationPolicy`. That's intentional — TDD.

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/trust/TrustGatedAttestationPolicyActivationTest.java`

**Interfaces:**
- Consumes: `CommitmentAttestationPolicy` (CDI-injected — will be `TrustGatedAttestationPolicy` once engine-ledger is on classpath)
- Consumes: `TrustScoreSource` (for seeding trust scores)
- Consumes: `DevtownTrustRoutingPolicyProvider` (existing — provides per-capability thresholds)
- Consumes: `LedgerEntryRepository` (for seeding WorkerDecisionEntry records)

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.app.trust;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.routing.TrustGatedAttestationPolicy;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.CommitmentAttestationPolicy;
import io.casehub.qhorus.api.spi.CommitmentContext;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestProfile(TrustScoringTestProfile.class)
@TestSecurity(user = "devtown-admin", roles = {"devtown-admin"})
class TrustGatedAttestationPolicyActivationTest {

    private static final String TENANT = "test-tenant";
    private static final String HIGH_TRUST_AGENT = "claude:trusted-reviewer@v1";
    private static final String LOW_TRUST_AGENT = "claude:untrusted-reviewer@v1";
    private static final String SECURITY_REVIEW = "security-review";

    @Inject CommitmentAttestationPolicy attestationPolicy;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject TrustScoreSource trustScoreSource;
    @Inject io.casehub.ledger.runtime.service.TrustScoreJob trustScoreJob;

    @Test
    void policyIsTrustGated() {
        assertThat(attestationPolicy).isInstanceOf(TrustGatedAttestationPolicy.class);
    }

    @Test
    void highTrustAgentDoneProducesHigherConfidence() {
        // Seed 15 SOUND attestations for high-trust agent (crosses minimumObservations=10)
        seedAttestations(HIGH_TRUST_AGENT, SECURITY_REVIEW, 15, AttestationVerdict.SOUND, 0.7);
        // Seed 15 mixed attestations for low-trust agent (below threshold)
        seedAttestations(LOW_TRUST_AGENT, SECURITY_REVIEW, 10, AttestationVerdict.SOUND, 0.7);
        seedAttestations(LOW_TRUST_AGENT, SECURITY_REVIEW, 5, AttestationVerdict.FLAGGED, 0.6);

        // Recompute trust scores
        QuarkusTransaction.requiringNew().run(() -> trustScoreJob.recomputeAll());

        // Both agents send DONE for security-review
        var context = new CommitmentContext(
                UUID.randomUUID().toString(), UUID.randomUUID(), "test-channel",
                UUID.randomUUID(), SECURITY_REVIEW);

        Optional<CommitmentAttestationPolicy.AttestationOutcome> highTrustResult =
                attestationPolicy.attestationFor(MessageType.DONE, HIGH_TRUST_AGENT, context);
        Optional<CommitmentAttestationPolicy.AttestationOutcome> lowTrustResult =
                attestationPolicy.attestationFor(MessageType.DONE, LOW_TRUST_AGENT, context);

        assertThat(highTrustResult).isPresent();
        assertThat(lowTrustResult).isPresent();
        assertThat(highTrustResult.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
        assertThat(lowTrustResult.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
        // High-trust agent's DONE should carry MORE confidence than low-trust agent's DONE
        assertThat(highTrustResult.get().confidence())
                .isGreaterThan(lowTrustResult.get().confidence());
        // Both should have agent attestor identity
        assertThat(highTrustResult.get().attestorId()).isEqualTo(HIGH_TRUST_AGENT);
        assertThat(highTrustResult.get().attestorType()).isEqualTo(ActorType.AGENT);
    }

    @Test
    void failureIsFlaggedRegardlessOfTrust() {
        var context = new CommitmentContext(
                UUID.randomUUID().toString(), UUID.randomUUID(), "test-channel",
                UUID.randomUUID(), SECURITY_REVIEW);

        Optional<CommitmentAttestationPolicy.AttestationOutcome> result =
                attestationPolicy.attestationFor(MessageType.FAILURE, HIGH_TRUST_AGENT, context);

        assertThat(result).isPresent();
        assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(result.get().confidence()).isEqualTo(0.6);
        assertThat(result.get().attestorId()).isEqualTo("system");
        assertThat(result.get().attestorType()).isEqualTo(ActorType.SYSTEM);
    }

    @Test
    void nullContextFallsBackToBaseConfidence() {
        Optional<CommitmentAttestationPolicy.AttestationOutcome> result =
                attestationPolicy.attestationFor(MessageType.DONE, HIGH_TRUST_AGENT, null);

        assertThat(result).isPresent();
        assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
        assertThat(result.get().confidence()).isEqualTo(0.7);
    }

    @Test
    void nonDischargeTypeReturnsEmpty() {
        var context = new CommitmentContext(
                UUID.randomUUID().toString(), UUID.randomUUID(), "test-channel",
                UUID.randomUUID(), SECURITY_REVIEW);

        Optional<CommitmentAttestationPolicy.AttestationOutcome> result =
                attestationPolicy.attestationFor(MessageType.QUERY, HIGH_TRUST_AGENT, context);

        assertThat(result).isEmpty();
    }

    private void seedAttestations(String agentId, String capabilityTag, int count,
                                   AttestationVerdict verdict, double confidence) {
        QuarkusTransaction.requiringNew().run(() -> {
            for (int i = 0; i < count; i++) {
                WorkerDecisionEntry entry = new WorkerDecisionEntry();
                entry.tenancyId = TENANT;
                entry.caseId = UUID.randomUUID();
                entry.actorId = agentId;
                entry.actorType = ActorType.AGENT;
                entry.capabilityTag = capabilityTag;
                entry.entryType = LedgerEntryType.EVENT;
                entry.timestamp = Instant.now();
                entry.strategyId = "trust-weighted";
                entry.rationale = "test seed";
                ledgerRepo.save(entry);

                LedgerAttestation att = new io.casehub.ledger.runtime.model.RuntimeLedgerAttestation();
                att.parentEntry = entry;
                att.attestorId = verdict == AttestationVerdict.SOUND ? agentId : "system";
                att.attestorType = verdict == AttestationVerdict.SOUND ? ActorType.AGENT : ActorType.SYSTEM;
                att.verdict = verdict;
                att.confidence = confidence;
                att.capabilityTag = capabilityTag;
                att.timestamp = Instant.now();
                ledgerRepo.saveAttestation(att);
            }
        });
    }
}
```

- [ ] **Step 2: Verify test does not compile (TrustGatedAttestationPolicy not yet available)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -am -q 2>&1 | head -20
```

Expected: Compilation error — `TrustGatedAttestationPolicy` class not found. This confirms the test will pass only after the engine-ledger implementation is published.

- [ ] **Step 3: Comment out the import and `isInstanceOf` check temporarily**

Comment out the `TrustGatedAttestationPolicy` import and the `policyIsTrustGated` test body so the remaining tests can compile. Mark with `// TODO: uncomment after engine-ledger SNAPSHOT published`. The other tests use only `CommitmentAttestationPolicy` (the SPI) and will fail at runtime assertion level, not compilation.

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/devtown/app/trust/TrustGatedAttestationPolicyActivationTest.java
git commit -m "test(#97): add TrustGatedAttestationPolicy activation test — red until engine-ledger ships

Refs #97"
```

---

### Task 3: engine-ledger — add qhorus-api dependency

**USER GO-AHEAD REQUIRED BEFORE THIS TASK.**

**Files:**
- Modify: `ledger/pom.xml` (in casehub-engine repo at `/Users/mdproctor/claude/casehub/engine`)

**Interfaces:**
- Produces: `CommitmentAttestationPolicy`, `CommitmentContext`, `MessageType` available on engine-ledger classpath

- [ ] **Step 1: Add casehub-qhorus-api dependency to engine-ledger pom.xml**

Add after the `casehub-ledger` dependency:

```xml
<!-- CommitmentAttestationPolicy SPI — TrustGatedAttestationPolicy implements it -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-qhorus-api</artifactId>
</dependency>
```

Version is managed by `casehub-engine-parent` BOM (inherits from `casehub-parent`).

- [ ] **Step 2: Verify build still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl ledger -am -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add ledger/pom.xml
git -C /Users/mdproctor/claude/casehub/engine commit -m "build(engine#<N>): add casehub-qhorus-api dependency to engine-ledger

Narrow coupling: CommitmentAttestationPolicy SPI, CommitmentContext record, MessageType enum.
Required for TrustGatedAttestationPolicy.

Refs casehubio/engine#<N>"
```

---

### Task 4: engine-ledger — TrustGatedAttestationPolicy unit tests (TDD red)

**Files:**
- Create: `ledger/src/test/java/io/casehub/ledger/routing/TrustGatedAttestationPolicyTest.java`

**Interfaces:**
- Consumes: `CommitmentAttestationPolicy` SPI (qhorus-api)
- Consumes: `CommitmentContext` record (qhorus-api)
- Consumes: `MessageType` enum (qhorus-api)
- Consumes: `TrustScoreSource` (ledger-api, mocked)
- Consumes: `TrustRoutingPolicyProvider` (engine-api, mocked)
- Consumes: `TrustRoutingPolicy` record (engine-api)
- Consumes: `AttestationVerdict` enum (ledger-api)
- Consumes: `CapabilityTag.GLOBAL` constant (ledger-api)
- Consumes: `ActorType` enum (platform-api)

- [ ] **Step 1: Write the full test class**

```java
package io.casehub.ledger.routing;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.CommitmentAttestationPolicy.AttestationOutcome;
import io.casehub.qhorus.api.spi.CommitmentContext;
import java.util.Map;
import java.util.Optional;
import java.util.OptionalDouble;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class TrustGatedAttestationPolicyTest {

  private TrustScoreSource source;
  private TrustRoutingPolicyProvider policyProvider;
  private TrustGatedAttestationPolicy policy;

  // threshold=0.7, minimumObservations=10, borderlineMargin=0.1
  private static final TrustRoutingPolicy ROUTING_POLICY =
      new TrustRoutingPolicy(0.7, 10, 0.1, 0.6, Map.of(), false, null);
  private static final String CAP = "security-review";
  private static final String ACTOR = "claude:reviewer@v1";
  private static final CommitmentContext CTX = new CommitmentContext(
      UUID.randomUUID().toString(), UUID.randomUUID(), "test-channel",
      UUID.randomUUID(), CAP);

  @BeforeEach
  void setUp() {
    source = mock(TrustScoreSource.class);
    policyProvider = mock(TrustRoutingPolicyProvider.class);
    policy = new TrustGatedAttestationPolicy(source, policyProvider);

    when(policyProvider.forCapability(any())).thenReturn(ROUTING_POLICY);
    when(source.capabilityScore(any(), any())).thenReturn(OptionalDouble.empty());
    when(source.decisionCount(any(), any())).thenReturn(0);
  }

  // ---- DONE + BOOTSTRAP ----

  @Test
  void done_bootstrap_noHistory_soundAtBaseConfidence() {
    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
    assertThat(result.get().confidence()).isEqualTo(0.7);
    assertThat(result.get().attestorId()).isEqualTo(ACTOR);
    assertThat(result.get().attestorType()).isEqualTo(ActorType.AGENT);
  }

  @Test
  void done_bootstrap_belowMinObservations_soundAtBaseConfidence() {
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.9));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(5); // below 10

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().confidence()).isEqualTo(0.7);
  }

  // ---- DONE + QUALIFIED ----

  @Test
  void done_qualified_highTrust_soundAtBoostedConfidence() {
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.9));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(15);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
    // 0.7 * (1.0 + (0.9 - 0.7)) = 0.7 * 1.2 = 0.84
    assertThat(result.get().confidence()).isCloseTo(0.84, within(0.001));
  }

  @Test
  void done_qualified_justPastBorderline_soundAtModestBoost() {
    // 0.81 with threshold=0.7, margin=0.1: isBorderline(0.81) = |0.81-0.7|=0.11 > 0.1 = false
    // passesThresholdCheck(0.81) = 0.81 >= 0.7 && !isBorderline = true → QUALIFIED
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.81));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(15);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    // 0.7 * (1.0 + (0.81 - 0.7)) = 0.7 * 1.11 = 0.777
    assertThat(result.get().confidence()).isCloseTo(0.777, within(0.001));
  }

  @Test
  void done_qualified_cappedAtOne() {
    // Score 1.0: 0.7 * (1.0 + (1.0 - 0.7)) = 0.7 * 1.3 = 0.91 — below cap
    // Score with very high trust: policy threshold 0.1 → 0.7 * (1.0 + 0.89) = 1.323 → capped
    TrustRoutingPolicy lowThreshold =
        new TrustRoutingPolicy(0.1, 10, 0.01, 0.6, Map.of(), false, null);
    when(policyProvider.forCapability(CAP)).thenReturn(lowThreshold);
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.99));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(15);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().confidence()).isEqualTo(1.0);
  }

  // ---- DONE + BORDERLINE ----

  @Test
  void done_borderlineAbove_soundAtBaseConfidence() {
    // 0.75 with threshold=0.7, margin=0.1: |0.75-0.7|=0.05 <= 0.1 → borderline
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.75));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(15);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().confidence()).isEqualTo(0.7);
  }

  @Test
  void done_borderlineBelow_soundAtBaseConfidence() {
    // 0.65 with threshold=0.7, margin=0.1: |0.65-0.7|=0.05 <= 0.1 → borderline
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.65));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(15);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().confidence()).isEqualTo(0.7);
  }

  // ---- DONE + BELOW_THRESHOLD ----

  @Test
  void done_belowThreshold_soundAtScaledConfidence() {
    // 0.5 with threshold=0.7, margin=0.1: |0.5-0.7|=0.2 > 0.1 → not borderline
    // 0.5 < 0.7 → not passesThresholdCheck → BELOW_THRESHOLD
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.5));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(15);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
    // max(0.05, 0.7 * 0.5) = max(0.05, 0.35) = 0.35
    assertThat(result.get().confidence()).isCloseTo(0.35, within(0.001));
  }

  @Test
  void done_belowThreshold_nearZero_flooredAtMinConfidence() {
    when(source.capabilityScore(ACTOR, CAP)).thenReturn(OptionalDouble.of(0.01));
    when(source.decisionCount(ACTOR, CAP)).thenReturn(15);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, CTX);

    assertThat(result).isPresent();
    // max(0.05, 0.7 * 0.01) = max(0.05, 0.007) = 0.05
    assertThat(result.get().confidence()).isEqualTo(0.05);
  }

  // ---- DONE + fallback (null/empty/GLOBAL capabilityTag) ----

  @Test
  void done_nullContext_soundAtBaseConfidence() {
    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, null);

    assertThat(result).isPresent();
    assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.SOUND);
    assertThat(result.get().confidence()).isEqualTo(0.7);
    assertThat(result.get().attestorId()).isEqualTo(ACTOR);
    assertThat(result.get().attestorType()).isEqualTo(ActorType.AGENT);
  }

  @Test
  void done_nullCapabilityTag_soundAtBaseConfidence() {
    CommitmentContext nullCapCtx = new CommitmentContext(
        UUID.randomUUID().toString(), UUID.randomUUID(), "ch", UUID.randomUUID(), null);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, nullCapCtx);

    assertThat(result).isPresent();
    assertThat(result.get().confidence()).isEqualTo(0.7);
  }

  @Test
  void done_emptyCapabilityTag_soundAtBaseConfidence() {
    CommitmentContext emptyCapCtx = new CommitmentContext(
        UUID.randomUUID().toString(), UUID.randomUUID(), "ch", UUID.randomUUID(), "");

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, emptyCapCtx);

    assertThat(result).isPresent();
    assertThat(result.get().confidence()).isEqualTo(0.7);
  }

  @Test
  void done_globalCapabilityTag_soundAtBaseConfidence() {
    CommitmentContext globalCtx = new CommitmentContext(
        UUID.randomUUID().toString(), UUID.randomUUID(), "ch", UUID.randomUUID(),
        CapabilityTag.GLOBAL);

    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DONE, ACTOR, globalCtx);

    assertThat(result).isPresent();
    assertThat(result.get().confidence()).isEqualTo(0.7);
  }

  // ---- FAILURE ----

  @Test
  void failure_flaggedAtBaseConfidence() {
    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.FAILURE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.FLAGGED);
    assertThat(result.get().confidence()).isEqualTo(0.6);
    assertThat(result.get().attestorId()).isEqualTo("system");
    assertThat(result.get().attestorType()).isEqualTo(ActorType.SYSTEM);
  }

  // ---- DECLINE ----

  @Test
  void decline_flaggedAtBaseConfidence() {
    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.DECLINE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.FLAGGED);
    assertThat(result.get().confidence()).isEqualTo(0.4);
    assertThat(result.get().attestorId()).isEqualTo("system");
    assertThat(result.get().attestorType()).isEqualTo(ActorType.SYSTEM);
  }

  // ---- RESPONSE (wrong vocabulary) ----

  @Test
  void response_flaggedAtLowConfidence() {
    Optional<AttestationOutcome> result =
        policy.attestationFor(MessageType.RESPONSE, ACTOR, CTX);

    assertThat(result).isPresent();
    assertThat(result.get().verdict()).isEqualTo(AttestationVerdict.FLAGGED);
    assertThat(result.get().confidence()).isEqualTo(0.3);
    assertThat(result.get().attestorId()).isEqualTo("system");
    assertThat(result.get().attestorType()).isEqualTo(ActorType.SYSTEM);
  }

  // ---- Non-discharge types ----

  @Test
  void query_returnsEmpty() {
    assertThat(policy.attestationFor(MessageType.QUERY, ACTOR, CTX)).isEmpty();
  }

  @Test
  void status_returnsEmpty() {
    assertThat(policy.attestationFor(MessageType.STATUS, ACTOR, CTX)).isEmpty();
  }

  @Test
  void command_returnsEmpty() {
    assertThat(policy.attestationFor(MessageType.COMMAND, ACTOR, CTX)).isEmpty();
  }

  @Test
  void event_returnsEmpty() {
    assertThat(policy.attestationFor(MessageType.EVENT, ACTOR, CTX)).isEmpty();
  }

  @Test
  void handoff_returnsEmpty() {
    assertThat(policy.attestationFor(MessageType.HANDOFF, ACTOR, CTX)).isEmpty();
  }
}
```

- [ ] **Step 2: Run test to verify it fails (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger -Dtest=TrustGatedAttestationPolicyTest -q 2>&1 | head -10
```

Expected: FAIL — `TrustGatedAttestationPolicy` not found

---

### Task 5: engine-ledger — TrustGatedAttestationPolicy implementation (TDD green)

**Files:**
- Create: `ledger/src/main/java/io/casehub/ledger/routing/TrustGatedAttestationPolicy.java`

**Interfaces:**
- Implements: `CommitmentAttestationPolicy.attestationFor(MessageType, String, CommitmentContext)`
- Consumes: `TrustScoreSource.capabilityScore(String, String)`, `TrustScoreSource.decisionCount(String, String)`
- Consumes: `TrustRoutingPolicyProvider.forCapability(String)`
- Consumes: `TrustRoutingPolicy.isBootstrap(int)`, `TrustRoutingPolicy.isBorderline(double)`, `TrustRoutingPolicy.passesThresholdCheck(double)`, `TrustRoutingPolicy.threshold()`

- [ ] **Step 1: Write the implementation**

```java
package io.casehub.ledger.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.CapabilityTag;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.spi.CommitmentAttestationPolicy;
import io.casehub.qhorus.api.spi.CommitmentContext;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.util.Optional;
import java.util.OptionalDouble;

@Alternative
@Priority(1)
@ApplicationScoped
public class TrustGatedAttestationPolicy implements CommitmentAttestationPolicy {

  static final double BASE_DONE_CONFIDENCE = 0.7;
  static final double BASE_FAILURE_CONFIDENCE = 0.6;
  static final double BASE_DECLINE_CONFIDENCE = 0.4;
  static final double BASE_RESPONSE_CONFIDENCE = 0.3;
  static final double MIN_CONFIDENCE_FLOOR = 0.05;

  private final TrustScoreSource source;
  private final TrustRoutingPolicyProvider policyProvider;

  @Inject
  public TrustGatedAttestationPolicy(
      final TrustScoreSource source, final TrustRoutingPolicyProvider policyProvider) {
    this.source = source;
    this.policyProvider = policyProvider;
  }

  @Override
  public Optional<AttestationOutcome> attestationFor(
      final MessageType terminalType, final String resolvedActorId,
      final CommitmentContext context) {

    return switch (terminalType) {
      case DONE -> Optional.of(attestDone(resolvedActorId, context));
      case FAILURE -> Optional.of(new AttestationOutcome(
          AttestationVerdict.FLAGGED, BASE_FAILURE_CONFIDENCE, "system", ActorType.SYSTEM));
      case DECLINE -> Optional.of(new AttestationOutcome(
          AttestationVerdict.FLAGGED, BASE_DECLINE_CONFIDENCE, "system", ActorType.SYSTEM));
      case RESPONSE -> Optional.of(new AttestationOutcome(
          AttestationVerdict.FLAGGED, BASE_RESPONSE_CONFIDENCE, "system", ActorType.SYSTEM));
      default -> Optional.empty();
    };
  }

  private AttestationOutcome attestDone(final String resolvedActorId,
                                         final CommitmentContext context) {
    if (context == null || !hasCapabilityTag(context)) {
      return soundAtConfidence(resolvedActorId, BASE_DONE_CONFIDENCE);
    }

    final String capabilityTag = context.capabilityTag();
    final OptionalDouble capScore = source.capabilityScore(resolvedActorId, capabilityTag);
    final int decCount = source.decisionCount(resolvedActorId, capabilityTag);
    final TrustRoutingPolicy routingPolicy = policyProvider.forCapability(capabilityTag);

    if (capScore.isEmpty() || routingPolicy.isBootstrap(decCount)) {
      return soundAtConfidence(resolvedActorId, BASE_DONE_CONFIDENCE);
    }

    final double score = capScore.getAsDouble();

    if (routingPolicy.isBorderline(score)) {
      return soundAtConfidence(resolvedActorId, BASE_DONE_CONFIDENCE);
    }

    if (routingPolicy.passesThresholdCheck(score)) {
      final double boosted = BASE_DONE_CONFIDENCE * (1.0 + (score - routingPolicy.threshold()));
      return soundAtConfidence(resolvedActorId, Math.min(1.0, boosted));
    }

    // BELOW_THRESHOLD
    final double scaled = BASE_DONE_CONFIDENCE * score;
    return soundAtConfidence(resolvedActorId, Math.max(MIN_CONFIDENCE_FLOOR, scaled));
  }

  private static boolean hasCapabilityTag(final CommitmentContext context) {
    final String tag = context.capabilityTag();
    return tag != null && !tag.isEmpty() && !CapabilityTag.GLOBAL.equals(tag);
  }

  private static AttestationOutcome soundAtConfidence(final String actorId,
                                                       final double confidence) {
    return new AttestationOutcome(AttestationVerdict.SOUND, confidence, actorId, ActorType.AGENT);
  }
}
```

- [ ] **Step 2: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger -Dtest=TrustGatedAttestationPolicyTest -q
```

Expected: All 20 tests PASS

- [ ] **Step 3: Run full engine-ledger test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger -q
```

Expected: All existing tests still pass

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add ledger/src/main/java/io/casehub/ledger/routing/TrustGatedAttestationPolicy.java ledger/src/test/java/io/casehub/ledger/routing/TrustGatedAttestationPolicyTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#<N>): TrustGatedAttestationPolicy — capability-scoped attestation confidence

Implements CommitmentAttestationPolicy with trust-score-modulated confidence:
- BOOTSTRAP/BORDERLINE: base confidence (0.7)
- QUALIFIED: boosted proportionally to distance above threshold
- BELOW_THRESHOLD: scaled by capability score, floored at 0.05
- FAILURE/DECLINE: FLAGGED (trust-independent)
- Non-discharge types: Optional.empty()

Displaces StoredCommitmentAttestationPolicy via @Alternative @Priority(1).
Activates automatically when casehub-engine-ledger is on the classpath.

Refs casehubio/engine#<N>"
```

---

### Task 6: Publish engine SNAPSHOT and make devtown tests green

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/trust/TrustGatedAttestationPolicyActivationTest.java` (uncomment the import and `isInstanceOf` check)

- [ ] **Step 1: Build and install engine-ledger SNAPSHOT locally**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl ledger -am -q -DskipTests -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

- [ ] **Step 2: Uncomment TrustGatedAttestationPolicy import and test in devtown**

Remove the TODO comment and uncomment the `TrustGatedAttestationPolicy` import and `policyIsTrustGated` test body.

- [ ] **Step 3: Run devtown integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TrustGatedAttestationPolicyActivationTest -q
```

Expected: All 4 tests PASS

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/devtown/app/trust/TrustGatedAttestationPolicyActivationTest.java
git commit -m "test(#97): TrustGatedAttestationPolicy activation test green

Proves CDI displacement, confidence modulation by trust phase, and
FAILURE/null-context fallback behaviour in a @QuarkusTest.

Refs #97"
```

---

## Self-Review Checklist

**1. Spec coverage:**
- ✅ `TrustGatedAttestationPolicy` class in engine-ledger `routing` package
- ✅ `@Alternative @Priority(1)` activation
- ✅ `TrustScoreSource` + `TrustRoutingPolicyProvider` injection
- ✅ All 5 phases (BOOTSTRAP, QUALIFIED, BORDERLINE, BELOW_THRESHOLD, fallback)
- ✅ Confidence modulation formulas match spec
- ✅ `MIN_CONFIDENCE_FLOOR = 0.05`
- ✅ `CapabilityTag.GLOBAL` fallback
- ✅ Non-discharge types return `Optional.empty()`
- ✅ All 4 `AttestationOutcome` fields (verdict, confidence, attestorId, attestorType)
- ✅ Constants match `StoredCommitmentAttestationPolicy` defaults
- ✅ Unit tests in engine-ledger
- ✅ Integration test in devtown
- ✅ Engine issue filed, devtown#97 updated
- ✅ Phase classification order mirrors `TrustCandidateClassifier.classifyOne()`
- ✅ qhorus-api dependency added to engine-ledger pom.xml

**2. Placeholder scan:** No TBD/TODO/placeholder in any step (except the temporary compilation workaround in Task 2 Step 3, which is explicitly removed in Task 6 Step 2).

**3. Type consistency:** All method signatures, class names, and constant names are consistent across tasks.
