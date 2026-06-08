# Layer 4 — Tamper-Evident Merge Decision Audit Trail + Compliance Report

**Issues:** devtown#73 (Layer 4), devtown#7 (compliance report)
**Branch:** `issue-73-layer4-ledger-audit`
**Date:** 2026-06-08
**Revision:** 4 (addresses 10-point review + 4-point follow-up + 2-point final review)

---

## Summary

Layer 4 makes devtown's existing `casehub-engine-ledger` wiring real: a domain-specific `MergeDecisionLedgerEntry` captures structured PR merge decisions in the ledger audit trail, a `MergeDecisionObserver` derives the decision from terminal case state, `ComplianceSupplement` records EU AI Act Art.12 metadata, and a compliance report endpoint (#7) assembles per-case evidence across four regulatory requirement dimensions.

The engine-level `CaseLedgerEntry` and `WorkerDecisionEntry` remain unchanged — they handle case lifecycle and worker dispatch events. Layer 4 adds the one domain event those generic entries cannot capture: the merge decision itself, with structured PR metadata.

### Tamper-evidence scope

The Merkle chain proves entry **ordering and existence** — it does not prove content integrity of individual fields. `LedgerMerkleTree.canonicalBytes()` hashes `subjectId|sequenceNumber|entryType|actorId|actorRole|occurredAt` only. Neither join-table columns (`decision`, `pr_number`) nor `supplementJson` are in the hash. This is a platform-wide characteristic, not specific to devtown.

**Consequence:** someone with database write access could alter `decision = REJECTED` to `APPROVED` without breaking the chain. The same exposure exists for `CaseLedgerEntry.caseStatus` and `WorkerDecisionEntry.trustScoreAtRouting`. Defence is DB-level access controls and audit logging.

**Foundation issue filed:** ledger#128 proposes adding `supplementJson` to the canonical bytes, which would make all supplement data hash-protected. Until that ships, devtown's tamper-evidence claim covers ordering/existence, not field integrity.

### Layer dependency: C3 → C4

