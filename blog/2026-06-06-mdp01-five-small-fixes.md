---
layout: post
title: "Five Small Fixes and One Red Herring"
date: 2026-06-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust-routing, gdpr, testing, memory]
---

## Five Small Fixes and One Red Herring

The CaseMemoryStore integration shipped last session. This session was cleanup — five issues ranging from a single-line trust routing fix to a GDPR erasure endpoint. None individually interesting, but three observations fell out of the work that are worth recording.

### The one-line fix that closes a gap

`DevtownTrustRoutingPolicyProvider` had a hardcoded `false` for `bootstrapEscalationRequired` — the engine field that controls whether zero-history agents get escalated to human oversight instead of being assigned sensitive work. The domain model already knew the answer: `RoutingPolicy.fallbackType` is present for security-review, architecture-review, and merge-executor, absent for style-review. One line — `routingPolicy.fallbackType().isPresent()` — and the wiring is complete. The engine's `TrustCandidateClassifier` already enforces the flag.

I'd been thinking of this as blocked on engine P1.3, but engine#415 delivered both the field and the enforcement. The blocker was a phantom.

### The tenancy red herring

`CaseMemoryIntegrationTest` was `@Disabled` with a comment blaming async tenancy — claiming `CurrentPrincipal` was unavailable in `@ObservesAsync` threads. I wrote a minimal reproduction test to isolate the question: fire a CDI async event, have the observer call `store.store()`, check the fact lands. It passed immediately.

`FixedCurrentPrincipal` is `@ApplicationScoped` — a singleton. It doesn't care which thread asks for it. The disabled comment was wrong about the cause. The actual blocker turned out to be `CaseLedgerEventCapture` throwing because `LEDGER_SUBJECT_SEQUENCE` doesn't exist in the H2 test database. A completely different problem, tracked now in its own issue.

The lesson isn't about CDI scoping — it's about disabled test comments aging into folklore. The original author probably hit the `LEDGER_SUBJECT_SEQUENCE` error, saw it involved async threads, and attributed it to the most plausible-sounding cause. Three months later, the comment was accepted as fact without anyone re-running the test.

### InMemoryMemoryStore's question filter

The code-area recall test failed on first attempt. The recaller builds a natural-language question — `"review history for app in repo1"` — and passes it to `MemoryQuery.withQuestion()`. The stored fact had text about security review findings. Empty result set, no error.

`InMemoryMemoryStore.query()` implements the question filter as `text.contains(question)` — a literal full-substring match. The entire question string must appear verbatim inside the stored text. Production adapters use tokenised FTS or vector search. The in-memory adapter silently filters out results that any real backend would return.

The fix was trivial — adjust the test data to include the question string. But the failure mode is worth knowing: a test passing with the JPA adapter and failing with the in-memory adapter, or vice versa, because their question semantics diverge completely.

### The rest

Configurable recall limits landed via three `PreferenceKey<IntPreference>` constants — contributor limit, code area limit, time window. `CaseMemoryRecaller` now resolves them from `PreferenceProvider` using `getOrDefault()`. The defaults match the previous hardcoded values. No behaviour change unless someone configures the YAML.

The GDPR erasure endpoint uses `POST /api/admin/memory/erase/contributor` with a JSON body instead of `DELETE /contributor/{login}` — the login in a URL path would leak to every HTTP access log between the client and the server. For a privacy-rights endpoint, that's exactly the wrong kind of irony.
