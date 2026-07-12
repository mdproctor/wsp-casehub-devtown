# Design Journal — issue-141-evidential-checker-v1-v4

## §1 Design Decisions — 2026-07-12

### Composition strategy: delegation with independent classification

EvidentialAttestationPolicy wraps TrustGatedAttestationPolicy rather than replacing it. Classification uses TrustRoutingPolicy's public API (~12 lines) — same methods as TrustCandidateClassifier but without routing-specific data (AgentCandidate, workloadScore). Three approaches considered; delegation chosen over refactoring the base class or full replacement.

### Evidential check scope: configurable per capability

New `Set<TrustPhase> evidentialCheckPhases` field on `TrustRoutingPolicy` in engine-api. Empty set = no checks (backward compatible). Per-capability configuration in DevtownTrustRoutingPolicyProvider follows the existing risk-proportional pattern — security-review checks BOOTSTRAP+BELOW_THRESHOLD+QUALITY_FAILED, style-review checks nothing.

### Failure consequence: fixed high confidence

Evidential violations are structural proof (zero messages on channel, confirming a FAILED obligation) — not probabilistic assessments. FLAGGED at 0.8 confidence regardless of trust score. The trust score determines whether checks run; once they prove the claim invalid, confidence reflects evidence quality.

### Foundation change: TrustPhase enum in engine-api

New policy-level vocabulary distinct from TrustCandidateClassifier.Phase (routing-specific). Five values: BOOTSTRAP, QUALIFIED, BORDERLINE, BELOW_THRESHOLD, QUALITY_FAILED. Filed as engine#711.

### Cross-repo roadmap

Created ROADMAP.md in workspace — phased delivery plan mapping foundation issues to devtown features. Four phases: Trust Intelligence → Merge Queue → Agent Coordination → Resilience. Key finding: blocks-ui#41 (devtown governance workbench migration) is the longest critical path, gating all UI work across all phases. GitHub Project created at casehubio/projects/3.
