# GDPR Art.17 Erasure Endpoint — Design Spec

**Issue:** casehubio/devtown#74
**Date:** 2026-06-14
**Status:** Revised (post-review)
**Foundation issues:** casehubio/ledger#140 (promote receipt to foundation), casehubio/platform#99 (cross-tenant memory erasure)
**Cross-repo issues filed:** aml#62, clinical#79, life#34

---

## Problem

Layer 4 wired `casehub-ledger` and declared GDPR erasure capability as "wired" in the compliance report. But there is no REST endpoint to trigger erasure, no tamper-evident record that erasure occurred, and no `CaseMemoryStore` cleanup. A compliance officer cannot prove erasure happened — they can only observe that an actor token no longer resolves.

## Design

Actor-scoped erasure endpoint with tamper-evident receipt and memory cleanup.

### API

```
POST /api/actors/{actorId}/erasure
Content-Type: application/json

{ "reason": "GDPR Art.17 request" }
```

POST not DELETE — erasure is an action with side effects, not resource deletion. Avoids HTTP clients/proxies stripping request bodies from DELETE.

Query parameters:
- `tenancyId` (default: `"default"`) — scopes memory erasure. Ledger identity erasure is global (ActorIdentity has no tenancy column). **Migration note:** `tenancyId` will be sourced from `CurrentPrincipal.tenancyId()` when auth lands (devtown#71), matching `MemoryAdminResource`'s pattern. The query param is a placeholder.
- **Cross-tenancy limitation:** GDPR erasure applies to a data subject across all tenancies, but `CaseMemoryStore.eraseEntity()` is per-tenant. The caller must invoke once per tenancy if the actor has memory records in multiple tenancies. Tracked: platform#99.

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
| `erasedActorToken` | `String` | `erased_actor_token` | The pseudonymous token captured via `ActorIdentityProvider.tokeniseForQuery()` BEFORE erasure destroys the mapping. Privacy-safe — stores the token, not the raw actor ID. |
| `reason` | `String` | `reason` | Free text — "GDPR Art.17 request" |
| `ledgerEntriesAffected` | `long` | `ledger_entries_affected` | Count from `LedgerErasureService` |
| `memoryRecordsErased` | `int` | `memory_records_erased` | Count from `CaseMemoryStore.eraseEntity()` |

`@DiscriminatorValue("ERASURE_RECEIPT")`. `domainContentBytes()` includes all four fields — follows the `MergeDecisionLedgerEntry` pattern.

The entry's own `actorId` is set to `devtown:gdpr-erasure` (system actor). The `erasedActorToken` field stores the pseudonymous token, not the raw actor ID — the tokenisation pipeline in `JpaLedgerEntryRepository.save()` only processes `LedgerEntry.actorId`, not subclass join table fields, so raw PII in a subclass field would persist permanently.

**subjectId:** `UUID.nameUUIDFromBytes(("erasure:" + token).getBytes())` — deterministic from the token. Repeated erasures of the same actor chain into the same Merkle tree. The token is already a pseudonym so no PII in the derivation.

**entryType:** `LedgerEntryType.EVENT` — an erasure is a system event, not a command or attestation.

**2. `GdprErasureService`** — `app/src/main/java/io/casehub/devtown/app/ledger/`

`@ApplicationScoped`. Orchestrates three-step erasure. Injects: `LedgerErasureService`, `CaseMemoryStore`, `ActorIdentityProvider`, `LedgerEntryRepository`.

**Operation order and transaction boundary:**

1. **Capture token** — `ActorIdentityProvider.tokeniseForQuery(rawActorId)` (read-only, before anything destructive). If tokenisation is disabled (`PassThroughActorIdentityProvider`), returns the raw ID.
2. **Memory erasure** (best-effort, outside JTA) — `CaseMemoryStore.eraseEntity(actorId, tenancyId)`. Catch `MemoryCapabilityException` → record 0. Executed first because it may be outside the JTA boundary; if the subsequent ledger TX fails and rolls back, memory erasure is harmless (no PII remains in memory).
3. **Ledger erasure + receipt persist** (atomic, same JTA transaction) — `LedgerErasureService.erase(rawActorId)` then `LedgerEntryRepository.save(receipt, tenancyId)`. Both use `@LedgerPersistenceUnit` EntityManager. If the receipt persist fails, the ledger erasure rolls back — no partial state.

**Transactional guarantee:** Ledger erasure and receipt persist are atomic (same JTA transaction). Memory erasure is best-effort, executed before the atomic pair. On memory failure, the receipt records `memoryRecordsErased=0` — the compliance officer knows memory erasure failed and can retry.

Edge cases:
- Actor not found in ledger: `ErasureResult.mappingFound=false`. Still proceeds with memory erasure and writes a receipt (the receipt records "no ledger data found" — useful for compliance audit).
- `CaseMemoryStore` throws `MemoryCapabilityException` (store exists but doesn't support `ERASE_ENTITY`): catch, record `memoryRecordsErased=0`.
- `NoOpCaseMemoryStore` active: returns 0 (confirmed in bytecode). No error.
- Repeated erasure of same actor: idempotent. Second call finds no mapping, writes another receipt with `ledgerEntriesAffected=0`.

**3. `ErasureReceipt`** — `review/src/main/java/io/casehub/devtown/review/compliance/`

Response DTO record: `actorId`, `erasedAt`, `ledgerEntriesAffected`, `memoryRecordsErased`, `receiptEntryId`, `reason`.

Lives in `devtown-review` (same tier as `PrReviewOutcome`, `CodeReviewComplianceEvidence`).

**4. `GdprErasureResource`** — `app/src/main/java/io/casehub/devtown/app/`

Thin REST dispatcher. `@ApplicationScoped`, `@Path("/api/actors/{actorId}/erasure")`, `@Produces(APPLICATION_JSON)`.

`POST` method accepts `@PathParam("actorId")`, `@QueryParam("tenancyId") @DefaultValue("default")`, and a request body with `reason`. Delegates to `GdprErasureService`.

**5. `GdprRequirement` record change**

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

`erasurePerformed` is true when at least one `ErasureReceiptLedgerEntry` exists whose `erasedActorToken` matches any `actorId` token in the case's ledger entries. This works because both the receipt and the entries store the same pseudonymous token — the match is token-to-token, not raw-to-raw. No identity mapping needed.

`erasureReceiptIds` lists the receipt entry IDs for traceability. Breaking change to the record — call-site migration is mechanical (add two args to the single constructor call in `CodeReviewComplianceService.buildGdpr()`).

**6. `CodeReviewComplianceService.buildGdpr()` enhancement**

Currently returns static values. After this change:
- Accept the `List<EntrySnapshot>` from the caller (already computed for the case)
- Collect unique `actorId` tokens from the snapshots
- Query for `ErasureReceiptLedgerEntry` records matching those tokens (JPQL: `SELECT e FROM ErasureReceiptLedgerEntry e WHERE e.erasedActorToken IN :tokens AND e.tenancyId = :tenancyId`)
- If found: `erasurePerformed=true`, `erasureReceiptIds` = list of receipt IDs

**7. Relationship to `MemoryAdminResource`**

`POST /api/admin/memory/erase/contributor` is an admin operational tool — memory-only, `@RolesAllowed("admin")`, uses `CurrentPrincipal.tenancyId()`. The new `POST /api/actors/{actorId}/erasure` is a GDPR compliance endpoint — ledger + memory, produces a tamper-evident receipt. They serve different purposes and coexist. Post-auth-retrofit (devtown#71), both will use `CurrentPrincipal.tenancyId()`.

### Migration

**V2004__erasure_receipt_ledger_entry.sql** — `db/devtown/migration/`

```sql
CREATE TABLE erasure_receipt_ledger_entry (
    id                       UUID NOT NULL,
    erased_actor_token       VARCHAR(255) NOT NULL,
    reason                   VARCHAR(1000),
    ledger_entries_affected  BIGINT NOT NULL,
    memory_records_erased    INTEGER NOT NULL,
    CONSTRAINT pk_erasure_receipt_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_erasure_receipt_ledger_entry_id
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);

CREATE INDEX idx_erasure_receipt_actor_token ON erasure_receipt_ledger_entry(erased_actor_token);
```

No `tenancy_id` column — inherited from `ledger_entry` via the FK join (V2003 established this pattern by removing the duplicate from `merge_decision_ledger_entry`).

### Testing

**Unit tests** (`GdprErasureServiceTest`):
- Happy path: actor exists, ledger entries found, memory records found → receipt with correct counts, `erasedActorToken` is the token not the raw ID
- Actor not found: `ErasureResult.mappingFound=false` → receipt with `ledgerEntriesAffected=0`
- Memory store returns 0 → receipt with `memoryRecordsErased=0`
- Memory store throws `MemoryCapabilityException` → caught, receipt with `memoryRecordsErased=0`
- Idempotent: second erasure of same actor → new receipt with zero counts
- Token capture happens before erasure: verify `tokeniseForQuery()` called before `erase()`

**`@QuarkusTest`** (`GdprErasureResourceTest`):
- `@TestProfile` with `casehub.ledger.identity.tokenisation.enabled=true` (per GE-20260531-46f8ab)
- Write ledger entries with known actor → `POST /api/actors/{actorId}/erasure` → verify 200, receipt counts, `resolve()` returns empty
- Verify `ErasureReceiptLedgerEntry` persisted in ledger with token (not raw ID)
- Verify `ErasureReceiptLedgerEntry.erasedActorToken` matches the token on the original entries

**Compliance report test:**
- Erase actor → `GET /api/compliance/code-review/{caseId}` → verify `GdprRequirement.erasurePerformed=true` and `erasureReceiptIds` contains the receipt ID

### Module placement

| Component | Module | Tier |
|-----------|--------|------|
| `ErasureReceiptLedgerEntry` | `devtown-app` | 3 (JPA entity, CDI) |
| `GdprErasureService` | `devtown-app` | 3 (CDI service) |
| `GdprErasureResource` | `devtown-app` | 3 (REST adapter) |
| `ErasureReceipt` | `devtown-review` | 2 (DTO) |
| `GdprRequirement` | `devtown-review` | 2 (existing — enhanced) |
| V2004 migration | `devtown-app` resources | 3 |

### Protocol checklist (ledger-subclass-extension)

- [x] Inheritance strategy: JOINED
- [x] No field shadowing of `LedgerEntry` fields
- [x] Join table migration in consumer repo (devtown), not `casehub-ledger`
- [x] Migration version V2004 — within devtown's allocated V2000+ range (V2003 taken by `merge_decision_repo_pr_index`)
- [x] No `tenancy_id` in join table — inherited from `ledger_entry` via FK join
- [x] `domainContentBytes()` includes subclass fields (follows `MergeDecisionLedgerEntry` pattern post-ledger#128)
- [x] `erasedActorToken` stores pseudonymous token, not raw PII

### Out of scope

- Per-case erasure (GDPR is actor-scoped, not case-scoped)
- `@RolesAllowed` enforcement (auth retrofit tracked in devtown#71 — endpoint is structurally ready)
- Foundation promotion of `ErasureReceiptLedgerEntry` (tracked in ledger#140)
- Cross-tenant memory erasure in a single call (tracked in platform#99)
