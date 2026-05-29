---
layout: post
title: "Layer 3: making obligation explicit"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [casehub, qhorus, layer-3, speech-act, cdi]
---

Layer 3 isn't about adding more code to the PR review. It's about making the obligation explicit.

Layer 1 (`PrReviewService`) asks specialist agents the same way you'd call a method: directly, synchronously, with no record of the request. The specialist returns findings or not, and that's that. There's no trace of what was asked, no mechanism to know if the answer was "done" or "I don't do that." Layer 3 changes both of those things.

The mechanism is casehub-qhorus. Every specialist review becomes a COMMAND — a speech act that creates a Commitment in the qhorus layer. When the specialist responds, it sends DONE (obligation fulfilled) or DECLINE (scope boundary, formally stated). Both are recorded in the `MessageLedgerEntry` chain. The PR orchestrator doesn't just get a result; it gets a verifiable record of who was asked, what they said, and whether they accepted the obligation or declined it.

## Why one channel per PR, not one per specialist

Before writing any code I had to decide how to structure the channels. The obvious approach: one channel per specialist per PR. Security has its own, architecture has its own, test coverage has its own. Clean separation.

The problem is oversight. If a PR needs a human governance gate — for an architectural decision, say — that gate is at the PR level, not at the specialist level. There is one oversight channel for the PR, not three separate ones. Creating `security-review-123/oversight`, `architecture-review-123/oversight`, and `test-coverage-review-123/oversight` implies three independent governance chains that don't exist in the model.

The right structure is one set of three channels per PR: `pr-review-{n}/work`, `pr-review-{n}/observe`, `pr-review-{n}/oversight`. All specialist COMMAND threads go on the same `/work` channel, differentiated by `correlationId` and `target`. That channel becomes the complete audit record for the review — every assignment and its resolution, in chronological order. Layer 4 will post diagnostic events to `/observe`. Layer 2's oversight gate will use `/oversight`. Creating the full layout in Layer 3 means future layers have somewhere to write without restructuring.

## What the dispatch actually looks like

The qhorus PLATFORM.md deep-dive describes `MessageDispatch.builder(channelId, sender, type, content)` as a positional factory. That's wrong — I found when decompiling the jar that there's no such overload. The builder is a no-arg factory with chained setters:

```java
messageService.dispatch(MessageDispatch.builder()
        .channelId(work.id)
        .sender(ORCHESTRATOR)           // "pr-orchestrator"
        .type(MessageType.COMMAND)
        .content(pr.repo() + "#" + pr.prNumber())
        .correlationId(correlationId)
        .target(agent.capability())     // sets obligor=capability in the Commitment
        .actorType(ActorType.SYSTEM)
        .build());
```

Setting `target=agent.capability()` is what makes the commitment queryable. Without it, `obligor=null` in the Commitment, and `commitmentStore.findOpenByObligor(capability, channelId)` returns empty even when obligations are open. The qhorus deontic semantics: a COMMAND obligates the receiver (target), not the sender.

Two other non-obvious API constraints: `inReplyTo` takes `messageId()` which returns `Long`, not `ledgerEntryId()` which is UUID — the docs don't distinguish them. And `actorType` is required at `build()` time even though it looks optional in the API; omitting it throws `IllegalArgumentException` with no indication from the signature.

## CDI displacement for tutorial layers

Adding Layer 3 alongside the existing Layer 5 (`PrReviewCaseService`) hits a CDI ambiguity: two non-`@DefaultBean @ApplicationScoped` beans implementing the same interface. Quarkus throws at startup.

The displacement pattern for tutorial harnesses is: `@DefaultBean` for Layer 1, `@Alternative @Priority(N)` for Layer N where N is the tutorial layer number. Layer 5 gets `@Priority(2)`, Layer 3 gets `@Priority(1)`. In the full build the highest-priority bean wins — Layer 5 is active. Layer 3 exists on the classpath for tutorial reading but is CDI-inactive at runtime. To test a specific layer independently, inject its concrete type directly: CDI always resolves a concrete-type injection to exactly that bean, regardless of priority ordering.

## DECLINE is not an error

The `ArchitectureReviewAgent` always declines:

```java
public ReviewerOutcome handle(PrPayload pr) {
    return new ReviewerOutcome.Declined("distributed transaction outside scope");
}
```

This is the teaching point. Layer 1 has no way to distinguish "I don't handle this" from "I looked and found nothing" — both return an empty findings list. Layer 3 gives DECLINE first-class status: a typed outcome, a Commitment that transitions to DECLINED, and a `MessageLedgerEntry` that any downstream system can query. The merge queue can see that architecture review was declined with a specific reason, not that it passed silently.

The ledger record of a DECLINE is as useful as the ledger record of a DONE. That's what Layer 3 establishes.
