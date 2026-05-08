# Epic 2: Domain Model Design Spec
2026-05-08 (revised after design review)

## Context

`devtown-domain` is the authoritative source of the software engineering domain vocabulary for casehub-devtown. Everything in Epics 3–10 depends on this vocabulary. No foundation dependency — pure Java.

This spec is the result of a systematic design review that challenged the original Gastown-derived vocabulary. The original 13 flat capability tags have been replaced with a typed, cohesive vocabulary that exploits the richer semantics of the CaseHub foundation.

---

## Platform Coherence

**Step 1 — Already exists?**
- `CapabilityTag.GLOBAL = "*"` in `casehub-ledger` is a sentinel — not a vocabulary. No overlap.
- `LedgerAttestation.capabilityTag` and `TrustGateService.meetsThreshold(actorId, capabilityTag, minTrust)` consume string tags — devtown's constants are the domain-specific values that flow into those slots.
- `ActorTrustScore.ScoreType.CAPABILITY` with `scope_key` and `ScoreType.DIMENSION` with `scope_key` — confirmed working; devtown's vocabulary values will flow directly into these.

**Step 2 — Right repo?** Yes. Software engineering domain concepts (PR review, security analysis, merge operations) belong in the application tier.

**Step 4 — Consistent with platform pattern?**
- `public static final String` constants — matches `CapabilityTag` in ledger.
- SPI in `domain.spi`, populated default in same module — matches engine's `NoOpWorkerProvisioner` pattern, refined: this is a **vocabulary/registry SPI** (not operational no-op), so the default is a populated implementation, not empty. PLATFORM.md updated to capture this distinction (parent#12).

**Known gap:** `ScoreType.CAPABILITY_DIMENSION` (composite per-capability quality score) does not yet exist in the ledger. Tracked: ledger#76, devtown#19. Designed to accommodate it when it ships — no rework needed.

---

## Design Review: What Changed From the Original Spec

The original spec had 13 flat capability tags copied from the Gastown vocabulary. After review:

| Original | Disposition | Reason |
|----------|------------|--------|
| `code-analysis` through `performance-analysis` (6) | → `ReviewDomain` | These are analytical work types — typed separately because trust scoring, routing semantics, and actor type (AI agent) are homogeneous |
| `ci-runner`, `merge-executor` (2) | → `AgentQualification` | Execution capabilities — trust scoring applies but semantics differ from analytical review |
| `human-approval-gate` | → `HumanDecision` (elevated, renamed) | Not a routing constraint — a formal accountability event with its own lifecycle, SLA, and trust model |
| `notify` | **Removed** | Connector call, not a trust-scored capability. `casehub-connectors` owns delivery; the case plan model calls it directly |
| `batch-bisect`, `coordinated-merge`, `coordinated-rollback` | **Removed** | Orchestration operations — expressed as CasePlanModel binding structures (Epics 4/5). Names also challenged: devtown#20 tracks them |
| *(new)* | → `HumanOversight` | Emerged from design: human called in when automated routing confidence is low — connects to `RoutingPolicy.borderlineMargin` and EU AI Act Art.12 |

---

## Types

### 1. `ReviewDomain`

What analytical work a PR needs. These strings also serve as `AgentQualification` values for the matching review type — an agent qualified for `security-review` handles PRs that need `security-review`. The string values flow into `LedgerAttestation.capabilityTag` and `TrustGateService`.

```java
public final class ReviewDomain {
    public static final String CODE_ANALYSIS        = "code-analysis";
    public static final String SECURITY_REVIEW      = "security-review";
    public static final String ARCHITECTURE_REVIEW  = "architecture-review";
    public static final String STYLE_REVIEW         = "style-review";
    public static final String TEST_COVERAGE        = "test-coverage";
    public static final String PERFORMANCE_ANALYSIS = "performance-analysis";
    private ReviewDomain() {}
}
```

