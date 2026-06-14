# GDPR Art.17 Erasure Endpoint — Design Spec

**Issue:** casehubio/devtown#74
**Date:** 2026-06-14
**Status:** Revised (post-review round 3)
**Foundation issues:** casehubio/ledger#140 (promote receipt to foundation), casehubio/ledger#142 (tokeniseForQuery Optional API), casehubio/platform#99 (cross-tenant memory erasure)
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
| `erasedActorToken` | `String` | `erased_actor_token` | Privacy-safe identifier for the erased actor. See token resolution below. |
| `reason` | `String` | `reason` | Free text — "GDPR Art.17 request" |
| `ledgerEntriesAffected` | `long` | `ledger_entries_affected` | Count from `LedgerErasureService` |
| `memoryRecordsErased` | `int` | `memory_records_erased` | Count from `CaseMemoryStore.eraseEntity()` |

`@DiscriminatorValue("ERASURE_RECEIPT")`. `domainContentBytes()` includes all four fields — follows the `MergeDecisionLedgerEntry` pattern.

**Token resolution for `erasedActorToken`:**

`ActorIdentityProvider.tokeniseForQuery(rawActorId)` is called before erasure. However, `InternalActorIdentityProvider.tokeniseForQuery()` falls through to the raw actor ID when no `ActorIdentity` mapping exists (`.orElse(rawActorId)` at line 37). This happens when the actor was never a HUMAN participant in a ledger entry (only HUMAN actors are tokenised — `tokenise()` skips non-HUMAN types) or when `PassThroughActorIdentityProvider` is active.

To prevent raw PII in the join table:

```java
String queryResult = actorIdentityProvider.tokeniseForQuery(rawActorId);
String erasedActorToken = queryResult.equals(rawActorId)
    ? sha256Hex("erasure:" + rawActorId)  // no mapping exists → hash fallback
    : queryResult;                         // real token → use it
```

The hash fallback is:
- Privacy-safe (SHA-256 is one-way)
- Deterministic (same actor → same hash → same subjectId for Merkle chaining)
- Verifiable (a DPO who knows the raw ID can hash it to confirm the receipt matches)

When a real token exists, the compliance cross-reference (token-to-token matching against case entries) works. When the hash fallback fires, the cross-reference won't match — correct, because the actor has no ledger entries to match against.

**Base entry fields:**

| Field | Value | Rationale |
|-------|-------|-----------|
| `actorId` | `"devtown:gdpr-erasure"` | System actor, not the erased actor |
| `actorType` | `ActorType.SYSTEM` | Prevents `tokenise()` from creating an `ActorIdentity` mapping for the system actor (HUMAN path fires when `actorType` is null) |
| `actorRole` | `"GDPR_COMPLIANCE"` | Follows `MergeDecisionObserver` pattern (`"ORCHESTRATOR"`) |
| `entryType` | `LedgerEntryType.EVENT` | Erasure is a system event, not a command or attestation |
| `subjectId` | `UUID.nameUUIDFromBytes(("erasure:" + erasedActorToken).getBytes())` | Deterministic within each token resolution path. The first erasure (real token) and subsequent erasures (hash fallback) produce different `subjectId` values because the token mapping is destroyed by the first erasure. |

**@NamedQuery:**

```java
@NamedQuery(
    name = "ErasureReceiptLedgerEntry.findByTokens",
    query = "SELECT e FROM ErasureReceiptLedgerEntry e WHERE e.erasedActorToken IN :tokens"
)
```

No `tenancyId` filter — ledger identity erasure is global (`ActorIdentity` has no tenancy column). The receipt's existence proves erasure regardless of which tenancy the request originated from.

**2. `GdprErasureService`** — `app/src/main/java/io/casehub/devtown/app/ledger/`

`@ApplicationScoped`. **No `@Transactional` annotation on the class or the method.** Orchestrates three-step erasure. Injects: `LedgerErasureService`, `CaseMemoryStore`, `ActorIdentityProvider`, `LedgerEntryRepository`.

**Operation order and transaction boundary:**

1. **Capture token** (no TX) — `ActorIdentityProvider.tokeniseForQuery(rawActorId)`. If result equals input, apply SHA-256 hash fallback. Read-only, auto-commit.
2. **Memory erasure** (no TX, best-effort) — Erase all devtown entity types for this actor. devtown stores memory under prefixed entity IDs (see `docs/specs/2026-06-05-case-memory-store-integration-design.md`):

   ```java
   int contributorCount = eraseEntitySafely(
       DevtownMemoryDomain.CONTRIBUTOR_PREFIX + rawActorId, tenancyId);
   int reviewerCount = eraseEntitySafely(
       DevtownMemoryDomain.REVIEWER_PREFIX + rawActorId, tenancyId);
   int memoryRecordsErased = contributorCount + reviewerCount;
   ```

   `eraseEntitySafely()` catches `MemoryCapabilityException` and returns 0. Code area entities (`module:`) don't contain contributor identity by design — no erasure needed. Executed before the atomic pair because it may be outside the JTA boundary; if the subsequent ledger TX fails and rolls back, memory erasure is harmless (no PII remains in memory after erasure).
3. **Ledger erasure + receipt persist** (atomic) — wrapped in `QuarkusTransaction.requiringNew().call(() -> { ... })` per the `CodeReviewComplianceService` pattern. `LedgerErasureService.erase()` and `LedgerEntryRepository.save()` both use `@LedgerPersistenceUnit` EntityManager and join this transaction via their own `@Transactional(REQUIRED)`. If the receipt persist fails, the ledger erasure rolls back — no partial state.

