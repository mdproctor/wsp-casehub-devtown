---
layout: post
title: "Learning From Your Own History"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cbr, precedent-activation, case-plan-model, trust]
series: issue-132-cbr-capability-activation
---

## Learning From Your Own History

The PR review CasePlanModel has always decided what to check based on what the code looks like right now. Content analysis reads the diff, flags security-sensitive files, and fires the security-review capability. If the analysis doesn't flag it, security review doesn't run.

That's fine until your third auth-adjacent PR in a row gets through without a security pass because the changed files don't pattern-match to the content analyzer's idea of "security-sensitive." The humans catch it in retrospective. The system doesn't learn.

CBR Phase 1 (#130, #131) built the retrieval service — finding similar past cases by module overlap, language, change size, and contributor. Phase 2 is about using what those precedents tell us. The core question: if 3 of 5 similar past PRs had security findings, should this PR get a security review even though the code diff doesn't look security-sensitive?

The answer is obviously yes. The implementation had two non-obvious decisions.

First, the signal. "Was this capability activated" is weaker than "did this capability find problems." If similar PRs all got security-reviewed but passed clean, that means content analysis is already doing its job for this class of PR — precedent activation would be redundant. The useful signal is findings-based: similar PRs had actual security issues. This required enriching `Precedent.capabilityOutcomes` from raw outcome strings to a `CapabilityOutcome` record carrying both outcome and detail, with a `hadFindings()` method as the single source of truth.

Second, the gating problem. The design review caught a critical flaw I'd missed. The existing goal conditions use the pattern `securitySensitive == false OR securityReview.outcome == "APPROVED"`. When the precedent binding fires (because `securitySensitive == false`), the first branch of that OR is already true — the case would complete and merge while the precedent-triggered review is still running. The fix: `(securitySensitive == false AND securityReview == null) OR outcome == "APPROVED"`. "Security is not needed" only holds when no review exists at all. Once any binding creates the review, the case gates on its outcome.

The pre-computation decision was also worth recording. Precedent data is immutable during the case lifecycle — set once at case creation, never invalidated by `revisePr()`. Evaluating thresholds once at recall time and injecting the result as `memory.precedentActivations` keeps threshold arithmetic in testable domain Java rather than buried in binding condition lambdas parsing serialized maps from the case context.
