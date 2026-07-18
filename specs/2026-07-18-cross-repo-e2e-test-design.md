# Cross-Repo Coordinated Merge End-to-End Integration Test Design

**Issue:** #160 (Epic #12 — Phase 2 final)
**Date:** 2026-07-18
**Branch:** issue-160-cross-repo-e2e-test

## Problem

All coordinated change components are implemented (#155–#159) but only unit-tested with Mockito. No `@QuarkusTest` exercises the full coordination flow: `CoordinatedChangeService` → engine-driven review cases → observer → parent case bindings → merge/rollback workers → terminal state. The multi-case, multi-YAML-model interaction through async CDI events is the riskiest seam and has zero integration coverage.

## Done When

"A 2-repo change set merges atomically, and a failure in one repo triggers automatic rollback of the other." — verified in a `@QuarkusTest` with phased checkpoint assertions.

## Design Decisions

### Full engine lifecycle for review cases

Review cases are started via `CoordinatedChangeService.start()` → `PrReviewApplicationService.startReview(pr, Map.of("coordinatedChange", true))` and driven to completion by signaling their context to satisfy pr-review.yaml goals. This proves:

- The `coordinatedChange` flag suppresses per-repo merge bindings (`.coordinatedChange != true` → `false`)
- The `coordinatedChange` flag satisfies `merge-completed` goal without an actual merge
- The engine evaluates goals, transitions cases, and fires `CaseLifecycleEvent`
- The observer receives events via `@ObservesAsync` CDI delivery (the production path)
- The observer correctly signals the parent case context

Capability bindings (code-analysis, style-review, etc.) would fire on review case creation because `PrReviewCaseHub` only registers a `merge-executor` worker. With no workers for other capabilities, the engine exhausts reroute attempts (`maxRerouteAttempts: 2`) and writes `status: "REROUTES_EXHAUSTED"`, eventually triggering human escalation bindings that create WorkItems. Per protocol PP-20260521-134c38, the test pre-seeds all capability keys with non-null PENDING values immediately after review case creation (see `preSeedCapabilityKeys` helper). This prevents binding guards (which check `== null`) from firing, avoiding reroute churn, WorkItem pollution, and DEEP_MERGE context races. `driveReviewToCompletion` then overwrites PENDING with APPROVED values.

### Observable @Alternative stubs (not Mockito)

Per project protocol, external clients use `@Alternative` static inner classes. The stubs are both programmable (queued outcomes) and observable (recorded calls), replacing Mockito `verify()`.

### Phased checkpoint assertions

Each test verifies every coordination phase independently. If the test fails, the checkpoint identifies exactly which phase broke. Phases:

1. **Initiation** — parent + review cases created, tracker populated
2. **Review lifecycle** — review cases reach terminal state
3. **Coordination bridge** — observer signals parent context
4. **Worker execution** — merge/rollback workers called with correct arguments
5. **Terminal state** — parent case status, context values

### cancelCase() for fault path

`driveReviewToFault()` uses `caseHubRuntime.cancelCase()` (produces CANCELLED) rather than signaling `pr.status = "closed"` (which trips `review-abandoned` failure goal, producing COMPLETED). Reason: the observer classifies COMPLETED as TERMINAL_SUCCESS. A review case that fails via a failure goal would be misrouted as a successful completion. This is a known semantic gap — the observer doesn't distinguish success-COMPLETED from failure-COMPLETED. Using cancelCase() exercises the correct coordination path. The semantic gap should be filed as a separate issue.

### linesChanged below human approval threshold

All test `RepoChangeEntry` objects use `linesChanged: 10` to prevent the `human-approval` humanTask binding from firing on review cases. The test controls the review lifecycle via signals, not human gates.

## Test Class

**File:** `app/src/test/java/io/casehub/devtown/app/CrossRepoCoordinatedMergeTest.java`

`@QuarkusTest` with 5 test methods, 2 `@Alternative` inner classes, 4 helper methods.

### @Alternative Inner Classes

**`TestMergeClient`** — `@Alternative @Priority(1) @ApplicationScoped`, implements `MergeClient`:

```
Queue<MergeOutcome> outcomes   — programmed per test; merge() polls; throws if empty
List<MergeCall> calls          — appends on every merge() invocation
record MergeCall(String owner, String repo, int prNumber, String headSha)
void enqueue(MergeOutcome...)  — convenience for programming
void reset()                   — clears queue + call list (@BeforeEach)
```

**`TestRevertClient`** — same pattern for `RevertClient`:

```
Queue<RevertOutcome> outcomes
List<RevertCall> calls
record RevertCall(String owner, String repo, String targetBranch, String mergeSha, String commitMessage)
void enqueue(RevertOutcome...)
void reset()
```

### Injections

```
CoordinatedChangeService          — entry point under test
CoordinatedChangeTracker          — verify tracker state
PrReviewCaseHub                   — signal review cases
CoordinatedChangeCaseHub          — signal parent case (idempotent guard test)
CaseInstanceRepository              — read case state (returns Uni<CaseInstance>)
CaseHubRuntime                    — cancelCase() for fault path
WorkItemQueries                   — verify human escalation WorkItem (scenario 3)
TestMergeClient                   — program + verify merge calls
TestRevertClient                  — program + verify revert calls
```

### Helper Methods

**`CoordinatedChangeRequest buildRequest(RepoChangeEntry... repos)`**

Builds request from varargs. All entries use `linesChanged: 10`, non-empty `changedPaths`, valid `contributor`.

**`void preSeedCapabilityKeys(UUID reviewCaseId)`**

Signals non-null PENDING values for all capability keys. Prefers the batch `caseHubRuntime.signal(reviewCaseId, Map<String, Object>)` API; falls back to sequential `signal(UUID, String, Object)` calls if the batch variant throws `UnsupportedOperationException` (it is a `default` method). Sequential ordering is safe: pre-seeding `codeAnalysis` first blocks `initial-analysis` and indirectly blocks all downstream bindings (they require `.codeAnalysis.complete == true`); pre-seeding `ci` blocks `run-ci`. Called immediately after `CoordinatedChangeService.start()` returns, before any `driveReviewToCompletion` call. Pre-seeding must arrive before the engine's asynchronous binding evaluation cycle — this holds because `startCase()` returns before bindings are evaluated, and the pre-seeding signal is queued ahead of the deferred evaluation. Per PP-20260521-134c38.

| Signal path | Value |
|-------------|-------|
| `codeAnalysis` | `{outcome: "PENDING"}` |
| `styleCheck` | `{outcome: "PENDING"}` |
| `testCoverage` | `{outcome: "PENDING"}` |
| `performanceAnalysis` | `{outcome: "PENDING"}` |
| `ci` | `{status: "PENDING"}` |

**`void driveReviewToCompletion(UUID reviewCaseId)`**

Signals the review case context to satisfy all four pr-review.yaml success goals (overwrites PENDING values from pre-seeding):

| Signal path | Value | Goal satisfied |
|-------------|-------|---------------|
| `codeAnalysis` | `{complete: true, securitySensitive: false, architectureCrossing: false}` | pr-approved (security/architecture conditions) |
| `styleCheck` | `{outcome: "APPROVED"}` | pr-approved |
| `testCoverage` | `{outcome: "APPROVED"}` | pr-approved |
| `performanceAnalysis` | `{outcome: "APPROVED"}` | pr-approved |
| `ci` | `{status: "passing"}` | ci-passing |
| `coordinatedChange` | already `true` from creation | merge-completed |

security-verified is satisfied by `codeAnalysis.securitySensitive == false and securityReview == null`.

**`void driveReviewToFault(UUID reviewCaseId)`**

Calls `caseHubRuntime.cancelCase(reviewCaseId)`. Produces `CaseLifecycleEvent` with status CANCELLED (TERMINAL_FAILURE). Observer signals `reviewFaulted` to parent.

**`void awaitCaseStatus(UUID caseId, CaseStatus expected)`**

Awaitility: `await().atMost(10, SECONDS).pollInterval(100, MILLISECONDS)` → reads `caseInstanceRepository.findByUuid(caseId).await().indefinitely()` → asserts `instance.getState()` matches expected status.

## Test Scenarios

### Scenario 1: Happy path — all reviews complete, merge succeeds

**Setup:** 2-repo request (engine #42, platform #99). TestMergeClient: 2× Success.

**Phase 1 — Initiation:**
- `coordinatedChangeService.start(request)` → outcome
- Assert parent case exists, status ACTIVE
- Assert both review case IDs non-null
- Assert `tracker.findByReviewCaseId()` returns correct parent mapping
- Assert parent context contains `reviewCases` mapping

**Phase 2 — Review lifecycle:**
- `driveReviewToCompletion(reviewCaseIdA)`
- `driveReviewToCompletion(reviewCaseIdB)`
- Await both review cases reach COMPLETED

**Phase 3 — Coordination bridge:**
- Await parent context `completedReviews.casehubio/engine.status` == `"completed"`
- Await parent context `completedReviews.casehubio/platform.status` == `"completed"`
- Await parent context `allReviewsCompleted` == `true`

**Phase 4 — Worker execution:**
- Await parent context `mergeResults` non-null
- Assert `testMergeClient.calls` size 2, order: engine then platform
- Assert `mergeResults` has 2 entries, both `status: "success"` with `mergeSha`

**Phase 5 — Terminal state:**
- `awaitCaseStatus(parentCaseId, CaseStatus.COMPLETED)`
- Assert review case contexts contain `coordinatedChange: true`
- Assert merge-executor was NOT called on review cases (flag suppressed per-repo merge)

**Phase 6 — EventLog verification:**
- `caseHubRuntime.eventLog(parentCaseId)` → list of `CaseEventLogRecord`
- Filter by `SIGNAL_RECEIVED` — assert entries exist with `payload` containing `completedReviews` paths for both repos
- Filter by `SIGNAL_RECEIVED` — assert entry with `payload` containing `allReviewsCompleted`
- Filter by `WORKER_EXECUTION_COMPLETED` — assert merge worker completion entry
- Filter by `GOAL_REACHED` — assert `all-repos-merged` goal reached
- Cross-case provenance linking (`causedByEntryId`) is not yet supported by the engine API — tracked as #163

### Scenario 2: Review faults — parent terminates, remaining cancelled

**Setup:** 2-repo request. No MergeClient outcomes (merge never reached).

**Phase 1:** Same as scenario 1.

**Phase 2 — Divergent lifecycle:**
- `driveReviewToCompletion(reviewCaseIdA)` — engine passes
- `driveReviewToFault(reviewCaseIdB)` — platform cancelled

**Phase 3 — Fault propagation:**
- Await parent context `reviewFaulted.repo` == `"casehubio/platform"`
- Await parent context `reviewFaulted.reason` == `"CANCELLED"`

**Phase 4 — Terminal + cancel propagation:**
- Await parent case terminal state — the `review-faulted` failure goal is satisfied, triggering `failure: anyOf: [review-faulted]`. Assert terminal (not a specific status — the engine may produce COMPLETED-with-failure-metadata or FAULTED; the test discovers which)
- Assert engine review case (reviewCaseIdA) stays COMPLETED — `cancelCase()` throws `IllegalStateException` on terminal cases (caught by observer's try/catch), so reviewCaseIdA is unchanged
- Assert `testMergeClient.calls` empty

**Phase 5 — EventLog verification:**
- `caseHubRuntime.eventLog(parentCaseId)` → list of `CaseEventLogRecord`
- Filter by `SIGNAL_RECEIVED` — assert entry with `payload` containing `reviewFaulted` with `repo: "casehubio/platform"` and `reason: "CANCELLED"`
- Filter by `GOAL_REACHED` — assert `review-faulted` goal reached
- Cross-case provenance linking not yet available (#163)

### Scenario 3: Rollback failure with human escalation

**Setup:** 3-repo request (engine, platform, work). TestMergeClient: engine Success, platform Failure. TestRevertClient: engine MergeConflict.

**Phases 1-3:** All 3 reviews complete, `allReviewsCompleted` signaled.

**Phase 4a — Merge:**
- Await `mergeResults` non-null
- Assert 2 entries: engine success, platform failed (work never attempted — stop-on-failure)
- Assert `testMergeClient.calls` size 2

**Phase 4b — Rollback:**
- Await `rollbackResults` non-null
- Assert `testRevertClient.calls` size 1 (only engine was merged)
- Assert `rollbackResults[0].status` == `"conflict"`

**Phase 4c — Human escalation:**
- Await WorkItem via `workItemQueries.scanAll()` — title "Coordinated rollback failed — manual revert required"
- Assert candidateGroups contains `human-oversight:general`
- Parent case ACTIVE (waiting for human)

### Scenario 4: Out-of-order completion

**Setup:** 2-repo request. Same MergeClient as scenario 1.

**Phase 1:** Same.

**Phase 2 — Reverse order:**
- `driveReviewToCompletion(reviewCaseIdB)` — platform first
- Await `completedReviews.casehubio/platform` present
- Assert `allReviewsCompleted` NOT yet true
- `driveReviewToCompletion(reviewCaseIdA)` — engine second

**Phases 3-5:** Identical to scenario 1 — same terminal state, same merge results.

### Scenario 5: Idempotent guard — rollback binding doesn't re-fire

**Setup:** 2-repo request. TestMergeClient: engine Success, platform Failure. TestRevertClient: engine Success.

**Phases 1-3:** All reviews complete, `allReviewsCompleted` signaled.

**Phase 4a — Merge + rollback:**
- Await `mergeResults` (engine success, platform failed)
- Await `rollbackResults` (engine reverted)
- Assert `testRevertClient.calls` size 1

**Phase 4b — Provoke re-evaluation:**
- Signal parent: `caseHubRuntime.signal(parentCaseId, "probe", "test")`
- Context change → engine re-evaluates bindings
- `rollback-on-merge-failure` condition `.rollbackResults == null` → FALSE → guard holds
- Awaitility `during(2, SECONDS)` — assert `testRevertClient.calls` remains size 1

**Phase 5:** Parent reaches terminal state (merge-failed failure goal).

## Protocols Applied

- **PP-20260521-134c38** (HITL test pre-seeding): Applied — `preSeedCapabilityKeys()` signals non-null PENDING values for all capability keys immediately after review case creation, preventing reroute/escalation binding churn.
- **failure-cascade-pattern.md** (failure cascade pattern): Informed the semantic gap observation — failure goals produce COMPLETED, not FAULTED. The observer COMPLETED semantics gap is tracked as #161.
- Each test uses unique case IDs via engine-generated UUIDs. Tracker accumulation across tests is safe because assertions filter by parent case ID.

## Known Issue: Observer COMPLETED semantics

`CoordinatedChangeObserver` classifies `CaseStatus.COMPLETED` as `TERMINAL_SUCCESS`. But per the failure-cascade-pattern protocol, failure goals also produce COMPLETED. A review case that hits `review-abandoned` (PR closed) would be misclassified as successfully completed, causing the coordinator to merge a closed PR.

This test uses `cancelCase()` to sidestep the gap. Filed as #161 — fix options include checking the completion outcome type in `CaseLifecycleEvent`, or having the engine produce a distinct status for failure-goal completion.

## Not In Scope

- REST endpoint or MCP tool for triggering coordinated changes
- Testing the `CoordinatedChangeTrackerHydrator` restart recovery path (tracked as #162)
- Webhook-driven review lifecycle (already covered by existing webhook handler tests)
