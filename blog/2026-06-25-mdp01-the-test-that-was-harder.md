---
layout: post
title: "The Test That Was Harder Than the Feature"
date: 2026-06-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust-routing, jpa, integration-testing, snapshot-drift]
---

The trust feedback loop in devtown was supposed to be the big epic. Two repos, new platform primitives in qhorus, upstream-first architecture. Three review rounds later, the scope collapsed to something embarrassing: write one test.

Everything else was already shipped. The routing infrastructure landed in devtown#57. The incident feedback endpoint — with its GDPR-safe idempotency, severity-based confidence mapping, and 15 integration tests — shipped under devtown#5. The positive feedback path (SOUND attestations on review completion) was live in qhorus's `LedgerWriteService` without devtown touching a line. The spec I wrote for two phases of cross-repo work was describing machinery that already existed.

The one thing nobody had done was prove the full chain works end-to-end. SOUND attestation → trust score materialised → routing uses scores → incident FLAGGED → score decreases → routing shifts. Each link was tested in isolation. No test had ever exercised them as a single chain.

That test took two days.

The first day was SNAPSHOT drift. The engine-api JAR in my local Maven cache had breaking changes that hadn't been published — `PlannedAction` and `Capability` moved to `casehub-worker-api` with new constructors. I migrated everything, then `mvn clean` pulled the published SNAPSHOT from GitHub Packages and the migration was wrong. The published JAR still had the original API. Hours of migration work reverted. Then the platform was rebuilt from source, and the migration turned out to be real after all. The stale cache had the right answer for the wrong reason.

The second day was the trust score. Ten SOUND attestations built a capability score of 0.917. One CRITICAL FLAGGED attestation was written against the same entry. Score after recompute: 0.917. Unchanged.

I filed ledger#157 — "PerActorTrustComputer must incorporate FLAGGED attestations into capability scores." The ledger team traced every code path and found it was correct. They were right. The computation was fine. The attestation was in the DB — a direct JPQL query returned both SOUND and FLAGGED. A direct `TrustScoreComputer.compute()` call with the same data correctly dropped the score to 0.513. The `TrustScoreJob.runComputation()` batch path wrote the correct score (0.884) to `actor_trust_score`. But `TrustGateService.currentScore()` returned 0.917.

The bug was a JPA first-level cache stale read. The test's ambient persistence context loaded the `ActorTrustScore` entity during an initial read. When `TrustScoreJob` wrote the updated score inside `QuarkusTransaction.requiringNew()`, it committed to a different `EntityManager`. The ambient one still had the old entity in its identity map. Every subsequent read through `TrustGateService` — which shared the ambient `EntityManager` via CDI request scope — returned the cached value without hitting the DB.

The fix was one line: wrap the read in `QuarkusTransaction.requiringNew()`.

```java
OptionalDouble score = QuarkusTransaction.requiringNew().call(
    () -> trustGateService.currentScore(agentId, capability));
```

The trust feedback loop works. Two agents with equal SOUND histories, one CRITICAL incident, recompute, routing shifts. The done-when is proven. But the test that proves it found more bugs than the feature it validates — a false bug report against a correct library, two rounds of SNAPSHOT migration whiplash, and a JPA cache behaviour that none of the ledger's own unit tests could have caught because they bypass the persistence layer entirely.

The spec started as an ambitious two-repo design and ended as one test file. The test started as a simple assertion and ended as a cross-repo debugging expedition. The interesting pattern: integration tests that exercise the full stack through CDI find a completely different class of bugs than unit tests — not logic errors, but wiring and cache coherence issues that only emerge when multiple services share a persistence context.