---
layout: post
title: "The Spec That Fixed Itself"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [ci-aggregation, github-api, spec-review, cdi]
series: issue-89-github-combined-status
---

*Part of a series on [#89 — GitHub REST API combined-status check](https://github.com/casehubio/devtown/issues/89). Previous: [The Security Layers That Don't Talk](2026-06-22-mdp02-the-security-layers-that-dont-talk.md).*

devtown's CI status aggregation had a quiet bug. When a `check_suite` webhook arrived with `conclusion: success`, the service signalled `ci.status = "passing"` immediately — even if other suites hadn't reported yet. First suite in, full green light. The root cause was obvious once you looked: an in-memory `ConcurrentHashMap` tracking suite results locally, with no knowledge of how many suites to expect. GitHub knew; we didn't ask.

The fix was architecturally straightforward — a `CiStatusClient` SPI in `domain/` (alongside the existing `MergeClient`), implemented by `GitHubCiStatusClient` in the `github/` module, querying the Checks API for the authoritative combined status. The in-memory map goes away entirely. GitHub is the single source of truth; we read it instead of replicating it.

What made this interesting wasn't the implementation — it was what the spec reviews caught before any code was written.

## Three rounds, four real bugs

The first draft treated every non-`success` conclusion as a failure. That's wrong. GitHub check suites can complete with `neutral` (Codecov, advisory linters) or `skipped` (path-filtered workflows that intentionally didn't run). Under the original rule, any repo using informational checks would get false CI failures. The fix: a `NON_BLOCKING` set — `success`, `neutral`, `skipped` — with everything else (`failure`, `cancelled`, `timed_out`, `action_required`, `stale`) treated as blocking.

The pagination handling was worse. The spec said "log a warning if `total_count` exceeds the returned list." That's the same aggregation gap bug wearing different clothes — aggregating an incomplete picture and hoping for the best. The conservative fix: if the page is incomplete, return `Pending`. Set `per_page=100` to push the threshold up, but the fallback has to be safe.

The SPI parameter convention was inconsistent. I'd proposed `getCombinedStatus(String repo, String headSha)` with a combined `"owner/repo"` string, while the peer SPI in the same package — `MergeClient.merge(String owner, String repo, ...)` — takes them separately. One `split("/")` at the call site costs nothing; two conventions in `io.casehub.devtown.domain` costs every future SPI author a decision.

The error handling was missing entirely. The in-memory approach can't fail with a network error. The new design introduces a REST call in the webhook processing path. If GitHub returns 403 (rate limit) and the implementation throws, the webhook handler returns 500, GitHub retries, and you get a retry storm against a rate-limited API. The fix: an `Unavailable(String reason)` variant on the sealed result type, following the `MergeOutcome.Failure(reason)` precedent. The service returns 200 to GitHub — webhook accepted, status indeterminate — and the next webhook retries the query.

## The annotation that was load-bearing

`PrReviewCaseService` carries `@Alternative @Priority(2)` — an anti-pattern that ARC42STORIES §8 explicitly calls out. Since I was already modifying the class, I removed it. Tests went red.

The problem: there are three implementations of `PrReviewApplicationService`. `PrReviewService` has `@DefaultBean`. `QhorusPrReviewService` has `@Alternative @Priority(1)`. `PrReviewCaseService` had `@Alternative @Priority(2)`. Remove `@Alternative` from the last one and CDI silently resolves to `QhorusPrReviewService` — activated alternatives take precedence over regular beans. No `AmbiguousResolutionException`. No warning. The symptom was a test asserting an empty tracker — completely disconnected from the cause.

The fix can't be done one bean at a time. All three implementations participate in a priority chain where every annotation set matters. I reverted and moved on — that's a separate piece of work.

The four conclusions from the spec probably matter more than the implementation itself. Specs are cheap to review; production bugs in CI aggregation are not.