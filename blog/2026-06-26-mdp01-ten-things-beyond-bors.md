---
layout: post
title: "Ten Things Beyond Bors"
date: 2026-06-26
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub DevTown]
tags: [merge-queue, gastown, casehub-engine, trust-routing]
---

Gastown's Refinery is production software. v1.0.1, battle-tested, doing exactly what it was built to do. It batches PRs, tests the tip of the batch, bisects on failure, rejects the culprit, merges the rest. The algorithm is Bors-style and it works.

The merge queue spec for devtown (Epic 4, devtown#11) doesn't replicate Refinery. It takes a structurally different approach — a CasePlanModel where the strategy is binding conditions, not compiled code. The architecture enables ten capabilities that go beyond what batch-then-bisect traditionally provides.

## What I wanted to get right

The merge queue is the first CaseHub feature where the Gastown comparison stops being theoretical. Layers 1–6 proved the foundation works: trust routing, human review gates, typed agent messaging, tamper-evident audit. But those are infrastructure — they don't do the thing Gastown actually does in production. A merge queue does.

I wanted the spec to be precise enough that someone reading it could verify every claim against the engine's actual schema. That turned out to be harder than expected.

## The two-tier split

The key design decision was separating the merge queue into two tiers: an imperative queue service and a reactive CasePlanModel.

Queue management — admission, priority ordering, batch formation — is scheduling. "Select the next N PRs satisfying these constraints, respecting a dependency DAG, weighted by trust scores with starvation prevention." That's an imperative algorithm. Shoehorning it into reactive binding conditions produces bindings that are really just procedural code pretending to be declarative.

Batch processing — test the tip, bisect on failure, merge on success, reject the culprit — is reactive. "If test failed and batch > 1, bisect." That's exactly what CasePlanModel bindings do well. The strategy IS the binding conditions. Different repos get different strategies by swapping the CasePlanModel, not by redeploying the queue service.

The split maps to the module structure: `devtown-merge` for the queue service, `merge-batch.yaml` for the CasePlanModel.

## Trust-weighted bisection

This is the headline architectural difference.

Traditional Bors-style merge queues bisect mechanically — split the batch in half by position, test both halves. The split uses no information about which half is more likely to contain the culprit. Every bisection round costs a CI run.

CaseHub has trust scores. Every PR author has a Bayesian Beta score from Layer 6 that reflects their historical review outcomes. A low-trust author's PR is statistically more likely to break the batch. By sorting PRs by trust before splitting, the low-trust PRs cluster in one sub-batch.

The worked example that convinced me: a batch of 8 where the culprit has trust 0.35 and everyone else is 0.7+. Trust-weighted bisection isolates the suspect half immediately. With two culprits, trust-weighted bisection clusters them in the same sub-batch — mechanical bisection can split them across halves, requiring both sub-trees to fully recurse.

The split itself is pluggable — a `BisectionSplitStrategy` SPI with three implementations. `TrustWeightedSplitStrategy` sorts by trust and splits at the midpoint. `IsolateOutlierStrategy` checks if one PR's trust is >2σ below the batch mean and isolates it solo. `BinarySplitStrategy` is the traditional positional split, kept for benchmarking.

## The schema alignment problem

The first version of the spec invented fields that don't exist in the engine's YAML schema. `workerContext` — not a real field. `{{ .batch.prs }}` — Jinja-style templates in a JQ-only schema. M-of-N sub-case group fields (`groupId`, `totalInGroup`) that exist in the Java API but aren't exposed in YAML.

Claude caught these during a schema-alignment review against the actual `CaseDefinition.yaml`. Every binding was checked against the schema's `unevaluatedProperties: false` constraint. The fixes were structural, not cosmetic:

- Replaced `workerContext` with proper capability `inputSchema`/`outputSchema` — the pattern already established by `pr-review.yaml`
- Separated bisection output keys (`bisectLeft`/`bisectRight`) to avoid LAST_WRITER_WINS collision
- Replaced manual retry counters with `outcomePolicy` reroute — the engine already tracks this natively
- Filed engine#574 to expose the M-of-N sub-case group fields in the YAML schema and fix per-child outputMapping

That last one is interesting. The engine's runtime fully supports M-of-N grouped sub-cases — `SubCaseExecutionHandler.handleGrouped()`, `SubCaseGroupRepository`, `SubCaseGroupPolicy` all work. But `CaseDefinitionYamlMapper.convertSubCase()` doesn't map the group fields. And `SubCaseCompletionService.handleGroupedCompletion()` only applies `outputMapping` from the child that triggers the M-of-N threshold — earlier-completing children's outputs are silently lost. For bisection, where left and right halves write to different keys, the first child's result disappears. ~35 lines to fix both.

## The recursive sub-case problem

Bisection is naturally recursive. A batch case spawns two sub-batch cases (left half, right half). Each sub-batch is the same CasePlanModel — test, bisect further or reject. The recursion terminates when batch size hits 1.

The engine blocks this. `SubCaseExecutionHandler` has an explicit circular guard: if a case tries to spawn a sub-case with the same namespace/name/version, the binding faults. The guard prevents infinite recursion, but it also prevents legitimate recursive decomposition.

The fix is engine#573 — replace the hard block with a bounded depth limit. `maxRecursionDepth: 10` on the SubCase definition handles batches up to 1024 PRs. The actual recursion depth for a batch of 10 is 3-4 levels. The change is ~30 lines and fully backward compatible — `maxRecursionDepth: 0` (the default) preserves the current hard block.

## Every path must terminate

The exhaustive state transition analysis found 15 distinct paths through the CasePlanModel. The first three spec versions had paths that hung:

- A batch of size 1 that fails its tip test — no goal fires, the sub-case never completes, the parent's M-of-N group waits forever. Fix: `single-pr-rejected` success goal.
- Human approves merge after reroute exhaustion — nothing re-fires because `merge-batch` requires `.mergeResult == null`. Fix: `merge-after-escalation` binding with `contextWrite` reset, following the `security-review-reduced-scope` pattern from `pr-review.yaml`.
- CI runners exhaust all reroutes — no binding or goal handles `.tipTest.status == "REROUTES_EXHAUSTED"`. Fix: `tip-test-escalation` humanTask.
- HumanTask SLA expires — writes `BLOCKED`, not `REJECTED`. Every failure goal initially checked only for `REJECTED`. Fix: expand all failure conditions to include `BLOCKED`.

Each of these follows a pattern already established in `pr-review.yaml`. The merge queue CasePlanModel is bigger — 11 bindings, 3 success goals, 3 failure goals — but the patterns are the same.

## The ten architectural differences

Each capability reflects a structural difference between the traditional Bors-style approach and CaseHub's CasePlanModel-based design:

| # | Capability | Traditional (Bors-style) | CaseHub |
|---|-----------|-------------------------|---------|
| 1 | Strategy as data | Compiled code | CasePlanModel YAML, per repo at runtime |
| 2 | Trust-weighted batch composition | FIFO batching | Batch size ∝ min trust; low-trust PRs get smaller batches |
| 3 | Trust-weighted bisection | Positional binary split | Split by trust — isolate likely culprits first |
| 4 | Priority lanes with starvation prevention | FIFO ordering | Composite score with time-decay |
| 5 | Dependency-aware ordering | Author-managed | DAG from labels + git base-branch |
| 6 | SLA-bounded queue wait | Not tracked | WorkItem per queued PR, tiered escalation |
| 7 | Adaptive batch sizing | Static configuration | Batch size adapts to recent failure rate |
| 8 | Cryptographic merge audit | Application logs | MergeDecisionLedgerEntry in Merkle chain |
| 9 | Human oversight for high-risk merges | Uniform merge path | ActionRiskClassifier gates by risk level |
| 10 | Recursive auditable bisection | Internal to the algorithm | Every level in EventLog with causal chain |

The foundation for all ten already exists. Trust scores from Layer 6. SLA tracking from Layer 2. Typed agent messaging from Layer 3. Tamper-evident audit from Layer 4. Adaptive case management from Layer 5. The merge queue is the first feature that composes all six layers into a single capability.

Two foundation gates remain before implementation: engine#573 (recursive sub-case depth limit) and engine#574 (M-of-N YAML schema + per-child outputMapping). Both are small, well-scoped changes to the engine — the runtime already supports the semantics, the gaps are in the YAML schema and one output-mapping behavior fix.
