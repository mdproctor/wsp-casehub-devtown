---
title: "Seven Issues, Three Specs, and the Engine's Actual Dispatch Chain"
date: 2026-06-22
author: Mark Proctor
projects: [casehub-devtown]
tags: [casehub, devtown, github-integration, ci-status, merge-execution, engine-internals]
---

## Seven Issues, Three Specs, and the Engine's Actual Dispatch Chain

The batch started with housekeeping: a Flyway migration that created `erasure_receipt_ledger_entry` while the JPA entity mapped to `devtown_erasure_receipt_ledger_entry`. The name mismatch only surfaced in tests using `LedgerEnabledTestProfile` — which switches from Hibernate `drop-and-create` to Flyway — and produced the singularly unhelpful `Table "DEVTOWN_ERASURE_RECEIPT_LEDGER_ENTRY" not found`. The fix was one line in the SQL. The diagnosis took longer than it should have.

The ErasureReceipt migration to the foundation version was cleaner. The foundation's `LedgerErasureService` already creates tamper-evident receipts when `casehub.ledger.erasure-receipt.enabled=true` — the devtown-specific entity was a parallel implementation that predated the foundation feature. Removing it and enabling the config flag eliminated 163 lines and a Flyway migration. The one subtlety: the foundation receipt stores the raw actor ID (`erasedActorId`), while the devtown version stored the tokenised form (`erasedActorToken`). After tokenisation, the identity mapping is deleted — you can't match tokens back to raw IDs. The compliance query had to shift from per-actor token matching to tenancy-level receipt presence.

## CI status integration: where the specs earned their keep

The CI status spec went through three review rounds. The first draft claimed `revisePr()` already nulled `.ci` — it doesn't; it nulls six analysis keys but not CI. That would have been a merge-on-stale-CI bug in production. The second draft proposed unconditionally pre-seeding `ci = { status: "pending" }` to suppress the YAML `run-ci` binding. Claude's reviewer pushed back: the binding and the webhook are two legitimate CI integration modes — external (GitHub Actions) and dispatched (agent via binding). A `devtown.ci.mode` config property is the right control point. The binding condition `.ci == null` naturally selects the mode based on the initial context. No YAML changes needed.

The SHA guard design had a data-source bug: I'd written "read from the tracker's PrPayload." The tracker stores the original payload at registration time — after a force-push updates `pr.headSha` via signal, the tracker still has the old SHA. The guard must read from the case context via `caseHub.query()`, with the tracker as a cache that `revisePr()` syncs explicitly.

Multi-suite aggregation is the remaining rough edge. A repo with two GitHub Actions workflows produces two `check_suite:completed` events per commit. Without knowing the total count, we can't distinguish "all suites passed" from "one of two passed." The spec uses pessimistic sticky-failure: once any suite signals failure, the status stays failing until the next push resets to pending. A false-passing window exists when two success events race. The full fix requires the GitHub REST API combined-status check — filed as a follow-up.

## Merge execution: the engine doesn't do what you think

The merge execution spec exposed the gap between how I assumed the engine dispatches workers and how it actually works. I proposed registering the merge function as a `@Named("merge-executor")` CDI bean. Claude's reviewer traced the engine bytecode: `CaseContextChangedEventHandler.publishWorkerSchedule()` checks `CaseDefinition.getWorkers()` — a CDI bean sitting in the container is invisible to this chain. Workers must be registered programmatically in the CaseDefinition.

That led to `PrReviewCaseHub` gaining a `@PostConstruct` method that calls `super.getDefinition()` and adds a Worker with `WorkerFunction.Sync`. The hexagonal boundary puts the domain port (`MergeClient`, `MergeOutcome` sealed interface) in `domain/` — pure Java, no engine types — and the `WorkerResult` adaptation in `app/`. The adaptation lambda extracts `pr.repo`, splits it into owner/name, parses `pr.id` to int, and delegates to `MergeClient.merge()`. Input extraction is adapter logic.

The `outputSchema: "{}"` finding was the most embarrassing miss. JQ `{}` constructs an empty object — it doesn't pass through the input. Every key in the worker's output is silently discarded. The merge SHA would have vanished before any handler saw it. Remove `outputSchema` entirely; null means raw output passes through.

The timing issue was deeper. Without the `merge-completed` goal, the case completes before the merge worker finishes: `rules()` schedules the worker (async), then `goals()` evaluates on the same cycle — all review goals are already satisfied, case transitions to COMPLETED, and the worker's output write is rejected because the case is no longer RUNNING. Adding `.merge_sha != null` to a `merge-completed` goal in the completion `allOf` keeps the case open until the merge output lands.

## The trust gate that silently permitted everything

`TrustGateWiringTest.deniesNonBootstrapAgentBelowFloor` had been failing since the ledger was integrated. The symptom: seeds trust score 0.15, expects `permits()` to return false (floor is 0.30), gets true. Two missing CDI activations: `JpaActorTrustScoreRepository` wasn't in `selected-alternatives` (the `@DefaultBean` no-op silently accepted `upsert()` calls and returned empty for all queries), and `ConfigFilePreferenceProvider` from `casehub-platform-config` wasn't Jandex-indexed (so `MockPreferenceProvider` returned empty preferences, floor resolved to 0.0, gate disabled). The GE for the trust score repository already existed in the garden — it just hadn't been applied to devtown's test config.

The qhorus type enforcement tests were a different contract mismatch. `StoredMessageTypePolicy.validate()` only throws `MessageTypeViolationException` for COMMAND and QUERY — all other types get advisory-only treatment. The tests expected strict rejection for EVENT, STATUS, and FAILURE. Switching from `assertThatThrownBy` to `assertThat(result.advisories()).isNotEmpty()` aligned the tests with qhorus's actual contract.

The spec review process on this batch — seven rounds across two features — caught real bugs that would have shipped: stale CI data surviving force-push, merge output silently discarded, premature case completion, and a worker registration mechanism that doesn't exist. The engine's actual dispatch chain, traced through decompiled bytecode, diverged from the API documentation in ways that only surface under test.
