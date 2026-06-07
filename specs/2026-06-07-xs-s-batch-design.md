# XS/S Batch — Issues #69, #70, #71, parent#179

**Date:** 2026-06-07
**Branch:** issue-69-xs-s-batch
**Covers:** devtown#69, devtown#70, devtown#71, casehubio/parent#179

---

## #69 — Fix LEDGER_SUBJECT_SEQUENCE H2 test failure

**Problem:** `CaseMemoryIntegrationTest.fullRoundTrip_emitThenRecall()` is `@Disabled` because a casehub-ledger SNAPSHOT introduced `LedgerSequenceAllocator`, which uses a native SQL `MERGE INTO ledger_subject_sequence` table. This table is created by a Flyway migration in `classpath:db/ledger/migration`, but devtown's `@QuarkusTest` uses `drop-and-create` with Flyway disabled — Hibernate never creates the table because it's not a JPA `@Entity`.

**Fix:** Switch from `JpaLedgerEntryRepository` to `InMemoryLedgerEntryRepository` in the test configuration. The in-memory implementation uses a `ConcurrentHashMap` — no SQL table required.

**Changes:**

1. `app/pom.xml` — add `casehub-ledger-memory` test dependency
2. `app/src/test/resources/application.properties`:
   - Add Jandex `index-dependency` for `casehub-ledger-memory`
   - Add `io.casehub.ledger.memory.InMemoryLedgerEntryRepository` to `quarkus.arc.selected-alternatives`
3. `CaseMemoryIntegrationTest.java` — remove `@Disabled` annotation from `fullRoundTrip_emitThenRecall()`

**Reference:** GE-20260607-ad3d62 — documents this exact failure and fix pattern.

---

## #70 — GDPR erasure audit trail logging

**Problem:** `MemoryAdminResource.eraseContributor()` performs erasure silently — no audit record of who requested it or what was erased.

**Fix:** Add SLF4J structured logging to the erasure endpoint.

**Changes:**

1. `MemoryAdminResource.java`:
   - Add `Logger` field
   - On successful erasure: `LOG.info("GDPR erasure completed — entityId={}, requestedBy={}, tenantId={}", ...)` using `CurrentPrincipal.actorId()` for the requester
   - On `UnsupportedOperationException`: `LOG.warn("GDPR erasure not supported — entityId={}, adapter={}", ...)`

**Rationale:** Application-level audit logging, not a ledger entry. The ledger is for tamper-evident records of case decisions. Admin operations belong in structured logs, queryable via the observability stack.

---

## #71 — Add authorization to MemoryAdminResource

**Problem:** `MemoryAdminResource` has no authorization — anyone reaching the endpoint can erase memory.

**Fix:** Add `@RolesAllowed("admin")` to the resource class.

**Changes:**

1. `MemoryAdminResource.java` — add `@RolesAllowed("admin")` at the class level

**Rationale:** The platform's RBAC infrastructure is implemented: `CurrentPrincipal.roles()` delegates to `groups()`, and `casehub-platform-oidc` reads roles from `SecurityIdentity.getRoles()`. `@RolesAllowed` is the prescribed mechanism per the auth-retrofit-readiness protocol. The annotation is architecturally correct now and enforces automatically when `casehub-platform-oidc` is added to the classpath.

**Test impact:** No security extension is active in devtown's test classpath, so `@RolesAllowed` is inert in tests — existing tests pass unchanged. When OIDC is adopted, tests will need `@TestSecurity` annotations (that's the OIDC adoption issue, not this one).

**Filed:** parent#187 — docs: update PLATFORM.md to reflect RBAC infrastructure is implemented.

---

## parent#179 — Doc sync: devtown as CaseMemoryStore consumer

**Problem:** PLATFORM.md's Cross-Repo Dependency Map has no entry for `casehub-platform-memory-inmem` → devtown. The `repos/casehub-devtown.md` deep-dive doesn't mention CaseMemoryStore usage.

**Fix:** Update parent repo documentation.

**Changes:**

1. `docs/PLATFORM.md` — add dependency row: `casehub-platform-memory-inmem` | `devtown` | `app` | In-memory CaseMemoryStore for test isolation
2. `docs/repos/casehub-devtown.md` — add CaseMemoryStore section documenting: memory-inmem for tests, emission chain (ReviewOutcomeObserver → CaseMemoryEmitter), recall chain (CaseMemoryRecaller), domain (`SOFTWARE_REVIEW`), entity patterns (`contributor:{login}`, `module:{repo}/{module}`)

**Cross-repo:** This is a docs-only change in `casehubio/parent`. Will be committed to the parent repo directly.
