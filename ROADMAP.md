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

### Docs

| # | Repo | Description |
|---|---|---|
| parent#361 | parent | Sync casehub-devtown.md for CBR Phase 1 |

### Phase 1 critical path

```
engine#711 → devtown#141 (EvidentialChecker)
                                              ↘
devtown#132 (CBR activation) ──────────────────→ devtown#98 (Trust UI)
devtown#133 (CBR matching)   ──────────────────↗
devtown#144 (test fix) — independent, do first
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

### Phase 2 critical path

```
engine#548 (composed goals) ─→ devtown#104 (batch git ops) ─→ devtown#134 (CBR risk scoring)
                                                             ↘
devtown#124 (supersede/relink) ──────────────────────────────→ devtown#119 (case browser)
devtown#127 (startup hydration)                                devtown#120 (dependency graph)
devtown#135 (bisection heuristics) — independent
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

### Phase 3 critical path

```
engine#510 (case SLA) ──→ devtown#136 (SLA calibration)
engine#327 (runtime expiresIn) ─↗

devtown#91 (RBAC) ──→ devtown#122 (message inbox)
                   ──→ devtown#123 (worker management)
devtown#138, #114, #137, #121 — independent
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

**For qhorus sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P2 (after #141) | qhorus#342 | V1/V4 evidential checks |

**For parent sessions:**

| Priority | Issue | What it unblocks |
|---|---|---|
| P1 | parent#361 | Doc sync (quick, trailing) |
| Defer | parent#227, #234, #294, #310 | Epic-level — needs design first |
