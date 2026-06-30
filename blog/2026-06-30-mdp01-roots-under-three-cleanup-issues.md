---
layout: post
title: "The Roots Under Three Cleanup Issues"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [merge-queue, uuid, mcp-tools, test-isolation]
---

I started this branch expecting mechanical fixes — stale names, misused UUIDs, unconditional log statements. The kind of work that fills a morning and leaves the commit history slightly tidier. What I found instead was a structural bug, a design inconsistency, and a missing metrics story.

The three issues were filed during code review of the merge queue work. #106 was "minor review findings." #116 was about surfacing a boolean return value. #102 was MCP tool enhancements. All small. All catalogued. None interesting on the surface.

**The UUID v3 that wasn't v5.** `MergeDecisionObserver` used `UUID.nameUUIDFromBytes()` — Java's built-in, which produces UUID v3 (MD5). The comment said v5. Java has no UUID v5 in the standard library. The fix was straightforward: implement RFC 4122 v5 (SHA-1 + namespace UUID) as a utility in `devtown-domain`. Fifteen lines. The RFC test vector for `python.org` against the DNS namespace confirms it — `886313e1-3b8a-5372-9b90-0c9aee199e5d`. Existing ledger entries keep their v3 UUIDs; new entries get v5. No migration needed.

**The dequeue that never worked.** This is where the "minor finding" turned structural. The flaky `MergeQueueSlaWorkItemTest.dequeue_obsoletesWorkItem` was reported as a test isolation issue. I traced the execution path and found it wasn't flaky at all — it was deterministically broken.

`MIN_BATCH_SIZE` defaults to 1. When `enqueue()` is called, it synchronously triggers `formAndDispatchBatches()`, which calls `formBatchesTransactional()`. With a single entry and minBatchSize=1, the composition policy batches it immediately. The entry transitions from QUEUED to IN_BATCH before `enqueue()` even returns. Then `dequeue()` calls `store.queued()` — which only returns QUEUED entries. The entry is IN_BATCH. Target is null. `obsoleteFromSystem()` is never called. The WorkItem stays PENDING.

The test passed "when run alone" because `dispatchBatch()` sometimes threw (no worker registered for `batch-ci-runner` in the test environment), triggering the compensating action that returned the entry to QUEUED. When the full suite warmed up the engine, dispatch succeeded more often, and the test failed more often.

The fix changed `MergeQueueStore.dequeue()` from returning `boolean` to `Optional<QueueEntry>`. The JPA implementation already has the entity in hand when it checks the status — returning it instead of discarding it eliminates the read-then-write split entirely. The service gets the WorkItem ID from the same transaction that performs the dequeue. And `MIN_BATCH_SIZE=100` in the test config prevents `enqueue()` from triggering batch formation in tests that aren't testing batch formation.

**The SLA that didn't match itself.** `list_problems` used a flat `devtown.queue.sla-minutes` ConfigProperty (120 minutes) for SLA breach detection. But `enqueue()` uses per-lane SLA from `PreferenceProvider` — PT1H for CRITICAL, PT4H for HIGH, PT8H for NORMAL. A CRITICAL PR with a 60-minute SLA wouldn't be flagged until 120 minutes. HIGH PRs would be flagged 120 minutes early. The flat config was a parallel path that contradicted the preference system. I removed it and added `detectSlaBreaches()` to `MergeQueueService`, which resolves the correct lane-specific SLA from the same preference system that `enqueue()` uses.

**The metrics that existed but didn't.** `get_merge_queue_metrics` was already in the MCP tool — but it returned queue depth, active batches, oldest wait time, and average trust score. The spec said throughput, failure rate, average wait time, and batch size distribution. Four fields missing. `MergeQueueStore` gained `completedBatchesSince(Instant)` and an aggregate `recentBatchFailureRate(int)` (the per-repository overload already existed). The metrics tool now computes all four from completed batch history.

The design review caught a real issue I'd missed: I'd framed the dequeue problem as a "TOCTOU race" in the spec, but the evidence chain described deterministic state sequencing, not a race. It also moved SLA breach detection from the MCP tool to the service — the MCP tool shouldn't own business logic. Both were the kind of correction that only an independent review surfaces.

Six commits. Three issues closed. One pre-existing dependency issue surfaced (`Worker.Builder` API mismatch) and one pre-existing bug filed (#118 — orphaned WorkItem on duplicate enqueue). The branch is small but the dequeue fix is the one I'd want a future reader to pay attention to — the `Optional<QueueEntry>` return type is the right interface, and understanding why requires tracing the batch formation path from `MIN_BATCH_SIZE` defaults through composition policy to the store.
