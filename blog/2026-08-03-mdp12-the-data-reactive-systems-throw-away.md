---
layout: post
title: "The Data Reactive Systems Throw Away"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [reactive-systems, event-sourcing, audit-trail, casehub-engine]
---

Every reactive system I've worked on has the same blind spot. A rule fires, a binding activates, a handler dispatches — and the state that caused it is gone. You can see what happened. You can see who acted. But the question "why did this fire?" has no answer in the audit trail.

CaseHub's engine evaluates bindings against a blackboard context. When a context layer changes — say, a security analysis completes and writes its findings — the engine re-evaluates every binding's filter and guard conditions against the new state. Bindings that pass get dispatched: workers scheduled, human tasks created, sub-cases spawned. The `WORKER_SCHEDULED` event log entry faithfully records the worker name, the capability, the binding, the input projection. Everything except the one thing an operator actually needs: what was in the context that made the condition true?

The data was right there. `CaseContextChangedEventHandler.rules()` has the `contextSnapshot` and the `changedLayer` — the exact state that triggered the evaluation. It evaluates the binding's JQ filter expression against it. The expression returns true. And then the snapshot is discarded, because nobody downstream asked for it.

This is the reactive system's version of write-ahead logging in reverse. We log the effects (worker scheduled, task created) but not the cause (context state at evaluation time). Any system that evaluates conditions against mutable state and dispatches on the result has this problem — whether it's a production rule engine, a CQRS event handler, or a workflow orchestrator watching for state transitions.

The fix is structurally simple: snapshot the changed layer content at evaluation time and thread it through the dispatch chain to the event log. We capture `contextSnapshot.layer(changedLayer).asJsonNode()` — the content of the specific layer whose mutation triggered re-evaluation, not the entire context. This is bounded (one layer, typically a few keys) and precisely relevant (it's the delta that caused the binding to fire).

The more interesting design question was where to store it. The obvious place is the event log metadata — it's audit data, and the event log is the audit trail. But the plan-items REST endpoint is the consumer, and joining plan items to event log entries at query time requires temporal correlation heuristics that break for repeatable bindings. Claude caught this during design review: if the same binding fires three times (legitimate in iterative case models), which `WORKER_SCHEDULED` event belongs to which plan item? Timestamp proximity is fragile.

The better answer: store the activation context directly on the `PlanItemRecord`. Each plan item carries its own activation snapshot from creation. No joins, no correlation, no ambiguity. The event log still gets the activation context in its metadata for audit completeness, but the plan-items endpoint maps it straight from the record.

While threading the activation context through five method signatures and two event types, we found a second structural issue. The engine had three separate CDI events for plan item terminal states: `PlanItemCompletedEvent`, `PlanItemFaultedEvent`, `PlanItemRejectedEvent`. Each carried different fields. Observers had to register for each one separately, and several transitions (CANCELLED, OBSOLETE) had no event at all. We consolidated into a single `PlanItemStateChangedEvent` carrying `previousStatus` and `newStatus` as strings — one event to observe, explicit filtering, every transition covered. Five handlers migrated, three event classes deleted.

The open question is whether the changed layer is enough. A binding's guard condition might reference data from multiple layers — the triggering layer is the delta, but the full condition context spans more. Full condition-referenced key extraction from arbitrary JQ expressions is the theoretically correct approach, but the engineering cost is high and the changed layer captures the operationally useful signal: what just happened that caused this to fire. The full multi-layer snapshot is there if we need it later — it's a parameter change, not an architecture change.
