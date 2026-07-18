# Tier 1 Design Gaps — Coordinated Change Lifecycle Fixes

**Issues:** #164, #161, #163, #162
**Branch:** `issue-164-tier1-design-gaps`
**Date:** 2026-07-18

Four design gaps found by #160 (cross-repo coordinated merge E2E test). All relate to the coordinated change lifecycle — the observer, the YAML case model, the EventLog provenance, and restart hydration.

---

## 1. merge-failed Goal Race (#164)

### Problem

The `merge-failed` failure goal in `coordinated-change.yaml` fires the moment any merge result has `status == "failed"`:

```yaml
- name: merge-failed
  kind: failure
  condition: >-
    .mergeResults != null and
    (.mergeResults | any(.status == "failed"))
```

The `rollback-on-merge-failure` binding fires on the same condition and the rollback worker runs. But the goal is already satisfied — the case terminates on the next evaluation cycle before `rollback-human-escalation` can fire. If a rollback has a merge conflict, there is no path to human escalation. The conflict is silently swallowed.

### Fix

The goal must wait for the rollback chain to resolve — either rollback succeeds, human escalation completes, or the coordination is abandoned:

```yaml
- name: merge-failed
  kind: failure
  condition: >-
    .mergeResults != null and
    (.mergeResults | any(.status == "failed")) and
    ((.rollbackResults != null and (.rollbackResults | all(.status == "success")))
      or .rollbackEscalation != null
      or .abandoned == true)
```

### Files changed

- `app/src/main/resources/casehub/devtown/coordinated-change.yaml` — goal condition

### Test changes

`CrossRepoCoordinatedMergeTest.rollbackFailure_mergeConflict_parentTerminatesAfterRollback`:
- Remove the comment documenting the gap (lines 348-352)
- After rollback fires, assert the human escalation WorkItem is created
- Resolve the WorkItem with RESOLVED outcome
- Assert the case terminates with `merge-failed` goal after escalation resolves

---

## 2. Observer COMPLETED Semantics (#161)

### Problem

`CoordinatedChangeObserver` uses `Set.of("COMPLETED")` for `TERMINAL_SUCCESS`. Per the failure-cascade-pattern protocol, failure goals also produce `CaseStatus.COMPLETED` — not `FAULTED`. A sub-case that hits `review-abandoned` (PR closed) transitions to COMPLETED with failure metadata, but the observer classifies it as successful, signaling `completedReviews` instead of `reviewFaulted`.

### Root cause

The information to distinguish success-COMPLETED from failure-COMPLETED exists in the engine — `CaseStatusChanged` carries `satisfiedGoalName` and `satisfiedGoalKind`, and `CaseStatusChangedHandler` writes them to EventLog metadata (line 111-114) and passes them to `CaseOutcomeObserver` (line 160). But they are **not** propagated to `CaseLifecycleEvent`, which is the CDI event that external observers receive.

### Fix — Engine (casehub-engine)

Add `satisfiedGoalName` and `satisfiedGoalKind` to the `CaseLifecycleEvent` record:

```java
public record CaseLifecycleEvent(
    UUID caseId,
    String tenancyId,
    String commandType,
    String eventType,
    String caseStatus,
    String actorId,
    String actorRole,
    String traceId,
    String caseDefinitionName,
    String namespace,
    JsonNode contextSnapshot,
    String satisfiedGoalName,   // new
    String satisfiedGoalKind    // new
) { ... }
```

Update factory methods to accept and propagate these fields. Update `CaseStatusChangedHandler` (line 194-201) to pass the goal metadata through.

### Fix — Devtown

Update `CoordinatedChangeObserver.onCaseLifecycle`:

```java
if ("COMPLETED".equals(event.caseStatus())) {
    if ("failure".equals(event.satisfiedGoalKind())) {
        caseHubRuntime.signal(entry.parentCaseId(), "reviewFaulted",
            Map.of("repo", entry.repo(), "reason", event.satisfiedGoalName()));
    } else {
        if (tracker.markCompleted(entry.parentCaseId(), entry.repo())) {
            caseHubRuntime.signal(entry.parentCaseId(),
                "completedReviews." + entry.repo(), ...);
        }
        if (tracker.tryTransitionToAllCompleted(entry.parentCaseId())) {
            caseHubRuntime.signal(entry.parentCaseId(), "allReviewsCompleted", true);
        }
    }
}
```

Remove the `TERMINAL_SUCCESS` / `TERMINAL_FAILURE` sets — the status+goalKind pair is the complete discriminator.

### Files changed

