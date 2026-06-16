---
layout: post
title: "The Feedback Loop Nobody Noticed"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [gastown, trust-routing, documentation, demo]
---

Six foundation layers shipped. The closed feedback loop — trust scores updating from agent behaviour, driving future routing without human intervention — is live. And the docs still described it as aspirational.

I'd been meaning to refresh the Gastown comparison documents for a while. They were last updated in late May, and a lot shipped since: Layer 3 (qhorus messaging), Layer 4 (ledger audit), Layer 6 (trust routing), GDPR erasure, the ActionRiskClassifier, the compliance report endpoint. The handover still listed devtown#3 (TrustWeightedSelectionStrategy) as the next thing to pick up.

## The zombie issue

That was the first surprise. Issue #3 described a `TrustWeightedSelectionStrategy implements WorkerSelectionStrategy` — a devtown-local class. What actually shipped was `TrustWeightedAgentStrategy` in `casehub-engine-ledger`, a foundation capability with a completely different SPI. The work was done under #57 (Layer 6), not #3. The issue had been sitting open for weeks as a zombie — everything it described was implemented, just tracked differently. The SPI names evolved during foundation development and the issue never caught up.

Closed it with a comment mapping each acceptance criterion to what shipped. Every item covered, most exceeded.

## Twenty-nine stale items

The systematic scan across all four documents — `gastown-casehub-analysis-v2.md`, `PROGRESS.md`, `orchestration-advantages.md`, and the v1 analysis — turned up 29 stale items across 8 sections. P1.3 (trust routing wired) still showed as pending. The orchestration advantages doc still described engine#186 as "the one remaining gap" — it shipped in May. The cross-deployment trust federation entry in §4.4 contradicted §7 in the same document.

The staleness wasn't surprising — we hadn't synced in three weeks. What was surprising was the competitive picture it revealed once corrected.

## The narrowing

When v2 was written, Gastown had five real advantages over CaseHub. After updating for everything that shipped:

- Cross-deployment reputation — parity (TrustExportService shipped)
- Escalation-during-wait — closed (work#225, engine#349 both done)
- Trust routing enforcement — closed (Layer 6)
- Notification consolidation — closed (parent#5)

What remains: recovery automation, concurrency throttling, and operational tooling. All operational. Everything CaseHub lacks can be built on its existing foundation. What Gastown lacks — the normative layer, the trust model, the compliance infrastructure — requires a rewrite.

## The v3 rewrite

I wrote a new `gastown-casehub-analysis-v3.md` from scratch rather than patching v2. Nine sections. The structure shifted: instead of leading with the comparison tables, it leads with what's built and working, then the comparison, then where each side leads, then the roadmap. A new §8 covers the critical path to demo — what needs building (three S-scale issues: mock workers, trust seeding, demo script) to get to a point where the architectural advantages are demonstrable.

Two demo paths: one for platform engineers (the closed feedback loop — submit a PR, watch trust routing improve by operating), one for engineering managers (compliance, audit trail, human oversight gates, GDPR erasure with tamper-evident receipts).

## The UI question

Searching for Gastown's UI surface turned up two community projects — gastown-gui (web) and gastown-viewer-intent (React + TUI). Neither is first-party. Both are operationally focused: bead tracking, agent status, convoy progress. Neither can show trust scores, obligation lifecycles, routing rationale, or compliance evidence — because Gastown doesn't have those concepts.

We have `casehub-ui` with composable panels. A demo UI built on it would show things no Gastown dashboard can: the trust score shifting in real time as the second PR routes differently, the commitment timeline from COMMAND to FULFILLED, the ActionRiskClassifier gating a consequential action.

The UI requirements spec (`devtown-ui-requirements.md`) landed with four phases: demo-ready (5 panels), operational depth (9 panels including reviewer profiles and SLA dashboard), Gastown parity (11 panels), and beyond-Gastown (4 panels exploiting structural advantages like trust trends, maturity heatmaps, and the binding evaluator). Shared panels go to a commons package; devtown-specific panels stay domain-scoped.

The critical path to demo turned out to be three small issues and a UI composition. The architecture is done. What's missing is glue.
