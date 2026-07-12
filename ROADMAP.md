# devtown Cross-Repo Roadmap

**Purpose:** Phased delivery plan mapping devtown features to foundation prerequisites.
Each phase delivers demonstrable capability — backend + UI together, not siloed.
Foundation repo sessions: check this to know what devtown needs from you.

**Last updated:** 2026-07-12

---

## Phase 1 — Trust Intelligence

**Demo story:** PR arrives → system routes to reviewers based on trust + CBR precedent → agent claims DONE but did nothing → evidential checker catches it → trust degrades → next PR routes to a different, proven agent. You can see all of this in the trust UI.

### Devtown

| # | Description | Scale | Complexity | Type |
|---|---|---|---|---|
| #141 | EvidentialChecker V1-V4 integration | M | Med | Backend |
| #132 | CBR-enhanced capability activation | M | Med | Backend |
| #133 | CBR-enhanced reviewer matching | M | Med | Backend |
| #98 | Trust visibility UI — scores, routing history, incidents | M | Med | UI |
| #144 | Fix TrustFeedbackClosedLoopTest (engine SNAPSHOT drift) | XS | Low | Fix |

### Foundation prerequisites

| # | Repo | Description | Blocks |
|---|---|---|---|
| engine#711 | engine | TrustPhase enum + evidentialCheckPhases | devtown#141 |
| ledger#175 | ledger | `@Transactional(REQUIRES_NEW)` for saveAttestation() | Attestation transaction safety for #141 |
| parent#361 | parent | Sync casehub-devtown.md for CBR Phase 1 | Docs only |

### Phase 1 critical path

```
engine#711 ──→ devtown#141 (EvidentialChecker)
ledger#175 ─↗                                 ↘
                                                devtown#98 (Trust UI)
devtown#132 (CBR activation) ─────────────────↗
devtown#133 (CBR matching)   ─────────────────↗
devtown#144 (test fix) — independent, do first
parent#361 (doc sync) — independent
```

---

## Phase 2 — Operational Merge Queue

**Demo story:** PRs enqueue → batch forms based on risk scoring from past similar merges → CI runs on batch tip → failure → bisection guided by similarity to past failures identifies the faulty PR → rejected → remainder merges. You can browse the case plan model driving this and see the dependency graph.

### Devtown

| # | Description | Scale | Complexity | Type |
|---|---|---|---|---|
| #104 | Batch branch management — git operations | M | Med | Backend |
| #124 | PR supersede/relink — case state transitions | M | Med | Backend |
| #127 | PrReviewCaseTracker startup hydration | M | Med | Backend |
| #134 | Merge queue batch risk scoring from CBR | S | Med | Backend |
| #135 | Bisection heuristics from CBR similarity | S | Med | Backend |
| #119 | CasePlanModel browser | M | Med | UI |
| #120 | Case dependency graph — D3 visualization | M | Med | UI |

### Foundation prerequisites

| # | Repo | Description | Blocks |
|---|---|---|---|
| engine#548 | engine | Composed GoalExpression — nested anyOf(allOf(...)) | Complex PR review goals |
| blocks#32 | blocks | Plan composition matching for CBR-based routing | devtown#134, #135 |
| blocks#34 | blocks | Signal enrichment SPI for non-LLM routing | Routing signal pipeline |
| blocks#37 | blocks | Feature weight support for CbrQuery routing | CBR weight tuning |
| ledger#172 | ledger | OutcomeRecord supplementary data — routing rationale in audit | Trust UI enrichment |
| work#288 | work | Queue summary REST endpoint for blocks-ui dashboard | Merge queue UI cards |

### Phase 2 critical path

```
engine#548 (composed goals) ─→ devtown#104 (batch git ops) ─→ devtown#134 (CBR risk scoring)
blocks#32, #34, #37 (CBR routing signals) ─────────────────↗       ↓
                                                            devtown#135 (bisection heuristics)
ledger#172 (routing rationale) ─→ devtown#119 (case browser)
work#288 (queue summary) ───────↗ devtown#120 (dependency graph)
devtown#124 (supersede/relink)
devtown#127 (startup hydration)
```

---

## Phase 3 — Agent Coordination

**Demo story:** Browse the full agent fleet — see who's working on what, what messages they've received, search past decisions to understand why a review went wrong. SLA adapts based on how long similar past reviews took. RBAC controls who can see what.

