# GitHub Webhook Receiver — Design Spec

**Issue:** devtown#15 (Epic 8: GitHub integration)
**Scope:** Webhook receiver for PR events (opened/updated/closed). Not CI status, merge executor, or PR analysis workers — those are separate components within #15.
**Date:** 2026-06-19
**Rev:** 4 (post-review round 3)

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
| `PrReviewCaseTracker` | `app/` | Adds `findActiveCaseByPr()` lookup with secondary index |

---

## Design Decisions

### Why not `WebhookInboundConnector` from casehub-connectors

GitHub PR webhooks are **domain events**, not messages. `casehub-connectors` is foundation infrastructure for generic messaging — Slack, Teams, IMAP. It belongs to the messaging abstraction boundary. PR lifecycle events (`opened`, `synchronize`, `closed`) are devtown application-tier domain logic that directly drives case lifecycle. They belong in the application tier, not the connector infrastructure.

Even if `InboundMessage` had a structured `Map<String,Object>` payload instead of a flat `content` string, the connector approach would still be wrong: it routes through `InboundConnectorService.receive()` → async `CDI Event<InboundMessage>` → an observer that reconstitutes the domain intent. Three hops through foundation messaging infrastructure for a direct domain operation. The architectural cost of violating the domain/messaging boundary is not justified by the small convenience of reusing header normalization and error handling that are trivially solvable in a JAX-RS endpoint.

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
| `closed` | `merged == true` | Mark case externally merged | `closePr(repo, prNumber, true)` |
| `closed` | `merged == false` | Mark case abandoned | `closePr(repo, prNumber, false)` |
| `reopened` | — | Start new case | `startReview(PrPayload)` |

### Context update on `synchronize`

Find the active case via tracker, then signal context updates.

**Signal ordering contract:** PR metadata signals MUST complete before analysis invalidation signals. The `initial-analysis` binding fires on `.codeAnalysis == null` and reads `pr.headSha` as input (`{ pr: .pr }`). Reversing the order causes re-analysis of stale code. Vert.x EventBus `request()` processes signals sequentially per call — code order equals execution order — but this ordering is a correctness requirement, not an implementation detail. A refactor that reorders these calls breaks correctness silently.

**Required signal order:**

1. `signal(caseId, "pr.headSha", newSha)` — update metadata FIRST
2. `signal(caseId, "pr.linesChanged", newLines)` — update metadata
3. `signal(caseId, "codeAnalysis", null)` — invalidation triggers binding with CURRENT metadata
4. `signal(caseId, "securityReview", null)` — order among remaining invalidations is irrelevant
5. `signal(caseId, "architectureReview", null)` — all guard on `.codeAnalysis.complete == true`
6. `signal(caseId, "styleCheck", null)`
7. `signal(caseId, "testCoverage", null)`
8. `signal(caseId, "performanceAnalysis", null)`

