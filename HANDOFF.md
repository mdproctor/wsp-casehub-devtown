# Handoff — Epic #237: test adaptation after upstream migrations

**Date:** 2026-10-14
**Branch:** `issue-237-test-adaptation-defaultbean-migrations`
**Last commit:** `e495980` fix: fix in-memory/JPA data split in ledger tests
**Baseline:** 46 failing tests (36F + 10E) → **13 remaining** (11F + 2E)

## What happened this session

### Issues completed: #238, #239 (fully), #240 (partial)

**#238 + #239 — REST security after @McpDomain migration (24 tests fixed):**
- Root cause: `deny-unannotated-endpoints=true` in test config + `@McpDomain` generated REST resources had no security annotations
- Fix: Added method-level `@RolesAllowed` to all `@McpDomain` API classes (`DevtownGovernanceApi`, `DevtownReviewApi`, `DevtownComplianceApi`, `DevtownTrustApi`) — the annotation processor propagates these to generated JAX-RS resources
- Security model: governance/trust queries → all authenticated roles; submitReview → ENGINEER/ADMIN/SERVICE; recordFeedback → ADMIN; erasure → DATA_CONTROLLER/ADMIN; eraseContributor → ADMIN
- Updated test URLs: webhook `/api/github/webhook` → `/api/github/receive`; erasure `/api/actors/{id}/erasure` → `/api/devtown/compliance/erasure/{id}`; incident feedback `/api/incident-feedback` → `/api/devtown/reviews/feedback`
- Switched erasure/memory-erase tests from JSON body to `queryParam()` (generated resources map simple params as `@QueryParam`)
- Added `@TestSecurity` to `GovernancePreferencesResourceTest` and `TrustEndpointTest`
- Added GDPR erasure logging to `DevtownComplianceApi.eraseContributor` (migrated from deleted `MemoryAdminResource`)

**#240 — Ledger test failures (9 tests fixed, 7 remaining):**
- Root cause 1: `IncidentFeedbackService.resolveMergeDecision()` used `EntityManager` named query (queries H2) while `InMemoryLedgerEntryRepository` stores data in memory
- Fix: Replaced `EntityManager` query with `ledgerRepo.findByTimeRange()` + type filter scan; injected `CurrentPrincipal` for tenancy
- `InMemoryLedgerEntryRepository`: implemented stub methods (findAttestationsByEntryId, findAttestationsByAttestorIdAndCapabilityTag, findAttestationsByEntryIdGlobal, findByActorId, findByTimeRange); auto-assign UUIDs on save/saveAttestation
- `MergeDecisionObserverTest`: replaced `EntityManager.createQuery()` with `ledgerRepo.findBySubjectId()` + filter
- CDI fix: excluded `LegacyTrustGateShim` — `LedgerCoreProducer` now provides `TrustGateService`; the shim's `@DefaultBean` didn't yield because CDI compares declared return types, not bean type hierarchy

### Key discovery: @McpDomain annotation processor security propagation
The `GraphQLResolverProcessor` reads `@RolesAllowed` and `@PermitAll` from **method-level** annotations on `@McpDomain` classes and propagates them to generated REST resources. Class-level annotations are NOT propagated. `@PlatformWebhook` methods automatically get `@PermitAll`.

## What needs to happen next

### #240 — 7 remaining ledger test failures

| Test | Failure | Root cause | Fix |
|------|---------|-----------|-----|
| IncidentFeedbackServiceTest.ambiguousPr_throws409 | 404 instead of 409 | findByTimeRange scan doesn't find seeded merge decisions (unclear why — timing or test ordering) | Debug why `findByTimeRange` returns empty for these 2 tests when happyPath works |
| IncidentFeedbackServiceTest.disambiguatedWithCaseId | 404 not found | Same root cause as above | Same |
| CodeReviewComplianceServiceTest | Unknown | Likely old REST URL or in-memory data split | Check surefire report |
| ComplianceErasureDetectionTest | Assertion failure | TokenisationEnabledTestProfile — tokenisation not working | Investigate ActorIdentityProvider tokenisation setup |
| GdprErasureResourceTest.erasure_pseudonymisesActor | token == rawActorId | Tokenisation assertion — actorIdentityProvider.tokeniseForQuery returns raw ID | Same tokenisation profile issue |
| TrustFeedbackClosedLoopTest | SOUND→FLAGGED | Verdict computation changed in ledger #221 | Trace verdict calculation path, update expected value |
| TrustGatedAttestationPolicyActivationTest | Unknown | Needs investigation | Check surefire report |

### #241 — 6 PrReviewQhorusLifecycleTest failures

**Root cause diagnosed:** Qhorus upgrade changed default `enforcementMode` from `ADVISORY` to `BLOCKING` when `enforcementMode` is `null` on channel creation. `QhorusPrReviewService` creates channels via old constructor that passes `null` for enforcement mode.

**Fix options:**
1. **(Recommended) Keep BLOCKING:** Update advisory tests to expect `EnforcementBlocked` exception. Tests like `observeChannel_advisoryOnSpeechActDispatch` send disallowed message types expecting soft warnings — now they get hard blocks.
2. **Explicit ADVISORY:** Pass `EnforcementMode.ADVISORY` in channel creation if advisory mode is intentionally wanted.

The `QhorusPrReviewService` creates channels at lines ~197/204/211 using the old 15-arg `ChannelCreateRequest` constructor.

### .plan queue status
- [x] #237 (epic parent)
- [~] #238 (done but .plan shows active)
- [ ] #239 (done)
- [ ] #240 (partial — 7 of 13 remaining)
- [ ] #241 (not started — 6 tests)

### Build command
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
