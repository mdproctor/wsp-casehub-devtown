# Coordinated Rollback Worker — Design Spec

**Issue:** devtown#158
**Epic:** #12 (Cross-repo coordinated merge)
**Date:** 2026-07-17

## Context

When a coordinated change merges repos sequentially and one fails, the already-merged repos must be reverted. The `rollback-on-merge-failure` binding in `coordinated-change.yaml` already fires on merge failure — this issue implements the worker that executes the reverts.

## Design

### Worker function

`CoordinatedChangeCaseHub.adaptCoordinatedRollback(Map<String, Object> input)` — registered in `augment()` alongside the existing merge worker.

**Input** (from YAML input projection `{ repos: .repos, mergeResults: .mergeResults }`):
- `repos` — original repo list with `owner`, `repo`, `prNumber`, `headSha`, `targetBranch`
- `mergeResults` — list of `{repo, status, mergeSha?, reason?}`

**Logic:**
1. Parse `repos` and `mergeResults` from input
2. Identify the failed repo (first entry with `status == "failed"`) for commit messages
3. Filter `mergeResults` to entries with `status == "success"`
4. For each successful merge:
   - Correlate with `repos` by matching `owner + "/" + repo` to get `targetBranch`
   - Call `RevertClient.revert(owner, repo, targetBranch, mergeSha, commitMessage)`
   - Map `RevertOutcome` to result map
5. Return `WorkerResult.of(Map.of("rollbackResults", results))`

**Best-effort:** Unlike the merge worker which stops on first failure, the rollback worker attempts ALL reverts. Each successful revert reduces damage — stopping early would leave merged repos unreversed.

**Commit message format:** `"Revert <owner>/<repo>#<prNumber> — coordinated rollback (merge failure in <failedRepo>)"`

**Output mapping per repo:**

| RevertOutcome | status | fields |
|---------------|--------|--------|
| Success | `"success"` | `revertPrNumber`, `revertSha` |
| MergeConflict | `"conflict"` | `revertPrNumber`, `reason` |
| Failure | `"failed"` | `reason` |

**Always returns `WorkerResult.of()`.** Revert failures are data in the output, not worker-level errors. The YAML decides what happens next.

### YAML changes

One new binding in `coordinated-change.yaml` — human escalation when any revert fails:

```yaml
- name: rollback-human-escalation
  on: { contextChange: {} }
  when: >-
    .rollbackResults != null and
    (.rollbackResults | any(.status != "success"))
  conflictResolverStrategy: DEEP_MERGE
  humanTask:
    title: "Coordinated rollback failed — manual revert required"
    candidateGroups: [human-oversight:routing-review]
    expiresIn: PT4H
    outputMapping: "{ rollbackEscalation: . }"
    outcomes: [RESOLVED, ABANDONED]
```

No changes to existing bindings, goals, or completion rules. The `merge-failed` failure goal already fires on merge failure. The engine waits for all active plan items (rollback worker, then human task if needed) before finalizing the case.

### Vocabulary

Add to `AgentQualification`:
```java
public static final String COORDINATED_ROLLBACK = "coordinated-rollback";
```

### Wiring

`CoordinatedChangeCaseHub`:
- Inject `RevertClient` alongside existing `MergeClient`
- Register worker in `augment()`: `Worker.builder().name("coordinated-rollback").capabilityName("coordinated-rollback").function(this::adaptCoordinatedRollback).build()`

### Idempotency

Handled structurally by the YAML condition `.rollbackResults == null` — once the worker writes output, the binding condition is no longer true. No EventLog check needed.

### Case lifecycle after rollback

1. Merge fails → `merge-failed` goal evaluates true → case eligible for failure completion
2. `rollback-on-merge-failure` binding fires → rollback worker starts (active plan item)
3. Worker completes → writes `rollbackResults` to blackboard
4. If any revert failed → `rollback-human-escalation` fires (another active plan item)
5. Human resolves/abandons → plan item completes
6. No more active items → case completes with failure

## What is NOT in scope

- Rollback on review failure (pre-merge) — nothing to revert
- Rollback on abandonment — separate concern, would need its own binding
- EventLog causedByEntryId linking — the engine captures binding activation in the EventLog automatically; explicit cross-referencing is an engine feature, not a worker concern
- New domain types — worker uses `Map<String, Object>` like the merge worker

## Tests

Unit tests on `CoordinatedChangeCaseHub.adaptCoordinatedRollback()` with mocked `RevertClient`:

1. **All reverts succeed** — 2 successful merges + 1 failed, both reverts succeed
2. **Best-effort on conflict** — repo A revert conflicts, repo B revert succeeds; both attempted
3. **All reverts fail** — all reverts return Failure; worker still returns Success
4. **Nothing to revert** — all merges failed (first repo failed immediately); empty rollbackResults
5. **Single repo** — one successful merge to revert; correlation logic works
6. **Commit message context** — verify commitMessage contains failed repo name

## Files changed

| File | Change |
|------|--------|
| `domain/.../AgentQualification.java` | Add `COORDINATED_ROLLBACK` constant |
| `app/.../CoordinatedChangeCaseHub.java` | Inject `RevertClient`, add `adaptCoordinatedRollback()`, register in `augment()` |
| `app/.../resources/casehub/devtown/coordinated-change.yaml` | Add `rollback-human-escalation` binding |
| `app/.../CoordinatedRollbackWorkerTest.java` | New test class — 6 test cases |
