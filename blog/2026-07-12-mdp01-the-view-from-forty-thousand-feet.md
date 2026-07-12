---
layout: post
title: "The View from Forty Thousand Feet"
date: 2026-07-12
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust, evidential-checking, roadmap, project-management]
series: issue-141-evidential-checker-v1-v4
---

*Part of a series on [#141 — EvidentialChecker V1-V4 integration](https://github.com/casehubio/devtown/issues/141). Previous: [The Missing Step](2026-07-10-mdp01-the-missing-step.md).*

I came in to implement #141 — the evidential checking layer for low-trust agents. I left with a cross-repo roadmap, a GitHub Project board, and a Gantt chart, having written zero lines of production code.

That's not a complaint. The design work was overdue.

**The design itself is straightforward.** `TrustGatedAttestationPolicy` modulates confidence but always returns SOUND for DONE claims. A below-threshold agent claims it finished a review, gets SOUND at reduced confidence, and the system has no way to verify the claim is real. `EvidentialChecker` already exists in qhorus — V1 through V4 checks that verify an agent actually did something: messages exist on the channel, the obligation wasn't already failed, artefacts exist, tokens match. The integration is a new `EvidentialAttestationPolicy` in devtown's `app/` module that wraps the trust-gated policy, classifies the agent's trust phase, and runs V1-V4 when the phase matches a per-capability configuration.

Three decisions shaped the design. First: the evidential check scope is configurable per capability via a new `Set<TrustPhase>` field on `TrustRoutingPolicy` in engine-api. Security reviews check BOOTSTRAP + BELOW_THRESHOLD + QUALITY_FAILED agents. Style reviews check nobody. The risk profile drives the verification intensity, not a global switch. Second: evidential violations produce FLAGGED at fixed 0.8 confidence — not scaled by trust score. Zero messages on a channel is a structural fact, not a probabilistic assessment. Third: the composition is delegation, not replacement. `EvidentialAttestationPolicy` wraps `TrustGatedAttestationPolicy` at `@Priority(2)`, classifies independently using the routing policy's public API, and only intervenes when violations are found.

Of four V1-V4 checks, two fire immediately (V2: channel has messages, V3: not confirming a failed obligation) and two are structurally ready but inert (V1: artefact UUID not in `CommitmentContext`, V4: message content not passed to the attestation policy). Filed qhorus#342 for the upstream enrichment.

**Then the roadmap happened.** I asked what cross-repo issues we need to track. Pulling the full backlog across engine, qhorus, ledger, blocks, blocks-ui, pages, work, worker, claudony, platform, parent, and connectors revealed something I hadn't seen in aggregate before: the UI pipeline is the longest critical path in the entire project. `pages#111` → `blocks-ui#49` → `blocks-ui#41` → every devtown UI feature in every phase. The engine work I'd been focused on (engine#711 for this issue, engine#548 for composed goals) is not even on the critical path. The trust visibility UI, the CasePlanModel browser, the case dependency graph, the agent inbox — all of them are gated on blocks-ui#41, which is gated on five pages infrastructure issues that haven't started.

I phased the roadmap around demo-able capability — each phase should let you show someone the system doing something real. Phase 1 is trust intelligence: an agent claims DONE, the system catches the lie, trust degrades, routing shifts. Phase 2 is the operational merge queue: PRs batch, test, bisect on failure. Phase 3 is agent coordination: browse the fleet, search past decisions, manage worker sessions. Phase 4 is resilience: failure recovery, time-travel context, alerting. But the UI work in each phase is blocked on the same pipeline, and nothing I do in devtown unblocks it.

The ARC42 stale scan at wrap found six references to closed issues still marked as pending or blocked. Chapters 3, 4, and 5 had "full chapter entry pending — content to be written from devtown#52/73/57" annotations from months ago. The issues closed. The content was partially there. The annotations never got cleaned up. I wrote the full chapter entries and removed the stale language. The `TrustGatedAttestationPolicy` section still said "deferred — blocked on qhorus#307" even though both #307 and devtown#97 shipped.

The Gantt chart made one thing visually undeniable: foundation backend work runs in parallel and finishes fast. The pages → blocks-ui pipeline is serial and long. If I want the demos to include a UI, the pages work needs to start now — not after Phase 1 backend is done.
