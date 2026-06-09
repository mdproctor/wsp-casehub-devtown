# Post-Merge Trust Feedback — FLAGGED Attestation on Incident-Linked Review

**Issue:** casehubio/devtown#5
**Branch:** issue-5-trust-feedback-flagged
**Date:** 2026-06-09
**Revision:** 4 (post-review)

---

## Problem

When a PR is merged after an agent's review, and that PR later causes a production incident, the system has no way to feed that outcome back into trust scoring. The agent that missed the issue continues to receive the same routing priority. This breaks the trust feedback loop: production outcomes must influence future routing decisions.

## Solution

A REST endpoint that accepts incident feedback against a merged PR, looks up which agents performed the relevant review capability, and writes a FLAGGED `LedgerAttestation` against each agent's `WorkerDecisionEntry`. The existing `TrustScoreJob` picks up the attestation on its next run and adjusts the agent's capability-scoped trust score downward.

## Design Decisions

### Attestation target: WorkerDecisionEntry

The FLAGGED attestation targets the `WorkerDecisionEntry` (not the `MergeDecisionLedgerEntry` or `CaseLedgerEntry`). The `WorkerDecisionEntry` directly links the agent, the capability tag, and the case — it records the routing decision that assigned a specific agent to a specific review task. This is the precise thing being flagged: the agent's performance of that capability on that case.

### Trust attribution chain — verified

`WorkerDecisionEventCapture` sets `entry.actorId = event.workerId()` — the agent, not the system. `TrustScoreJob` groups `LedgerEntry` records by `actorId`, finds associated `LedgerAttestation` records via `findAttestationsForEntries(entryIds)`, and passes them to `PerActorTrustComputer.computeForActor(actorId, ...)`. The FLAGGED attestation's `ledgerEntryId` points to the `WorkerDecisionEntry` whose `actorId` is the agent. The attribution chain is correct.

### Lookup path: PR identity → caseId → WorkerDecisionEntries

1. If `caseId` provided → use directly, verify an APPROVED `MergeDecisionLedgerEntry` exists
2. If no `caseId` → query `MergeDecisionLedgerEntry` by (repository, prNumber) where `decision=APPROVED`
   - 0 results → 404
   - 1 result → use its `caseId`
   - 2+ results → 409 Conflict with candidate caseIds
3. Query `WorkerDecisionEntry` by caseId via `LedgerEntryRepository.findBySubjectId(caseId)` — returns all `LedgerEntry` subclasses for that subject. Filter by `instanceof WorkerDecisionEntry` and `capabilityTag == reviewCapability`. This is the established pattern from `CodeReviewComplianceService`.
   - 0 matching → 404
4. Idempotency check: for each matching `WorkerDecisionEntry`, query existing attestations to check for a prior submission of the same incident (see Idempotency section)
5. Write a `LedgerAttestation` per matching entry (skipping entries already attested for this incident)

### Query mechanism for (repository, prNumber)

`MergeDecisionLedgerEntry` gains a `@NamedQuery`:

```java
@NamedQuery(
    name = "MergeDecisionLedgerEntry.findApprovedByRepoAndPr",
    query = "SELECT m FROM MergeDecisionLedgerEntry m WHERE m.repository = :repo AND m.prNumber = :prNumber AND m.decision = 'APPROVED'"
)
```

The service uses `@LedgerPersistenceUnit EntityManager` (same qualifier as `TrustScoreJob`) to execute the named query. This is the standard JPA pattern for domain-specific queries on `LedgerEntry` subclasses.

### Flyway V2003 migration

V2002 indexes `case_id` and `tenancy_id` but not `(repository, pr_number)`. The named query joins through `ledger_entry` (JOINED inheritance) and filters on `merge_decision_ledger_entry` columns — without an index this is a full table scan.

```sql
-- V2003__merge_decision_repo_pr_index.sql
CREATE INDEX idx_merge_decision_repo_pr ON merge_decision_ledger_entry(repository, pr_number);
```

### Capability validation: ReviewDomain only

