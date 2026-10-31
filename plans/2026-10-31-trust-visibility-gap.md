# Trust Visibility Gap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #287 — Trust visibility gap — restore TrustScoreSnapshot
**Issue group:** #287

**Goal:** Restore trust score trend data and before/after gate scores in the governance UI by wiring `TrustQueryService` to the ledger's `TrustScoreSnapshot` API.

**Architecture:** Two stub methods in `TrustQueryService` get real implementations. `trustTrend()` queries `TrustScoreSnapshot` named queries via EntityManager. `loadFeedback()` gains a `workerId` parameter and batch-queries snapshots to populate before/after trust scores in `GateDecision` records. No new classes, no new dependencies, no schema changes.

**Tech Stack:** Java 21, Quarkus 3.39.3, JPA named queries, ledger-api model classes

## Global Constraints

- Use EntityManager directly with named queries (D1) — not `TrustScoreSnapshotRepository` SPI
- `TrustScoreSnapshotBase` is in `io.casehub.ledger.api.model` (ledger-api, already on classpath)
- Named queries are defined on `TrustScoreSnapshot` entity (ledger-jpa-common, already on classpath)
- All edits are in `app/` module only — no cross-module changes
- IntelliJ MCP for structural editing; verify with `ide_diagnostics` after edits

---

## Batch 1: Trust trend and gate score restoration

### Task 1: Implement `trustTrend()` — query snapshot history

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/governance/TrustQueryServiceTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/TrustQueryService.java`

**Interfaces:**
- Consumes: `TrustScoreSnapshotBase` (ledger-api), named queries `TrustScoreSnapshot.findByActorAndCapability` and `TrustScoreSnapshot.findByActorGlobal`
- Produces: `trustTrend(String actorId, String capabilityTag, int limit)` returns `List<TrustTrendPoint>` with real data

- [ ] **Step 1: Replace the placeholder test with failing tests**

Replace `trustTrend_returnsEmptyUntilSnapshotEntityRestored` with three new tests. Use `ide_replace_member` on the existing test, then `ide_insert_member` for the additional tests.

```java
@SuppressWarnings("unchecked")
@Test
void trustTrend_queriesCapabilitySnapshots() {
    var snap1 = new TrustScoreSnapshotBase("agent-1", ScoreType.CAPABILITY,
            "security-review", null, 0.75, 0.70, Instant.parse("2026-08-02T10:00:00Z"));
    var snap2 = new TrustScoreSnapshotBase("agent-1", ScoreType.CAPABILITY,
            "security-review", null, 0.70, 0.65, Instant.parse("2026-08-01T10:00:00Z"));

    final TypedQuery<TrustScoreSnapshotBase> mockQuery = mock(TypedQuery.class);
    when(em.createNamedQuery("TrustScoreSnapshot.findByActorAndCapability", TrustScoreSnapshotBase.class))
            .thenReturn(mockQuery);
    when(mockQuery.setParameter(anyString(), any())).thenReturn(mockQuery);
    when(mockQuery.setMaxResults(anyInt())).thenReturn(mockQuery);
    when(mockQuery.getResultList()).thenReturn(List.of(snap1, snap2));

    final var result = service.trustTrend("agent-1", "security-review", 30);

    assertThat(result).hasSize(2);
    assertThat(result.get(0).score()).isEqualTo(0.75);
    assertThat(result.get(0).previousScore()).isEqualTo(0.70);
    assertThat(result.get(0).timestamp()).isEqualTo(Instant.parse("2026-08-02T10:00:00Z"));
    assertThat(result.get(1).score()).isEqualTo(0.70);
}

@SuppressWarnings("unchecked")
@Test
void trustTrend_queriesGlobalSnapshotsWhenNoCapability() {
    var snap = new TrustScoreSnapshotBase("agent-1", ScoreType.GLOBAL,
            null, null, 0.72, 0.68, Instant.parse("2026-08-02T10:00:00Z"));

    final TypedQuery<TrustScoreSnapshotBase> mockQuery = mock(TypedQuery.class);
    when(em.createNamedQuery("TrustScoreSnapshot.findByActorGlobal", TrustScoreSnapshotBase.class))
            .thenReturn(mockQuery);
    when(mockQuery.setParameter(anyString(), any())).thenReturn(mockQuery);
    when(mockQuery.setMaxResults(anyInt())).thenReturn(mockQuery);
    when(mockQuery.getResultList()).thenReturn(List.of(snap));

    final var result = service.trustTrend("agent-1", null, 30);

    assertThat(result).hasSize(1);
    assertThat(result.get(0).score()).isEqualTo(0.72);
}

