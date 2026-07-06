# Trust-Gated Attestation Policy

**Issue:** devtown#97, engine#TBD (to be filed)
**Date:** 2026-07-06
**Branch:** issue-97-trust-gated-attestation

---

## Problem

`StoredCommitmentAttestationPolicy` (qhorus-runtime `@DefaultBean`) treats all DONE messages identically — SOUND at 0.7 confidence regardless of the agent's trust history. A first-time agent and a proven high-trust agent produce the same attestation weight.

This means trust scores converge slowly and an unreliable agent's DONE carries the same conviction as a reliable agent's DONE until enough incidents accumulate to degrade the score externally.

## Solution

A `TrustGatedAttestationPolicy` in **engine-ledger** that modulates attestation confidence based on the agent's capability trust score. Same verdict (SOUND for DONE), different conviction.

### Why engine-ledger

The trust classification family already lives in `io.casehub.ledger.routing`:
- `TrustCandidateClassifier` — classifies candidates by trust phase
- `TrustWeightedAgentStrategy` — routes by trust scores
- `TrustWeightedImplementationRoutingStrategy` — same pattern for implementations
- `DefaultTrustRoutingPolicyProvider` — `@DefaultBean` policy provider

Same trust dependencies (`TrustScoreSource`, `TrustRoutingPolicyProvider`), same concern (trust-based gating decisions), same module. This adds `casehub-qhorus-api` as a new compile dependency to engine-ledger — the module's first qhorus dependency. The coupling is narrow: only the `CommitmentAttestationPolicy` SPI, `CommitmentContext` record, and `MessageType` enum. `casehub-qhorus-api` is a thin API module with no runtime dependencies; the alternative (placing the class in qhorus-runtime) would require a reverse dependency on engine-api for `TrustRoutingPolicyProvider`, creating a worse coupling direction. Not blocks — fails all three scope criteria (no LLM, no classical AI, configuration plumbing only).

### Why not an EvidentialChecker dependency

Issue #97 originally referenced `EvidentialChecker.checkObligation()` for low-trust agents. Inspection shows `checkObligation()` is a pure vocabulary validation — "is the terminal type one of DONE/FAILURE/DECLINE?" — with zero store dependencies. The three-line check is inlined in the policy. No qhorus-runtime dependency needed.

The full V1–V4 benchmark integration (`EvidentialChecker.check()`) is a separate concern: it requires `DataStore`, `MessageStore`, `CommitmentStore` (all qhorus-runtime), operates on `BenchmarkContext` (not `CommitmentContext`), and applies variant-specific verification. This is deferred to devtown#141, which gates on this policy existing. Issue #97's scope covers capability-scoped confidence modulation; evidential verification for BELOW_THRESHOLD agents follows separately.

---

## Design

### Class: `TrustGatedAttestationPolicy`

**Package:** `io.casehub.ledger.routing`
**Annotations:** `@Alternative @Priority(1) @ApplicationScoped`
**Implements:** `CommitmentAttestationPolicy`
**Displaces:** `StoredCommitmentAttestationPolicy` (`@DefaultBean` in qhorus-runtime)

**Injected dependencies:**
- `TrustScoreSource` — capability scores and decision counts
- `TrustRoutingPolicyProvider` — per-capability thresholds and minimum observations

### Attestation Logic

