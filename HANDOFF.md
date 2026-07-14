# HANDOFF — 2026-07-14

## Last Session

S/XS batch sweep — 11 commits on `feat/s-xs-batch` covering 15 issues. CBR Phase 2 refinements (per-capability thresholds, similarity-weighted accumulation, activationSource attribution, weight refinement from outcome feedback), evidential violation persistence, merge queue failure rate API with alerting, and 8 new MCP tools. Two issues blocked on foundation gaps (startup hydration needs durable persistence; SLA calibration needs timing data in CBR pipeline). Two closed as no-action-needed/stale (#149, #110). Landed as `127d13e` on main. Gantt v3 generated with vertical phase bands and dependency-traced issue set.

## Immediate Next Step

Start with #124 (PR supersede/relink) — highest standalone value, no dependencies.

## What's Left

- **parent#361** — docs: sync casehub-devtown.md for CBR Phase 1+2 · XS · Low

## Cross-Module

**Blocked by:**
- `blocks-ui` — blocks-ui#41 (gates ALL devtown UI: #98, #119, #120, #123) · L · Med
- `neocortex` + `engine` — timing data in PlanTrace/PlanCbrCase for SLA calibration (#136) · S · Med
- `engine` — durable CaseInstanceRepository + list query API for startup hydration (#127) · M · Med
- `claudony` — worker provisioning for batch branch git ops (#104) · M · Med

## What's Next

**Tier 1 — Unblocked now:**

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #124 | PR supersede/relink — case state transitions + audit trail | M | Med | Independent, highest standalone value |
| #134 | Batch risk scoring from precedent (CBR P3) | S | Med | Extends CBR into merge queue |
| #135 | Bisection heuristics — similarity to past failures (CBR P3) | S | Med | Smarter bisection on batch failure |
| parent#361 | Doc sync casehub-devtown.md for CBR | XS | Low | Trailing obligation |
| #150 | Migrate CaseMemoryEmitter to MemoryEmitter | S | Low | Depends on neocortex#64 |

**Tier 2 — Blocked on foundation (file prereq issues, then wait):**

| # | Description | Scale | Complexity | Blocker |
|---|-------------|-------|------------|---------|
| #104 | Batch branch git ops | M | Med | claudony workers |
| #127 | PrReviewCaseTracker startup hydration | S | Med | Engine query API (no issue filed) |
| #136 | SLA calibration from past assignments | S | Low | Timing data in neocortex + engine |

**Tier 3 — Blocked on blocks-ui#41:**

| # | Description | Scale | Complexity |
|---|-------------|-------|------------|
| #98 | Trust visibility UI | M | Med |
| #119 | CasePlanModel browser | M | Med |
| #120 | Case dependency graph | M | Med |
| #123 | Worker session mgmt UI | S | Med |

**Tier 4 — Epics, long horizon:**

| # | Description | Scale | Complexity |
|---|-------------|-------|------------|
| #12 | Epic 5: Cross-repo coordinated merge | XL | High |
| #16 | Epic 9: Notification wiring | XL | High |
| #81 | Doltgres time-travel (P1.5) | L | High |
| #24 | Contributor trust for OSS | XL | High |

## References

- Blog: `blog/2026-07-14-mdp02-small-issues-structural-gaps.md`
- Gantt v3: `~/Downloads/devtown-roadmap-gantt-v3.pdf`
