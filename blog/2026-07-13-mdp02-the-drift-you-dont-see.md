---
layout: post
title: "The Drift You Don't See"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [snapshot-drift, quarkus, testing]
series: issue-144-fix-trust-feedback-test
---

## The Drift You Don't See

SNAPSHOT dependencies have a failure mode that doesn't announce itself. Everything compiles. Your IDE shows no errors. Then you run a test and the JVM can't find a class that existed last week.

The `TrustFeedbackClosedLoopTest` was deleted two days ago because `AgentAssignment` — the return type from `AgentRoutingStrategy.select()` — no longer existed. The engine had replaced it with `RoutingResult`, a sealed interface with `Selected`, `Unresolvable`, and `Escalated` variants. The fix was mechanical: swap `AgentAssignment.Assigned` for `RoutingResult.Selected`, `workerId()` for `executorId()`. Ten minutes of work.

Except fixing that import uncovered `CaseLifecycleEvent` gaining three new record fields. Two more test files broken. And fixing those revealed the real problem: `casehub-engine-work-adapter`'s published SNAPSHOT still referenced `PlanItemStatus`, which the engine had renamed to `TaskStatus` weeks earlier. Every `@QuarkusTest` in devtown was dead — Hibernate entity enhancement fails if it can't resolve a field type.

The interesting part was where the shim had to go. I put a compatibility `PlanItemStatus` enum in devtown's `app/src/main/java`. Compilation passed. Quarkus still couldn't find it. The split-package classloader — `io.casehub.engine.common.internal.model` lives in both `casehub-engine-common` and `casehub-devtown-app` — and Quarkus loads that package exclusively from the jar that owns it. The devtown class was invisible at augmentation time.

The fix: add the shim to `casehub-engine-common` itself, where the package canonically lives. `mvn install` locally, and every downstream consumer sees it.

One more surprise waited. The restored test's `seedWorkerDecision` saved a `LedgerEntry` and its `LedgerAttestation` in the same `QuarkusTransaction.requiringNew()` block. That worked when the test was first written. It doesn't now — `saveAttestation` gained `@Transactional(REQUIRES_NEW)`, which suspends the outer transaction. The entry isn't committed yet; the attestation can't find it. Split into two transaction blocks. Test passes.

Three layers of drift from one deleted import. The engine SNAPSHOT, the work-adapter SNAPSHOT, and a transaction annotation change — each invisible until the previous layer was peeled back.
