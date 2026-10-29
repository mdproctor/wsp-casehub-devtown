---
layout: post
title: "The Review Intelligence Workbench: What GitHub Can't Show You About Your PR"
date: 2026-10-29
entry_type: article
subtype: diary
projects: [casehubio/devtown]
tags: [ui, architecture, review-intelligence, timeline, cbr, trust-routing]
series: issue-216-review-workbench-ui
---

# The Review Intelligence Workbench: What GitHub Can't Show You About Your PR

GitHub shows you a diff. DevTown shows you what you need to know *about* that diff.

That's the one-sentence pitch for the Review Intelligence Workbench — the UI overhaul I built today for DevTown's governance layer. The backend intelligence has existed for months: trust-weighted routing rationale, CBR precedents from similar past PRs, contributor history with GitHub profile data, policy context explaining why *this* reviewer was selected for *this* capability. The gap was that none of it was surfaced coherently.

The old PR Cases tab was a flat list with a detail pane — structurally identical to what you'd find in any project management tool. Click a PR, see its status. That's fine for tracking. It's useless for *understanding*.

## The Design Problem

A code reviewer — human or AI — looking at a PR needs to answer questions GitHub can't:

- Has this contributor had security findings on similar code before?
- Why was *this* agent selected for the security review, and who else was considered?
- The agent found two findings — how often do similar findings turn out to be real issues?
- There's a human approval gate — what does the human need to decide, and how has this candidate group decided similar reviews historically?

All of this data existed in DevTown's services. `ContributorDetail` had the merge history and GitHub intelligence. `CbrRetrievalService` could find analogous past PRs. `TrustQueryService` had the routing rationale with candidate alternatives. `SlaCalibrationStore` had precedent-based SLA estimates. The intelligence was computed and stored — it just wasn't visible.

## The Timeline Spine

The core interaction model is a two-pane layout. The left side is a **timeline spine** — a vertical flow of active cases, each rendered as a case header followed by compact decision-event nodes.

![Review Intelligence Workbench layout](images/workbench-layout.svg)

Each case in the spine shows its lifecycle as clickable nodes: `● Opened`, `● Code analysis: security-sensitive`, `● Security review → agent-3`, `◐ Security review: 2 findings`, `○ Human gate — pending`. Click any node, and the right pane fills with contextual intelligence for that specific moment in the review lifecycle.

The spine has a smart default for expand/collapse. Cases with active work — in-progress reviews, pending human decisions — expand automatically. Completed cases collapse to a single row with a coloured status indicator. The idea is that the spine naturally adapts to volume: with five active cases, most are expanded and you see the full flow. At twenty cases, completed work compresses and you focus on what needs attention.

A toggle chip switches to a dense table view for ops scanning — the flat list isn't gone, it's just no longer the default.

## Why Flat Divs, Not Timeline Instances

The first instinct was to stack one `agency-timeline` component per case. The existing timeline component handles event ordering, category filtering, pagination, and detail expansion. Wrapping twenty of them in a scrollable container sounds reasonable.

It's wrong for this use case.

![Spine architecture comparison](images/spine-architecture.svg)

Each `agency-timeline` creates its own shadow DOM subtree with filter state, pagination logic, and endpoint binding. The spine doesn't need any of that — it's a flat list of case headers and event nodes. The per-case features (filtering, pagination, detail expansion) are exactly what *shouldn't* happen in the spine, where the goal is compactness and fast scanning.

The spine uses a dedicated `spineNodeStrategy` that synthesises compact labels from event metadata — "Security review → agent-3" is built by combining the event type, capability name, and actor from the `AGENT_ROUTED` event metadata. The full `agency-timeline` with `reviewTimelineStrategy` remains in the intelligence pane for the detailed view — the right component at the right layer.

The flat-div approach also makes virtual scrolling trivial to add later. Uniform-height divs in a single DOM tree are a standard windowing problem. Heterogeneous component subtrees with independent shadow roots would require recreating component instances during scroll, which defeats the purpose.

## The Intelligence Pane

