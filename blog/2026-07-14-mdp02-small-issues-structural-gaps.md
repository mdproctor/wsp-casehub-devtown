---
layout: post
title: "Small Issues, Structural Gaps"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub devtown]
tags: [cbr, merge-queue, mcp-tools]
---

## Small Issues, Structural Gaps

The S/XS backlog had grown to fifteen issues — individually small, collectively a drag on the roadmap. We swept through them in a single session on one branch: eleven commits, twenty-three files, the kind of focused batch work where each issue takes minutes rather than hours.

The interesting part wasn't the work itself. It was what the work revealed about where the foundation still has gaps.

## The CBR refinements

Four issues in the CBR Phase 2 cluster landed cleanly. Per-capability activation thresholds replace the global `minFindings`/`minFraction` with a `Function<String, ActivationThreshold>` — security-review can now have a lower bar than architecture-review without touching the policy logic. The threshold mechanism itself shifted from flat counting to similarity-weighted accumulation: a 0.95-similarity precedent now contributes 0.95 to the evidence sum, not 1.0. This makes the system naturally sceptical of weak matches.

The weight refinement step closes the CBR Revise loop. `CbrWeightAdjuster` applies EMA-based adjustments to similarity dimension weights after each outcome attestation — if file-path overlap keeps predicting findings that don't materialise, its weight decays. A minimum sample threshold of ten prevents premature adjustment from a handful of early cases.

## What the backlog exposed

Two issues I expected to close turned out to be structurally blocked — and the blockers are in the foundation, not in devtown.

`PrReviewCaseTracker` startup hydration (#127) needs the tracker to rebuild from durable state on restart. But both the tracker and the underlying `CaseInstanceRepository` are in-memory. There's no durable state to hydrate from, and the engine SPI has no list query for non-terminal cases. This is a prerequisite chain: durable persistence first, then a query API, then hydration.

SLA calibration from similar past assignments (#136) needs timing data — how long did each precedent's review take? The CBR pipeline stores outcomes and features but no timestamps. `PlanTrace` and `PlanCbrCase` in neocortex-memory-api carry no duration. Adding it means a cross-repo change through neocortex and into the engine's `CbrCaseRetainObserver`.

Neither of these is surprising in retrospect. The CBR pipeline was designed for similarity and outcomes, not for operational metrics. But it's a useful signal: as devtown moves from "can the system route?" to "how well does it route?", the foundation needs temporal data it currently discards.

## Eight new MCP tools

With the pages/blocks-ui dependency still blocking all UI work, I built the backend query layer as MCP tools instead. `find_similar_cases` wraps the CBR retrieval service. `search_memory_by_contributor` and `search_memory_by_capability` expose the case memory store. `get_agent_messages` queries the engine event log filtered to agent dispatch and completion events. The merge queue got `get_failure_rates_by_repository` and `evaluate_failure_rate_alerts` — the alert fires a CDI event when per-repository failure rates exceed a configurable threshold over a minimum batch count.

The tool count is now 19 read + 6 write. When pages/blocks-ui ships, these become the data layer the UI consumes. Nothing wasted.
