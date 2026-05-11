# Handoff — devtown Epic 2 complete

2026-05-11

## What shipped this session

**Epic 1 (scaffold)** — pushed to origin. Five-module Quarkus Maven project (`devtown-domain`, `devtown-review`, `devtown-queue`, `devtown-github`, `devtown-app`). Quarkus boots, CI/publish workflows wired.

**Epic 2 (domain model)** — implemented, tested, pushed, issue #9 closed.
- `ReviewDomain`, `AgentQualification`, `HumanDecision`, `HumanOversight` — 4 typed constant classes replacing the original 13-tag flat namespace
- `DevtownTrustDimension` — 3 dimensions: `review-thoroughness`, `false-positive-rate`, `scope-calibration`
- `RoutingPolicy` record — threshold, minimumObservations, borderlineMargin, fallbackType, rationale; `isBootstrap(int)` and `isBorderline(double)` implement the trust maturity model
- `CapabilityRegistry` SPI — `domain/spi/`, pure Java
- `DevtownCapabilityRegistry` — populated default (10 capabilities, 4 routing policies); `isKnown()` delegates through `capabilities()` for correct subclass behaviour
- `CapabilityRegistryBean` — `@ApplicationScoped` CDI wrapper in `devtown-app`
- 65 tests passing (domain unit + Quarkus CDI integration)

## Active follow-ups

| Issue | What | Priority |
|-------|------|----------|
| devtown#21 | `isKnown()` as default method on `CapabilityRegistry` SPI | Before Epic 3 routing consumers wire against SPI |
| devtown#22 | Bump assertj-core from 3.24.2 to latest | Before Epic 3 |
| devtown#19 | Dependency tracker for ledger#76 (CAPABILITY_DIMENSION composite trust scores) | Passive — unblocks Phase 3 routing |
| devtown#20 | CaseOperation naming/placement review (BATCH_BISECT etc.) | Epics 4/5 |
| parent#14 | Trust maturity model → PLATFORM.md | Parent session |

## Living documents updated

- `docs/PROGRESS.md` — DT-001–DT-006 all Implemented; foundation gate table current
- `docs/gastown-casehub-analysis-v2.md` — section 12 summary updated
- `docs/PROGRESS.md` — ledger#76 (CAPABILITY_DIMENSION) foundation gate added
- CLAUDE.md (project) — vocabulary, trust dimensions, routing policies updated to implemented design
- PLATFORM.md (parent, local) — SPI default pattern refined (operational no-op vs vocabulary populated default); commit pending in parent session via parent#12

## Next: Epic 3

**PR Review CasePlanModel** (devtown#10) — content-driven routing and parallel checks. Requires:
- The domain vocabulary from Epic 2 ✅
- Foundation: P0.1 ✅, P0.2 ✅ — bindings and trust scoring wired
- Foundation gap: P1.3 ⚠️ (TrustWeightedSelectionStrategy) — routing thresholds designed but not yet enforced at assignment
- Foundation gap: HITL wiring ⚠️ — HumanDecision lifecycle not end-to-end yet

Before implementing: resolve devtown#21 and devtown#22. Run Platform Coherence Protocol against local parent docs.

## Key reference docs

- Spec: `specs/2026-05-08-epic2-domain-model-design.md`
- Plan (used): `plans/2026-05-09-epic2-domain-model.md`
- Foundation gates + improvements: `docs/PROGRESS.md` in project repo
- Gastown comparison: `docs/gastown-casehub-analysis-v2.md` (section 12 = improvement log summary)
- Blog: `blog/2026-05-11-mdp01-the-vocabulary-problem.md`
