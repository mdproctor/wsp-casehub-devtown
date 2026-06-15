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

## §2 CaseMemoryIntegrationTest Fix (#72)

**Three root causes** — none was the originally suspected async emitter issue:

1. **`Instance.destroy()` on `@ApplicationScoped` singleton.** Quarkus ArC's `Instance.destroy()` removes the contextual instance from the application context — destroying the `InMemoryMemoryStore` singleton and its `ConcurrentHashMap`. Every subsequent `Instance.get()` creates a fresh, empty instance. Both `CaseMemoryEmitter` and `CaseMemoryRecaller` had this bug. CDI spec says `destroy()` on normal-scoped beans is a no-op — ArC diverges.

2. **`CaseMemoryRecaller.withQuestion()` substring mismatch.** `InMemoryMemoryStore.query()` filters: `m.text().contains(query.question())`. The question `"review history for app in casehubio/devtown"` is never a substring of the stored fact `"The app module in casehubio/devtown received a style review..."`. Entity-ID-scoped queries don't need semantic search — removed `withQuestion()`.

3. **Engine signal race (engine#494).** `CaseHubRuntime.signal()` dispatches to the Vert.x event loop; `startCase()` returns the caseId before the case is cached. Worked around by pre-seeding the outcome in the initial context.

**Garden entry candidate:** `Instance.destroy()` on `@ApplicationScoped` beans in Quarkus ArC — silent data loss.