```
attestationFor(terminalType, resolvedActorId, context):

  1. Non-discharge types — return empty
     if terminalType not in {DONE, FAILURE, DECLINE, RESPONSE}:
       return Optional.empty()  // cannot discharge a commitment

  2. Wrong vocabulary — RESPONSE to a COMMAND obligation
     if terminalType is RESPONSE:
       return FLAGGED at BASE_RESPONSE_CONFIDENCE (0.3)
              attestorId = "system", attestorType = SYSTEM

  3. Negative outcomes (trust-independent)
     FAILURE → FLAGGED at BASE_FAILURE_CONFIDENCE (0.6)
               attestorId = "system", attestorType = SYSTEM
     DECLINE → FLAGGED at BASE_DECLINE_CONFIDENCE (0.4)
               attestorId = "system", attestorType = SYSTEM

  4. DONE — resolve trust phase
     if context is null OR capabilityTag is null/empty/GLOBAL ("*"):
       return SOUND at BASE_DONE_CONFIDENCE (0.7)
              attestorId = resolvedActorId, attestorType = AGENT

     capScore = source.capabilityScore(resolvedActorId, capabilityTag)
     decCount = source.decisionCount(resolvedActorId, capabilityTag)
     policy   = policyProvider.forCapability(capabilityTag)

     if capScore is empty OR policy.isBootstrap(decCount):
       phase = BOOTSTRAP
     elif policy.isBorderline(capScore):
       phase = BORDERLINE
     elif policy.passesThresholdCheck(capScore):
       phase = QUALIFIED
     else:
       phase = BELOW_THRESHOLD

  5. DONE — map phase to confidence
     BOOTSTRAP:       BASE_DONE_CONFIDENCE (0.7)
     BORDERLINE:      BASE_DONE_CONFIDENCE (0.7)
     QUALIFIED:       min(1.0, BASE_DONE_CONFIDENCE × (1.0 + (capScore - threshold)))
     BELOW_THRESHOLD: max(MIN_CONFIDENCE_FLOOR, BASE_DONE_CONFIDENCE × capScore)

  All DONE outcomes return verdict = SOUND,
  attestorId = resolvedActorId, attestorType = AGENT

  Phase classification mirrors TrustCandidateClassifier.classify() — same order,
  same boundary methods (isBootstrap, isBorderline, passesThresholdCheck). Phase 3
  quality floors are intentionally omitted: quality floors gate routing eligibility,
  not attestation confidence.
```

### Confidence modulation rationale

- **BOOTSTRAP** — no track record. Base confidence is the neutral starting point. Neither boosted nor penalised.
- **BORDERLINE** — score within `borderlineMargin` of threshold (above or below). In the routing model, borderline agents trigger human oversight escalation. For attestation, they're in the uncertainty zone — not enough signal to boost or penalise. Base confidence, same as bootstrap.
- **QUALIFIED** — agent has proven reliable above threshold and not borderline. Confidence boosted proportionally to distance above `threshold` (not `threshold + borderlineMargin`). This deliberately creates a qualification bonus at the BORDERLINE→QUALIFIED boundary — crossing from "uncertain" to "proven" is a categorical transition, not a smooth interpolation. The BELOW_THRESHOLD→BORDERLINE boundary has the same step-function character. An agent at 0.81 (just past borderline with defaults) gets `0.7 × 1.11 = 0.777`; at 0.9 gets `0.7 × 1.2 = 0.84`. Capped at 1.0.
- **BELOW_THRESHOLD** — agent has track record but it's poor. Confidence scaled by capability score, floored at `MIN_CONFIDENCE_FLOOR` (0.05). An agent at 0.5 gets `0.7 × 0.5 = 0.35`. An agent at 0.01 gets `max(0.05, 0.7 × 0.01) = 0.05` — the floor ensures attestations always carry some evidential weight and prevents trust from becoming unrecoverable. (`OutcomeRecord` rejects confidence ≤ 0.0, so the floor also guards against validation failures.) Their DONE still counts, but carries less conviction.

### Constants

```java
static final double BASE_DONE_CONFIDENCE = 0.7;
static final double BASE_FAILURE_CONFIDENCE = 0.6;
static final double BASE_DECLINE_CONFIDENCE = 0.4;
static final double BASE_RESPONSE_CONFIDENCE = 0.3;
static final double MIN_CONFIDENCE_FLOOR = 0.05;
```

Same defaults as `StoredCommitmentAttestationPolicy`. No config injection — trust modulation is the dynamic tuning mechanism. When `TrustGatedAttestationPolicy` activates via classpath, any `casehub.qhorus.attestation.*` config properties set for `StoredCommitmentAttestationPolicy` are silently ignored — this is intentional: the static base values become the starting point for trust-modulated confidence, and the modulation itself is the tuning mechanism. Apps that need different base values override the entire policy.

### Activation

