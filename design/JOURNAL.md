# Design Journal — issue-13-trust-weighted-routing

### Scope reduction — everything but the test was already shipped §9.4 Layer 6

The spec started as two-repo upstream-first architecture (TrustGatedAttestationPolicy in qhorus, RetroactiveAttestationService). Three review rounds established that the routing infrastructure (devtown#57), incident feedback (devtown#5), and positive attestation flow (qhorus LedgerWriteService) were all already shipped. The epic reduced to: MCP tool + E2E closed-loop test + issue filing.

### TrustGatedAttestationPolicy deferred — CommitmentContext lacks capabilityTag §9.4 Layer 6

The attestation policy design uses capabilityScore() not globalScore() for per-capability trust gating. But CommitmentContext doesn't carry capabilityTag — LedgerWriteService extracts it from the COMMAND JSON AFTER calling the policy. Filed qhorus#307 to move extraction before the policy call. TrustGatedAttestationPolicy (devtown#97) blocked on this.

### FLAGGED attestation feedback loop proven end-to-end §9.4 Layer 6

E2E test proves: SOUND attestations → TrustScoreJob materialises capability scores → TrustWeightedAgentStrategy routes to higher-trust agent → IncidentFeedbackService writes FLAGGED → TrustScoreJob recomputes lower score → routing shifts. The WEIGHTED_MAJORITY aggregation masking issue (ledger#157) was fixed upstream. JPA first-level cache stale reads required fresh transactions on the read side (GE-20260625-aaf3d4).
