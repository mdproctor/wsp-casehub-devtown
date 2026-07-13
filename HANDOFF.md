# HANDOFF — 2026-07-14

## Last Session

#133 CBR-enhanced reviewer matching shipped. `ExperienceAnalyser` shared utility in engine-api, `TrustWeightedAgentStrategy` enhanced with `cbrWeight` scoring in engine-ledger, `DevtownTrustRoutingPolicyProvider` wired with per-capability cbrWeight defaults (0.2 for review capabilities), `pr-review.yaml` gains CBR config with JQ feature extraction. Design review caught scoring formula penalty bug, phantom types, and outcome weight mismatch ($10.83, 2 rounds). Engine branch `issue-133-cbr-experience-analyser` (3 commits, not yet merged to engine main). Devtown landed as `27c03ce` on main.

## Immediate Next Step

Engine branch `issue-133-cbr-experience-analyser` in casehub-engine needs merging to engine main (3 commits: ExperienceAnalyser, TrustRoutingPolicy.cbrWeight, TrustWeightedAgentStrategy enhancement). Run `work-end` on that branch or merge manually.

## What's Left

- **Engine branch** — `issue-133-cbr-experience-analyser` in casehub-engine: 3 commits not yet on engine main · S · Low
- **parent#361** — docs: sync casehub-devtown.md for CBR Phase 1+2 · XS · Low
- **blocks#55** — refactor CbrAgentRoutingStrategy to use shared ExperienceAnalyser · S · Low (blocked on engine merge)

## Cross-Module

**Blocked by:**
- `pages` / `blocks-ui` — pages#111, blocks-ui#41 (gate ALL devtown UI) · L · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #124 | PR supersede/relink backend | M | Med | Independent |
| #146 | Per-capability precedent activation thresholds | S | Low | Extends #132 |
| #147 | Similarity-weighted evidence accumulation | S | Med | Extends #132 |
| blocks-ui#41 | blocks-ui Phase 1 — consume shipped components | L | Med | Waiting on blocks-ui |

## References

- Spec: `specs/issue-133-cbr-reviewer-matching/2026-07-13-cbr-reviewer-matching-design.md`
- Blog: `blog/2026-07-14-mdp01-teaching-routing-to-remember.md`
- Design review: `~/adr/casehub-devtown/cbr-reviewer-matching-20260713-215946/`
