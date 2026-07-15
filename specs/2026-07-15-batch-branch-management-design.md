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

            // 2. Create branch ref
            String branchName = "merge-queue/batch-" + batchId;
            gitApi.createRef(owner, repo,
                Map.of("ref", "refs/heads/" + branchName, "sha", baseSha));

            // 3. Merge each PR SHA in order
            for (PrRef pr : prs) {
                try {
                    gitApi.merge(owner, repo,
                        Map.of("base", branchName, "head", pr.headSha()));
                } catch (WebApplicationException e) {
                    if (e.getResponse().getStatus() == 409) {
                        return new BatchBranchOutcome.MergeConflict(pr.number(), branchName);
                    }
                    return new BatchBranchOutcome.Failure(
                        "merge failed for PR #" + pr.number() + ": HTTP " + e.getResponse().getStatus());
                }
            }

            // 4. Read final tip SHA
            String tipSha = gitApi.getRef(owner, repo, "heads/" + branchName).sha();
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

### 5.2 Worker Adapter

```java
WorkerResult adaptBatchCiRunner(Map<String, Object> input) {
    @SuppressWarnings("unchecked")
    Map<String, Object> batch = (Map<String, Object>) input.get("batch");
    String repository = (String) batch.get("repository");
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

This means the engine retries with a different worker (if available). For a merge conflict, retrying is pointless — the same conflict will recur. But with only one registered `batch-ci-runner` worker, the engine exhausts reroute attempts and the plan item FAULTs, which is correct behaviour. A future optimisation could use `FAULT` directly for conflicts, but that requires per-failure-reason outcomePolicy which the engine does not yet support.

---

## 6. Cleanup Observer

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
        if (!"merge-batch".equals(event.caseDefinitionName())) return;

        JsonNode context = event.contextSnapshot();
        if (context == null) return;

        JsonNode batch = context.path("batch");
        String repository = batch.path("repository").asText(null);
        if (repository == null) return;
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
- Filters on `caseDefinitionName == "merge-batch"` — only merge queue cases, not PR review cases
- Derives branch name from batch ID using the same `merge-queue/batch-{id}` convention — no extra state stored
- Fire-and-forget — cleanup failure is logged but doesn't affect the case (already terminal)
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
| `GitRefTest` | SHA extraction from nested object structure |

Uses Mockito for `GitHubGitApi` — these are unit tests of the adapter logic, not integration tests of the REST client.

### 8.3 Integration Tests (@QuarkusTest, app/)

| Test | Covers |
|------|--------|
| `BatchCiRunnerWorkerTest` | Worker adapter: maps batch context → `BatchBranchClient` call → `WorkerResult` |
| | Created outcome → `WorkerResult.of` with status/branch/tipSha |
| | MergeConflict outcome → `WorkerResult.failed` with conflict details |
| | Failure outcome → `WorkerResult.failed` |
| `BatchBranchCleanupObserverTest` | Terminal CaseLifecycleEvent for merge-batch → calls `deleteBatchBranch` |
| | Non-terminal event → no cleanup |
| | Non-merge-batch event → no cleanup |
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

- **v1 (2026-07-15):** Initial design. Traced complete merge queue lifecycle to identify exact git operations. Two-method port interface, GitHub Git Data API implementation, CDI cleanup observer on CaseLifecycleEvent.
