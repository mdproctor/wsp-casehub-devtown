# HANDOFF — 2026-06-29

## Last Session

Implemented all 5 deferred merge queue integration points (#100): PreferenceProvider (typed keys, YAML config), SLA WorkItems (per-PR with lane-based SLA + breach observer), MergeDecisionLedgerEntry enrichment (batch metadata columns), queue persistence (JPA + SELECT FOR UPDATE), PR review case routing (mutually exclusive bindings). Design reviewed via 5-round adversarial review. Code reviewed — 4 findings fixed (nullable workItemId, double batch formation call, batch record cleanup, JSON escaping). Also fixed #80 (CDI @DefaultBean stubs for MergeClient/CiStatusClient + Jandex for github module). Garden entry GE-20260629-500611 captured H2 UUID native query gotcha.

## Immediate Next Step

Pick from What's Next — #103 (adaptive batch sizing) is the natural continuation since persistence now enables history queries.

## What's Left

- **devtown#97** — TrustGatedAttestationPolicy · M · Med · blocked on qhorus#307
- **devtown#106** — minor review findings: UUID v3 vs v5, stale binding name reference, flaky SLA test · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #103 | Adaptive batch sizing — compute recentFailureRate from history | S | Low | Persistence enables this |
| #101 | GitHub webhook receiver for merge queue admission | M | Med | Third enqueue path |
| #102 | MCP tool enhancements — SLA breach problems + metrics | M | Med | |
| #12 | Cross-repo coordinated merge | XL | High | Merge queue is the foundation |
| #85 | PR governance dashboard | M | Med | Requires casehub-ui |
| #81 | Full gt seance with Doltgres | L | High | |

## References

- `specs/2026-06-28-merge-queue-deferred-design.md` — reviewed spec (workspace + project)
- `adr/casehub-devtown/merge-queue-deferred-20260628-231351/` — adversarial review workspace
