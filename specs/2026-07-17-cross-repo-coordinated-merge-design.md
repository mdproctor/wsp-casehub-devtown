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
- Each review IS a standard pr-review case — same lifecycle, same `pr-review.yaml` CasePlanModel (at `review/src/main/resources/devtown/pr-review.yaml`), same workers, same webhook routing.
- `PrReviewCaseService.startReview()` calls `caseTracker.register()`, so each review case is registered with `PrReviewCaseTracker`.
- `GitHubWebhookResource` injects `PrReviewApplicationService` and calls `service.signalCiStatus()`. Inside `PrReviewCaseService` (the implementation), `signalCiStatus()` queries `PrReviewCaseTracker.findActiveCaseByPr(repo, prNumber)` to resolve the case. Per-repo webhook routing works unchanged — no handler modifications needed.
- No engine changes. No coupling to engine internals.

**Divergence from epic #12:** Epic #12 and issues #156, #159 were written assuming engine sub-case primitives. This spec chooses application-layer coordination instead, for the reasons above. The epic, #156, and #159 descriptions must be updated to reflect this design decision before implementation begins. `docs/orchestration-advantages.md` §4 uses sub-case bindings as an illustrative example of cross-repo coordination — the example shows what is *possible*, not what is *prescribed*. The application-layer approach achieves the same outcomes (atomic merge-all-or-rollback, auditable coordination decisions, automatic failure propagation) through different mechanisms.

## Coordination Flow

```
1. Trigger → CoordinatedChangeService.start(request)
2. Start parent case via CoordinatedChangeCaseHub.startCase(context)
   - Parent case has repos list, no active bindings yet
3. For each repo in request (all-or-none — see §Partial Failure Atomicity):
   - PrReviewCaseService.startReview(prPayload) → reviewCaseId
   - CoordinatedChangeTracker.register(parentCaseId, repo, reviewCaseId)
   On failure after any reviews started: cancel all started reviews + parent case
3b. Signal parent with review case mapping:
   - caseHubRuntime.signal(parentCaseId, "reviewCases", {repo → reviewCaseId})
   - Persists mapping to parent case context (enables hydration on restart)
4. Each repo's PR goes through standard pr-review lifecycle
   - Code analysis, security review, CI, human gates — all existing
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
      outputProjection: "{ mergeResults: . }"

    - name: coordinated-rollback
      description: "Reverts already-merged repos on failure"
      inputProjection: "{ repos: .repos, mergeResults: .mergeResults }"
      outputProjection: "{ rollbackResults: . }"

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
    @Inject CaseHubRuntime caseHubRuntime;

    public CoordinatedChangeOutcome start(CoordinatedChangeRequest request) {
        // 1. Build initial context with repos list
        // 2. Start parent case via caseHub.startCase(context) → parentCaseId
        // 3. Start all review cases (all-or-none):
        //    Map<String, UUID> started = new LinkedHashMap<>();
        //    try {
        //        for each repo in request:
        //            UUID reviewCaseId = reviewService.startReview(prPayload)
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
    // CoordinationState: {repos: Map<String, UUID>, completedRepos: Set<String>, allCompletedFired: AtomicBoolean}

    void register(UUID parentCaseId, String repo, UUID reviewCaseId);
    Entry findByReviewCaseId(UUID reviewCaseId);
    Set<UUID> findReviewCaseIds(UUID parentCaseId);
    boolean markCompleted(UUID parentCaseId, String repo);
    boolean tryTransitionToAllCompleted(UUID parentCaseId);
    boolean isPartOfCoordinatedChange(UUID reviewCaseId);
}
```

**Race condition prevention:** `tryTransitionToAllCompleted()` uses `AtomicBoolean.compareAndSet(false, true)` — exactly one concurrent caller succeeds when the last review completes. This prevents duplicate `allReviewsCompleted` signals to the parent case. The method returns `true` only for the thread that wins the CAS.

