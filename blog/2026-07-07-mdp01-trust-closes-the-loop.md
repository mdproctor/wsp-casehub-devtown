---
layout: post
title: "Trust Closes the Loop"
date: 2026-07-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust, attestation, cdi, snapshot-drift]
---

# CaseHub — Trust Closes the Loop

**Date:** 2026-07-07
**Type:** phase-update

---

## What I was trying to achieve: wire trust-modulated attestation confidence end-to-end

The trust scoring pipeline in devtown has been accumulating capability scores from review outcomes for weeks. But attestation — the moment an agent's DONE message becomes a trust-relevant record — treated every agent identically. A first-timer and a proven reviewer both produced SOUND at 0.7 confidence. The scores existed; they just didn't feed back into anything.

Issue #97 closes that loop: when an agent completes a review, its attestation confidence now reflects its trust history for that specific capability.

## What we believed going in: the policy belongs in devtown

The original issue description had me thinking this was a devtown feature — something that would sit alongside `DevtownTrustRoutingPolicyProvider` and use the domain-specific thresholds directly. Three spec review rounds changed that. The policy's only differentiating dependency is `TrustScoreSource`, which lives in engine-ledger alongside `TrustCandidateClassifier` and `TrustWeightedAgentStrategy`. Same concern, same module.

## The CDI gap that tests don't catch

The implementation landed in engine-ledger cleanly — `@Alternative @Priority(1)` displaces the `@DefaultBean` `StoredCommitmentAttestationPolicy` when engine-ledger is on the classpath. Devtown's job was just an integration test proving the activation and confidence modulation with real scores.

What caught us was SNAPSHOT drift. After installing the latest engine SNAPSHOT locally, `mvn test` passed — 338 tests, all green. `mvn install` failed with 34 unsatisfied CDI dependencies.

The engine had added reactive SPI interfaces (`ReactiveCaseInstanceRepository`, `ReactiveEventLogRepository`, and their cross-tenant variants). `@QuarkusTest` augmentation only validates beans reachable from the test's injection graph. `quarkus:build` validates everything. So tests pass, production build fails — silently, until you try to package.

The fix was straightforward once we understood the cause: four `@ApplicationScoped` wrappers extending the InMemory implementations. A useful discovery from the type hierarchy — each `InMemoryReactive*` base implements both the standard and cross-tenant interfaces, so two wrappers satisfy four CDI injection points.

## The defensive fallback Claude flagged

Claude caught a spec-implementation gap while verifying the engine class against the design spec. The spec explicitly requires a try/catch around the trust-lookup path in `attestDone()` — if `TrustScoreSource` or `TrustRoutingPolicyProvider` throws, fall back to base confidence rather than propagating the exception. The implementation had no try/catch.

This matters because `LedgerWriteService.writeAttestation()` calls `attestationFor()` outside its own try/catch. An unhandled exception rolls back the entire `@Transactional(REQUIRES_NEW)` transaction — including the message ledger entry. A transient cache failure would compromise the tamper-evident audit trail.

Filed engine#679, fixed it on engine main, pushed. Two lines of production code, two unit tests.

## Where this leaves the trust pipeline

The feedback loop is now closed: review outcomes accumulate as attestations → attestations feed trust score computation → trust scores modulate future attestation confidence. A reliable agent's DONE carries more conviction; an unreliable agent's DONE carries less. The system self-corrects without human intervention.

The confidence formula is deliberately asymmetric. Qualified agents get a modest boost above base (0.7 × 1.2 = 0.84 at score 0.9). Below-threshold agents get scaled down hard (0.7 × 0.5 = 0.35 at score 0.5), floored at 0.05 so attestations never become weightless. Bootstrap and borderline agents stay at base — not enough signal to move in either direction.

What's still missing is the evidential verification path for below-threshold agents. Right now their DONE is accepted at low confidence. devtown#141 gates on this policy existing and would add `EvidentialChecker` integration — actually verifying the agent's work rather than just weighting its word.