### 2. `AgentQualification`

Execution capabilities — things agents are qualified to execute, with trust scoring applied to outcome quality. Distinct from review domains: these are operational, not analytical.

```java
public final class AgentQualification {
    public static final String CI_RUNNER      = "ci-runner";
    public static final String MERGE_EXECUTOR = "merge-executor";
    private AgentQualification() {}
}
```

### 3. `HumanDecision`

Formal accountability events requiring human judgment. Not a routing constraint — a first-class vocabulary type with dedicated actor type enforcement (`ActorType.HUMAN`), `casehub-work` WorkItem lifecycle (SLA, business hours, delegation, escalation), and trust accumulation on human actors.

Naming: "decision" rather than "gate" or "approval" — a gate is something you pass through; a decision is a formal act by a named accountable person. GDPR Art.22 oversight requirement is met structurally.

```java
public final class HumanDecision {
    public static final String PR_APPROVAL = "human-decision:pr-approval";
    private HumanDecision() {}
}
```

### 4. `HumanOversight`

Human called in because the automated routing model is uncertain. Not a domain decision — a system-level check. Triggers when:
- An agent's trust score is within `RoutingPolicy.borderlineMargin` of the threshold
- No agent meets the minimum threshold for a required capability (fleet gap)
- The trust model has too few observations (`minimumObservations` not reached — Beta prior not stabilised)
- Multiple consecutive DECLINEs suggest a fleet capability gap

This is EU AI Act Art.12 territory — human oversight when automated confidence is marginal. Gastown cannot provide this concept structurally because it has no trust model to be uncertain about.

```java
public final class HumanOversight {
    public static final String ROUTING_REVIEW = "human-oversight:routing-review";
    private HumanOversight() {}
}
```

### 5. `DevtownTrustDimension`

Quality dimension labels for `LedgerAttestation.trustDimension`. Three dimensions, all auto-computed from the normative layer.

```java
public final class DevtownTrustDimension {
    public static final String REVIEW_THOROUGHNESS  = "review-thoroughness";
    public static final String FALSE_POSITIVE_RATE  = "false-positive-rate";
    public static final String SCOPE_CALIBRATION    = "scope-calibration";
    private DevtownTrustDimension() {}
}
```

**Why three, not four:**
- `REVIEW_THOROUGHNESS` — recall: does the agent find issues that later cause incidents?
- `FALSE_POSITIVE_RATE` — precision: does the agent flag things that weren't real problems?
- `SCOPE_CALIBRATION` — normative: does the agent correctly DECLINE work outside its capability? Maps directly to the DECLINED commitment — only possible because CaseHub has formal speech acts. Gastown cannot measure this.

**Why not `security-specialist`:** per-capability quality is expressed by combining `capabilityTag="security-review"` with `trustDimension="review-thoroughness"` on a single `LedgerAttestation`. This is how the ledger is designed to work. A separate `security-specialist` dimension would duplicate `ScoreType.CAPABILITY`. Once ledger#76 ships, the composite `CAPABILITY_DIMENSION` score type makes per-capability quality a first-class ledger concept.

**Why not Gastown's `creativity`:** not auto-computable from normative layer events. If human attestors want to score it manually, the `DIMENSION` mechanism supports it without a declared constant.

---

## `RoutingPolicy`

A value object replacing the original hard-coded thresholds. Policy artifacts are configurable per deployment, overridable per binding in the CasePlanModel, and auditable as case facts.

```java
public record RoutingPolicy(
    OptionalDouble threshold,           // minimum trust to route to this capability
    OptionalInt minimumObservations,    // trust score requires N attestations to be credible
    OptionalDouble borderlineMargin,    // threshold ± margin → trigger HumanOversight
    Optional<String> fallbackType,      // what to route to if no agent meets threshold
    String rationale                    // why this threshold — captured for audit
) {
    public boolean isBorderline(double trustScore) {
        return borderlineMargin.isPresent() && threshold.isPresent()
            && trustScore >= threshold.getAsDouble()
            && trustScore < threshold.getAsDouble() + borderlineMargin.getAsDouble();
    }
}
```

