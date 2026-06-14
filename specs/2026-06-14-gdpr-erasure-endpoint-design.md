# GDPR Art.17 Erasure Endpoint — Design Spec

**Issue:** casehubio/devtown#74
**Date:** 2026-06-14
**Status:** Draft
**Foundation issue:** casehubio/ledger#140 (consolidation — promote receipt to foundation when second consumer surfaces)
**Cross-repo issues filed:** aml#62, clinical#79, life#34

---

## Problem

Layer 4 wired `casehub-ledger` and declared GDPR erasure capability as "wired" in the compliance report. But there is no REST endpoint to trigger erasure, no tamper-evident record that erasure occurred, and no `CaseMemoryStore` cleanup. A compliance officer cannot prove erasure happened — they can only observe that an actor token no longer resolves.

## Design

Actor-scoped erasure endpoint with tamper-evident receipt and memory cleanup.

### API

```
DELETE /api/actors/{actorId}/erasure
Content-Type: application/json

{ "reason": "GDPR Art.17 request" }
```

Returns:

```json
{
  "actorId": "john.doe@example.com",
  "erasedAt": "2026-06-14T10:30:00Z",
  "ledgerEntriesAffected": 12,
  "memoryRecordsErased": 3,
  "receiptEntryId": "550e8400-e29b-41d4-a716-446655440000",
  "reason": "GDPR Art.17 request"
}
```

HTTP status codes:
- `200` — erasure performed, receipt returned
- `200` with `ledgerEntriesAffected=0, memoryRecordsErased=0` — actor not found or no data to erase (idempotent — no 404; GDPR does not penalise erasing nothing)

### Components

**1. `ErasureReceiptLedgerEntry`** — `app/src/main/java/io/casehub/devtown/app/ledger/`

JOINED subclass of `LedgerEntry`. Records the erasure event in the Merkle chain.

| Field | Type | Column | Notes |
|-------|------|--------|-------|
| `erasedActorId` | `String` | `erased_actor_id` | Raw actor ID before tokenisation destroys the mapping |
| `reason` | `String` | `reason` | Free text — "GDPR Art.17 request" |
| `ledgerEntriesAffected` | `long` | `ledger_entries_affected` | Count from `LedgerErasureService` |
| `memoryRecordsErased` | `int` | `memory_records_erased` | Count from `CaseMemoryStore.eraseEntity()` |

`@DiscriminatorValue("ERASURE_RECEIPT")`. `domainContentBytes()` includes all four fields — follows the `MergeDecisionLedgerEntry` pattern.

The entry's own `actorId` is set to `devtown:gdpr-erasure` (system actor) — the erased actor's identity is in `erasedActorId`, which itself becomes an opaque token if that actor's mapping is later destroyed (consistent — the receipt records what was erased, but the receipt itself is also subject to the privacy layer).

**2. `GdprErasureService`** — `app/src/main/java/io/casehub/devtown/app/ledger/`

`@ApplicationScoped`. Orchestrates the three-step erasure:

1. `LedgerErasureService.erase(actorId)` → `ErasureResult` (pseudonymises ledger identity)
2. `CaseMemoryStore.eraseEntity(actorId, tenancyId)` → `int` count (cleans contributor memory)
3. Persist `ErasureReceiptLedgerEntry` with counts and reason
4. Return `ErasureReceipt` record

Transaction boundary: the entire operation runs in one `@Transactional` boundary. If memory erasure fails, the whole thing rolls back — no partial erasure.

Edge cases:
- Actor not found in ledger: `ErasureResult.mappingFound=false`. Still proceeds with memory erasure and writes a receipt (the receipt records "no ledger data found" — useful for compliance audit).
- `CaseMemoryStore` is `NoOpCaseMemoryStore`: returns 0. No error — the store capability check is not needed here; 0 is the correct answer when no memory backend is configured.
- Repeated erasure of the same actor: idempotent. Second call finds no mapping, writes another receipt with `ledgerEntriesAffected=0`. Compliance officer can see both receipts.

**3. `ErasureReceipt`** — `review/src/main/java/io/casehub/devtown/review/compliance/`

Response DTO record: `actorId`, `erasedAt`, `ledgerEntriesAffected`, `memoryRecordsErased`, `receiptEntryId`, `reason`.

Lives in `devtown-review` (same tier as `PrReviewOutcome`, `CodeReviewComplianceEvidence`).

**4. `GdprErasureResource`** — `app/src/main/java/io/casehub/devtown/app/`

Thin REST dispatcher. `@ApplicationScoped`, `@Path("/api/actors/{actorId}/erasure")`, `@Produces(APPLICATION_JSON)`.