**Engine:**
- `common/src/main/java/io/casehub/engine/common/spi/event/CaseLifecycleEvent.java` — add fields + factory methods
- `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — propagate goal metadata

**Devtown:**
- `app/src/main/java/io/casehub/devtown/app/CoordinatedChangeObserver.java` — check goal kind
- `app/src/test/java/io/casehub/devtown/app/CoordinatedChangeObserverTest.java` — update tests

### Test changes

`CrossRepoCoordinatedMergeTest`:
- Replace `driveReviewToFault(reviewCaseId)` implementation: instead of `cancelCase()`, signal `abandoned: true` on the sub-case to trigger the `review-abandoned` failure goal. This exercises the real failure-COMPLETED path.
- Verify the observer signals `reviewFaulted` (not `completedReviews`) when a sub-case completes via failure goal.

---

## 3. Cross-Case EventLog Provenance (#163)

### Problem

The `signal()` API has no way to pass caller metadata that gets stored in the EventLog. When the observer signals the parent case on sub-case completion, the EventLog entry contains only `origin: SIGNAL` — no record of which sub-case caused the signal.

### Fix — Engine (casehub-engine)

Add a new `signal()` overload to `CaseHubRuntime`:

```java
default CompletionStage<Void> signal(
    UUID caseId, String path, Object value, Map<String, Object> signalMetadata) {
  return signal(caseId, path, value);
}
```

Update `SignalReceivedEvent` to carry optional `signalMetadata`. Update `SignalReceivedEventHandler.buildSignalEventLog` to merge signalMetadata into the EventLog metadata node.

Update `CaseHubRuntimeImpl.signal()` to pass through the metadata.

### Fix — Devtown

`CoordinatedChangeObserver` passes provenance on every signal to the parent case:

```java
caseHubRuntime.signal(entry.parentCaseId(), key, value,
    Map.of("causedByCaseId", entry.reviewCaseId().toString(),
           "causedByEvent", event.eventType()));
```

### Files changed

**Engine:**
- `api/src/main/java/io/casehub/api/engine/CaseHubRuntime.java` — new overload
- `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java` — implement
- `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java` — pass metadata through
- `common/src/main/java/io/casehub/engine/common/internal/event/SignalReceivedEvent.java` — add field
- `runtime/src/main/java/io/casehub/engine/internal/engine/handler/SignalReceivedEventHandler.java` — merge metadata into EventLog

**Devtown:**
- `app/src/main/java/io/casehub/devtown/app/CoordinatedChangeObserver.java` — pass provenance

### Test changes

- Engine: unit test verifying signalMetadata appears in EventLog entry metadata
- Devtown E2E: verify EventLog entries on parent case carry `causedByCaseId` linking to sub-case

---

## 4. TrackerHydrator Restart Recovery (#162)

### Problem

`CoordinatedChangeTrackerHydrator.onStartup()` logs "hydration deferred" but performs no actual hydration. On restart, the in-memory `CoordinatedChangeTracker` is empty — any in-flight coordinated changes become orphaned.

### Fix

Follow the existing `PrReviewCaseTrackerHydrator` pattern:

1. Inject `CaseInstanceRepository` and `CurrentPrincipal`
2. On startup, query `findByNamespaceAndName("devtown", "coordinated-change", tenancyId)`
3. Filter to non-terminal statuses (STARTING, RUNNING, WAITING, SUSPENDED)
4. Extract `reviewCases` map from each case's context
5. Call `tracker.register()` for each entry
6. Check if any repos already have `completedReviews` entries and call `tracker.markCompleted()` accordingly

### Files changed

- `app/src/main/java/io/casehub/devtown/app/CoordinatedChangeTrackerHydrator.java` — implement hydration
- `app/src/test/java/io/casehub/devtown/app/CoordinatedChangeTrackerHydratorTest.java` — update test

### Test

Unit test that:
1. Pre-populates CaseInstanceRepository with a coordinated-change case instance
2. Runs hydration
3. Asserts the tracker has the correct entries
4. Asserts the tracker reports correct completion state for partially-completed coordinations

---

## Implementation Order

1. **#161 engine changes first** — CaseLifecycleEvent goal metadata. This is a prerequisite for the devtown observer fix and unblocks the correct E2E test path.
2. **#163 engine changes** — signal metadata overload. Can be done in the same engine commit as #161.
3. **#164 YAML fix** — merge-failed goal condition. Depends on #161 for correct E2E test coverage.
4. **#161 devtown changes** — observer fix + E2E test update. Uses the engine changes from step 1.
5. **#163 devtown changes** — observer provenance. Uses the engine changes from step 2.
6. **#162 hydrator** — independent, can be done any time.
7. **E2E test updates** — consolidate all test changes.

Steps 1-2 are a single engine commit. Steps 3-7 are devtown commits.

---

## Cross-Repo Impact

**casehub-engine changes:** CaseLifecycleEvent is in `engine-common` (SPI module). All consumers of CaseLifecycleEvent need recompilation but not code changes — the new fields are additive. Checked callers:
- `CoordinatedChangeObserver` (devtown) — will be updated
- `MergeDecisionObserver` (devtown) — uses `CaseLifecycleEvent` but only reads caseId/caseStatus/contextSnapshot — no change needed
- Ledger CDI observers — if any exist, they construct from factory methods which will have backward-compatible overloads

**signal() overload:** default method with fallback to existing signal() — no breaking change for existing callers.
