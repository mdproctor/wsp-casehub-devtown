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
- Each review IS a standard pr-review case — same lifecycle, same YAML, same workers, same webhook routing.
- `PrReviewCaseService.startReview()` calls `caseTracker.register()`, so each review case is registered with `PrReviewCaseTracker`.
- `GitHubWebhookResource.handleCheckSuite()` calls `signalCiStatus(repo, prNumber, ...)` which looks up `caseTracker.findActiveCaseByPr(repo, prNumber)`. Per-repo webhook routing works unchanged.
- No engine changes. No coupling to engine internals.

## Coordination Flow

```
1. Trigger → CoordinatedChangeService.start(request)
2. Start parent case via CoordinatedChangeCaseHub.startCase(context)
   - Parent case has repos list, no active bindings yet
3. For each repo in request:
   - PrReviewCaseService.startReview(prPayload) → reviewCaseId
   - CoordinatedChangeTracker.register(parentCaseId, repo, reviewCaseId)
4. Each repo's PR goes through standard pr-review lifecycle
   - Code analysis, security review, CI, human gates — all existing
5. Webhooks arrive per-repo
   - Existing GitHubWebhookResource routes correctly via PrReviewCaseTracker
   - No webhook handler changes needed
6. Review case completes → CaseLifecycleEvent
   - CoordinatedChangeObserver detects it's tracked
   - Signals parent context: completedReviews[repo] = {status: "completed"}
7. ALL reviews complete
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

    public CoordinatedChangeOutcome start(CoordinatedChangeRequest request) {
        // 1. Build initial context with repos list
        // 2. Start parent case via caseHub.startCase(context)
        // 3. For each repo: reviewService.startReview(prPayload)
        // 4. Register each reviewCaseId with tracker
        // 5. Return parentCaseId + reviewCaseIds map
    }
}
```

**`CoordinatedChangeTracker`** — `@ApplicationScoped`, in-memory mapping:
```java
@ApplicationScoped
public class CoordinatedChangeTracker {
    // parentCaseId → CoordinationState
    // CoordinationState: {repos: Map<String, UUID>, completedRepos: Set<String>}

    void register(UUID parentCaseId, String repo, UUID reviewCaseId);
    Entry findByReviewCaseId(UUID reviewCaseId);
    Set<UUID> findReviewCaseIds(UUID parentCaseId);
    boolean markCompleted(UUID parentCaseId, String repo);
    boolean allCompleted(UUID parentCaseId);
}
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
        // If the parent case reaches terminal state, cancel remaining reviews
    }
}
```

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

**Existing active review for a repo:** `PrReviewCaseService.startReview()` checks for an existing case on the same repo+prNumber. If found, it calls `revisePr()` instead of creating a new case. `CoordinatedChangeService.start()` must check for conflicts before starting and reject or supersede.

**Out-of-order completion:** Reviews complete independently. Observer counts completions. Order does not matter.

**One review faults, others still running:** Observer signals `reviewFaulted` immediately. Parent failure goal fires. `onParentTerminal` handler cancels remaining reviews.

**Merge order:** Sequential, in the order repos appear in the request. Caller controls ordering by arranging the repos list.

**Partial merge failure:** Repos A and B merge successfully, repo C fails. The worker returns results for all three. The `merge-failed` goal fires. The `rollback-on-merge-failure` binding fires (worker implemented in #158, not this batch).

## Risk: GE-20260521-9188c1

Garden entry documents that `when:` conditions on contextChange bindings may fire unconditionally. If still present, both `merge-all-repos` and `rollback-on-merge-failure` would fire on every context change regardless of their conditions. Mitigation: verify during implementation. If the issue persists, guard inside the worker functions with early-return on unmet preconditions.

## Not In Scope

- `coordinated-rollback` worker implementation (#158)
- REST endpoint or MCP tool for triggering coordinated changes (follows naturally from the port interface)
- End-to-end integration test (#160)
- Cross-repo integration CI (CI that spans multiple repos — not a current requirement)
