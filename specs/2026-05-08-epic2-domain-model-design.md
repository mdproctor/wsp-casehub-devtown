# Epic 2: Domain Model Design Spec
2026-05-08

## Context

`devtown-domain` is the authoritative source of the software engineering domain vocabulary for casehub-devtown. Everything in Epics 3–10 (case plan models, routing, merge queue, GitHub integration) depends on this vocabulary. No foundation dependency — pure Java.

This spec covers the three types and the extensibility SPI that Epic 2 produces.

## Platform Coherence

**Step 1 — Already exists?** The ledger's `CapabilityTag` class defines `GLOBAL = "*"` as a sentinel. `LedgerAttestation.capabilityTag` and `TrustGateService.meetsThreshold(actorId, capabilityTag, minTrust)` both accept the capability tag as a plain `String`. DevTown's constants are the domain-specific string values that flow into those slots — no duplication.

**Step 2 — Right repo?** Yes. Capability tags, trust dimensions, and routing thresholds require knowledge of software engineering (PRs, security review, merge queues). They belong in the application tier, not the foundation.

**Step 4 — Consistent with platform pattern?** The `public static final String` pattern matches `CapabilityTag` in `casehub-ledger`. The SPI placement follows the platform rule: interface in `domain.spi`, default implementation in the owning module.