### Devtown

| # | Description | Scale | Complexity | Type |
|---|---|---|---|---|
| #136 | SLA calibration from similar past assignments | S | Low | Backend |
| #138 | Similarity weight refinement from feedback | S | Med | Backend |
| #114 | Default trust score for webhook-admitted PRs | S | Low | Backend |
| #91 | RBAC role topology — engineer, auditor, data-controller | M | Med | Backend |
| #121 | Case memory browser — searchable prior decisions | S | Med | UI |
| #122 | Agent channel message inbox | S | Med | UI |
| #123 | Worker session management — spawn, stop, restart | M | Med | UI |
| #137 | CBR memory browser — similarity search | S | Low | UI |

### Foundation prerequisites

| # | Repo | Description | Blocks |
|---|---|---|---|
| engine#510 | engine | Case-level SLA — time-triggered binding | Overall PR review deadline |
| engine#327 | engine | HumanTaskTarget runtime-evaluated expiresIn | Per-case SLA variation |
| claudony#85 | claudony | Agent onboarding template from CaseDefinition | Worker session lifecycle |
| worker#3 | worker | Worker execution model — async, timeout, context | Agent coordination foundation |
| worker#10 | worker | Engine SyncAgentWorkerFunctionHandler delegation | Worker API integration |
| platform#134 | platform | Shared embedding similarity utility | CBR weight refinement |

### Phase 3 critical path

```
engine#510 (case SLA) ──→ devtown#136 (SLA calibration)
engine#327 (runtime expiresIn) ─↗

worker#3 (exec model) ──→ claudony#85 (agent onboarding) ──→ devtown#123 (worker management)
worker#10 (handler delegation) ─↗
platform#134 (embedding similarity) ──→ devtown#138 (weight refinement)

devtown#91 (RBAC) ──→ devtown#122 (message inbox)
devtown#114, #137, #121 — independent
```

---

## Phase 4 — Resilience and Recovery

**Demo story:** Reviewer agent fails mid-review → system detects failure → spawns replacement with full context from prior reasoning (time-travel via Doltgres) → dashboard shows failure trends → alerts fire on sustained degradation.

### Devtown

| # | Description | Scale | Complexity | Type |
|---|---|---|---|---|
| #81 | Full gt seance — Doltgres time-travel reasoning access | L | High | Backend |
| #108 | Dashboard batch failure rate trends | S | Low | UI |
| #109 | Alerting on sustained high batch failure rates | S | Med | Backend |

### Foundation prerequisites

| # | Repo | Description | Blocks |
|---|---|---|---|
| engine#501 | engine | Semantic failure routing — DECLINED/FAILED handling | Failure cascade recovery |
| engine#571 | engine | Enrich CaseLifecycleEvent with context snapshot | Richer failure signals |
| P1.5 | engine | Doltgres backend | devtown#81 (gt seance) |

---

## Cross-Cutting: UI Pipeline

Devtown UI features in every phase depend on the blocks-ui component pipeline. This is the shared dependency chain:

```
pages#111 (blocks-ui hosting foundation)
pages#129 (data-table component)           ──→ blocks-ui components mature
pages#139 (modal/dialog)                   ──→ blocks-ui#41 (devtown governance workbench migration)
pages#138 (action button)                  ──→ devtown UI features in Phases 1-3
pages#154 (wire pages-table into runtime)
```

### blocks-ui — devtown-specific

| # | Repo | Description | Blocks |
|---|---|---|---|
| blocks-ui#41 | blocks-ui | Devtown governance workbench — migration plan | All devtown UI (#98, #119, #120, etc.) |
| blocks-ui#35 | blocks-ui | Cross-repo component migration tracking | Visibility into pipeline |
| blocks-ui#47 | blocks-ui | Audit-trail-viewer row expand/collapse fix | Trust visibility (#98) |
| blocks-ui#49 | blocks-ui | Lit components consume TypedDataSet natively | Data-driven UI |
| blocks-ui#50 | blocks-ui | Recover PagesTable tests for TypedDataSet | Test coverage |

### pages — infrastructure

