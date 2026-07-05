# HANDOFF — 2026-07-06

## Last Session

Closed housekeeping batch (#94, #93, #118, #125, #140) + SNAPSHOT drift fixes (#139). Fixed orphaned WorkItem on duplicate enqueue, migrated PrReviewCaseService to PreferenceProvider, cleaned up CurrentPrincipal/qhorus exclude-types. Migrated 49 files across qhorus repackage (runtime→api, records), engine de-reactive, ledger API migration. Created Epic 11 (#129) for Case-Based Reasoning with 9 child issues. Squashed 19→1, pushed to origin/main.

## Immediate Next Step

Start #97 (TrustGatedAttestationPolicy) — unblocked, trust pipeline is end-to-end. Run `/work` to begin.

## What's Left

- **devtown#139** — SNAPSHOT drift test CDI wiring (production compiles, tests blocked on engine ReactiveCaseInstanceRepository + review module MapCaseContext.layer()) · S · Med
- **devtown#127** — PrReviewCaseTracker startup hydration (depends on engine query API) · S · Med
- **devtown#128** — cursor-based pagination for governance REST endpoints · S · Low
- **devtown#124** — supersede/relink backend (scoped out of #85) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #97 | TrustGatedAttestationPolicy — capability-scoped evidential verification | M | Med | Unblocked |
| #129 | Epic 11: Case-Based Reasoning (9 child issues) | XL | High | Phase 1: #130 + #131 |
| #12 | Cross-repo coordinated merge | XL | High | All three admission paths operational |
| #119 | CasePlanModel browser view | M | Med | Blocked on engine REST API |
| #120 | Case dependency graph (D3) | M | High | Needs data exposure |
| #121 | Case memory browser | S | Low | Subsumed by #137 when CBR ships |
| #122 | Agent channel message inbox | S | Low | REST exists, needs UI |
| #123 | Worker session management | M | Med | Blocked on claudony |

## References

- `specs/2026-06-30-governance-workbench-design.md` — governance spec (project repo)
- `blog/2026-06-30-mdp01-a-governance-surface-for-casehub.md` — governance diary
