# HANDOFF — 2026-07-13

## Last Session

#132 CBR-enhanced capability activation implemented and closed. `PrecedentActivationPolicy` evaluates findings-based evidence from similar past cases against dual thresholds (minFindings=2, minFraction=0.4). Pre-computed at recall time by `CaseMemoryRecaller`, serialized as `memory.precedentActivations`. Two precedent-triggered bindings in both Java DSL and YAML case definitions with `activationSource: "precedent"` audit trail. Critical goal gating fix ensures cases gate on precedent-triggered review outcomes. Design review caught the gating flaw ($13.22, 3 rounds). `Precedent` moved to `domain/cbr/`, `CapabilityOutcome` consolidates findings logic. Landed as `28026e0` on main.

## Immediate Next Step

Pick next work. CBR Phase 2 continues with #133 (reviewer matching), or #124 (supersede/relink) is independent.

## What's Left

- **#124** — supersede/relink backend · M · Med
- **parent#361** — docs: sync casehub-devtown.md for CBR Phase 1+2 · XS · Low

## Cross-Module

**Blocked by:**
- `pages` / `blocks-ui` — pages#111, blocks-ui#41 (gate ALL devtown UI) · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #133 | CBR-enhanced reviewer matching | M | Med | Engine #505 closed, unblocked |
| #124 | PR supersede/relink backend | M | Med | Independent |
| #146 | Per-capability precedent activation thresholds | S | Low | Extends #132 |
| #147 | Similarity-weighted evidence accumulation | S | Med | Extends #132 |
| blocks-ui#41 | blocks-ui Phase 1 — consume shipped components | L | Med | Waiting on blocks-ui |

## References

- Spec: `specs/issue-132-cbr-capability-activation/2026-07-13-cbr-capability-activation-design.md`
- Blog: `blog/2026-07-13-mdp03-learning-from-your-own-history.md`
- Roadmap: `ROADMAP.md` (workspace root)
