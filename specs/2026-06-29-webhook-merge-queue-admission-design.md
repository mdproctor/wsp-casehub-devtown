# Webhook Merge Queue Admission

**Issue:** devtown#101
**Date:** 2026-06-29
**Status:** Approved

## Problem

The merge queue has three admission paths (spec §3.1):
1. PR Review Case completion → `enqueue-for-merge` binding ✅ (devtown#100)
2. MCP tool `enqueue_pr` ✅ (devtown#11)
3. GitHub webhook `pull_request.labeled` with `merge-ready` — **not yet implemented**

Path 3 is for PRs not managed by the PR Review Case — teams using GitHub's native review flow that want merge queue benefits (batch testing, bisection, trust-weighted ordering) without CaseHub PR review.

## Design

### Port interface (`merge/`)

```java
// merge/src/main/java/io/casehub/devtown/merge/MergeQueueAdmissionPort.java
public interface MergeQueueAdmissionPort {
    void admit(int prNumber, String repository, String headSha, String author);
}
```

**Why `merge/`:** The merge queue module defines the merge queue's public API — both consumer-facing (this port) and provider-facing (`MergeQueueStore` SPI). This mirrors `casehub-work-api` which has both consumer and provider types. `review/` would be the wrong domain — the merge queue spec explicitly separates these concerns.

**Why these params:** The webhook doesn't know trust score, priority lane, or dependencies. Those are domain decisions for the implementation. The port takes raw identity data; the adapter stays thin.

**Why void:** `MergeQueueStore.enqueue()` is idempotent. The webhook returns 200 regardless. No domain outcome to communicate.

### Implementation (`MergeQueueService`)

`MergeQueueService` implements `MergeQueueAdmissionPort` directly — no new class:

```java
@ApplicationScoped
public class MergeQueueService implements MergeQueueAdmissionPort {

    @Override
    public void admit(int prNumber, String repository, String headSha, String author) {
        QueuedPr pr = new QueuedPr(prNumber, repository, headSha, author,
            0.5, PriorityLane.NORMAL, Instant.now(), Set.of());
        enqueue(pr);
    }
}
```

Defaults match the CasePlanModel path (`PrReviewMergeQueueAdapter`: trustScore=0.5, lane=NORMAL). `enqueue()` handles WorkItem creation, persistence, and batch formation.

### Webhook payload (`GitHubPullRequestEvent`)

GitHub's `pull_request.labeled` event includes a top-level `label` object:
```json
{"action": "labeled", "label": {"name": "merge-ready"}, "pull_request": {...}}
```

Add to the existing record:
```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubPullRequestEvent(
    String action,
    int number,
    PullRequest pull_request,
    Repository repository,
    Label label
) {
    // existing nested records unchanged

    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Label(String name) {}
}
```

`label` is nullable — `@JsonIgnoreProperties(ignoreUnknown = true)` on the outer record means Jackson sets it to `null` for all other actions. No impact on existing handlers.

### Webhook handler (`GitHubWebhookResource`)

**New dependency:**
```java
@Inject
MergeQueueAdmissionPort admissionPort;
```

`github/pom.xml` adds a compile dependency on `devtown-merge`.

**New action case** in the `handlePullRequest` switch:
```java
case "labeled" -> handleLabeled(event);
```

**Handler method:**
```java
private Response handleLabeled(GitHubPullRequestEvent event) {
    if (event.label() == null || !"merge-ready".equals(event.label().name())) {
        return ok(Map.of("status", "ignored", "action", "labeled",
                         "reason", "not-merge-ready"));
    }
    if (event.pull_request().draft()) {
        return ok(Map.of("status", "ignored", "reason", "draft"));
    }

    admissionPort.admit(
        event.number(),
        event.repository().full_name(),
        event.pull_request().head().sha(),
        event.pull_request().user().login()
    );

    return ok(Map.of("status", "accepted", "action", "merge-queue-enqueued"));
}
```

Three guards: wrong label → ignore, draft → ignore, otherwise enqueue. Idempotent — duplicate events and GitHub retries are safe.

### Module dependencies

```
github/ adds:
  - devtown-merge (new — for MergeQueueAdmissionPort)

app/ already depends on:
  - devtown-merge (for MergeQueueStore — no change)
```

## Tests

**Unit (github — `GitHubWebhookResourceTest`):**
- `labeled` + `merge-ready` → 200 accepted, `admissionPort.admit()` called with correct args
- `labeled` + different label → 200 ignored, `admissionPort` not called
- `labeled` + null label → 200 ignored
- `labeled` + draft PR → 200 ignored, `admissionPort` not called
- `labeled` + bad signature → 401 (existing pattern)

**Unit (github — `GitHubPullRequestEventTest`):**
- Parse `labeled` webhook JSON — `label.name()` extracted
- Parse `opened` webhook JSON — `label()` is null (no regression)

**Integration (app — merge queue tests):**
- CDI resolves `MergeQueueAdmissionPort` to `MergeQueueService`
- `admit()` creates a queue entry via `enqueue()`
- `admit()` is idempotent — second call for same PR/repo is no-op

## Not in scope

- Configurable label name (hardcoded `merge-ready` for now)
- Approval/CI validation at admission (the batch tests CI; the label is the human signal)
- `unlabeled` action handling (removing `merge-ready` does not dequeue — dequeue is explicit via MCP)
