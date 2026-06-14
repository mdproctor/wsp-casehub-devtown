# Design Journal — issue-74-gdpr-test-oversight

## §1 GDPR Art.17 Erasure Endpoint (#74)

**Decision:** Actor-scoped erasure with tamper-evident receipt (Approach B), not thin delegation (Approach A).

**Why:** The act of erasure must itself be auditable. A compliance officer needs to prove erasure happened — not just observe that an actor token no longer resolves. `ErasureReceiptLedgerEntry` (JOINED, V2004) makes erasure a Merkle-chained event.

**Key design choices (4 review rounds, 20 findings):**

1. **`erasedActorToken` stores the pseudonymous token, never raw PII.** `tokeniseForQuery()` falls through to raw ID when no mapping exists (`.orElse(rawActorId)`). SHA-256 hash fallback prevents PII in the join table.

2. **No `@Transactional` on service.** Memory erasure is best-effort (outside JTA). Ledger erasure + receipt persist are atomic via `QuarkusTransaction.requiringNew()`. Order: memory first → ledger+receipt atomic.

3. **Compliance cross-reference via token-to-token matching.** The receipt's `erasedActorToken` matches `actorId` tokens on case entries — no identity resolution needed. Works because both sides store the same pseudonymous token. `@NamedQuery` has no `tenancyId` filter because erasure is global.

4. **Memory erasure uses prefixed entity IDs.** `DevtownMemoryDomain.CONTRIBUTOR_PREFIX + rawActorId` and `REVIEWER_PREFIX + rawActorId`. Code area entities (`module:`) omitted — they don't contain contributor identity.

**Foundation issues filed:** ledger#140 (promote receipt), ledger#142 (tokeniseForQuery Optional API), platform#99 (cross-tenant memory erasure).

**Cross-repo issues:** aml#62, clinical#79, life#34 — deep GDPR erasure for other harnesses.
