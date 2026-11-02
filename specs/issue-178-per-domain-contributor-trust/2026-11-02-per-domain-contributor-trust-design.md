# Per-Domain Contributor Trust Splitting

**Issue:** casehubio/devtown#178
**Date:** 2026-11-02
**Status:** Design approved

## Problem

The contributor trust model scores all PR contributions under a single `pr-contribution` capability with two quality dimensions (`merge-rate`, `first-attempt-quality`). A contributor who consistently produces poor security-related PRs but excellent docs PRs gets a blended score that obscures domain-specific quality differences.

Per-domain splitting allows the system to score contributors differently based on which review domains they contribute to — enabling domain-specific routing and intake decisions.

## Prerequisites

Issue #178 identifies preconditions: sufficient observation counts per domain, evidence that contributors show different quality across domains, and 6+ months of single-capability model data. The design addresses these architecturally via staged activation (D3): aggregate scoring operates from day one, per-domain scoring activates only when a domain's observation count meets its minimum threshold.

## Design

### Capability naming convention

Compound capability tags: `pr-contribution:<review-domain>`.

`ContributorTrustCapability` gains a factory method:

```java
public static String forDomain(String reviewDomain) {
    if (!ReviewDomain.FINDINGS_CAPABILITIES.contains(reviewDomain)) {
        throw new IllegalArgumentException("Unknown review domain: " + reviewDomain);
    }
    return PR_CONTRIBUTION + ":" + reviewDomain;
}

public static boolean isDomainScoped(String capabilityTag) {
    return capabilityTag.startsWith(PR_CONTRIBUTION + ":");
}
```

Validates against `FINDINGS_CAPABILITIES` — the subset that provides discriminative signal. `code-analysis` is excluded because it fires for every PR and would be statistically identical to the aggregate.

The resulting capabilities:
- `pr-contribution:security-review`
- `pr-contribution:architecture-review`
- `pr-contribution:style-review`
- `pr-contribution:test-coverage`
- `pr-contribution:performance-analysis`

These are **not** registered in `DevtownCapabilityRegistry.ALL_CAPABILITIES` — that set is for agent routing capabilities. Compound contributor trust tags are validated at construction by `forDomain()` against `ReviewDomain.FINDINGS_CAPABILITIES`.

### Domain source

Domains are derived from the `FINDINGS_CAPABILITIES` review bindings the PR case actually activated. This uses ground truth (which review process ran) rather than prediction (what file paths suggest).

`PrLifecycleEvent` is enriched with `Set<String> reviewDomains` on all subtypes:
- `Merged`, `Rejected`, `ChangesRequested` — populated by the caller with the set of `FINDINGS_CAPABILITIES` bindings that fired on the case
- `TriageRejected` — `Set.of()` (empty set, no case exists yet)

The caller already has case context when constructing the event; pushing domain resolution to event creation time keeps the attestation observer domain-agnostic.

### Attestation production

`ContributorAttestationPolicy` is parameterized to accept a capability tag:

```java
private AttestationIntent intent(UUID entryId, UUID subjectId,
                                  AttestationVerdict verdict, double confidence,
                                  String capabilityTag,
                                  Map<String, Double> dimensions, String evidence) {
    return new AttestationIntent(
            entryId, subjectId, verdict, confidence,
            capabilityTag,
            ATTESTOR_ID, ActorType.SYSTEM, ATTESTOR_ROLE,
            dimensions, evidence,
            CONTRIBUTOR_OUTCOME_NS, null);
}
```

The `evaluate()` method gains a `Set<String> reviewDomains` parameter. For each outcome, it produces:
1. The existing aggregate attestation tagged with `PR_CONTRIBUTION`
2. One domain-scoped attestation per activated domain, tagged with `forDomain(domain)`

If `reviewDomains` is empty (TriageRejected, auto-merged PRs, or any case with no `FINDINGS_CAPABILITIES` bindings), only the aggregate attestation is produced.

**All six outcomes explicitly:**

