# HANDOFF — devtown

## Last Session

Closed #288 (ledger integration testing — verify write path in devtown context). Extended `MergeDecisionObserverTest` and `GdprErasureResourceTest`, created new `WorkerDecisionWritePathTest`. All 11 ledger tests pass. Blog entry written: "The Write Path Nobody Tested."

Decomposed #289 into epic with 5 child issues (#292–#296). Branch `issue-289-wire-foundation-spis` scaffolded with .plan and queue populated.

## Immediate Next Step

`work continue` on branch `issue-289-wire-foundation-spis`. Start with **#292 — Wire TriageHandler SPI**. This replaces `NoOpTriageHandler` with domain-specific review failure triage. Check platform #673 for the SPI contract before designing.

## Queue — Epic #289

| # | Title | Scale | Complexity | Priority |
|---|-------|-------|------------|----------|
| #292 | Wire TriageHandler SPI — domain-specific review failure triage | M | Med | High |
| #293 | Wire PlaybookWorkerFunctionProvider — YAML-defined reviewer agent behavior | M | Med | High |
| #294 | Wire SubCaseGroup bindingName — richer merge queue bisection tracking | S | Low | Medium |
| #295 | Evaluate HTTP agent sidecar for Claudony invocation | S | Med | Low |
| #296 | Evaluate protocol lifecycle hooks for reviewer interaction state | S | Med | Low |

## Recommended Order

1. **#292 TriageHandler** — highest production impact, replaces NoOp default
2. **#293 PlaybookWorkerFunctionProvider** — proof of concept, one capability
3. **#294 SubCaseGroup bindingName** — small, straightforward
4. **#295 HTTP agent sidecar** — evaluation, may close as rejected-with-ADR
5. **#296 Protocol lifecycle hooks** — evaluation, may close as rejected-with-ADR

## References

- Epic: casehubio/devtown#289
- Landed: #288 as `5f872e3` on main
- Blog: `blog/2026-10-31-mdp02-the-write-path-nobody-tested.md`