**Preserve: human approvals.** This is an explicit policy choice. Rationale:
- Human approval is expensive — SLA-bounded, consumes reviewer time. A fixup commit should not invalidate a thorough human review.
- The re-triggered code-analysis detects if new changes alter the review scope (new security-sensitive files, new API crossings). If scope changes, the relevant specialist binding fires regardless of existing approvals.
- The audit trail records the approval SHA vs merge SHA. The compliance report captures the delta naturally.
- Mirrors Gastown's default: approvals persist across pushes unless explicitly dismissed.
- **Configurable in future:** per-repo policy via `PreferenceProvider` when scoped preferences are wired (parent#26). Teams that want stale-approval dismissal can opt in.

**Engine mechanism (verified):** `CaseHubRuntime.signal(caseId, "codeAnalysis", null)` sets the context key to null via `WritablePanelImpl.setPath()`. The null value increments the context version, triggering `contextChange` binding re-evaluation. JQ condition `.codeAnalysis == null` evaluates true for both null-valued keys and absent keys. DEEP_MERGE does not interfere — it is the conflict resolver strategy on binding **outputs** (worker results merged into context), not on signals. Signals bypass the conflict resolver entirely.

This is the canonical **context invalidation pattern** for CaseHub harnesses: signal the key to null, let bindings re-fire against the null check.

### Prerequisite fix: mutable sub-maps in initial context

**`PrReviewCaseService.review()` must use mutable maps for sub-map values.** The current code constructs `prContext` with `Map.of(...)` (unmodifiable). `WritablePanelImpl` copies the top-level `data` via `putAll(initial)` but does NOT deep-copy sub-maps. The `pr` key still points to the original `Map.of()` instance. Sub-path signals like `signal(caseId, "pr.headSha", newSha)` navigate into the sub-map and call `current.put("headSha", newSha)` on the unmodifiable map — `UnsupportedOperationException` at runtime.

**Fix in `PrReviewCaseService`:**
```java
// Before (broken for sub-path signals):
var prContext = Map.<String, Object>of("id", ..., "repo", ..., ...);

// After (mutable — supports sub-path signals):
var prContext = new LinkedHashMap<>(Map.of("id", ..., "repo", ..., ...));
```

This is also an engine bug — `WritablePanelImpl` should defensively deep-copy initial data to ensure all sub-maps are mutable. Filed as engine#NNN. The devtown fix is the immediate resolution; the engine fix prevents all future harnesses from hitting the same issue.

Top-level signals (e.g., `signal(caseId, "codeAnalysis", null)`) are unaffected — they call `this.data.put(key, value)` on the top-level `LinkedHashMap`, which is always mutable.

### Context update on `closed`

- **`merged == false`:** Signal `pr.status = "closed"`. The `review-abandoned` failure goal (`'.pr.status == "closed" or .pr.status == "superseded"'`) triggers. Case completes via failure path.
- **`merged == true`:** Signal `pr.status = "merged"`. All three success goals short-circuit to satisfied (each includes `.pr.status == "merged" or ...`). Case completes via success path. See CasePlanModel update below.

### CasePlanModel update — external merge completion path

The current success goals and completion spec have no path for externally merged PRs. Signaling `pr.status = "merged"` matches neither the success goals (they check review approvals + CI) nor the failure goals (they check for "closed"/"superseded"). The case would be stuck.

**Fix: add `.pr.status == "merged" or` to each success goal's JQ condition.** When a PR is externally merged, all three success goals short-circuit to satisfied, and the existing `allOf: [pr-approved, security-verified, ci-passing]` completion spec triggers without modification.

```yaml
- name: pr-approved
  kind: success
  condition: >-
    .pr.status == "merged" or
    ((.codeAnalysis.securitySensitive == false or .securityReview.outcome == "APPROVED") and
     (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
     .styleCheck.outcome == "APPROVED" and
     .testCoverage.outcome == "APPROVED" and
     .performanceAnalysis.outcome == "APPROVED")

- name: security-verified
  kind: success
  condition: >-
    .pr.status == "merged" or
    .codeAnalysis.securitySensitive == false or
    .securityReview.outcome == "APPROVED"

- name: ci-passing
  kind: success
  condition: '.pr.status == "merged" or .ci.status == "passing"'
```

Completion spec unchanged: `allOf: [pr-approved, security-verified, ci-passing]`.

**Why this approach instead of a separate `externally-merged` goal:** The engine's `GoalExpression` schema supports `List<String> allOf` and `List<String> anyOf` as flat lists of goal names — no nested composition (`anyOf(allOf(...), goal)` is not expressible). A separate goal would require composed completion expressions the engine doesn't support. The right design is a separate goal with engine support for composed expressions — filed as engine#NNN. The `.pr.status == "merged" or` approach is the pragmatic solution within the current engine.

**Observability impact:** `pr-approved: true` when the PR was externally merged, even though no formal approvals were received. Semantically defensible: merge access implies approval authority. The audit trail distinguishes external merge (webhook `closed` with `merged=true`) from merge-executor binding — the distinction is in the event log, not in goal satisfaction.

### `reopened` — old and new case relationship

1. `findActiveCaseByPr()` returns only non-terminal cases (`CaseTrackingStatus.isTerminal()` already supports this)
2. The old case remains in the primary `cases` map with terminal status — historical record, not deleted
3. The new case has no relationship to the old case — a reopened PR is a fresh review
4. If linking is needed, that's the governance dashboard (devtown#85)

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
    UPDATED,              // case found and context updated
    NO_ACTIVE_CASE,       // no running case for this PR
    ALREADY_COMPLETED,    // case already reached success (merged or all goals passed)
    ALREADY_ABANDONED     // case already reached failure (closed, rejected, blocked)
}
```

**Note on `ALREADY_COMPLETED` / `ALREADY_ABANDONED`:** These values are forward-compatible for devtown#80 (persistent store). With the current in-memory tracker, both are unreachable — the `prIndex` entry is removed on terminal status, so `findActiveCaseByPr()` returns empty for terminal cases and the service returns `NO_ACTIVE_CASE`. When the persistent store ships, `CaseInstanceRepository` queries can distinguish terminal states, making these values producible.

### `@DefaultBean` displacement — all implementations updated

Rename: `review(PrPayload)` → `startReview(PrPayload)`. All three implementations updated:

| Implementation | Layer | New lifecycle method behavior |
|---------------|-------|-------------------------------|
| `PrReviewService @DefaultBean` | L1 | `startReview()` keeps existing naive logic. `revisePr()` → `NO_ACTIVE_CASE`. `closePr()` → `NO_ACTIVE_CASE`. Baseline has no case tracking. |
| `QhorusPrReviewService @Priority(1)` | L3 | Same as baseline — no case tracking in this layer. `revisePr()` → `NO_ACTIVE_CASE`. `closePr()` → `NO_ACTIVE_CASE`. |
| `PrReviewCaseService @Priority(2)` | L5 | Real implementation — case lookup, context update via engine signals. |

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

### Configuration wiring

The secret is injected via `@ConfigProperty` into `GitHubWebhookResource` and passed as a method parameter to the verifier:

```java
GitHubSignatureVerifier.verify(rawBody, signatureHeader, webhookSecret)
```

The verifier has no field state — it is a stateless utility with a static method (or stateless instance). No CDI, no injection. This makes it testable with plain JUnit — pass known secret, body, and expected HMAC.

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
| Case closed (merged) | 200 | `{"status": "accepted", "action": "case-merged"}` |
| Case closed (abandoned) | 200 | `{"status": "accepted", "action": "case-abandoned"}` |
| No active case for update/close | 200 | `{"status": "accepted", "action": "no-active-case"}` |
| Already completed | 200 | `{"status": "accepted", "action": "already-completed"}` |
| Already abandoned | 200 | `{"status": "accepted", "action": "already-abandoned"}` |
| Processing error (JSON parse, NPE, unexpected) | 500 | `{"error": "internal"}` + log raw payload at ERROR |

**200 for expected no-ops; 500 for genuine exceptions.** The spec's idempotency design makes retries safe — `startReview` deduplicates, `revisePr`/`closePr` are naturally idempotent. GitHub retries on 5xx, which is correct: it applies pressure to fix bugs rather than silently losing events. Raw payload is logged at ERROR on 500 for manual replay capability.

---

## Configuration

```properties
devtown.github.webhook-secret=${GITHUB_WEBHOOK_SECRET:}
```

Single `@ConfigProperty`. Environment variable override. No per-repo secrets, no PreferenceProvider — single-tenant for now.

---

## Case Tracker — Secondary Index

`PrReviewCaseTracker` adds a secondary `ConcurrentHashMap<String, UUID>` keyed by `repo + "#" + prNumber` → case ID. Updated on `register()`. Removed in `onCaseLifecycle()` when `CaseTrackingStatus.fromCaseStatus(event.caseStatus()).isTerminal()` — the same `@ObservesAsync CaseLifecycleEvent` observer that already calls `updateStatus()`. No separate cleanup method, no consistency gap.

```java
public Optional<CaseInfo> findActiveCaseByPr(String repo, int prNumber) {
    UUID caseId = prIndex.get(repo + "#" + prNumber);
    if (caseId == null) return Optional.empty();
    CaseInfo info = cases.get(caseId);
    if (info == null || info.status().isTerminal()) return Optional.empty();
    return Optional.of(info);
}
```

**Known limitation:** The tracker is in-memory. Server restarts lose all tracking. `revisePr()` and `closePr()` return `NO_ACTIVE_CASE` after restart. Resolution: devtown#80 (production persistence backend) + `CaseInstanceRepository` query for active cases by context field. Not in scope for this spec.

---

## Module Dependencies

```
github/ depends on:
  - devtown-domain (vocabulary types)
  - devtown-review (PrReviewApplicationService port, PrPayload, LifecycleResult)
  - jakarta.ws.rs-api (explicit provided scope — JAX-RS resource annotations)
  - jackson-databind (transitive via quarkus-rest-client-jackson — @JsonIgnoreProperties)
  - quarkus-rest-client + jackson (already declared — outbound API for future workers)

