# Layer 4 — Tamper-Evident Merge Decision Audit Trail + Compliance Report

**Issues:** devtown#73 (Layer 4), devtown#7 (compliance report)
**Branch:** `issue-73-layer4-ledger-audit`
**Date:** 2026-06-08

---

## Summary

Layer 4 makes devtown's existing `casehub-engine-ledger` wiring real: a domain-specific `MergeDecisionLedgerEntry` captures structured PR merge decisions in the Merkle-chained audit trail, and a compliance report endpoint (#7) assembles per-case evidence across four regulatory requirement dimensions. The engine-level `CaseLedgerEntry` and `WorkerDecisionEntry` remain unchanged — they handle case lifecycle and worker dispatch events. Layer 4 adds the one domain event those generic entries cannot capture: the merge decision itself.

## Prior State

`casehub-engine-ledger` is already a compile dependency in `app/pom.xml` with:
- Jandex indexing configured (prod and test)
- Flyway migration locations: `db/engine-ledger/migration` (V2000 `case_ledger_entry`, V2001 `worker_decision_entry`)
- Trust routing YAML configs loaded (`trust-routing.yaml`, `trust-gate.yaml`)
- `casehub.ledger.enabled=false` in test properties (disables `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` in tests)
- `TrustWeightedAgentStrategy`, `WorkerDecisionEventCapture`, `CaseLedgerEventCapture` all discoverable via CDI

What's missing: no domain-specific ledger entry for merge decisions, no `ComplianceSupplement` attachment, no compliance report endpoint, no tests exercising ledger writes.

---

## Part 1: MergeDecisionLedgerEntry (#73)

### Entity

`MergeDecisionLedgerEntry extends LedgerEntry` in `app/src/main/java/io/casehub/devtown/app/ledger/`.

| Column | Type | Nullable | Purpose |
|--------|------|----------|---------|
| `pr_number` | INTEGER | NOT NULL | PR number in the repository |
| `repository` | VARCHAR(255) | NOT NULL | `owner/repo` identifier |
| `commit_sha` | VARCHAR(40) | nullable | HEAD commit SHA at decision time |
| `decision` | VARCHAR(20) | NOT NULL | APPROVED, REJECTED, or BLOCKED |
| `case_id` | UUID | NOT NULL | Convenience alias for `subjectId` (same value) |
| `tenancy_id` | VARCHAR(64) | NOT NULL | Tenant isolation — index-only scans without base table join |

JPA annotations:
- `@Entity`, `@Table(name = "merge_decision_ledger_entry")`, `@DiscriminatorValue("MERGE_DECISION")`
- Indexes on `case_id` and `tenancy_id`

### CDI Event

`MergeDecisionEvent` record in `review/` (pure Java, no framework deps):

```java
public record MergeDecisionEvent(
    UUID caseId,
    String tenancyId,
    int prNumber,
    String repository,
    String commitSha,
    String decision,
    String actorId,
    ActorType actorType,
    String actorRole
) {}
```

Fired via `Event<MergeDecisionEvent>.fireAsync()` from the existing service code (`ReviewOutcomeObserver` or `PrReviewCaseService`) when a PR review case reaches its terminal outcome.

### Writer Bean

`MergeDecisionLedgerWriter` in `app/src/main/java/io/casehub/devtown/app/ledger/`:

- `@ApplicationScoped`
- Observes `MergeDecisionEvent` via `@ObservesAsync`
- `@Transactional`
- Guards on `ledgerConfig.enabled()`
- Owns `sequenceNumber` computation via `CaseLedgerEntryRepository.findLatestBySubjectId()`
- Attaches `ComplianceSupplement` with EU AI Act Art.12 fields:
  - `algorithmRef = "casehub-devtown:pr-review-v1"`
  - `humanOverrideAvailable = true`
  - `contestationUri = "/api/reviews/{prNumber}/contest"`
  - `confidenceScore` — null unless trust score is available from routing

### Flyway Migration