**`CoordinatedChangeTrackerHydrator`** — rebuilds tracker state on startup, following the `PrReviewCaseTrackerHydrator` pattern:
```java
@ApplicationScoped
public class CoordinatedChangeTrackerHydrator {
    @Inject CaseInstanceRepository caseInstanceRepository;
    @Inject CoordinatedChangeTracker tracker;
    @Inject CurrentPrincipal principal;

    void onStartup(@Observes StartupEvent event) {
        // Query active cases with caseDefinitionName = "coordinated-change"
        // For each: extract repos from initial context,
        //           reviewCaseIds from "reviewCases" context key (written by step 4 of start())
        // Re-register with tracker, mark already-completed reviews
    }
}
```

Hydration source is `CaseInstanceRepository.findByStatus()` filtered by `caseDefinitionName`, extracting the `reviewCases` mapping from the parent case context (written by the signal in step 4 of `CoordinatedChangeService.start()`) — the same pattern used by `PrReviewCaseTrackerHydrator`. No EventLog queries required.
```

**`CoordinatedChangeObserver`** — `CaseLifecycleEvent` observer:
```java
@ApplicationScoped
public class CoordinatedChangeObserver {
    @Inject CoordinatedChangeTracker tracker;
    @Inject CaseHubRuntime caseHubRuntime;

    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        // 1. Check if event.caseId() is tracked as a per-repo review
        // 2. If terminal + success: mark completed, signal parent context
        // 3. If terminal + failure: signal parent reviewFaulted
        // 4. If all completed: signal parent allReviewsCompleted = true
    }

    void onParentTerminal(@ObservesAsync CaseLifecycleEvent event) {
        // If the parent case reaches terminal state (FAULTED, COMPLETED, CANCELLED),
        // cancel remaining non-terminal reviews via CaseHubRuntime.cancelCase(reviewCaseId).
        // This transitions each review case to CANCELLED, which fires CaseLifecycleEvent
        // with caseStatus=CANCELLED. The pr-review.yaml's review-abandoned goal
        // (condition: '.pr.status == "closed" or .pr.status == "superseded"') does NOT
        // fire from engine cancellation — the case is already terminal.
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

**Existing active review for a repo:** `PrReviewCaseService.startReview()` checks for an existing case on the same repo+prNumber. If found, it calls `revisePr()` instead of creating a new case. `CoordinatedChangeService.start()` checks each repo against `CoordinatedChangeTracker.isPartOfCoordinatedChange()` before starting. If any PR is already part of an active coordinated change, the request is **rejected** — the caller must cancel the existing coordinated set first. Partial overlap between coordinated sets would create ambiguous completion semantics (one set waiting for a review that another set might cancel).

**Out-of-order completion:** Reviews complete independently. Observer counts completions. Order does not matter.

**One review faults, others still running:** Observer signals `reviewFaulted` immediately. Parent failure goal fires. `onParentTerminal` handler cancels remaining reviews.

**Merge order:** Sequential, in the order repos appear in the request. Caller controls ordering by arranging the repos list.

**Partial failure during start:** If `PrReviewCaseService.startReview()` fails for one repo after others have already started, `CoordinatedChangeService.start()` cancels all already-started review cases and the parent case via `caseHubRuntime.cancelCase()`, then throws `CoordinatedChangeStartFailedException`. This ensures atomicity: either all review cases are created, or none remain active. Cleanup failures during cancellation are logged but do not suppress the original exception.

**Partial merge failure:** Repos A and B merge successfully, repo C fails. The worker returns results for all three. The `merge-failed` goal fires. The `rollback-on-merge-failure` binding fires (worker implemented in #158, not this batch).

## Risk: GE-20260521-9188c1

Garden entry documents that `when:` conditions on contextChange bindings may fire unconditionally. If still present, both `merge-all-repos` and `rollback-on-merge-failure` would fire on every context change regardless of their conditions. Mitigation: verify during implementation. If the issue persists, guard inside the worker functions with early-return on unmet preconditions.

## Not In Scope

- `coordinated-rollback` worker implementation (#158)
- REST endpoint or MCP tool for triggering coordinated changes (follows naturally from the port interface)
- End-to-end integration test (#160)
- Cross-repo integration CI (CI that spans multiple repos — not a current requirement)
