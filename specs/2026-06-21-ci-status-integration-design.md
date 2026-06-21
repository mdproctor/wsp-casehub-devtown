# CI Status Integration — check_suite/check_run webhook events

**Issue:** devtown#86
**Date:** 2026-06-21
**Status:** Approved

## Problem

The `ci-passing` goal in the PR review CasePlanModel checks `.ci.status == "passing"` but no external signal populates this value. GitHub Actions CI runs independently and reports results via `check_suite` and `check_run` webhook events. Without this integration, the ci-passing goal never resolves from real CI data.

## Architecture

Same hexagonal adapter pattern as the existing `pull_request` webhook handler in the `github/` module. No new modules — extends the existing webhook resource.

### check_suite — CI goal resolution

`check_suite` carries the aggregate CI result. When the suite completes, the handler signals `ci.status` on the active case.

**Event flow:**

1. GitHub sends `check_suite` with `action: "completed"`
2. `GitHubWebhookResource` dispatches to `handleCheckSuite()`
3. Extract `conclusion`, `head_sha`, and `pull_requests[]` from the event
4. For each PR in the suite: find the active case via `PrReviewApplicationService.signalCiStatus()`
5. SHA guard: only signal if the suite's `head_sha` matches the case's current `pr.headSha`
6. Signal `ci.status = "passing"` if `conclusion == "success"`, else `ci.status = "failing"`

**Signal semantics:**

- `"passing"` → satisfies `ci-passing` goal, unblocks merge binding
- `"failing"` → does NOT satisfy goal, but DOES prevent the `run-ci` binding from re-firing (`.ci != null`)
- On PR synchronize (new push), `revisePr()` nulls `.ci` — the cycle restarts

### check_run — per-job visibility

`check_run` events carry individual job results (lint, test, build). They do not change `ci.status` — that is `check_suite`'s role. They enrich the case context with per-job detail.

**Event flow:**

1. GitHub sends `check_run` with `action: "completed"`
2. Handler extracts `name`, `conclusion`, `head_sha`, `pull_requests[]`
3. For each PR: find active case, apply SHA guard
4. Signal `ci.checks.<name>` with `{ conclusion, completedAt }`

**Value:** MCP tools (`inspect_review`) gain per-job visibility. Future bindings can react to specific check failures (e.g., "if lint failed, dispatch style-review with reduced scope").

## SHA guard

GitHub can deliver a `check_suite`/`check_run` for a previous commit after a new push. Without SHA matching, a passing suite for the OLD commit would incorrectly satisfy the goal for NEW code.

The guard: `PrReviewCaseService.signalCiStatus()` reads the current `pr.headSha` from the case tracker and rejects signals where the event's `head_sha` does not match.

## DTOs

### GitHubCheckSuiteEvent

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubCheckSuiteEvent(
    String action,
    CheckSuite check_suite,
    Repository repository
) {
    public record CheckSuite(
        String conclusion, String head_sha, String status,
        List<PullRequest> pull_requests
    ) {}
    public record PullRequest(int number, Head head) {
        public record Head(String sha) {}
    }
    public record Repository(String full_name) {}
}
```

### GitHubCheckRunEvent

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubCheckRunEvent(
    String action,
    CheckRun check_run,
    Repository repository
) {
    public record CheckRun(
        String name, String status, String conclusion,
        String head_sha, List<PullRequest> pull_requests
    ) {}
    public record PullRequest(int number, Head head) {
        public record Head(String sha) {}
    }
    public record Repository(String full_name) {}
}
```

## Service layer

### PrReviewApplicationService

New methods added to the port interface:

```java
LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, String status);
LifecycleResult signalCheckRun(String repo, int prNumber, String headSha, String checkName, String conclusion);
```

### PrReviewCaseService (active implementation)

```
signalCiStatus(repo, prNumber, headSha, status):
  1. Find active case by repo+prNumber via tracker
  2. Read case's current pr.headSha from tracker payload
  3. SHA guard: if headSha != case's headSha → return NO_ACTIVE_CASE
  4. Signal ci.status = status on the case
  5. Return UPDATED
```

```
signalCheckRun(repo, prNumber, headSha, checkName, completedAt):
  1. Find active case by repo+prNumber via tracker
  2. SHA guard: if headSha != case's headSha → return NO_ACTIVE_CASE
  3. Signal ci.checks.<checkName> = { conclusion, completedAt } on the case
  4. Return UPDATED
```

### PrReviewService (@DefaultBean)

No-op: return `LifecycleResult.NO_ACTIVE_CASE` for both methods.

### QhorusPrReviewService

Delegates to `PrReviewCaseService` (inherited — no override needed since it extends the interface).

## Files touched

| File | Change |
|------|--------|
| `github/.../GitHubCheckSuiteEvent.java` | New DTO |
| `github/.../GitHubCheckRunEvent.java` | New DTO |
| `github/.../GitHubWebhookResource.java` | Add `check_suite` and `check_run` dispatch arms |
| `review/.../PrReviewApplicationService.java` | Add `signalCiStatus()` and `signalCheckRun()` |
| `app/.../PrReviewCaseService.java` | Implement with SHA guard |
| `app/.../PrReviewService.java` | No-op default |
| Tests: DTO deserialization, webhook dispatch, SHA guard, signal flow | |

## What this does NOT include

- GitHub REST client for querying CI status — webhook-push is sufficient
- CI re-trigger capability — out of scope for this issue
- Invalidation of `ci.status` on synchronize — already handled by `revisePr()` nulling `.ci`
