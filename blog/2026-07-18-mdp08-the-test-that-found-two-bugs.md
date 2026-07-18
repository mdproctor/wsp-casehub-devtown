---
layout: post
title: "The Test That Found Two Bugs Before It Passed"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [testing, coordination, casehub-engine, integration]
---

Epic #12's acceptance criterion is a single sentence: "A 2-repo change set merges atomically, and a failure in one repo triggers automatic rollback of the other." Five issues built the machinery. The sixth — the integration test — was supposed to prove it works. It did, but it also proved two things don't.

## The coordination chain

The interesting thing about this test is how many async boundaries it crosses. `CoordinatedChangeService` starts a parent case and N review cases. Each review runs through the full `pr-review.yaml` lifecycle in the engine. When a review case reaches terminal state, the engine fires a `CaseLifecycleEvent` via CDI `@ObservesAsync`. The `CoordinatedChangeObserver` picks it up, signals the parent case context. When all reviews complete, the parent's YAML binding dispatches the merge worker. If merge fails, the rollback binding fires. If rollback fails, a humanTask escalation binding creates a WorkItem.

That's four CasePlanModel evaluation cycles, two YAML files, one CDI event bridge, and three worker dispatches. The unit tests covered each piece. Nothing covered the chain.

## What the test actually found

The happy path worked first try — reviews complete, observer bridges completion to the parent, merge fires, case reaches COMPLETED. Five scenarios, all green. But two design gaps surfaced along the way.

**The observer doesn't distinguish success from failure.** `CoordinatedChangeObserver` classifies `CaseStatus.COMPLETED` as terminal success. But failure goals in the engine also produce COMPLETED — with failure metadata, not a distinct status. A review case that hits `review-abandoned` (PR closed) would be misclassified as successfully completed. The coordinator would then try to merge a closed PR.

I sidestepped this by using `cancelCase()` for the fault path (which produces CANCELLED, correctly classified as failure). But the real gap needs fixing — the observer needs to inspect the completion outcome, not just the status enum.

**The merge-failed goal fires before rollback can escalate.** The `coordinated-change.yaml` has a `merge-failed` failure goal that triggers when `mergeResults` contains a failure. The `rollback-on-merge-failure` binding fires on the same condition. The engine dispatches the binding first — so the rollback worker does execute. But when rollback completes and writes `rollbackResults`, the next evaluation cycle sees `merge-failed` still true and terminates the case. The `rollback-human-escalation` binding never gets a chance to fire.

This is a goal-vs-binding race in the YAML design. The fix is straightforward: condition `merge-failed` on rollback completion, not just merge failure. But finding it required the full chain — no unit test would have caught a timing dependency between goal evaluation and binding dispatch across consecutive engine cycles.

## The Quarkus classloader trap

One gotcha worth recording: injecting `CoordinatedChangeService` (the concrete `@ApplicationScoped` class) instead of `CoordinatedChangePort` (the interface) produces a `LinkageError` at test instantiation. The interface and implementation are loaded by different Quarkus classloaders in the multi-module build, and the JVM's itable verification catches the disagreement on domain types in the method signature. It compiles fine. It fails at runtime with no useful pointer to the fix.

The whole point of an integration test is to find these seams — the places where independently correct components interact incorrectly. This one found two in the coordination layer and one in the test infrastructure. All three are now tracked as issues; the test passes by working around them. When the fixes land, the workarounds become the regression tests.
