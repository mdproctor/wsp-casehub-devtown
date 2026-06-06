# XS/S Batch Design — devtown#62, #65, #66, #67, #68

**Date:** 2026-06-06
**Branch:** issue-62-xs-s-batch
**Covers:** devtown#62, #65, #66, #67, #68

---

## Issue #62 — Wire bootstrapEscalationRequired through DevtownTrustRoutingPolicyProvider

**Problem:** `TrustRoutingPolicy` (engine#415) added `bootstrapEscalationRequired` boolean.
`DevtownTrustRoutingPolicyProvider.forCapability()` hardcodes `false` on line 73, ignoring
the domain's `RoutingPolicy.fallbackType` declaration.

**Fix:** Map `routingPolicy.fallbackType().isPresent()` → `bootstrapEscalationRequired`.
Capabilities with a declared fallbackType (merge-executor, security-review, architecture-review)
get escalation. Capabilities without (style-review) don't.

**Changes:**
- `DevtownTrustRoutingPolicyProvider.forCapability()` — replace `false` with
  `routingPolicy.fallbackType().isPresent()`
- `DevtownTrustRoutingPolicyProviderTest` — add tests:
  - merge-executor has `bootstrapEscalationRequired = true`
  - style-review has `bootstrapEscalationRequired = false`
  - unknown capability returns DEFAULT (bootstrapEscalationRequired = false)

**No domain model changes.** `RoutingPolicy.fallbackType` already carries the information.

---

## Issue #68 — Code-area recall test + empty changedPaths edge case

**Gap 1:** `CaseMemoryRecallerTest` does not exercise the code-area recall path (multi-entity
query, RELEVANCE ordering, module normalization).

**Fix:** Add test that pre-populates a code-area fact with `module:repo1/app` entityId,
calls `recaller.recall()` with a PR whose `changedPaths` normalizes to `app`, asserts
`codeAreaHistory` is non-empty with correct entityId.

**Gap 2:** `CaseMemoryEmitterTest` has no test for empty `changedPaths`. When empty,
`buildFacts()` produces exactly 2 facts (contributor + reviewer).

**Fix:** Add test firing event with `changedPaths = List.of()`, assert 2 facts, assert
no code-area entity type present.

**No production code changes.**

---

## Issue #65 — Configurable recall limits via PreferenceKey

**Problem:** `CaseMemoryRecaller` hardcodes contributor limit (10), code area limit (15),
and time window (90 days).

**New class:** `MemoryRecallKeys` in `domain/.../memory/` — three `PreferenceKey<IntPreference>`
constants at scope `casehubio.devtown.memory-recall`:
- `CONTRIBUTOR_LIMIT` — default 10
- `CODE_AREA_LIMIT` — default 15
- `TIME_WINDOW_DAYS` — default 90

**Modified class:** `CaseMemoryRecaller` — inject `PreferenceProvider`, resolve at scope
`casehubio/devtown/memory-recall`, use keys with null-safe fallback to defaults
(per GE-20260530-9cdfb5: `MapPreferences.get()` returns null when key absent).

**Tests:**
- `MemoryRecallKeysTest` (domain, unit) — parse round-trip for each key
- `CaseMemoryRecallerTest` — add test with populated `PreferenceProvider` overriding
  contributor limit to 1, verify only 1 result returned

---

## Issue #66 — GDPR erasure endpoint for contributor memory

**Approach:** Admin REST endpoint, not scheduled job. GDPR Art.17 is erasure-on-request.

**New class:** `MemoryAdminResource` in `app/`
- `@Path("/api/memory")`
- `DELETE /contributor/{login}` — calls `store.eraseEntity("contributor:" + login, principal.tenancyId())`
- Returns 204 No Content
- Tenant-scoped via `CurrentPrincipal.tenancyId()`

**No application service layer** — straight pass-through to platform SPI.

Code area facts do not contain contributor PII by design (verified by existing
`code_area_text_does_not_contain_contributor_login` test in `CaseMemoryEmitterTest`).

**Tests:** `MemoryAdminResourceTest` (`@QuarkusTest`)
- Store contributor fact, call DELETE, verify query returns empty
- DELETE for non-existent contributor returns 204 (idempotent)

---

## Issue #67 — Integration test async tenancy fix

**Analysis:** `CaseMemoryIntegrationTest` is `@Disabled` with comment claiming
`CurrentPrincipal` is unavailable in `@ObservesAsync` threads. This is incorrect —
`FixedCurrentPrincipal` is `@ApplicationScoped` (singleton), not request-scoped.
It is accessible from any thread including `@ObservesAsync` executor threads.

The async emission chain:
1. `ReviewOutcomeObserver` reads `caseInstance.tenancyId` (set at case start from FixedCurrentPrincipal)
2. Fires `ReviewCompletedEvent` with that tenantId
3. `CaseMemoryEmitter` passes it to `MemoryInput.tenantId`
4. `InMemoryMemoryStore.assertTenant()` compares against the same `FixedCurrentPrincipal` singleton

Both values are `DEFAULT_TENANT_ID`. No request scope involved.

**Fix:** Remove `@Disabled`. If the test fails for a genuinely different reason, fix that
specific root cause. Fallback: create `NoTenantCheckMemoryStore` test `@Alternative @Priority(11)`
subclass overriding `assertTenant()` to no-op.

---

## Platform Coherence

All 5 issues build on existing platform primitives with no new SPIs, no placement
violations, and no boundary concerns:

- **#62**: Consistent with `trust-maturity-model.md` protocol
- **#65**: Follows per-field `PreferenceKey<T>` pattern (GE-20260530-9b5bbe)
- **#66**: Uses `CaseMemoryStore.eraseEntity()` SPI; auth-retrofit-ready
- **#67, #68**: Test-only — no platform implications

**Out of scope:** parent#179 (doc sync) is cross-repo — cannot commit to parent from
this session. Already tracked as a GitHub issue.
