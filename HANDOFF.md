# HANDOFF — 2026-07-20

## Last Session

Completed #16 (Epic 9: Notification wiring). Discovered the platform notification system already exists in casehub-platform — pivoted from building custom infrastructure to wiring devtown as a consumer. Six SubscribableEvent POJOs, four bridge observers, one subscription registrar. Design review caught 14 fabricated APIs in round 1; all resolved by round 5. Full pipeline activation depends on connectors#86 (delivery bridge) and platform SubscriptionEngine (matching pipeline).

## Immediate Next Step

Pick next work. Tier 2 UI issues unblocked (blocks-ui#41 blocks-ui side complete). #119 and #123 wait on blocks-ui#87; #120 needs a blocks-ui issue filed; #98 needs a compose-vs-new-component decision. Tier 3 epics available.

## Cross-Module

**Blocked by:**
- `connectors` — connectors#86 (notification delivery bridge — in progress) · M · Med
- `platform` — SubscriptionEngine + NotificationDispatcher (not yet implemented) · L · High

## UI Issue Status (evaluated 2026-07-20)

**#119 CasePlanModel browser** — covered by blocks-ui#87 (composable case-explorer, in progress on `issue-87-composable-case-explorer` branch). Spec: `blocks-ui/docs/specs/2026-07-20-composable-case-explorer-design.md`. Devtown work: register entity types with endpoints/renderers, no custom components needed. **Wait for blocks-ui#87 to land, then wire.**

**#123 Worker session mgmt UI** — also covered by blocks-ui#87. `EntityStateContributor` SPI + `<entity-command-bar>` handle worker state/commands generically. Devtown work: implement `AgentWorkerStateContributor`, `FlowWorkerStateContributor`, `HumanWorkerStateContributor`. **Wait for blocks-ui#87 to land, then implement SPIs.**

**#120 Case dependency graph** — explicitly out of scope in blocks-ui#87 §13: "many-to-many relationship visualization, fundamentally different from parent→child tree hierarchy, requires graph data model and D3/similar." **Needs a new blocks-ui issue filed before devtown work can begin.**

**#98 Trust visibility UI** — resolved: all three data surfaces exist in blocks-ui (`trust-score-panel`, `routing-rationale`, `trust-feedback-display`). Composition is domain-agnostic → filed blocks-ui#89 (`trust-workbench` composite). **Work slot 8 created** on branch `issue-89-trust-workbench`. Devtown#98 becomes: consume `<trust-workbench>` with endpoints (S/Low). Blocked on blocks-ui#89.

## What's Left

- Unrecovered artifacts on 14 closed branches — hygiene scan surfaced specs and blogs never promoted to workspace main · S · Low

## What's Next

**Tier 2 — UI (unblocked, but dependencies noted above):**

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #98 | Trust visibility UI | S | Low | Blocked on blocks-ui#89 — work slot 8 |
| #119 | CasePlanModel browser | M | Med | Waiting on blocks-ui#87 |
| #120 | Case dependency graph | M | Med | Needs blocks-ui issue filed first |
| #123 | Worker session mgmt UI | S | Med | Waiting on blocks-ui#87 |

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
- blocks-ui case-explorer spec: `blocks-ui/docs/specs/2026-07-20-composable-case-explorer-design.md`
