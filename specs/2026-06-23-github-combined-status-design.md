# GitHub Combined-Status Check — Design Spec

**Issue:** devtown#89
**Date:** 2026-06-23
**Status:** Approved

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
    CombinedCiStatus getCombinedStatus(String repo, String headSha);
}
```

`repo` is `"owner/repo"` format, consistent with how repo identifiers flow through the rest of the system. The implementation splits internally.

```java
package io.casehub.devtown.domain;

public sealed interface CombinedCiStatus {
    record Passing() implements CombinedCiStatus {}
    record Failing(String summary) implements CombinedCiStatus {}
    record Pending(int completed, int total) implements CombinedCiStatus {}
}
```

- `Passing` — all suites complete and successful
- `Failing(summary)` — at least one suite failed; summary describes what (e.g. "2 of 5 suites failed")
- `Pending(completed, total)` — suites still running; counts for logging

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
        @PathParam("ref") String ref);
}
```

`CheckSuitesResponse` maps the GitHub response:
- `total_count: int`
- `check_suites: List<CheckSuite>` where each has `id: long`, `status: String` ("queued"/"in_progress"/"completed"), `conclusion: String` (null/"success"/"failure"/"neutral"/etc.)

**Implementation** — `GitHubCiStatusClient` (`@ApplicationScoped`, implements `CiStatusClient`):

- Splits `repo` into owner/repo
- Calls `api.listCheckSuites(owner, repo, headSha)`
- If `total_count == 0` → `Pending(0, 0)` (no CI configured, don't signal passing)
- If any suite has `status != "completed"` → `Pending(completedCount, totalCount)`
- If all completed and any has `conclusion != "success"` → `Failing(summary)`
- If all completed and all `conclusion == "success"` → `Passing()`

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
- After recording the individual suite, call `ciStatusClient.getCombinedStatus(repo, headSha)`
- Signal based on result:
  - `Passing` → `caseHub.signal(caseId, "ci.status", "passing")`
  - `Failing` → `caseHub.signal(caseId, "ci.status", "failing")`
  - `Pending` → no `ci.status` signal (wait for next webhook)

### Edge Cases

**Pagination:** GitHub paginates check-suites at 30 results. The initial implementation logs a warning if `total_count` exceeds the returned list size. Pagination support can be added if a repo actually has >30 suites per commit.

**Zero check suites:** `total_count: 0` → `Pending(0, 0)`. Don't signal passing when nothing has run.

## Test Strategy

### `PrReviewCaseServiceCiStatusTest` (update existing)

Mock `CiStatusClient` alongside the existing mock `PrReviewCaseHub`:

- `signalCiStatus_githubConfirmsPassing` — returns `Passing` → signals "passing"
- `signalCiStatus_githubSaysPending` — returns `Pending(1, 3)` → no `ci.status` signal
- `signalCiStatus_githubConfirmsFailing` — returns `Failing` → signals "failing"
- `signalCiStatus_staleSha_returnsStaleEvent` — unchanged, `CiStatusClient` never called
- `signalCiStatus_noActiveCase_returnsNoActiveCase` — unchanged, `CiStatusClient` never called

Remove tests that depend on the in-memory map (multi-suite aggregation, sticky-failure policy).

### `GitHubCiStatusClientTest` (new)

Mock `GitHubChecksApi` responses:

- All suites completed + success → `Passing`
- All completed, one failure → `Failing` with summary
- Some suites in_progress → `Pending` with counts
- Zero suites → `Pending(0, 0)`

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
