# HANDOFF — 2026-07-20

## Last Session

Completed #16 (Epic 9: Notification wiring). Evaluated all Tier 2 UI issues (#98, #119, #120, #123) — all four belong in blocks-ui as platform components, not devtown-specific views. Filed blocks-ui#89 (trust-workbench). blocks-ui#87 (case-explorer) delivered — #119 and #123 unblocked. Three work slots created.

## Immediate Next Step

Three work slots ready for implementation. Open a CLI in any slot and run `work-start`.

## Active Work Slots

| Slot | Branch | Issue | Repos | What to do |
|------|--------|-------|-------|------------|
| 8 | `issue-89-trust-workbench` | blocks-ui#89 | blocks-ui | Build trust-workbench composite (score panel + routing history + feedback). Single-actor focus, layout B (split-workbench), list+detail pattern. |
| 10 | `issue-119-caseplanmodel-browser` | devtown#119 | devtown | Register entity types with endpoints, consume `<case-definition-browser>` / `<case-explorer>`. Add PR-specific detail renderers. |
| 11 | `issue-123-worker-session-mgmt` | devtown#123 | devtown, engine | Implement EntityStateContributor SPIs (Agent, Flow, Human). Engine: SPI + REST + CDI aggregation. Frontend: register worker entity types. |

## Cross-Module

**Blocked by:**
- `connectors` — connectors#86 (notification delivery bridge — in progress) · M · Med
- `platform` — SubscriptionEngine + NotificationDispatcher (not yet implemented) · L · High

## UI Issue Status (evaluated 2026-07-20)

**#119 CasePlanModel browser** — blocks-ui#87 delivered. Devtown: register entity types, consume convenience components. **Work slot 10.** S/Low.

**#123 Worker session mgmt UI** — blocks-ui#87 delivered. Devtown + engine: implement EntityStateContributor SPIs, register worker entity types. **Work slot 11.** M/Med.

**#120 Case dependency graph** — out of scope in blocks-ui#87 §13. **Needs a new blocks-ui issue filed before devtown work can begin.**

**#98 Trust visibility UI** — filed blocks-ui#89. Design decisions made: single-actor focus, split-workbench layout B, list+detail routing history (not stacked rationale), scrollable detail (not tabs). **Work slot 8.**

## What's Left

- Unrecovered artifacts on 14 closed branches · S · Low
- #120 needs blocks-ui issue filed · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #98 | Trust visibility UI | S | Low | Blocked on blocks-ui#89 — slot 8 |
| #119 | CasePlanModel browser | S | Low | Unblocked — slot 10 |
| #120 | Case dependency graph | M | Med | Needs blocks-ui issue filed |
| #123 | Worker session mgmt UI | M | Med | Unblocked — slot 11 |
| #81 | Doltgres time-travel (P1.5) | L | High | — |
| #24 | Contributor trust for OSS | XL | High | — |

## References

- blocks-ui case-explorer spec: `blocks-ui/docs/specs/2026-07-20-composable-case-explorer-design.md`
- blocks-ui#89 trust-workbench design decisions captured in this session's conversation
- Landed commit: `7227ab2` on casehubio/devtown main
