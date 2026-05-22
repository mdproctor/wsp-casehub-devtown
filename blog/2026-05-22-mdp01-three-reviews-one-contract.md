---
layout: post
title: "Three Reviews, One Contract"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [casehub-devtown]
tags: [sla, design, code-review]
---

With the HITL lifecycle test passing, Layer 2 has a clear brief: add a formal
deadline and escalation path to the human review WorkItem. Currently the
`human-approval` binding creates a WorkItem with no expiry, no candidate group,
and no response when nobody acts. The case stalls silently. Layer 2 closes that.

## Where the SPI landed — not where I planned

I expected to put `SlaBreachPolicy` in a new `platform/apps-api` module — a
framework-free home for application-layer SPIs that aren't foundation primitives.
We spent time designing the module structure, filed the issue.

Then we looked harder at the dependency graph. Every consumer of this SPI already
has `casehub-work-api` on its classpath. `BreachedTask` — the value type the
policy receives — mirrors WorkItem fields directly. The SPI is inherently tied
to WorkItem expiry. Platform filed ADR-0007 confirming the placement:
`casehub-work-api` is right for now. The separate module waits until a genuinely
work-independent cross-app SPI appears.

## Chained, and the case for separating it

My spec embedded `thenOnBreach` on `EscalateTo` and `Extend` — the obvious
shape for chaining. The casehub-work implementation chose differently: a separate
`Chained` sealed record.

```java
record Chained(BreachDecision primary, BreachDecision fallback)
    implements BreachDecision {}

default BreachDecision thenOnBreach(BreachDecision fallback) {
    return new Chained(this, fallback);
}
```

Each decision type stays pure — no nullable continuations. The executor handles
chaining in exactly one switch arm. The fluent construction still reads naturally.

What I didn't see until we verified it against the execution model: sequential
multi-tier escalation doesn't need `Chained` at all. When the escalated WorkItem
expires, `ExpiryLifecycleService` calls `onBreach()` again. By then,
`ctx.task().candidateGroups()` already contains the escalation group set during
the first tier. The policy reads this:

```java
if (ctx.task().candidateGroups().contains(escalationGroup)) {
    return new Fail(terminalReason); // second breach — terminal
}
return EscalateTo.to(escalationGroup).withDeadline(escalationDuration);
```

No serialization, no state storage. The WorkItem's own fields carry the tier.
I confirmed the same pattern works for AML (compliance officer → head of compliance,
30-day FinCEN SLA) and clinical (Grade 3/4 adverse event, immediate fail on breach).
`Chained` still earns its place — for "escalate to this group or fail if it's
empty" within a single breach event — but the sequential case is simpler.

## Three rounds of review

The casehub-work team designed the `SlaBreachPolicy` wiring in parallel. Claude
reviewed their design three times.

The first pass found three issues. `Path.of()` with zero arguments throws —
they needed `Path.root()` as a null-scope fallback. It didn't exist on any
platform branch, despite the design document stating it was "already committed
to platform main." `preferenceProvider.preferencesFor()` is the wrong method
name — it's `resolve()`. And `EscalateTo` had no `deadline` field, so
applications couldn't control the escalated task's lifetime.

The second pass caught `SlaBreachEvent` firing with the `Chained` wrapper rather
than the leaf decision. An observer wanting to know "escalated or failed?" had to
traverse the Chained record. The fix: `executeBreachDecision` returns the leaf,
fires the event with that.

The third pass surfaced the sharpest issue. `BreachExecutionFailed` — the private
exception thrown when `EscalateTo(empty groups)` tries to execute — is only caught
by the `Chained` branch. A policy returning bare `EscalateTo.to()` without wrapping
it lets the exception reach the `@Transactional` boundary. The transaction rolls
back. Every WorkItem processed in that scheduler tick is un-done. On the next tick,
the same items appear again. Infinite retry, completely silent.

Two fixes: validate at the factory so `EscalateTo.to()` with zero groups throws
immediately at construction, and catch at the top-level dispatch to apply a `Fail`
rather than propagating.

`Path.root()` still isn't on platform main. Until it is, the SlaBreachPolicy
wiring won't compile, and Layer 2 implementation can't start.
