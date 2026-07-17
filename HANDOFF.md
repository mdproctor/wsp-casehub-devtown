# HANDOFF — 2026-07-17

## Last Session

Epic #12 Phase 1 completed. #156 (cross-repo CasePlanModel), #157 (coordinated-merge worker), and #159 (webhook routing / coordination tracking) implemented and closed on one branch. Application-layer coordination over engine sub-cases — per-repo reviews are standard pr-review cases with a `coordinatedChange` context flag suppressing auto-merge. `CoordinatedChangeTracker` with `AtomicBoolean` for exactly-once completion transition, `CoordinatedChangeObserver` for lifecycle event signaling, `CoordinatedChangeService` with all-or-none startup atomicity. 7-round adversarial design review ($16.95, 25 issues, 16 verified, 9 accepted). Landed as `390f084` on main.

## Immediate Next Step

Pick next epic #12 work. Phase 2 is now unblocked: #158 (coordinated-rollback worker) and #160 (end-to-end integration test). #158 uses `RevertClient` from #155 and the rollback binding already declared in `coordinated-change.yaml`. #160 depends on #158.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Next

**Tier 1 — Unblocked now (Epic #12 Phase 2):**

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #158 | Coordinated-rollback worker — revert merges on sub-case FAULT | M | Med | Rollback binding declared in YAML; uses RevertClient from #155 |

**Tier 2 — Blocked on Phase 2:**

| # | Description | Scale | Complexity | Blocker |
|---|-------------|-------|------------|---------|
| #160 | End-to-end integration test | M | Med | #158 |

**Tier 3 — Other open work:**

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Spec: `docs/specs/2026-07-17-cross-repo-coordinated-merge-design.md`
- Design review: `~/adr/casehub-devtown/cross-repo-coordinated-merge-20260717-121432/`
- Blog: `blog/2026-07-17-mdp06-when-subcases-arent-subcases.md`
- Garden: GE-20260717-f7dc41 (CaseHubRuntime.startCase parentCaseId bypass gotcha)
