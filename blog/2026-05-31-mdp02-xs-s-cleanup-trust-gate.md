---
layout: post
title: "Seven issues off the board, one trust gate wired"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust-gate, qhorus, layer-6, cleanup]
---

Batch work day. I wanted to close a pile of S and XS issues before moving to Layer 4, and the result was less about new code than about finding out which issues had already been answered by earlier work.

## The dead issues

Three closed as superseded: capability tags (#2), routing thresholds (#4), and trust dimension definitions (#6). All three were written before Layer 6 shipped. `ReviewDomain`, `RoutingPolicy`, and `DevtownTrustDimension` — everything those issues described, we'd already built differently and better. I added closing comments explaining the supersession rather than just closing them silently.

One (#34, casehub-persistence-hibernate) turned out to be a more interesting dead-end. The issue described adding the hibernate persistence module to fix Quarkus augmentation failures. devtown#40 fixed those by a different mechanism — `@ApplicationScoped` subclasses of the in-memory SPI implementations. But there was still the question of whether we should add the JPA module for production. When I looked at it properly, the engine JPA repositories use Hibernate Reactive, which requires PostgreSQL. H2 doesn't work with reactive drivers. So "add the dep" doesn't get you to production Hibernate — it's a full datasource design problem. Closing with a detailed explanation, because the next person to look at it should understand why it's not an XS fix.

One more (#18, composite capability registry) the issue itself said not to implement until a second contributor exists. Fine — leave it.

## Moving DoublePreference out of domain.trust

Before the gate work, a small but overdue fix (#59): `DoublePreference` was in `domain.trust` and `IntPreference` in `domain.sla`. Neither has any coupling to those domains — they're generic scalar preference types. We moved both to `io.casehub.devtown.domain.preferences`, updated all import sites via IntelliJ's refactoring tools, 82 domain tests pass.

It was a ten-minute fix that I'd been avoiding because it felt trivial. It wasn't trivial to leave wrong — that kind of cross-domain import leaks into test helpers and new code without anyone noticing.

## The trust gate

devtown#58 was the real work: enabling the Qhorus trust gate for devtown agents. The gate sits between routing and commitment — routing selects the best available agent for a capability, the gate decides whether that agent is allowed to receive a COMMAND at all.

The first design decision: should the gate be global or per-capability? The answer is structural. `ObligorTrustContext` — the object passed to `ObligorTrustPolicy.permits()` — carries the obligor ID, channel ID, and channel name. No capability tag. `pr-review-{n}/work` identifies a PR-scoped channel, not a capability. You can't implement per-capability gate logic because you don't know what capability is being invoked at gate time.

Which also means the gate and routing aren't duplicating each other. Routing applies per-capability thresholds at selection time. The gate applies a global floor at commitment time. Different questions, different layers.

Bootstrap exemption was the other design question: what do we do with agents who have no trust history at all? `TrustGateService.currentScore(actorId)` returns `Optional.empty()` when there are no ledger observations. That's the signal. If there's no data, permit — the platform rule is "never block on missing trust data." The three-branch logic in `permits()` falls out cleanly: gate disabled if floor ≤ 0.0, permit if `currentScore` is empty (bootstrap), check score against floor otherwise.

One thing I'd misread in LAYER-LOG before we started: Layer 6's gap note says devtown#58 addresses the merge-executor bootstrap risk. It doesn't — can't, structurally. A bootstrap agent dispatched to execute a merge passes the gate because `currentScore` is empty and the gate exempts bootstraps. The merge-executor bootstrap problem is in the routing layer, not the gate layer. I filed devtown#62 for that separately: when no agent meets `minimumObservations` for `merge-executor`, route to `HumanOversight.ROUTING_REVIEW` instead of falling through to an unproven agent.

The gate configuration itself goes in `trust-gate.yaml` — a separate file from `trust-routing.yaml`, one YAML file per logical concern. Floor is 0.30, deliberately low. This is a safety net for agents with a consistently poor recorded history, not a quality filter. Routing handles quality.

Integration testing surfaced a trap: `ActorTrustScoreRepository.updateGlobalTrustScore()` sounds like the right way to seed a trust score for testing. It isn't. The method patches `globalTrustScore` (the EigenTrust share field) on an existing row — it doesn't create rows, and it doesn't set `trustScore`, which is what `TrustGateService.currentScore()` reads. Calling it on a non-existent actor is a silent no-op. `upsert()` with `ScoreType.GLOBAL` is what actually seeds a readable global trust score.

We ended up with 46 tests: 6 unit tests (all branch paths, pure Java, no Quarkus), 4 integration tests (CDI wiring, bootstrap permit, below-threshold deny, global score contract). Claude spotted that the integration tests were missing the at-floor boundary case — score exactly equal to 0.30 should permit, and while the unit tests covered it, the integration tests didn't.
