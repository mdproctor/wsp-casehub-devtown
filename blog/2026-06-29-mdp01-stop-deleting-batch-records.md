---
layout: post
title: "Stop Deleting Your Batch Records"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [merge-queue, adaptive-batch-sizing, webhook]
---

The merge queue shipped last session with a design gap I knew was there: batch records deleted on completion. `ActiveBatchEntity` was ephemeral — once a batch merged or bisected, the record disappeared. The adaptive batch sizing formula was already in `DefaultBatchCompositionPolicy`, waiting for a `recentFailureRate` that never arrived because there was nothing to compute it from.

The fix was obvious once I stopped treating batch records as active-only state. A batch has a lifecycle: dispatched, then completed with an outcome. Deleting it on completion is throwing away the outcome — the one piece of data the system needs to learn from. Two columns (`completed_at`, `succeeded`) and a WHERE clause on `activeBatches()` turned ephemeral state into a durable history. The SPI break — `deleteBatch` replaced by `completeBatch` — forces every caller to record whether the batch succeeded or failed. No caller can silently discard the outcome anymore.

The failure rate computation itself is almost trivial. JPQL query, `setMaxResults(window)`, count the failures, divide. The `FAILURE_RATE_WINDOW` preference key was already there with a default of 20 — it had been waiting since the merge queue spec for something to read it. The formula `adaptiveMax = max(minBatchSize, floor(trustMax × (1 - recentFailureRate)))` was already implemented in `DefaultBatchCompositionPolicy`. Wiring the rate into `formBatchesTransactional` was a one-line replacement of a hardcoded `0.0`.

One decision worth recording: the failure rate counts root batch outcomes only. When bisection triggers, the root batch is marked `succeeded=false`. Sub-cases have no batch records and are not counted. This is intentionally conservative — bisection is a recovery mechanism that costs log₂(batchSize) additional CI cycles. The formula's job is to find the batch size that avoids triggering bisection, not just to avoid unrecoverable failures. The system self-corrects: smaller batches fail less, which raises `adaptiveMax` back toward `maxBatchSize`.

The webhook admission (#101) was the smaller of the two features but had the more interesting architectural question: where does `MergeQueueAdmissionPort` live? The merge queue is a distinct domain concern from PR review — the spec says so, the module structure says so, the CasePlanModel binding says so. Putting the port in `review/` would have been convenient (`github/` already depends on it) but architecturally dishonest. It went into `merge/`, where the merge queue's other abstractions live. The port takes raw identity data — PR number, repo, SHA, author — and the implementation fills in domain defaults (trust 0.5, lane NORMAL). The webhook handler stays thin: check the label, check for draft, call the port.

The `enqueue` return type change from `void` to `boolean` was a side effect of needing `AdmissionResult` (ENQUEUED vs ALREADY_QUEUED) for the webhook response. But it exposed something the design review caught: the MCP tool `enqueuePr` and the CasePlanModel adapter both discard the boolean, always reporting "ENQUEUED" regardless. A tracked issue now, not a crisis — the enqueue is idempotent either way — but an example of how a clean design change surfaces latent imprecision in existing code.

The merge queue now has all three admission paths operational. The CasePlanModel binding for PRs that go through CaseHub review. The MCP tool for operator overrides. And the webhook for teams using GitHub's native review flow who want batch testing and bisection without the full CaseHub PR review case. All three converge on `MergeQueueService.enqueue()` with the same idempotency guarantee.
