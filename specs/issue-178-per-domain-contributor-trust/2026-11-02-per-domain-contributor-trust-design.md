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

These are registered in `DevtownCapabilityRegistry.ALL_CAPABILITIES` alongside the existing aggregate. No routing policies are assigned to compound capabilities — they are contributor trust tags, not agent routing capabilities.

### Domain source

Domains are derived from the `FINDINGS_CAPABILITIES` review bindings the PR case actually activated. This uses ground truth (which review process ran) rather than prediction (what file paths suggest).

`PrLifecycleEvent` remains unchanged — it is a sealed interface and `TriageRejected` has no review domains (no case exists yet).

### Attestation production

`PrLifecycleAttestationObserver` is extended:

1. For events with `caseId` (Merged, Rejected, ChangesRequested): look up which `FINDINGS_CAPABILITIES` bindings were activated on the case
2. For each activated domain: produce a domain-scoped attestation tagged with `ContributorTrustCapability.forDomain(domain)`
3. The existing aggregate `pr-contribution` attestation continues unchanged
4. `TriageRejected` (no caseId): produces aggregate-only attestations — no domain signal exists before case creation

A Merged PR that triggered security-review and architecture-review produces **three** attestations:
- `pr-contribution` (aggregate — same as today)
- `pr-contribution:security-review`
- `pr-contribution:architecture-review`

Each domain attestation carries the same verdict, confidence, and dimension scores as the aggregate. The domain is distinguished by the capability tag, not the dimensions. Dimension scores (`merge-rate`, `first-attempt-quality`) apply orthogonally — `capabilityDimensionScore("alice", "pr-contribution:security-review", "merge-rate")` gives "merge rate in the security domain".

### Deterministic entry IDs

Domain-scoped attestations extend the existing deterministic ID scheme:

```java
CONTRIBUTOR_OUTCOME_NS + "|" + repo + ":" + prNumber + ":" + outcome + ":" + domain
```

The aggregate attestation retains the existing format (no domain suffix). Same PR/outcome/domain always produces the same UUID.

### Coexistence and transition

The aggregate `pr-contribution` and per-domain capabilities coexist. They are non-additive: `capabilityScore("alice", "pr-contribution")` and `capabilityScore("alice", "pr-contribution:security-review")` are independent ledger queries with different capability keys.

Per-domain scores are preferred over the aggregate when that domain's observation count meets a minimum threshold:

| Domain | Minimum observations |
|--------|---------------------|
| security-review | 10 |
| architecture-review | 8 |
| style-review | 5 |
| test-coverage | 5 |
| performance-analysis | 5 |

Below the threshold, consumers fall back to the aggregate score.

### Auto-merged PRs

Auto-merged PRs do not produce per-domain attestations (no review bindings fire). Contributors who earn auto-merge have already demonstrated sufficient quality through reviewed PRs. The aggregate score covers them. Per-domain scoring is most valuable during trust-building, not after trust is established.

### Bootstrap

`BootstrapScoreComputer` is unchanged. GitHub profile data has no domain granularity. Bootstrap seeds the aggregate `pr-contribution` only. Per-domain scoring builds from actual reviewed PRs, starting from zero observations.

## Components changed

| Component | Module | Change |
|-----------|--------|--------|
| `ContributorTrustCapability` | domain | Add `forDomain(String)`, `isDomainScoped(String)` |
| `DevtownCapabilityRegistry` | domain | Register compound capabilities from `FINDINGS_CAPABILITIES` |
| `PrLifecycleAttestationObserver` | app | Look up activated bindings, produce per-domain attestations |
| `ContributorAttestationPolicy` | app | No change — aggregate attestation logic unchanged |
| `GovernanceQueryService` | app | Expose per-domain scores in contributor fleet/detail |
| `BootstrapScoreComputer` | domain | No change — aggregate-only bootstrap |
| `PrLifecycleEvent` | review | No change — sealed interface unchanged |
| Foundation (ledger, engine) | — | No change — compound strings are just strings |

## Testing

### Unit tests

- `ContributorTrustCapabilityTest` — `forDomain()` produces correct compound tags, rejects unknown domains (including `code-analysis`), `isDomainScoped()` classification
- `ContributorAttestationPolicyTest` — existing tests unchanged; aggregate attestation semantics preserved
- `DevtownCapabilityRegistryTest` — compound capabilities registered, `isKnown()` returns true for all domain-scoped tags

### Integration tests

- `PrLifecycleAttestationObserverTest` — Merged event with 2 activated review domains produces 3 attestations (1 aggregate + 2 domain-scoped); TriageRejected produces 1 aggregate-only; ChangesRequested with 1 domain produces 2 attestations
- Entry ID determinism — same PR/outcome/domain produces same UUID; different domains produce different UUIDs; aggregate ID unchanged from current behaviour
- `GovernanceQueryService` — contributor detail returns per-domain scores when available, shows observation counts per domain, falls back to aggregate when below threshold

### Not in scope

- Ledger scoring internals — foundation concern
- Review binding activation logic — existing CasePlanModel tests
- Frontend per-domain display — separate issue

## References

- `domain/src/main/java/io/casehub/devtown/domain/ContributorTrustCapability.java` — current single capability
- `domain/src/main/java/io/casehub/devtown/domain/ContributorTrustDimension.java` — quality dimensions
- `domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java` — FINDINGS_CAPABILITIES subset
- `domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java` — capability registration
- `app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java` — attestation production
- `review/src/main/java/io/casehub/devtown/review/PrLifecycleEvent.java` — sealed event interface
- `domain/src/main/java/io/casehub/devtown/domain/trust/BootstrapScoreComputer.java` — bootstrap scoring
- `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java` — trust query API
- `../parent/docs/platform/routing.md` — four-layer routing architecture, score computation
- `decisions.md` — D1 through D6 with rationale and trade-offs
