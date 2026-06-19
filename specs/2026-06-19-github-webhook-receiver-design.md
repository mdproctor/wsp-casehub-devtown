# GitHub Webhook Receiver — Design Spec

**Issue:** devtown#15 (Epic 8: GitHub integration)
**Scope:** Webhook receiver for PR events (opened/updated/closed). Not CI status, merge executor, or PR analysis workers — those are separate components within #15.
**Date:** 2026-06-19

---

## Summary

A JAX-RS endpoint in the `github/` module that receives GitHub webhook POST requests for pull request events, verifies their authenticity via HMAC-SHA256 signature, parses the payload into typed domain objects, and invokes the appropriate lifecycle operation on `PrReviewApplicationService`.

---

## Architecture

```
GitHub POST → GitHubWebhookResource (github/)
           → verify signature (GitHubSignatureVerifier)
           → parse to GitHubPullRequestEvent (minimal DTO)
           → route by X-GitHub-Event header, then by action
           → map to domain types (GitHubPayloadMapper)
           → call PrReviewApplicationService (port in review/)
           → PrReviewCaseService (impl in app/) manages case lifecycle
```

Hexagonal architecture: `github/` is an inbound adapter, `PrReviewApplicationService` is the port, `PrReviewCaseService` is the implementation.

---

## Components

| Class | Module | Purpose |
|-------|--------|---------|
| `GitHubWebhookResource` | `github/` | JAX-RS endpoint at `/api/github/webhook`, event type + action routing |
| `GitHubSignatureVerifier` | `github/` | HMAC-SHA256 verification with constant-time comparison |
| `GitHubPullRequestEvent` | `github/` | Minimal nested record DTO for GitHub webhook payload |
| `GitHubPayloadMapper` | `github/` | DTO → `PrPayload` translation |
| `PrReviewApplicationService` | `review/` | Port interface — `startReview()`, `revisePr()`, `closePr()` |
| `LifecycleResult` | `review/` | Enum for lifecycle operation outcomes |
| `PrReviewCaseService` | `app/` | Implementation — case engine interaction, case lookup |
| `PrReviewCaseTracker` | `app/` | Adds `findActiveCaseByPr(repo, prNumber)` lookup |

---

## Design Decisions

### Why not `WebhookInboundConnector` from casehub-connectors

The connector SPI normalises webhooks into `InboundMessage` — a messaging abstraction with a flat `content` string field designed for Slack/Teams/IMAP messages. GitHub PR events are structured domain events (repo, PR number, action, head SHA, changed files, contributor) that directly feed case context. Forcing them through `InboundMessage` means serialising to JSON string and immediately deserialising back — a lossy intermediary that adds indirection without value.

### Why not CDI async events

`Event.fireAsync()` returns a `CompletionStage` but the webhook handler cannot wait on it and return a meaningful synchronous HTTP response to GitHub. The CDI event approach also adds event types and observers for a system with a single consumer — indirection without current benefit.

### Why `LifecycleResult` enum instead of void/exceptions

"No active case found" is a normal condition (PR created before devtown was configured, or case already expired). Throwing for expected conditions is wrong. Void hides whether the operation had effect. The enum gives the webhook handler enough information to return a meaningful response body without exposing domain internals.

---

## Event Handling

### Dispatch structure

Two-level routing: `X-GitHub-Event` header → action field.

```java
// Level 1: event type
return switch (eventType) {
    case "pull_request" -> handlePullRequest(body);
    default -> ignored(eventType);
};

// Level 2: action (within pull_request)
return switch (event.action()) {
    case "opened"          -> handleOpened(event);
    case "ready_for_review" -> handleReadyForReview(event);
    case "synchronize"     -> handleSynchronize(event);
    case "closed"          -> handleClosed(event);
    case "reopened"        -> handleReopened(event);
    default                -> ignored(event.action());
};
```

### Event → domain operation mapping

| Action | Condition | Domain operation | Port method |
|--------|-----------|-----------------|-------------|
| `opened` | `draft == false` | Start case | `startReview(PrPayload)` |
| `opened` | `draft == true` | Ignore | — |
| `ready_for_review` | — | Start case (draft→ready) | `startReview(PrPayload)` |
| `synchronize` | — | Update existing case | `revisePr(repo, prNumber, headSha, linesChanged)` |
| `closed` | `merged == true` | Mark case merged | `closePr(repo, prNumber, true)` |
| `closed` | `merged == false` | Mark case abandoned | `closePr(repo, prNumber, false)` |
| `reopened` | — | Start new case | `startReview(PrPayload)` |

### Context update on `synchronize`

Find the active case via tracker, then update case context:
- **Update:** `pr.headSha` → new SHA, `pr.linesChanged` → new count
- **Null out (stale):** `codeAnalysis`, `securityReview`, `architectureReview`, `styleCheck`, `testCoverage`, `performanceAnalysis`
- **Preserve:** human approvals, policy, memory context

Bindings re-fire because their `when` conditions check for null analysis fields (e.g., `.codeAnalysis == null`).

### Idempotency

By design: `startReview()` checks if an active case already exists for the PR (via `PrReviewCaseTracker.findActiveCaseByPr()`). If yes, treats as `synchronize`. Handles GitHub retries without delivery ID tracking.

---

## Port Interface

```java
// review/src/main/java/io/casehub/devtown/review/PrReviewApplicationService.java
public interface PrReviewApplicationService {
    PrReviewOutcome startReview(PrPayload pr);
    LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged);
    LifecycleResult closePr(String repo, int prNumber, boolean merged);
}
```

