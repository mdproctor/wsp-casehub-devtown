# HANDOFF — 2026-07-13

## Last Session

#141 EvidentialChecker V1-V4 implemented and closed. `EvidentialAttestationPolicy` wraps `TrustGatedAttestationPolicy` at CDI `@Priority(2)` — runs V2/V3 checks for DONE claims from agents in configured low-trust phases. Per-capability phase configuration in `DevtownTrustRoutingPolicyProvider`. Also fixed seven SNAPSHOT drift issues from engine/qhorus/platform API changes. Landed as `b5d4964` on main.

## Immediate Next Step

Pick next work. CBR Phase 2 (#132, #133) or #124 (supersede/relink) are candidates.

## What's Left

- **#144** — TrustFeedbackClosedLoopTest broken (engine SNAPSHOT drift, `PlanItemStatus` rename in work-adapter) · XS · Low
- **#124** — supersede/relink backend · M · Med
- **parent#361** — docs: sync casehub-devtown.md for CBR Phase 1 · XS · Low

## Cross-Module

**Blocked by:**
- `pages` / `blocks-ui` — pages#111, blocks-ui#41 (gate ALL devtown UI) · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #132 | CBR-enhanced capability activation | M | Med | Check engine #505 first |
| #133 | CBR-enhanced reviewer matching | M | Med | Check engine #505 first |
| #124 | PR supersede/relink backend | M | Med | Independent |
| blocks-ui#41 | blocks-ui Phase 1 — consume shipped components | L | Med | Waiting on blocks-ui |

## References

- Spec: `docs/specs/2026-07-12-evidential-checker-integration-design.md`
- Roadmap: `ROADMAP.md` (workspace root)
- Blog: `blog/2026-07-13-mdp01-trust-but-verify.md`
