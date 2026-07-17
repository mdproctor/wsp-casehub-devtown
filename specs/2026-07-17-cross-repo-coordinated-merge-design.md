# Cross-Repo Coordinated Merge Design

**Issues:** #156, #157, #159 (Epic #12)
**Date:** 2026-07-17
**Branch:** issue-156-cross-repo-coordinated-merge

## Problem

A cross-repo change set — multiple PRs across multiple repositories that must all be merged together or none at all. The engine's sub-case model handles static sub-case counts (e.g., bisect-left/right = exactly 2). Cross-repo coordination needs N sub-cases where N is determined at runtime by the number of repositories in the change set.

## Design Decision

Per-repo reviews are standard pr-review cases started through `PrReviewCaseService.startReview()`. They are NOT engine sub-cases. The coordination between them is pure application logic.

**Why not engine sub-cases:**
- `SubCaseExecutionHandler` is the only path that writes `SUBCASE_STARTED` EventLog entries, creates `SubCaseGroup` records, marks PlanItems DELEGATED, and sets parent WAITING state. Calling `CaseHubRuntime.startCase()` with `parentCaseId` bypasses all of this.
- `SubCaseCompletionService.handleCompletion()` searches for `SUBCASE_STARTED` EventLog entries to find the parent. Without them, child completion does not propagate.
- The cross-repo knowledge (which repos, how to correlate, what constitutes "all merged") is software-engineering domain logic that belongs in devtown, not in generic engine primitives.

**Why standard pr-review cases:**
- Each review IS a standard pr-review case — same `pr-review.yaml` CasePlanModel (at `review/src/main/resources/devtown/pr-review.yaml`), same review workers, same webhook routing. Coordinated reviews suppress per-repo auto-merge via a `coordinatedChange` context flag (see §pr-review.yaml Modifications for Coordinated Mode).
- `PrReviewCaseService.startReview()` calls `caseTracker.register()`, so each review case is registered with `PrReviewCaseTracker`.
- `GitHubWebhookResource` injects `PrReviewApplicationService` and calls `service.signalCiStatus()`. Inside `PrReviewCaseService` (the implementation), `signalCiStatus()` queries `PrReviewCaseTracker.findActiveCaseByPr(repo, prNumber)` to resolve the case. Per-repo webhook routing works unchanged — no handler modifications needed.
- No engine changes. No coupling to engine internals. No webhook handler changes.

**pr-review.yaml modifications:** The YAML is extended with a `coordinatedChange` context guard on merge bindings and an extended `merge-completed` goal condition. These are additive changes — standalone reviews (where `coordinatedChange` is absent) are unaffected. See §pr-review.yaml Modifications for Coordinated Mode.

**Divergence from epic #12:** Epic #12 and issues #156, #159 were written assuming engine sub-case primitives. This spec chooses application-layer coordination instead, for the reasons above. The epic, #156, and #159 descriptions must be updated to reflect this design decision before implementation begins. `docs/orchestration-advantages.md` §4 uses sub-case bindings as an illustrative example of cross-repo coordination — the example shows what is *possible*, not what is *prescribed*. The application-layer approach achieves the same outcomes (atomic merge-all-or-rollback, auditable coordination decisions, automatic failure propagation) through different mechanisms.

## Coordination Flow