| # | Repo | Description | Blocks |
|---|---|---|---|
| pages#111 | pages | Foundation for blocks-ui component hosting | blocks-ui components |
| pages#129 | pages | pages-data-table component | blocks-ui tables |
| pages#154 | pages | Wire pages-table into runtime | blocks-ui migration |
| pages#139 | pages | Modal/dialog component | blocks-ui overlays |
| pages#138 | pages | Action button component | blocks-ui interactions |

### connectors — agent observation

| # | Repo | Description | Blocks |
|---|---|---|---|
| connectors#69 | connectors | Embed qhorus workbench for live agent observation | Agent UI in Phase 3 |

**Priority signal for pages/blocks-ui sessions:** devtown's UI phases depend on blocks-ui#41 landing. The pages infrastructure issues above are the critical path to making that possible.

---

## Deferred — No Active Plan

### Epics (need design work before phasing)

| # | Repo | Description |
|---|---|---|
| devtown#12 | devtown | Cross-repo coordinated merge |
| devtown#16 | devtown | Notification wiring — casehub-connectors |
| devtown#1 | devtown | App-level trust capabilities |
| devtown#24 | devtown | Contributor trust for open source |
| parent#227 | parent | CBR as platform capability |
| parent#234 | parent | Reactive case container |
| parent#294 | parent | Reusable platform primitives |
| parent#310 | parent | casehub-blocks |

### Infrastructure (coordinate carefully)

| # | Repo | Description | Notes |
|---|---|---|---|
| engine#635 | engine | Rename io.casehub.api → io.casehub.engine.api | Breaks all consumers — coordinate timing |
| parent#340 | parent | Audit: remove unnecessary CDI | 175 classes across 27 repos |
| parent#140 | parent | Multi-tenancy state audit | Cross-cutting |
| parent#359 | parent | LLM documentation infrastructure | Tooling |

### Follow-ups (from completed work, low urgency)

| # | Repo | Description | From |
|---|---|---|---|
| qhorus#342 | qhorus | CommitmentContext enrichment for V1/V4 | devtown#141 |
| devtown#145 | devtown | Persist BenchmarkViolation in attestations | devtown#141 |
| devtown#110 | devtown | Update parent merge queue spec §3.1 | Merge queue work |
| devtown#95 | devtown | CI repository_dispatch trigger | Infrastructure |

---

## Foundation Priority Summary

**For engine sessions — what devtown needs, in order:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P0 (now) | engine#711 | devtown#141 — current branch |
| P1 (Phase 2) | engine#548 | Complex PR review case goals |
| P2 (Phase 3) | engine#510, engine#327 | Case-level SLA, per-case expiry |
| P3 (Phase 4) | engine#501, engine#571 | Failure routing, lifecycle events |
| Defer | engine#635 | Namespace rename — schedule when quiet |

**For ledger sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P0 (Phase 1) | ledger#175 | Attestation transaction safety for #141 |
| P1 (Phase 2) | ledger#172 | Routing rationale in audit trail → trust UI |
| Defer | ledger#137 | Artifact trust scoring |

**For qhorus sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P2 (after #141) | qhorus#342 | V1/V4 evidential checks |

**For blocks sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P1 (Phase 2) | blocks#32, #34, #37 | CBR routing signal pipeline |

**For blocks-ui sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P0 (all phases) | blocks-ui#41 | Devtown governance workbench — gates all devtown UI |
| P1 | blocks-ui#47 | Audit-trail-viewer fix |
| P1 | blocks-ui#49, #50 | TypedDataSet integration |

**For pages sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P0 (gates blocks-ui) | pages#111 | Foundation for blocks-ui hosting |
| P1 | pages#129, #154 | Data table components |
| P1 | pages#138, #139 | Action button, modal/dialog |

**For worker sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P2 (Phase 3) | worker#3, worker#10 | Worker execution model, handler delegation |

**For claudony sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P2 (Phase 3) | claudony#85 | Agent onboarding from CaseDefinition |

**For platform sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P2 (Phase 3) | platform#134 | Shared embedding similarity for CBR |
| Defer | platform#147, #146 | Notification system (gates devtown#16) |

**For connectors sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P2 (Phase 3) | connectors#69 | Qhorus workbench for agent observation |

**For parent sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P1 | parent#361 | Doc sync (quick, trailing) |
| Defer | parent#227, #234, #294, #310 | Epic-level — needs design first |