| Outcome | Has caseId | Per-domain attestations |
|---------|-----------|------------------------|
| `MERGED` | yes | One per activated FINDINGS_CAPABILITIES domain |
| `MERGED_WITH_REWORK` | yes | One per activated FINDINGS_CAPABILITIES domain |
| `CHANGES_REQUESTED` | yes | One per activated FINDINGS_CAPABILITIES domain |
| `REJECTED` | yes | One per activated FINDINGS_CAPABILITIES domain |
| `ABANDONED_AFTER_REVIEW` | yes | One per activated FINDINGS_CAPABILITIES domain |
| `TRIAGE_REJECTED` | no | None — aggregate only |

A Merged PR that triggered security-review and architecture-review produces **three** attestations:
- `pr-contribution` (aggregate — same as today)
- `pr-contribution:security-review`
- `pr-contribution:architecture-review`

**Same-verdict limitation:** Each domain attestation carries the same verdict, confidence, and dimension scores as the aggregate. This means a `CHANGES_REQUESTED` due to a security finding penalizes all activated domains equally. Per-domain differentiation occurs **across PRs** (a contributor whose security PRs get rejected but architecture PRs get merged), not **within** a single multi-domain PR. This is architecturally correct for the current event model — domain-specific verdicts would require review agents to tag which domain caused the outcome, which is a separate concern (not in scope).

### Deterministic entry IDs

Domain-scoped attestations extend the existing deterministic ID scheme:

```java
CONTRIBUTOR_OUTCOME_NS + "|" + repo + ":" + prNumber + ":" + outcome + ":" + domain
```

The aggregate attestation retains the existing format (no domain suffix). Same PR/outcome/domain always produces the same UUID.

### Coexistence and transition

The aggregate `pr-contribution` and per-domain capabilities coexist. They are non-additive: `capabilityScore("alice", "pr-contribution")` and `capabilityScore("alice", "pr-contribution:security-review")` are independent ledger queries with different capability keys.

Per-domain scores are preferred over the aggregate when that domain's observation count meets a minimum threshold. These thresholds are stored as `RoutingPolicy` entries on the compound capabilities in `DevtownCapabilityRegistry.POLICIES`:

| Domain | Minimum observations |
|--------|---------------------|
| security-review | 10 |
| architecture-review | 8 |
| style-review | 5 |
| test-coverage | 5 |
| performance-analysis | 5 |

Below the threshold, consumers fall back to the aggregate score.

### Consumer changes

Three existing consumers of contributor trust scores are affected:

**`GovernanceQueryService.contributorFleet()`** — currently shows aggregate `PR_CONTRIBUTION` score. Extended to include per-domain breakdown when available: a `Map<String, Double> domainScores` field on `ContributorFleetEntry` showing domain → score for domains that meet their minimum observation threshold. The aggregate score remains the primary display value.

**`GovernanceQueryService.contributorDetail()`** — currently shows `capabilityScores` map. Extended to include per-domain observation counts alongside scores. The `allCapabilityScores()` query from `TrustGateService` automatically includes compound capability scores — no new ledger queries needed.

**`ContributorIntakePolicy` (if exists)** — uses the aggregate `PR_CONTRIBUTION` score for intake classification. Per-domain scoring does not change intake logic. Intake is a coarse-grained gate; domain-specific differentiation applies at routing time, not intake time.

The domain-vs-aggregate fallback logic lives in a new `ContributorTrustResolver` helper in the `app` module:

```java
public OptionalDouble resolveScore(String contributorId, String domain) {
    Optional<RoutingPolicy> policy = capabilityRegistry.policy(
        ContributorTrustCapability.forDomain(domain));
    if (policy.isPresent()) {
        int observations = trustGateService.decisionCount(
            contributorId, ContributorTrustCapability.forDomain(domain));
        if (observations >= policy.get().minimumObservations().orElse(0)) {
            return trustGateService.capabilityScore(
                contributorId, ContributorTrustCapability.forDomain(domain));
        }
    }
    return trustGateService.capabilityScore(
        contributorId, ContributorTrustCapability.PR_CONTRIBUTION);
}
```