app/ depends on:
  - casehub-devtown-github (new — adds the github module to the app classpath)
```

`jakarta.ws.rs-api` is declared as an explicit `provided`-scope dependency in `github/pom.xml`. The annotations are technically available transitively from `quarkus-rest-client`, but that transitive path is incidental (outbound REST client happens to share the same API jar as inbound JAX-RS). Making it explicit documents intent: this module defines JAX-RS resources.

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
| `GitHubSignatureVerifier` | Valid signature, invalid signature, missing header, malformed prefix, empty body, timing-safe comparison (does not short-circuit) |
| `GitHubPullRequestEvent` parsing | Sample webhook JSON for each action type. Unknown fields ignored. Nested records parsed correctly. |
| `GitHubPayloadMapper` | Field mapping correctness. `changedPaths` is empty list. `linesChanged = additions + deletions`. |
| `GitHubWebhookResource` | Mock `PrReviewApplicationService`. Correct method called per action. Draft filtering. Signature rejection (401). Unhandled events return 200 ignored. Processing exceptions return 500. |
| `PrReviewCaseService.revisePr()` | Signal ordering: recording mock of `CaseHubRuntime.signal()` captures `(path, value)` pairs and asserts `pr.headSha` and `pr.linesChanged` are signaled BEFORE `codeAnalysis` is nulled. |

---

## Issues to File

| Target repo | Issue | Reason |
|-------------|-------|--------|
| casehub-engine | `WritablePanelImpl` should deep-copy initial sub-maps | `Map.of()` sub-map values cause `UnsupportedOperationException` on sub-path signals — any harness can hit this |
| casehub-engine | Composed `GoalExpression` — nested `anyOf(allOf(...), goal)` | Enables separate `externally-merged` goal without polluting individual goal conditions |

---

## Out of Scope

- **CI status integration** (`check_suite` / `check_run` events) — separate component within #15
- **Merge execution** (GitHub API merge call) — separate component within #15
- **PR diff analysis** (fetch changed files) — code-analysis capability worker
- **GitHub App JWT auth** — needed for outbound API calls, not for receiving webhooks
- **PR governance dashboard** — devtown#85
- **`pull_request_review` events** (external review tracking) — future enhancement
- **Webhook secret rotation** — GitHub supports dual-signature during rotation window (both old and new `X-Hub-Signature-256` headers sent). Production receivers should verify against both secrets. Track as a future hardening task.