**Field purposes:**
- `threshold` — the minimum. Below this, never route.
- `minimumObservations` — credibility gate. A score of 0.85 from 2 attestations is noise; from 50 it is signal. New agents route to lower-stakes work first.
- `borderlineMargin` — the uncertainty band. Agents within this margin above threshold get a `HumanOversight` flag before assignment. Connects `RoutingPolicy` to `HumanOversight` vocabulary.
- `fallbackType` — what to do when no agent qualifies: escalate to `HumanOversight`, route to a backup capability, hold. Policy-defined.
- `rationale` — why does security-review have a 0.70 threshold? Captured as a string for audit records.

---

## `CapabilityRegistry` SPI

Located in `io.casehub.devtown.domain.spi`. Pure Java — no CDI annotations.

```java
public interface CapabilityRegistry {
    Set<String> capabilities();
    Optional<RoutingPolicy> policy(String capability);
    boolean isKnown(String capability);
}
```

`policy()` returns `Optional<RoutingPolicy>` — richer than `OptionalDouble`. Not all capabilities have a routing policy (`code-analysis` has none — any agent capable of analysis is assigned). `isKnown()` allows validation without a separate lookup.

---

## `DevtownCapabilityRegistry`

The populated default implementation — plain Java, no CDI annotations — in `io.casehub.devtown.domain`.

Returns all capabilities across all four types, with routing policies for trust-sensitive ones:

| Capability | Type | Threshold | Min observations | Borderline margin | Rationale |
|-----------|------|-----------|-----------------|-------------------|-----------|
| `code-analysis` | ReviewDomain | — | — | — | Any capable agent |
| `security-review` | ReviewDomain | 0.70 | 10 | 0.05 | Security mistakes reach production; 10 obs = credible score |
| `architecture-review` | ReviewDomain | 0.65 | 8 | 0.05 | Design mistakes are expensive to reverse |
| `style-review` | ReviewDomain | 0.50 | 5 | — | Baseline; borderline not worth oversight overhead |
| `test-coverage` | ReviewDomain | — | — | — | Any capable agent |
| `performance-analysis` | ReviewDomain | — | — | — | Any capable agent |
| `ci-runner` | AgentQualification | — | — | — | Deterministic — pass/fail is self-evident |
| `merge-executor` | AgentQualification | 0.80 | 15 | 0.05 | Merge is irreversible; needs substantial history |
| `human-decision:pr-approval` | HumanDecision | — | — | — | Human actor routing; threshold not applicable |
| `human-oversight:routing-review` | HumanOversight | — | — | — | Triggered by borderlineMargin on other capabilities |

`capabilities()` returns an immutable set of all capability strings. `policy()` returns the `RoutingPolicy` for trust-sensitive capabilities, `Optional.empty()` for others. `isKnown()` delegates to `capabilities().contains(capability)`.

`policy(null)` throws `NullPointerException("capability must not be null")`.

---

## File Layout

```
devtown-domain/src/main/java/io/casehub/devtown/
  domain/
    ReviewDomain.java
    AgentQualification.java
    HumanDecision.java
    HumanOversight.java
    DevtownTrustDimension.java
    RoutingPolicy.java
    DevtownCapabilityRegistry.java
  domain/spi/
    CapabilityRegistry.java

devtown-domain/src/test/java/io/casehub/devtown/domain/
  ReviewDomainTest.java
  AgentQualificationTest.java
  HumanDecisionTest.java
  HumanOversightTest.java
  DevtownTrustDimensionTest.java
  RoutingPolicyTest.java
  DevtownCapabilityRegistryTest.java

devtown-app/src/main/java/io/casehub/devtown/app/
  CapabilityRegistryBean.java     @ApplicationScoped, extends DevtownCapabilityRegistry

devtown-app/src/test/java/io/casehub/devtown/app/
  DevtownBootTest.java            extended: verify CapabilityRegistry CDI discovery
```

