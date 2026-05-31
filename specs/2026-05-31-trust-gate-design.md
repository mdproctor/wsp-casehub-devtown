# devtown Trust Gate — Design Spec
**Issue:** casehubio/devtown#58
**Date:** 2026-05-31
**Branch:** issue-59-s-xs-cleanup

## What this gate actually does (and what it doesn't)

The gate is a **safety net for agents with a persistently poor recorded history**. It blocks agents whose global trust score falls below a configurable floor from receiving any COMMAND.

It does **not** protect against the merge-executor bootstrap risk described in LAYER-LOG §Layer 6 §Known gaps. The gate always exempts bootstrap agents (no observations → permit). A zero-history agent dispatched to execute a merge passes the gate by design, because the platform rule is "never block on missing trust data." The merge-executor bootstrap gap — where Phase 0 availability routing can select a zero-history agent for an irreversible operation — is a routing-layer concern tracked in devtown#62 (HumanOversight when no agent meets minimumObservations for merge-executor).

This framing is correct. The gate and routing enforce different properties at different layers:
- **Routing**: which agent is the best fit for this capability right now?
- **Gate**: does this agent have a reliably poor *recorded* history that disqualifies it globally?

## Why the gate is global, not per-capability

Per-capability gate logic was considered and rejected for a structural reason: `ObligorTrustContext` carries only `obligorId`, `channelId`, and `channelName` — not a capability tag. The channel name (`pr-review-{n}/work`) identifies a PR-scoped channel, not a capability; multiple capabilities route through the same channel for a given PR. Per-capability gate semantics cannot be implemented with the current SPI regardless of any logical argument.

Logical reasons follow from the structural constraint:
- Routing already applies per-capability thresholds at selection time; re-enforcing them at the gate would duplicate policy across two layers.
- The qhorus `ObligorTrustPolicy` SPI is a coarse global floor by design.

Platform extension path: if per-capability gate enforcement is needed in future, file against casehub-qhorus to add capability tag to `ObligorTrustContext`. That is deferred; this issue does not require it.

## Why bootstrap exemption uses `Optional.empty()`

`TrustGateService.currentScore(actorId)` returns `Optional.empty()` when an agent has no ledger observations. This is the exact bootstrap detection signal needed — no separate observation count query, no additional configuration.

`TrustScoreJob` has a `GlobalScoreStrategy` field and `ActorTrustScoreRepository` exposes `updateGlobalTrustScore()`, confirming that a global aggregate score is computed and stored separately from per-capability scores. `findByActorId()` (no capability filter) returns this global score; `currentScore(actorId)` delegates to it.

**Dependency to verify:** the integration test must confirm that `currentScore(actorId)` returns `Optional.empty()` for a genuinely unseen agent (no rows in `ActorTrustScore` for that actor), and returns a value once a global score row exists. If `TrustScoreJob` only writes capability-scoped rows and never writes a global aggregate, `currentScore(actorId)` would always return empty and the gate would be permanently dormant. The test catches this assumption failure.

## Components

### 1. `TrustGatePreferenceKeys` — `domain/trust/`

PreferenceKey constants for the trust gate YAML scope. Follows the same pattern as `TrustRoutingPolicyKeys`.

```java
public final class TrustGatePreferenceKeys {
    /**
     * Global trust floor for non-bootstrap agents. 0.0 = gate disabled.
     * Note: constructor default is unused at runtime — MapPreferences.get()
     * returns null for absent keys. See null guard in DevtownObligorTrustPolicy.
     */
    public static final PreferenceKey<DoublePreference> MIN_OBLIGOR_TRUST =
        new PreferenceKey<>(
            "casehubio.devtown.trust-gate",
            "min-obligor-trust",
            DoublePreference.of(0.0),
            DoublePreference::parse);

    private TrustGatePreferenceKeys() {}
}
```

Scope: `casehubio/devtown/trust-gate` (no per-capability suffix — global gate).

### 2. `DevtownObligorTrustPolicy` — `app/routing/`

`@ApplicationScoped` (no `@DefaultBean`) — displaces `DefaultObligorTrustPolicy @DefaultBean` via CDI priority.

```
permits(ObligorTrustContext ctx):
  1. Resolve prefs from PreferenceProvider at scope casehubio/devtown/trust-gate
  2. thresholdPref = prefs.get(TrustGatePreferenceKeys.MIN_OBLIGOR_TRUST)
     floor = thresholdPref != null ? thresholdPref.value() : 0.0
  3. If floor <= 0.0 → return true (gate disabled)
  4. score = trustGateService.currentScore(ctx.obligorId())
  5. If score.isEmpty() → return true (bootstrap — no observations yet)
  6. return score.get() >= floor
```

Three branches, no loops, no side effects. Fails open on absent data (steps 3 and 5). The null guard in step 2 is required because `MapPreferences.get()` returns null for absent YAML keys — it does not fall back to the `PreferenceKey` constructor default.

