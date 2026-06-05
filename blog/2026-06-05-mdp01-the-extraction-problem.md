---
layout: post
title: "The Extraction Problem"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [casehub, memory, architecture, cdi]
---

Every platform capability looks straightforward from the SPI side. `CaseMemoryStore.store(MemoryInput)` takes a typed record and persists a fact. `query(MemoryQuery)` returns a list. Clean, immutable, well-designed. The devtown side — *consuming* that SPI to store reviewer outcomes from PR review cases — is where the complexity hides.

## Where the outcome actually lives

The PR review case runs on the engine's CasePlanModel. When a reviewer binding completes, the engine fires `PlanItemCompletedEvent` with three fields: `caseId`, `planItemId`, and `trackingKey`. None of these is the outcome. The outcome lives in the case context — an untyped `Map<String, Object>` — after the binding's JQ `outputMapping` compressed the result into a key like `securityReview.outcome`.

The first design had the memory emitter observing `PlanItemCompletedEvent` directly, querying the case context, extracting the PR metadata and outcome, and calling `store()`. One component, simple pipeline.

The spec reviewer tore it apart.

## The hard part nobody designs for

The extraction — pulling structured reviewer outcome data from an untyped case context map in an async CDI observer with no request scope — is the actual design challenge. Every other component in the pipeline is mechanical. The emitter takes a typed event and builds `MemoryInput` records. The recaller queries by entity ID and returns a `MemoryContext`. Both are pure functions, trivially unit-testable. The extraction component touches engine internals, handles reactive types (`Uni<CaseInstance>` blocked on a managed executor thread), works around missing tenant context, and parses an untyped map where the key naming is inconsistent across bindings (`securityReview.outcome` vs `humanApproval.status` — a Layer 2 legacy).

Isolating this in a single `ReviewOutcomeObserver` that fires a typed `ReviewCompletedEvent` was the key structural decision. The emitter never sees `CaseContext`, never calls `await()`, never worries about tenant scope. Layer 6 (trust routing), when it arrives, gets a clean typed event to consume — without coupling to the emitter or re-deriving the extraction.

## Two things the engine doesn't tell you

`PlanItemCompletedEvent` only fires for worker completions — not for context signals. If a binding resolves because `caseHub.signal()` updates the case context, the event never fires. The Javadoc says "fires on COMPLETED terminal state" without clarifying which code path produces that state. In production, reviewer bindings complete through the worker path (qhorus DONE → engine), so the observer works. In tests, you have to fire the CDI event directly.

The second gotcha is quieter. `InMemoryMemoryStore.assertTenant()` reads `CurrentPrincipal` to verify the caller's tenant matches the input. In an `@ObservesAsync` thread, there is no request scope. The assertion fails, the emitter's catch-all swallows the exception, and facts are silently never stored. The integration test times out waiting for facts that will never arrive — and the timeout looks like a slow async chain, not a security assertion failure.

Both are now garden entries. Both would have been production bugs.

## The pattern that fell out

The three-entity model — contributor, reviewer, code area — each gets a `MemoryInput` with natural-language text for semantic embedding and structured attributes for exact-match queries. Code area entities use module-level normalization (find `/src/` in the path, everything before it is the module name), not file-level, because two PRs touching different files in `app/` need shared history. Code area text deliberately omits the contributor login — GDPR says `eraseEntity("contributor:mdproctor")` should be sufficient without cross-entity scrubbing.

The `hasRiskSignals()` logic on `MemoryContext` is fail-closed: unknown outcome details default to risk. DECLINED is excluded — it's a routing signal, not a risk signal. Claude caught a bug in the initial spec where a null `outcome-detail` would have silently classified every clean review as safe.

Memory is enrichment, not critical path. The recaller runs before `startCase()`, returns `MemoryContext.EMPTY` on any failure, and the case opens normally. The emitter swallows store failures at WARN. No adapter installed means the no-op `@DefaultBean` handles everything — zero overhead.
