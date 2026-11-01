# HANDOFF — devtown

## Last Session

Closed epic #289 (Wire new foundation SPIs) — all 5 child issues complete:

- **#292 — Wire TriageHandler SPI.** `ReviewTriageHandler` in `review/`, `@ApplicationScoped @Priority(1)`. Classifies DECLINED → Retry, unrecoverable → Abandon, transient → Retry while budget remains. Displaces both `NoOpTriageHandler` and `LlmTriageHandler`.

- **#293 — Wire PlaybookWorkerFunctionProvider.** CDI producer in `app/spi/PlaybookProviderProducer.java`. Style-review playbook YAML as proof of concept. Contributor guide updated.

- **#294 — Wire SubCaseGroup bindingName.** `BisectionSubCase` record in `BatchStatus`. Governance query joins `SubCaseGroupRepository` to surface bisection role.

- **#295 — Evaluate HTTP agent sidecar.** Rejected — ADR-0002. Devtown invokes `AgentProvider` directly in-JVM; HTTP endpoint adds overhead for zero gain.

- **#296 — Evaluate protocol lifecycle hooks.** Rejected — ADR-0003. Review state belongs in CasePlanModel context, not qhorus channel-level hooks.

Epic #289 closed. Blog entry revised: "The SPIs That Were Already There."

## Immediate Next Step

**#285 — Replace NoOp clients with real GitHub implementations.** Highest-value concrete issue remaining. Replaces `NoOpMergeClient`, `NoOpCiStatusClient`, `NoOpBatchBranchClient`, `NoOpRevertClient` with real GitHub implementations. Same CDI displacement pattern used throughout this epic.

## Open Issues

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|------------|-------|
| #285 | Replace NoOp clients with real GitHub implementations | L | Med | Recommended next |
| #224 | Epic: Intelligence workbench — wire 20+ blocks-ui | XL | Med | Needs decomposition |
| #208 | Epic: Production readiness — close gaps | XL | High | Needs decomposition |
| #200 | Engine GOAP / LLM Decomposition | L | High | Blocked on engine GOAP |
| #178 | Per-domain contributor trust splitting | M | Med | Idea |
| #177 | Federated contributor trust | L | High | Blocked on P2.1 |

## References

- Landed: epic #289 as `fd3eef2` + `d741375` on main
- ADRs: `docs/adr/0002-reject-http-agent-sidecar.md`, `docs/adr/0003-reject-protocol-lifecycle-hooks.md`
- Blog: `blog/2026-11-01-mdp01-the-spis-that-were-already-there.md`