**Platform protocol update:** During this design, the PLATFORM.md "no-op defaults" rule was found to be too coarse. A refined rule was committed to `casehub-parent` (parent#12): operational SPIs get a no-op default; vocabulary/registry SPIs get a *populated* default. The distinction: can the system function with an empty implementation? If yes → no-op. If no → populated.

## Types

### `DevtownCapability`

Constants class. Values are the string capability tags that flow directly into:
- `LedgerAttestation.capabilityTag`
- `TrustGateService.meetsThreshold(actorId, capabilityTag, minTrust)`
- `casehub-engine` `Capability.name` when building case definitions (Epic 3)

| Constant | Value |
|----------|-------|
| `CODE_ANALYSIS` | `"code-analysis"` |
| `SECURITY_REVIEW` | `"security-review"` |
| `ARCHITECTURE_REVIEW` | `"architecture-review"` |
| `STYLE_REVIEW` | `"style-review"` |
| `TEST_COVERAGE` | `"test-coverage"` |
| `PERFORMANCE_ANALYSIS` | `"performance-analysis"` |
| `CI_RUNNER` | `"ci-runner"` |
| `MERGE_EXECUTOR` | `"merge-executor"` |
| `HUMAN_APPROVAL_GATE` | `"human-approval-gate"` |
| `NOTIFY` | `"notify"` |
| `BATCH_BISECT` | `"batch-bisect"` |
| `COORDINATED_MERGE` | `"coordinated-merge"` |
| `COORDINATED_ROLLBACK` | `"coordinated-rollback"` |

Final class, private constructor. No Quarkus or foundation dependencies.

### `DevtownTrustDimension`

Constants class. Values are the dimension labels passed to `LedgerAttestation.trustDimension` for continuous quality scoring (separate from binary verdict attestations).

| Constant | Value | Measures |
|----------|-------|---------|
| `REVIEW_THOROUGHNESS` | `"review-thoroughness"` | Does the agent find issues that later cause incidents? |
| `FALSE_POSITIVE_RATE` | `"false-positive-rate"` | Does the agent flag issues that turn out to be non-issues? |
| `SECURITY_SPECIALIST` | `"security-specialist"` | Track record on security-sensitive reviews specifically |
| `SCOPE_AWARENESS` | `"scope-awareness"` | Does the agent correctly identify when it should DECLINE vs attempt? |

Final class, private constructor. No Quarkus or foundation dependencies.

### `CapabilityRegistry` SPI

Located in `io.casehub.devtown.domain.spi`. Pure Java interface — no CDI annotations.

```java
public interface CapabilityRegistry {
    Set<String> capabilities();
    OptionalDouble threshold(String capability);
    boolean isKnown(String capability);
}
```

`threshold()` returns `OptionalDouble` because not all capabilities have a routing threshold (e.g. `notify` has no minimum trust requirement — any worker can deliver a notification). `isKnown()` allows validation code to reject unrecognised tags without a separate threshold lookup.

### `DevtownCapabilityRegistry`

Located in `io.casehub.devtown.domain`. The populated default implementation — plain Java, no CDI annotations.

Returns all 13 capabilities and their routing thresholds:

| Capability | Minimum trust | Rationale |
|-----------|--------------|-----------|
| `security-review` | 0.70 | Security mistakes reach production |
| `architecture-review` | 0.65 | Design mistakes are expensive to reverse |
| `merge-executor` | 0.80 | Merge is irreversible |
| `style-review` | 0.50 | Baseline — any competent agent |
| All others | (empty) | No minimum trust gate |

`capabilities()` returns an immutable set of all 13 tags. `threshold()` returns the threshold if one exists, `OptionalDouble.empty()` otherwise. `isKnown()` delegates to `capabilities().contains(capability)`.

## CDI wiring (devtown-app)

A thin `@ApplicationScoped` bean in `devtown-app` wraps `DevtownCapabilityRegistry`:

```java
@ApplicationScoped
public class CapabilityRegistryBean extends DevtownCapabilityRegistry {}
```

This is the only CDI-aware piece. Overriding: deploy a bean annotated `@ApplicationScoped @Alternative @Priority(1)` that implements `CapabilityRegistry` — the CDI container selects the highest-priority alternative.

Composite multi-contributor pattern deferred (devtown#18 tracks use cases and justify conditions).

## File layout

```
devtown-domain/src/main/java/io/casehub/devtown/
  domain/
    DevtownCapability.java
    DevtownTrustDimension.java
    DevtownCapabilityRegistry.java
  domain/spi/
    CapabilityRegistry.java

devtown-domain/src/test/java/io/casehub/devtown/domain/
  DevtownCapabilityTest.java
  DevtownTrustDimensionTest.java
  DevtownCapabilityRegistryTest.java

devtown-app/src/main/java/io/casehub/devtown/app/
  CapabilityRegistryBean.java

devtown-app/src/test/java/io/casehub/devtown/app/
  DevtownBootTest.java  ← extended to verify CDI discovery
```

## Testing strategy

| Test type | What | Location |
|-----------|------|----------|
| Correctness | All 13 constants non-null, non-blank, unique as a set | `DevtownCapabilityTest` |
| Correctness | All 4 trust dimension constants non-null, non-blank, unique | `DevtownTrustDimensionTest` |
| Correctness | Threshold values match spec exactly (security-review=0.70, architecture-review=0.65, style-review=0.50, merge-executor=0.80) | `DevtownCapabilityRegistryTest` |
| Correctness | `capabilities()` returns all 13 tags | `DevtownCapabilityRegistryTest` |
| Robustness | `threshold()` for unknown tag returns `OptionalDouble.empty()` | `DevtownCapabilityRegistryTest` |
| Robustness | `threshold(null)` throws `NullPointerException` with message `"capability must not be null"` | `DevtownCapabilityRegistryTest` |
| Robustness | `isKnown()` for unknown tag returns false | `DevtownCapabilityRegistryTest` |
| Happy path | `isKnown()` returns true for every constant in `DevtownCapability` | `DevtownCapabilityRegistryTest` |
| Happy path | All thresholds in `[0.0, 1.0]` range | `DevtownCapabilityRegistryTest` |
| Integration | `CapabilityRegistryBean` injectable as `CapabilityRegistry` in Quarkus CDI context | `DevtownBootTest` extended |

No E2E tests at this layer — capability routing is exercised end-to-end in Epic 3 when case plan models fire bindings.

## Null policy

`threshold(null)` — throw `NullPointerException` with message `"capability must not be null"`. Consistent with `Objects.requireNonNull` idiom used in the engine. Callers that reach routing with a null capability tag have a bug; fail fast.

## Out of scope

- No Quarkus CDI annotations in `devtown-domain`
- No persistence, no Flyway, no datasource configuration
- No capability metadata beyond tag string and routing threshold (input/output schemas belong in Epic 3 when `CasePlanModel` YAML is written)
- No trust dimension scoring logic — dimension *names* only; scoring is in `casehub-ledger`