### Auto-merged PRs

Auto-merged PRs pass `Set.of()` (empty reviewDomains) — the policy produces aggregate-only attestations. This is handled by the empty-set check in `evaluate()`, not by implicit absence of data.

### Bootstrap

`BootstrapScoreComputer` is unchanged. GitHub profile data has no domain granularity. Bootstrap seeds the aggregate `pr-contribution` only. Per-domain scoring builds from actual reviewed PRs, starting from zero observations.

## Components changed

| Component | Module | Change |
|-----------|--------|--------|
| `ContributorTrustCapability` | domain | Add `forDomain(String)`, `isDomainScoped(String)` |
| `ContributorAttestationPolicy` | app | Parameterize `intent()` to accept capability tag; `evaluate()` gains `reviewDomains` parameter and produces per-domain attestations |
| `PrLifecycleEvent` | review | Add `Set<String> reviewDomains` to all subtypes (`Set.of()` for TriageRejected) |
| `PrLifecycleAttestationObserver` | app | Pass `event.reviewDomains()` to policy |
| `DevtownCapabilityRegistry` | domain | Add `RoutingPolicy` entries for compound capabilities (minimum observation thresholds) |
| `ContributorTrustResolver` | app | New helper — domain-vs-aggregate fallback logic |
| `GovernanceQueryService` | app | Expose per-domain scores and observation counts in contributor fleet/detail |
| `BootstrapScoreComputer` | domain | No change — aggregate-only bootstrap |
| Foundation (ledger, engine) | — | No change — compound strings are just strings |

## Testing

### Unit tests

- `ContributorTrustCapabilityTest` — `forDomain()` produces correct compound tags, rejects unknown domains (including `code-analysis`), `isDomainScoped()` classification
- `ContributorAttestationPolicyTest` — existing aggregate tests preserved; new tests: `evaluate()` with non-empty reviewDomains produces aggregate + per-domain attestations; empty reviewDomains produces aggregate only; each outcome × domain combination tested
- `ContributorTrustResolverTest` — falls back to aggregate when domain observation count below threshold; uses domain score when above threshold; handles missing domain score gracefully

### Integration tests

- `PrLifecycleAttestationObserverTest` — Merged event with 2 review domains produces 3 attestations (1 aggregate + 2 domain-scoped); TriageRejected produces 1 aggregate-only; MERGED_WITH_REWORK with 1 domain produces 2 attestations; ABANDONED_AFTER_REVIEW with domains produces per-domain attestations
- Entry ID determinism — same PR/outcome/domain produces same UUID; different domains produce different UUIDs; aggregate ID unchanged from current behaviour
- `GovernanceQueryService` — contributor detail returns per-domain scores when available, shows observation counts per domain; contributor fleet shows domain breakdown

### Not in scope

- Ledger scoring internals — foundation concern
- Review binding activation logic — existing CasePlanModel tests
- Domain-specific verdicts (which domain caused a ChangesRequested) — requires review agent outcome model changes
- Frontend per-domain display — file as separate issue during implementation

## References

- `domain/src/main/java/io/casehub/devtown/domain/ContributorTrustCapability.java` — current single capability
- `domain/src/main/java/io/casehub/devtown/domain/ContributorTrustDimension.java` — quality dimensions
- `domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java` — FINDINGS_CAPABILITIES subset
- `domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java` — capability registration and routing policies
- `app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java` — attestation production
- `app/src/main/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserver.java` — CDI event observer
- `review/src/main/java/io/casehub/devtown/review/PrLifecycleEvent.java` — sealed event interface
- `domain/src/main/java/io/casehub/devtown/domain/trust/BootstrapScoreComputer.java` — bootstrap scoring
- `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java` — trust query API
- `engine/common-core/src/main/java/io/casehub/engine/common/spi/PlanItemStore.java` — case plan item queries
- `../parent/docs/platform/routing.md` — four-layer routing architecture, score computation
- `decisions.md` — D1 through D6 with rationale and trade-offs
