---
layout: post
title: "Layer 5: the case definition lands"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: log
projects: [casehub-devtown]
tags: [casehub, yaml, caseplanmodel, quarkus, tdd]
---

The PR review CasePlanModel shipped today. Nine bindings, three goals, content-driven routing that actually routes on what the code contains rather than what the author declared.

The most significant thing we built isn't the YAML — it's what the YAML represents. Security review fires only when code analysis finds security-sensitive code. Architecture review fires only when analysis finds cross-API changes. Style, test coverage, and performance analysis fire simultaneously once analysis completes. The merge binding fires when all conditions are met, in whatever order they happen to arrive. Gastown's Refinery expresses this as a workflow with explicit parallel declarations. We just listed the conditions. The engine evaluates them all on every context change.

Before any of this could land, I wanted to understand the engine's case definition architecture properly. Turns out it's inherited directly from CNCF Serverless Workflow 1.0 via quarkus-flow — a three-layer structure: YAML on disk, a generated schema model for Jackson deserialization, and a canonical API model (`CaseDefinition`) that the engine actually runs against. The fluent DSL builders target the same canonical model but can use `LambdaExpressionEvaluator` — Java predicates that can't be serialized to YAML. Everything expressible in YAML is expressible in the fluent DSL; the reverse isn't true. This distinction matters for testing: we write binding condition tests against the fluent DSL factory, no YAML parsing required, no Quarkus needed.

That test strategy — 28 unit tests against a `PrReviewCaseDefinition` factory using `LambdaExpressionEvaluator` conditions, plus a single `@QuarkusTest` to verify the YAML parses correctly — held up well. Except for one thing.

The final code review flagged a Critical. The `pr-approved` goal required `.securityReview.outcome == "APPROVED"` unconditionally. For a PR where `securitySensitive == false`, the security-review binding correctly never fires, so `securityReview` is never written to context. JQ evaluates `null == "APPROVED"` as false. The goal is permanently unsatisfied. The case never completes.

We'd fixed the merge binding for the same problem in an earlier commit. Claude caught it in the goal condition. The TDD didn't — because we'd written tests for binding conditions but not goal conditions. The binding conditions test whether the right work gets dispatched. The goal conditions test whether the case can complete. They're different things. After the fix we added ten goal condition tests to make sure it doesn't regress.

The CDI wiring produced one non-obvious surprise. Adding `casehub-engine` runtime to the app module pulled in `casehub-ledger` transitively, which requires a `ReactiveLedgerEntryRepository` implementation. `casehub-engine-testing` provides in-memory stubs for engine repositories but not ledger repositories. The `@QuarkusTest` failed with `ClassSelector resolution failed` — nothing in that error points to CDI or the missing ledger SPI. Claude figured it out and wrote the `InMemoryLedgerEntryRepository` stub. It's 60 lines of `Uni.createFrom().item(...)`, but it's in the test source tree, not production, so it's just infrastructure.

The `@DefaultBean` displacement pattern held up exactly as designed. `NaivePrReviewService` — the Layer 1 teaching baseline — stays in the build but becomes inactive the moment `PrReviewCaseService` is on the classpath. No deletion, no configuration change, just CDI's normal priority resolution. For the tutorial sequence, this means Layer 1 is still visible and readable at Layer 5. The gap comments in the naive service explain precisely what each layer above it closes.

One protocol gap we spotted and fixed: the engine's `CaseDefinitionYamlMapper` hardcodes `new JQExpressionEvaluator(string)` at every expression conversion site. That should delegate to a pluggable `ExpressionEvaluatorFactory` — the same architecture SW 1.0 uses with `expressionLang`. Filed against the engine. The YAML already declares `expressionLang: jq`; when the engine reads it, there'll be somewhere to route.
