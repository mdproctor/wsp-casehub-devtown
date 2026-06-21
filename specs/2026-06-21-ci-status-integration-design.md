# CI Status Integration — check_suite/check_run webhook events

**Issue:** devtown#86
**Date:** 2026-06-21
**Status:** Approved (rev 2 — post review)

## Problem

The `ci-passing` goal in the PR review CasePlanModel checks `.ci.status == "passing"` but no external signal populates this value. GitHub Actions CI runs independently and reports results via `check_suite` and `check_run` webhook events. Without this integration, the ci-passing goal never resolves from real CI data.

## Architecture

Same hexagonal adapter pattern as the existing `pull_request` webhook handler in the `github/` module. No new modules — extends the existing webhook resource.

## CI lifecycle — the "pending" sentinel

CI status uses a three-state lifecycle: `pending` → `passing` | `failing`.

**On case start:** `PrReviewCaseService.startReview()` pre-seeds `.ci = { status: "pending" }` in the initial context map passed to `caseHub.startCase()`. This prevents the `run-ci` YAML binding from firing (`.ci != null`), making the webhook the sole CI authority.

**On synchronize (force-push):** `revisePr()` signals `ci` with `{ status: "pending" }` — added as a 7th invalidation signal after the existing six. Signal ordering: metadata updates (headSha, linesChanged) BEFORE all invalidations (including ci reset), consistent with the existing contract from the webhook receiver spec §"Context update on synchronize."

**Rationale for unconditional pre-seeding:** The `run-ci` binding was a placeholder for internal CI dispatch. Real deployments have external CI (GitHub Actions, Jenkins) that reports back via webhooks. The harness should always expect external CI signaling — dispatching CI as a case worker is architecturally wrong because CI is a parallel process, not a dispatched task.

**Effect on YAML:** The `run-ci` binding remains in `pr-review.yaml` but never fires. It serves as documentation of the CI capability and as a fallback path for future non-webhook CI integrations that explicitly null `.ci` to opt in.

## check_suite — CI goal resolution

`check_suite` carries the aggregate CI result per workflow. When the suite completes, the handler signals `ci.status` on the active case.

**Event flow:**

1. GitHub sends `check_suite` with `action: "completed"` (other actions: `requested`, `rerequested` — ignored)
2. `GitHubWebhookResource` dispatches to `handleCheckSuite()`
3. Extract `conclusion`, `head_sha`, `id`, and `pull_requests[]` from the event
4. If `pull_requests` is empty → return `{ "status": "ignored", "reason": "no-pull-requests" }`
5. For each PR in the suite: call `service.signalCiStatus(repo, prNumber, headSha, suiteId, conclusion)`
6. SHA guard inside `PrReviewCaseService`: reject if `head_sha` doesn't match the case's current HEAD

**Signal semantics:**

- `conclusion == "success"` AND no other received suite for this SHA has a non-success conclusion → `ci.status = "passing"`
- Any non-success conclusion → `ci.status = "failing"`
- **Failures are sticky** until the next push resets to `"pending"` — prevents the race where a passing suite triggers merge while a failing suite is still in flight

**Per-suite tracking:** Each suite result is written to `ci.suites.<suiteId>` for observability before the aggregate `ci.status` is evaluated.

### Multi-workflow aggregation

A repository with multiple GitHub Actions workflows (ci.yml, lint.yml) produces separate `check_suite:completed` events per commit. Without aggregation, the first passing suite could trigger merge while the second is still running.

**Aggregation strategy (pessimistic, webhook-only):**

After writing each suite result to `ci.suites.<id>`, evaluate the aggregate:
- If ANY received suite has a non-success conclusion → `ci.status = "failing"`
- If ALL received suites have `conclusion == "success"` → `ci.status = "passing"`
- Otherwise → leave as `"pending"`

**Known limitation:** We don't know the total expected suite count without the GitHub REST API. A "passing" status based on received suites may be premature if additional workflows exist but haven't reported yet. The pessimistic sticky-failure policy mitigates the worst case (false merge on partial CI), but a brief false-"passing" window exists.

