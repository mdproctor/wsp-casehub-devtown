---
layout: post
title: "Six Issues, Three Fixes, Two Surprises"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [allowedWriters, dsl-companion, cdi, qhorus]
---

The batch started with six open issues and the optimistic assumption that most were small. Three turned out to need no code at all — the work they described had already been done by earlier sessions. The remaining three each surfaced something that the issue description hadn't anticipated.

## The writer that wasn't there

Setting `allowedWriters` on the Qhorus channels looked straightforward — pass the orchestrator identity as the fifth parameter to `ChannelService.create()`, add a validation guard, done. But the moment the ACL was active, every test that dispatched a DONE or DECLINE message on behalf of an agent started failing. The existing code was using the agent's capability name as the sender: `sender(agent.capability())`. With `allowedWriters = "pr-orchestrator"`, that's a rejected write.

The fix — changing the sender to ORCHESTRATOR for all dispatches — is correct for Layer 3, where the orchestrator simulates everything. But the surprise is that the ACL enforcement surfaced a design inconsistency that had been invisible. In Layer 6, when real agents dispatch their own responses, they'll need to be added to the channel's writer list at assignment time via `channelService.setAllowedWriters()`. The enforcement didn't just close the security gap; it made the Layer 3 → Layer 6 transition explicit in the code.

## The class that was already there

The DSL companion issue asked for a new production class. There was already one in `review/src/test/` — 177 lines, nine capabilities, lambda evaluators, 28 binding condition tests depending on it. Creating a second class would have meant two parallel definitions that could drift.

Promoting it to `review/src/main/` was the obvious move, but the promotion exposed three divergences from the YAML that the test class had carried since it was written. The `human-approval` binding used a capability target instead of `HumanTaskTarget`. The capability schemas were all `"{}"` instead of matching the YAML's specific `inputSchema`/`outputSchema`. And the capability count was nine instead of eight — `human-decision:pr-approval` had been listed as a capability even though it's a human task binding, not a capability.

None of these divergences had caused test failures because `PrReviewBindingConditionTest` only exercises binding conditions, not the structural shape of the definition. The equivalence test — comparing the DSL output against `CaseDefinitionYamlMapper.load()` — is what catches structural drift. Without it, the promoted class would have carried wrong metadata into production.

## The definition that wasn't found

The `CaseMemoryIntegrationTest` diagnosis went deeper than expected. The `@Disabled` annotation said "async case definition registration race." The actual error — `CaseDefinition not found for case` — came from `SchedulerService.registerScheduledTriggers()`, which calls `getCaseDefinition()` and throws on null. The pr-review case has zero scheduled triggers. The method would have returned `voidItem()` if it got past the null check.

The round-trip works from the test thread — `getCaseMetaModel()` returns non-null, `getCaseDefinition()` with that same model returns non-null. But inside the event bus handler on a worker thread, the same call returns null. `ConcurrentHashMap` guarantees visibility. The `CaseKey` is value-based. The `CaseMetaModel` is the same Java object reference (local event bus codec passes by reference). I couldn't find the root cause in this session — it's filed as [engine#444](https://github.com/casehubio/engine/issues/444) for investigation on the engine side.

The second issue underneath — the `CaseMemoryEmitter` not storing facts despite the `ReviewCompletedEvent` being captured — is a CDI async observer delivery question that needs its own investigation.
