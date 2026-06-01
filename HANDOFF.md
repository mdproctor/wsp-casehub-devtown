# HANDOFF — 2026-06-01

## Last Session

S/XS cleanup branch (`issue-59-s-xs-cleanup`) closed. Moved `DoublePreference`/`IntPreference`
to `domain.preferences` (#59). Closed superseded issues #2, #4, #6 (all addressed by Layer 6),
closed #34 (Hibernate Reactive incompatible with H2 — not an XS fix), deferred #18.
Implemented Qhorus trust gate (#58): `DevtownObligorTrustPolicy @ApplicationScoped`, global
floor 0.30 via `trust-gate.yaml`, bootstrap exemption via `Optional.empty()`, 46 tests pass.
Fixed pre-existing CDI build failure (`DefaultTestPrincipal` + `MockGroupMembershipProvider`
ambiguity in production augmentation). Filed devtown#62 (merge-executor bootstrap → HumanOversight
— the trust gate structurally cannot address this; `ObligorTrustContext` carries no capability tag).

## Immediate Next Step

*Unchanged — `git show HEAD~1:HANDOFF.md`*

(Create Layer 4 issue — casehub-ledger tamper-evident audit trail — then brainstorm.
Design jointly with devtown#43; both use the `/observe` channel. Wait for platform#48
response before devtown#43 brainstorm.)

## What's Left

- **devtown#62** — merge-executor bootstrap path to HumanOversight when no agent meets minimumObservations · S · Med
- **parent#115** — Replace AML hardcoded trust policy with per-field `PreferenceKey` (devtown#57 reference impl) · S · Low
- **parent#120** — Add `trust-maturity-model.md` to protocols INDEX files · XS · Low
- **parent#121** — Sync casehub-devtown.md Layer 6 status and two new deps · XS · Low
- **parent#122** — Add engine-ledger + platform-config dep rows to PLATFORM.MD · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Layer 4 — casehub-ledger tamper-evident audit trail (new issue needed) | L | Med | Design with devtown#43 jointly — both use /observe |
| #43 | CaseMemoryStore — contributor + reviewer context | M | Med | Wait for platform#48 response before brainstorm |
| #62 | merge-executor bootstrap → HumanOversight (TrustWeightedSelectionStrategy) | S | Med | Gate cannot fix this — routing layer change needed |
| — | Layer 7 — Gastown comparison | M | Med | After Layer 4 |

## References

- `blog/2026-05-31-mdp02-xs-s-cleanup-trust-gate.md` — this session's diary
- `specs/2026-05-31-trust-gate-design.md` — trust gate design spec (promoted to project docs/specs/)
- Garden: GE-20260531-769f9c (updateGlobalTrustScore silent no-op — use upsert with ScoreType.GLOBAL)
- Protocols: PP-20260531-ef4da8 (domain.preferences placement), PP-20260531-ea9945 (one YAML per concern)
