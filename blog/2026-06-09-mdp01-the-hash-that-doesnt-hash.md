---
layout: post
title: "The Hash That Doesn't Hash"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [ledger, merkle, compliance, layer-4]
---

The CaseHub ledger's Merkle chain had always been described as "tamper-evident." Layer 4's job was to wire it into devtown for merge decisions — a domain-specific `LedgerEntry` subclass capturing whether a PR was approved or rejected, with EU AI Act compliance metadata attached. Straightforward integration work, or so I thought.

The interesting part wasn't the wiring. It was what the spec review process uncovered about the foundation itself.

## What the spec review found

I wrote the spec with a clear mental model: `MergeDecisionLedgerEntry` extends `LedgerEntry`, the Merkle chain hashes the entry, the decision is tamper-evident. The first review came back with ten findings. The critical ones were mechanical — a Flyway version collision (V2000 already taken by engine-ledger), no bean observing terminal case state to actually produce the merge decision. Fair.

The finding that changed the design was #3: the `decision` field is not in the Merkle hash. I'd assumed `LedgerMerkleTree.leafHash()` covered the full entry. Claude decompiled the bytecode:

```java
String canonical = String.join("|",
    subjectId, sequenceNumber, entryType,
    actorId, actorRole, occurredAt);
```

Six fields. `subjectId|sequenceNumber|entryType|actorId|actorRole|occurredAt`. Not `decision`. Not `pr_number`. Not `supplementJson`. The chain proves ordering and existence — not content integrity. Someone with database access could change `REJECTED` to `APPROVED` without breaking the chain.

This isn't a devtown bug. It's how the foundation works. `CaseLedgerEntry.caseStatus` has the same exposure. The platform team accepted this trade-off for the same reason it's accepted in most append-only ledgers: the hash chain proves the sequence of events happened in a specific order. Field-level integrity is a database access control concern.

The reviewer initially suggested storing the decision in `supplementJson` since "supplement data serialises to the base table, which IS hash-protected." Wrong — I verified it wasn't. Neither join-table columns nor supplement JSON are in the canonical bytes. The correction sharpened the spec's tamper-evidence scope section into something honest rather than aspirational.

## The persistence unit trap

The second round surfaced a real bug in the existing codebase. `CaseLedgerEntryRepository` (in engine-ledger) extends `JpaLedgerEntryRepository`. The parent injects `@LedgerPersistenceUnit EntityManager` — correct. The subclass adds its own `@Inject EntityManager caseEm` — unqualified — and uses it for `findByCaseId()`, `findWorkerDecisionsByCaseId()`, and `findLatestByCaseId()`.

In a single-PU deployment, both resolve to the same EntityManager. In devtown's two-PU setup (default for casehub-work, qhorus for ledger), the unqualified one silently queries the wrong database and returns empty. No error. No warning. Empty results.

The spec's observer code initially called `findLatestByCaseId()` to set up a causal link between the merge decision and the case lifecycle entry. The final review caught that this call would always return empty — making `causedByEntryId` permanently null, not "best-effort null from race conditions" as the spec described.

The fix: use `findLatestBySubjectId()` (inherited from the parent, correct PU) with an `instanceof CaseLedgerEntry` filter. And inject `LedgerEntryRepository` — the generic interface — not the engine-ledger subclass with its buggy methods.

## What we built

The implementation itself landed cleanly once the spec was right. `MergeDecisionObserver` watches for terminal `CaseLifecycleEvent` transitions — `COMPLETED` maps to `APPROVED`, `CANCELLED` to `REJECTED`, `FAULTED` produces nothing (infrastructure error, not a domain decision). It looks up the `CaseInstance` for PR metadata, writes the entry with a `ComplianceSupplement` for EU AI Act Art.12, and sets a best-effort `causedByEntryId` link.

The compliance report endpoint assembles per-case evidence across four dimensions: audit chain integrity (Merkle verification + inclusion proofs), trust routing (which agent was selected and why), SLA compliance (human review deadlines), and GDPR capability (erasure and pseudonymisation wired). The SLA dimension returns GAP in this layer — there's no ledger-to-WorkItem link yet.

One thing the compliance service implementation discovered that the spec didn't anticipate: `LedgerVerificationService.verify()` is `@Transactional(REQUIRED)`. When it throws — and it throws when no Merkle frontier exists — the Narayana transaction manager marks the joining transaction as rollback-only before the catch clause runs. Every subsequent JPA operation in that transaction silently fails. The fix is `QuarkusTransaction.requiringNew()` for each verification call, with an `EntrySnapshot` pattern to carry entity data across transaction boundaries.

The spec went through four revisions and sixteen review findings before implementation started. The implementation had zero surprises — because the surprises had already happened.
