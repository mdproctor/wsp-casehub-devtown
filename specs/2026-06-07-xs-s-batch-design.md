# XS/S Batch — Issues #69, #70, #71, parent#179

**Date:** 2026-06-07
**Branch:** issue-69-xs-s-batch
**Covers:** devtown#69, devtown#70, devtown#71, casehubio/parent#179

**Implementation order:** #69 → #70 → #71 → parent#179

#70 and #71 both modify `MemoryAdminResource.java`. #70 (logging) is committed first so that #71 (auth annotation) is a clean single-line addition. This avoids the test needing `@TestSecurity` awareness for the logging assertions.

---

## #69 — Fix LEDGER_SUBJECT_SEQUENCE H2 test failure

**Problem:** `CaseMemoryIntegrationTest.fullRoundTrip_emitThenRecall()` is `@Disabled` because a casehub-ledger SNAPSHOT introduced `LedgerSequenceAllocator`, which uses a native SQL `MERGE INTO ledger_subject_sequence` table. This table is created by a Flyway migration in `classpath:db/ledger/migration`, but devtown's `@QuarkusTest` uses `drop-and-create` with Flyway disabled — Hibernate never creates the table because it's not a JPA `@Entity`.

**Root cause analysis:** The error trace is:

```
CaseLedgerEventCapture.onCaseLifecycleEvent()
  → @Inject CaseLedgerEntryRepository ledgerRepo
    → CaseLedgerEntryRepository extends JpaLedgerEntryRepository
      → JpaLedgerEntryRepository.save() calls LedgerSequenceAllocator.nextSequenceNumber()
        → MERGE INTO ledger_subject_sequence  ← table not found
```

`CaseLedgerEventCapture` (casehub-engine-ledger) injects `CaseLedgerEntryRepository` — a **concrete class** that extends `JpaLedgerEntryRepository`, not the `LedgerEntryRepository` interface. The ledger-memory module's `InMemoryLedgerEntryRepository` implements `LedgerEntryRepository` — a different type. CDI will not substitute it for `CaseLedgerEntryRepository` injection points. `@DefaultBean` on `CaseLedgerEntryRepository` only yields when another bean of the **same type** exists.

GE-20260607-ad3d62 describes the general pattern for downstream consumers using `JpaLedgerEntryRepository` directly. The devtown case is structurally different because casehub-engine-ledger interposes `CaseLedgerEntryRepository` between the consumer and the base repo.

**Existing devtown code:** `app/src/test/java/.../InMemoryLedgerEntryRepository.java` already exists — implements `ReactiveLedgerEntryRepository` (Uni-based reactive interface). This satisfies reactive injection points. It is unrelated to the `CaseLedgerEntryRepository` (blocking JPA) that `CaseLedgerEventCapture` injects.

**Fix: Approach (A) — disable the ledger in tests.**

Set `casehub.ledger.enabled=false` in test `application.properties`. Both `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` already check `ledgerConfig.enabled()` and return early (lines 53 and 64 respectively). `save()` is never called, `LedgerSequenceAllocator` is never invoked, the missing table is never touched.

This is the simplest correct fix. `casehub.ledger.enabled` is a **write-side master switch** — it gates `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` only. It does not affect `TrustGateService`, `TrustWeightedAgentStrategy`, or `ActorTrustScoreRepository`. The three existing trust-related `@QuarkusTest` classes (`TrustRoutingActivationTest`, `TrustGateWiringTest`, `DevtownObligorTrustPolicyTest`) inject ledger trust-scoring beans and are unaffected by this setting.

No devtown tests currently assert on ledger entries (verified: no references to `CaseLedgerEntry`, `WorkerDecisionEntry`, `findByCaseId`, or `ledgerRepo` in `app/src/test/`). Disabling the ledger writes in tests breaks nothing.

**Why not the other approaches:**
- **(B) Exclude captures from CDI:** More targeted but fragile — `quarkus.arc.exclude-types` is a string list that can fall out of sync with class renames. `casehub.ledger.enabled=false` is the supported config knob that the captures already respect.
- **(C) In-memory `CaseLedgerEntryRepository` alternative:** Most work, and unnecessary — no devtown test currently needs ledger entry assertions. If that changes, this approach can be revisited.

