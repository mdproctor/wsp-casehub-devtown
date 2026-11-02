# Decisions — Per-Domain Contributor Trust Splitting (#178)

## Prerequisites

Issue #178 states preconditions for per-domain scoring: sufficient observation counts per domain, evidence that contributors show different quality across domains, and 6+ months of single-capability model data. These are implementation-time prerequisites, not design constraints. D3's staged activation design addresses them architecturally: aggregate scoring operates from day one, per-domain scoring activates per-domain only when that domain's observation count meets its minimum threshold. Implementation should validate observation counts before enabling per-domain scoring for each domain.

## D1: Domain source for contributor trust

**Choice:** Derive domains from the `FINDINGS_CAPABILITIES` subset of ReviewDomain capabilities the PR review case actually bound (security-review, architecture-review, style-review, test-coverage, performance-analysis). `code-analysis` is excluded — it fires for every PR and adds no discriminative signal.
**Alternatives:**
- File-path heuristics — uses code analysis predictions rather than ground truth of which review process was applied; accuracy depends on heuristic quality
- Explicit contributor declaration — adds friction and can be gamed
- Code content analysis — accurate but adds complexity and latency
- Full REVIEW_CAPABILITIES set — includes code-analysis which fires for every PR, making that domain statistically identical to the aggregate
**Rationale:** The review case already classifies PR content by binding domain-specific review capabilities. Reusing that signal avoids duplication and uses ground truth (which bindings actually fired) rather than predictions (what code analysis flagged). `FINDINGS_CAPABILITIES` is already used in production for this kind of domain discrimination (`DescriptorDrivenReviewerAgentProducer`, `PrReviewCaseHub`).
**Trade-offs:** Domains are limited to what the review case recognizes. PRs that skip review (e.g. auto-merged) won't produce domain-scoped attestations — the aggregate score covers these (see D3, D6).
**Sources:** ReviewDomain.java (domain/src/main/java — FINDINGS_CAPABILITIES), DevtownCapabilityRegistry.java, routing.md (platform docs), DescriptorDrivenReviewerAgentProducer.java:34, PrReviewCaseHub.java:55
**Exploration:** quick
**Status:** revised — R1-01: clarified FINDINGS_CAPABILITIES (excludes code-analysis) as domain source instead of ambiguous "ReviewDomain capabilities"

## D2: Multi-domain attestation strategy

**Choice:** One attestation per domain — a MERGED PR that triggered security-review and architecture-review produces separate attestations for each domain.
**Alternatives:**
- Single attestation for primary domain — simpler but loses signal from secondary domains
- Single attestation with weighted split — complex and mixes concerns
- Fractional confidence split (confidence: 1/N per domain) — misuses confidence field, which represents certainty in the attestation verdict, not a portfolio weight
**Rationale:** Independent accumulation per domain preserves the Bayesian Beta model's per-capability semantics. Each domain builds its own independent (α, β) posterior. A cross-domain contributor's security-domain score converges at the same rate as a security-only specialist — both get one observation per security-domain PR. The aggregate `pr-contribution` (D3) serves cross-domain queries with one attestation per PR.
**Trade-offs:** More attestation entries in the ledger per PR. Each domain's observation count is independent — no cross-domain accumulation bias because `TrustScoreSource.capabilityScore()` queries per-capability independently.
**Sources:** ContributorAttestationPolicy.java, AttestationIntent (confidence field semantics), TrustScoreSource API (per-capability queries), routing.md Layer 1
**Exploration:** quick
**Status:** captured

## D3: Coexistence with aggregate capability

**Choice:** Keep the aggregate `pr-contribution` alongside per-domain capabilities. Bootstrap phase uses the aggregate; per-domain scoring activates per-domain only when that domain's observation count meets a minimum threshold.
**Alternatives:**
- Replace entirely — all scoring per-domain from day one; risk of thin data in rarely-triggered domains
**Rationale:** The aggregate provides a reliable fallback when per-domain observations are sparse. The two levels are non-additive: `capabilityScore("alice", "pr-contribution")` and `capabilityScore("alice", "pr-contribution:security-review")` are independent ledger queries with different capability keys — they cannot be accidentally summed. Consumers query whichever granularity has sufficient data.
**Transition criteria:** Each per-domain capability requires a minimum observation count before it is preferred over the aggregate. The implementation defines thresholds per domain, following the pattern in `DevtownCapabilityRegistry.POLICIES` (security-review: 10 minimumObservations, architecture-review: 8, style-review: 5). Below the threshold, consumers fall back to the aggregate score.
**Trade-offs:** Two levels of scoring to maintain. Consumers must decide which level to query — the minimum observation threshold provides a clear decision rule.
**Sources:** BootstrapScoreComputer.java, ContributorTrustCapability.java, DevtownCapabilityRegistry.java (POLICIES/minimumObservations pattern), issue #178 body ("Bayesian Beta needs sufficient observations per capability")
**Exploration:** quick
**Status:** revised — R1-03: added explicit transition criteria (minimum observation threshold per domain), clarified non-additive relationship between aggregate and per-domain scores, documented staged activation semantics

## D4: Data flow — domain info into attestation

