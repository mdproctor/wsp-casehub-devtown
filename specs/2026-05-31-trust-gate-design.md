# devtown Trust Gate — Design Spec
**Issue:** casehubio/devtown#58
**Date:** 2026-05-31
**Branch:** issue-59-s-xs-cleanup

## Problem

devtown#57 (Layer 6) shipped trust-weighted reviewer selection but left the Qhorus trust gate disabled (`casehub.qhorus.commitment.min-obligor-trust=0.0`). The gate is a separate enforcement layer from routing: routing selects the best available agent for a capability; the gate blocks agents below a global trust floor from receiving COMMANDs at all.

Bootstrap agents (Phase 0 — no ledger observations yet) must not be blocked. Enabling the gate at a non-zero threshold without exempting them would reject legitimate agents who simply haven't accumulated enough history.

## Decision: Global gate, not per-capability

The gate and routing answer different questions:
- **Routing**: which agent is the best fit for this capability?
- **Gate**: is this agent trustworthy enough to hold a commitment at all?

Per-capability gate logic was considered and rejected because:
1. `ObligorTrustContext` carries channel name, not capability tag. The channel format `pr-review-{n}/work` identifies a PR-scoped channel, not a specific capability — multiple capabilities route through the same channel for a given PR.
2. Routing already enforces per-capability thresholds at selection time. A per-capability gate would double-enforce the same policy in two layers.
3. The qhorus `ObligorTrustPolicy` SPI is a coarse global floor by design.

## Decision: Implicit bootstrap exemption

`TrustGateService.currentScore(obligorId)` returns `Optional.empty()` when an agent has no ledger observations. This is the exact bootstrap detection signal needed — no separate observation count query, no additional configuration. Aligns with the platform rule: never block on missing trust data.

## Components

### 1. `TrustGatePreferenceKeys` — `domain/trust/`

PreferenceKey constants for the trust gate YAML scope. Follows the same pattern as `TrustRoutingPolicyKeys`.

```java
public final class TrustGatePreferenceKeys {
    public static final PreferenceKey<DoublePreference> MIN_OBLIGOR_TRUST =
        new PreferenceKey<>(
            "casehubio.devtown.trust-gate",
            "min-obligor-trust",
            DoublePreference.of(0.0),   // 0.0 = gate disabled
            DoublePreference::parse);

    private TrustGatePreferenceKeys() {}
}
```

Scope: `casehubio/devtown/trust-gate` (no per-capability suffix — global gate).

### 2. `DevtownObligorTrustPolicy` — `app/routing/`

`@ApplicationScoped` (no `@DefaultBean`) — displaces `DefaultObligorTrustPolicy @DefaultBean` via CDI priority.

```
permits(ObligorTrustContext ctx):
  1. Resolve threshold from YAML at scope casehubio/devtown/trust-gate
  2. If threshold <= 0.0 → permit (gate disabled)
  3. currentScore = trustGateService.currentScore(ctx.obligorId())
  4. If currentScore is empty → permit (bootstrap — no observations)
  5. If currentScore.get() >= threshold → permit
  6. Else → deny
```

Three branches, no loops, no side effects. Fails open on any absent data (steps 2 and 4).

Injects: `TrustGateService`, `PreferenceProvider`. Both already available in `app/` module.

### 3. `trust-routing.yaml` — new scope entry

```yaml
  # Global trust gate — minimum trust floor for non-bootstrap agents.
  # Bootstrap agents (no ledger observations) are always exempt.
  # 0.30 is a permissive production floor — a safety net, not a quality filter.
  # Routing (Layer 6) handles per-capability quality; the gate handles the global floor.
  - scope: casehubio/devtown/trust-gate
    casehubio.devtown.trust-gate.min-obligor-trust: "0.30"
```

0.30 is deliberately low. It blocks only agents with a consistently poor overall record; it does not replace per-capability routing thresholds.

### 4. `application.properties` — no change needed

`casehub.qhorus.commitment.min-obligor-trust` remains at the qhorus default (0.0). Once `DevtownObligorTrustPolicy @ApplicationScoped` is wired, it displaces `DefaultObligorTrustPolicy` entirely — the qhorus property is irrelevant.

## Module placement rationale

`DevtownObligorTrustPolicy` goes in `app/routing/` (not `review/`) because it requires:
- `TrustGateService` — from `casehub-ledger` (runtime), not available in `review/` which has only `casehub-ledger-api`
- `PreferenceProvider` — from `casehub-platform-api`, not in `review/pom.xml`

The `app/` module has both. Placement is consistent with `DevtownTrustRoutingPolicyProvider`.

## Tests

### Unit tests — `DevtownObligorTrustPolicyTest` (pure Java, no Quarkus)

| Scenario | Setup | Expected |
|----------|-------|----------|
| Gate disabled | threshold = 0.0, any score | permits |
| Bootstrap agent | threshold = 0.30, currentScore = empty | permits |
| Score above floor | threshold = 0.30, score = 0.50 | permits |
| Score at floor | threshold = 0.30, score = 0.30 | permits |
| Score below floor | threshold = 0.30, score = 0.20 | denies |
| Null preference (no YAML) | threshold pref = null | permits (fails open) |

Use `@Alternative` static inner classes for `TrustGateService` and `PreferenceProvider` test doubles, per `spi-testing-alternative-inner-classes.md` protocol.

### Integration test — `TrustGateWiringTest` (`@QuarkusTest`)

1. Verify `DevtownObligorTrustPolicy` is the active CDI bean (not `DefaultObligorTrustPolicy`).
2. Inject `DevtownObligorTrustPolicy` directly; call `permits()` with a test `ObligorTrustContext`:
   - Agent with no trust record → permits (bootstrap path)
   - Agent with persisted score below 0.30 → denies (below-threshold non-bootstrap path)

For the below-threshold case: directly insert an `ActorTrustScore` record via `ActorTrustScoreRepository` (or via ledger attestation API). H2 in-memory datasource handles the schema. The test does NOT need a qhorus channel or a real COMMAND — `DevtownObligorTrustPolicy.permits()` is called directly to keep the test focused.

## Acceptance criteria

- [ ] `TrustGatePreferenceKeys.MIN_OBLIGOR_TRUST` added to `domain/trust/`
- [ ] `DevtownObligorTrustPolicy` wired as `@ApplicationScoped` in `app/routing/`
- [ ] `DefaultObligorTrustPolicy` displaced (verified by wiring test)
- [ ] YAML: `casehubio/devtown/trust-gate` scope with `min-obligor-trust: "0.30"`
- [ ] 6 unit test cases pass
- [ ] Integration test: bootstrap agent permits, below-threshold non-bootstrap denies
- [ ] All existing tests pass
