# HANDOFF — 2026-07-21

## Last Session

Evaluated Tier 2 UI issues. All four (#98, #119, #120, #123) belong in blocks-ui as platform components. Filed blocks-ui#89 (trust-workbench). blocks-ui#87 (case-explorer) delivered — #119 and #123 unblocked. Wrote comprehensive CasePlanModel browser spec with full audit — discovered 3 engine API gaps and 5 engine issues to file. Scope upgraded from S/Low to L/Med.

## Immediate Next Step

Open a CLI in slot 17 and run `work-start`. File 5 engine issues before starting implementation.

## Active Work Slots

| Slot | Branch | Issue | Repos | What to do |
|------|--------|-------|-------|------------|
| 8 | `issue-89-trust-workbench` | blocks-ui#89 | blocks-ui | Build trust-workbench composite (score panel + routing history + feedback). Single-actor focus, layout B (split-workbench), list+detail pattern. |
| 11 | `issue-123-worker-session-mgmt` | devtown#123 | devtown, engine | Implement EntityStateContributor SPIs (Agent, Flow, Human). Engine: SPI + REST + CDI aggregation. Frontend: register worker entity types. |
| 17 | `issue-119-caseplanmodel-browser` | devtown#119 | devtown, engine | Spec: `specs/2026-07-21-caseplanmodel-browser-design.md`. Engine first: ExpressionEvaluator.displayExpression(), MilestoneRuntimeState, goal reached state, case entity REST endpoints, WebSocket events. Then devtown: worker EntityStateContributors, frontend registrations, 3 detail renderers. |

## Cross-Module

**Blocked by:**
- `connectors` — connectors#86 (notification delivery bridge — in progress) · M · Med
- `platform` — SubscriptionEngine + NotificationDispatcher (not yet implemented) · L · High

## UI Issue Status (evaluated 2026-07-20, spec written 2026-07-21)

**#119 CasePlanModel browser** — full audit complete. 3 engine API gaps: ExpressionEvaluator display, MilestoneRuntimeState, goal reached tracking. 5 engine issues to file. **Work slot 17.** L/Med.

**#123 Worker session mgmt UI** — blocks-ui#87 delivered. Devtown + engine: implement EntityStateContributor SPIs. **Work slot 11.** M/Med.

**#120 Case dependency graph** — out of scope in blocks-ui#87 §13. **Needs a new blocks-ui issue filed.**

**#98 Trust visibility UI** — filed blocks-ui#89. Design decisions: single-actor focus, split-workbench layout B, list+detail routing history, scrollable detail. **Work slot 8.**

## What's Left

- Unrecovered artifacts on 14 closed branches · S · Low
- #120 needs blocks-ui issue filed · XS · Low
- 5 engine issues to file for #119 (spec §8) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #119 | CasePlanModel browser | L | Med | Slot 17 — engine + devtown |
| #98 | Trust visibility UI | S | Low | Blocked on blocks-ui#89 — slot 8 |
| #120 | Case dependency graph | M | Med | Needs blocks-ui issue filed |
| #123 | Worker session mgmt UI | M | Med | Slot 11 |
| #81 | Doltgres time-travel (P1.5) | L | High | — |
| #24 | Contributor trust for OSS | XL | High | — |

## References

- CasePlanModel browser spec: `specs/2026-07-21-caseplanmodel-browser-design.md`
- blocks-ui case-explorer spec: `blocks-ui/docs/specs/2026-07-20-composable-case-explorer-design.md`
- blocks-ui#89 trust-workbench design decisions captured in session conversation
