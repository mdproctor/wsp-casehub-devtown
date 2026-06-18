# HANDOFF — 2026-06-18

## Last Session

Shipped devtown#17 (Epic 10: Observability and operational tooling). Built 11 MCP tools (8 read + 3 write) + W3C PROV-DM export in `DevtownMcpTools.java`. Full Gastown CLI parity achieved. Fixed 3 pre-existing ledger API breaks along the way. Filed 5 issues: devtown#80 (production persistence), devtown#81 (full gt seance), devtown#83 (@QuarkusTest broken), devtown#84 (minor MCP tool gaps), engine#523 (CaseInstanceRepository listing methods).

## Immediate Next Step

Pick next work. Pause stack has `issue-14-failure-handling` (#14). Three priorities: (a) fix @QuarkusTest (devtown#83 — blocks all integration testing), (b) resume #14 failure handling, (c) pick from backlog.

## What's Left

- **devtown#80** — activate production persistence backend (remove compile-scope persistence-memory) · M · Med
- **devtown#83** — @QuarkusTest broken by Hibernate/scheduler exception · S · Med
- **devtown#84** — DevtownMcpTools minor gaps (WorkItemStore, reroute policy context) · XS · Low
- **engine#523** — add listing methods to CaseInstanceRepository SPI · S · Low
- **devtown#81** — full gt seance equivalent with Doltgres time-travel · L · High
- **parent#207** — distributed ledger: app-specific LedgerEntry subclass persistence · XL · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #14 | Failure handling — DECLINED vs FAILED routing | L | High | Paused — spec complete, implementation not started |
| #83 | Fix @QuarkusTest — Hibernate scheduler exception | S | Med | Blocks all integration testing |
| — | Demo harness (mock workers + trust seeding + demo script) | S | Low | No issue filed |
| — | casehub-ui Phase 1 panels | M | Med | Requires casehub-ui repo |
| #24 | Contributor trust for open source PR routing | XL | High | Idea/proposal |

## References

- `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java` — 11 MCP tools
- `app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java` — event-sourced read model
- `docs/specs/issue-17-observability-tooling/` — design spec (promoted to project)
- `specs/issue-17-observability-tooling/` — workspace copy of design spec
