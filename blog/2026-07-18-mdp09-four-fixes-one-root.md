---
layout: post
title: "Four Fixes, One Root"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [coordination, observer, engine-api, event-log, casehub-engine]
series: issue-164-tier1-design-gaps
---

*Continues from [The Test That Found Two Bugs Before It Passed](2026-07-18-mdp08-the-test-that-found-two-bugs.md).*

The previous session's E2E test found four design gaps in the coordinated change lifecycle. This session fixed all of them. What made it interesting: the four issues looked independent but shared a common root — the engine's external event boundary wasn't carrying enough information for application-level observers to do their job.

## The information gap at the CDI boundary

Issue #161 was the one that restructured my thinking about the other three. The `CoordinatedChangeObserver` uses `CaseLifecycleEvent` to watch sub-cases and signal the parent when reviews complete or fault. It checked for `COMPLETED` status as success, `FAULTED` as failure. Clean and simple — except the failure-cascade-pattern protocol says failure goals also produce `COMPLETED`. A sub-case hitting `review-abandoned` (PR closed) transitions to COMPLETED with failure metadata. The observer was signaling `completedReviews` for a closed PR.

The fix looked like it needed a new way to distinguish the two. But tracing the engine's internals revealed something: the information already existed. `CaseStatusChanged` carries `satisfiedGoalName` and `satisfiedGoalKind`. The `CaseStatusChangedHandler` writes them to EventLog metadata. `CaseOutcomeObserver` receives them. The data flows through every internal checkpoint — then gets dropped at the CDI boundary. `CaseLifecycleEvent` was the last record updated when goal-aware completions were added, and nobody noticed the gap because `COMPLETED` used to always mean success.

Two new fields on the record, one factory method update in the handler. The observer can now check `satisfiedGoalKind().equals("failure")` and route accordingly.

## The rollback chain that couldn't finish

Issue #164 was a YAML condition that fired too early. The `merge-failed` goal evaluated true the moment any merge result had `status == "failed"`:

```yaml
condition: '.mergeResults != null and (.mergeResults | any(.status == "failed"))'
```

The rollback binding fires on the same condition and the rollback worker runs — but the goal is already satisfied. The case terminates before `rollback-human-escalation` can fire. If a rollback has a merge conflict, the conflict is silently swallowed.

The fix: the goal waits for the rollback chain to resolve. It now requires rollback success, or human escalation completion, or explicit abandonment. Three extra lines of jq.

## Signal provenance and restart recovery

Issue #163 added a `signal()` overload that accepts caller metadata, merged into the EventLog entry. The observer now passes `causedByCaseId` and `causedByEvent` on every signal to the parent case — every coordination decision in the parent EventLog links back to the sub-case event that caused it.

Issue #162 was the simplest: the `CoordinatedChangeTrackerHydrator` logged "hydration deferred" on startup and did nothing. The in-memory tracker would lose all state on restart. Following the existing `PrReviewCaseTrackerHydrator` pattern — query `CaseInstanceRepository` for active coordinated-change cases, extract the `reviewCases` map, register each entry — took twenty minutes.

## A dependency wall

The E2E test for #164 exposed a pre-existing problem: `casehub-work`'s `WorkItemRef` constructor signature doesn't match between `casehub-work-api` and `casehub-work-runtime`. Every humanTask binding throws `NoSuchMethodError` at WorkItem creation time. The rollback-human-escalation binding fires correctly — the engine logs show `Publishing HumanTaskScheduleEvent` — but the WorkItem never materialises. Filed as #165.

The test proves #164 works by a different path: if `rollbackResults` appears in the parent case context, the rollback worker completed, which means the case was still alive after the merge failure. Before the fix, the merge-failed goal would have terminated it immediately. The escalation-to-resolution chain can't be tested end-to-end until #165 is resolved.
