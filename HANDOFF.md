*Updated: engine#523 closed — removed from backlog.*

# HANDOFF — 2026-06-21

## Last Session

Designed and shipped the GitHub webhook receiver (devtown#15). Three rounds of spec review (17 review points across architecture, engine internals, and completion path). Implementation: hexagonal adapter in `github/` module with JAX-RS endpoint, HMAC-SHA256 verification, typed DTO, signal ordering contract, and CasePlanModel updates for external merge. Discovered engine shallow-copy bug (engine#547) and GoalExpression composition limitation (engine#548). Filed devtown#85 (PR governance dashboard), devtown#86 (CI status), devtown#87 (merge execution).

## Immediate Next Step

Pick next work. Top priorities: (a) fix @QuarkusTest (#83 — blocks integration testing), (b) CI status integration (#86 — next event type handler in the webhook endpoint), (c) demo harness (mock workers + trust seeding + script).

## What's Left

- **devtown#80** — activate production persistence backend · M · Med
- **devtown#83** — @QuarkusTest broken by Hibernate/scheduler exception · S · Med
- **devtown#84** — DevtownMcpTools minor gaps · XS · Low
- **devtown#82** — migrate ErasureReceiptLedgerEntry to foundation version · S · Low
- **engine#547** — WritablePanelImpl should deep-copy initial sub-maps · S · Low
- **engine#548** — composed GoalExpression (nested anyOf/allOf) · M · Med
- **devtown#81** — full gt seance with Doltgres time-travel · L · High
- **parent#207** — distributed ledger: app-specific LedgerEntry subclass persistence · XL · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #83 | Fix @QuarkusTest — Hibernate scheduler exception | S | Med | Blocks all integration testing |
| #86 | CI status integration — check_suite/check_run events | M | Med | Next webhook event type |
| #87 | Merge execution — GitHub API merge worker | M | Med | Requires GitHub App/PAT auth |
| #85 | PR governance dashboard — supersede/relink | M | Med | Requires casehub-ui |
| #11 | Merge queue — CasePlanModel batch-then-bisect | XL | High | Spec not started |
| — | Demo harness (mock workers + trust seeding + demo script) | S | Low | No issue filed |

## References

- `specs/2026-06-19-github-webhook-receiver-design.md` — webhook receiver design spec (rev 4, 17 review points)
- `plans/attic/issue-15-github-webhook-receiver/` — implementation plan (archived)
- `blog/2026-06-21-mdp01-domain-events-are-not-messages.md` — session diary entry
- `github/src/main/java/io/casehub/devtown/github/` — new webhook receiver code
