---
layout: post
title: "Attestations From Events, Not Reviews"
date: 2026-08-03
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust-scoring, attestation-pipeline, sealed-interfaces, idempotency, contributor-trust]
---

The reviewer attestation path already works. An agent completes a review, qhorus resolves the commitment, `CommitmentAttestationPolicy` writes the verdict. But contributor trust is a different animal. Nobody "reviews" a contributor — we observe what happens to their PR and infer quality from the outcome.

That's the design question I started with: how do you turn a lifecycle event into a trust signal? A merge isn't a review. A rejection isn't a finding. They're outcomes that carry information about the person who submitted the work, and the information depends on context the event itself doesn't carry.

## Two actors, one ledger

The existing reviewer path scores agents: `WorkerDecisionEntry` records what the agent did, `LedgerAttestation` records how well. The contributor path needs to score humans: `ContributorOutcomeLedgerEntry` records what happened to the PR, attestations record what that implies about the contributor.

Same ledger. Same Bayesian Beta math. Different actor type (`HUMAN` vs `AGENT`), different capability key (`pr-contribution` vs `security-review`), different trigger mechanism (lifecycle event vs commitment resolution). The spec for this grew out of a question I kept coming back to: what's the minimum new infrastructure needed, and what's already there?

The answer was less than I expected. The scoring pipeline doesn't care who it's scoring — it cares about attestation verdicts, confidence values, and dimension scores. What we needed was a way to get those attestations written from PR events instead of from agent completions.

## The blocks SPI

Claude and I built `AttestationIntent` as a blocks-level record — one intent can carry multiple dimension scores, and `DefaultAttestationIntentWriter` expands them into individual `LedgerAttestation` records. A merged-first-attempt PR produces one intent with `{merge-rate: 1.0, first-attempt-quality: 1.0}`, which becomes two attestations. Deterministic UUIDs (v3 via `nameUUIDFromBytes`) make the writes idempotent — the mining path and the live webhook path produce identical IDs for the same PR event.

I initially designed a generic `LifecycleAttestationPolicy<E>` interface for blocks. The structure review killed it — dead architecture with no runtime dispatch. The mapping logic is application-specific. It stayed as a concrete `ContributorAttestationPolicy` in devtown, which is where it belongs.

## Event ordering

The subtlest decision was when to fire the lifecycle event relative to the terminal signal. `closePr()` does two things: fire a `PrLifecycleEvent.Merged` (which triggers the attestation observer), and signal the case engine that the PR is merged. The original code signalled the engine first.

That's wrong. If the observer fails after the terminal signal, the case is already closed. GitHub retries the webhook, but `findActiveCaseByPr()` returns empty — the attestation is permanently lost. Fire the lifecycle event first. If the observer throws, the case stays active, the webhook retries, and the attestation gets another chance.

## Sealed interfaces for PR events

`PrLifecycleEvent` is a sealed interface with four record variants: `Merged`, `Rejected`, `ChangesRequested`, `TriageRejected`. Each carries exactly what the observer needs — no more, no less. `Merged` and `Rejected` carry `reviewRounds` resolved by the service layer from case context before firing. The observer never reaches into the case engine.

`TriageRejected` is forward-declared — no producer exists yet. Epic 3's `IntakeQueueService.reject()` will fire it. The observer handler is already wired and tested with manually constructed events.

## What landed

Two commits on main. The first is Epic 1's domain model (intake lanes, classification policy). The second is the full pipeline: blocks SPI, lifecycle events, webhook extensions (`pull_request_review.submitted` for `changes_requested`), port interface changes (`PrClosePayload` replaces the old three-arg `closePr`), JPA entity with V2012 migration, dedicated writer with transaction boundary enforcement, CDI observer with `@Priority(100)` for Epic 5 composition safety.

Epics 3 through 6 can now run in parallel. The attestation pipeline is the foundation — intake classification reads trust scores, vouching writes endorsement attestations through the same writer, history mining replays through the same deterministic ID formula. One pipeline, six outcomes, two trust dimensions.
