# GitHub Combined-Status Check — Design Spec

**Issue:** devtown#89
**Date:** 2026-06-23
**Status:** Approved (rev 3 — post-review round 2)

## Problem

`PrReviewCaseService.signalCiStatus()` aggregates CI status locally using an in-memory `ConcurrentHashMap<UUID, ConcurrentHashMap<Long, String>>`. This causes two bugs:

1. **Aggregation gap:** The service doesn't know the total expected suite count. When the first suite completes as "success", it signals `ci.status = "passing"` even though other suites haven't reported yet.
2. **Atomicity race:** Two concurrent `check_suite` webhooks can both read zero existing suites and both signal "passing".

Both are symptoms of aggregating locally instead of querying the source of truth.

## Solution

Query GitHub's Checks API for the authoritative combined status before signaling `ci.status`. Remove the in-memory aggregation map entirely. GitHub is the single source of truth for CI status — devtown should read it, not replicate it.

## Design

### Domain SPI — `CiStatusClient`

New interface in `domain/` (alongside `MergeClient`):

```java
package io.casehub.devtown.domain;

public interface CiStatusClient {
    CombinedCiStatus getCombinedStatus(String owner, String repo, String headSha);
}
```

Parameters follow the `MergeClient.merge(String owner, String repo, ...)` convention — owner and repo are separate. The caller splits the combined `"owner/repo"` string at the call site (same as `GitHubMergeClient` callers).

```java
package io.casehub.devtown.domain;

public sealed interface CombinedCiStatus {
    record Passing() implements CombinedCiStatus {}
    record Failing(String summary) implements CombinedCiStatus {}
    record Pending(int completed, int total) implements CombinedCiStatus {}
    record Unavailable(String reason) implements CombinedCiStatus {}
}
```

- `Passing` — all suites complete and non-blocking
- `Failing(summary)` — at least one suite has a blocking conclusion (e.g. "2 of 5 suites failed")
- `Pending(completed, total)` — suites still running, or pagination returned an incomplete view; counts for logging. `Pending(0, 0)` means no check suites exist — CI may not be configured or may not have started yet; either way, don't signal passing.
- `Unavailable(reason)` — the API call failed (network error, rate limit, transient 500). Follows the `MergeOutcome.Failure(reason)` precedent: infrastructure errors are caught in the GitHub implementation and returned as a domain result type. The service treats this identically to `Pending` — no `ci.status` signal, next webhook retries the query. Logged at WARN.

### Check Suite Conclusion Classification

GitHub check suites report one of these conclusions: `success`, `failure`, `neutral`, `cancelled`, `timed_out`, `action_required`, `stale`, `skipped`.

