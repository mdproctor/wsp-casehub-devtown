# Epic 6 — Trust Feedback Loop: Closing the Prescriptive→Normative→Evaluative Loop

**Issue:** devtown#13  
**Date:** 2026-06-23  
**Branch:** issue-13-trust-weighted-routing

---

## Context

Trust-weighted reviewer routing is already wired (devtown#57): `TrustWeightedAgentStrategy` activated via `casehub-engine-ledger`, `DevtownTrustRoutingPolicyProvider` supplies per-capability policies, `WorkerDecisionEventCapture` records routing decisions. The routing infrastructure captures *decisions*, not *outcomes*.

This epic closes the feedback loop — the system that makes trust scores meaningful:

- **Positive feedback:** when a reviewer completes their assignment (qhorus DONE), write a SOUND attestation. Trust score increases.
- **Negative feedback:** when a production incident is traced to a missed review, write a FLAGGED attestation. Trust score decreases.
- **Trust-threshold gating:** high-trust agents earn presumption of honesty on DONE. Low-trust agents are structurally prepared for evidential verification (not yet wired — awaits qhorus EvidentialChecker extraction).

---

## Normative + Evidential Verification

Qhorus benchmarking revealed that `CommitmentState.FULFILLED` is not trustworthy on its own. A RESPONSE message closes a commitment as FULFILLED even when the agent used wrong vocabulary (should have sent DONE for a COMMAND obligation). Any system deriving meaning from "was the commitment fulfilled?" — including trust scores — needs an evidential check.

Three layers:
1. **Normative layer** — what qhorus already provides: typed messages, commitment lifecycle, ledger. Makes agent interactions structured and observable.
2. **Trust-gated attestation** — reads trust scores to decide whether DONE earns SOUND immediately or should be evidentially verified. **Built in this epic.**
3. **Evidential layer** — reads what the normative layer recorded and verifies obligations were honestly resolved. **Deferred — awaits qhorus EvidentialChecker extraction.**

The hook point (`CommitmentAttestationPolicy`) is a published SPI in `casehub-qhorus-api`. The trust-gated policy overrides the default with `@Alternative @Priority(1)`.

---

## Architecture — Two Repos, Two Phases

### Phase 1: Qhorus (upstream platform primitives)

Two new components in `casehub-qhorus` runtime. Zero new dependencies — qhorus already depends on `casehub-ledger-api` and `casehub-ledger` runtime.

#### `TrustGatedAttestationPolicy`

```
Module:     casehub-qhorus runtime
Package:    io.casehub.qhorus.runtime.ledger
CDI:        @Alternative @Priority(1) @ApplicationScoped
Displaces:  StoredCommitmentAttestationPolicy @DefaultBean
Activates:  Only when TrustScoreSource is satisfied (casehub-ledger on classpath)
```

Implements `CommitmentAttestationPolicy`. Wraps `StoredCommitmentAttestationPolicy`:

- **Non-DONE verdicts (FAILURE, DECLINE):** pass through unchanged — FLAGGED attestation as per default policy.
- **DONE verdicts:** read agent's global trust score via `TrustScoreSource.globalScore(resolvedActorId)`.
  - Score ≥ threshold OR score absent (bootstrap — no history): pass through SOUND unchanged. Presumption of honesty.
  - Score < threshold: pass through SOUND for now (placeholder). When `EvidentialChecker` is extracted, this branch runs the evidential check and potentially downgrades to FLAGGED with lower confidence.
- **Bootstrap safety:** agents with no trust history (`OptionalDouble.empty()`) are treated as trusted — consistent with Phase 0 of the trust maturity model. No data = no penalty.

Threshold configurable via `QhorusConfig`: `casehub.qhorus.attestation.trust-threshold` (default 0.6).

**Why `@Alternative @Priority(1)`, not `@ApplicationScoped`:** `StoredCommitmentAttestationPolicy` is `@DefaultBean`. An `@ApplicationScoped` bean would also displace it. But `@Alternative @Priority(1)` preserves the override chain — application-tier harnesses can further override with `@Alternative @Priority(100)`. This is the correct CDI pattern for platform-tier overrides that consumers may need to extend.

#### `RetroactiveAttestationService`

```
Module:     casehub-qhorus runtime
Package:    io.casehub.qhorus.runtime.ledger
CDI:        @ApplicationScoped
```

Generic mechanism: write a FLAGGED attestation against a previous ledger entry. Any harness can use this to say "we later learned this outcome was bad."

```java
public record IncidentAttestation(
    UUID ledgerEntryId,     // the original entry being retroactively flagged
    String evidence,        // free-text description of what went wrong
    String attestorId,      // who reported the incident
    ActorType attestorType, // HUMAN or SYSTEM
    String capabilityTag,   // capability context (e.g., "security-review")
    String tenancyId
) {}

public UUID reportIncident(IncidentAttestation incident);
```

- Validates `ledgerEntryId` exists via `LedgerEntryRepository.findEntryById()`
- Writes `LedgerAttestation` with `verdict=FLAGGED`, `confidence=0.9` (high — explicit report), capability tag from input
- Returns attestation ID
- `TrustScoreJob` picks up the FLAGGED attestation and decrements the actor's trust score

**Future promotion candidate:** the raw attestation write mechanics may eventually move to `casehub-ledger` itself. The `RetroactiveAttestationService` API uses only ledger types (`LedgerAttestation`, `LedgerEntryRepository`). For now, qhorus is the right home because the service is composed with commitment semantics.

---

### Phase 2: Devtown (domain-specific wiring)

#### `PostMergeIncidentHandler`

```
Module:     app
Package:    io.casehub.devtown.app.incident
CDI:        @ApplicationScoped
```

Accepts domain-specific incident reports and delegates to `RetroactiveAttestationService`:

1. Incident arrives with `(repo, prNumber, description, capabilityTag?)`
2. Correlate to the original PR review case via `PrReviewCaseTracker` (`repo:prNumber` → `caseId`)
3. Find reviewer ledger entries — query `LedgerEntryRepository.findBySubjectId(caseId, tenancyId)`, filter for `WorkerDecisionEntry` with matching capability tag
4. For each matching reviewer entry: call `RetroactiveAttestationService.reportIncident()`

If `capabilityTag` is present, flag only that capability's reviewer. If absent, flag all reviewers who touched the PR.

#### REST Entry Point

```
POST /incidents

{
  "repo": "casehubio/engine",
  "prNumber": 547,
  "description": "Production NPE traced to missing null check in WritablePanelImpl",
  "capabilityTag": "security-review"
}
```

On a dedicated `IncidentResource` — incidents are a separate concern from PR review lifecycle.

#### MCP Tool

```
report_incident(repo, prNumber, description, capabilityTag?)
```

In `DevtownMcpTools`. Same `PostMergeIncidentHandler` underneath. Returns confirmation with case ID and flagged reviewer entries.

#### End-to-End Closed-Loop Test

`@QuarkusTest` proving the done-when:

1. Seed two agents with different trust scores (agent-A at 0.85, agent-B at 0.55 for `security-review`)
2. Start a PR review case requiring `security-review`
3. Assert agent-A is selected (higher trust × blendFactor outscores agent-B)
4. Submit an incident report against the PR
5. Assert FLAGGED attestation written against agent-A's review entry
6. Trigger trust score recompute (`TrustScoreJob` or `IncrementalTrustUpdateObserver`)
7. Assert agent-A's trust score has decreased
8. Start a second PR review case
9. Assert routing has shifted — agent-A's reduced score changes the selection outcome

---

## Step 0 — Pre-existing Test Fix

`PrReviewCaseDefinitionEquivalenceTest.dslMatchesYaml` fails: `expected: null but was: "{}"`. Upstream engine-api schema serialization change. Mechanical fix — align the DSL-built definition with the current YAML parsing behaviour.

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

## Issues to File

### Qhorus (prerequisites for full evidential verification — not blocking this epic)

1. **Extract EvidentialChecker to publishable module** — move from `examples/agent-communication/` to `casehub-qhorus` runtime or new `casehub-qhorus-audit` optional module.

2. **Extend CommitmentAttestationPolicy SPI with CommitmentContext** — current `attestationFor(MessageType, String)` is insufficient for evidential checking. Add `CommitmentContext(correlationId, channelId, taskType)` parameter.

3. **Close RESPONSE-path attestation gap** — RESPONSE fulfilling a COMMAND commitment produces no attestation. Should produce FLAGGED (wrong terminal type for COMMAND obligation).

### Devtown

4. **Trust visibility UI** — trust scores by capability, routing history, incident reports. Brainstorm-first framing — casehub-pages is new and UI strategy is evolving. Data APIs (REST + `TrustExportService`) available from this epic.

5. **Qhorus trust gate configuration** (existing devtown#58) — confirmed still open. Bootstrap exemption design before enabling `casehub.qhorus.commitment.min-obligor-trust`.

---

## Build Order

1. Implement and test in qhorus (Phase 1)
2. Publish qhorus SNAPSHOT
3. Implement and test in devtown (Phase 2)

---

## Out of Scope

- Evidential verification (awaits qhorus EvidentialChecker extraction)
- UI/dashboard (brainstorm-first, casehub-pages TBD)
- Qhorus trust gate configuration (devtown#58)
- RESPONSE-path attestation fix (qhorus issue)

---

## Done-When

Two agents with different trust scores compete for a security review; the higher-trust agent wins. A subsequent production incident reduces the responsible agent's score and shifts all future routing away from it — without human configuration.