@SuppressWarnings("unchecked")
@Test
void trustTrend_returnsEmptyForUnknownActor() {
    final TypedQuery<TrustScoreSnapshotBase> mockQuery = mock(TypedQuery.class);
    when(em.createNamedQuery(anyString(), eq(TrustScoreSnapshotBase.class)))
            .thenReturn(mockQuery);
    when(mockQuery.setParameter(anyString(), any())).thenReturn(mockQuery);
    when(mockQuery.setMaxResults(anyInt())).thenReturn(mockQuery);
    when(mockQuery.getResultList()).thenReturn(List.of());

    final var result = service.trustTrend("unknown", "security-review", 30);

    assertThat(result).isEmpty();
}
```

Add imports to test file:
```java
import io.casehub.ledger.api.model.TrustScoreSnapshotBase;
import io.casehub.ledger.api.model.ScoreType;
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl app -Dtest=TrustQueryServiceTest -Dmaven.test.failure.ignore=false`
Expected: FAIL — `trustTrend` returns empty list, tests expect populated results.

- [ ] **Step 3: Implement `trustTrend()`**

Use `ide_replace_member` to replace the `trustTrend` method body in `TrustQueryService.java`.

Remove the TODO comment on line 81 and replace the stub implementation:

```java
@Transactional
public List<TrustTrendPoint> trustTrend(String actorId, String capabilityTag, int limit) {
    String queryName = capabilityTag != null
            ? "TrustScoreSnapshot.findByActorAndCapability"
            : "TrustScoreSnapshot.findByActorGlobal";

    var query = em.createNamedQuery(queryName, TrustScoreSnapshotBase.class)
            .setParameter("actorId", actorId)
            .setMaxResults(Math.min(limit, 200));

    if (capabilityTag != null) {
        query.setParameter("capabilityTag", capabilityTag);
    }

    return query.getResultList().stream()
            .map(s -> new TrustTrendPoint(s.occurredAt, s.score, s.previousScore))
            .toList();
}
```

Add imports to `TrustQueryService.java`:
```java
import io.casehub.ledger.api.model.TrustScoreSnapshotBase;
import io.casehub.ledger.api.model.ScoreType;
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl app -Dtest=TrustQueryServiceTest -Dmaven.test.failure.ignore=false`
Expected: ALL PASS

- [ ] **Step 5: Verify with `ide_diagnostics`**

Run `ide_diagnostics` on `TrustQueryService.java` to confirm no compilation errors.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/governance/TrustQueryService.java \
       app/src/test/java/io/casehub/devtown/app/governance/TrustQueryServiceTest.java
git commit -m "feat(#287): implement trustTrend with TrustScoreSnapshot named queries

Refs #287"
```

---

### Task 2: Implement `loadFeedback()` before/after trust scores

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/governance/TrustQueryServiceTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/TrustQueryService.java`

**Interfaces:**
- Consumes: `TrustScoreSnapshotBase` (ledger-api), named query `TrustScoreSnapshot.findByActorAndTimeRange`, `ScoreType` enum
- Produces: `loadFeedback(UUID entryId, String workerId)` populates `GateDecision.trustScoreBefore` and `trustScoreAfter` from matching snapshots

- [ ] **Step 1: Write failing test — scores populated from snapshots**

Add new test using `ide_insert_member`. This test creates an attestation with a `trustDimension`, a matching snapshot, and verifies the before/after scores are populated.

```java
@SuppressWarnings("unchecked")
@Test
void routingDetail_populatesTrustScoresFromSnapshots() {
    final UUID entryId = UUID.randomUUID();
    final var entry = new WorkerDecisionEntry();
    entry.id = entryId;
    entry.workerId = "agent-1";
    entry.capabilityTag = "security-review";
    entry.routingRationale = """
            {"strategyId":"trust-weighted","selected":{"workerId":"agent-1",\
            "trustScore":0.72,"workloadScore":0.8,"phase":"QUALIFIED",\
            "observations":14,"finalScore":0.74},"alternatives":[],"policy":{}}""";

    final var attestation = new LedgerAttestation();
    attestation.verdict = AttestationVerdict.FLAGGED;
    attestation.attestorId = "devtown:incident-feedback";
    attestation.evidence = "Incident INC-001";
    attestation.occurredAt = Instant.parse("2026-08-02T15:00:00Z");
    attestation.trustDimension = "review-thoroughness";

    var snapshot = new TrustScoreSnapshotBase("agent-1", ScoreType.DIMENSION,
            null, "review-thoroughness", 0.65, 0.72,
            Instant.parse("2026-08-02T15:00:00Z"));

    when(em.find(WorkerDecisionEntry.class, entryId)).thenReturn(entry);

    final TypedQuery<Object> attestationQuery = mock(TypedQuery.class, "attestationQuery");
    final TypedQuery<Object> snapshotQuery = mock(TypedQuery.class, "snapshotQuery");

    when(em.createQuery(anyString(), any(Class.class))).thenReturn(attestationQuery);
    when(attestationQuery.setParameter(anyString(), any())).thenReturn(attestationQuery);
    when(attestationQuery.getResultList()).thenReturn(List.of(attestation));

    when(em.createNamedQuery(anyString(), eq(TrustScoreSnapshotBase.class)))
            .thenReturn(snapshotQuery);
    when(snapshotQuery.setParameter(anyString(), any())).thenReturn(snapshotQuery);
    when(snapshotQuery.getResultList()).thenReturn(List.of(snapshot));

    final var detail = service.routingDetail("agent-1", entryId);

    assertThat(detail.feedback()).hasSize(1);
    final var gate = detail.feedback().get(0);
    assertThat(gate.trustScoreBefore()).isEqualTo(0.72);
    assertThat(gate.trustScoreAfter()).isEqualTo(0.65);
    assertThat(gate.dimension()).isEqualTo("review-thoroughness");
}