```java
// review/src/main/java/io/casehub/devtown/review/LifecycleResult.java
public enum LifecycleResult {
    UPDATED,           // case found and context updated
    NO_ACTIVE_CASE,    // no running case for this PR
    ALREADY_TERMINAL   // case already completed or abandoned
}
```

Rename: `review(PrPayload)` → `startReview(PrPayload)`. All callers updated.

---

## GitHub Payload DTO

Minimal nested records in `github/`. Snake_case field names matching GitHub's JSON — these are boundary types, never leak into the domain.

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubPullRequestEvent(
    String action,
    int number,
    PullRequest pull_request,
    Repository repository
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record PullRequest(
        Head head, Base base, User user,
        boolean draft, boolean merged,
        int additions, int deletions, int changed_files
    ) {
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Head(String sha) {}
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Base(String ref) {}
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record User(String login) {}
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Repository(String full_name) {}
}
```

### Mapping to `PrPayload`

```
repository.full_name        → repo
number                      → prNumber
pull_request.head.sha       → headSha
pull_request.base.ref       → baseRef
pull_request.additions +
  pull_request.deletions    → linesChanged
pull_request.user.login     → contributor
(not available from webhook) → changedPaths = List.of()
```

`changedPaths` is empty from the webhook path. The code-analysis capability worker fetches files via GitHub API as part of its work.

---

## Signature Verification

Pure Java class. No CDI, no Quarkus — testable in isolation.

- Compute HMAC-SHA256 of raw body bytes using configured webhook secret
- Compare with `X-Hub-Signature-256` header value (format: `sha256=<hex>`)
- Use `MessageDigest.isEqual()` for constant-time comparison (per PP-20260529-b7765c)
- Parse the `sha256=` prefix; reject if missing or malformed

---

## HTTP Response Design

| Scenario | Status | Response body |
|----------|--------|---------------|
| Bad/missing signature | 401 | `{"error": "invalid signature"}` |
| Unhandled event type | 200 | `{"status": "ignored", "event": "<type>"}` |
| Unhandled action | 200 | `{"status": "ignored", "action": "<action>"}` |
| Draft PR opened | 200 | `{"status": "ignored", "reason": "draft"}` |
| Case started | 200 | `{"status": "accepted", "action": "case-started"}` |
| Case updated | 200 | `{"status": "accepted", "action": "case-updated"}` |
| Case closed | 200 | `{"status": "accepted", "action": "case-closed"}` |
| No active case for update/close | 200 | `{"status": "accepted", "action": "no-active-case"}` |
| Processing error (JSON parse failure, service exception) | 200 | `{"status": "error"}` + log at ERROR level internally |

Always 200 except for signature failure. GitHub retries on 5xx — we don't want retries because the case engine handles its own error recovery. Response bodies appear in GitHub's webhook delivery logs for debugging.

---

## Configuration

```properties
devtown.github.webhook-secret=${GITHUB_WEBHOOK_SECRET:}
```

Single `@ConfigProperty`. Environment variable override. No per-repo secrets, no PreferenceProvider — single-tenant for now.

---

## Module Dependencies

```
github/ depends on:
  - devtown-domain (vocabulary types)
  - devtown-review (PrReviewApplicationService port, PrPayload, LifecycleResult)
  - quarkus-rest-client + jackson (already declared — outbound API for future workers)
  - jackson-databind (for @JsonIgnoreProperties — transitive via quarkus-rest-client-jackson)

app/ depends on:
  - casehub-devtown-github (new — adds the github module to the app classpath)
```

The `github/` module defines JAX-RS resources using `jakarta.ws.rs-api` annotations (transitive dep from `quarkus-rest-client`). Quarkus discovers them when the module is on `app/`'s classpath.

---

## Edge Cases

**Fork PRs:** `pull_request.head.sha` points to a commit in the fork. The receiver just records the SHA. Fork-aware API access is the code-analysis worker's concern.

**Multi-tenancy:** Webhooks have no user session. The service implementation uses a configured default tenant ID. Per-repo tenant mapping can be added later.

**Concurrent `opened` deliveries (retry):** Idempotent by design — `startReview()` checks for existing active case and treats as `synchronize` if found. Practically impossible race; harmless if it occurs.

---

## Testing Strategy

All pure unit tests — no `@QuarkusTest` (devtown#83 blocks integration testing).

| Component | What to test |
|-----------|-------------|
| `GitHubSignatureVerifier` | Valid signature, invalid signature, missing header, malformed prefix, empty body |
| `GitHubPullRequestEvent` parsing | Sample webhook JSON for each action type. Unknown fields ignored. |
| `GitHubPayloadMapper` | Field mapping correctness. `changedPaths` is empty list. `linesChanged = additions + deletions`. |
| `GitHubWebhookResource` | Mock `PrReviewApplicationService`. Correct method called per action. Draft filtering. Signature rejection (401). Unhandled events return ignored. |

---

## Out of Scope

- **CI status integration** (`check_suite` / `check_run` events) — separate component within #15
- **Merge execution** (GitHub API merge call) — separate component within #15
- **PR diff analysis** (fetch changed files) — code-analysis capability worker
- **GitHub App JWT auth** — needed for outbound API calls, not for receiving webhooks
- **PR governance dashboard** — devtown#85
- **`pull_request_review` events** (external review tracking) — future enhancement