Automatic via classpath. When `casehub-engine-ledger` is on the classpath, `@Alternative @Priority(1)` displaces `StoredCommitmentAttestationPolicy` (`@DefaultBean`). Same pattern as `TrustWeightedAgentStrategy`.

---

## What devtown does

Nothing for the policy itself. Devtown already has `DevtownTrustRoutingPolicyProvider` supplying per-capability policies with domain-specific thresholds. The trust-gated attestation activates automatically when the engine-ledger SNAPSHOT is picked up.

Devtown's contribution is an integration test proving the closed-loop behaviour with real trust scores in a CDI container.

---

## Testing

### engine-ledger: `TrustGatedAttestationPolicyTest` (unit)

Mock `TrustScoreSource` and `TrustRoutingPolicyProvider`. Pure unit tests:

| Case | Terminal type | Trust phase | Expected |
|------|--------------|-------------|----------|
| DONE + BOOTSTRAP | DONE | No history | SOUND @ 0.7, attestor = resolvedActorId/AGENT |
| DONE + QUALIFIED (0.9, threshold 0.7) | DONE | Above threshold | SOUND @ 0.84, attestor = resolvedActorId/AGENT |
| DONE + QUALIFIED boundary (0.81, threshold 0.7, margin 0.1) | DONE | Just past borderline | SOUND @ 0.777, attestor = resolvedActorId/AGENT |
| DONE + BORDERLINE above (0.75, threshold 0.7, margin 0.1) | DONE | Borderline | SOUND @ 0.7, attestor = resolvedActorId/AGENT |
| DONE + BORDERLINE below (0.65, threshold 0.7, margin 0.1) | DONE | Borderline | SOUND @ 0.7, attestor = resolvedActorId/AGENT |
| DONE + BELOW_THRESHOLD (0.5) | DONE | Below threshold | SOUND @ 0.35, attestor = resolvedActorId/AGENT |
| DONE + BELOW_THRESHOLD near-zero (0.01) | DONE | Below threshold | SOUND @ 0.05 (floor), attestor = resolvedActorId/AGENT |
| DONE + null capabilityTag | DONE | — | SOUND @ 0.7 (fallback) |
| DONE + null context | DONE | — | SOUND @ 0.7 (fallback) |
| DONE + GLOBAL capabilityTag ("*") | DONE | — | SOUND @ 0.7 (fallback) |
| FAILURE (any trust) | FAILURE | Any | FLAGGED @ 0.6, attestor = "system"/SYSTEM |
| DECLINE (any trust) | DECLINE | Any | FLAGGED @ 0.4, attestor = "system"/SYSTEM |
| RESPONSE (wrong vocab) | RESPONSE | Any | FLAGGED @ 0.3, attestor = "system"/SYSTEM |
| QUERY (non-discharge) | QUERY | Any | Optional.empty() |
| STATUS (non-discharge) | STATUS | Any | Optional.empty() |

### devtown: integration test

Extends or complements `TrustFeedbackClosedLoopTest`. Proves in a `@QuarkusTest`:
1. `TrustGatedAttestationPolicy` displaces `StoredCommitmentAttestationPolicy` (CDI activation)
2. High-trust agent DONE produces higher confidence than low-trust agent DONE
3. Confidence values match the phase-based calculation with real `DevtownTrustRoutingPolicyProvider` thresholds

---

## Issue plan

1. **File engine issue** — `feat: TrustGatedAttestationPolicy — capability-scoped attestation confidence` in casehubio/engine
2. **Implement in engine-ledger** — class + unit tests
3. **Publish engine SNAPSHOT** — devtown picks up automatically
4. **Update devtown#97** — reference engine issue as dependency, scope to integration test
5. **Implement devtown integration test** — prove activation and confidence modulation

---

## Out of scope

- Configurable base confidence values (apps override the whole policy)
- Full `EvidentialChecker` V1–V4 benchmark integration for BELOW_THRESHOLD agents (devtown#141 — requires qhorus-runtime `DataStore`/`MessageStore`/`CommitmentStore` dependencies)
- Quality floor checks for attestation (quality floors apply to routing, not attestation gating)
- New `TrustConfidenceMapping` SPI (YAGNI — override the policy if different mapping needed)
