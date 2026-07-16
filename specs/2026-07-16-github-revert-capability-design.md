# GitHub Revert Capability — Design Spec

**Issue:** devtown#155
**Epic:** devtown#12 (Cross-repo coordinated merge)
**Date:** 2026-07-16

## Problem

The coordinated-rollback worker (#158) needs to undo merges in already-merged
repos when a sub-case faults. GitHub has no "revert" REST endpoint. The existing
`github/` module provides branch create/delete, PR merge, git merge, and CI
polling — but no revert capability.

## Prior Art (existing github module)

| Capability | Domain SPI | GitHub impl | REST API |
|---|---|---|---|
| PR merge | `MergeClient` | `GitHubMergeClient` | `GitHubMergeApi` (PUT pulls/merge) |
| Batch branch ops | `BatchBranchClient` | `GitHubBatchBranchClient` | `GitHubGitApi` (refs, merges) |
| CI status | `CiStatusClient` | `GitHubCiStatusClient` | `GitHubChecksApi` (check-suites) |
| **Revert** | **missing** | **missing** | **missing** |

All existing SPIs accept `(String owner, String repo)` — inherently multi-repo.
Cross-repo coordination is the workers' responsibility, not the API client's.

## Approach: PR-based revert

Target branches are typically protected — direct ref updates are blocked. The
revert goes through a PR, matching what a human would do and creating a
GitHub-native audit trail alongside the ledger's cryptographic record.

### Flow

```
1. GET  /repos/{o}/{r}/git/commits/{mergeSha}     → merge commit (validate ≥2 parents)
2. GET  /repos/{o}/{r}/git/commits/{parentSha}     → first parent's tree SHA
3. POST /repos/{o}/{r}/git/commits                 → revert commit
4. DELETE /repos/{o}/{r}/git/refs/heads/revert/...  → idempotent cleanup (may 404)
5. POST /repos/{o}/{r}/git/refs                    → temp branch at revert commit
6. GET  /repos/{o}/{r}/pulls?head=...&base=...     → find existing revert PR
7. POST /repos/{o}/{r}/pulls                       → revert PR (only if step 6 found none)
8. PUT  /repos/{o}/{r}/pulls/{n}/merge             → merge the revert PR
9. DELETE /repos/{o}/{r}/git/refs/heads/revert/...  → cleanup (skipped on MergeConflict)
```

Steps 4+5 use the delete-before-create idempotency pattern from
`GitHubBatchBranchClient`. Step 6 finds any existing open revert PR for the same
head/base pair — on retry, the previous attempt's PR is reused rather than
creating a duplicate. Step 8 uses `merge_method: "merge"` (not squash) to
preserve the revert commit's tree identity for auditability.

### Why first parent's tree

A merge commit has two parents: the target branch (first parent) and the merged
branch (second parent). Setting `tree = first parent's tree` undoes the merge's
file changes. Setting `parent = [mergeSha]` preserves linear history — the revert
commit appears after the merge, not instead of it. This is exactly what
`git revert -m 1 <merge-sha>` does.

## Domain SPI

### RevertClient

```java
package io.casehub.devtown.domain;

public interface RevertClient {
    RevertOutcome revert(String owner, String repo, String targetBranch, String mergeSha, String commitMessage);
}
```

### RevertOutcome

```java
package io.casehub.devtown.domain;

public sealed interface RevertOutcome {
    record Success(int revertPrNumber, String revertSha) implements RevertOutcome {}
    record MergeConflict(int revertPrNumber, String reason) implements RevertOutcome {}
    record Failure(String reason) implements RevertOutcome {}
}
```

- `Success` — revert PR merged; carries PR number (GitHub audit trail) and merge
  SHA (EventLog record).
- `MergeConflict` — revert PR could not be auto-merged. Covers content conflicts
  (409), branch protection requirements such as required status checks or reviews
  (405/422), and any other GitHub-side merge block. Carries the PR number so the
  rollback worker's escalation to `HumanOversight.ROUTING_REVIEW` can reference
  the PR directly. The revert PR and temp branch are left intact for human
  resolution.
- `Failure` — API error, network failure, or unexpected state. The rollback
  worker can retry via the engine's `OutcomePolicy` reroute loop.

## GitHubGitApi Extensions

Two new endpoints on the existing `@RegisterRestClient(configKey = "github-api")`
interface:

```java
@GET
@Path("/{owner}/{repo}/git/commits/{sha}")
GitCommit getCommit(@PathParam("owner") String owner,
                    @PathParam("repo") String repo,
                    @PathParam("sha") String sha);

@POST
@Path("/{owner}/{repo}/git/commits")
GitCommit createCommit(@PathParam("owner") String owner,
                       @PathParam("repo") String repo,
                       Map<String, Object> body);
```

### GitCommit record

```java
package io.casehub.devtown.github;

import java.util.List;

public record GitCommit(String sha, GitCommitTree tree, List<GitCommitParent> parents) {
    public record GitCommitTree(String sha) {}
    public record GitCommitParent(String sha) {}
}
```

## GitHubPullRequestApi

New REST client interface for PR operations:

```java
package io.casehub.devtown.github;

@RegisterRestClient(configKey = "github-api")
@Path("/repos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface GitHubPullRequestApi {

    @GET
    @Path("/{owner}/{repo}/pulls")
    List<Map<String, Object>> listPullRequests(@PathParam("owner") String owner,
                                                @PathParam("repo") String repo,
                                                @QueryParam("head") String head,
                                                @QueryParam("base") String base,
                                                @QueryParam("state") String state);

    @POST
    @Path("/{owner}/{repo}/pulls")
    Map<String, Object> createPullRequest(@PathParam("owner") String owner,
                                          @PathParam("repo") String repo,
                                          Map<String, Object> body);
}
```

Same `configKey` as all other GitHub REST clients — shares base URL and auth.
Returns raw `Map<String, Object>` / `List<Map<String, Object>>` matching the
existing convention.

## GitHubRevertClient

`@ApplicationScoped`, implements `RevertClient`. Constructor-injected with
`@RestClient GitHubGitApi`, `@RestClient GitHubPullRequestApi`,
`@RestClient GitHubMergeApi`.

### Revert flow

1. **Get merge commit** — `gitApi.getCommit(owner, repo, mergeSha)`. Validate it
   has at least two parents (merge commit). A single-parent SHA indicates a caller
   bug — the rollback worker should only pass merge commit SHAs from
   `MergeOutcome.Success.mergeSha()`.
2. **Get first parent** — `gitApi.getCommit(owner, repo, parents.getFirst().sha())`.
   Extract `tree.sha()` — this is the pre-merge tree.
3. **Create revert commit** — `gitApi.createCommit(owner, repo, body)` with
   `tree = parentCommit.tree().sha()`, `parents = [mergeSha]`,
   `message = commitMessage`.
4. **Create temp branch** — delete-before-create at
   `refs/heads/revert/<short-sha>` pointing to the revert commit.
5. **Find or create revert PR** — search for existing open PR via
   `prApi.listPullRequests(owner, repo, owner + ":revert/<short-sha>", targetBranch, "open")`.
   If found, reuse it (the branch recreation in step 4 already points to the
   correct commit). If not found, create via
   `prApi.createPullRequest(owner, repo, body)` with
   `head = "revert/<short-sha>"`, `base = targetBranch`,
   `title = commitMessage`.
6. **Merge revert PR** — `mergeApi.merge(owner, repo, prNumber, body)` with
   `merge_method = "merge"`. If 409 (conflict), 405 (branch protection), or 422
   (not mergeable) → return `MergeConflict(prNumber, reason)`. The revert PR and
   temp branch are left intact for human resolution.
7. **Cleanup temp branch** — `gitApi.deleteRef(owner, repo, "heads/revert/<short-sha>")`.
   Swallowed on failure — temp branch is harmless if orphaned. Skipped when
   step 6 returns `MergeConflict`.

### Error handling

Each step catches `WebApplicationException` and returns the appropriate
`RevertOutcome` variant:

| Step | Failure mode | Outcome |
|------|-------------|---------|
| 1–3 | API error (4xx/5xx) | `Failure` |
| 1 | Merge commit has <2 parents | `Failure("not a merge commit: expected ≥2 parents, got N")` |
| 4–5 | API error | `Failure` (temp branch cleanup attempted) |
| 6 | 409 Conflict | `MergeConflict(prNumber, reason)` — branch and PR left intact |
| 6 | 405 Branch protection | `MergeConflict(prNumber, reason)` — branch and PR left intact |
| 6 | 422 Not mergeable | `MergeConflict(prNumber, reason)` — branch and PR left intact |
| 6 | Other API error | `Failure` (temp branch cleanup attempted) |
| 7 | Any error | Swallowed — logged, not surfaced |

Steps 4–6 are wrapped so that temp branch cleanup runs on `Failure` outcomes.
On `MergeConflict`, cleanup is skipped — the temp branch and PR are left intact
so a human can resolve the conflict and merge the PR manually.

## NoOpRevertClient

```java
@DefaultBean
@ApplicationScoped
public class NoOpRevertClient implements RevertClient {
    @Override
    public RevertOutcome revert(String owner, String repo,
                                String targetBranch, String mergeSha,
                                String commitMessage) {
        return new RevertOutcome.Failure("no revert client configured");
    }
}
```

Follows the `@DefaultBean` displacement pattern — displaced by
`GitHubRevertClient @ApplicationScoped` when the `github` module is on the
classpath.

## File inventory

| Module | File | New/Modified |
|---|---|---|
| `domain/` | `RevertClient.java` | New |
| `domain/` | `RevertOutcome.java` | New |
| `github/` | `GitHubGitApi.java` | Modified — 2 new methods |
| `github/` | `GitCommit.java` | New |
| `github/` | `GitHubPullRequestApi.java` | New |
| `github/` | `GitHubRevertClient.java` | New |
| `app/spi/` | `NoOpRevertClient.java` | New |
| `domain/test` | `RevertOutcomeTest.java` | New |
| `github/test` | `GitCommitTest.java` | New |
| `github/test` | `GitHubRevertClientTest.java` | New |

## Test plan

| Test class | Module | What it covers |
|---|---|---|
| `RevertOutcomeTest` | `domain/test` | Sealed interface exhaustiveness — pattern match all three variants |
| `GitCommitTest` | `github/test` | Record construction and accessor correctness — tree SHA, parent list, nested records |
| `GitHubRevertClientTest` | `github/test` | Unit tests with mocked REST clients: |
| | | — Happy path: full 7-step flow, verify correct API calls and returned `Success` |
| | | — Merge conflict: step 6 returns 409 → `MergeConflict(prNumber, reason)`, temp branch NOT cleaned up |
| | | — Branch protection block: step 6 returns 405 → `MergeConflict(prNumber, reason)`, branch and PR left intact |
| | | — Not mergeable: step 6 returns 422 → `MergeConflict(prNumber, reason)`, branch and PR left intact |
| | | — API error on getCommit (step 1) → `Failure` |
| | | — API error on createCommit (step 3) → `Failure` |
| | | — API error on PR search/create (step 5) → `Failure`, temp branch cleaned up |
| | | — Other API error on merge (step 6, e.g., 500) → `Failure`, temp branch cleaned up |
| | | — Cleanup failure swallowed: step 7 deleteRef throws → still returns `Success` |
| | | — Idempotent temp branch: step 4 deleteRef 404 swallowed, createRef succeeds |
| | | — Non-merge commit: getCommit returns 1 parent → `Failure("not a merge commit: expected ≥2 parents, got 1")` |
| | | — PR reuse on retry: existing open PR found via listPullRequests → reused, no createPullRequest called |
| | | — Commit message: verify commitMessage appears in revert commit body and PR title |

## No pom.xml changes

`github/pom.xml` already declares all required dependencies:
`casehub-devtown-domain`, `quarkus-rest-client`, `quarkus-rest-client-jackson`,
JUnit, AssertJ, Mockito.

## Integration with rollback worker (#158)

The rollback worker receives a list of `{owner, repo, targetBranch, mergeSha}`
from the parent case context. For each already-merged repo:

```java
String message = "Revert merge " + mergeSha.substring(0, 7)
    + " — coordinated rollback (case " + caseId + ")";
RevertOutcome outcome = revertClient.revert(owner, repo, targetBranch, mergeSha, message);
switch (outcome) {
    case Success s    -> // record revert in EventLog, continue
    case MergeConflict mc -> // escalate to HumanOversight.ROUTING_REVIEW with mc.revertPrNumber()
    case Failure f    -> // engine OutcomePolicy handles retry/reroute
}
```

The rollback worker is #158's scope, not this issue's. This issue delivers the
SPI and GitHub implementation it will consume.
