---
layout: post
title: "The vocabulary problem"
date: 2026-05-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
---

The first thing devtown needed was a vocabulary — a set of names for the types of work that happen in software development. The original spec had thirteen: `code-analysis`, `security-review`, `human-approval-gate`, `notify`, `batch-bisect`, and eight others. Flat. One namespace.

I didn't like it.

The problem isn't the names — most of them are right. The problem is that they're not the same kind of thing. `security-review` is something an agent is *qualified* for, trust-scored on, with a meaningful minimum threshold. `batch-bisect` is a structural move a case plan makes when tip-of-batch CI fails. `notify` is a call to a Slack connector. Treating all thirteen as equivalent meant we'd end up calling `TrustGateService.meetsThreshold()` on `notify`. That makes no sense.

I went through the list with Claude. We ended up with four typed classes: `ReviewDomain` (what analytical work a PR needs), `AgentQualification` (execution capabilities with trust scoring), `HumanDecision`, and `HumanOversight`. The last two took argument.

Claude's first suggestion was to drop `human-approval-gate` entirely — it's a routing constraint, not a capability, `ActorType.HUMAN` handles it. I pushed back. Human review in a code review system is not a routing detail. A named person formally approving a PR is a compliance event. The `casehub-work` WorkItem lifecycle — SLA, delegation, escalation, business hours — is purpose-built for this. We ended up elevating it: `HumanDecision`, first-class, with its own actor enforcement and its own trust accumulation.

`HumanOversight` emerged from a harder question. If an agent's trust score is 0.71 and the threshold is 0.70, do you route to them? Maybe. But what if they only have three attestations? The Bayesian Beta prior hasn't stabilised. A score of 0.71 from three events is noise; from fifty it's signal. `RoutingPolicy` carries `minimumObservations` for this: don't treat a score as meaningful until there's enough history behind it. When an agent is within `borderlineMargin` of the threshold — qualified but barely — `HumanOversight` fires to spot-check the routing decision. Not to block it; to validate it.

The challenge I kept coming back to was practical. A new deployment has no trust history. If routing requires 0.70 trust and ten observations, everything blocks on day one. That's worse than Gastown.

We formalised this as a four-phase maturity model. Phase 0: availability routing, identical to Gastown. Phase 1: threshold enforced for agents with history, availability for new ones. Phase 2: full enforcement with borderline detection. Phase 3: per-capability quality floors. The `RoutingPolicy.isBootstrap(int)` method makes the current phase explicit:

```java
public boolean isBootstrap(int agentObservations) {
    return minimumObservations.isPresent()
        && agentObservations < minimumObservations.getAsInt();
}
```

Routing code doesn't need to know about thresholds and margins. It just asks the policy. The system works identically to Gastown on day one and improves automatically as evidence accumulates.

On the implementation side, everything landed in `devtown-domain` — pure Java, no Quarkus — with a thin `@ApplicationScoped` wrapper in `devtown-app` for CDI. The one interesting catch: we'd written `isKnown()` to call the backing static field directly, bypassing the virtual `capabilities()` method. Claude caught it in review. Any subclass overriding `capabilities()` would have gotten a broken `isKnown()`. Making it a default method on the SPI interface is the proper fix — still pending before the first routing consumer wires against it.
