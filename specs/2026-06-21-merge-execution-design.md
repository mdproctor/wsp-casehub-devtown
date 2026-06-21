# Merge Execution — merge-executor worker via GitHub API

**Issue:** devtown#87
**Date:** 2026-06-21
**Status:** Approved (rev 4 — final, PlanItem note, extensibility note)

## Problem

The merge binding in `pr-review.yaml` fires capability `merge-executor` when all review goals + CI passing are satisfied. No worker exists to execute the merge — `NoOpReactiveWorkerProvisioner` throws `ProvisioningException` and the binding is auto-exhausted. PRs pass review but are never merged.

## Architecture

The engine dispatches `merge-executor` to a `Worker` registered programmatically in the `CaseDefinition`. The Worker wraps a `WorkerFunction.Sync` that calls the GitHub merge API via a domain-level `MergeClient` port interface.

**Why not @Named CDI lookup:** The engine does not perform CDI lookup for workers. Worker dispatch goes through `CaseDefinition.getWorkers()` → `AgentCandidateFactory.buildCandidates()` → `AgentRoutingStrategy.select()`. A CDI bean is invisible to this chain. Workers must be registered in the CaseDefinition.

**Why capability dispatch for a local sync function:** The YAML binding + Worker model is the engine's native dispatch mechanism. Using a different mechanism for merge would be inconsistent. The merge could become an external service (merge queue, CI/CD pipeline) or a claudony agent in future deployments — the capability model enables this without rearchitecting. Trust scaffolding overhead is zero for a single-impl Worker — `AgentCandidateFactory` returns one candidate, `AgentRoutingStrategy` selects it directly.

## Worker registration

`PrReviewCaseHub` registers the merge-executor Worker eagerly in `@PostConstruct`:

```java
@Inject MergeClient mergeClient;

@PostConstruct
void registerWorkers() {
    CaseDefinition def = super.getDefinition();
    Capability mergeCap = def.getCapabilities().stream()
        .filter(c -> "merge-executor".equals(c.getName()))
        .findFirst().orElseThrow();
    def.getWorkers().add(Worker.builder()
        .name("merge-executor")
        .capabilities(mergeCap)
        .function(input -> adaptMerge(input))
        .build());
}

private WorkerResult adaptMerge(Map<String, Object> input) {
    @SuppressWarnings("unchecked")
    Map<String, Object> pr = (Map<String, Object>) input.get("pr");
    String repo = (String) pr.get("repo");
    String[] parts = repo.split("/");
    int prNumber = Integer.parseInt((String) pr.get("id"));
    String headSha = (String) pr.get("headSha");

    return switch (mergeClient.merge(parts[0], parts[1], prNumber, headSha)) {
        case MergeOutcome.Success s -> WorkerResult.of(Map.of("merge_sha", s.mergeSha()));
        case MergeOutcome.Failure f -> WorkerResult.failed(f.reason());
    };
}
```

**Why @PostConstruct, not getDefinition() override:** `YamlCaseHub.getDefinition()` uses double-checked locking, but an override's `noneMatch + add` runs outside that synchronized block. Two concurrent callers could both add the Worker. `@PostConstruct` runs once during CDI initialization on a single thread — no race.

**Initialization ordering:** `@PostConstruct` calls `super.getDefinition()`, which lazy-loads the YAML using injected `ObjectMapper` and `ExpressionEngineRegistry`. Quarkus CDI guarantees `@Inject` fields are set before `@PostConstruct` runs, so the dependencies are available.

**Input extraction:** The `adaptMerge` lambda handles the engine-specific input map structure (repo parsing, id→int conversion). This is adapter logic — it belongs in `app/`, not in the domain or github modules.

## Hexagonal boundary

| Layer | Module | Class | Responsibility |
|-------|--------|-------|---------------|
| Port | `domain/` | `MergeClient` (interface) | `MergeOutcome merge(String owner, String repo, int prNumber, String headSha)` |
| Result | `domain/` | `MergeOutcome` (sealed interface) | `Success(String mergeSha)` \| `Failure(String reason)` |
| Adapter | `github/` | `GitHubMergeClient` | Calls GitHub merge API, returns `MergeOutcome` |
| Wiring | `app/` | `PrReviewCaseHub` | Injects `MergeClient`, adapts to `WorkerFunction.Sync`, registers Worker |

**Why domain/, not review/:** The three-tier protocol says `domain/` = pure Java, no framework coupling. `MergeClient` expresses the domain concept "merge a PR" — its contract should not depend on the engine's `WorkerResult` type. The `WorkerResult` adaptation happens in `app/` (the wiring layer), where the engine dependency already exists.

`github/pom.xml` already depends on `casehub-devtown-domain`, so no new dependency needed.

## GitHubMergeClient (github module)

`@ApplicationScoped` bean implementing `MergeClient`. Uses Quarkus REST Client (`@RegisterRestClient`) for the outbound HTTP call.

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

`merge-method` accepts `squash` (default), `rebase`, or `merge`. The `sha` parameter is a server-side guard — GitHub rejects the merge if the PR's HEAD has changed.

**Dependencies:** `github/pom.xml` already has `quarkus-rest-client` and `quarkus-rest-client-jackson`. No changes needed.

**Response handling:**
- 200 OK → extract `sha` (merge commit SHA) → `MergeOutcome.Success(sha)`
- 409 Conflict → `MergeOutcome.Failure("merge conflict or SHA mismatch")`
- 405 Not Allowed → `MergeOutcome.Failure("branch protection prevents merge")`
- 422 Unprocessable → `MergeOutcome.Failure("PR not in mergeable state")`
- Network/other → `MergeOutcome.Failure("api error: " + message)`

