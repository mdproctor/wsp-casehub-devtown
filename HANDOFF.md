# HANDOFF — 2026-06-22

## Last Session

Branch `issue-90-wire-platform-oidc` closed. OIDC security wiring shipped: default-deny via `deny-unannotated-endpoints=true`, `@RolesAllowed(DevtownRoles.ADMIN)` on 5 REST endpoints, `@PermitAll` on webhook, tenant isolation via `CurrentPrincipal.tenancyId()` (REST + MCP), MCP path-based auth, `ActorStateResource` excluded pending engine#557. Spec went through 5 review rounds.

## Immediate Next Step

Pick next work. Top candidates: demo harness (mock workers + trust seeding + script), or #89 GitHub REST API combined-status check.

## What's Left

- **devtown#80** — activate production persistence backend · M · Med
- **engine#547** — WritablePanelImpl deep-copy bug · S · Low
- **engine#548** — composed GoalExpression · M · Med
- **engine#557** — ActorStateResource @PermitAll (filed this session) · XS · Low
- **devtown#81** — full gt seance with Doltgres · L · High
- **parent#207** — distributed ledger subclass persistence · XL · High
- **devtown#89** — GitHub REST API combined-status check · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #89 | GitHub REST API combined-status for multi-suite CI | S | Med | Eliminates aggregation gap |
| #85 | PR governance dashboard — supersede/relink | M | Med | Requires casehub-ui |
| #11 | Merge queue — CasePlanModel batch-then-bisect | XL | High | Spec not started |
| #13 | Trust-weighted reviewer routing (layer 6) | XL | High | Epic 6 — prerequisite for meaningful demo |
| — | Demo harness (mock workers + trust seeding + script) | M | Low | Deferred until layer 6 lands |

## References

- `specs/2026-06-22-oidc-wiring-design.md` — OIDC wiring spec (rev 5, promoted to project)
- `blog/2026-06-22-mdp02-the-security-layers-that-dont-talk.md` — session diary