ARC42STORIES §9.2 states "C3 before C4: qhorus messaging generates the `MessageLedgerEntry` chain that makes C4's tamper-evident audit meaningful." Layer 3 (devtown#52) is already complete — this dependency is satisfied.

The compliance report queries `CaseLedgerEntry`, `WorkerDecisionEntry`, and `MergeDecisionLedgerEntry` — it does not query `MessageLedgerEntry`. The qhorus commitment chain records per-agent COMMAND/RESPONSE/DECLINE interactions at the message level. The compliance report focuses on case-level events: lifecycle transitions, worker dispatch decisions, and the merge decision. The per-message audit trail enriches the overall audit record but is not required for the compliance report's four requirement dimensions.

## Prior State

`casehub-engine-ledger` is already a compile dependency in `app/pom.xml` with:
- Jandex indexing configured (prod and test)
- Flyway migration locations: `db/engine-ledger/migration` (V2000 `case_ledger_entry`, V2001 `worker_decision_entry`)
- Trust routing YAML configs loaded (`trust-routing.yaml`, `trust-gate.yaml`)
- `casehub.ledger.enabled=false` in test properties (disables `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` in tests)
- `TrustWeightedAgentStrategy`, `WorkerDecisionEventCapture`, `CaseLedgerEventCapture` all discoverable via CDI

What's missing: no domain-specific ledger entry for merge decisions, no `ComplianceSupplement` attachment, no compliance report endpoint, no tests exercising ledger writes, no observer for terminal case state.

---

## Part 1: MergeDecisionLedgerEntry (#73)

### Entity

`MergeDecisionLedgerEntry extends LedgerEntry` in `app/src/main/java/io/casehub/devtown/app/ledger/`.

| Column | Type | Nullable | Purpose |
|--------|------|----------|---------|
| `pr_number` | INTEGER | NOT NULL | PR number in the repository |
| `repository` | VARCHAR(255) | NOT NULL | `owner/repo` identifier |
| `commit_sha` | VARCHAR(40) | nullable | HEAD commit SHA at decision time |
| `decision` | VARCHAR(20) | NOT NULL | APPROVED or REJECTED (see Decision Semantics) |
| `case_id` | UUID | NOT NULL | Convenience alias for `subjectId` (same value) |
| `tenancy_id` | VARCHAR(64) | NOT NULL | Tenant isolation — index-only scans without base table join |

JPA annotations:
- `@Entity`, `@Table(name = "merge_decision_ledger_entry")`, `@DiscriminatorValue("MERGE_DECISION")`
- Indexes on `case_id` and `tenancy_id`

### Decision Semantics

Two terminal outcomes, derived from `CaseStatus`:

| CaseStatus | Decision | Meaning |
|------------|----------|---------|
| `COMPLETED` | `APPROVED` | All goals met (`pr-approved` ∧ `security-verified` ∧ `ci-passing`). The merge binding fires. |
| `CANCELLED` | `REJECTED` | Case explicitly aborted — a human reviewer or the orchestrator determined the PR should not merge. |

`FAULTED` does **not** produce a merge decision. It is an infrastructure error (engine failure, CDI wiring error, unhandled exception), not a domain event. The compliance report surfaces FAULTED cases separately under the audit chain requirement as entries without a corresponding merge decision.

**Why no BLOCKED:** A case in `WAITING` or `RUNNING` with unsatisfiable goals (e.g., security review returned REJECTED but the case hasn't been cancelled) is effectively blocked — but it is not terminal. The engine does not detect goal impossibility. The case remains open until explicitly cancelled or until a timeout fires. BLOCKED is a runtime observation, not a decision event. Surfacing stuck cases is an operational concern (devtown#17, Epic 10).

### MergeDecisionObserver

A new `@ApplicationScoped` bean in `app/src/main/java/io/casehub/devtown/app/ledger/` that observes `CaseLifecycleEvent`, derives the merge decision, and writes the ledger entry directly.

No intermediate `MergeDecisionEvent` record. No separate writer bean. The observer is the single writer for `MergeDecisionLedgerEntry` — since only one code path produces merge decisions, the `harness-ledger-writer` protocol's dedicated-writer requirement does not apply (that protocol targets the multi-writer concurrent-sequence-number problem).

```java
@ApplicationScoped
public class MergeDecisionObserver {

    @Inject CrossTenantCaseInstanceRepository caseInstanceRepo;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject LedgerConfig ledgerConfig;

    @Transactional
    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (!ledgerConfig.enabled()) return;

        String decision = switch (event.caseStatus()) {
            case "COMPLETED" -> "APPROVED";
            case "CANCELLED" -> "REJECTED";
            default -> null;
        };
        if (decision == null) return;

        CaseInstance ci = caseInstanceRepo.findByUuid(event.caseId())
            .await().atMost(Duration.ofSeconds(5));
        if (ci == null) return;

        CaseContext ctx = ci.getCaseContext();
        if (ctx == null) return;

        String repo = ctx.getPathAsString("pr.repo");
        String prIdStr = ctx.getPathAsString("pr.id");
        String headSha = ctx.getPathAsString("pr.headSha");
        if (repo == null || prIdStr == null) return;

        int prNumber;
        try { prNumber = Integer.parseInt(prIdStr); }
        catch (NumberFormatException e) { return; }

        MergeDecisionLedgerEntry entry = new MergeDecisionLedgerEntry();
        entry.subjectId = event.caseId();
        entry.caseId = event.caseId();
        entry.tenancyId = event.tenancyId();
        entry.entryType = LedgerEntryType.EVENT;
        entry.prNumber = prNumber;
        entry.repository = repo;
        entry.commitSha = headSha;
        entry.decision = decision;
        entry.actorId = "system";
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "ORCHESTRATOR";
        entry.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);

        // Link to the CaseLedgerEntry that recorded the terminal state transition.
        // Uses findLatestBySubjectId() (inherited from JpaLedgerEntryRepository,
        // correct @LedgerPersistenceUnit) — NOT findLatestByCaseId() which uses
        // an unqualified EntityManager (engine#450).
        // Best-effort: if CaseLedgerEventCapture hasn't committed yet (observer
        // ordering is undefined for @ObservesAsync), causedByEntryId stays null.
        ledgerRepo.findLatestBySubjectId(event.caseId())
            .filter(latest -> latest instanceof CaseLedgerEntry cle
                && event.caseStatus().equals(cle.caseStatus))
            .ifPresent(latest -> entry.causedByEntryId = latest.id);

        ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "casehub-devtown:pr-review-v1";
        cs.humanOverrideAvailable = true;
        cs.contestationUri = "/api/reviews/" + prNumber + "/contest";
        entry.attach(cs);

        ledgerRepo.save(entry);
    }
}
```

**Why the observer writes directly:** `JpaLedgerEntryRepository.save()` handles sequence number allocation (via `LedgerSequenceAllocator.nextSequenceNumber()`), digest computation, Merkle frontier update, and supplement serialisation. The caller's only responsibility is populating domain fields and calling `save()`. Any `sequenceNumber` set by the caller is overwritten by the infrastructure.

**Actor fields:** The observer sets `actorId = "system"` and `actorRole = "ORCHESTRATOR"` because the merge decision is a system-level derivation from accumulated case context, not a direct human or agent action. `CaseLifecycleEvent` carries `actorId = null` and `actorRole = "System"` for engine-triggered transitions — the observer translates to the ledger's identity model.

**Observer race with `CaseLedgerEventCapture`:** Both this observer and `CaseLedgerEventCapture` (foundation) observe `@ObservesAsync CaseLifecycleEvent`. Both call `ledgerRepo.save()` with `subjectId = caseId`. CDI async observer ordering is undefined — the merge decision entry may receive a lower sequence number than the COMPLETED lifecycle entry. `LedgerSequenceAllocator` serialises allocation correctly (no data corruption), but logical ordering is unguaranteed.

The `causedByEntryId` link makes the causal relationship explicit regardless of sequence numbers. The compliance report's audit chain walks both `sequenceNumber` order (for timeline) and `causedByEntryId` (for provenance). If the lifecycle entry is committed after the merge decision (race), `causedByEntryId` is null — the compliance report reports PARTIAL for the provenance dimension, which is accurate: the entries exist but the causal link was not captured.

**`CrossTenantCaseInstanceRepository` tech debt:** `findByUuid()` returns `Uni<CaseInstance>`. The blocking `.await()` inside an `@ObservesAsync` observer works (the async observer runs on a worker thread, not the event loop), but `CrossTenantCaseInstanceRepository`'s contract says "for startup recovery services only." This is accepted tech debt, identical to `ReviewOutcomeObserver`. Resolution: when `CaseLifecycleEvent` carries the full case context (or at least PR metadata from the initial input data), the `CrossTenantCaseInstanceRepository` lookup becomes unnecessary.

### Flyway Migration

`V2002__merge_decision_ledger_entry.sql` in `app/src/main/resources/db/devtown/migration/`:

V2002 because `db/engine-ledger/migration/` already contains V2000 (`case_ledger_entry`) and V2001 (`worker_decision_entry`). Flyway merges all configured locations into a single ordered sequence — a second V2000 in `db/devtown/migration/` would produce a duplicate-version error at startup.

```sql
CREATE TABLE merge_decision_ledger_entry (
    id          UUID NOT NULL,
    pr_number   INTEGER NOT NULL,
    repository  VARCHAR(255) NOT NULL,
    commit_sha  VARCHAR(40),
    decision    VARCHAR(20) NOT NULL,
    case_id     UUID NOT NULL,
    tenancy_id  VARCHAR(64) NOT NULL,
    CONSTRAINT pk_merge_decision_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_merge_decision_ledger_entry_id
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);

CREATE INDEX idx_merge_decision_entry_case_id ON merge_decision_ledger_entry(case_id);
CREATE INDEX idx_merge_decision_entry_tenancy_id ON merge_decision_ledger_entry(tenancy_id);
```

### Configuration Changes

`application.properties` additions:

```properties
# Add devtown migration path
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/engine-ledger/migration,classpath:db/devtown/migration

# Add devtown ledger entity package to qhorus PU
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model,io.casehub.devtown.app.ledger
```

---

## Part 2: Compliance Report Endpoint (#7)

### Domain Model

All domain records in `review/` (pure Java):

**`RequirementStatus`** — enum: `CLOSED`, `PARTIAL`, `GAP`, `BREACHED`

**`CodeReviewComplianceEvidence`** — top-level composite:
```java
public record CodeReviewComplianceEvidence(
    UUID caseId,
    Instant generatedAt,
    AuditChainRequirement auditChain,
    ReviewSlaRequirement reviewSla,
    TrustRoutingRequirement trustRouting,
    GdprRequirement gdpr
) {}
```

No `signature` field — the Merkle tree root for the case is already inside `AuditChainRequirement` (computed via `LedgerVerificationService.treeRoot()`). A top-level cryptographic signature of the report itself would require signing infrastructure that doesn't exist. Add it when that infrastructure ships — don't carry a dead field.

**`AuditChainRequirement`** — Merkle chain integrity for EU AI Act Art.12:
- Walks `CaseLedgerEntry` + `WorkerDecisionEntry` + `MergeDecisionLedgerEntry` for the case
- Verifies Merkle chain via `LedgerVerificationService.verify()`
- Reports inclusion proofs per entry
- Status: CLOSED (chain verified, merge decision linked), PARTIAL (entries exist but chain unverified or missing link), GAP (no entries)

**`ReviewSlaRequirement`** — human review SLA compliance:
- Looks up the human approval WorkItem for this case
- Reports deadline, completion time, and whether SLA was met
- Status: CLOSED (completed within deadline), BREACHED (completed after or deadline passed), PARTIAL (WorkItem exists, pending), GAP (no WorkItem)

**`TrustRoutingRequirement`** — trust-weighted reviewer routing for EU AI Act Art.14:
- Reports `WorkerDecisionEntry` records: which agent, trust score at routing, threshold applied
- Status: CLOSED (all dispatched capabilities have routing records), PARTIAL (some missing), GAP (no dispatches)

**`GdprRequirement`** — static capability declaration (not runtime verification):
- Declares that `LedgerErasureService` (Art.17) is on classpath and `ActorIdentityProvider` tokenisation (pseudonymisation) is active
- This is a capability declaration for regulators — "the mechanism exists and is wired" — not evidence that erasure was invoked for this specific case
- Status: always CLOSED when `casehub-ledger` runtime is wired

Supporting records: `LedgerEventRecord`, `InclusionProofRecord`, `RoutingDecisionRecord` — value types carrying per-entry detail within each requirement.

### Service

`CodeReviewComplianceService` in `app/src/main/java/io/casehub/devtown/app/ledger/`:

- `@ApplicationScoped`
- Injects: `LedgerEntryRepository` (blocking), `LedgerVerificationService`, `EntityManager` (default PU — work entities only)
- `findEvidence(UUID caseId) → Optional<CodeReviewComplianceEvidence>`
- Queries all ledger entries via `LedgerEntryRepository.findBySubjectId(caseId)`, then filters by `instanceof` to separate `CaseLedgerEntry`, `WorkerDecisionEntry`, and `MergeDecisionLedgerEntry`
- Assembles each requirement section from the filtered entries

**Persistence unit discipline:** devtown has two persistence units — default (casehub-work) and qhorus (ledger + qhorus entities). `LedgerEntryRepository` (via `JpaLedgerEntryRepository`) injects `@LedgerPersistenceUnit EntityManager` — this resolves to the qhorus PU in devtown's configuration (`casehub.ledger.datasource=qhorus`). All ledger queries go through `LedgerEntryRepository`, never through a raw EntityManager.

The unqualified `EntityManager` is injected **only** for WorkItem SLA lookup (`em.find(WorkItem.class, taskId)`) — `WorkItem` is on the default PU, so this is correct. An alternative is to use `WorkItemService` from casehub-work-api, but `em.find()` is simpler for a read-only lookup by ID and avoids pulling in the full service dependency graph.

Note: `CaseLedgerEntryRepository` (engine-ledger) injects an unqualified `EntityManager caseEm` for its own `findByCaseId()` method — this resolves to the wrong PU in devtown's multi-PU setup. This is a pre-existing issue in `CaseLedgerEntryRepository`; the compliance service avoids it by using the parent `LedgerEntryRepository.findBySubjectId()` which uses the correctly qualified `@LedgerPersistenceUnit` EntityManager.

The pattern follows the same shape as casehub-aml's compliance evidence service (`AmlComplianceEvidenceService` in `casehub-aml/app`): query all ledger entries for a subject, filter by subclass type, assemble structured requirement sections, verify the Merkle chain, look up WorkItem SLA data via EntityManager. That class is a peer-repo reference — not a dependency.

### REST Endpoint

`CodeReviewComplianceResource` in `app/src/main/java/io/casehub/devtown/app/`:

- `@Path("/api/compliance/code-review")`
- `@ApplicationScoped`
- `GET /{caseId}` — returns `CodeReviewComplianceEvidence` as JSON (200) or 404 if no evidence

---

## Part 3: Testing

### Test profile

A `@QuarkusTestProfile` implementation (`LedgerEnabledTestProfile`) provides the overrides needed for ledger-write tests. This is required because global test `application.properties` sets `casehub.ledger.enabled=false` and uses `database.generation=drop-and-create` — two build-time-equivalent properties that cannot both be true in the same augmentation.

```java
public class LedgerEnabledTestProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
        return Map.of(
            "casehub.ledger.enabled", "true",
            "quarkus.flyway.qhorus.migrate-at-start", "true",
            "quarkus.hibernate-orm.qhorus.database.generation", "none",
            "quarkus.datasource.qhorus.jdbc.url",
                "jdbc:h2:mem:devtown-ledger-test;MODE=PostgreSQL;DB_CLOSE_ON_EXIT=FALSE"
        );
    }
}
```

Tests using this profile get Flyway-managed schema (including `ledger_entry`, `ledger_subject_sequence`, `case_ledger_entry`, `worker_decision_entry`, `merge_decision_ledger_entry`) instead of Hibernate `drop-and-create`.

### Ledger write test

`MergeDecisionObserverTest` — `@QuarkusTest` with `@TestProfile(LedgerEnabledTestProfile.class)`:

- Fire a `CaseLifecycleEvent` with `caseStatus = "COMPLETED"` for a case with seeded PR context
- Assert `MergeDecisionLedgerEntry` persisted with `decision = "APPROVED"`, correct PR fields
- Assert `ComplianceSupplement` attached with `algorithmRef`, `humanOverrideAvailable`, `contestationUri`
- Fire `CaseLifecycleEvent` with `caseStatus = "CANCELLED"` → assert `decision = "REJECTED"`
- Fire `CaseLifecycleEvent` with `caseStatus = "FAULTED"` → assert no `MergeDecisionLedgerEntry` written
- Sequence numbers: `LedgerSequenceAllocator` handles allocation via atomic `MERGE INTO` — the test verifies entries exist with monotonically increasing sequence numbers

### Compliance report test

`CodeReviewComplianceServiceTest` — `@QuarkusTest` with `@TestProfile(LedgerEnabledTestProfile.class)`:

- Seed ledger entries for a case (case lifecycle, worker decision, merge decision)
- Call `findEvidence(caseId)`
- Assert all four requirement sections populated with correct statuses
- Assert audit chain CLOSED when Merkle chain verifies and merge decision is linked
- Assert trust routing CLOSED when all dispatched capabilities have `WorkerDecisionEntry` records

### Existing tests unaffected

All existing tests continue with `casehub.ledger.enabled=false` and `database.generation=drop-and-create` — no changes needed.

---

## Module Placement Summary

| File | Module | Rationale |
|------|--------|-----------|
| `RequirementStatus`, `CodeReviewComplianceEvidence`, requirement records | `review/` | Domain types, pure Java |
| `MergeDecisionLedgerEntry` | `app/ledger/` | JPA entity extending `LedgerEntry` runtime class |
| `MergeDecisionObserver` | `app/ledger/` | CDI bean — observes `CaseLifecycleEvent`, writes ledger entry |
| `LedgerEnabledTestProfile` | `app/test/` | `@QuarkusTestProfile` for ledger-write tests |
| `CodeReviewComplianceService` | `app/ledger/` | CDI bean — `LedgerEntryRepository` (qhorus PU) + default `EntityManager` (work PU for WorkItem) |
| `CodeReviewComplianceResource` | `app/` | REST resource |
| `V2002__merge_decision_ledger_entry.sql` | `app/resources/db/devtown/migration/` | Flyway migration |

---

## Deferred Scope

| Item | Status | Issue |
|------|--------|-------|
| Case-opened LedgerEntry subclass | Not needed — `CaseLedgerEntry` handles case lifecycle | — |
| GDPR Art.17 erasure REST endpoint | Filed | devtown#74 |
| Post-merge FLAGGED attestation on incident | Already tracked | devtown#5 |
| Merkle verification REST endpoint | Covered by Epic 10 | devtown#17 |
| Distributed ledger — app join tables vs foundation DB | Architectural question filed | parent#207 |
| Content integrity in Merkle hash | Foundation enhancement filed | ledger#128 |
| Stuck-case detection (unsatisfiable goals) | Operational tooling | devtown#17 |

---

## Protocol Compliance

| Protocol | Status |
|----------|--------|
| `ledger-subclass-extension` | ✅ JOINED, V2002, consumer-owned, domain-agnostic leaf hash |
| `harness-ledger-writer` | ✅ N/A — single writer; protocol applies only when multiple services write the same entry type |
| `dual-trail-audit-pattern` | ✅ CDI event observed async, separate from operational trail |
| `flyway-version-range-allocation` | ✅ V2002 in `db/devtown/migration/` — V2000-V2001 taken by engine-ledger |
| `flyway-repo-scoped-migration-path` | ✅ `db/devtown/migration/` not `db/migration/devtown/` |
| `module-tier-structure` | ✅ Pure Java in review/, CDI in app/ |
| `harness-rest-resource-blocking-applicationscoped` | ✅ REST resource is `@ApplicationScoped` |

---

## Review Issue Resolution

| # | Severity | Finding | Resolution |
|---|----------|---------|------------|
| 1 | Critical | V2000 conflicts with engine-ledger | V2002 — engine-ledger owns V2000-V2001 |
| 2 | Critical | Event production side unspecified | `MergeDecisionObserver` bean defined — observes `CaseLifecycleEvent`, filters terminal states, looks up case context, derives decision, writes entry directly |
| 3 | Significant | Decision value not in Merkle hash | Documented as platform-wide characteristic. Neither join-table columns nor supplements are hash-protected. Foundation issue ledger#128 filed. Defence is DB-level access controls. |
| 4 | Significant | Sequence number ownership claim wrong | Removed. `LedgerSequenceAllocator` owns allocation via atomic `MERGE INTO`. Writer calls `save()`; infrastructure handles sequencing, hashing, and frontier updates. |
| 5 | Moderate | Phantom `AmlComplianceEvidenceService` reference | Clarified as a peer-repo pattern reference, not a dependency. Pattern described inline. |
| 6 | Moderate | C3→C4 dependency weaker than claimed | Documented: C3 is complete (devtown#52). Compliance report queries case-level entries, not `MessageLedgerEntry`. Dependency is satisfied; completeness is enhanced by Layer 3 but not blocked on it. |
| 7 | Design | `MergeDecisionEvent` mixes domain and infrastructure | Eliminated the intermediate event entirely. Observer derives ledger fields from `CaseLifecycleEvent` + case context — no domain event carries infrastructure concerns. |
| 8 | Design | Decision semantics undefined | Defined: APPROVED (COMPLETED), REJECTED (CANCELLED). FAULTED is not a merge decision. BLOCKED removed — not a terminal state. |
| 9 | Minor | GdprRequirement always-CLOSED | Documented as "static capability declaration, not runtime verification" |
| 10 | Minor | Test profile underspecified | `LedgerEnabledTestProfile` class defined with explicit `getConfigOverrides()` |

### Revision 3 — follow-up review (4 findings)

| # | Severity | Finding | Resolution |
|---|----------|---------|------------|
| R2-1 | Moderate | `signature` field on `CodeReviewComplianceEvidence` undefined | Removed — Merkle root is inside `AuditChainRequirement`; no signing infrastructure exists. Add when it does. |
| R2-2 | Low | Observer race with `CaseLedgerEventCapture` — unordered sequence numbers | `causedByEntryId` set to the latest `CaseLedgerEntry` matching the terminal status (best-effort — null if race). Compliance report uses both sequence order and `causedByEntryId` for audit chain. |
| R2-3 | Low | `CrossTenantCaseInstanceRepository` blocking tech debt | Documented as accepted tech debt, same as `ReviewOutcomeObserver`. Resolution path: `CaseLifecycleEvent` carries case context directly. |
| R2-4 | Moderate | Raw `EntityManager` resolves to wrong PU | All ledger queries via `LedgerEntryRepository` (correct PU via `@LedgerPersistenceUnit`). Unqualified `EntityManager` used only for `WorkItem` lookup (default PU, correct for work entities). Pre-existing bug in `CaseLedgerEntryRepository.findByCaseId()` documented. |

### Revision 4 — final review (2 findings)

| # | Severity | Finding | Resolution |
|---|----------|---------|------------|
| R3-1 | Bug | Observer's `findLatestByCaseId()` uses wrong PU via buggy `caseEm` — `causedByEntryId` always null | Replaced with `findLatestBySubjectId()` (inherited, `@LedgerPersistenceUnit`) + `instanceof CaseLedgerEntry` filter. |
| R3-2 | Minor | Observer injects `CaseLedgerEntryRepository` but only needs `LedgerEntryRepository` | Changed to `LedgerEntryRepository` — narrower dependency, avoids buggy `caseEm`-backed methods, consistent with compliance service. |
