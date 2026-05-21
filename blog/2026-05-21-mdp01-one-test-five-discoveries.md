---
layout: post
title: "One test, five engine discoveries"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: log
projects: [casehub-devtown]
tags: [casehub, hitl, quarkus, cdi, testing, engine]
---

The plan was straightforward. The human-approval WorkItem binding was wired
in the previous session, so writing an integration test should have been a
matter of plumbing — start a case, confirm a WorkItem appears, complete it,
verify the case context updates, assert the case completes. Half a day, maybe.

It took a full day and surfaced five engine issues.

## The PENDING guard that fired too early

The first failure was immediate. `HumanTaskScheduleHandler` — the component
that creates the WorkItem when a humanTask binding fires — logged that the
PlanItem was already RUNNING when it received the event and skipped the
WorkItem creation. Nothing was persisted. No error.

Claude traced this to `PlanningStrategyLoopControl.indexSelectedForCompletion()`,
which marks every selected binding's PlanItem RUNNING synchronously before
publishing the dispatch event. By the time the handler received the event,
the PENDING guard had nothing to latch onto. The fix: relax the guard to
accept RUNNING status on first receipt.

## The when: discovery

The bigger find came from the test setup. We pre-seeded the initial case context
to suppress capability bindings — giving `styleCheck`, `testCoverage`, and
`performanceAnalysis` non-null values so their `when:` conditions would be
false and they wouldn't fire. The idea was clean. It didn't work.

All bindings fired on every `CONTEXT_CHANGED` regardless.

Claude traced it through `CaseContextChangedEventHandler.rules()`. For
`contextChange: {}` triggers, `getFilter()` returns null. The expression engine
treats null as "always true." The binding's `when:` field — the JQ expression
developers write expecting it to gate execution — is stored on `Binding.when`
but never evaluated at runtime. The unit tests test `when:` conditions directly
via `LambdaExpressionEvaluator.test()` and they pass, which makes the problem
invisible until something exercises the engine's actual binding evaluator.

This means every `contextChange` binding fires on every engine tick regardless
of what its `when:` condition says. The deduplication that prevents repeat
WorkItem creation comes from the blackboard's PlanItem state, not from the
condition. A pre-existing architectural assumption just became a filed issue.

## Two more from the adapter

Once the WorkItem was being created correctly, the context update — the step
where completing the WorkItem feeds back into the case — reliably failed to
happen.

`WorkItemLifecycleAdapter.onWorkItemLifecycle()` called
`ReactiveUtils.runOnSafeVertxContext()` to schedule the outputMapping and
context-changed fire on the Vert.x event loop. The call timed out consistently.
Claude replaced it with a direct `Uni.await()` on the CDI executor thread, which
resolved the issue.

The adapter also turned out not to be called at all via CDI `@ObservesAsync`
in the devtown context — even though the bean was injectable and its Jandex
index was correct. A test-scope `@ApplicationScoped` observer in the app module
received the same events without issue. The adapter works correctly in the
engine's own test module; the discrepancy is filed and the test calls the
adapter directly as a verified workaround.

## What shipped

The test passes. It verifies the WorkItem is created, CDI async delivery works
in this runtime, the adapter applies the outputMapping and updates the case
context, and the case completes once all goals are satisfied.

Five engine issues are filed and tracked. The `when:` conditions finding in
particular changes how I think about YAML case definitions — those conditions
are documentation and test scaffolding right now, not runtime filters. That's
worth knowing before anyone writes a case definition assuming otherwise.