**Transactional guarantee:** Ledger erasure and receipt persist are atomic (same JTA transaction via `QuarkusTransaction.requiringNew()`). Memory erasure is best-effort, executed before the atomic pair. On memory failure, the receipt records `memoryRecordsErased=0` — the compliance officer knows memory erasure needs retry.

Edge cases:
- Actor not found in ledger: `ErasureResult.mappingFound=false`. Still proceeds with memory erasure and writes a receipt (the receipt records "no ledger data found" — useful for compliance audit).
- `CaseMemoryStore` throws `MemoryCapabilityException` (store exists but doesn't support `ERASE_ENTITY`): caught by `eraseEntitySafely()`, returns 0 per prefix.
- `NoOpCaseMemoryStore` active: `eraseEntity()` returns 0 (confirmed in bytecode). No error.
- Repeated erasure of same actor: idempotent. Second call finds no mapping, writes another receipt with `ledgerEntriesAffected=0`.

**`DevtownMemoryDomain` entity ID prefix constants** — `domain/src/main/java/io/casehub/devtown/domain/memory/`

Add to the existing `DevtownMemoryDomain` class:

```java
public static final String CONTRIBUTOR_PREFIX = "contributor:";
public static final String REVIEWER_PREFIX = "reviewer:";
public static final String MODULE_PREFIX = "module:";
```

Update existing inline usages in `CaseMemoryEmitter`, `CaseMemoryRecaller`, and `MemoryAdminResource` to use these constants. Breaking change — mechanical, grep-and-replace.

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

`erasurePerformed` is true when at least one `ErasureReceiptLedgerEntry` exists whose `erasedActorToken` matches any `actorId` token in the case's ledger entries. This works because both the receipt and the entries store the same pseudonymous token — the match is token-to-token. When the hash fallback was used (no mapping existed), the match won't fire — correct, because the actor has no ledger entries in the case.

`erasureReceiptIds` lists the receipt entry IDs for traceability. Breaking change to the record — call-site migration is mechanical (add two args to the single constructor call in `CodeReviewComplianceService.buildGdpr()`).

**6. `CodeReviewComplianceService.buildGdpr()` enhancement**

Currently returns static values. After this change:
- Accept the `List<EntrySnapshot>` from the caller (already computed for the case)
- Collect unique `actorId` tokens from the snapshots
- Query via named query `ErasureReceiptLedgerEntry.findByTokens` — no `tenancyId` filter (erasure is global; the receipt's existence proves erasure regardless of originating tenancy)
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
- Happy path: actor exists, ledger entries found, memory records found → receipt with correct counts, `erasedActorToken` is the token not the raw ID, memory erased for both `contributor:` and `reviewer:` prefixes
- Hash fallback: actor with no mapping → `erasedActorToken` is SHA-256 hash, not raw ID
- Actor not found: `ErasureResult.mappingFound=false` → receipt with `ledgerEntriesAffected=0`
- Memory store returns 0 → receipt with `memoryRecordsErased=0`
- Memory store throws `MemoryCapabilityException` → caught, receipt with `memoryRecordsErased=0`
- Idempotent: second erasure of same actor → new receipt with zero counts
- Token capture happens before erasure: verify `tokeniseForQuery()` called before `erase()`
- Verify `actorType=SYSTEM` on the receipt entry (prevents tokenisation of system actor)

**`@QuarkusTest`** (`GdprErasureResourceTest`):
- `@TestProfile` with `casehub.ledger.identity.tokenisation.enabled=true` (per GE-20260531-46f8ab)
- Write ledger entries with known actor → `POST /api/actors/{actorId}/erasure` → verify 200, receipt counts, `resolve()` returns empty
- Verify `ErasureReceiptLedgerEntry` persisted in ledger with token (not raw ID)
- Verify `ErasureReceiptLedgerEntry.erasedActorToken` matches the token on the original entries
- Verify receipt entry has `actorType=SYSTEM`, `actorRole="GDPR_COMPLIANCE"`

**Compliance report test:**
- Erase actor → `GET /api/compliance/code-review/{caseId}` → verify `GdprRequirement.erasurePerformed=true` and `erasureReceiptIds` contains the receipt ID
- Verify compliance query works cross-tenancy: erase in tenancy A, report in tenancy B → receipt still found (no tenancyId filter)

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
- [x] `erasedActorToken` stores pseudonymous token or SHA-256 hash — never raw PII
- [x] `actorType = ActorType.SYSTEM` — prevents tokenisation of system actor
- [x] `@NamedQuery` for compliance lookup (follows `MergeDecisionLedgerEntry` pattern)

### Out of scope

- Per-case erasure (GDPR is actor-scoped, not case-scoped)
- `@RolesAllowed` enforcement (auth retrofit tracked in devtown#71 — endpoint is structurally ready)
- Foundation promotion of `ErasureReceiptLedgerEntry` (tracked in ledger#140)
- Cross-tenant memory erasure in a single call (tracked in platform#99)
- `ActorIdentityProvider.tokeniseForQuery()` → `Optional<String>` return type (tracked in ledger#142 — eliminates the need for hash fallback detection)

### Implementation notes

- **Fix `MemoryAdminResource` exception catch** — pre-existing bug: catches `UnsupportedOperationException` but `CaseMemoryStore.eraseEntity()` default throws `MemoryCapabilityException extends RuntimeException` (sibling, not subclass). Dormant because all active stores override `eraseEntity()`. Fix during implementation: change catch to `MemoryCapabilityException`.
