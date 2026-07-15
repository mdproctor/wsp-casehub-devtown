# Batch Branch Management — git operations for merge queue batch testing

**Issue:** devtown#104
**Date:** 2026-07-15
**Status:** Approved
**Parent:** Epic #11 (Merge queue)
**Spec:** §7.3 of `2026-06-26-merge-queue-design.md`

---

## 1. Problem Statement

The merge queue CasePlanModel (`merge-batch.yaml`) defines a `batch-ci-runner` capability that "tests the tip-of-batch against the target branch." No worker exists to execute this — tests register mock workers inline. In production, the worker needs to create a temporary merge branch combining all batch PRs, make it available for CI, and clean up after the case completes.

The spec §7.3 lists six steps. Three are this worker's responsibility (create branch, merge PRs, push). One is external (CI runs). One is merge-executor scope (fast-forward target). One is lifecycle cleanup (delete branch). This spec covers the worker and the cleanup.

---

## 2. Lifecycle Trace — Git Operations by Phase

| Phase | Trigger | Git Operation | Actor |
|-------|---------|--------------|-------|
| Tip test | `test-batch-tip` binding | Create branch from targetBranch HEAD | `batch-ci-runner` worker |
| Tip test | `test-batch-tip` binding | Merge each PR SHA into branch | `batch-ci-runner` worker |
| CI runs | External (GitHub Actions, etc.) | None from devtown | CI system reads the branch |
| CI result | `check_suite.completed` webhook | None — signals `tipTest` via `caseHub.signal()` | Webhook handler |
| Bisection | Recursive sub-case | Same as tip test (smaller batch) | `batch-ci-runner` worker |
| Completion | `CaseLifecycleEvent` terminal | Delete batch branch | `BatchBranchCleanupObserver` |

**Not in scope:** Fast-forwarding targetBranch (merge-executor), individual PR merging via GitHub PR API (merge-executor), CI monitoring (webhook/connector), queue admission and batch formation (MergeQueueService).

---

## 3. Port Interface

### 3.1 BatchBranchClient

```java
// domain/src/main/java/io/casehub/devtown/domain/BatchBranchClient.java

public interface BatchBranchClient {

    BatchBranchOutcome createBatchBranch(
        String owner, String repo,
        String targetBranch, String batchId,
        List<PrRef> prs
    );

    BranchDeleteOutcome deleteBatchBranch(
        String owner, String repo,
        String branchName
    );
}
```

**Why separate from MergeClient:** `MergeClient` is about PR lifecycle (merge a single PR via the Pulls API). `BatchBranchClient` is about temporary branch lifecycle (create/delete refs via the Git Data API). Different responsibilities, different callers, different failure modes, different GitHub API endpoints.

### 3.2 PrRef

```java
// domain/src/main/java/io/casehub/devtown/domain/PrRef.java

public record PrRef(int number, String headSha) {}
```

Minimal value object — only what the git operation needs. No trust score, no priority, no author. The worker adapter extracts these from the batch context map.

### 3.3 BatchBranchOutcome

```java
// domain/src/main/java/io/casehub/devtown/domain/BatchBranchOutcome.java

public sealed interface BatchBranchOutcome {
    record Created(String branchName, String tipSha) implements BatchBranchOutcome {}
    record MergeConflict(int conflictPrNumber, String branchName) implements BatchBranchOutcome {}
    record Failure(String reason) implements BatchBranchOutcome {}
}
```

- **Created**: all PRs merged successfully. Branch exists on remote, ready for CI.
- **MergeConflict**: a specific PR's SHA could not be merged. The partial branch still exists (cleanup observer handles deletion on case terminal state).
- **Failure**: infrastructure error (API error, branch already exists, target branch not found).

### 3.4 BranchDeleteOutcome

```java
// domain/src/main/java/io/casehub/devtown/domain/BranchDeleteOutcome.java

public sealed interface BranchDeleteOutcome {
    record Deleted(String branchName) implements BranchDeleteOutcome {}
    record NotFound(String branchName) implements BranchDeleteOutcome {}
    record Failure(String reason) implements BranchDeleteOutcome {}
}
```

`NotFound` is not an error — idempotent. The branch may have been deleted manually or the creation never completed.

---

## 4. GitHub Implementation

### 4.1 GitHubGitApi

