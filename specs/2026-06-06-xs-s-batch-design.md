# XS/S Batch Design — devtown#62, #65, #66, #67, #68

**Date:** 2026-06-06
**Branch:** issue-62-xs-s-batch
**Covers:** devtown#62, #65, #66, #67, #68

---

## Implementation Order

Issues are ordered by dependency:

1. **#62** — standalone wiring change, no deps on other issues
2. **#67** — fix integration test infrastructure (enables #68 confidence)
3. **#68** — add test coverage (builds on verified test infrastructure from #67)
4. **#65** — configurable recall limits (modifies CaseMemoryRecaller, tests rely on #68 coverage)
5. **#66** — GDPR erasure endpoint (standalone, but benefits from all memory tests passing)

After each issue: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` must pass.

---

## Issue #62 — Wire bootstrapEscalationRequired through DevtownTrustRoutingPolicyProvider

**Scale:** XS (wiring change only — one line in the provider, three test methods). The
GitHub issue labels say S/Med because the issue describes the full behavioral fix (routing
to HumanOversight when no agent meets minimumObservations). But engine#415 shipped both the
`bootstrapEscalationRequired` field AND the enforcement logic in `TrustCandidateClassifier`.
The devtown-side wiring IS the remaining work — setting the flag correctly is all that's
left. The engine already consumes it.

**Closure plan:** This change closes devtown#62. The end-to-end behavior (bootstrap agents
escalate to HumanOversight for merge-executor) is delivered by engine#415 enforcement +
this wiring.

**Problem:** `DevtownTrustRoutingPolicyProvider.forCapability()` hardcodes `false` for
`bootstrapEscalationRequired` on line 73, ignoring the domain's `RoutingPolicy.fallbackType`
declaration.

**Fix:** Map `routingPolicy.fallbackType().isPresent()` → `bootstrapEscalationRequired`.

| Capability | fallbackType | bootstrapEscalationRequired |
|---|---|---|
| merge-executor | ROUTING_REVIEW | true |
| security-review | ROUTING_REVIEW | true |
| architecture-review | ROUTING_REVIEW | true |
| style-review | empty | false |
| unknown capability | returns DEFAULT | false (DEFAULT has false) |

Unknown capabilities return `TrustRoutingPolicy.DEFAULT` because they have no registry
entry — no routing policy means no escalation requirement. This is correct: escalation
is a domain design decision expressed through the registry, not a default.

**Changes:**
- `DevtownTrustRoutingPolicyProvider.forCapability()` — replace `false` with
  `routingPolicy.fallbackType().isPresent()`
- `DevtownTrustRoutingPolicyProviderTest` — add tests:
  - merge-executor has `bootstrapEscalationRequired = true`
  - style-review has `bootstrapEscalationRequired = false`
  - unknown capability returns DEFAULT (bootstrapEscalationRequired = false)

**No domain model changes.** `RoutingPolicy.fallbackType` already carries the information.

---

## Issue #67 — Integration test async tenancy fix

**Ordered second** to validate test infrastructure before adding coverage in #68.

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

**Potential failure modes beyond tenancy:**
- **Timing:** Test already uses `await().atMost(20, SECONDS)` with polling — adequate
- **Event ordering:** CDI `@ObservesAsync` has no ordering guarantees, but the chain uses
  `ReviewCompletedEventCapture` as a synchronization point before final assertions
- **Thread visibility:** `InMemoryMemoryStore` uses `ConcurrentHashMap` and
  `CopyOnWriteArrayList` — both thread-safe with full visibility guarantees

**Approach:**
1. Write a minimal `@QuarkusTest` that fires a CDI async event, has an `@ObservesAsync`
   handler call `store.store()`, then asserts the fact is queryable. This isolates the
   assertTenant-through-async question from the full round-trip complexity.
2. If the minimal test passes → remove `@Disabled` from the full integration test
3. If it fails → create `NoTenantCheckMemoryStore` test `@Alternative @Priority(11)`

**Fix:** Remove `@Disabled` and remove the stale comment about async tenancy unavailability.
Update the test class javadoc to note that `FixedCurrentPrincipal` is `@ApplicationScoped`
and therefore available in async observer threads.

---

## Issue #68 — Code-area recall test + empty changedPaths edge case

**Ordered third** — builds on the test infrastructure validated by #67.

**Gap 1 (code-area recall):** `CaseMemoryRecallerTest` has a comment at lines 26–32
claiming "InMemoryMemoryStore state doesn't persist across recaller.recall() calls in
test context." **This comment is stale and incorrect.** Evidence:

- `InMemoryMemoryStore` is `@ApplicationScoped` — a CDI singleton with shared state
- `recall_with_contributor_history_returns_populated_context` (line 75) successfully
  stores a fact then queries it back through `recaller.recall()` — proving state persists
- `store_and_query_directly_works` (line 157) stores a module-level fact and queries it
  back successfully

The existing passing tests prove cross-bean state persistence works. The code-area recall
test should work the same way.

**Fix:** Add test that:
1. Pre-populates a code-area fact with `module:repo1/app` entityId
2. Calls `recaller.recall()` with a PR whose `changedPaths` contains `app/src/main/Foo.java`
   (normalizes to `app` via `ModulePathNormalizer`)
3. Asserts `codeAreaHistory` is non-empty with correct entityId
4. Remove the stale comment at lines 26–32

**Gap 2 (empty changedPaths):** Straightforward — no concerns.

**Fix:** Add test in `CaseMemoryEmitterTest` firing event with `changedPaths = List.of()`,
assert exactly 2 facts (contributor + reviewer), assert no code-area entity type.

**No production code changes.**

---

## Issue #65 — Configurable recall limits via PreferenceKey

**Problem:** `CaseMemoryRecaller` hardcodes contributor limit (10), code area limit (15),
and time window (90 days).

**New class:** `MemoryRecallKeys` in `domain/.../memory/`

Three `PreferenceKey<IntPreference>` constants:
- `CONTRIBUTOR_LIMIT` — default 10
- `CODE_AREA_LIMIT` — default 15
- `TIME_WINDOW_DAYS` — default 90

PreferenceKey namespace (dot-separated, used for map key lookup):
`casehubio.devtown.memory-recall`

Resolution path (slash-separated, used for hierarchical preference lookup via
`SettingsScope.of()`): `casehubio/devtown/memory-recall`

**Modified class:** `CaseMemoryRecaller` — inject `PreferenceProvider`, resolve preferences
at the resolution path, use `Preferences.getOrDefault(key)` (not manual null-handling —
`getOrDefault()` on the `Preferences` interface already consults `key.defaultValue()` on
null). This follows the established pattern from `SlaBreachPolicy`.

**Placement:** `CaseMemoryRecaller` stays in `app/` (Tier 3, CDI wiring). Injecting
`PreferenceProvider` (a platform type) is correct at this tier.

**Tests:**
- `MemoryRecallKeysTest` (domain, unit) — parse round-trip for each key
- `CaseMemoryRecallerTest` — add test with populated `PreferenceProvider` overriding
  contributor limit to 1, verify only 1 result returned

**Relevant garden entries:**
- GE-20260530-9b5bbe: Per-field PreferenceKey pattern — follow this structure
- GE-20260530-9cdfb5: `MapPreferences.get()` returns null when key absent — mitigated
  by using `getOrDefault()` instead

---

## Issue #66 — GDPR erasure endpoint for contributor memory

**Approach:** Admin REST endpoint, not scheduled job. GDPR Art.17 is erasure-on-request.

**eraseEntity() availability:** `InMemoryMemoryStore` DOES implement `eraseEntity()` —
verified from decompiled source. It calls `assertTenant()` then removes matching
`(tenantId, entityId)` keys from the ConcurrentHashMap. Not a blocker.

**New class:** `MemoryAdminResource` in `app/`
- `@Path("/api/admin/memory")` — admin-prefixed path anticipates future admin surfaces
  (`/api/admin/routing/...`, `/api/admin/trust/...`)
- `POST /erase/contributor` with JSON body `{"login": "..."}` — avoids PII in URL path
  (DELETE with login in path leaks to HTTP access logs, CDN logs, load balancer logs)
- Calls `store.eraseEntity("contributor:" + login, principal.tenancyId())`
- Returns 204 No Content
- Tenant-scoped via `CurrentPrincipal.tenancyId()` — prevents cross-tenant erasure

**EntityId format:** Matches `CaseMemoryEmitter.buildContributorFact()` at line 63:
`"contributor:" + pr.contributor()`.

**No application service layer** — straight pass-through to platform SPI.

Code area facts do not contain contributor PII by design (verified by existing
`code_area_text_does_not_contain_contributor_login` test in `CaseMemoryEmitterTest`).

**Race condition with in-flight emissions:** If a PR review is in progress during erasure,
`CaseMemoryEmitter` will create new contributor facts on the next `ReviewCompletedEvent`.
This is by design — GDPR Art.17 requires erasure of existing data, not suppression of
future processing. If future-suppression is needed, that is a separate consent-withdrawal
mechanism (not in scope).

**Authorization:** No `@RolesAllowed` annotation now — consistent with the platform's
auth-retrofit-readiness protocol (no auth annotations on REST resources until RBAC is
implemented). The thin resource structure supports adding `@RolesAllowed("admin")` when
RBAC arrives.

**Deferred: erasure audit trail.** GDPR Art.5(2) requires demonstrating compliance. The
endpoint should log who requested erasure, when, and what was erased. Filed as a follow-up
issue (created during implementation).

**Tests:** `MemoryAdminResourceTest` (`@QuarkusTest`)
- Store contributor fact, call POST erase, verify query returns empty
- POST erase for non-existent contributor returns 204 (idempotent)

---

## Platform Coherence

All 5 issues build on existing platform primitives with no new SPIs, no placement
violations, and no boundary concerns:

- **#62**: Consistent with `trust-maturity-model.md` — every capability with a routing
  policy MUST declare a fallbackType; engine#415 enforces the flag
- **#65**: Follows per-field `PreferenceKey<T>` pattern; uses `getOrDefault()` API
- **#66**: Uses `CaseMemoryStore.eraseEntity()` SPI; auth-retrofit-ready; admin-prefixed path
- **#67, #68**: Test-only — no platform implications

**Out of scope:** parent#179 (doc sync) is cross-repo — cannot commit to parent from
this session. Already tracked as a GitHub issue.

**Follow-up issue to file:** GDPR erasure audit trail logging (devtown).
