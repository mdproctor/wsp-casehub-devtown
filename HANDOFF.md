# HANDOFF — 2026-06-23

## Last Session

Branch `issue-89-github-combined-status` closed. Replaced in-memory CI aggregation (`ConcurrentHashMap`) with GitHub Checks API as single source of truth. `CiStatusClient` SPI in domain, `GitHubCiStatusClient` in github module with NON_BLOCKING conclusion classification (success/neutral/skipped), pagination safety, `Unavailable` error handling. Spec went through 3 review rounds catching 4 production bugs before code was written. Drive-by `@Alternative @Priority(2)` fix reverted — discovered 3-bean CDI chain gotcha (GE-20260623-c651a1). Pre-existing compilation failure in review/ module (engine API breaking change — `PlannedAction` and `Capability` renamed/removed upstream).

## Immediate Next Step

Fix the review module compilation failure — `PlannedAction` and `Capability` from `casehub-engine-api` no longer exist. This blocks `mvn clean install`. File an issue if not already tracked.

## What's Left

- **Review module compilation failure** — `PlannedAction`/`Capability` removed from engine API SNAPSHOT · S · Med
- **devtown#80** — activate production persistence backend · M · Med
- **engine#547** — WritablePanelImpl deep-copy bug · S · Low
- **engine#548** — composed GoalExpression · M · Med
- **engine#557** — ActorStateResource @PermitAll · XS · Low
- **devtown#81** — full gt seance with Doltgres · L · High
- **parent#207** — distributed ledger subclass persistence · XL · High
- **CDI @Alternative @Priority chain cleanup** — all three PrReviewApplicationService impls need coordinated fix · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #13 | Trust-weighted reviewer routing (layer 6) | XL | High | Epic 6 — prerequisite for meaningful demo |
| #85 | PR governance dashboard — supersede/relink | M | Med | Requires casehub-ui |
| #11 | Merge queue — CasePlanModel batch-then-bisect | XL | High | Spec not started |
| — | Demo harness (mock workers + trust seeding + script) | M | Low | Deferred until layer 6 lands |

## References

- `specs/2026-06-23-github-combined-status-design.md` — combined-status spec (rev 3, promoted to project)
- `blog/2026-06-23-mdp01-the-spec-that-fixed-itself.md` — session diary
