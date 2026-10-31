# Trust Visibility Gap — Restore TrustScoreSnapshot

**Issue:** casehubio/devtown#287
**Branch:** issue-287-trust-visibility-gap
**Date:** 2026-10-31

## Problem

Two TODOs in `TrustQueryService.java` reference `TrustScoreSnapshot` which was believed removed from the ledger SNAPSHOT. Investigation reveals the entity was never removed — it exists with a complete API in the ledger. The TODOs were written during a transient SNAPSHOT build issue.

**Current state:**
- `trustTrend()` (line 83) returns `Collections.emptyList()`
- `loadFeedback()` (line 200) hardcodes `0.0` for `trustScoreBefore` and `trustScoreAfter`

**Impact:** The governance UI's trust trend charts show no data, and routing decision detail views show zeroed gate scores instead of actual before/after trust deltas.

## Ledger API Available

The ledger provides everything needed:

| Component | Location | Role |
|-----------|----------|------|
| `TrustScoreSnapshotBase` | `ledger-api` | Fields: actorId, scoreType, capabilityTag, dimensionKey, **score**, **previousScore**, occurredAt |
| `TrustScoreSnapshot` | `ledger-jpa-common` | JPA entity with named queries, ordered by `occurredAt DESC` |
| `TrustScoreSnapshotRepository` | `ledger-api` SPI | find by global/capability/dimension/timeRange |
| `JpaTrustScoreSnapshotRepository` | `ledger-runtime` | `@DefaultBean @ApplicationScoped`, already in devtown's CDI container |
| `PerActorTrustComputerCore` | `ledger-core` | Saves snapshots (GLOBAL, CAPABILITY, DIMENSION, CAPABILITY_DIMENSION) on every trust recomputation |

Named queries available on `TrustScoreSnapshot`:
- `findByActorGlobal` — by actorId, GLOBAL scoreType
- `findByActorAndCapability` — by actorId + capabilityTag, CAPABILITY scoreType
- `findByActorAndDimension` — by actorId + dimensionKey, DIMENSION scoreType
- `findByActorAndTimeRange` — by actorId + time window

All ordered by `occurredAt DESC`.

## Design

No new classes. Two method implementations in `TrustQueryService`, one signature change to `loadFeedback()`.

### Fix 1: `trustTrend()`

Replace the empty-list stub with a real query.

**Logic:**
1. If `capabilityTag` is non-null: use named query `TrustScoreSnapshot.findByActorAndCapability` with actorId + capabilityTag
2. If `capabilityTag` is null: use named query `TrustScoreSnapshot.findByActorGlobal` with actorId
3. Apply `setMaxResults(Math.min(limit, 200))` for pagination (cap at 200, matching existing pattern from `routingHistory`)
4. Map each `TrustScoreSnapshotBase` → `TrustTrendPoint(occurredAt, score, previousScore)`

**Query uses EntityManager directly** (D1) — matches existing class pattern, enables `setMaxResults()` which the SPI doesn't support.

### Fix 2: `loadFeedback()` before/after scores

Change signature from `loadFeedback(UUID entryId)` to `loadFeedback(UUID entryId, String workerId)`. The `routingDetail()` method already has `actorId` (which is the workerId) available and passes it through.

**Logic:**
1. Fetch attestations as before
2. If attestations are empty, return empty list (unchanged)
3. Compute time window: min/max `occurredAt` from attestations, with 2-second buffer on each side
4. Single query: `TrustScoreSnapshot.findByActorAndTimeRange` for `workerId` across the buffered window
5. For each attestation, find the best-matching snapshot:
   - If attestation has `trustDimension` set: prefer snapshot with matching `dimensionKey` and nearest `occurredAt`
   - If attestation has `capabilityTag` set (non-GLOBAL): prefer snapshot with matching `capabilityTag` and nearest `occurredAt`
   - Otherwise: prefer GLOBAL snapshot with nearest `occurredAt`
6. If match found: `trustScoreBefore = snapshot.previousScore`, `trustScoreAfter = snapshot.score`
7. If no match: fall back to `0.0, 0.0` (graceful degradation — same as current behaviour)

**Batch approach** (D2) — one DB query for all attestations, local matching. Avoids N+1.

### Imports Required

```java
import io.casehub.ledger.api.model.TrustScoreSnapshotBase;
import io.casehub.ledger.api.model.ScoreType;
```

No new dependencies — `casehub-ledger` is already in `app/pom.xml`.

## Testing

### Unit tests (TrustQueryServiceTest)

**Replace** `trustTrend_returnsEmptyUntilSnapshotEntityRestored`:
- `trustTrend_queriesCapabilitySnapshots` — mock EM with named query returning 2 snapshots, verify mapping to TrustTrendPoint
- `trustTrend_queriesGlobalSnapshotsWhenNoCapability` — null capabilityTag → uses global named query
- `trustTrend_returnsEmptyForUnknownActor` — empty result list → empty response

**Add** loadFeedback before/after tests:
- `routingDetail_populatesTrustScoresFromSnapshots` — attestation with matching snapshot → scores populated
- `routingDetail_fallsBackToZeroWhenNoSnapshots` — no matching snapshots → 0.0 (existing behaviour preserved)

### No integration test changes

The `@QuarkusTest` layer is not affected — `TrustQueryService` is unit-tested with mocked EntityManager.

## REST API

No changes. `DevtownTrustApi.trustTrend()` already delegates to `TrustQueryService.trustTrend()` — the response type `TrustTrendPoint` is unchanged.

## Frontend

No changes. The frontend TypeScript type `TrustScoreSnapshot` in `graph-stencil-case` already has matching fields. The governance UI currently renders "Data unavailable" for empty trend data — once the endpoint returns real data, charts will populate automatically.

## References

- `TrustQueryService.java` lines 81-85, 190-208 — the two TODOs
- `TrustScoreSnapshot.java` (ledger-jpa-common) — JPA entity with named queries
- `TrustScoreSnapshotBase.java` (ledger-api) — base model with score/previousScore
- `TrustScoreSnapshotRepository.java` (ledger-api) — SPI interface
- `JpaTrustScoreSnapshotRepository.java` (ledger-runtime) — JPA implementation, `@DefaultBean`
- `PerActorTrustComputerCore.java` (ledger-core) — snapshot creation on trust recomputation
- `DevtownTrustApi.java` — REST endpoint (no changes needed)
- `2026-10-29-review-intelligence-workbench-design.md` — documents the TrustScoreSnapshot gap as known limitation