`DELETE` method accepts `@PathParam("actorId")`, `@QueryParam("tenancyId") @DefaultValue("default")`, and a request body with `reason`. Delegates to `GdprErasureService`.

**5. Compliance report enhancement** — `CodeReviewComplianceService.buildGdpr()`

Currently returns static `erasureCapabilityWired=true`. After this change:
- Query for `ErasureReceiptLedgerEntry` records where `erasedActorId` matches any actor in the case's ledger entries
- If found: `GdprRequirement` gains `erasurePerformed=true`, `erasureReceiptIds` list
- If not: current behaviour (capability wired, no erasure performed)

### Migration

**V2003__erasure_receipt_ledger_entry.sql** — `db/devtown/migration/`

```sql
CREATE TABLE erasure_receipt_ledger_entry (
    id                       UUID NOT NULL,
    erased_actor_id          VARCHAR(255) NOT NULL,
    reason                   VARCHAR(1000),
    ledger_entries_affected  BIGINT NOT NULL,
    memory_records_erased    INTEGER NOT NULL,
    tenancy_id               VARCHAR(64) NOT NULL,
    CONSTRAINT pk_erasure_receipt_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_erasure_receipt_ledger_entry_id
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);

CREATE INDEX idx_erasure_receipt_erased_actor ON erasure_receipt_ledger_entry(erased_actor_id);
CREATE INDEX idx_erasure_receipt_tenancy_id ON erasure_receipt_ledger_entry(tenancy_id);
```

### Testing

**Unit tests** (`GdprErasureServiceTest`):
- Happy path: actor exists, ledger entries found, memory records found → receipt with correct counts
- Actor not found: `ErasureResult.mappingFound=false` → receipt with `ledgerEntriesAffected=0`
- Memory store returns 0 → receipt with `memoryRecordsErased=0`
- Idempotent: second erasure of same actor → new receipt with zero counts

**`@QuarkusTest`** (`GdprErasureResourceTest`):
- `@TestProfile` with `casehub.ledger.identity.tokenisation.enabled=true` (per GE-20260531-46f8ab)
- Write ledger entries with known actor → `DELETE /api/actors/{actorId}/erasure` → verify 200, receipt counts, `resolve()` returns empty
- Verify `ErasureReceiptLedgerEntry` persisted in ledger

**Compliance report test:**
- Erase actor → `GET /api/compliance/code-review/{caseId}` → verify `GdprRequirement` reflects erasure receipt

### Module placement

| Component | Module | Tier |
|-----------|--------|------|
| `ErasureReceiptLedgerEntry` | `devtown-app` | 3 (JPA entity, CDI) |
| `GdprErasureService` | `devtown-app` | 3 (CDI service) |
| `GdprErasureResource` | `devtown-app` | 3 (REST adapter) |
| `ErasureReceipt` | `devtown-review` | 2 (DTO) |
| `GdprRequirement` | `devtown-review` | 2 (existing — enhanced) |
| V2003 migration | `devtown-app` resources | 3 |

### Protocol checklist (ledger-subclass-extension)

- [x] Inheritance strategy: JOINED
- [x] No field shadowing of `LedgerEntry` fields
- [x] Join table migration in consumer repo (devtown), not `casehub-ledger`
- [x] Migration version V2003 — within devtown's allocated V2000+ range
- [x] `domainContentBytes()` includes subclass fields (follows `MergeDecisionLedgerEntry` pattern post-ledger#128)

### `GdprRequirement` record change

Current:
```java
public record GdprRequirement(
    String requirementId, String citation, String mechanism,
    RequirementStatus status,
    boolean erasureCapabilityWired, boolean pseudonymisationActive
)
```

After:
```java
public record GdprRequirement(
    String requirementId, String citation, String mechanism,
    RequirementStatus status,
    boolean erasureCapabilityWired, boolean pseudonymisationActive,
    boolean erasurePerformed, List<UUID> erasureReceiptIds
)
```

`erasurePerformed` is true when at least one `ErasureReceiptLedgerEntry` exists for any actor in the case's entries. `erasureReceiptIds` lists the receipt entry IDs for traceability. Breaking change to the record — call-site migration is mechanical (add two args to the single constructor call in `CodeReviewComplianceService.buildGdpr()`).

### Out of scope

- Per-case erasure (GDPR is actor-scoped, not case-scoped)
- `@RolesAllowed` enforcement (auth retrofit tracked in devtown#71 — endpoint is structurally ready)
- Foundation promotion of `ErasureReceiptLedgerEntry` (tracked in ledger#140)