New MicroProfile REST client for the GitHub Git Data API. Shares `configKey = "github-api"` with `GitHubMergeApi` — same token, same base URL.

```java
// github/src/main/java/io/casehub/devtown/github/GitHubGitApi.java

@RegisterRestClient(configKey = "github-api")
@Path("/repos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface GitHubGitApi {

    @GET
    @Path("/{owner}/{repo}/git/ref/{ref}")
    GitRef getRef(@PathParam("owner") String owner,
                  @PathParam("repo") String repo,
                  @PathParam("ref") String ref);

    @POST
    @Path("/{owner}/{repo}/git/refs")
    GitRef createRef(@PathParam("owner") String owner,
                     @PathParam("repo") String repo,
                     Map<String, String> body);

    @DELETE
    @Path("/{owner}/{repo}/git/refs/{ref}")
    void deleteRef(@PathParam("owner") String owner,
                   @PathParam("repo") String repo,
                   @PathParam("ref") String ref);

    @POST
    @Path("/{owner}/{repo}/merges")
    Map<String, Object> merge(@PathParam("owner") String owner,
                              @PathParam("repo") String repo,
                              Map<String, String> body);
}
```

**`GitRef`** — minimal response record:
```java
// github/src/main/java/io/casehub/devtown/github/GitRef.java

public record GitRef(String ref, Object object) {
    public String sha() {
        if (object instanceof Map<?, ?> m) {
            return (String) m.get("sha");
        }
        return null;
    }
}
```

The GitHub Git Refs API returns `{ "ref": "refs/heads/...", "object": { "sha": "..." } }`. The `sha()` accessor handles the nested structure.

### 4.2 GitHubBatchBranchClient

```java
// github/src/main/java/io/casehub/devtown/github/GitHubBatchBranchClient.java

@ApplicationScoped
public class GitHubBatchBranchClient implements BatchBranchClient {

    private final GitHubGitApi gitApi;

    public GitHubBatchBranchClient(@RestClient GitHubGitApi gitApi) {
        this.gitApi = gitApi;
    }

    @Override
    public BatchBranchOutcome createBatchBranch(
            String owner, String repo,
            String targetBranch, String batchId,
            List<PrRef> prs) {
        try {
            // 1. Resolve targetBranch HEAD SHA
            String baseSha = gitApi.getRef(owner, repo, "heads/" + targetBranch).sha();
            if (baseSha == null) {
                return new BatchBranchOutcome.Failure(
                    "target branch '" + targetBranch + "' has no resolvable SHA");
            }

            // 2. Ensure idempotent — delete any stale branch from a previous attempt
            String branchName = "merge-queue/batch-" + batchId;
            try {
                gitApi.deleteRef(owner, repo, "heads/" + branchName);
            } catch (WebApplicationException ignored) {
                // 422 = doesn't exist — expected on first attempt
            }

            // 3. Create branch ref
            gitApi.createRef(owner, repo,
                Map.of("ref", "refs/heads/" + branchName, "sha", baseSha));

            // 4. Merge each PR SHA in order
            for (PrRef pr : prs) {
                try {
                    gitApi.merge(owner, repo,
                        Map.of("base", branchName, "head", pr.headSha(),
                               "commit_message", "Merge PR #" + pr.number() + " into " + branchName));
                } catch (WebApplicationException e) {
                    if (e.getResponse().getStatus() == 409) {
                        return new BatchBranchOutcome.MergeConflict(pr.number(), branchName);
                    }
                    return new BatchBranchOutcome.Failure(
                        "merge failed for PR #" + pr.number() + ": HTTP " + e.getResponse().getStatus());
                }
            }

            // 5. Read final tip SHA
            String tipSha = gitApi.getRef(owner, repo, "heads/" + branchName).sha();
            if (tipSha == null) {
                return new BatchBranchOutcome.Failure(
                    "batch branch '" + branchName + "' has no resolvable SHA after merges");
            }
            return new BatchBranchOutcome.Created(branchName, tipSha);

        } catch (WebApplicationException e) {
            return new BatchBranchOutcome.Failure("api error: HTTP " + e.getResponse().getStatus());
        } catch (Exception e) {
            return new BatchBranchOutcome.Failure("api error: " + e.getMessage());
        }
    }

    @Override
    public BranchDeleteOutcome deleteBatchBranch(String owner, String repo, String branchName) {
        try {
            gitApi.deleteRef(owner, repo, "heads/" + branchName);
            return new BranchDeleteOutcome.Deleted(branchName);
        } catch (WebApplicationException e) {
            if (e.getResponse().getStatus() == 422) {
                return new BranchDeleteOutcome.NotFound(branchName);
            }
            return new BranchDeleteOutcome.Failure("delete failed: HTTP " + e.getResponse().getStatus());
        } catch (Exception e) {
            return new BranchDeleteOutcome.Failure("delete failed: " + e.getMessage());
        }
    }
}
```