Click a node, see its context. The intelligence pane is a single component that switches content based on node type. Five views, each with different data requirements:

![Intelligence pane data flow](images/intelligence-flow.svg)

**Case header** — click the PR itself. See the contributor's trust profile, CBR precedents (past PRs with similar code changes and what happened), code classification (security-sensitive? architecture-crossing?), and the routing policy that determined how this PR would be reviewed.

**Dispatch node** — click "Security review → agent-3". See why agent-3 was selected: its trust score for the security-review capability, how many open commitments it has, and — for completed reviews — the full candidate list with scores showing who else was considered and why they were excluded. In-progress dispatches show basic rationale from event metadata; the full candidate alternatives become available when the `WorkerDecisionEntry` is written on completion.

**Findings node** — click "Security review: 2 findings". See the findings themselves with severity, confidence, and file locations. Below that, CBR precedent comparison: "Similar finding in PR #67 three months ago — confirmed as real issue. 92% historical confirmation rate for token rotation findings in this codebase."

**Human gate** — click the pending human decision. See what needs deciding, who the candidate group is, the work item's SLA status, and how this group has decided similar reviews historically.

Each view fires its own parallel REST fetches. A lightweight dedup cache — a `Map<string, {promise, timestamp}>` with thirty-second TTL — prevents duplicate requests during click-around exploration. Click a dispatch node, click the case header, click the dispatch again — the second dispatch click returns cached data instantly.

## Two Backend Additions

The spec originally claimed "no new backend needed, just wiring." The review process caught two gaps.

First, `CbrRetrievalService.findSimilar()` was only exposed through the MCP/GraphQL layer — no REST endpoint for the frontend to call. I added `GET /api/devtown/reviews/{caseId}/similar-cases` to `DevtownReviewApi`, extracting PR parameters from the case's `PrPayload` so the frontend only needs to pass the case ID.

Second, the dispatch intelligence view needs routing rationale with candidate alternatives, which lives in `WorkerDecisionEntry` records. But the existing `ReviewDetail.RoutingDecision` record only carried `(capability, reason, confidence, bindingName)` — no link back to the `WorkerDecisionEntry` that has the full candidate scoring. I enriched the record with an `entryId` field, correlated by case ID and capability tag. For in-progress dispatches the entry doesn't exist yet (it's written on completion), so `entryId` is null and the intelligence pane shows "Full rationale available after review completes."

## The Tab Restructure

The old six-tab layout mixed operational monitoring with review intelligence with debug tools:

| Before | After |
|--------|-------|
| PR Cases | **Review Workbench** (spine + intelligence) |
| Reviewers | Reviewers (unchanged) |
| Contributors | Contributors (unchanged) |
| Workers | *removed — sessions fold into Operations* |
| System | System (refocused on calibration) |
| Definitions | *removed — accessible via URL* |
| — | **Operations** (problems, events, sessions) |
| — | **Merge Queue** (dedicated queue view) |

The Merge Queue tab is the key structural change beyond the workbench itself. The old design mixed queued PRs with review-lifecycle cases in the same list, differentiated by filter chips. Now they're in separate tabs with distinct components — the Review Workbench scopes to review-lifecycle cases, the Merge Queue shows queued PRs, active batches, and queue metrics with its own split-pane layout.

## What This Opens Up

The intelligence pane's architecture — node-type switching with parallel fetches and a dedup cache — is built to handle more node types without structural changes. When DevTown adds cross-repo coordinated merges, the parent case's coordination events will appear as spine nodes with their own intelligence views: sub-case status, rollback risk, and coordination decision rationale.

The CBR precedent data also gets more valuable as cases accumulate. Right now the confirmation rate is computed client-side from a small precedent pool. As the casebase grows, the "92% historical confirmation rate" for token rotation findings becomes a genuinely predictive signal, not just a curiosity.

The point isn't that DevTown replaces GitHub's diff viewer. GitHub shows the change. DevTown shows the *context* — what you need to know about the contributor, the code area, the review process, and the system's track record with similar changes. The Review Workbench makes that context a click away from any decision event in the review lifecycle.