`reviewCapability` is validated against `ReviewDomain` constants only — not `DevtownCapabilityRegistry.capabilities()` which includes `CI_RUNNER`, `MERGE_EXECUTOR`, `PR_APPROVAL`, and `ROUTING_REVIEW`. Incident feedback is about review misses; flagging an agent for poor CI execution or merge performance is semantically wrong.

`ReviewDomain` gains a validation set:

```java
public static final Set<String> REVIEW_CAPABILITIES = Set.of(
    CODE_ANALYSIS, SECURITY_REVIEW, ARCHITECTURE_REVIEW,
    STYLE_REVIEW, TEST_COVERAGE, PERFORMANCE_ANALYSIS
);
```

The service validates `feedback.reviewCapability()` against `ReviewDomain.REVIEW_CAPABILITIES` and returns 400 for unknown values.

### All agents flagged for the capability

When multiple agents performed the same capability (e.g. two agents did `security-review`), all are flagged. An incident means the capability collectively failed — the system cannot determine which specific agent missed the issue.

### Severity-to-confidence mapping

| Severity | Confidence | Rationale |
|----------|-----------|-----------|
| LOW | 0.3 | Minor issue — may not be reviewer's fault |
| MEDIUM | 0.5 | Meaningful miss |
| HIGH | 0.7 | Significant miss — strong signal |
| CRITICAL | 0.9 | Severe miss — very strong negative signal |

Encoded as an enum (`IncidentSeverity`) with the confidence value as a field. Avoids the binary-else enum misclassification bug (GE-20260603-753526).

### Trust dimension: review-thoroughness

All incident-feedback attestations use `DevtownTrustDimension.REVIEW_THOROUGHNESS`. A post-merge incident means the reviewer missed something that should have been caught — that is a thoroughness signal regardless of the specific capability.

### Tenancy

The attestation's tenancy comes from `MergeDecisionLedgerEntry.tenancyId` — the merge decision already carries the correct tenant context. This avoids depending on `CurrentPrincipal.tenancyId()` which requires OIDC to be active. The service reads the tenancyId from the looked-up entry and passes it to `saveAttestation()`.

### Idempotency

An incident is a fact. Recording the same fact twice doubles the trust impact in the Bayesian model — `TrustScoreJob` processes all attestations, so two FLAGGED attestations for the same incident produce a stronger downward adjustment than one. This is incorrect: the signal strength should come from `IncidentSeverity`, not from submission count.

The service is **idempotent** using `findAttestationsByAttestorIdAndCapabilityTag("devtown:incident-feedback", reviewCapability, tenancyId)` — the attestor-scoped query that calls `tokeniseForQuery()` internally (line 162 of `JpaLedgerEntryRepository`). This is tokenisation-proof by design: it handles both `PassThroughActorIdentityProvider` (development) and `InternalActorIdentityProvider` (GDPR production) without coupling to the tokenisation mechanism.

The idempotency check flow:

```
allByAttestor = ledgerRepo.findAttestationsByAttestorIdAndCapabilityTag(
    "devtown:incident-feedback", reviewCapability, tenancyId)

for each WorkerDecisionEntry wde:
    existing = allByAttestor.stream()
        .filter(a -> wde.id.equals(a.ledgerEntryId))
        .filter(a -> a.evidence.startsWith("Incident " + incidentId + ":"))
        .findFirst()
    if existing.isPresent():
        skip — already recorded (include in flaggedAgents, don't persist again)
```

Single query, then client-side filtering by `ledgerEntryId` and evidence prefix. The query returns all incident-feedback attestations for the given capability across the tenant — in practice a small set (production incidents are rare events).

**Why not `findAttestationsByEntryIdAndCapabilityTag`:** That method returns raw stored `attestorId` values without de-tokenisation. Under `InternalActorIdentityProvider`, `attestorId` is a UUID token in the database, so a client-side comparison against `"devtown:incident-feedback"` would always fail. The attestor-scoped query is the ledger's published API for "find attestations I wrote."

The evidence format `"Incident {incidentId}: {description}"` is a contract: the `incidentId` prefix is the deduplication key.

