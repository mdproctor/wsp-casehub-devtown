---
layout: post
title: "The undo button for cross-repo merges"
date: 2026-07-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cross-repo, coordination, rollback, case-model]
series: issue-158-coordinated-rollback-worker
---

*Part of a series on [#158 — coordinated-rollback worker](https://github.com/casehubio/devtown/issues/158). Previous: [When sub-cases aren't sub-cases](2026-07-17-mdp06-when-subcases-arent-subcases.md).*

Cross-repo coordinated merge shipped yesterday. The happy path works — all repos merge sequentially, the parent case completes. But the merge worker already contained the answer to what happens when one fails: it stops on first failure and returns what it accomplished. The merge results array has N-1 successes and one failure at the end.

The rollback worker is the mirror image. Where merge stops early, rollback must not. If reverting repo A fails with a conflict (someone pushed to the target branch after the original merge), you still need to try repo B. Each successful revert reduces damage. Best-effort, always.

The design decision that matters is where failure handling lives. The worker could detect a failed revert and return `WorkerResult.failed()` — letting the engine FAULT the case. But that kills the case without human intervention. The alternative: the worker always returns Success, with revert outcomes as *data*. A separate YAML binding fires a `humanTask` when any revert has `status != "success"`. The case stays alive, the human gets paged, and the conflict details are in the blackboard for them to act on.

This follows the same pattern the merge worker already established. Merge failures aren't worker-level errors — they're domain outcomes recorded in the data. The YAML goals react to the data. The rollback worker extends this consistently: `rollbackResults` is data, the `rollback-human-escalation` binding is the orchestrator.

The design review caught something I'd missed: the human escalation binding needed a null guard. Without `.rollbackEscalation == null` in the `when` clause, it fires on every context change after the first revert failure — infinitely re-creating human tasks. Every `humanTask` binding in the codebase has this guard. I'd skipped it.

It also caught a semantic mismatch: I'd used `HumanOversight.ROUTING_REVIEW` for the escalation, but routing review is specifically about routing confidence — borderline trust scores, fleet gaps. A failed git revert is infrastructure failure, not a routing decision. `HumanOversight.GENERAL` is the right candidate group.

The idempotency question turned out to be simpler than the issue suggested. The issue asked for EventLog-based deduplication — check whether a rollback already ran before executing. But the YAML condition `.rollbackResults == null` already prevents re-fire. Once the worker writes its output, the binding condition is false. The engine's plan-item lifecycle prevents concurrent execution. Belt and suspenders against a problem the YAML already solves structurally.