---

## Testing Strategy

### Constants (correctness)

| Test class | What | Type |
|-----------|------|------|
| `ReviewDomainTest` | All 6 constants non-null, non-blank, unique, no duplicates across other types | Correctness |
| `AgentQualificationTest` | Both constants non-null, non-blank, unique, no overlap with ReviewDomain | Correctness |
| `HumanDecisionTest` | Constant non-null, non-blank, prefixed `human-decision:` | Correctness |
| `HumanOversightTest` | Constant non-null, non-blank, prefixed `human-oversight:` | Correctness |
| `DevtownTrustDimensionTest` | All 3 constants non-null, non-blank, unique | Correctness |

### `RoutingPolicy` (correctness + robustness)

| Test | Type |
|------|------|
| `isBorderline()` returns true when score is within threshold + margin | Correctness |
| `isBorderline()` returns false when score is below threshold | Robustness |
| `isBorderline()` returns false when margin is empty | Robustness |
| `isBorderline()` returns false when score is above threshold + margin | Correctness |
| Record equality — two policies with same fields are equal | Correctness |

### `DevtownCapabilityRegistry` (happy path + robustness)

| Test | Type |
|------|------|
| `capabilities()` contains all 10 capability strings | Correctness |
| `capabilities()` returns immutable set | Robustness |
| `policy("security-review")` threshold = 0.70 | Correctness |
| `policy("security-review")` minimumObservations = 10 | Correctness |
| `policy("security-review")` borderlineMargin = 0.05 | Correctness |
| `policy("architecture-review")` threshold = 0.65 | Correctness |
| `policy("merge-executor")` threshold = 0.80, minimumObservations = 15 | Correctness |
| `policy("code-analysis")` returns `Optional.empty()` | Correctness |
| `policy("ci-runner")` returns `Optional.empty()` | Correctness |
| All threshold values in `[0.0, 1.0]` | Correctness |
| `isKnown("security-review")` returns true | Happy path |
| `isKnown("unknown")` returns false | Robustness |
| `policy(null)` throws `NullPointerException` | Robustness |
| `isKnown(null)` throws `NullPointerException` | Robustness |
| `isBorderline()` on security-review policy with borderline score | Happy path |

### Integration

| Test | Type |
|------|------|
| `CapabilityRegistryBean` injectable as `CapabilityRegistry` in Quarkus CDI context | Integration |
| `CapabilityRegistryBean.capabilities()` returns full set in live context | Integration |

---

## Null Policy

`policy(null)` and `isKnown(null)` throw `NullPointerException("capability must not be null")`. Callers that reach routing with a null capability tag have a bug — fail fast.

---

## Ledger Dependency Note

Per-capability quality dimensions (e.g., security-review thoroughness specifically, not global thoroughness) require `ScoreType.CAPABILITY_DIMENSION` in the ledger (ledger#76, devtown#19). `DevtownCapabilityRegistry` and `RoutingPolicy` are designed to accommodate this when it ships — `policy()` can be extended to include per-capability dimension floors without changing the SPI contract.

---

## Out of Scope

- No Quarkus CDI annotations in `devtown-domain`
- No persistence, Flyway, or datasource configuration
- No capability metadata beyond tag string and routing policy (input/output schemas belong in Epic 3 CasePlanModel YAML)
- No trust dimension scoring logic — dimension *names* only; scoring is in `casehub-ledger`
- `BATCH_BISECT`, `COORDINATED_MERGE`, `COORDINATED_ROLLBACK` deferred to Epics 4/5 with naming review (devtown#20)
- `NOTIFY` removed — `casehub-connectors` handles notification delivery directly
