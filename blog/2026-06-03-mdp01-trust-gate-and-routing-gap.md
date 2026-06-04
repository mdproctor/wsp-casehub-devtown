---
layout: post
title: "The Trust Gate and the Routing Gap"
date: 2026-06-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust-routing, casehub-engine, architecture]
---

The trust gate I shipped last session does the right thing: agents with no ledger
history are always permitted. You can't build a trust floor on nothing. Blocking a
bootstrap agent means blocking every agent the first time they touch a capability —
that's not a safety policy, it's a startup tax.

The issue I was sitting with in devtown#62 is different. The gate operates after
routing selects an agent. The routing strategy — `TrustWeightedAgentStrategy` — scores
BOOTSTRAP candidates by workload: `1 / (1 + runningJobs)`. Always positive, always
assignable. The gate sees them, grants permission (correctly — no trust history), and
a zero-history agent executes a merge.

Merge is irreversible. That's the gap.

The obvious devtown fix is a new `@Alternative @Priority(N)` routing strategy: check
whether all candidates are in BOOTSTRAP phase for a capability with a declared
`fallbackType`, escalate to human oversight before scoring. But reading PLATFORM.md
stopped me — `@Priority(2)` is already reserved for `SemanticAgentRoutingStrategy` in
`casehub-engine-ai`. A devtown strategy at `@Priority(3)` would override it. More
importantly: the domain model already anticipated this. `DevtownCapabilityRegistry`
declares `merge-executor` with `fallbackType = Optional.of(HumanOversight.ROUTING_REVIEW)`.
That field exists, is set correctly, and is never used.

The fix belongs upstream. We traced the gap with Claude: `TrustRoutingPolicy` (the
foundation record) has no `bootstrapFallbackType` field, so `DevtownTrustRoutingPolicyProvider`
has nowhere to pass the value. Add the field to `TrustRoutingPolicy` in engine-api,
check it in `TrustWeightedAgentStrategy.select()` before scoring, and devtown maps
`RoutingPolicy.fallbackType()` through the provider — one method call. Every domain
app gets the behaviour automatically. Filed as engine#415.