```
1. Trigger → CoordinatedChangeService.start(request)
1b. Pre-check: for each repo, verify no active review case exists (standalone or coordinated)
   - PrReviewCaseTracker.findActiveCaseByPr(repo, prNumber) must return empty
   - Reject entire request if any repo has an active review (see §Edge Cases)
2. Start parent case via CoordinatedChangeCaseHub.startCase(context)
   - Parent case has repos list, no active bindings yet
3. For each repo in request (all-or-none — see §Partial Failure Atomicity):
   - PrReviewCaseService.startReview(prPayload, Map.of("coordinatedChange", true)) → PrReviewOutcome(caseId)
   - CoordinatedChangeTracker.register(parentCaseId, repo, outcome.caseId())
   On failure after any reviews started: cancel all started reviews + parent case
3b. Signal parent with review case mapping:
   - caseHubRuntime.signal(parentCaseId, "reviewCases", {repo → reviewCaseId})
   - Persists mapping to parent case context (enables hydration on restart)
4. Each repo's PR goes through standard pr-review lifecycle (merge suppressed)
   - Code analysis, security review, CI, human gates — all existing
   - Merge bindings suppressed by `coordinatedChange` flag in context
   - Review case completes via extended `merge-completed` goal when all reviews pass
5. Webhooks arrive per-repo
   - Existing GitHubWebhookResource routes correctly via PrReviewCaseTracker
   - No webhook handler changes needed
6. Review case completes → CaseLifecycleEvent
   - CoordinatedChangeObserver detects it's tracked
   - Signals parent context: completedReviews[repo] = {status: "completed", reviewCaseId: "<uuid>"}
7. ALL reviews complete (atomic transition — see §Race Condition Prevention)
   - Observer signals parent: allReviewsCompleted = true
   - Parent YAML binding merge-all-repos fires
   - Coordinated-merge worker merges all repos sequentially
8. ANY review faults
   - Observer signals parent: reviewFaulted = {repo, reason}
   - Parent failure goal fires immediately
   - Cancel propagation stops remaining reviews
```

## CasePlanModel YAML

File: `app/src/main/resources/casehub/devtown/coordinated-change.yaml`

```yaml
dsl: "0.1"
version: "1.0.0"
name: coordinated-change
namespace: devtown
title: Cross-repo coordinated change — merge all or rollback on fault

spec:
  capabilities:
    - name: coordinated-merge
      description: "Merges all repos in the change set sequentially"
      inputProjection: "{ repos: .repos }"
      outputProjection: "{ mergeResults: .mergeResults }"

    - name: coordinated-rollback
      description: "Reverts already-merged repos on failure"
      inputProjection: "{ repos: .repos, mergeResults: .mergeResults }"
      outputProjection: "{ rollbackResults: .rollbackResults }"

  goals:
    - name: all-repos-merged
      kind: success
      condition: >-
        .mergeResults != null and
        (.mergeResults | length > 0) and
        (.mergeResults | all(.status == "success"))

    - name: review-faulted
      kind: failure
      condition: '.reviewFaulted != null'

    - name: merge-failed
      kind: failure
      condition: >-
        .mergeResults != null and
        (.mergeResults | any(.status == "failed"))

    - name: coordination-abandoned
      kind: failure
      condition: '.abandoned == true'

  completion:
    success:
      allOf: [all-repos-merged]
    failure:
      anyOf: [review-faulted, merge-failed, coordination-abandoned]

  bindings:
    - name: merge-all-repos
      on: { contextChange: {} }
      when: '.allReviewsCompleted == true and .mergeResults == null'
      capability: coordinated-merge
      outcomePolicy:
        onDecline: FAULT
        onFailure: FAULT

    - name: rollback-on-merge-failure
      on: { contextChange: {} }
      when: >-
        .mergeResults != null and
        (.mergeResults | any(.status == "failed")) and
        .rollbackResults == null
      capability: coordinated-rollback
      outcomePolicy:
        onDecline: FAULT
        onFailure: FAULT
```