Not all non-success conclusions indicate failure. `neutral` (informational checks — Codecov, advisory linters) and `skipped` (path-filtered workflows that intentionally didn't run) are non-blocking:

```java
private static final Set<String> NON_BLOCKING = Set.of("success", "neutral", "skipped");
```

Blocking conclusions (`failure`, `cancelled`, `timed_out`, `action_required`, `stale`) represent genuine CI problems.

Classification:
- All completed and all `NON_BLOCKING` → `Passing`
- All completed and any blocking → `Failing(summary)`
- Any not completed → `Pending(completedCount, totalCount)`

### GitHub Implementation

**REST client interface** — `GitHubChecksApi` in `github/`, reusing the `github-api` config key:

```java
@RegisterRestClient(configKey = "github-api")
@Path("/repos")
public interface GitHubChecksApi {
    @GET
    @Path("/{owner}/{repo}/commits/{ref}/check-suites")
    CheckSuitesResponse listCheckSuites(
        @PathParam("owner") String owner,
        @PathParam("repo") String repo,
        @PathParam("ref") String ref,
        @QueryParam("per_page") int perPage);
}
```

`CheckSuitesResponse` maps the GitHub response:
- `total_count: int`
- `check_suites: List<CheckSuite>` where each has `id: long`, `status: String` ("queued"/"in_progress"/"completed"), `conclusion: String` (null/"success"/"failure"/"neutral"/etc.)

**Implementation** — `GitHubCiStatusClient` (`@ApplicationScoped`, implements `CiStatusClient`):

- Calls `api.listCheckSuites(owner, repo, headSha, 100)` — `per_page=100` (GitHub max) to minimize pagination
- If `total_count == 0` → `Pending(0, 0)`
- If `total_count > check_suites.size()` → `Pending(completedInPage, total_count)` — incomplete view, conservative fallback (log warning)
- If any suite has `status != "completed"` → `Pending(completedCount, totalCount)`
- If all completed and any has a blocking conclusion → `Failing(summary)`
- If all completed and all non-blocking → `Passing()`
- On `WebApplicationException` or `Exception` → `Unavailable(reason)` with HTTP status or exception message

No new dependencies — `quarkus-rest-client` and `quarkus-rest-client-jackson` are already in `github/pom.xml`. No new configuration — reuses the existing `github-api` config key (same base URL and auth as `GitHubMergeApi`).

### Changes to `PrReviewCaseService`

**Remove:**
- `ciSuiteResults` field (`ConcurrentHashMap<UUID, ConcurrentHashMap<Long, String>>`)
- `ciSuiteResults.remove(caseId)` in `revisePr()`
- Local aggregation logic (`anyFailed`/`allSuccess`)

**Keep:**
- SHA freshness check
- Individual suite recording in case context (`ci.suites.<suiteId>`) — audit trail

**Add:**
- `@Inject CiStatusClient ciStatusClient`
- After recording the individual suite, split `repo` into `owner`/`repo` and call `ciStatusClient.getCombinedStatus(owner, repo, headSha)`
- Signal based on result:
  - `Passing` → `caseHub.signal(caseId, "ci.status", "passing")`
  - `Failing` → `caseHub.signal(caseId, "ci.status", "failing")`
  - `Pending` → no `ci.status` signal (wait for next webhook)
  - `Unavailable` → no `ci.status` signal, log at WARN (next webhook retries)

### Re-runs

Check suite re-runs work naturally: re-running a failed suite triggers a new `check_suite.completed` webhook → API query → updated combined status reflecting the re-run result.

### Edge Cases

**Pagination:** The request uses `per_page=100` (GitHub's maximum). If `total_count` still exceeds the returned list size (>100 check suites per commit), return `Pending(completedInPage, total_count)` — conservative fallback. This is the same aggregation gap bug the spec exists to fix, triggered by a different cause.

**Zero check suites:** `total_count: 0` → `Pending(0, 0)`. Don't signal passing when nothing has run.

## Test Strategy

### `PrReviewCaseServiceCiStatusTest` (update existing)

Mock `CiStatusClient` alongside the existing mock `PrReviewCaseHub`:

- `signalCiStatus_githubConfirmsPassing` — returns `Passing` → signals "passing"
- `signalCiStatus_githubSaysPending` — returns `Pending(1, 3)` → no `ci.status` signal
- `signalCiStatus_githubConfirmsFailing` — returns `Failing` → signals "failing"
- `signalCiStatus_githubUnavailable` — returns `Unavailable` → no `ci.status` signal
- `signalCiStatus_staleSha_returnsStaleEvent` — unchanged, `CiStatusClient` never called
- `signalCiStatus_noActiveCase_returnsNoActiveCase` — unchanged, `CiStatusClient` never called
- `signalCiStatus_alwaysRecordsSuiteRegardlessOfCombinedStatus` — call with each `CombinedCiStatus` variant (`Passing`, `Pending`, `Failing`, `Unavailable`) and verify `ci.suites.<suiteId>` is written in all four cases. The audit trail write is unconditional — this test makes that invariant explicit.

Remove tests that depend on the in-memory map (multi-suite aggregation, sticky-failure policy).

### `GitHubCiStatusClientTest` (new)

Mock `GitHubChecksApi` responses:

- All suites completed + success → `Passing`
- All completed, mix of success and neutral → `Passing` (neutral is non-blocking)
- All completed, one skipped rest success → `Passing` (skipped is non-blocking)
- All completed, one failure → `Failing` with summary
- All completed, one cancelled → `Failing` (cancelled is blocking)
- All completed, one timed_out → `Failing` (timed_out is blocking — makes the design decision visible and reviewable)
- Some suites in_progress → `Pending` with counts
- Zero suites → `Pending(0, 0)`
- `total_count` exceeds returned list size → `Pending` (incomplete view)
- API throws `WebApplicationException` → `Unavailable` with reason
- API throws `ProcessingException` (network) → `Unavailable` with reason

## Files Changed

| File | Change |
|------|--------|
| `domain/.../CiStatusClient.java` | New — SPI interface |
| `domain/.../CombinedCiStatus.java` | New — sealed result type |
| `github/.../GitHubChecksApi.java` | New — MP REST Client interface |
| `github/.../CheckSuitesResponse.java` | New — API response record |
| `github/.../GitHubCiStatusClient.java` | New — SPI implementation |
| `github/.../GitHubCiStatusClientTest.java` | New — unit tests |
| `app/.../PrReviewCaseService.java` | Modified — remove in-memory map, inject and call `CiStatusClient` |
| `app/.../PrReviewCaseServiceCiStatusTest.java` | Modified — mock `CiStatusClient`, update/remove tests |

## Notes

**§8 anti-pattern (out of scope):** `PrReviewCaseService` uses `@Alternative @Priority(2)` — the exact CDI priority gymnastics that ARC42STORIES §8 calls out as an anti-pattern. The correct pattern is `@ApplicationScoped` without `@DefaultBean` (CDI displacement handles the rest). This predates this spec and is orthogonal, but worth a drive-by fix during implementation since we're already modifying the class.
