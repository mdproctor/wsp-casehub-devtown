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
    (.rollbackResults | any(.status != "success")) and
    .rollbackEscalation == null
  conflictResolverStrategy: DEEP_MERGE
  humanTask:
    title: "Coordinated rollback failed — manual revert required"
    candidateGroups: [human-oversight:general]
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

**Deviation from issue #158:** The issue suggests checking EventLog for a prior rollback entry. This spec uses the YAML-guard pattern instead (`.rollbackResults == null` on the `rollback-on-merge-failure` binding), matching the merge worker's approach (`.mergeResults == null`). Once the worker writes output, the binding condition is false.

**Crash recovery:** If the worker crashes before writing `rollbackResults`, the binding's `outcomePolicy: { onFailure: FAULT }` transitions the case to FAULTED — the binding does not re-fire. In the unlikely event of a double-revert (e.g., engine restart during the crash window before the fault is recorded), the second attempt produces `MergeConflict` or `Failure` outcomes — benign noise, not data corruption, since GitHub revert PRs are themselves mergeable only once.

### Case lifecycle after rollback

1. Merge fails → `merge-failed` goal evaluates true → case eligible for failure completion
2. `rollback-on-merge-failure` binding fires → rollback worker starts (active plan item)
3. Worker completes → writes `rollbackResults` to blackboard
4. If all reverts succeed → no further bindings fire → case completes with failure
5. If any revert failed → `rollback-human-escalation` fires (another active plan item)
6. Human resolves/abandons → `rollbackEscalation` written to blackboard → plan item completes
7. No more active items → case completes with failure

**Human task expiry:** If the 4-hour deadline expires without human action, `DefaultSlaBreachPolicy.onBreach()` fires. If an escalation group is configured in SLA preferences, the task escalates to that group with a new deadline. If not configured (or already escalated to the configured group), the task fails with `escalation-group-not-configured` or the configured terminal reason. In either case, `rollbackEscalation` is written with the expiry/failure outcome and the case proceeds to completion. The case does NOT stall — human task expiry always produces a terminal outcome.

## What is NOT in scope

- Rollback on review failure (pre-merge) — nothing to revert
- Rollback on abandonment — separate concern, would need its own binding
- Explicit `causedByEntryId` linking — the parent spec (cross-repo-coordinated-merge-design §EventLog Audit Trail) uses `SIGNAL_RECEIVED` entries for coordination decisions, providing temporal audit ordering with `reviewCaseId` cross-references in signal payloads. The rollback worker follows the same pattern: the engine logs binding activation when `rollback-on-merge-failure` fires, and per-repo revert outcomes are recorded in the case context as `rollbackResults`. Together with the parent case's coordination signals, this provides a complete audit trail without requiring explicit `causedByEntryId` chains. Per-repo rollback EventLog entries are a potential future enhancement
- New domain types — worker uses `Map<String, Object>` like the merge worker

## Tests

### Worker unit tests

`CoordinatedRollbackWorkerTest` — tests on `CoordinatedChangeCaseHub.adaptCoordinatedRollback()` with mocked `RevertClient`:

1. **All reverts succeed** — 2 successful merges + 1 failed, both reverts succeed
2. **Best-effort on conflict** — repo A revert conflicts, repo B revert succeeds; both attempted
3. **All reverts fail** — all reverts return Failure; worker still returns Success
4. **Nothing to revert** — all merges failed (first repo failed immediately); empty rollbackResults
5. **Single repo** — one successful merge to revert; correlation logic works
6. **Commit message context** — verify commitMessage contains failed repo name

### Binding condition tests

`CoordinatedChangeBindingConditionTest` — tests on YAML binding `when:` conditions:

7. **rollback fires on merge failure** — `mergeResults` has failure, `rollbackResults` null → `rollback-on-merge-failure` evaluates true
8. **rollback does not re-fire** — `rollbackResults` already set → `rollback-on-merge-failure` evaluates false
9. **escalation fires on revert failure** — `rollbackResults` has non-success, `rollbackEscalation` null → `rollback-human-escalation` evaluates true
10. **escalation does not re-fire after human completion** — `rollbackResults` has failures, `rollbackEscalation` already set → `rollback-human-escalation` evaluates false
11. **escalation does not fire when all reverts succeed** — `rollbackResults` all success → `rollback-human-escalation` evaluates false

### CaseHub definition tests

Updates to `CoordinatedChangeCaseHubTest`:

12. **`hasTwoBindings` → `hasThreeBindings`** — assert `hasSize(3)`, names include `rollback-human-escalation`
13. **`hasCoordinatedRollbackWorker`** — new test, parallel to existing `hasCoordinatedMergeWorker`

## Files changed

| File | Change |
|------|--------|
| `domain/.../AgentQualification.java` | Add `COORDINATED_ROLLBACK` constant |
| `app/.../CoordinatedChangeCaseHub.java` | Inject `RevertClient`, add `adaptCoordinatedRollback()`, register in `augment()` |
| `app/.../resources/casehub/devtown/coordinated-change.yaml` | Add `rollback-human-escalation` binding |
| `app/.../CoordinatedRollbackWorkerTest.java` | New test class — 6 worker test cases |
| `app/.../CoordinatedChangeBindingConditionTest.java` | New test class — 5 binding condition tests |
| `app/.../CoordinatedChangeCaseHubTest.java` | Update binding count (2→3), add `hasCoordinatedRollbackWorker` test |