The `coordinated-rollback` capability is declared but the worker is NOT implemented in this batch (#158). The binding exists so the YAML is structurally complete.

## pr-review.yaml Modifications for Coordinated Mode

Coordinated reviews use the same `pr-review.yaml` lifecycle but must not auto-merge — the coordinator handles all merges atomically. Three additive changes to `pr-review.yaml` achieve this:

**1. Guard merge bindings with `coordinatedChange` flag:**

Both `merge-direct` and `enqueue-for-merge` bindings gain `.coordinatedChange != true` in their `when` conditions. When `coordinatedChange` is absent (standalone reviews), JQ evaluates `null != true` → `true`, so standalone behavior is unchanged. When `coordinatedChange` is `true` (coordinated reviews), the guard evaluates to `false` and the binding does not fire.

```yaml
# merge-direct — add guard (shown as first condition)
when: >-
  .coordinatedChange != true and
  .merge_sha == null and
  .pr.status != "merged" and
  ...existing conditions...

# enqueue-for-merge — same guard
when: >-
  .coordinatedChange != true and
  .enqueueResult == null and
  .merge_sha == null and
  .pr.status != "merged" and
  ...existing conditions...
```

**2. Extend `merge-completed` goal for coordinated reviews:**

The `merge-completed` goal gains `.coordinatedChange == true` as an alternative. For coordinated reviews, the merge obligation is delegated to the parent coordinator — the review case considers it satisfied. The completion block (`allOf: [pr-approved, security-verified, ci-passing, merge-completed]`) remains unchanged; all four goals must be true. For coordinated reviews, `merge-completed` fires via the flag, while `pr-approved`, `security-verified`, and `ci-passing` still require all reviews to pass and CI to pass before the case reaches terminal SUCCESS.

```yaml
- name: merge-completed
  kind: success
  condition: '.coordinatedChange == true or .pr.status == "merged" or .merge_sha != null'
```

**3. Inject the flag via `startReview` additional context:**

`CoordinatedChangeService.start()` calls `reviewService.startReview(prPayload, Map.of("coordinatedChange", true))`. The `coordinatedChange` flag is merged into `initialContext` at case creation, so it is available from the first context evaluation — no race condition with binding evaluation.

**End-to-end coordinated path:**

1. `startReview(pr, Map.of("coordinatedChange", true))` creates review case with flag in context
2. Reviews proceed: code analysis, security, CI, human gates — all standard bindings fire normally
3. All reviews pass and CI passes → `pr-approved`, `security-verified`, `ci-passing` become true
4. `merge-completed` is already true (via `.coordinatedChange == true`)
5. Merge bindings do NOT fire (guarded by `.coordinatedChange != true`)
6. Completion `allOf` is satisfied → review case reaches terminal SUCCESS
7. `CaseLifecycleEvent(SUCCESS)` → observer signals parent → coordinator merges all repos

## Components

### Domain types (`domain/`)

**`CoordinatedChangeRequest`** — input record:
```java
public record CoordinatedChangeRequest(List<RepoChangeEntry> repos) {}

public record RepoChangeEntry(
    String owner, String repo, int prNumber,
    String headSha, String targetBranch, String contributor,
    List<String> changedPaths, int linesChanged) {}
```

**`CoordinatedMergeResult`** — per-repo merge outcome:
```java
public sealed interface CoordinatedMergeResult {
    String repo();
    record Success(String repo, String mergeSha) implements CoordinatedMergeResult {}
    record Failure(String repo, String reason) implements CoordinatedMergeResult {}
}
```

**`AgentQualification.COORDINATED_MERGE`** — new capability tag constant.

### Port interface (`review/`)

**`CoordinatedChangePort`** — alongside `PrReviewApplicationService`:
```java
public interface CoordinatedChangePort {
    CoordinatedChangeOutcome start(CoordinatedChangeRequest request);
}
```

**`CoordinatedChangeOutcome`** — result of starting a coordinated change:
```java
public record CoordinatedChangeOutcome(UUID parentCaseId, Map<String, UUID> reviewCaseIds) {}
```

**API change: `PrReviewApplicationService`** — add `startReview` overload with additional context:
```java
public interface PrReviewApplicationService {
    PrReviewOutcome startReview(PrPayload pr);
    PrReviewOutcome startReview(PrPayload pr, Map<String, Object> additionalContext);
    // ...existing methods unchanged...
}
```
The single-argument method delegates to the two-argument method with `Map.of()`. In `PrReviewCaseService`, the implementation merges `additionalContext` into `initialContext` before calling `caseHub.startCase()`:
```java
initialContext.putAll(additionalContext);
UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().join();
```
This is a general-purpose extension — `PrReviewApplicationService` has no knowledge of coordinated changes. The coordination flag is an opaque context entry that `pr-review.yaml` conditions evaluate.

**API change: `PrReviewOutcome`** — add `caseId` field:
```java
public record PrReviewOutcome(String verdict, List<String> findings, UUID caseId) {}
```
Currently `PrReviewOutcome(String verdict, List<String> findings)` does not expose the case identifier. A caller that starts a review should know what case was created. `PrReviewCaseService.startReview()` generates the `caseId` at line 107 but discards it in the return value — this is a design gap in the existing API, not specific to coordinated changes. The `caseId` field is `null` when no case was created (e.g., if the method were to take a revise-only path, though the coordinated flow's pre-check prevents this — see §Edge Cases).

### Application layer (`app/`)

**`CoordinatedChangeCaseHub`** — extends `YamlCaseHub`:
```java
@ApplicationScoped
public class CoordinatedChangeCaseHub extends YamlCaseHub {
    @Inject MergeClient mergeClient;

    public CoordinatedChangeCaseHub() {
        super("casehub/devtown/coordinated-change.yaml");
    }

    @Override
    protected void augment(CaseDefinition definition) {
        definition.getWorkers().add(Worker.builder()
            .name("coordinated-merge")
            .capabilityName("coordinated-merge")
            .function(this::adaptCoordinatedMerge)
            .build());
    }

    WorkerResult adaptCoordinatedMerge(Map<String, Object> input) {
        // Sequential merge, stop on first failure
        // Returns {mergeResults: [{repo, status, mergeSha?}...]}
    }
}
```

**`CoordinatedChangeService`** — implements `CoordinatedChangePort`:
```java
@ApplicationScoped
public class CoordinatedChangeService implements CoordinatedChangePort {
    @Inject CoordinatedChangeCaseHub caseHub;
    @Inject PrReviewApplicationService reviewService;
    @Inject CoordinatedChangeTracker tracker;
    @Inject PrReviewCaseTracker prReviewCaseTracker;
    @Inject CaseHubRuntime caseHubRuntime;

    public CoordinatedChangeOutcome start(CoordinatedChangeRequest request) {
        // 0. Pre-check: reject if any repo has an active review case (standalone or coordinated)
        //    for each repo in request:
        //        Optional<CaseInfo> active = prReviewCaseTracker.findActiveCaseByPr(repo, prNumber)
        //        if active.isPresent():
        //            if tracker.isPartOfCoordinatedChange(active.get().caseId()):
        //                throw ConflictingCoordinatedChangeException(repo, prNumber)
        //            else:
        //                throw ActiveReviewExistsException(repo, prNumber, active.get().caseId())
        // 1. Build initial context with repos list
        // 2. Start parent case via caseHub.startCase(context) → parentCaseId
        // 3. Start all review cases (all-or-none):
        //    Map<String, UUID> started = new LinkedHashMap<>();
        //    try {
        //        for each repo in request:
        //            PrReviewOutcome outcome = reviewService.startReview(prPayload, Map.of("coordinatedChange", true))
        //            UUID reviewCaseId = outcome.caseId()
        //            tracker.register(parentCaseId, repo, reviewCaseId)
        //            started.put(repo, reviewCaseId)
        //    } catch (Exception e) {
        //        started.values().forEach(caseHubRuntime::cancelCase);
        //        caseHubRuntime.cancelCase(parentCaseId);
        //        throw new CoordinatedChangeStartFailedException(request, e);
        //    }
        // 4. Signal parent with mapping for hydration:
        //    caseHubRuntime.signal(parentCaseId, "reviewCases", started)
        // 5. Return parentCaseId + started map
    }
}
```

**`CoordinatedChangeTracker`** — `@ApplicationScoped`, in-memory mapping with atomic completion transition:
```java
@ApplicationScoped
public class CoordinatedChangeTracker {
    // parentCaseId → CoordinationState
    // CoordinationState: {repos: Map<String, UUID>, completedRepos: Set<String>,
    //                     allCompletedFired: AtomicBoolean, parentTerminal: AtomicBoolean}

    void register(UUID parentCaseId, String repo, UUID reviewCaseId);
    Entry findByReviewCaseId(UUID reviewCaseId);
    Set<UUID> findReviewCaseIds(UUID parentCaseId);
    boolean markCompleted(UUID parentCaseId, String repo);
    boolean tryTransitionToAllCompleted(UUID parentCaseId);
    boolean isPartOfCoordinatedChange(UUID reviewCaseId);
    void markParentTerminal(UUID parentCaseId);
    boolean isParentTerminal(UUID parentCaseId);
}
```

**Race condition prevention:** `tryTransitionToAllCompleted()` uses `AtomicBoolean.compareAndSet(false, true)` — exactly one concurrent caller succeeds when the last review completes. This prevents duplicate `allReviewsCompleted` signals to the parent case. The method returns `true` only for the thread that wins the CAS.

**Cancel feedback loop prevention:** When the parent case reaches terminal state, `onParentTerminal` cancels remaining review cases. Each cancellation fires a `CaseLifecycleEvent` with `caseStatus=CANCELLED`. Without a guard, `onCaseLifecycle` would try to signal `reviewFaulted` to the already-terminal parent — producing error noise for every cancelled review. The `parentTerminal` flag on `CoordinationState` prevents this: `onParentTerminal` calls `tracker.markParentTerminal(parentCaseId)` before cancelling reviews, and `onCaseLifecycle` checks `tracker.isParentTerminal()` before signaling. This is preferred over filtering all CANCELLED events because external cancellation of a review (not coordinator-initiated) should still propagate to the parent when it is non-terminal.

**`CoordinatedChangeTrackerHydrator`** — rebuilds tracker state on startup and replays missed completion signals:
```java
@ApplicationScoped
public class CoordinatedChangeTrackerHydrator {
    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject CoordinatedChangeTracker tracker;
    @Inject CaseHubRuntime caseHubRuntime;
    @Inject CurrentPrincipal principal;

    void onStartup(@Observes StartupEvent event) {
        // Phase 1: Rebuild tracker state
        // Query non-terminal cases with caseDefinitionName = "coordinated-change"
        // For each parent case:
        //   - Extract repos from initial context
        //   - Extract reviewCaseIds from "reviewCases" context key (written by step 3b)
        //   - Re-register with tracker

        // Phase 2: Reconcile review case status and replay missed signals
        // For each review case in the tracker:
        //   - Query CaseInstance via caseInstanceRepository.findByUuid(reviewCaseId, tenancyId)
        //   - If terminal+success AND not in parent context "completedReviews":
        //       tracker.markCompleted(parentCaseId, repo)
        //       caseHubRuntime.signal(parentCaseId, "completedReviews." + repo, {status, reviewCaseId})
        //   - If terminal+failure AND parent context "reviewFaulted" is null:
        //       caseHubRuntime.signal(parentCaseId, "reviewFaulted", {repo, reason})
        // After all reviews reconciled:
        //   - If tracker.tryTransitionToAllCompleted(parentCaseId):
        //       caseHubRuntime.signal(parentCaseId, "allReviewsCompleted", true)
    }
}
```

Hydration source is `CaseInstanceRepository.findByStatus()` filtered by `caseDefinitionName`, extracting the `reviewCases` mapping from the parent case context (written by the signal in step 4 of `CoordinatedChangeService.start()`). Phase 1 follows the `PrReviewCaseTrackerHydrator` pattern. Phase 2 goes beyond it: `PrReviewCaseTrackerHydrator` only rebuilds a routing index for webhooks — missing a webhook during downtime means a stale CI status, not a stuck case. The coordinated change hydrator drives state machine transitions, so it must detect review cases that completed during downtime and replay the corresponding signals to the parent case context. Without this, a review completing during restart would leave the parent waiting indefinitely.

**`CoordinatedChangeObserver`** — `CaseLifecycleEvent` observer:
```java
@ApplicationScoped
public class CoordinatedChangeObserver {
    @Inject CoordinatedChangeTracker tracker;
    @Inject CaseHubRuntime caseHubRuntime;

    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        // 1. Check if event.caseId() is tracked as a per-repo review
        //    Entry entry = tracker.findByReviewCaseId(event.caseId());
        //    if (entry == null) return;
        // 2. If parent already terminal, skip — avoids feedback loop when
        //    coordinator-initiated cancellations fire CaseLifecycleEvents
        //    back to the observer (see §Cancel Feedback Loop Prevention)
        //    if (tracker.isParentTerminal(entry.parentCaseId())) return;
        // 3. If terminal + success: mark completed, signal parent context
        // 4. If terminal + failure/cancelled: signal parent reviewFaulted
        // 5. If all completed: signal parent allReviewsCompleted = true
    }

    void onParentTerminal(@ObservesAsync CaseLifecycleEvent event) {
        // If the parent case reaches terminal state (FAULTED, COMPLETED, CANCELLED):
        // 1. Mark parent terminal in tracker BEFORE cancelling reviews:
        //    tracker.markParentTerminal(event.caseId())
        //    This ensures subsequent CANCELLED events from child reviews are
        //    filtered by onCaseLifecycle's parent-terminal guard.
        // 2. Cancel remaining non-terminal reviews via CaseHubRuntime.cancelCase(reviewCaseId).
        //    This transitions each review case to CANCELLED, which fires CaseLifecycleEvent
        //    with caseStatus=CANCELLED. The pr-review.yaml's review-abandoned goal
        //    (condition: '.pr.status == "closed" or .pr.status == "superseded"') does NOT
        //    fire from engine cancellation — the case is already terminal.
        // GitHub-side cleanup (removing status checks, posting cancel comments) is handled
        // by a separate CaseLifecycleEvent observer in the github module, not here.
    }
}
```

## EventLog Audit Trail

Coordination decisions are auditable through the engine's existing `SIGNAL_RECEIVED` EventLog entries — no custom event types or engine changes required. Every call to `caseHubRuntime.signal()` writes a `SIGNAL_RECEIVED` entry with the signal path and value as payload. The coordination signals produce these entries on the parent case's EventLog:

1. **Review case mapping** — `signal(parentCaseId, "reviewCases", {repo → reviewCaseId})` records which review cases belong to this coordinated change.

2. **Review completion** — `signal(parentCaseId, "completedReviews.{repo}", {status, reviewCaseId})` records each repo's review outcome.

3. **All reviews completed** — `signal(parentCaseId, "allReviewsCompleted", true)` records the completion threshold transition.

4. **Review faulted** — `signal(parentCaseId, "reviewFaulted", {repo, reason})` records which review faulted and why.

Each review case's own EventLog records its internal lifecycle (code analysis, CI, human gates) through the standard `pr-review.yaml` bindings. The parent case EventLog records coordination decisions. Together they provide a complete audit trail visible to `CaseHistoryResource`, with cross-references via `reviewCaseId` values in signal payloads.

## Coordinated-merge Worker (#157)

The worker receives `{repos: [...]}` from parent case context.

```
For each repo in repos (sequential, in input order):
  1. Extract owner, repo, prNumber, headSha from context
  2. MergeClient.merge(owner, repo, prNumber, headSha)
  3. Record: {repo: "owner/repo", status: "success", mergeSha: "abc123"}
     or:     {repo: "owner/repo", status: "failed", reason: "merge conflict"}
  4. On failure: STOP — do not merge remaining repos
Return: WorkerResult.of({mergeResults: [...]})
  // outputProjection "{ mergeResults: .mergeResults }" extracts the array
  // from worker output and stores it as context.mergeResults = [...]
```

Stop-on-failure is deliberate: already-merged repos stay merged; the `rollback-on-merge-failure` binding fires when the failure goal matches, and the rollback worker (#158) handles revert.

**Dependency on #155:** `MergeClient` already exists at `io.casehub.devtown.domain.MergeClient` with implementation `GitHubMergeClient` in the github module. Issue #155 has been rescoped to `RevertClient` only (for the rollback worker #158). The merge worker (#157) has **no dependency on #155** — it uses the existing `MergeClient` directly.

## Webhook Routing (#159)

**No changes to `GitHubWebhookResource`.** The existing handler routes `check_suite`/`check_run` events via `PrReviewCaseTracker.findActiveCaseByPr(repo, prNumber)`. Since per-repo reviews are standard pr-review cases registered with the tracker, webhook routing works unchanged.

#159 scope reduces to:
1. `CoordinatedChangeTracker` — maps parentCaseId → {repo → reviewCaseId}
2. `CoordinatedChangeObserver` — lifecycle event observer, signals parent on review completion/fault
3. Cancel propagation — stops remaining reviews when parent terminates

## Initial Context Structure

When the parent case starts, context contains:

```json
{
  "repos": [
    {
      "owner": "casehubio",
      "repo": "engine",
      "prNumber": 42,
      "headSha": "abc123",
      "targetBranch": "main"
    },
    {
      "owner": "casehubio",
      "repo": "platform",
      "prNumber": 99,
      "headSha": "def456",
      "targetBranch": "main"
    }
  ]
}
```

The observer signals additional context as reviews progress:

```json
{
  "completedReviews": {
    "casehubio/engine": {"status": "completed"},
    "casehubio/platform": {"status": "completed"}
  },
  "allReviewsCompleted": true
}
```

Or on fault:

```json
{
  "reviewFaulted": {
    "repo": "casehubio/platform",
    "reason": "FAULTED"
  }
}
```

## Edge Cases

**Existing active review for a repo:** `CoordinatedChangeService.start()` pre-checks each repo via `PrReviewCaseTracker.findActiveCaseByPr(repo, prNumber)` before starting any cases. If any PR has an active review case of any kind, the request is **rejected**:
- **Coordinated review** (tracked by `CoordinatedChangeTracker.isPartOfCoordinatedChange()`): throws `ConflictingCoordinatedChangeException`. Partial overlap between coordinated sets would create ambiguous completion semantics.
- **Standalone review**: throws `ActiveReviewExistsException`. Without this check, `PrReviewCaseService.startReview()` would call `revisePr()` on the existing case instead of creating a new one — the coordinator would then track a pre-existing standalone review at an arbitrary lifecycle stage, with no guarantee its completion semantics align with coordination requirements. The caller must cancel or wait for the existing review before initiating a coordinated change.

**Out-of-order completion:** Reviews complete independently. Observer counts completions. Order does not matter.

**One review faults, others still running:** Observer signals `reviewFaulted` immediately. Parent failure goal fires. `onParentTerminal` handler cancels remaining reviews.

**Merge order:** Sequential, in the order repos appear in the request. Caller controls ordering by arranging the repos list.

**Partial failure during start:** If `PrReviewCaseService.startReview()` fails for one repo after others have already started, `CoordinatedChangeService.start()` cancels all already-started review cases and the parent case via `caseHubRuntime.cancelCase()`, then throws `CoordinatedChangeStartFailedException`. This ensures atomicity: either all review cases are created, or none remain active. Cleanup failures during cancellation are logged but do not suppress the original exception.

**Partial merge failure:** Repos A and B merge successfully, repo C fails. The worker returns results for all three. The `merge-failed` goal fires. The `rollback-on-merge-failure` binding fires (worker implemented in #158, not this batch).

## Risk: Duplicate binding dispatch (ChoreographyLoopControl)

`CaseContextChangedEventHandler` correctly evaluates `when:` conditions — bindings whose conditions evaluate to false are excluded (verified from engine source, lines 221–223). The garden entry GE-20260521-9188c1's concern about unconditional firing is disproven.

The **actual** risk is `ChoreographyLoopControl` (the default `LoopControl` implementation), whose Javadoc states: "Pure choreography: every rule whose trigger condition matched is scheduled for execution without deliberate ordering or prioritisation. ... no dedup mechanism exists in this path." This means: if a binding's `when` condition remains true across consecutive context changes, the binding fires each time. The `merge-all-repos` binding's `when: '.allReviewsCompleted == true and .mergeResults == null'` prevents re-dispatch AFTER the worker writes results (`mergeResults` becomes non-null), but during worker execution — before results are written — a stray context change would cause duplicate dispatch.

**Practical impact:** Low. No context changes are expected between `allReviewsCompleted` being set and `mergeResults` being written (the merge worker runs synchronously in the binding's worker execution). Mitigation: guard inside worker functions with early-return on unmet preconditions (idempotency check).

## Not In Scope

- `coordinated-rollback` worker implementation (#158)
- REST endpoint or MCP tool for triggering coordinated changes (follows naturally from the port interface)
- End-to-end integration test (#160)
- Cross-repo integration CI (CI that spans multiple repos — not a current requirement)