**Follow-up:** File a separate issue for GitHub REST API combined-status verification before signaling "passing" — eliminates the aggregation gap entirely.

## check_run — per-job enrichment

`check_run` events carry individual job results (lint, test, build). They do not change `ci.status` — that is `check_suite`'s role. They enrich the case context with per-job detail.

**Event flow:**

1. GitHub sends `check_run` with `action: "completed"` (other actions: `created`, `rerequested`, `requested_action` — ignored)
2. Handler extracts `name`, `conclusion`, `completed_at`, `head_sha`, `pull_requests[]`
3. If `pull_requests` is empty → return `{ "status": "ignored", "reason": "no-pull-requests" }`
4. For each PR: call `service.signalCheckRun(repo, prNumber, headSha, name, conclusion, completedAt)`
5. SHA guard inside `PrReviewCaseService`: reject if stale

**Value:** MCP tools (`inspect_review`) gain per-job visibility. Future bindings can react to specific check failures (e.g., "if lint failed, dispatch style-review with reduced scope").

**Port method rationale:** `signalCheckRun` exists as a port method for hexagonal consistency — all case mutations go through the port. The `@DefaultBean` and `QhorusPrReviewService` implementations are no-ops, but the port boundary ensures future binding conditions can be wired without rearchitecting the adapter layer.

## SHA guard

GitHub can deliver a `check_suite`/`check_run` for a previous commit after a new push. Without SHA matching, a passing suite for the OLD commit would incorrectly satisfy the goal for NEW code.

**Data source:** The SHA guard reads the current HEAD from the case context via `caseHub.query(caseId, "pr.headSha")`, NOT from the tracker's `PrPayload`. The tracker stores the original payload at registration time — after `revisePr()` signals a new headSha, the tracker's value is stale.

**Tracker sync:** `PrReviewCaseTracker` gains an `updateHeadSha(UUID caseId, String newSha)` method. `revisePr()` calls it alongside the context signal, keeping the tracker and case context consistent.

**Return value:** When the SHA doesn't match, the method returns `LifecycleResult.STALE_EVENT` (new enum value), not `NO_ACTIVE_CASE` — the case exists but the event is outdated. The webhook handler maps this to `{ "status": "ignored", "reason": "stale-sha" }`.

## DTOs

