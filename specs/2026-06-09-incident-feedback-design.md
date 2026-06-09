# Post-Merge Trust Feedback — FLAGGED Attestation on Incident-Linked Review

**Issue:** casehubio/devtown#5
**Branch:** issue-5-trust-feedback-flagged
**Date:** 2026-06-09

---

## Problem

When a PR is merged after an agent's review, and that PR later causes a production incident, the system has no way to feed that outcome back into trust scoring. The agent that missed the issue continues to receive the same routing priority. This breaks the trust feedback loop: production outcomes must influence future routing decisions.

## Solution

A REST endpoint that accepts incident feedback against a merged PR, looks up which agents performed the relevant review capability, and writes a FLAGGED `LedgerAttestation` against each agent's `WorkerDecisionEntry`. The existing `TrustScoreJob` picks up the attestation on its next run and adjusts the agent's capability-scoped trust score downward.

## Design Decisions

### Attestation target: WorkerDecisionEntry

The FLAGGED attestation targets the `WorkerDecisionEntry` (not the `MergeDecisionLedgerEntry` or `CaseLedgerEntry`). The `WorkerDecisionEntry` directly links the agent, the capability tag, and the case — it records the routing decision that assigned a specific agent to a specific review task. This is the precise thing being flagged: the agent's performance of that capability on that case.

### Lookup path: PR identity → caseId → WorkerDecisionEntries

1. If `caseId` provided → use directly, verify an APPROVED `MergeDecisionLedgerEntry` exists
2. If no `caseId` → query `MergeDecisionLedgerEntry` by (repository, prNumber) where `decision=APPROVED`
   - 0 results → 404
   - 1 result → use its `caseId`
   - 2+ results → 409 Conflict with candidate caseIds
3. Query `WorkerDecisionEntry` by caseId, filter by `capabilityTag = reviewCapability`
   - 0 matching → 404
4. Write a `LedgerAttestation` per matching entry

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

---

## REST API

```
POST /api/reviews/incident-feedback
Content-Type: application/json
```

### Request body

```json
{
  "repository": "casehubio/devtown",
  "prNumber": 42,
  "incidentId": "INC-789",
  "severity": "HIGH",
  "description": "Timing attack in cryptographic code",
  "reviewCapability": "security-review",
  "caseId": null
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `repository` | String | Yes | GitHub repo slug (e.g. `casehubio/devtown`) |
| `prNumber` | int | Yes | PR number |
| `incidentId` | String | Yes | External incident tracker ID |
| `severity` | String | Yes | `LOW`, `MEDIUM`, `HIGH`, or `CRITICAL` |
| `description` | String | Yes | What went wrong |
| `reviewCapability` | String | Yes | Which capability missed the issue (must match a `ReviewDomain` constant) |
| `caseId` | UUID | No | Disambiguates when multiple cases exist for the same PR |

### Responses

| Status | When | Body |
|--------|------|------|
| 200 | Attestations written | `IncidentFeedbackResult` |
| 400 | Invalid severity, unknown capability tag, missing required field | Error message |
| 404 | No APPROVED merge decision for (repo, prNumber), or no agent performed the capability | Error message |
| 409 | Multiple APPROVED cases for same PR, no caseId provided | List of candidate caseIds |

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
    UUID caseId              // optional
) {}
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

| Field | Value |
|-------|-------|
| `ledgerEntryId` | `WorkerDecisionEntry.id` |
| `subjectId` | `WorkerDecisionEntry.subjectId` (= caseId) |
| `attestorId` | `"devtown:incident-feedback"` |
| `attestorType` | `ActorType.SYSTEM` |
| `verdict` | `AttestationVerdict.FLAGGED` |
| `capabilityTag` | The `reviewCapability` from the request |
| `trustDimension` | `DevtownTrustDimension.REVIEW_THOROUGHNESS` |
| `confidence` | From `IncidentSeverity` |
| `evidence` | `"Incident {incidentId}: {description}"` |

---

## Module Placement

| File | Module | Rationale |
|------|--------|-----------|
| `IncidentSeverity` | `domain/` | Pure Java enum |
| `IncidentFeedback` | `domain/` | Pure Java record |
| `IncidentFeedbackResult`, `FlaggedAgent` | `domain/` | Pure Java records |
| `IncidentFeedbackService` | `app/ledger/` | Depends on JPA, LedgerEntryRepository |
| `IncidentFeedbackResource` | `app/` | Thin REST, delegates to service |

No new dependencies. No new Flyway migrations. No CDI displacement chain.

---

## Testing

### Unit tests (domain/, no Quarkus)

- `IncidentSeverity` — all four confidence mappings
- `IncidentFeedback` validation — null/blank checks

### Integration tests (@QuarkusTest, app/)

| Test | Verifies |
|------|----------|
| Happy path — single case | 200, attestation written with correct fields |
| Multiple agents for same capability | Two WorkerDecisionEntries → two attestations |
| Unknown PR | 404 |
| No agent for capability | Valid case, no agent did `security-review` → 404 |
| Ambiguous PR | Two APPROVED cases for same PR → 409 with candidate caseIds |
| Disambiguated with caseId | Ambiguous + explicit caseId → 200 |
| Invalid capability tag | `"made-up"` → 400 |
| Each severity level | Correct confidence per severity |
| Attestation field correctness | `attestorId`, `attestorType`, `verdict`, `trustDimension`, `evidence` format |
| Not idempotent | Same feedback twice → two separate attestation records |

Tests use `InMemoryLedgerEntryRepository` for attestation writes. Pre-seed `MergeDecisionLedgerEntry` and `WorkerDecisionEntry` via repository for lookup tests.

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
- **Trust decay acceleration on FLAGGED** — referenced in the issue as `quarkus-ledger#55`; this is a ledger-side feature independent of devtown