### 4.3 NoOpBatchBranchClient

```java
// app/src/main/java/io/casehub/devtown/app/spi/NoOpBatchBranchClient.java

@DefaultBean
@ApplicationScoped
public class NoOpBatchBranchClient implements BatchBranchClient {

    @Override
    public BatchBranchOutcome createBatchBranch(
            String owner, String repo,
            String targetBranch, String batchId,
            List<PrRef> prs) {
        return new BatchBranchOutcome.Failure("no batch branch client configured");
    }

    @Override
    public BranchDeleteOutcome deleteBatchBranch(String owner, String repo, String branchName) {
        return new BranchDeleteOutcome.Failure("no batch branch client configured");
    }
}
```

---

## 5. Worker Registration

### 5.1 MergeBatchCaseHub.augment()

The `batch-ci-runner` worker is added alongside the existing `bisection-splitter` and `pr-reject-and-notify` workers:

```java
@Inject BatchBranchClient batchBranchClient;

@Override
protected void augment(CaseDefinition definition) {
    // existing workers (bisection-splitter, pr-reject-and-notify)...

    definition.getWorkers().add(Worker.builder()
        .name("batch-ci-runner")
        .capabilityName("batch-ci-runner")
        .function(this::adaptBatchCiRunner)
        .build());
}
```

### 5.1.1 Prerequisite: BatchSlice repository propagation

The bisection sub-case bindings (`bisect-left`, `bisect-right`) pass `{ batch: .splitResult.left }` as sub-case context. The split result comes from `MergeBatchCaseHub.sliceToMap()` which converts `BatchSlice` records. For the worker adapter to read `batch.repository` in bisection sub-cases, `BatchSlice` must carry `repository`:

```java
public record BatchSlice(
        String id,
        String repository,
        String targetBranch,
        List<QueuedPr> prs,
        int size,
        String parentBatchId,
        int bisectionDepth,
        String bisectionStrategy,
        String riskLevel
) {}
```

`BisectionSplitStrategy.split()` gains a `repository` parameter, and `MergeBatchCaseHub.sliceToMap()` includes `map.put("repository", slice.repository())`. These are changes to existing `queue` and `app` module code that must land alongside this spec.

**bisection-splitter inputSchema fix:** The existing `bisection-splitter` capability's inputSchema passes only `prs` and `strategy` — not the `batch` object. The `adaptBisectionSplit()` adapter reads `input.get("batch")` which returns null, causing all batch-level metadata (batchId, targetBranch, bisectionDepth, riskLevel) to fall through to hardcoded defaults. This pre-existing bug also prevents the adapter from providing `repository` to `split()`.

Fix: add `batch: .batch` to the inputSchema, consistent with `batch-ci-runner` which already uses `inputSchema: "{ batch: .batch }"`:

```yaml
inputSchema: '{ prs: .batch.prs, strategy: (.batch.bisectionStrategy // "trust-weighted"), batch: .batch }'
```

This makes all batch metadata available to the adapter, fixing both the new `repository` requirement and the pre-existing defaults bug in one change.

### 5.2 Worker Adapter