### GitHubCheckSuiteEvent

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubCheckSuiteEvent(
    String action,
    CheckSuite check_suite,
    Repository repository
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record CheckSuite(
        long id, String conclusion, String head_sha, String status,
        List<PullRequest> pull_requests
    ) {}
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record PullRequest(int number, Head head) {
        public record Head(String sha) {}
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
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
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record CheckRun(
        String name, String status, String conclusion, String completed_at,
        String head_sha, List<PullRequest> pull_requests
    ) {}
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record PullRequest(int number, Head head) {
        public record Head(String sha) {}
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Repository(String full_name) {}
}
```

## LifecycleResult

Add `STALE_EVENT` to the enum:

```java
public enum LifecycleResult {
    UPDATED,
    NO_ACTIVE_CASE,
    ALREADY_COMPLETED,
    ALREADY_ABANDONED,
    STALE_EVENT
}
```

## Service layer

### PrReviewApplicationService

New methods added to the port interface:

```java
LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion);
LifecycleResult signalCheckRun(String repo, int prNumber, String headSha, String checkName, String conclusion, Instant completedAt);
```

### PrReviewCaseService (active implementation, @Alternative @Priority(2))

```
signalCiStatus(repo, prNumber, headSha, suiteId, conclusion):
  1. Find active case by repo+prNumber via tracker
  2. If no active case → return NO_ACTIVE_CASE
  3. Read case's current pr.headSha via caseHub.query(caseId, "pr.headSha")
  4. SHA guard: if headSha != current → return STALE_EVENT
  5. Signal ci.suites.<suiteId> = { conclusion, completedAt: now } on the case
  6. Aggregate: if ANY received suite is non-success → signal ci.status = "failing"
     else if ALL received suites are success → signal ci.status = "passing"
  7. Return UPDATED
```

```
signalCheckRun(repo, prNumber, headSha, checkName, conclusion, completedAt):
  1. Find active case by repo+prNumber via tracker
  2. If no active case → return NO_ACTIVE_CASE
  3. SHA guard via caseHub.query — if stale → return STALE_EVENT
  4. Signal ci.checks.<checkName> = { conclusion, completedAt } on the case
  5. Return UPDATED
```

### PrReviewService (@DefaultBean — Layer 1 naive fallback)

No-op: return `LifecycleResult.NO_ACTIVE_CASE` for both methods.

### QhorusPrReviewService (@Alternative @Priority(1) — Layer 3)

No-op: return `LifecycleResult.NO_ACTIVE_CASE` for both methods. Does not delegate to `PrReviewCaseService` — it is a standalone implementation of `PrReviewApplicationService`, not a subclass.

### PrReviewCaseTracker

New method:
```java
public void updateHeadSha(UUID caseId, String newSha)
```
Creates a new `CaseInfo` with an updated `PrPayload` (new headSha), preserving all other fields.

### revisePr() changes

`PrReviewCaseService.revisePr()` gains two new operations:
1. `caseHub.signal(caseId, "ci", Map.of("status", "pending"))` — 7th invalidation signal, after the existing six
2. `caseTracker.updateHeadSha(caseId, newHeadSha)` — keeps tracker consistent with case context

Signal order (extended from webhook receiver spec):
1. `signal(caseId, "pr.headSha", newSha)` — metadata FIRST
2. `signal(caseId, "pr.linesChanged", newLines)` — metadata
3. `signal(caseId, "codeAnalysis", null)` — invalidation
4–8. remaining invalidations (securityReview, architectureReview, styleCheck, testCoverage, performanceAnalysis)
9. `signal(caseId, "ci", Map.of("status", "pending"))` — CI invalidation
10. `caseTracker.updateHeadSha(caseId, newHeadSha)` — tracker sync

### startReview() changes

`PrReviewCaseService.startReview()` adds `ci` to the initial context:
```java
initialContext.put("ci", Map.of("status", "pending"));
```

## CDI wiring summary

| Class | Annotation | Priority | Active at runtime |
|-------|-----------|----------|-------------------|
| `PrReviewService` | `@DefaultBean` | lowest | No (displaced) |
| `QhorusPrReviewService` | `@Alternative @Priority(1)` | 1 | No (displaced by Priority 2) |
| `PrReviewCaseService` | `@Alternative @Priority(2)` | 2 | **Yes** |

## Files touched

| File | Change |
|------|--------|
| `github/.../GitHubCheckSuiteEvent.java` | New DTO |
| `github/.../GitHubCheckRunEvent.java` | New DTO |
| `github/.../GitHubWebhookResource.java` | Add `check_suite` and `check_run` dispatch arms with action filtering |
| `review/.../PrReviewApplicationService.java` | Add `signalCiStatus()` and `signalCheckRun()` |
| `review/.../LifecycleResult.java` | Add `STALE_EVENT` |
| `app/.../PrReviewCaseService.java` | Implement CI signal with SHA guard; pre-seed `ci` in `startReview()`; add CI invalidation + tracker sync in `revisePr()` |
| `app/.../PrReviewService.java` | No-op `signalCiStatus()` and `signalCheckRun()` |
| `app/.../QhorusPrReviewService.java` | No-op `signalCiStatus()` and `signalCheckRun()` |
| `app/.../mcp/PrReviewCaseTracker.java` | Add `updateHeadSha()` method |
| `app/.../mcp/CaseInfo.java` | Add `withHeadSha()` method |
| Tests | DTO deserialization, webhook dispatch (action filtering, empty PRs), SHA guard (from case context not tracker), CI pre-seeding, revisePr CI invalidation, multi-suite aggregation, signal flow, STALE_EVENT mapping |

## What this does NOT include

- GitHub REST client for combined-status verification — file follow-up issue for aggregation gap
- CI re-trigger capability — out of scope
- Removal of `run-ci` binding from YAML — remains as documentation and opt-in fallback
