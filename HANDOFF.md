# Handoff — devtown slot 208

**Date:** 2026-10-02
**Branch:** main (work landed)
**Last commit:** 763e95d fix: SNAPSHOT compatibility

## What happened

Landed #190 — SNAPSHOT compatibility fix for Quarkus 3.39, ledger/engine
package moves, CDI wiring, and frontend URL alignment. Build is green
(main + test compilation). 38 files, mechanical fixes only.

## What's next

- **#200** Engine GOAP / LLM Decomposition (L/High) — only actionable open issue
- **#177, #178** — parked ideas, not ready

## Known issues

- `BlocksBeans` excluded from CDI — upstream blocks needs `@DefaultBean` on producers
- `WorkStrategyContributor` excluded — stale `EngineStrategyResolver` import in work-engine-adapter
- Visual verification of contributor trust UI not done (build works, quarkus:dev not run)
- npm tarball fix (`grouped-data-view.tgz` alias) applied to local Maven repo only
