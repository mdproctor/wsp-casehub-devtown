---
layout: post
title: "The notification system I almost built"
date: 2026-07-20
type: phase-update
entry_type: note
subtype: diary
projects: [devtown]
tags: [platform-coherence, notifications, design-review]
---

I spent the first hour of this session designing a notification system from scratch.

The requirement was straightforward — six notification scenarios for devtown: review assignments, merge success and failure, stalled commitments, case faults, SLA escalation. Wire them to Slack and Teams via casehub-connectors. I'd studied how casehub-work and casehub-clinical handle their notifications. Work uses a fire-and-forget dispatcher with notification rules. Clinical uses durable delivery with retry, ledger audit, and skip-path tracking. I mapped the devtown scenarios into two tiers — informational (fire-and-forget) and audit-critical (durable) — and was deep into designing the target resolution cascade when the question arrived: "we already have notifications in platform?"

We did. `casehub-platform` has a full notification subsystem — `SubscribableEvent`, `SubscriptionStore`, `NotificationDispatcher`, `DeliveryChannelRegistry`, `ChannelRouter`, `SuppressionEvaluator`, `DigestFlushScheduler`, user preferences per channel, engagement tracking. Everything I was proposing to build, including several capabilities I hadn't thought of yet.

Claude had missed it entirely. Four research subagents explored connectors, clinical patterns, platform docs, and devtown events — none surfaced the platform notification modules. The connectors repo doc mentioned `NotificationDispatcher` in passing; Claude read it as casehub-work's dispatcher, not the platform's. The failure mode is instructive: Claude explored what it was asked to explore (connector SPIs, clinical patterns) and confidently synthesised from that subset. The platform notification system was 30 lines away in the casehub-platform directory listing, but no agent was pointed there.

The corrected design is dramatically simpler. devtown provides `SubscribableEvent` POJOs and subscription definitions. The platform handles everything downstream — target resolution, suppression, digests, channel routing, delivery tracking. Where I'd planned a target resolution cascade, config properties, retry logic, and ledger integration, the spec now says: fire a CDI event, register a subscription, done.

The adversarial design review caught a second class of problem. Round 1 raised 14 issues — and most weren't design flaws. They were fabricated APIs. `engine.insert()` doesn't exist (the subscription engine isn't built yet). `registerIfAbsent()` doesn't exist on `SubscriptionStore`. `EscalationPolicy` isn't a real class — the actual mechanism is `SlaBreachEvent`. `CaseLifecycleEvent.status()` should be `caseStatus()` and returns a `String`, not an enum. The spec was internally consistent but grounded on phantom APIs — method names that sound right but don't compile.

The fix pattern was the same each time: IntelliJ MCP → `ide_find_definition` → read the actual record/class signature → rewrite the spec against verified types. By round 5 the reviewer approved — all 19 findings verified in the updated spec. The one accepted finding was a container heading with no actionable change.

Implementation after the review was clean — 20 files, three commits, all tests passing on the first full build. The verified API signatures meant zero surprises: `WatchdogConditionType.OBLIGATION_FAN_OUT` and `CONVERSATION_STALL` are the correct enum values, `BreachType.COMPLETION_EXPIRED` not `COMPLETION`, `WorkItem.scope` is a `String` not a `Path`. Every field name had been checked twice — once by the reviewer, once by the implementor.

The gap that remains: the platform's `SubscriptionEngine` and `NotificationDispatcher` aren't implemented yet. devtown's event POJOs and subscriptions are registered and tested, but the matching-and-dispatch pipeline won't fire until the platform ships it. And the connector delivery bridge (connectors#86) is in progress separately. When both land, devtown's notifications activate without any additional devtown code.

Two things I want to remember from this session. First: "does the platform already have this?" is the question that should come before any design, not after. Claude's research was thorough but scoped to what I asked about — connectors and existing app patterns. Nobody asked "what does casehub-platform provide for notifications?" until the user did. Second: adversarial design review on AI-generated specs is not optional. A spec that reads well, has consistent internal logic, and passes a self-review can still be grounded on APIs that don't exist. The review's value wasn't in finding design flaws — the design was sound. It was in discovering that the implementation would not compile.