**Choice:** Enrich at the `PrLifecycleAttestationObserver` level. The observer pattern-matches the event subtype; for subtypes that carry `caseId` (Merged, Rejected, ChangesRequested), it looks up which FINDINGS_CAPABILITIES bindings were activated from the case context. For `TriageRejected` (no caseId, no case, no review), the observer produces aggregate-only attestations. PrLifecycleEvent remains unchanged.
**Alternatives:**
- Enrich PrLifecycleEvent with `Set<String> reviewDomains` — forces a semantically invalid field onto TriageRejected (no review domains exist before case creation)
- Query the case at attestation time from the policy — adds engine runtime dependency at policy level
- Separate event per domain — cleaner separation but more events and event types
**Rationale:** The observer already has full CDI context (`@ApplicationScoped`, injected dependencies) and already pattern-matches event subtypes. `ContributorOutcomeLedgerWriter.extractCaseId()` performs the same subtype dispatch. The case context is accessible from the observer for Merged/Rejected/ChangesRequested. TriageRejected continues producing aggregate-only attestations with `FLAGGED`/`confidence:0.3`, preserving the trust signal for low-quality submissions.
**Trade-offs:** The observer becomes domain-aware (it needs to look up binding activation from the case). This is a responsibility expansion, but the observer is already the translation layer between lifecycle events and attestation intents.
**Depends on:** D1 (domain source)
**Sources:** PrLifecycleEvent.java (sealed interface, TriageRejected has no caseId), PrLifecycleAttestationObserver.java (subtype pattern matching), ContributorOutcomeLedgerWriter.java:87-91 (extractCaseId pattern), ContributorAttestationPolicy.java:56-59 (TRIAGE_REJECTED attestation)
**Exploration:** quick
**Status:** revised — R1-04: changed from PrLifecycleEvent enrichment to observer-level enrichment; PrLifecycleEvent sealed interface unchanged; TriageRejected produces aggregate-only attestations

## D5: Capability name modeling

**Choice:** Compound tag convention — `pr-contribution:<review-domain>` strings. ContributorTrustCapability gets a `forDomain(String)` factory method deriving from ReviewDomain constants.
**Alternatives:**
- Separate flat constants (PR_DOCS, PR_SECURITY) — discoverable but duplicates ReviewDomain's list
- Evidence metadata — domain in metadata, not capability key; doesn't support per-domain scoring
- CAPABILITY_DIMENSION axis (dimensionKey for domain) — overloads dimensionKey which is used for quality metrics (merge-rate, first-attempt-quality); mixing domain classification and quality measurement on the same axis prevents orthogonal queries
**Rationale:** The ledger keys scores by `capabilityName` — compound strings are just strings, no foundation changes. The convention is self-documenting and queryable via prefix match. The factory method avoids duplicating ReviewDomain's constant list. The compound format preserves the dimension axis for quality metrics: `capabilityDimensionScore("alice", "pr-contribution:security-review", "merge-rate")` gives "merge rate in the security domain" — correct orthogonality of (domain capability) × (quality dimension). Agent routing capabilities (`security-review`) and contributor trust capabilities (`pr-contribution:security-review`) are lexically distinguishable by the colon-separated prefix.
**Trade-offs:** Convention-based — no compile-time enforcement that the domain part is valid. Mitigated by the factory method validating against ReviewDomain.FINDINGS_CAPABILITIES. Implementation must register compound capabilities in `DevtownCapabilityRegistry.ALL_CAPABILITIES`.
**Depends on:** D1 (domain source), D3 (coexistence)
**Sources:** ContributorTrustCapability.java, ReviewDomain.java (FINDINGS_CAPABILITIES), DevtownCapabilityRegistry.java, ScoreType.CAPABILITY_DIMENSION / ContributorTrustDimension.java (existing quality dimension usage)
**Exploration:** quick
**Status:** captured

## D6: Auto-merged PR trust gap

**Choice:** Auto-merged PRs do not produce per-domain attestations. Highly trusted contributors with sparse per-domain data fall back to their aggregate score (D3).
**Alternatives:**
- Synthetic domain attestations for auto-merged PRs — fabricates domain signal without actual domain-specific review
- Require review for all PRs to ensure domain coverage — defeats the purpose of trust-based auto-merge
- Use code analysis output to derive synthetic domain signal — uses prediction rather than ground truth, inconsistent with D1
**Rationale:** Auto-merge is a system-controlled decision based on existing trust — contributors cannot choose it, so there is no perverse incentive. Contributors who earn auto-merge have already demonstrated sufficient quality to bypass domain-specific review. Per-domain scoring is most valuable for contributors still going through review, where the system needs domain-specific differentiation for routing and intake decisions. The aggregate score provides a reliable fallback for established contributors.
**Trade-offs:** The system has more domain-specific knowledge about newer contributors than established ones. This is architecturally correct: you need domain-specific scrutiny during trust-building, not after trust is established. However, if a highly trusted contributor's quality degrades in a specific domain, the per-domain model may not detect it until enough non-auto-merged PRs accumulate.
**Depends on:** D1 (domain source), D3 (aggregate fallback)
**Sources:** PrLifecycleAttestationObserver.java, ContributorAttestationPolicy.java, contributor-trust-proposal.md (queue routing by trust)
**Exploration:** quick — surfaced by R1-07 review challenge
**Status:** captured
