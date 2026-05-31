---
layout: post
title: "Layer 6: when trust scores route reviewers"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust-routing, casehub-platform, layer6, preferences]
---

The session started with cleanup — checking which of the blocking issues from the handover were actually still open. Engine#326, engine#337, engine#336, qhorus#199: all closed. Nothing blocking P1.3.

Then I asked an apparently casual question: the platform had been building casehub-memory recently, the CaseMemoryStore SPI. Were there opportunities for devtown?

What followed wasn't what I expected. Claude worked through the SPI — read the actual code, not just the issue descriptions — and came back with a concrete analysis: contributor history on PR open, reviewer agent semantic context, code area history. Then it spotted something the platform team hadn't thought through: the SPI is well-designed for the memory layer but the *surrounding design is missing*. No standard emission pattern. No attribute key conventions. No multi-entity recall. No documented position on `text` field semantics for semantic adapters.

I mentioned that I'd added casehub-memory almost by accident, without a clear picture of how other apps would use it. That gap is exactly why devtown matters as the reference application — it's the first consumer working out what the platform actually needs from the consumer side. We filed platform#48 before touching any implementation code. One of the better uses of half an hour this project has seen.

---

Layer 6 itself was the main work. The goal: wire `TrustWeightedAgentStrategy` from `casehub-engine-ledger` into devtown, backed by per-capability routing policies that reflect what's actually at stake in each review domain. Security review is not the same as style review. Merge is irreversible. The threshold, the observation minimum, the quality floors — these are not arbitrary.

The brainstorm surfaced one decision I hadn't planned on making: `DevtownTrustDimension.FALSE_POSITIVE_RATE`. The name implies higher = worse. Every other trust dimension in the industry convention is higher = better — recall, precision, calibration. Storing `false-positive-rate` as a raw rate inverts the quality floor semantics silently. We renamed it to `PRECISION` before any ledger data existed against the old name, which made the rename free. The right time to fix a semantic error is before it has consequences.

The other design decision that saved implementation pain: the spec review caught that `DevtownCapabilityRegistry.POLICIES` already holds threshold, minimumObservations, and borderlineMargin. Putting those same values in YAML would mean two places to change for any policy update — and they'd diverge. We split the responsibilities: the three base fields come from the registry, blendFactor and quality floors (absent from `RoutingPolicy`) go in YAML. The YAML file ended up half the length we'd originally planned.

One spec review finding I pushed back on. The reviewer claimed that an agent with a trust score exactly equal to the threshold and borderlineMargin=0.0 would be classified as `EXCLUDED_PHASE2B`. That's wrong. With `Math.abs(score - threshold) <= 0.0`, score at exactly the threshold evaluates to `true` — the agent is BORDERLINE, not EXCLUDED. The reviewer's proposed fix (use 0.001 as minimum margin) would have made the borderline zone wider and increased escalation risk for style-review, the opposite of the design intent. We kept borderlineMargin=0.0 for style-review and documented the floating-point rationale: Bayesian Beta trust scores are continuous floats, and realising exactly 0.50 isn't achievable in practice.

---

The implementation ran on subagent-driven development — nine tasks dispatched sequentially, each reviewed before the next started. Two code quality findings after the first batch: `allFloorKeys()` was allocating a new map on every call (should be a static constant), and the test asserting YAML load was choosing a threshold that matched the DEFAULT anyway and wouldn't catch a silent failure. Both fixed before moving forward.

35 tests pass. `TrustWeightedAgentStrategy` activates by classpath presence. `DevtownTrustRoutingPolicyProvider` displaces `DefaultTrustRoutingPolicyProvider` automatically. The YAML configures per-capability blend factors and quality floors. The LAYER-LOG Layer 6 entry is written.

Trust scores don't exist yet — Layer 4 (the attestation trail) isn't wired, so all agents are still in Phase 0 BOOTSTRAP, routing by availability. The infrastructure to move beyond that is in place.
