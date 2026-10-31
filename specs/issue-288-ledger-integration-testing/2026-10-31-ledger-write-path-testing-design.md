# Ledger Write Path Integration Testing — Design Spec

**Issue:** casehubio/devtown#288
**Branch:** issue-288-ledger-integration-testing
**Date:** 2026-10-31

## Problem

`casehub.ledger.enabled=false` in the default test configuration. While 7 existing tests use `LedgerEnabledTestProfile`, none verifies the engine-level capture beans (`CaseLedgerEventCapture`, `WorkerDecisionEventCapture`) actually write entries through the CDI observer path in devtown context. The Merkle frontier — the tamper-evidence mechanism — is completely untested.

## Scope

Three changes to close the gap between what's tested and what runs in production:

1. **Extend `MergeDecisionObserverTest`** — verify engine's `CaseLedgerEntry` is written alongside devtown's `MergeDecisionLedgerEntry`, and that the Merkle frontier advances
2. **New `WorkerDecisionWritePathTest`** — fire `WorkerDecisionEvent`, verify `WorkerDecisionEntry` with trust enrichment, verify Merkle frontier
3. **Extend `GdprErasureResourceTest`** — verify Merkle frontier integrity after erasure

## Design

### 1. MergeDecisionObserverTest extensions

**Profile:** `LedgerEnabledTestProfile` (unchanged)

**New injections:**
- `LedgerMerkleFrontierRepository` — to query frontier state

**New test method: `completedCase_writesCaseLedgerEntryAndAdvancesFrontier()`**

Fires `CaseLifecycleEvent` (COMPLETED) — same pattern as existing `completedCase_writesApprovedMergeDecision()`. Asserts:
- `CaseLedgerEntry` exists for the caseId (written by engine's `CaseLedgerEventCapture`)
- Entry has correct lifecycle fields: `eventType=CASE_COMPLETED`, `caseStatus=COMPLETED`, `actorId=system`
- Entry has non-null `digest` (computed by `save()`)
- Merkle frontier for subjectId has `Integer.bitCount(entryCount)` nodes
- Frontier nodes have non-null `hash` values

**Why a new method instead of extending the existing one:** The existing test asserts `MergeDecisionLedgerEntry` (devtown's observer). Both observers fire on the same event, but they write different entry types. Separate test methods keep failure signals distinct — a frontier regression won't mask a decision-field regression.

### 2. WorkerDecisionWritePathTest (new class)

**Profile:** `TrustScoringTestProfile` — enables both `casehub.ledger.enabled=true` and `casehub.ledger.trust-score.enabled=true`

**Injections:**
- `Event<WorkerDecisionEvent>` — to fire the CDI event
- `LedgerEntryRepository` — to query written entries
- `LedgerMerkleFrontierRepository` — to verify frontier
- `@LedgerPersistenceUnit EntityManager` — for type-specific queries

**Test: `workerDecision_writesEntryWithTrustEnrichment()`**

1. Fire `WorkerDecisionEvent` with: known caseId, tenancyId, workerId, capabilityTag (e.g. `code-analysis`), traceId
2. `await().atMost(5, SECONDS)` for async observer
3. Assert `WorkerDecisionEntry` exists for caseId:
   - `workerId` matches
   - `capabilityTag` matches
   - `trustScoreAtRouting` is non-null (populated by `TrustScoreSource`)
   - `thresholdApplied` is non-null (populated by `TrustRoutingPolicyProvider`)
   - `entryType` is `EVENT`
   - `digest` is non-null
4. Assert Merkle frontier has `Integer.bitCount(1) = 1` node with non-null hash

**Test: `multipleDecisions_frontierAdvancesCorrectly()`**

1. Fire 3 `WorkerDecisionEvent`s for the same caseId (different workerIds)
2. Assert 3 `WorkerDecisionEntry` rows
3. Assert Merkle frontier has `Integer.bitCount(3) = 2` nodes

**Trust dependency note:** `TrustScoreSource.capabilityScore()` and `TrustRoutingPolicyProvider.forCapability()` use real beans. If no trust data exists for the worker/capability, they return defaults (null score, default threshold). The test asserts non-null when trust-score is enabled — the materialization produces a default score. If this assumption proves wrong at test time, we may need to seed a trust attestation first.

### 3. GdprErasureResourceTest extension

**Profile:** `TokenisationEnabledTestProfile` (unchanged — already extends ledger-enabled)

**New injection:**
- `LedgerMerkleFrontierRepository`

**New test: `erasure_preservesMerkleFrontierIntegrity()`**

1. Fire a `CaseLifecycleEvent` (COMPLETED) to create entries through the capture path (not seeded directly) — this ensures the entry has a computed digest and frontier
2. Verify frontier exists for the subjectId before erasure
3. Run erasure via REST endpoint
4. Verify:
   - Frontier nodes still exist for the subjectId (erasure pseudonymises actor fields, doesn't delete entries)
   - `ErasureReceiptLedgerEntry` has non-null `digest`
   - Total frontier node count reflects the additional receipt entry

## What's NOT in scope

- Testing `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` internals — they're engine code, tested in engine
- Testing trust scoring accuracy — that's `casehub-ledger`'s responsibility
- Adding new Flyway migrations — existing migrations cover all required tables
- Changing `casehub.ledger.enabled=false` default — it stays as default

## Acceptance criteria mapping

| AC | Test |
|----|------|
| At least one `@QuarkusTest` with `casehub.ledger.enabled=true` | Already met (7 tests); new tests add to this |
| Ledger entry creation on case completion | `completedCase_writesCaseLedgerEntryAndAdvancesFrontier()` |
| Worker decision entry on review completion | `workerDecision_writesEntryWithTrustEnrichment()` |
| Merkle frontier integrity | All three test classes verify frontier |
| GDPR erasure tamper-evident receipt | `erasure_preservesMerkleFrontierIntegrity()` |

## References

- `MergeDecisionObserverTest.java` — existing test pattern (fireAsync + await)
- `GdprErasureResourceTest.java` — existing erasure test pattern
- `LedgerEnabledTestProfile.java` — H2 + Flyway + excluded beans
- `TrustScoringTestProfile.java` — ledger + trust scoring enabled
- `CaseLedgerEventCapture` (engine-ledger-core) — CDI adapter + plain Java capture
- `WorkerDecisionEventCapture` (engine-ledger-core) — same pattern
- `LedgerMerkleFrontierRepository` SPI — `findBySubjectId()` returns frontier nodes; count = `Integer.bitCount(entryCount)`
- `WorkerDecisionEvent` — record: caseId, tenancyId, workerId, capabilityTag, traceId, selectionContext, reasoning
- PP-20260521-134c38 — HITL test pre-seeding protocol (not directly needed but related CDI event testing pattern)
