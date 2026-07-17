---
layout: post
title: "When sub-cases aren't sub-cases"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cross-repo, coordination, case-model, engine]
---

Cross-repo coordinated merge sounds like sub-case orchestration. The engine has sub-cases. The YAML has `subCase:` bindings with `groupId` and M-of-N semantics. The merge queue already uses them for bisection — two sub-cases, statically declared.

The problem is the word "dynamic." A coordinated change has N repositories, where N is determined at runtime. The YAML declares bindings statically. There is no forEach, no iteration, no "spawn one per element in this array."

I spent time tracing the engine internals to understand what `CaseHubRuntime.startCase()` actually does when you pass `parentCaseId`. The answer: almost nothing useful. It sets a field on the `CaseInstance`. That's it. No `SUBCASE_STARTED` EventLog entry, no `SubCaseGroup` creation, no `PlanItem` delegation, no WAITING state. All of that lifecycle management lives exclusively in `SubCaseExecutionHandler`, which only fires from YAML sub-case bindings — the code path you cannot reach programmatically without coupling to engine internals.

So the question becomes: do you extend the engine, or do you solve it in the application?

I chose the application. Each per-repo review is a standard pr-review case started through the existing `startReview()` path. It registers with `PrReviewCaseTracker`, so webhook routing works unchanged. The coordination is a separate parent case with a `CoordinatedChangeTracker` and a `CaseLifecycleEvent` observer that signals the parent when reviews complete or fault.

The design review caught something I'd missed. The existing `pr-review.yaml` has two merge bindings — `merge-direct` and `enqueue-for-merge` — that fire independently when a PR's reviews pass. In a coordinated change, each repo's PR would auto-merge the moment its reviews completed, breaking the all-or-nothing atomicity. The fix is a `coordinatedChange` context flag injected at `startReview()` time. The YAML evaluates `.coordinatedChange != true` on merge bindings — when the flag is absent (standalone reviews), JQ evaluates `null != true` to `true`, so nothing changes. When it's present, merge is suppressed and the coordinator handles it.

This pattern — an opaque context flag that modifies YAML binding behaviour without the port interface knowing about it — turns out to be the cleanest way to extend a CasePlanModel's behaviour from the outside. The `PrReviewApplicationService` has no knowledge of coordinated changes. It just accepts additional context and merges it into the case. The YAML conditions do the rest.

The coordinated-merge worker itself is straightforward: iterate the repos in order, call `MergeClient.merge()` for each, stop on first failure. The `outputProjection` maps the result into the parent context where the goal conditions evaluate it. The rollback worker will use `RevertClient` when it's built — the binding is declared in the YAML but unwired for now.

The interesting architectural decision is what I didn't build: engine sub-cases, engine extensions, parent-child lifecycle management. The coordination lives entirely in devtown. The engine provides signals and context; the application decides what they mean.
