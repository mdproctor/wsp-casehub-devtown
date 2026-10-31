## D1: Snapshot query strategy — EntityManager with named queries

**Choice:** Use EntityManager directly with TrustScoreSnapshot named queries + `setMaxResults()`
**Alternatives:**
- Inject `TrustScoreSnapshotRepository` SPI — cleaner abstraction but no pagination support, inconsistent with existing class pattern
**Rationale:** Every other method in `TrustQueryService` uses `em.createQuery()`/`em.createNamedQuery()`. The SPI's `findCapabilitySnapshots()` returns all results without limit, which is wasteful for `trustTrend()` that typically needs only the last 30. EntityManager lets us apply `setMaxResults()` efficiently.
**Trade-offs:** Slightly tighter coupling to JPA named queries, but the class is already fully JPA-coupled
**Sources:** TrustQueryService.java (existing EM pattern), TrustScoreSnapshot.java (named queries), TrustScoreSnapshotRepository.java (SPI methods)
**Exploration:** quick
**Status:** captured

## D2: Attestation-snapshot correlation for before/after scores

**Choice:** Batch query by workerId + time range, match locally by dimension/capability + nearest timestamp
**Alternatives:**
- Per-attestation query — simple but N+1 DB queries per routing detail call
- Global snapshots only — loses per-dimension granularity
**Rationale:** One DB query retrieves all relevant snapshots for the attestation time window. Local matching by dimension/capability + nearest `occurredAt` gives correct per-dimension scores. Efficient and accurate.
**Trade-offs:** Slightly more complex matching logic; if snapshot timing diverges significantly from attestation timing, matches could be imprecise — mitigated by a small time buffer on the query window
**Sources:** PerActorTrustComputerCore.computeForActor() (snapshot creation timing), LedgerAttestation.java (attestation fields), TrustScoreSnapshotBase.java (score/previousScore fields)
**Exploration:** quick
**Status:** captured
