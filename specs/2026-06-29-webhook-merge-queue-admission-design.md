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
    AdmissionResult admit(int prNumber, String repository, String headSha, String author);
}

// merge/src/main/java/io/casehub/devtown/merge/AdmissionResult.java
public enum AdmissionResult {
    ENQUEUED,
    ALREADY_QUEUED
}
```

**Why `merge/`:** This is a hexagonal port — the compile-time interface that lets `github/` reach `MergeQueueService` (in `app/`) without depending on `app/`. It follows the standard devtown pattern: port in the domain module, implementation in the runtime module. The other two admission paths (`PrReviewMergeQueueAdapter`, `DevtownMcpTools`) are already in `app/` and call `MergeQueueService.enqueue()` directly — they don't need the port because they're inside the implementation module.

**Why these params:** The webhook doesn't know trust score, priority lane, or dependencies. Those are domain decisions for the implementation. The port takes raw identity data; the adapter stays thin.

**Why `AdmissionResult`:** Matches the established webhook handler pattern — every `handle*` method in `GitHubWebhookResource` returns differentiated responses. `ENQUEUED` vs `ALREADY_QUEUED` gives the response body enough information without exposing domain internals. Consistent with `LifecycleResult` on the PR lifecycle path.

### Implementation (`MergeQueueService`)

`MergeQueueService` implements `MergeQueueAdmissionPort` directly — no new class:

```java
@ApplicationScoped
public class MergeQueueService implements MergeQueueAdmissionPort {

    @Override
    public AdmissionResult admit(int prNumber, String repository, String headSha, String author) {
        QueuedPr pr = new QueuedPr(prNumber, repository, headSha, author,
            0.5, PriorityLane.NORMAL, Instant.now(), Set.of());
        boolean inserted = enqueue(pr);
        return inserted ? AdmissionResult.ENQUEUED : AdmissionResult.ALREADY_QUEUED;
    }
}
```

Defaults match the CasePlanModel path (`PrReviewMergeQueueAdapter`: trustScore=0.5, lane=NORMAL). `enqueue()` handles WorkItem creation, persistence, and batch formation. Returns `false` when the entry already exists (idempotent duplicate).

**Latency note:** `enqueue()` calls `formAndDispatchBatches()` synchronously, which acquires a `SELECT FOR UPDATE` lock and may start a `CaseInstance`. For a single PR admission, this typically completes in <1 second. GitHub's 10-second webhook timeout is not at risk. If the handler does time out, GitHub retries — safe because `enqueue()` is idempotent. Async batch formation is a future optimization if latency becomes a measured concern.

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

    var result = admissionPort.admit(
        event.number(),
        event.repository().full_name(),
        event.pull_request().head().sha(),
        event.pull_request().user().login()
    );

    String action = switch (result) {
        case ENQUEUED -> "merge-queue-enqueued";
        case ALREADY_QUEUED -> "already-queued";
    };
    return ok(Map.of("status", "accepted", "action", action));
}
```

Three guards: wrong label → ignore, draft → ignore, otherwise enqueue. Response body differentiates `ENQUEUED` from `ALREADY_QUEUED` — consistent with the `lifecycleAction()` pattern used by all other `handle*` methods. Idempotent — duplicate events and GitHub retries are safe.

### Module dependencies

```
github/ adds:
  - devtown-merge (new — for MergeQueueAdmissionPort)

app/ already depends on:
  - devtown-merge (for MergeQueueStore — no change)
```

## Tests

**Unit (github — `GitHubWebhookResourceTest`):**
- `labeled` + `merge-ready` → 200 accepted, action `merge-queue-enqueued`, `admissionPort.admit()` called with correct args
- `labeled` + `merge-ready` + already queued → 200 accepted, action `already-queued`
- `labeled` + different label → 200 ignored, `admissionPort` not called
- `labeled` + null label → 200 ignored
- `labeled` + draft PR → 200 ignored, `admissionPort` not called
- `labeled` + bad signature → 401 (existing pattern)

**Unit (github — `GitHubPullRequestEventTest`):**
- Parse `labeled` webhook JSON — `label.name()` extracted
- Parse `opened` webhook JSON — `label()` is null (no regression)

**Existing test impact:** Adding `Label label` to the `GitHubPullRequestEvent` record changes the canonical constructor. However, existing tests in `GitHubWebhookResourceTest` construct events via JSON strings (`prEvent()` helper), not via the record constructor — Jackson handles the missing `label` field by passing `null`. The existing `unknownAction_returns200Ignored` test (which uses `"labeled"` as action) verifies backward compatibility. No existing tests break; new tests for the `labeled` action path are additive.

**Integration (app — merge queue tests):**
- CDI resolves `MergeQueueAdmissionPort` to `MergeQueueService`
- `admit()` creates a queue entry via `enqueue()`, returns `ENQUEUED`
- `admit()` is idempotent — second call for same PR/repo returns `ALREADY_QUEUED`

## Design Decisions — Deferred Items

### Approval/CI validation at admission — intentional divergence from §3.1

The parent merge queue design (§3.1) states the webhook "validates approval + CI status before enqueuing." This spec intentionally refines that requirement:

- **Label as approval signal:** The `merge-ready` label IS the human approval for this path. Adding it is a deliberate act by someone with repo access — the label is the admission credential. Programmatic GitHub approval checks would duplicate this signal and conflate CaseHub's admission model with GitHub's review model.
- **Batch tests CI:** The batch tip test validates CI against the merged batch. Pre-checking CI at admission is a point-in-time snapshot that may be stale (CI still running, flaky test about to be fixed). The batch test is definitive.
- **Parent spec update:** §3.1's "validates approval + CI status" was the initial high-level design. This spec refines it based on implementation analysis. The parent spec will be updated. **Tracked: devtown#102.**

### `unlabeled` action handling — intentional asymmetry

Removing `merge-ready` does not dequeue. This asymmetry is a deliberate design choice:

- Labels can be accidentally removed by branch protection rules, CI bots, label-management automation, or mis-clicks. Automatic dequeue on label removal risks unintended revocation of a queued PR.
- Queue revocation has cascading effects: SLA WorkItem obsolescence, dependent PR cascade dequeue, author notification. These side effects warrant a deliberate action (MCP `dequeue_pr` tool), not an accidental label change.
- The webhook path is for teams using GitHub's native flow. Those teams also have access to MCP tools for queue management — the MCP session is for the CI/tooling integration, not the individual developer.

Future enhancement: configurable per-repo `unlabeled → dequeue` behavior via PreferenceProvider. **Tracked: devtown#103.**

### Configurable label name

Hardcoded `merge-ready` for now. Configurable via PreferenceProvider in a future iteration. **Tracked: devtown#104.**