Injects: `TrustGateService`, `PreferenceProvider`. Both available in `app/` module. `TrustGateService` is in `casehub-ledger` runtime (not `casehub-ledger-api`), which is why this class belongs in `app/` not `review/` (which has only `casehub-ledger-api`).

### 3. `trust-gate.yaml` — new file at `app/src/main/resources/casehub/devtown/`

```yaml
# Global trust gate — minimum trust floor for non-bootstrap agents.
# Bootstrap agents (no ledger observations) are always exempt.
# 0.30 is a permissive production floor — a safety net, not a quality filter.
# Routing (Layer 6) handles per-capability quality; the gate handles the global floor.
entries:
  - scope: casehubio/devtown/trust-gate
    casehubio.devtown.trust-gate.min-obligor-trust: "0.30"
```

Separate file — trust gate and trust routing are independent concerns with different scopes.

### 4. `application.properties` — add trust-gate file

```properties
casehub.platform.config.files=classpath:casehub/devtown/trust-routing.yaml,classpath:casehub/devtown/trust-gate.yaml
```

### 5. `casehub.qhorus.commitment.min-obligor-trust` — no change

Stays at qhorus default (0.0 — gate disabled at qhorus level). Once `DevtownObligorTrustPolicy @ApplicationScoped` is wired, it displaces `DefaultObligorTrustPolicy` entirely and the qhorus property is irrelevant.

## Module placement rationale

`DevtownObligorTrustPolicy` goes in `app/routing/` (not `review/`) because it requires:
- `TrustGateService` — from `casehub-ledger` runtime, absent from `review/pom.xml` which has only `casehub-ledger-api`
- `PreferenceProvider` — from `casehub-platform-api`, absent from `review/pom.xml`

Consistent with `DevtownTrustRoutingPolicyProvider` placement.

## Tests

### Unit tests — `DevtownObligorTrustPolicyTest` (pure Java, no Quarkus)

Use `@Alternative` static inner classes for test doubles per `spi-testing-alternative-inner-classes.md`.

Both rows below exercise the same branch (step 3: `floor <= 0.0 → true`) but via distinct code paths — the null guard is a separate path from an explicit 0.0 preference.

| Scenario | Setup path | Expected |
|----------|------------|----------|
| Gate disabled (null pref) | `prefs.get()` returns null → null guard → floor = 0.0 | permits |
| Gate disabled (explicit 0.0) | `prefs.get()` returns `DoublePreference.of(0.0)` → floor = 0.0 | permits |
| Bootstrap agent | threshold = 0.30, currentScore = empty | permits |
| Score above floor | threshold = 0.30, score = 0.50 | permits |
| Score at floor | threshold = 0.30, score = 0.30 | permits |
| Score below floor | threshold = 0.30, score = 0.20 | denies |

### Integration test — `TrustGateWiringTest` (`@QuarkusTest`)

**CDI note:** `TrustGateService` is already discoverable in `@QuarkusTest` — `TrustWeightedAgentStrategy` (indexed via `casehub-engine-ledger`) injects it, so it is already wired in the current test context. The test-specific gotcha is `ActorTrustScore` JPA entity seeding: inserting a trust score row requires either using the `ActorTrustScoreRepository` bean directly or inserting via `EntityManager`. The test must confirm the H2 schema includes the `actor_trust_score` table (created by Flyway or schema generation in test mode).

**Test cases:**

1. **CDI wiring**: verify `DevtownObligorTrustPolicy` is the active `ObligorTrustPolicy` bean (not `DefaultObligorTrustPolicy`).

2. **Bootstrap path**: call `permits()` with a `ObligorTrustContext` for an agent with no `ActorTrustScore` rows → must return true.

3. **Below-threshold non-bootstrap path**: insert an `ActorTrustScore` row for a test agent with global score 0.15, threshold 0.30 configured → call `permits()` → must return false.

4. **Global score assumption check**: insert an `ActorTrustScore` row for capability `security-review` only (no global row) → call `trustGateService.currentScore(actorId)` directly (inject `TrustGateService` as a separate `@Inject` field alongside the policy) → verify it returns empty (confirming the gate exempts agents with only capability scores but no global aggregate, and revealing any assumption about global score population being automatic). `TrustGateService` is `@ApplicationScoped` and resolves cleanly from the test context.

## Acceptance criteria

- [ ] `TrustGatePreferenceKeys.MIN_OBLIGOR_TRUST` added to `domain/trust/`
- [ ] `DevtownObligorTrustPolicy` wired as `@ApplicationScoped` in `app/routing/`
- [ ] `DefaultObligorTrustPolicy` displaced (verified by wiring test)
- [ ] `trust-gate.yaml` created at `casehub/devtown/trust-gate.yaml`
- [ ] `application.properties` updated to include `trust-gate.yaml`
- [ ] 6 unit test cases pass
- [ ] 4 integration test cases pass (including bootstrap path and below-threshold rejection)
- [ ] All existing 35 tests pass
- [ ] devtown#62 filed (merge-executor bootstrap → HumanOversight) ✅ done