**Architectural follow-up (engine#436):** The root cause is that `CaseLedgerEntryRepository` extends `JpaLedgerEntryRepository` (a concrete JPA class) rather than depending on `LedgerEntryRepository` via composition. If it injected `LedgerEntryRepository`, swapping in the in-memory alternative would work naturally. This is a casehub-engine-ledger design issue, not a devtown issue.

**Changes:**

1. `app/src/test/resources/application.properties` — add `casehub.ledger.enabled=false` (no `%test.` prefix needed — this file is already test-only). Place it near the existing datasource section. The main `application.properties` has `casehub.ledger.datasource=qhorus` but no `enabled` override, so the ledger defaults to `enabled=true` in dev/prod — this split is intentional.
2. `CaseMemoryIntegrationTest.java` — remove `@Disabled` annotation from `fullRoundTrip_emitThenRecall()`
3. `ARC42STORIES.MD` §9.4 Layer — casehub-engine — add to Pattern to replicate step 8: "Set `casehub.ledger.enabled=false` in test `application.properties` if `casehub-engine-ledger` is on the classpath. Without this, `CaseLedgerEventCapture` calls `JpaLedgerEntryRepository.save()` → `LedgerSequenceAllocator.nextSequenceNumber()` → `MERGE INTO ledger_subject_sequence`, which fails in H2 `drop-and-create` mode (the table is created by Flyway, not Hibernate)." Also update the existing Gotcha to clarify that the `InMemoryLedgerEntryRepository` stub (step 7) solves reactive CDI resolution, while `casehub.ledger.enabled=false` solves the blocking write chain — both are needed.

**Verification:** Run `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` and confirm `CaseMemoryIntegrationTest.fullRoundTrip_emitThenRecall` passes.

---

## #70 — GDPR erasure audit trail logging

**Problem:** `MemoryAdminResource.eraseContributor()` performs erasure silently — no audit record of who requested it or what was erased.

**Requirement (from issue body):** GDPR Art.5(2) requires demonstrating compliance. Log: who requested, when, what was erased, outcome. The issue explicitly calls for logging **before and after** the `eraseEntity()` call.

**Fix:** Add SLF4J structured logging to the erasure endpoint — before the attempt, after success, and on failure.

**Changes:**

1. `MemoryAdminResource.java`:
   - Refactor from field injection to **constructor injection** — move `CaseMemoryStore` and `CurrentPrincipal` from `@Inject` fields to constructor parameters. This makes the class testable without CDI and explicit about its dependencies. Since #70 is already modifying this class, the change fits naturally.
   - Add `Logger` field (`org.jboss.logging.Logger` — Quarkus standard, static final)
   - **Before erasure:** `LOG.infof("GDPR erasure requested — entityId=%s, requestedBy=%s, tenantId=%s", entityId, principal.actorId(), principal.tenancyId())` — ensures the request is recorded even if the operation crashes mid-execution
   - **After successful erasure:** `LOG.infof("GDPR erasure completed — entityId=%s, requestedBy=%s, tenantId=%s", ...)`
   - **On `UnsupportedOperationException`:** `LOG.warnf("GDPR erasure not supported by active adapter — entityId=%s, requestedBy=%s", ...)`

**Known limitation (platform#72):** `eraseEntity()` returns `void` — there is no way to log how many records were actually deleted. This is a `CaseMemoryStore` API limitation. The log records what was requested and that the operation completed without exception. Filed platform#72 to change the return type to `int` — the migration is mechanical (three implementations).

**Tests:** Two test classes cover the logging — split by technique.

**Success-path log test** (`MemoryAdminResourceTest.java`, existing `@QuarkusTest`): Add a test that registers a `java.util.logging.Handler` on `"io.casehub.devtown.app.MemoryAdminResource"` before calling the endpoint via REST-assured (existing pattern). JBoss Logging delegates to JUL in the Quarkus test environment. Assert:
- Erasure request log appears before completion log
- Both logs contain the entityId and actorId
- Format matches expected structured fields

**UnsupportedOperationException log test** (new plain unit test, e.g. `MemoryAdminResourceUnitTest.java`): The active `InMemoryMemoryStore` supports `eraseEntity()` — it won't throw. With constructor injection, construct the resource directly with `Mockito.mock()`:

```java
var store = mock(CaseMemoryStore.class);
doThrow(UnsupportedOperationException.class).when(store).eraseEntity(any(), any());
var principal = mock(CurrentPrincipal.class);
when(principal.actorId()).thenReturn("admin-user");
when(principal.tenancyId()).thenReturn("default");
var resource = new MemoryAdminResource(store, principal);
// register JUL handler, call eraseContributor(), verify WARN log
```

This follows devtown's existing mock pattern (`CaseMemoryEmitterTest`, `CaseMemoryRecallerPreferenceTest`) and adds no new dependencies. `devtown` already has `mockito-core` in test scope.

**Rationale:** Application-level audit logging, not a ledger entry. The ledger is for tamper-evident records of case decisions. Admin operations belong in structured logs, queryable via the observability stack. But because this is GDPR compliance logging, test coverage for the log format matters more than usual — a silent regression could break demonstrable compliance.

---

## #71 — Add authorization to MemoryAdminResource

**Problem:** `MemoryAdminResource` has no authorization — anyone reaching the endpoint can erase memory.

**Fix:** Add `@RolesAllowed("admin")` to the resource class.

**Changes:**

1. `MemoryAdminResource.java` — add `@RolesAllowed("admin")` at the class level
2. Import: `jakarta.annotation.security.RolesAllowed` (the standard JAX-RS annotation, not any Quarkus-specific variant)

**Rationale:** The platform's RBAC infrastructure is implemented: `CurrentPrincipal.roles()` is a default method that delegates to `groups()`, and `casehub-platform-oidc` ships `OidcCurrentPrincipal @RequestScoped` which reads roles from `SecurityIdentity.getRoles()`. `@RolesAllowed` is the prescribed mechanism per the auth-retrofit-readiness protocol. The annotation is architecturally correct now and enforces automatically when `casehub-platform-oidc` is added to the classpath.

**Test impact:** No security extension is active in devtown's test classpath, so `@RolesAllowed` is inert in tests — existing tests pass unchanged. When OIDC is adopted, tests will need `@TestSecurity` annotations (that's the OIDC adoption issue, not this one).

**Role name convention (parent#189):** The role name `"admin"` is not documented as a platform convention. This is the first use of a named role in any CaseHub harness. Filed parent#189 to document the role name convention (which role names exist, what they mean, how they map to OIDC groups) before production deployment.

**Related:** parent#187 (docs: update PLATFORM.md to reflect RBAC infrastructure is implemented) — already closed.

**Pre-existing note:** `MemoryAdminResource` has `@Consumes(APPLICATION_JSON)` but no `@Produces`. The 501 error path returns a plain string entity, not JSON. Deliberately not adding `@Produces(APPLICATION_JSON)` — the 204 success path has no body, and the 501 body is a plain error string. If the 501 response is ever structured as JSON, add `@Produces` at that time.

---

## parent#179 — Doc sync: devtown as CaseMemoryStore consumer

**Problem:** PLATFORM.md's Cross-Repo Dependency Map has no entry for `casehub-platform-memory-inmem` → devtown. The `repos/casehub-devtown.md` deep-dive doesn't mention CaseMemoryStore usage.

**Fix:** Update parent repo documentation.

**Changes:**

1. `docs/PLATFORM.md` — add dependency row: `casehub-platform-memory-inmem` | `devtown` | `app` | In-memory CaseMemoryStore for test isolation
2. `docs/repos/casehub-devtown.md` — add CaseMemoryStore section documenting: memory-inmem for tests, emission chain (ReviewOutcomeObserver → CaseMemoryEmitter), recall chain (CaseMemoryRecaller), domain (`SOFTWARE_REVIEW`), entity patterns (`contributor:{login}`, `module:{repo}/{module}`)

**Cross-repo mechanics:** This is a docs-only change in `casehubio/parent`. Per CLAUDE.md: "Never commit or push to peer repo directories." This will be committed in a separate parent session or via GitHub PR from a parent branch — not from this devtown session. The parent workspace HANDOFF.md will be updated per the cross-repo HANDOFF convention.

---

## Follow-up issues filed

1. **engine#436** — `CaseLedgerEntryRepository` extends `JpaLedgerEntryRepository` (concrete class) instead of injecting `LedgerEntryRepository` (interface) via composition. Every downstream consumer hits the same `LEDGER_SUBJECT_SEQUENCE` problem because the in-memory alternative can't substitute for the concrete class.
2. **parent#189** — document `@RolesAllowed` role name convention — which roles exist, what they mean, how they map to OIDC groups.
3. **platform#72** — `CaseMemoryStore.eraseEntity()` should return `int` (count of records deleted) for GDPR Art.5(2) compliance logging.