## YAML changes to pr-review.yaml

### merge-executor capability: remove outputSchema

```yaml
- name: merge-executor
  description: "Executes the merge when all goals are satisfied"
  inputSchema: "{ pr: .pr }"
```

`outputSchema` removed entirely. When absent, `DefaultWorkerExecutor.applyOutputSchema()` passes through raw output unchanged. The merge SHA lands as `merge_sha` at the context root.

**Why not `outputSchema: "{}"`:** JQ `{}` constructs an empty object, discarding all output keys. The merge_sha would be lost.

### merge binding: add guard, outcomePolicy, conflictResolverStrategy

```yaml
- name: merge
  on: { contextChange: {} }
  when: >-
    .merge_sha == null and
    .pr.status != "merged" and
    (.codeAnalysis.securitySensitive == false or .securityReview.outcome == "APPROVED") and
    (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
    .styleCheck.outcome == "APPROVED" and
    .testCoverage.outcome == "APPROVED" and
    .performanceAnalysis.outcome == "APPROVED" and
    (.pr.linesChanged <= .policy.humanApprovalThreshold or .humanApproval.outcome == "APPROVED") and
    .ci.status == "passing"
  capability: merge-executor
  conflictResolverStrategy: DEEP_MERGE
  outcomePolicy:
    onDecline: FAULT
    onFailure: FAULT
    onExpired: FAULT
```

**`.merge_sha == null` guard:** Every other capability binding guards against its own output. Without this guard, the merge binding's complex condition evaluates to TRUE on every context change after the first merge — only to be filtered out by `filterToDispatchable()`. The guard short-circuits to FALSE immediately and documents idempotency intent. The engine's PlanItem model (`addPlanItemIfAbsent` + `filterToDispatchable`) also prevents double-dispatch structurally, but the guard is clearer and cheaper.

**outcomePolicy:** Explicit FAULT on all failure modes. The engine's default `OutcomePolicy()` is `REROUTE×3` — omitting the policy would cause three futile retry attempts against a single worker. FAULT transitions the case to FAULTED state, which is genuinely terminal.

**conflictResolverStrategy:** DEEP_MERGE for consistency with every other binding.

### New goal: merge-completed

```yaml
- name: merge-completed
  kind: success
  condition: '.pr.status == "merged" or .merge_sha != null'
```

### Updated completion

```yaml
completion:
  success:
    allOf:
      - pr-approved
      - security-verified
      - ci-passing
      - merge-completed
```

### Why merge-completed is required

Without it, the case completes before the merge worker finishes:

1. Context change → `rules()` fires merge binding → worker SCHEDULED (async event bus)
2. `goals()` evaluates on SAME cycle → pr-approved, security-verified, ci-passing all TRUE → case COMPLETES
3. Worker finishes later → handler checks case state → COMPLETED → early returns (line 121-124)
4. **Merge output is never written to case context**

With `merge-completed` checking `.merge_sha != null`:
1. Review goals satisfied → merge binding fires → worker scheduled
2. Goals evaluate → merge-completed FALSE (merge_sha null) → case stays RUNNING
3. Worker completes → output written → context change → merge-completed TRUE → case COMPLETES

The `pr.status == "merged"` short-circuit handles external merges — the merge binding doesn't fire (condition includes `.pr.status != "merged"`), and the goal short-circuits cleanly.

## Audit trail

With the `merge-completed` goal preventing premature completion:

1. Merge worker writes `merge_sha` to case context
2. Context change triggers goal evaluation → case COMPLETES
3. `MergeDecisionObserver` fires on COMPLETED → reads both `pr.headSha` and `merge_sha` from context
4. `MergeDecisionLedgerEntry.commitSha` stores the **merge commit SHA** (not the PR HEAD SHA)

**Fix to MergeDecisionObserver:** Line 75 currently reads `ctx.getPathAsString("pr.headSha")`. Change to prefer `merge_sha` when available:
```java
String mergeSha = ctx.getPathAsString("merge_sha");
entry.commitSha = mergeSha != null ? mergeSha : headSha;
```

For REJECTED decisions (case CANCELLED), `merge_sha` is null — falls back to `pr.headSha` correctly.

## Files

| File | Module | Change |
|------|--------|--------|
| `domain/.../MergeClient.java` | domain | New — port interface for merge execution |
| `domain/.../MergeOutcome.java` | domain | New — sealed interface: Success \| Failure |
| `github/.../GitHubMergeClient.java` | github | New — REST client implementing MergeClient |
| `app/.../PrReviewCaseHub.java` | app | @PostConstruct Worker registration with adaptMerge lambda |
| `app/.../ledger/MergeDecisionObserver.java` | app | Read merge_sha from context instead of pr.headSha |
| `review/.../resources/devtown/pr-review.yaml` | review | Remove merge-executor outputSchema, add merge binding guard + outcomePolicy + conflictResolverStrategy, add merge-completed goal, update completion |
| `app/src/main/resources/application.properties` | app | Add devtown.github.token, devtown.github.merge-method |
| Tests | All | Domain types, Worker registration, merge client (mock HTTP), adaptMerge (mock client), YAML goal/binding changes, MergeDecisionObserver merge_sha preference |

## What this does NOT include

- Retry/reroute on merge failure — FAULT is terminal, operator intervenes
- Merge queue batch-then-bisect — devtown#11 (separate epic)
- GitHub App authentication — PAT is sufficient for v1
- Commit message customization — uses GitHub's default squash message
- Merge-specific outcome variants (Conflict, Blocked) — `MergeOutcome` is a sealed interface; when merge queue (#11) or retry logic is added, new variants can be added and Java's exhaustiveness checking forces every `switch` (including `adaptMerge`) to handle them mechanically