```java
WorkerResult adaptBatchCiRunner(Map<String, Object> input) {
    @SuppressWarnings("unchecked")
    Map<String, Object> batch = (Map<String, Object>) input.get("batch");
    String repository = (String) batch.get("repository");
    if (repository == null || !repository.contains("/")) {
        return WorkerResult.failed("batch context missing or invalid 'repository': " + repository);
    }
    String[] parts = repository.split("/");
    String targetBranch = (String) batch.get("targetBranch");
    String batchId = (String) batch.get("id");

    @SuppressWarnings("unchecked")
    List<Map<String, Object>> prMaps = (List<Map<String, Object>>) batch.get("prs");
    List<PrRef> prs = prMaps.stream()
        .map(m -> new PrRef(
            ((Number) m.get("number")).intValue(),
            (String) m.get("headSha")))
        .toList();

    return switch (batchBranchClient.createBatchBranch(parts[0], parts[1], targetBranch, batchId, prs)) {
        case BatchBranchOutcome.Created c ->
            WorkerResult.of(Map.of(
                "status", "branch-created",
                "branch", c.branchName(),
                "tipSha", c.tipSha()));
        case BatchBranchOutcome.MergeConflict mc ->
            WorkerResult.failed(
                "merge conflict on PR #" + mc.conflictPrNumber(),
                Map.of("status", "conflict",
                       "conflictPr", mc.conflictPrNumber(),
                       "branch", mc.branchName()));
        case BatchBranchOutcome.Failure f ->
            WorkerResult.failed(f.reason());
    };
}
```

Via `outputSchema: "{ tipTest: . }"`, the worker output becomes `tipTest` in the case context. CI feedback later overwrites `tipTest` via `signal()` with `{ status: "passing" }` or `{ status: "failing" }`.

### 5.3 Conflict and outcomePolicy interaction

When `batch-ci-runner` returns `WorkerResult.failed()`, the merge-batch.yaml `outcomePolicy` applies:

```yaml
outcomePolicy:
  onFailure: REROUTE
  maxRerouteAttempts: 2
```

