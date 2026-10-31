## D1: Test layout strategy

**Choice:** Extend existing test classes + one new class for WorkerDecisionEvent
**Alternatives:**
- Single comprehensive test class — cleaner isolation but duplicates event-firing setup and adds CDI boot
- Per-concern split (3 new classes) — maximum isolation but three additional CDI boots and significant duplication
**Rationale:** Reuses existing infrastructure (MergeDecisionObserverTest already fires CaseLifecycleEvent, GdprErasureResourceTest already has erasure setup), minimises CDI boot overhead, keeps related assertions together. Only WorkerDecisionEvent genuinely needs a new class.
**Trade-offs:** Existing tests grow larger; if they fail, the failure surface is wider
**Sources:** MergeDecisionObserverTest.java, GdprErasureResourceTest.java, LedgerEnabledTestProfile.java
**Exploration:** quick
**Status:** captured

## D2: Trust dependency strategy for WorkerDecisionEvent test

**Choice:** Use real trust beans via TrustScoringTestProfile
**Alternatives:**
- Stub TrustScoreSource and TrustRoutingPolicyProvider — adds indirection for no benefit since beans are already available
**Rationale:** TrustScoringTestProfile already enables both ledger and trust scoring. WorkerDecisionEventCapture uses TrustScoreSource to populate trust fields — testing the actual production write path is the whole point of #288. No extra setup needed.
**Trade-offs:** Test is coupled to trust infrastructure — if trust beans break, this test breaks too (acceptable: that's a real production failure mode)
**Sources:** TrustScoringTestProfile.java, WorkerDecisionEventCapture (engine-ledger-core)
**Exploration:** quick
**Status:** captured
