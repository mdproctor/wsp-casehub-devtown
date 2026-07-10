---
layout: post
title: "The Missing Step"
date: 2026-07-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cbr, similarity, memory, trust-routing]
series: issue-130-pr-similarity-model
---

The garden entry that started this was blunt: devtown's trust routing is degenerate CBR — Retain and Reuse, but no Retrieve and no Revise. Trust scores flatten per-case variation. A reviewer with 0.72 overall might be 0.95 on concurrency PRs and 0.40 on API design PRs, and the routing treats them as interchangeable.

I wanted structured similarity, not embeddings. Deterministic, explainable, auditable — every routing decision traceable through the existing ledger. The spec landed quickly: five dimensions of Jaccard-based comparison (file paths, modules, languages, change size, contributor), weighted and normalised. The design review ran four rounds, raised 21 issues, and the most consequential finding was that I'd planned to use the platform's `CbrCaseMemoryStore` without checking whether it could actually handle set-valued features. It can't — no `FeatureField.SetValued`, no custom similarity passthrough, no per-dimension breakdown. Four concrete gaps. We documented the migration path for when neocortex gains the missing type, and built on `CaseMemoryStore` instead.

The implementation surfaced one assumption the spec got wrong. The spec said to emit the feature vector before `caseHub.startCase()` with a pre-generated UUID. Claude found that `CaseHub.startCase()` returns the UUID — there's no overload that accepts one. The fix was straightforward: emit after `startCase()` returns the caseId, not before. The vector is immutable once the PR is submitted, so the ordering doesn't affect correctness.

`PrFeatureVector` extracts from primitive parameters — not from `PrPayload` directly, which would have created a domain→review module dependency the design review caught. The extraction maps file extensions to languages, normalises paths to modules via the existing `ModulePathNormalizer`, and detects test and config files by pattern. JSON array serialisation for the sets (not comma-separated — file paths can contain commas). Round-trip fidelity is the test that matters: `fromAttributes(vector.toAttributes())` must equal the original.

`WeightedJaccardSimilarity` computes all five dimensions, produces a `SimilarityScore` with a per-dimension breakdown map. Jaccard on two empty sets returns 1.0 — standard mathematical convention. Two PRs that both lack a feature are identical on that dimension. Weights default to modules=1.5 (strongest structural signal), file-paths=1.0, change-size=1.0, languages=0.5, contributor=0.5. All configurable via `PreferenceProvider` for Phase 4 weight refinement.

`DefaultCbrRetrievalService` scans `CaseMemoryStore` for case-vector facts, filters by repo and time window, scores against the query, ranks, and enriches the top-K with outcome facts from the contributor entity. The enrichment follows the same `MemoryQuery.forEntity("contributor:" + contributor).withCaseId(caseId)` pattern that `CaseMemoryRecaller` already uses — the design review verified this against the actual API.

`MemoryContext` now carries precedents alongside contributor and code area history. `hasRiskSignals()` checks precedent outcomes too — a past similar PR that failed is a risk signal. The whole CBR pipeline is fail-open: scan failure, scoring failure, enrichment failure — each returns empty, logged, case proceeds.

The four platform gaps in `CbrCaseMemoryStore` are worth tracking. When neocortex gains `FeatureField.SetValued` with Jaccard similarity, the migration is mechanical — only the storage and retrieval adapters in `app/` change. The domain model is untouched.
