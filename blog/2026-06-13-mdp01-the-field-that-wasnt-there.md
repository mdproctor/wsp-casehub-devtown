---
layout: post
title: "The Field That Wasn't There"
date: 2026-06-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust, attestation, tokenisation, field-shadowing, incident-feedback]
---

The trust feedback loop in devtown has always had a gap: an agent reviews a PR, the PR merges, and then two weeks later a production incident reveals the agent missed something obvious. The agent's trust score never hears about it. The Bayesian model keeps routing security-critical work to an agent that demonstrably missed a security issue.

Closing that gap turned out to be straightforward in concept — a REST endpoint that accepts incident reports, finds which agents reviewed the PR, and writes a FLAGGED attestation against each one. The existing `TrustScoreJob` picks it up on the next run. The design is clean: `IncidentSeverity` maps to attestation confidence (LOW=0.3 through CRITICAL=0.9), `ReviewDomain.REVIEW_CAPABILITIES` validates that you can only flag review capabilities (not CI runners or merge executors), and `MergeDecisionLedgerEntry` gains a `@NamedQuery` to look up cases by PR identity.

The spec review caught something I wouldn't have found during implementation. The idempotency check — "has this incident already been recorded against this agent?" — originally used `findAttestationsByEntryIdAndCapabilityTag`, which returns attestations filtered by the entry they're attached to. The plan was to filter client-side by `attestorId == "devtown:incident-feedback"`. The problem: `JpaLedgerEntryRepository.saveAttestation()` tokenises the `attestorId` on every write. Under `InternalActorIdentityProvider` (the GDPR production path), `"devtown:incident-feedback"` becomes a UUID token in the database. The client-side comparison would always fail — the stored value is a token, not the original string.

The fix was to use `findAttestationsByAttestorIdAndCapabilityTag` instead — the attestor-scoped query that calls `tokeniseForQuery()` internally. One method call difference, but the first path silently breaks idempotency under GDPR tokenisation. The deeper issue: system actors like `"devtown:incident-feedback"` shouldn't be tokenised at all — they aren't natural persons — but that's a foundation fix (filed as ledger#130).

Implementation itself was blocked by a three-day foundation cascade. The ledger had recently added `tenancyId` to `LedgerEntry` (the base class), but three JOINED-inheritance subclasses — `CaseLedgerEntry`, `WorkerDecisionEntry`, and devtown's own `MergeDecisionLedgerEntry` — each still declared their own `tenancyId` field. Java field shadowing made this invisible: `cle.tenancyId = "test-tenant"` appeared to work because Hibernate's bytecode enhancement transparently routed the access to the subclass field. But `em.persist()` reads the base class field via reflection — which was never set. The symptom was `PropertyValueException: not-null property references a null or transient value` on a field the test code had explicitly set.

The diagnostic that cracked it was simple once I thought to try it: read both the subclass and base fields via reflection on the same object. The subclass field was `"test-tenant"`. The base field was `null`. Two fields with the same name, invisible to Java code, visible only through reflection. The fix was to remove the shadowing declarations from all subclasses — the base class field handles it via JPA inheritance.

Three foundation issues, three repos, three sessions to fix them (ledger#131, engine#460, qhorus#269), then the actual implementation landed in a day. The service itself is modest: look up the merge decision, find the worker assignments, check idempotency, write attestations. But the path to getting there surfaced real architectural issues in the foundation — the kind that would have bitten every application, not just devtown.
