# HANDOFF — 2026-07-07

## Last Session

Fixed all 20 ledger-enabled `@QuarkusTest` failures (#139 SNAPSHOT drift + #142 stale schema). Root cause of #142: V2003 Flyway migration explicitly dropped `tenancy_id` from `merge_decision_ledger_entry`. Secondary fixes: V2002 FK removed (standalone entity), `@Transactional` → `QuarkusTransaction.requiringNew()` on `@ObservesAsync` observer, supplemental JPQL query for standalone `MergeDecisionLedgerEntry` in ComplianceService. 333/333 tests pass. Squashed 3→1, pushed to origin/main.

## Immediate Next Step

Start #97 (TrustGatedAttestationPolicy) — now unblocked by the ledger test fixes. Run `/work` to begin.

## What's Left

- **devtown#127** — PrReviewCaseTracker startup hydration (depends on engine query API) · S · Med
- **devtown#128** — cursor-based pagination for governance REST endpoints · S · Low
- **devtown#124** — supersede/relink backend (scoped out of #85) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #97 | TrustGatedAttestationPolicy — capability-scoped evidential verification | M | Med | Unblocked by #142 fix |
| #129 | Epic 11: Case-Based Reasoning (9 child issues) | XL | High | Phase 1: #130 + #131 |
| #12 | Cross-repo coordinated merge | XL | High | All three admission paths operational |
| #119 | CasePlanModel browser view | M | Med | Blocked on engine REST API |

## Known Limitation

`MergeDecisionLedgerEntry` extends `@MappedSuperclass` (not `JpaLedgerEntry` JOINED). Consequences: not in Merkle hash chain (compliance PARTIAL), not in `findBySubjectId()` queries (needs supplemental JPQL), supplement hydration requires `supplementJson` parsing. Migrating to `JpaLedgerEntry` needs investigation (#142 notes the attempt in #139 broke the async observer).

## References

- Garden: GE-20260707-649b02 (Flyway DROP COLUMN gotcha), GE-20260707-4e41c3 (schema-management override)
- Garden: GE-20260429-da95ec REVISED — added QuarkusTransaction.requiringNew() alternative
