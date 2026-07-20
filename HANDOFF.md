# HANDOFF — 2026-07-20

## Last Session

Completed #16 (Epic 9: Notification wiring). Discovered the platform notification system already exists in casehub-platform — pivoted from building custom infrastructure to wiring devtown as a consumer. Six SubscribableEvent POJOs, four bridge observers, one subscription registrar. Design review caught 14 fabricated APIs in round 1; all resolved by round 5. Full pipeline activation depends on connectors#86 (delivery bridge) and platform SubscriptionEngine (matching pipeline).

## Immediate Next Step

Pick next work. Tier 2 UI issues blocked on blocks-ui#41. Tier 3 epics available.

## Cross-Module

**Blocked by:**
- `blocks-ui` — blocks-ui#41 (gates ALL devtown UI: #98, #119, #120, #123) · L · Med
- `connectors` — connectors#86 (notification delivery bridge — in progress) · M · Med
- `platform` — SubscriptionEngine + NotificationDispatcher (not yet implemented) · L · High

## What's Left

- Unrecovered artifacts on 14 closed branches — hygiene scan surfaced specs and blogs never promoted to workspace main · S · Low

## What's Next

**Tier 2 — Blocked on blocks-ui#41:**

| # | Description | Scale | Complexity |
|---|-------------|-------|------------|
| #98 | Trust visibility UI | M | Med |
| #119 | CasePlanModel browser | M | Med |
| #120 | Case dependency graph | M | Med |
| #123 | Worker session mgmt UI | S | Med |

**Tier 3 — Epics, long horizon:**

| # | Description | Scale | Complexity |
|---|-------------|-------|------------|
| #81 | Doltgres time-travel (P1.5) | L | High |
| #24 | Contributor trust for OSS | XL | High |

## References

- Landed commit: `7227ab2` on casehubio/devtown main
- ADR: `docs/adr/0001-use-platform-notification-system.md`
- Spec: `docs/specs/issue-16-notification-wiring/2026-07-20-notification-wiring-design.md`
- Blog: `2026-07-20-mdp10-notification-system-i-almost-built.md`