The engine reroutes to another worker. With only one registered `batch-ci-runner` worker, reroute attempts exhaust quickly. After exhaustion, the engine writes `{ status: "REROUTES_EXHAUSTED" }` to `tipTest` (via the capability's `outputSchema`), triggering the `tip-test-escalation` binding.

The delete-before-create pattern in §4.2 (step 2) ensures reroutes are idempotent — each attempt deletes any stale branch from the previous attempt, creates a fresh branch, and genuinely retries the merge. Without this, reroutes would fail on `createRef` with HTTP 422 ("Reference already exists"), masking the real conflict as an infrastructure error.

**Escalation chain for merge conflict:**

1. Worker returns `WorkerResult.failed()` with `{ status: "conflict", conflictPr: N }` → engine applies REROUTE
2. Reroute to same worker → deletes stale branch, creates fresh, same deterministic conflict recurs
3. After 2 attempts, engine writes `tipTest.status = "REROUTES_EXHAUSTED"`
4. `tip-test-escalation` binding fires → human task (`repo-maintainers`, expires: 2h)
5. Human outcomes:
   - **RETRY** → `tip-test-after-escalation` clears `tipTest`, re-dispatches with `onFailure: FAULT`
   - **REJECT_BATCH** → `tip-test-terminal-failure` goal fires, case terminates
   - **BLOCKED** → same as REJECT_BATCH

For merge conflicts, rerouting is pointless — the conflict is deterministic. The two reroute attempts are wasted. A future optimisation could short-circuit conflicts directly to escalation via per-failure-reason outcomePolicy, which the engine does not yet support (see [#4](https://github.com/mdproctor/wsp-casehub-devtown/issues/4)).

### 5.4 `tipTest` lifecycle

The `tipTest` context variable is shared between two writers — the `batch-ci-runner` worker sets the initial state, and the CI webhook handler overwrites it with the CI result:

| State | Writer | Trigger |
|-------|--------|---------|
| `null` | (initial) | `test-batch-tip` binding fires |
| `{ status: "branch-created", branch, tipSha }` | Worker (success) | Case waits for CI signal |
| `{ status: "conflict", conflictPr, branch }` | Worker (failure) | REROUTE via outcomePolicy |
| `{ status: "passing" }` | CI webhook via `signal()` | `merge-batch` or `human-merge-approval` fires |
| `{ status: "failing" }` | CI webhook via `signal()` | `compute-bisection-split` or `reject-single-pr` fires |
| `{ status: "REROUTES_EXHAUSTED" }` | Engine | `tip-test-escalation` fires |

**Signal overwrites:** When the CI webhook signals `{ status: "passing" }`, it replaces the entire `tipTest` value — the `branch` and `tipSha` fields from the worker's initial write are lost. This is acceptable: no downstream binding reads those fields, and the webhook handler has access to branch information from the GitHub event payload independently.

---

## 6. Cleanup Observer

### 6.0.1 Prerequisite: CaseTrackingStatus SUPERSEDED mapping

`CaseTrackingStatus.fromCaseStatus()` is missing the `"SUPERSEDED"` case — it falls through to the default `RUNNING`, so `RUNNING.isTerminal()` returns false. SUPERSEDED cases would not trigger cleanup, leaking the batch branch.

Fix in `app/src/main/java/io/casehub/devtown/app/mcp/CaseTrackingStatus.java`:

```java
public static CaseTrackingStatus fromCaseStatus(String caseStatus) {
    return switch (caseStatus) {
        case "COMPLETED" -> COMPLETED;
        case "FAULTED" -> FAULTED;
        case "CANCELLED" -> CANCELLED;
        case "SUPERSEDED" -> SUPERSEDED;   // ← added
        case "WAITING" -> WAITING;
        default -> RUNNING;
    };
}
```

This is a pre-existing bug in `CaseTrackingStatus` that affects any observer relying on `isTerminal()`. Must land before or alongside this spec.

### 6.1 BatchBranchCleanupObserver

```java
// app/src/main/java/io/casehub/devtown/app/BatchBranchCleanupObserver.java

@ApplicationScoped
public class BatchBranchCleanupObserver {

    private static final Logger LOG = Logger.getLogger(BatchBranchCleanupObserver.class);

    @Inject BatchBranchClient batchBranchClient;

    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (event.caseStatus() == null) return;
        if (!CaseTrackingStatus.fromCaseStatus(event.caseStatus()).isTerminal()) return;
        if (!"devtown".equals(event.namespace())) return;
        if (!"merge-batch".equals(event.caseDefinitionName())) return;

        JsonNode context = event.contextSnapshot();
        if (context == null) return;

        JsonNode batch = context.path("batch");
        String repository = batch.path("repository").asText(null);
        if (repository == null || !repository.contains("/")) return;
        String[] parts = repository.split("/");

        String batchId = batch.path("id").asText(null);
        if (batchId == null) return;
        String branchName = "merge-queue/batch-" + batchId;

        var result = batchBranchClient.deleteBatchBranch(parts[0], parts[1], branchName);
        switch (result) {
            case BranchDeleteOutcome.Deleted d ->
                LOG.infof("Cleaned up batch branch %s for case %s", d.branchName(), event.caseId());
            case BranchDeleteOutcome.NotFound nf ->
                LOG.debugf("Batch branch %s already gone for case %s", nf.branchName(), event.caseId());
            case BranchDeleteOutcome.Failure f ->
                LOG.warnf("Failed to clean up batch branch for case %s: %s", event.caseId(), f.reason());
        }
    }
}
```

**Design choices:**
- Filters on `namespace == "devtown"` and `caseDefinitionName == "merge-batch"` — scoped to devtown merge queue cases only
- Derives branch name from batch ID using the same `merge-queue/batch-{id}` convention — no extra state stored
- Fire-and-forget — cleanup failure is logged but doesn't affect the case (already terminal). If GitHub is unavailable at cleanup time, the branch leaks. A periodic sweep job would address accumulated leaks — see [#5](https://github.com/mdproctor/wsp-casehub-devtown/issues/5)
- Fires for every terminal batch case including bisection sub-cases — each created its own branch, each needs cleanup

---

## 7. Module Placement

| File | Module | Rationale |
|------|--------|-----------|
| `BatchBranchClient` | `domain/` | Port interface — pure Java, no framework deps |
| `PrRef` | `domain/` | Domain value object |
| `BatchBranchOutcome` | `domain/` | Sealed result type for the port |
| `BranchDeleteOutcome` | `domain/` | Sealed result type for the port |
| `GitHubGitApi` | `github/` | REST client interface — GitHub boundary |
| `GitRef` | `github/` | GitHub API response record |
| `GitHubBatchBranchClient` | `github/` | Port implementation — GitHub adapter |
| `NoOpBatchBranchClient` | `app/spi/` | `@DefaultBean` fallback |
| Worker adapter (`adaptBatchCiRunner`) | `app/` | In `MergeBatchCaseHub` — CDI wiring layer |
| `BatchBranchCleanupObserver` | `app/` | CDI observer — application-layer lifecycle |

Follows the three-tier rule: `domain/` = pure Java, `github/` = external boundary, `app/` = CDI wiring.

---

## 8. Testing Strategy

### 8.1 Unit Tests (domain/)

| Test | Covers |
|------|--------|
| `PrRefTest` | Record equality, construction |
| `BatchBranchOutcomeTest` | Sealed type exhaustiveness, pattern matching |
| `BranchDeleteOutcomeTest` | Sealed type exhaustiveness |

### 8.2 Unit Tests (github/)

| Test | Covers |
|------|--------|
| `GitHubBatchBranchClientTest` | Happy path: all PRs merge → Created |
| | Conflict on 2nd PR → MergeConflict with correct PR number and branch name |
| | Conflict on 1st PR → MergeConflict (no successful merges before conflict) |
| | Target branch not found → Failure |
| | Create ref fails (branch exists) → Failure |
| | API error mid-merge (non-409) → Failure |
| | Delete happy path → Deleted |
| | Delete not found (422) → NotFound |
| | Delete API error → Failure |
| | Branch already exists from previous attempt → deletes and recreates successfully |
| | Target branch returns null SHA → Failure with descriptive message |
| | Batch branch returns null SHA after merges → Failure |
| `GitRefTest` | SHA extraction from nested object structure |

Uses Mockito for `GitHubGitApi` — these are unit tests of the adapter logic, not integration tests of the REST client.

### 8.3 Integration Tests (@QuarkusTest, app/)

| Test | Covers |
|------|--------|
| `BatchCiRunnerWorkerTest` | Worker adapter: maps batch context → `BatchBranchClient` call → `WorkerResult` |
| | Created outcome → `WorkerResult.of` with status/branch/tipSha |
| | MergeConflict outcome → `WorkerResult.failed` with conflict details |
| | Failure outcome → `WorkerResult.failed` |
| | Missing repository in batch context → `WorkerResult.failed` with descriptive message |
| | Invalid repository format (no `/`) → `WorkerResult.failed` |
| `BatchBranchCleanupObserverTest` | Terminal CaseLifecycleEvent for merge-batch → calls `deleteBatchBranch` |
| | Non-terminal event → no cleanup |
| | Non-merge-batch event → no cleanup |
| | Non-devtown namespace event → no cleanup |
| | SUPERSEDED case status → triggers cleanup |
| | Invalid repository format in context → no cleanup |
| | Null context → no cleanup |
| | Missing batch/repository in context → no cleanup |
| | Delete returns NotFound → logs debug, no error |
| | Delete returns Failure → logs warning |
| `MergeQueueBatchLifecycleTest` | Existing tests continue to pass (mock worker registration pattern unchanged) |

### 8.4 Not in scope

- GitHub API integration tests (require live GitHub token)
- CI webhook → signal flow (connector concern)
- Merge execution after batch test passes (merge-executor concern)

---

## 9. Revision History

- **v4 (2026-07-15):** Review round 3 fix. Extended §5.1.1 prerequisite with `bisection-splitter` inputSchema change — added `batch: .batch` to make batch metadata available to the adapter. Fixes both the new `repository` requirement and pre-existing defaults bug.
- **v3 (2026-07-15):** Review round 2 fixes. Made `createBatchBranch()` idempotent with delete-before-create pattern (§4.2 step 2), fixing stale branch blocking reroutes after merge conflict. Added §6.0.1 prerequisite for `CaseTrackingStatus.fromCaseStatus()` SUPERSEDED mapping.
- **v2 (2026-07-15):** Review round 1 fixes. Added `repository` to `BatchSlice` prerequisite (§5.1.1). Added null checks for `GitRef.sha()` and merge commit message (§4.2). Added namespace filter and format validation to cleanup observer (§6.1). Added input validation in worker adapter (§5.2). Documented full REROUTES_EXHAUSTED escalation chain (§5.3) and `tipTest` lifecycle (§5.4). Filed [#3](https://github.com/mdproctor/wsp-casehub-devtown/issues/3) (RepoRef domain type), [#4](https://github.com/mdproctor/wsp-casehub-devtown/issues/4) (per-failure-reason outcomePolicy), [#5](https://github.com/mdproctor/wsp-casehub-devtown/issues/5) (cleanup sweep job) for deferred items.
- **v1 (2026-07-15):** Initial design. Traced complete merge queue lifecycle to identify exact git operations. Two-method port interface, GitHub Git Data API implementation, CDI cleanup observer on CaseLifecycleEvent.
