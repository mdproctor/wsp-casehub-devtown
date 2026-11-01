# HANDOFF — devtown

## Last Session

Implemented three issues from epic #289 (Wire new foundation SPIs):

- **#292 — Wire TriageHandler SPI.** `ReviewTriageHandler` in `review/` module, `@ApplicationScoped @Priority(1)`. Classifies DECLINED → Retry (engine re-routes), unrecoverable → Abandon, transient/unknown → Retry while budget remains. 14 unit tests + 2 CDI displacement integration tests. Displaces both `NoOpTriageHandler` and `LlmTriageHandler`.

- **#293 — Wire PlaybookWorkerFunctionProvider.** CDI producer in `app/spi/PlaybookProviderProducer.java` registers `PlaybookWorkerFunctionProvider` as a bean. Style-review playbook YAML created in `review/src/main/resources/devtown/playbooks/style-review.yaml` (3-step agent-session flow). 3 compilation tests + 1 CDI wiring test. Contributor guide updated with both triage and playbook SPI documentation.

- **#294 — Wire SubCaseGroup bindingName.** Engine already populates `SubCaseGroup.bindingName` from binding name during sub-case creation. Added `BisectionSubCase` record to `BatchStatus` and query from `SubCaseGroupRepository` in `GovernanceQueryService.batchStatus()`. Tests updated.

## Immediate Next Step

`work continue` on branch `issue-289-wire-foundation-spis`. Start with **#295 — Evaluate HTTP agent sidecar for Claudony invocation**. This is a research/evaluation task — read platform #666 SPI contract and compare with current Claudony wiring. May close as rejected-with-ADR.

## Queue — Epic #289

| # | Title | Scale | Complexity | Status |
|---|-------|-------|------------|--------|
| #292 | Wire TriageHandler SPI | M | Med | Done |
| #293 | Wire PlaybookWorkerFunctionProvider | M | Med | Done |
| #294 | Wire SubCaseGroup bindingName | S | Low | Done |
| #295 | Evaluate HTTP agent sidecar for Claudony invocation | S | Med | Next |
| #296 | Evaluate protocol lifecycle hooks for reviewer interaction state | S | Med | Queued |

## References

- Epic: casehubio/devtown#289
- Branch: `issue-289-wire-foundation-spis`
- Commits: `9c7885a`, `ad04c85`, `1bdacea`, `c799173`