`V2000__merge_decision_ledger_entry.sql` in `app/src/main/resources/db/devtown/migration/`:

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
    GdprRequirement gdpr,
    String signature
) {}
```

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
- Status: CLOSED (all dispatched capabilities have routing attestations), PARTIAL (some missing), GAP (no dispatches)

**`GdprRequirement`** — GDPR capability declaration:
- Static check: `LedgerErasureService` on classpath, pseudonymisation active
- Always CLOSED if ledger runtime is wired (it is)

Supporting records: `LedgerEventRecord`, `InclusionProofRecord`, `RoutingDecisionRecord` — value types carrying per-entry detail within each requirement.

### Service

`CodeReviewComplianceService` in `app/src/main/java/io/casehub/devtown/app/ledger/`:

- `@ApplicationScoped`
- Injects: `LedgerEntryRepository`, `LedgerVerificationService`, `CaseLedgerEntryRepository`, `EntityManager`
- `findEvidence(UUID caseId) → Optional<CodeReviewComplianceEvidence>`
- Queries ledger by `subjectId = caseId`, filters by entry type, assembles each requirement section
- Follows `AmlComplianceEvidenceService` pattern

### REST Endpoint

`CodeReviewComplianceResource` in `app/src/main/java/io/casehub/devtown/app/`:

- `@Path("/api/compliance/code-review")`
- `GET /{caseId}` — returns `CodeReviewComplianceEvidence` as JSON (200) or 404 if no evidence

---

## Part 3: Testing

### Ledger write test

`MergeDecisionLedgerWriterTest` — `@QuarkusTest` with test profile overrides:

```properties
casehub.ledger.enabled=true
quarkus.flyway.qhorus.migrate-at-start=true
quarkus.hibernate-orm.qhorus.database.generation=none
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:devtown-ledger-test;MODE=PostgreSQL;DB_CLOSE_ON_EXIT=FALSE
```

Test cases:
- Fire `MergeDecisionEvent` → assert `MergeDecisionLedgerEntry` persisted with correct fields
- Assert `ComplianceSupplement` attached with `algorithmRef`, `humanOverrideAvailable`, `contestationUri`
- Fire two events for same case → assert sequence numbers increment (1, 2)

### Compliance report test

`CodeReviewComplianceServiceTest` — `@QuarkusTest` with same ledger-enabled profile:

- Seed ledger entries for a case (case lifecycle, worker decision, merge decision)
- Call `findEvidence(caseId)`
- Assert all four requirement sections populated with correct statuses

### Existing tests unaffected

All existing tests continue with `casehub.ledger.enabled=false` — no changes needed.

---

## Module Placement Summary

| File | Module | Rationale |
|------|--------|-----------|
| `MergeDecisionEvent` | `review/` | Domain event record, pure Java |
| `RequirementStatus`, `CodeReviewComplianceEvidence`, requirement records | `review/` | Domain types, pure Java |
| `MergeDecisionLedgerEntry` | `app/ledger/` | JPA entity extending `LedgerEntry` runtime class |
| `MergeDecisionLedgerWriter` | `app/ledger/` | CDI bean, ledger runtime dependency |
| `CodeReviewComplianceService` | `app/ledger/` | CDI bean, ledger + EntityManager |
| `CodeReviewComplianceResource` | `app/` | REST resource |
| `V2000__merge_decision_ledger_entry.sql` | `app/resources/db/devtown/migration/` | Flyway migration |

---

## Deferred Scope

| Item | Status | Issue |
|------|--------|-------|
| Case-opened LedgerEntry subclass | Not needed — `CaseLedgerEntry` handles case lifecycle | — |
| GDPR Art.17 erasure REST endpoint | Filed | devtown#74 |
| Post-merge FLAGGED attestation on incident | Already tracked | devtown#5 |
| Merkle verification REST endpoint | Covered by Epic 10 | devtown#17 |
| Distributed ledger — app join tables vs foundation DB | Architectural question filed | parent#207 |

---

## Protocol Compliance

| Protocol | Status |
|----------|--------|
| `ledger-subclass-extension` | ✅ JOINED, V2000, consumer-owned, domain-agnostic leaf hash |
| `harness-ledger-writer` | ✅ Dedicated writer bean, single sequenceNumber owner |
| `dual-trail-audit-pattern` | ✅ CDI event observed async, separate from operational trail |
| `flyway-version-range-allocation` | ✅ V2000 in `db/devtown/migration/` |
| `flyway-repo-scoped-migration-path` | ✅ `db/devtown/migration/` not `db/migration/devtown/` |
| `module-tier-structure` | ✅ Pure Java in review/, CDI in app/ |
| `harness-rest-resource-blocking-applicationscoped` | ✅ REST resource is `@ApplicationScoped` |
