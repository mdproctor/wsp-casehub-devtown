---
layout: post
title: "Domain Events Are Not Messages"
date: 2026-06-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [hexagonal, webhook, github, casehub-engine]
---

The webhook receiver for devtown started with a deceptively simple question: should GitHub PR events go through casehub-connectors?

The connector library has `WebhookInboundConnector` — an abstract base with `handle(WebhookRequest) → WebhookResult` that normalises inbound webhooks into `InboundMessage`. Slack messages, Teams messages, IMAP emails all flow through it. At first glance, a GitHub webhook is just another inbound HTTP payload.

But PR lifecycle events aren't messages. A PR being opened, updated with new commits, or merged is a domain event that directly drives case lifecycle. Routing it through `InboundConnectorService.receive()` → async `Event<InboundMessage>` → an observer that reconstructs the domain intent is three hops through foundation messaging infrastructure for what should be a direct domain operation. Even if `InboundMessage` had structured payload fields instead of a flat `content` string, the architectural cost would be the same — you're pushing application-tier domain logic through foundation messaging plumbing.

The receiver ended up as a direct JAX-RS endpoint in the `github/` module — a hexagonal adapter that translates GitHub's webhook JSON into `PrPayload` and calls `PrReviewApplicationService`. Clean boundary, no intermediary.

### The Map That Wouldn't Mutate

The interesting discovery was in the engine, not the webhook code. When a PR gets new commits pushed (GitHub's `synchronize` event), the running case needs its context updated — new `headSha`, new `linesChanged`, then null out stale analysis fields so bindings re-fire.

The engine's `signal(caseId, "pr.headSha", newSha)` navigates into the case context's `pr` sub-map and calls `put("headSha", newSha)`. Straightforward — except the initial context was constructed with `Map.of(...)`, which returns an unmodifiable map. `WritablePanelImpl` shallow-copies the top level but not sub-maps, so the `pr` key still points to the original `Map.of()` instance. `UnsupportedOperationException` at runtime, with no compile-time warning.

The fix is one line — `new LinkedHashMap<>(Map.of(...))` — but the root cause is in the engine's constructor. Every harness that passes `Map.of()` for a nested context value will hit the same wall the first time they try a sub-path signal.

### Order Matters

Signal ordering turned out to be a correctness requirement, not a style preference. The `initial-analysis` binding fires when `.codeAnalysis == null` and reads `pr.headSha` as its input. If you null `codeAnalysis` before updating `headSha`, the binding fires with the old SHA — the code-analysis worker starts analysing stale code. Metadata first, then invalidation. The Vert.x event bus processes signals sequentially per call, so code order equals execution order, but the ordering is a contract that a future refactor could silently break.

### The Goal That Doesn't Compose

The other engine discovery: `GoalExpression` in the schema supports flat `allOf` and `anyOf` lists of goal names, but not nested composition. I needed "success if (all reviews pass AND CI passes) OR (PR was merged externally)" — `anyOf(allOf(...), goal)` — but the schema can't express it. The pragmatic fix was prepending `.pr.status == "merged" or` to each success goal's JQ condition. When a PR is merged externally, all goals short-circuit to satisfied.

It works, but it pollutes individual goal semantics — `pr-approved` reports as satisfied even when no formal approvals were received. The right design is a separate `externally-merged` goal with engine support for composed completion expressions.

The distinction between "works" and "right" is the whole point of this platform. The workaround ships; the engine issue tracks the proper fix.