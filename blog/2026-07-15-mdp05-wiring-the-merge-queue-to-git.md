# CaseHub DevTown — Wiring the Merge Queue to Git

**Date:** 2026-07-15
**Type:** phase-update

---

## What I was trying to achieve: making batch-ci-runner real

The merge queue CasePlanModel has been working for months — batching PRs, bisecting failures, routing through trust scores. But the `batch-ci-runner` capability had no production worker behind it. Every integration test registered a mock inline: a lambda that returned `{ status: "passing" }` and moved on. The actual git operations — creating a temporary branch, merging PR SHAs into it, pushing for CI — didn't exist.

Issue #104 had been sitting in the backlog marked "blocked on claudony workers." I traced through the spec and the existing code and concluded the blocker was wrong. The merge-executor worker already proved the pattern: a `Worker.builder().function(...)` wrapping a hexagonal port interface, registered in `augment()`. No AI agent needed. The git operations are mechanical — create a ref, merge N commits, delete the ref when done.

## What we believed going in: the spec was complete

The merge queue design spec (§7.3) described six steps. I assumed those mapped cleanly onto the implementation. They didn't — not because the spec was wrong, but because it described the full lifecycle as one block. The git operations actually split across two distinct actors at two distinct lifecycle points: branch creation during the `test-batch-tip` binding, and branch deletion when the case reaches a terminal state.

That separation matters. The worker creates the branch and returns. CI runs externally. A webhook signals the result back. The cleanup observer watches for `CaseLifecycleEvent` and deletes the branch. Two separate components, two separate responsibilities, connected only by the branch naming convention.

## The design review caught what I missed

Claude ran a four-round adversarial review against the spec. Three findings changed the design:

The idempotency bug was the sharpest catch. When `batch-ci-runner` returns `WorkerResult.failed()` on a merge conflict, the engine's `outcomePolicy` reroutes to another worker. But the stale branch from the first attempt still exists. The rerouted attempt calls `createRef` and gets HTTP 422 — "Reference already exists." Every reroute fails for the wrong reason, masking the real conflict as an infrastructure error. The fix: delete-before-create. Each attempt starts clean.

The `BatchSlice` repository gap was more subtle. Bisection sub-cases create new `merge-batch` cases with a subset of PRs. The sub-case context comes from `sliceToMap()`, which converts `BatchSlice` records. But `BatchSlice` had no `repository` field — it was never needed before because no worker had to call a GitHub API from the bisection path. Adding `repository` to the record, propagating it through all four split strategy implementations, and fixing the `bisection-splitter` inputSchema to pass `batch: .batch` (not just `prs` and `strategy`) was a cross-cutting change across `queue/`, `merge/`, and `app/`.

The third finding — that `CaseTrackingStatus.fromCaseStatus("SUPERSEDED")` doesn't map to SUPERSEDED — turned out to be a false positive. SUPERSEDED is a devtown-internal state, not an engine status. The existing test explicitly asserts this. But the investigation was useful: it forced me to trace exactly which statuses the engine fires in `CaseLifecycleEvent` and confirm the cleanup observer's filter logic is correct.

## What the implementation looks like

The port interface is two methods: `createBatchBranch()` and `deleteBatchBranch()`. Both return sealed types — `Created | MergeConflict | Failure` and `Deleted | NotFound | Failure`. The GitHub adapter uses three REST endpoints from the Git Data API: `POST /git/refs`, `POST /merges`, `DELETE /git/refs`. All remote — no local git clone needed.

The worker adapter in `MergeBatchCaseHub.augment()` extracts batch metadata from the case context, builds a `List<PrRef>`, calls the port, and maps the sealed outcome to `WorkerResult`. Input validation catches missing or malformed `repository` before the API call.

The cleanup observer watches `CaseLifecycleEvent`, filters on `namespace == "devtown"` and `caseDefinitionName == "merge-batch"`, and fires for every terminal state. Each bisection sub-case creates its own branch and gets its own cleanup — the branch naming convention (`merge-queue/batch-{id}`) keeps them distinct.

## What this unblocks

The merge queue now has a complete execution path from batch formation through CI testing to cleanup. What's missing: the webhook handler that signals CI results back to the case (connector concern), and the merge-executor variant for batch merging (fast-forwarding the target branch after all PRs in the batch pass). Both are separate issues.