@SuppressWarnings("unchecked")
@Test
void routingDetail_fallsBackToZeroWhenNoSnapshots() {
    final UUID entryId = UUID.randomUUID();
    final var entry = new WorkerDecisionEntry();
    entry.id = entryId;
    entry.workerId = "agent-1";
    entry.capabilityTag = "security-review";
    entry.routingRationale = """
            {"strategyId":"trust-weighted","selected":{"workerId":"agent-1",\
            "trustScore":0.72,"workloadScore":0.8,"phase":"QUALIFIED",\
            "observations":14,"finalScore":0.74},"alternatives":[],"policy":{}}""";

    final var attestation = new LedgerAttestation();
    attestation.verdict = AttestationVerdict.APPROVED;
    attestation.attestorId = "reviewer-1";
    attestation.evidence = "LGTM";
    attestation.occurredAt = Instant.parse("2026-08-02T15:00:00Z");
    attestation.trustDimension = "review-thoroughness";

    when(em.find(WorkerDecisionEntry.class, entryId)).thenReturn(entry);

    final TypedQuery<Object> attestationQuery = mock(TypedQuery.class, "attestationQuery");
    final TypedQuery<Object> snapshotQuery = mock(TypedQuery.class, "snapshotQuery");

    when(em.createQuery(anyString(), any(Class.class))).thenReturn(attestationQuery);
    when(attestationQuery.setParameter(anyString(), any())).thenReturn(attestationQuery);
    when(attestationQuery.getResultList()).thenReturn(List.of(attestation));

    when(em.createNamedQuery(anyString(), eq(TrustScoreSnapshotBase.class)))
            .thenReturn(snapshotQuery);
    when(snapshotQuery.setParameter(anyString(), any())).thenReturn(snapshotQuery);
    when(snapshotQuery.getResultList()).thenReturn(List.of());

    final var detail = service.routingDetail("agent-1", entryId);

    assertThat(detail.feedback()).hasSize(1);
    assertThat(detail.feedback().get(0).trustScoreBefore()).isEqualTo(0.0);
    assertThat(detail.feedback().get(0).trustScoreAfter()).isEqualTo(0.0);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl app -Dtest=TrustQueryServiceTest -Dmaven.test.failure.ignore=false`
Expected: FAIL — `routingDetail_populatesTrustScoresFromSnapshots` fails because scores are hardcoded 0.0.

- [ ] **Step 3: Implement `loadFeedback()` with snapshot lookup**

Use `ide_replace_member` to replace the `loadFeedback` method. Change signature to `loadFeedback(UUID entryId, String workerId)`. Remove the TODO comment on line 200.

```java
@SuppressWarnings("unchecked")
private List<GateDecision> loadFeedback(UUID entryId, String workerId) {
    List<LedgerAttestation> attestations = em
            .createQuery("SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId = :entryId", LedgerAttestation.class)
            .setParameter("entryId", entryId)
            .getResultList();

    if (attestations.isEmpty()) {
        return Collections.emptyList();
    }

    Instant minTime = attestations.stream().map(a -> a.occurredAt).filter(t -> t != null).min(Instant::compareTo).orElse(null);
    Instant maxTime = attestations.stream().map(a -> a.occurredAt).filter(t -> t != null).max(Instant::compareTo).orElse(null);

    List<TrustScoreSnapshotBase> snapshots = Collections.emptyList();
    if (minTime != null && maxTime != null) {
        snapshots = em.createNamedQuery("TrustScoreSnapshot.findByActorAndTimeRange", TrustScoreSnapshotBase.class)
                .setParameter("actorId", workerId)
                .setParameter("from", minTime.minusSeconds(2))
                .setParameter("to", maxTime.plusSeconds(2))
                .getResultList();
    }

    final List<TrustScoreSnapshotBase> snapshotsFinal = snapshots;
    return attestations.stream().map(a -> {
        double before = 0.0;
        double after = 0.0;
        TrustScoreSnapshotBase match = findMatchingSnapshot(a, snapshotsFinal);
        if (match != null) {
            before = match.previousScore;
            after = match.score;
        }
        return new GateDecision(
                a.verdict != null ? a.verdict.name() : "",
                a.attestorId,
                a.evidence,
                before, after,
                a.trustDimension != null ? a.trustDimension : "");
    }).toList();
}
```

- [ ] **Step 4: Add `findMatchingSnapshot` private helper**

Use `ide_insert_member` to add the helper method after `loadFeedback`:

```java
private TrustScoreSnapshotBase findMatchingSnapshot(LedgerAttestation attestation,
        List<TrustScoreSnapshotBase> snapshots) {
    if (attestation.occurredAt == null) {
        return null;
    }
    TrustScoreSnapshotBase best = null;
    long bestDistance = Long.MAX_VALUE;
    for (var snapshot : snapshots) {
        boolean dimensionMatch = attestation.trustDimension != null
                && attestation.trustDimension.equals(snapshot.dimensionKey);
        boolean globalMatch = snapshot.scoreType == ScoreType.GLOBAL;
        if (!dimensionMatch && !globalMatch) {
            continue;
        }
        if (snapshot.occurredAt == null) {
            continue;
        }
        long distance = Math.abs(java.time.Duration.between(
                attestation.occurredAt, snapshot.occurredAt).toMillis());
        if (dimensionMatch && (best == null || best.scoreType == ScoreType.GLOBAL || distance < bestDistance)) {
            best = snapshot;
            bestDistance = distance;
        } else if (globalMatch && best == null) {
            best = snapshot;
            bestDistance = distance;
        }
    }
    return best;
}
```

- [ ] **Step 5: Update `routingDetail()` call site**

Use `ide_replace_member` to update the call from `loadFeedback(entryId)` to `loadFeedback(entryId, actorId)` in `routingDetail`:

```java
List<GateDecision> feedback = loadFeedback(entryId, actorId);
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl app -Dtest=TrustQueryServiceTest -Dmaven.test.failure.ignore=false`
Expected: ALL PASS

- [ ] **Step 7: Run full module tests**

Run: `mvn test -pl app`
Expected: ALL PASS — no regressions

- [ ] **Step 8: Verify with `ide_diagnostics`**

Run `ide_diagnostics` on `TrustQueryService.java` to confirm no compilation errors.

- [ ] **Step 9: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/governance/TrustQueryService.java \
       app/src/test/java/io/casehub/devtown/app/governance/TrustQueryServiceTest.java
git commit -m "feat(#287): populate gate decision trust scores from snapshots

Wire loadFeedback to query TrustScoreSnapshot by actor and time range,
matching attestations to their corresponding score changes by dimension
and nearest timestamp. Falls back to 0.0 when no snapshot matches.

Closes #287"
```

---

## References

- [2026-10-31-trust-visibility-gap-design.md] — design spec this plan implements
- [TrustQueryService.java:81-85, 190-208] — the two TODO stubs
- [TrustScoreSnapshot.java] (ledger-jpa-common) — JPA entity with named queries
- [TrustScoreSnapshotBase.java] (ledger-api) — base model with score/previousScore
- [TrustScoreSnapshotRepository.java] (ledger-api) — SPI (not used, D1 decision)
- [PerActorTrustComputerCore.java] (ledger-core) — snapshot creation confirms timing relationship
- [GitHub #287] — Trust visibility gap — restore TrustScoreSnapshot
