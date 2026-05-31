# HANDOFF — 2026-05-31

## Last Session

Layer 6 (trust routing) complete and merged to casehubio/devtown main. Wired
`TrustWeightedAgentStrategy` from `casehub-engine-ledger` with
`DevtownTrustRoutingPolicyProvider` (YAML-backed per-capability policies).
`FALSE_POSITIVE_RATE` renamed to `PRECISION` — semantics corrected before any
ledger data existed. 35 tests pass. Subagent-driven development, 9 tasks.

Also: identified CaseMemoryStore consumption opportunities for devtown and filed
platform#48 (consumer requirements from devtown's perspective — emission pattern,
attribute conventions, multi-entity recall, temporal relevance).

## Immediate Next Step

Create Layer 4 issue (casehub-ledger tamper-evident audit trail), then brainstorm.
Layer 4 is unblocked — engine#326 closed. Design Layer 4 + devtown#43 (CaseMemoryStore)
jointly: the `/observe` channel drives both the ledger audit trail and future
memory facts. platform#48 should be addressed before devtown#43 brainstorm.

## What's Left

- **devtown#58** — Qhorus trust gate (`casehub.qhorus.commitment.min-obligor-trust`); needs bootstrap exemption design · M · Med
- **parent#115** — Replace AML hardcoded trust policy pattern with per-field `PreferenceKey` (devtown#57 is reference impl) · S · Low
- **parent#120** — Add `trust-maturity-model.md` to protocols INDEX files · XS · Low
- **parent#121** — Sync casehub-devtown.md Layer 6 status and two new deps · XS · Low
- **parent#122** — Add engine-ledger + platform-config dep rows to PLATFORM.MD · XS · Low
- **devtown#59** — Move `DoublePreference` + `IntPreference` to `domain.preferences` package · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Layer 4 — casehub-ledger tamper-evident audit trail (new issue needed) | L | Med | Design with devtown#43 jointly — both use /observe |
| #43 | CaseMemoryStore — contributor + reviewer context | M | Med | Wait for platform#48 response before brainstorm |
| — | Layer 7 — Gastown comparison | M | Med | After Layer 4 |
| #58 | Qhorus trust gate configuration | M | Med | Bootstrap exemption design needed |

## References

- `blog/2026-05-31-mdp01-layer6-trust-routing.md` — this session's diary
- `docs/specs/2026-05-30-layer6-trust-routing-design.md` — Layer 6 design spec
- Garden: GE-20260530-9cdfb5 (MapPreferences null return), GE-20260530-0bee65 (plain JAR Flyway), GE-20260530-9b5bbe (per-field PreferenceKey)
- Protocol: PP-20260531-300481 — harness-trust-policy-source-split
