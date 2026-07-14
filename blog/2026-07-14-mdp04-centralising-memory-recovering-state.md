---
layout: post
title: "Centralising Memory, Recovering State"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub]
series: issue-150-memory-emitter-hydration
---

Two pieces of work that look unrelated but share a theme: devtown had scattered patterns where centralised ones now exist.

`CaseMemoryEmitter` and `FeatureVectorEmitter` both injected `Instance<CaseMemoryStore>` and rolled their own error handling — null checks, try/catch, warn logging. The neocortex `MemoryEmitter` bridge (neocortex#64) already centralises all of this: consistent error logging, `SecurityException` passthrough, partial-failure reporting for batch writes. The migration was mechanical — replace the injection, remove the defensive code, let the bridge handle it. The emitters shrank by half.

The more interesting piece was startup hydration. `PrReviewCaseTracker` is an in-memory materialised view — a `ConcurrentHashMap` of active PR review cases that the MCP tools and governance queries read from. On restart, it's empty. The assumption was that the engine's `CaseInstanceRepository` lacked a query API for non-terminal instances. It doesn't — `findByStatus(CaseStatus, tenancyId)` has been on the SPI since the repository was written. The blocker was imaginary.

The hydrator is a separate `@ApplicationScoped` class that observes `StartupEvent`, queries for `STARTING`, `RUNNING`, `WAITING`, and `SUSPENDED` cases, extracts the PR payload from the case context (where `PrReviewCaseService.startReview()` stores it under the `"pr"` key), and registers each into the tracker. The status is corrected after registration — `register()` always sets `RUNNING`, so `WAITING` and `SUSPENDED` cases get a follow-up `updateStatus()` call.

While running the full test suite, three pre-existing configuration bugs surfaced. The `quarkus.arc.selected-alternatives` referenced `MemoryPlanItemStore` — a class that doesn't exist. The actual class is `InMemoryPlanItemStore`. CDI silently fell through to the JPA implementation, which then threw `ContextNotActive` on async threads that had no request context. The SLA breach policy was configured to resolve `"no-op"` by default, but `NoOpSlaBreachPolicy` was excluded from CDI to avoid ambiguity — so the resolver found nothing. And both `PrReviewCaseTracker` and `MergeBatchCompletionObserver` passed `event.caseStatus()` into a `switch` statement without null-checking, despite the event contract explicitly allowing null for non-status transitions.

The silent CDI fallthrough is the kind of bug that hides for weeks because nothing fails loudly — the wrong bean satisfies the type contract, and the failure mode (missing request context on an async thread) looks like a threading issue, not a wiring issue.
