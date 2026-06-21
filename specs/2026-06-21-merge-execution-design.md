# Merge Execution — merge-executor worker via GitHub API

**Issue:** devtown#87
**Date:** 2026-06-21
**Status:** Approved (rev 2 — post review, 8 findings addressed)

## Problem

The merge binding in `pr-review.yaml` fires capability `merge-executor` when all review goals + CI passing are satisfied. No worker exists to execute the merge — `NoOpReactiveWorkerProvisioner` throws `ProvisioningException` and the binding is auto-exhausted. PRs pass review but are never merged.

## Architecture

The engine dispatches `merge-executor` to a `Worker` registered programmatically in the `CaseDefinition`. The Worker wraps a `WorkerFunction.Sync` that calls the GitHub merge API via a `MergeClient` port interface.

**Why not @Named CDI lookup:** The engine does not perform CDI lookup for workers. Worker dispatch goes through `CaseDefinition.getWorkers()` → `AgentCandidateFactory.buildCandidates()` → `AgentRoutingStrategy.select()`. A CDI bean is invisible to this chain. Workers must be registered in the CaseDefinition.

**Why capability dispatch for a local sync function:** The YAML binding + Worker model is the engine's native dispatch mechanism. Using a different mechanism for merge would be inconsistent. The merge could become an external service (merge queue, CI/CD pipeline) or a claudony agent in future deployments — the capability model enables this without rearchitecting. Trust scaffolding overhead is zero for a single-impl Worker — `AgentCandidateFactory` returns one candidate, `AgentRoutingStrategy` selects it directly.

## Worker registration

`PrReviewCaseHub` overrides `getDefinition()` to programmatically add the merge-executor Worker after YAML parsing:

```java
@Override
public CaseDefinition getDefinition() {
    CaseDefinition def = super.getDefinition();
    if (def.getWorkers().stream().noneMatch(w -> w.getName().equals("merge-executor"))) {
        Capability mergeCap = def.getCapabilities().stream()
            .filter(c -> "merge-executor".equals(c.getName()))
            .findFirst().orElseThrow();
        def.getWorkers().add(Worker.builder()
            .name("merge-executor")
            .capabilities(mergeCap)
            .function(mergeClient::merge)
            .build());
    }
    return def;
}
```

The `mergeClient` is injected via the `MergeClient` port interface (see hexagonal boundary below).

## Hexagonal boundary

| Layer | Module | Class | Responsibility |
|-------|--------|-------|---------------|
| Port | `review/` | `MergeClient` (interface) | `WorkerResult merge(Map<String, Object> input)` |
| Adapter | `github/` | `GitHubMergeClient` | Calls GitHub merge API, returns `WorkerResult` |
| Wiring | `app/` | `PrReviewCaseHub` | Injects `MergeClient`, registers Worker in CaseDefinition |

`MergeClient` returns `WorkerResult` directly — the Worker function signature is `Function<Map<String, Object>, WorkerResult>`. No separate `MergeResult` record needed.

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

**Dependencies:** Add `quarkus-rest-client` and `quarkus-rest-client-jackson` to `github/pom.xml`.

**Response handling:**
- 200 OK → extract `sha` (merge commit SHA) → `WorkerResult.of(Map.of("merge_sha", sha))`
- 409 Conflict → `WorkerResult.failed("merge conflict or SHA mismatch")`
- 405 Not Allowed → `WorkerResult.failed("branch protection prevents merge")`
- 422 Unprocessable → `WorkerResult.failed("PR not in mergeable state")`
- Network/other → `WorkerResult.failed("api error: " + message)`

## YAML changes to pr-review.yaml

### merge-executor capability: remove outputSchema

```yaml
- name: merge-executor
  description: "Executes the merge when all goals are satisfied"
  inputSchema: "{ pr: .pr }"
```

`outputSchema` removed entirely. When absent, `DefaultWorkerExecutor.applyOutputSchema()` passes through raw output unchanged. The merge SHA lands as `merge_sha` at the context root.

**Why not `outputSchema: "{}"`:** JQ `{}` constructs an empty object, discarding all output keys. The merge_sha would be lost.

### merge binding: add outcomePolicy and conflictResolverStrategy

```yaml
- name: merge
  on: { contextChange: {} }
  when: >-
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
| `review/.../MergeClient.java` | review | New — port interface for merge execution |
| `github/.../GitHubMergeClient.java` | github | New — REST client implementing MergeClient |
| `github/pom.xml` | github | Add quarkus-rest-client, quarkus-rest-client-jackson deps |
| `app/.../PrReviewCaseHub.java` | app | Override getDefinition() to register merge Worker |
| `app/.../ledger/MergeDecisionObserver.java` | app | Read merge_sha from context instead of pr.headSha |
| `review/.../resources/devtown/pr-review.yaml` | review | Remove merge-executor outputSchema, add merge binding outcomePolicy + conflictResolverStrategy, add merge-completed goal, update completion |
| `app/src/main/resources/application.properties` | app | Add devtown.github.token, devtown.github.merge-method |
| Tests | All | Worker registration, merge client (mock HTTP), merge function (mock client), YAML goal/binding changes, MergeDecisionObserver merge_sha preference |

## What this does NOT include

- Retry/reroute on merge failure — FAULT is terminal, operator intervenes
- Merge queue batch-then-bisect — devtown#11 (separate epic)
- GitHub App authentication — PAT is sufficient for v1
- Commit message customization — uses GitHub's default squash message
