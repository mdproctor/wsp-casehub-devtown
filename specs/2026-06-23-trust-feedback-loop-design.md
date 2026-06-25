# Epic 6 — Trust Feedback Loop: Proving the Closed Loop

**Issue:** devtown#13  
**Date:** 2026-06-23  
**Branch:** issue-13-trust-weighted-routing  
**Revision:** 3 (post-review — four corrections: config flag, method name, seed pattern, cache coherence)

---

## What's Already Done

The trust feedback loop is structurally complete. Two prior epics shipped all the machinery:

**Layer 6 — Trust-weighted routing (devtown#57):**
- `TrustWeightedAgentStrategy` activated via `casehub-engine-ledger`
- `DevtownTrustRoutingPolicyProvider` supplies per-capability policies from `DevtownCapabilityRegistry` + YAML
- `WorkerDecisionEventCapture` records routing decisions as `WorkerDecisionEntry`
- `TrustRoutingActivationTest` proves CDI wiring

**Incident feedback (devtown#5, commit `8da3e21`):**
- `IncidentFeedbackService` — writes FLAGGED attestation against reviewer's `WorkerDecisionEntry`
- `IncidentFeedbackResource` — `POST /api/incident-feedback` with `@RolesAllowed(ADMIN)`
- `IncidentSeverity` — severity-to-confidence mapping (LOW=0.3, MEDIUM=0.5, HIGH=0.7, CRITICAL=0.9)
- `ReviewDomain.REVIEW_CAPABILITIES` — capability validation
- `MergeDecisionLedgerEntry` with `@NamedQuery("findApprovedByRepoAndPr")` — durable (repo, prNumber) → caseId lookup
- `V2003__merge_decision_repo_pr_index.sql`
- `IncidentFeedbackServiceTest` — 15 integration tests including idempotency, GDPR tokenisation, severity mapping
- Spec: `docs/specs/2026-06-09-incident-feedback-design.md` (revision 4)

**Positive feedback loop — already live:**
SOUND attestations flow automatically on every review completion:
`DONE` → `LedgerWriteService.record()` → `StoredCommitmentAttestationPolicy` returns SOUND at 0.7 confidence → `writeAttestation()` with `capabilityTag` extracted from COMMAND JSON → `QhorusLedgerEntryRepository.saveAttestation()` → fires `AttestationRecordedEvent` → `IncrementalTrustUpdateObserver.onAttestationRecorded()` → `PerActorTrustComputer.computeForActor()` → trust score updated immediately.

**Foundation capabilities (verified in published JARs, not stale):**
- `CommitmentAttestationPolicy` has 3-arg overload: `attestationFor(MessageType, String, CommitmentContext)`
- `CommitmentContext(correlationId, channelId, channelName, commitmentId)` — already exists
- `StoredCommitmentAttestationPolicy` handles `case RESPONSE → FLAGGED` at `config.attestation().responseConfidence()`
- `EvidentialChecker` is at `io.casehub.qhorus.runtime.audit.EvidentialChecker` `@DefaultBean @ApplicationScoped` — already extracted from examples

---

## What This Epic Delivers

The loop is wired but unproven. The done-when requires an end-to-end test that exercises every link in the chain — from review assignment through attestation accumulation through incident feedback through routing shift. This test IS the deliverable.

### Deliverables

| # | Component | Module | Status |
|---|-----------|--------|--------|
| 0 | Fix equivalence test failure | review | Needed |
| 1 | `report_incident` MCP tool | app/mcp | Needed |
| 2 | End-to-end closed-loop test | app/test | Needed — **primary deliverable** |
| 3 | File issues | — | Needed |

---

## Step 0 — Equivalence Test Fix

`PrReviewCaseDefinitionEquivalenceTest.dslMatchesYaml` fails: `expected: null but was: "{}"`. Upstream engine-api schema serialization change. Mechanical fix — align the DSL-built definition with the current YAML parsing behaviour.

---

## MCP Tool — `report_incident`

Dual entry point alongside the existing REST endpoint. Same `IncidentFeedbackService` underneath.

```
report_incident(repository, prNumber, incidentId, severity, description, reviewCapability, caseId?)
```

In `DevtownMcpTools`. Returns `IncidentFeedbackResult` (caseId, attestationsWritten, flaggedAgents). Maps directly to `IncidentFeedback` record and calls `IncidentFeedbackService.recordFeedback()`.

---

## End-to-End Closed-Loop Test

**Why this test matters:** The routing layer, attestation flow, incident feedback, and trust score computation were built independently across multiple epics and foundation modules. No test has ever exercised them as a single chain. The done-when requires proof that SOUND attestations build trust, FLAGGED attestations degrade it, and the routing layer responds to the shift — automatically, without configuration.

**What the test must prove (and what seeding trust scores would NOT prove):** A test that seeds `ActorTrustScore` rows directly and asserts routing behaviour only proves the routing layer reads scores — already proven in devtown#57's `TrustRoutingActivationTest`. The new thing to prove is that attestations flow through `LedgerWriteService` → `IncrementalTrustUpdateObserver` → materialized trust scores → routing changes.

### Test Structure

```
@QuarkusTest
TrustFeedbackClosedLoopTest
```

**Phase 1 — Build trust from attestations (not seeded scores)**

1. Register two agents (agent-alpha, agent-beta) — both start BOOTSTRAP (no trust history)
2. For each agent, seed N `WorkerDecisionEntry` records directly (same pattern as `IncidentFeedbackServiceTest.seedWorkerDecision()`), then write SOUND `LedgerAttestation` records against each entry via `LedgerEntryRepository.saveAttestation()`:
   - `WorkerDecisionEntry.actorId` = the agent ID (not `"system"`)
   - `LedgerAttestation.capabilityTag` = `"security-review"`
   - `LedgerAttestation.verdict` = `SOUND`, `confidence` = 0.7
   - `saveAttestation()` fires `AttestationRecordedEvent` → `IncrementalTrustUpdateObserver` → trust score materialized
   - Seeding entries directly (not firing `@ObservesAsync WorkerDecisionEvent`) avoids async synchronisation hazards. The event-dispatch path is already tested by `WorkerDecisionEventCapture`'s own tests. This test proves the attestation→trust→routing chain.
3. Repeat enough cycles to cross `minimumObservations` (10 for security-review) for both agents
4. Assert both agents are now QUALIFIED (no longer BOOTSTRAP) with computed capability scores

**Phase 2 — Degrade one agent via incident feedback**

5. Create an APPROVED `MergeDecisionLedgerEntry` for a test PR that agent-alpha reviewed
6. Call `IncidentFeedbackService.recordFeedback()` with `severity=CRITICAL` (confidence 0.9) targeting agent-alpha's `security-review` capability
7. Assert FLAGGED attestation written against agent-alpha's `WorkerDecisionEntry`
8. Assert `IncrementalTrustUpdateObserver` fired — agent-alpha's trust score decreased (not via batch `TrustScoreJob` — the incremental path must fire automatically after `saveAttestation()`)

**Phase 3 — Prove routing shift**

9. Query `TrustGateService.currentScore(actorId, "security-review")` for both agents
10. Assert agent-alpha's score < agent-beta's score (was equal or higher before incident)
11. Call `TrustWeightedAgentStrategy.select()` with both agents as candidates for `security-review`
12. Assert agent-beta is selected (agent-alpha's degraded score shifts the blended ranking)

**Test infrastructure notes:**
- Trust scoring requires three config flags in test properties:
  - `casehub.ledger.trust-score.enabled=true` (parent flag — defaults false; without it, `IncrementalTrustUpdateObserver` silently no-ops)
  - `casehub.ledger.trust-score.incremental.enabled=true`
  - `casehub.ledger.trust-score.materialization.enabled=true`
- `TrustScoreSource` cache coherence: ensure `MaterializedTrustScoreSource` is the active implementation (reads directly from JPA table, not cached). If `CachedTrustScoreSource` is active, incremental writes may not be visible to the routing assertion in Phase 3. Verify the test profile activates the materialized source, or flush the cache between Phase 2 and Phase 3.
- `WorkerDecisionEntry.actorId` must be set to the agent ID (not `"system"`) — this is how `TrustScoreJob`/`IncrementalTrustUpdateObserver` attributes trust
- The test uses H2 with JPA-backed `LedgerEntryRepository` — same pattern as `IncidentFeedbackServiceTest`
- Pre-seed via `QuarkusTransaction.requiringNew()` — same pattern as `CodeReviewComplianceServiceTest`

---

## Known Limitations

### Bootstrap attestation inflation window

New agents in Phase 0 (no trust history) are routed by availability and every DONE claim becomes SOUND at 0.7 confidence — building trust from an assumed-honest baseline. If an agent's first N reviews are poor, those SOUND attestations inflate trust before any correction mechanism (retroactive incident report) can fire. The only correction path is manual incident reports.

This is inherent to the trust maturity model: Phase 0 = Gastown parity. Trust routing is additive — it never degrades baseline capability. The system cannot punish agents for not having history. The window closes automatically once `minimumObservations` is reached and the agent transitions from BOOTSTRAP to QUALIFIED. After that point, poor performance is reflected in lower capability scores.

Named here so it's explicit, not left as an implication of the Phase 0 design.

---

### FLAGGED attestations ignored in capability score computation (discovered during implementation)

`PerActorTrustComputer` computes CAPABILITY scores from SOUND attestations only. FLAGGED attestations are persisted but do not decrease the capability-scoped trust score. This blocks the full negative feedback loop: incident reports write FLAGGED attestations, but the routing layer (which uses `capabilityScore()`) is unaffected.

Tracked: casehubio/ledger#157

---

## Issues Filed

### Qhorus

1. **qhorus#307 — Add `capabilityTag` to `CommitmentContext`** — `LedgerWriteService.writeAttestation()` extracts the capability tag from the COMMAND's JSON content via `extractCapabilityTag(commandEntry.content)`, but this happens **after** calling `attestationPolicy.attestationFor()`. Move the extraction before the policy call and add `capabilityTag` as a field in `CommitmentContext`.

### Ledger

2. **ledger#157 — FLAGGED attestations must affect capability scores** — `PerActorTrustComputer` ignores FLAGGED verdicts for capability score computation. Discovered during E2E testing. Blocks the full trust feedback loop.

### Devtown

3. **devtown#97 — TrustGatedAttestationPolicy** — once qhorus#307 and ledger#157 ship, implement a `CommitmentAttestationPolicy` override that uses `capabilityScore()` (not `globalScore()`) to gate DONE→SOUND attestations. Blocked on both upstream issues.

4. **devtown#98 — Trust visibility UI** — trust scores by capability, routing history, incident reports. Brainstorm-first framing — casehub-pages is new and UI strategy is evolving.

---

## Per-Capability Routing Policies (existing — no changes)

Already implemented in devtown#57. Included for reference:

| Capability | threshold | minObs | borderlineMargin | blendFactor | qualityFloors |
|---|---|---|---|---|---|
| `security-review` | 0.70 | 10 | 0.05 | 0.70 | `review-thoroughness ≥ 0.60` |
| `architecture-review` | 0.65 | 8 | 0.05 | 0.70 | `review-thoroughness ≥ 0.60` |
| `style-review` | 0.50 | 5 | 0.0 | 0.50 | — |
| `merge-executor` | 0.80 | 15 | 0.05 | 0.80 | `precision ≥ 0.70` |

---

## Out of Scope

- **TrustGatedAttestationPolicy** — blocked on CommitmentContext capabilityTag enrichment (qhorus issue). Deferred.
- **Evidential verification integration** — EvidentialChecker exists; integration deferred until TrustGatedAttestationPolicy is implemented.
- **UI/dashboard** — brainstorm-first, casehub-pages TBD.
- **Qhorus trust gate configuration** (devtown#58) — existing tracked issue.

---

## Done-When

Two agents with different trust scores compete for a security review; the higher-trust agent wins. A subsequent production incident reduces the responsible agent's score and shifts all future routing away from it — without human configuration.

The E2E test proves this by exercising every link: COMMAND→DONE→SOUND attestation→trust score materialization→FLAGGED attestation→trust score degradation→routing shift. No seeded scores — trust is built from attestations.
