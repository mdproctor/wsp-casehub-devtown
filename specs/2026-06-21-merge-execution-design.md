# Merge Execution — merge-executor worker via GitHub API

**Issue:** devtown#87
**Date:** 2026-06-21
**Status:** Approved

## Problem

The merge binding in `pr-review.yaml` fires capability `merge-executor` when all review goals + CI passing are satisfied. No worker exists to execute the merge — `NoOpWorkerProvisioner @DefaultBean` handles the dispatch (does nothing). PRs pass review but are never merged.

## Architecture

The engine dispatches `merge-executor` to a `WorkerFunction.Sync` — an inline Java function registered as a `@Named("merge-executor")` CDI bean. The function calls GitHub's merge API and returns the merge commit SHA.

### GitHubMergeClient (github module)

Thin REST client wrapping GitHub's merge API:

```
PUT /repos/{owner}/{repo}/pulls/{pull_number}/merge
{
  "merge_method": "squash",
  "sha": "{head_sha}"
}
```

**Config:**
```properties
devtown.github.token=${GITHUB_TOKEN:}
devtown.github.merge-method=squash
```

`merge-method` accepts `squash` (default), `rebase`, or `merge`. The `sha` parameter is a server-side guard — GitHub rejects the merge if the PR's HEAD has changed since the case context was last updated.

**Response handling:**
- 200 OK → extract `sha` (merge commit SHA) from response body
- 409 Conflict → merge conflict or SHA mismatch
- 405 Not Allowed → branch protection prevents merge
- 422 Unprocessable → PR not in mergeable state
- Network/other → propagate as failure

Returns a `MergeResult` record: `MergeResult(String mergeSha)` on success, throws `GitHubMergeException(int statusCode, String message)` on failure.

### MergeExecutorFunction (app module)

`WorkerFunction.Sync` implementation. Registered as `@Named("merge-executor") @ApplicationScoped`.

**Input:** `{ pr: { repo, id, headSha } }` — extracted from case context by the binding's `inputSchema`.

**Logic:**
1. Extract `repo`, `id` (PR number), `headSha` from the input map
2. Parse `repo` into `owner/repo` components
3. Call `GitHubMergeClient.merge(owner, repo, prNumber, headSha)`
4. On success: return `WorkerResult.of(Map.of("merge_sha", result.mergeSha()))`
5. On failure: return `WorkerResult.failed(exception.getMessage())`

**Output:** The `outputSchema: "{}"` in the YAML means the output is written to the case context root. The merge SHA lands in `merge_sha` at the top level.

### Audit trail

The merge SHA flows into the existing audit chain without additional wiring:
1. `WorkerResult.output` → written to case context by the engine
2. Case completes (all goals satisfied + merge binding done) → engine fires `CaseLifecycleEvent`
3. `MergeDecisionObserver` (Layer 4, already exists) creates `MergeDecisionLedgerEntry` with the merge decision and SHA
4. The ledger entry is tamper-evident (Merkle chain)

### Error handling

Merge failure is terminal for v1 — the case records the failure in the event log. The operator observes via MCP tools (`inspect_review`, `list_problems`). No automatic retry or reroute.

The merge binding's `outcomePolicy` is not set in the YAML (unlike review capability bindings which have `onFailure: REROUTE`). Merge is a one-shot operation — if it fails, human intervention is required.

## Files

| File | Module | Change |
|------|--------|--------|
| `github/.../GitHubMergeClient.java` | github | New — REST client for merge API |
| `github/.../MergeResult.java` | github | New — success result record |
| `github/.../GitHubMergeException.java` | github | New — failure exception |
| `app/.../merge/MergeExecutorFunction.java` | app | New — WorkerFunction.Sync for merge-executor |
| `app/src/main/resources/application.properties` | app | Add github.token, github.merge-method |
| `github/.../GitHubMergeClientTest.java` | github | New — mock HTTP tests |
| `app/.../merge/MergeExecutorFunctionTest.java` | app | New — mock client tests |

## What this does NOT include

- Retry/reroute on merge failure — terminal for v1
- Merge queue batch-then-bisect — devtown#11 (separate epic)
- GitHub App authentication — PAT is sufficient for v1
- Commit message customization — uses GitHub's default squash message