**Foundation improvement (non-blocking):** SYSTEM actors should not be tokenised — they are not natural persons and GDPR pseudonymisation serves no privacy purpose for them. A one-line guard in `JpaLedgerEntryRepository.saveAttestation()` — `if (attestorType != ActorType.SYSTEM)` before tokenising — would make the entry-scoped query work for system attestations and eliminate unnecessary `ActorIdentity` rows. Filed as [casehub-ledger#130](https://github.com/casehubio/ledger/issues/130). The spec's attestor-scoped query approach works correctly regardless of whether this foundation fix ships.

**Response semantics for idempotent calls:** `attestationsWritten` counts only **newly persisted** attestations. In the fully idempotent case (all entries already attested for this incident), `attestationsWritten = 0`. `flaggedAgents` always includes **all** agents for the capability — both newly attested and previously attested — so the caller sees the complete picture regardless of whether this is a first or repeated submission.

### Ledger disabled check

If `LedgerConfig.enabled()` returns false, the endpoint returns 503 Service Unavailable. This is a synchronous endpoint where the caller expects attestations to be written — silently skipping (as async observers do) is the wrong semantics.

---

## REST API

```
POST /api/incident-feedback
Content-Type: application/json
```

Independent resource at its own path — not nested under `/api/reviews` to avoid path overlap with `PrReviewResource`.

### Request body

```json
{
  "repository": "casehubio/devtown",
  "prNumber": 42,
  "incidentId": "INC-789",
  "severity": "HIGH",
  "description": "Timing attack in cryptographic code",
  "reviewCapability": "security-review"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `repository` | String | Yes | GitHub repo slug (e.g. `casehubio/devtown`) |
| `prNumber` | int | Yes | PR number |
| `incidentId` | String | Yes | External incident tracker ID |
| `severity` | String | Yes | `LOW`, `MEDIUM`, `HIGH`, or `CRITICAL` |
| `description` | String | Yes | What went wrong |
| `reviewCapability` | String | Yes | Which capability missed the issue — must be in `ReviewDomain.REVIEW_CAPABILITIES` |
| `caseId` | UUID | No | Disambiguates when multiple cases exist for the same PR |

`caseId` is nullable — Jackson handles absent JSON fields as null in records without any annotation.

### Responses

| Status | When | Body |
|--------|------|------|
| 200 | Attestations written (or already exist for this incident — idempotent) | `IncidentFeedbackResult` |
| 400 | Invalid severity, unknown capability tag, missing required field | Error message |
| 404 | No APPROVED merge decision for (repo, prNumber), or no agent performed the capability | Error message |
| 409 | Multiple APPROVED cases for same PR, no caseId provided | List of candidate caseIds |
| 503 | Ledger disabled | Error message |

### Response body (200)

```json
{
  "caseId": "a1b2c3d4-...",
  "attestationsWritten": 2,
  "flaggedAgents": [
    { "agentId": "claude:analyst@v1", "capabilityTag": "security-review", "attestationId": "..." },
    { "agentId": "claude:security@v2", "capabilityTag": "security-review", "attestationId": "..." }
  ]
}
```

### Auth

`@RolesAllowed("admin")` — inert until `casehub-platform-oidc` is on classpath. Follows the `MemoryAdminResource` pattern.

---

## Domain Model (new types in `domain/`)

### IncidentSeverity

```java
public enum IncidentSeverity {
    LOW(0.3),
    MEDIUM(0.5),
    HIGH(0.7),
    CRITICAL(0.9);

    private final double confidence;
    // constructor + getter
}
```

### IncidentFeedback

```java
public record IncidentFeedback(
    String repository,
    int prNumber,
    String incidentId,
    IncidentSeverity severity,
    String description,
    String reviewCapability,
    UUID caseId              // optional — null when absent from JSON
) {}
```

Pure Java, no annotations — compiles in `domain/` which has no Jackson dependency. Jackson handles absent JSON fields as null in records by default.

### ReviewDomain addition

```java
public static final Set<String> REVIEW_CAPABILITIES = Set.of(
    CODE_ANALYSIS, SECURITY_REVIEW, ARCHITECTURE_REVIEW,
    STYLE_REVIEW, TEST_COVERAGE, PERFORMANCE_ANALYSIS
);
```

### IncidentFeedbackResult / FlaggedAgent

```java
public record IncidentFeedbackResult(
    UUID caseId,
    int attestationsWritten,
    List<FlaggedAgent> flaggedAgents
) {}

public record FlaggedAgent(
    String agentId,
    String capabilityTag,
    UUID attestationId
) {}
```

---

## Attestation Fields

Each `LedgerAttestation` written by the service:

| Field | Value | Rationale |
|-------|-------|-----------|
| `ledgerEntryId` | `WorkerDecisionEntry.id` | The routing decision being flagged |
| `subjectId` | `WorkerDecisionEntry.subjectId` (= caseId) | Standard: attestation subject matches entry subject |
| `attestorId` | `"devtown:incident-feedback"` | System actor identity |
| `attestorType` | `ActorType.SYSTEM` | Automated system, not a person |
| `attestorRole` | `"INCIDENT_FEEDBACK"` | Function of the attestor — consistent with `"ORCHESTRATOR"`, `"WORKER"` patterns |
| `verdict` | `AttestationVerdict.FLAGGED` | Negative outcome signal |
| `capabilityTag` | The `reviewCapability` from the request | Scopes the trust impact to one capability |
| `trustDimension` | `DevtownTrustDimension.REVIEW_THOROUGHNESS` | Missed finding = thoroughness signal |
| `confidence` | From `IncidentSeverity` | Signal strength from incident severity |
| `dimensionScore` | `null` | This is a signal, not a measured score — confidence carries the signal strength |
| `evidence` | `"Incident {incidentId}: {description}"` | Structured format — incidentId prefix is the idempotency key |
| `occurredAt` | *(auto-set by @PrePersist)* | |
| `id` | *(auto-set by @PrePersist)* | Read after persist for `FlaggedAgent.attestationId` |

---

## Module Placement

| File | Module | Rationale |
|------|--------|-----------|
| `IncidentSeverity` | `domain/` | Pure Java enum |
| `IncidentFeedback` | `domain/` | Pure Java record |
| `IncidentFeedbackResult`, `FlaggedAgent` | `domain/` | Pure Java records |
| `ReviewDomain.REVIEW_CAPABILITIES` | `domain/` | Validation set on existing class |
| `IncidentFeedbackService` | `app/ledger/` | Depends on JPA, LedgerEntryRepository, `@LedgerPersistenceUnit EntityManager` |
| `IncidentFeedbackResource` | `app/` | Thin REST at `/api/incident-feedback`, delegates to service |
| `V2003__merge_decision_repo_pr_index.sql` | `app/src/main/resources/db/devtown/migration/` | Index for (repository, pr_number) lookup |

---

## Testing

### Unit tests (domain/, no Quarkus)

- `IncidentSeverity` — all four confidence mappings
- `ReviewDomain.REVIEW_CAPABILITIES` — contains exactly the six analytical capabilities; does not contain CI_RUNNER, MERGE_EXECUTOR, PR_APPROVAL, ROUTING_REVIEW

### Integration tests (@QuarkusTest, app/)

| Test | Verifies |
|------|----------|
| Happy path — single case | 200, attestation written with correct fields (all 11 fields verified) |
| Multiple agents for same capability | Two WorkerDecisionEntries → two attestations |
| Unknown PR | 404 |
| No agent for capability | Valid case, no agent did `security-review` → 404 |
| Ambiguous PR | Two APPROVED cases for same PR → 409 with candidate caseIds |
| Disambiguated with caseId | Ambiguous + explicit caseId → 200 |
| caseId provided, no APPROVED merge decision | Explicit caseId but no APPROVED MergeDecisionLedgerEntry for it → 404 |
| Invalid capability tag | `"made-up"` → 400 |
| Non-review capability tag | `"ci-runner"` → 400 (rejected by `ReviewDomain.REVIEW_CAPABILITIES`) |
| Each severity level | Correct confidence per severity |
| Attestation field correctness | `attestorId`, `attestorType`, `attestorRole`, `verdict`, `trustDimension`, `dimensionScore`, `evidence` format |
| Idempotent — same incident twice | Second submission returns 200 with existing attestations, no new records written |
| Different incidents — same entry | Two different incidentIds against same entry → two attestations (not deduplicated) |
| Ledger disabled | `LedgerConfig.enabled() == false` → 503 |
| Tenancy propagation | Attestation tenancyId matches `MergeDecisionLedgerEntry.tenancyId` |
| attestationId in response | `FlaggedAgent.attestationId` matches the persisted `LedgerAttestation.id` (read after persist) |

Tests use `@TestProfile(LedgerEnabledTestProfile.class)` with H2 via JPA-backed `LedgerEntryRepository`. Pre-seed `MergeDecisionLedgerEntry` and `WorkerDecisionEntry` via `QuarkusTransaction.requiringNew().call(() -> ledgerRepo.save(entry))` — same pattern as `CodeReviewComplianceServiceTest`.

**Test seeding correctness:** `WorkerDecisionEntry.actorId` must be set to the worker/agent ID (matching production's `WorkerDecisionEventCapture` which sets `actorId = workerId`), not to `"system"`. The downstream `TrustScoreJob` attribution depends on this field.

---

## Downstream Effects

Once a FLAGGED attestation is written:

1. **TrustScoreJob** picks it up on next run → updates the agent's Bayesian Beta model for the flagged capability
2. **TrustWeightedAgentStrategy** (casehub-engine-ledger) reads updated scores → routes future work away from agents with degraded capability scores
3. **Trust maturity model** (protocol: `trust-maturity-model.md`) — the agent's score may drop below `RoutingPolicy.threshold`, causing it to be excluded from that capability's routing pool until sustained good performance recovers the score

No changes needed in the foundation — the entire feedback loop is already wired. This feature is the missing signal source.

---

## Out of Scope

- **Automatic incident detection** — this is a manual API; automated detection (e.g. from CI failure signals) is a separate feature
- **Incident retraction** — if an incident is later determined to not be the reviewer's fault, writing a SOUND or ENDORSED attestation against the same entry would counterbalance; no delete mechanism needed
- **Trust decay acceleration on FLAGGED** — referenced in the original issue as a ledger-side feature independent of devtown

---

## Review History

**Revision 1 → 2:** Nine findings from spec review. Accepted: query mechanism specified (@NamedQuery), V2003 migration added, capability validation restricted to ReviewDomain.REVIEW_CAPABILITIES, attestorRole/dimensionScore fields specified, idempotency via evidence prefix matching, REST path moved to /api/incident-feedback, tenancy from MergeDecisionLedgerEntry, LedgerConfig.enabled() → 503, Jackson @JsonInclude. Trust attribution chain verified correct from bytecode (WorkerDecisionEntry.actorId = workerId).

**Revision 2 → 3:** Four findings. (1) Removed @JsonInclude/@Nullable from IncidentFeedback — domain/ has no Jackson dependency, and the annotation controls serialization not deserialization; Jackson handles absent fields as null by default. (2) Fixed test infrastructure — InMemoryLedgerEntryRepository is for the reactive path only; tests use @TestProfile(LedgerEnabledTestProfile.class) with JPA-backed H2, seeding via QuarkusTransaction.requiringNew(). (3) Clarified attestationsWritten semantics: counts newly persisted only; flaggedAgents includes all (new + existing). (4) Specified step 3 query mechanism: findBySubjectId(caseId) + instanceof WorkerDecisionEntry filter. Non-blocking: test seeding must use actorId = workerId (not "system"), named query tenancy note, LAZY init supplement access.

**Revision 3 → 4:** Tokenisation architecture traced end-to-end. The idempotency check was broken: `saveAttestation()` tokenises `attestorId` unconditionally (JpaLedgerEntryRepository line 127-128), so client-side comparison against `"devtown:incident-feedback"` after using the entry-scoped query would always fail under `InternalActorIdentityProvider`. Switched to `findAttestationsByAttestorIdAndCapabilityTag` which calls `tokeniseForQuery()` internally (line 162) — tokenisation-proof by design. Foundation improvement filed: SYSTEM actors should be exempt from tokenisation (casehub-ledger issue — non-blocking, spec works either way).
