---
layout: post
title: "Supersede, Score, Split"
date: 2026-07-14
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub devtown]
tags: [lifecycle, cbr, merge-queue, bisection]
---

## Supersede, Score, Split

Three issues, one branch, one theme: the merge queue and PR lifecycle need to know about history.

The supersede work adds a missing lifecycle state. When a contributor closes PR #42 and opens #43 as its replacement, the old case should die with a forwarding address. `SUPERSEDED` joins `COMPLETED`, `FAULTED`, and `CANCELLED` as a terminal state, but it carries links — `supersededBy` and `supersedes` — that connect the two cases in the audit trail. The tracker, the service port, every implementation layer, and a new MCP tool all got the same treatment. The design choice worth noting: `SUPERSEDED` is a devtown-level status, not an engine concept. The engine sees `CANCELLED`; devtown knows *why* it was cancelled and who replaced it.

Risk scoring and bisection heuristics are both CBR Phase 3 — extending case-based reasoning into the merge queue. The risk assessor is simple: find similar past cases, count how many caused batch failures, return the ratio. A PR where three of five historical matches failed gets a 0.6 risk score. The batch composition policy uses this to prevent grouping multiple high-risk PRs together — one per batch, so a failure is isolated rather than compounded.

The more interesting change was the bisection API. `BisectionSplitStrategy` and `BatchSlice` had been carrying `List<Map<String, Object>>` since their first draft — untyped maps flowing through the entire bisection chain because the engine's context is map-based. We replaced them with `List<QueuedPr>` throughout, converting at the engine boundary in `MergeBatchCaseHub`. The `PrecedentBisectionStrategy` that followed was clean to write because it could sort on `riskScore()` directly instead of casting from a map.

The precedent strategy itself is straightforward: sort candidates by risk score descending, split at the midpoint. The highest-risk PR lands in the left half — the half tested first during bisection. When no risk data exists (all scores 0.0), it falls back to binary split. No magic, just informed ordering. The average bisection round count drops because the most likely culprit is tested first rather than discovered by elimination.
