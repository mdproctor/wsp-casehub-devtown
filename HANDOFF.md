# HANDOFF — 2026-08-04

## Last Session

Completed #153 — governance SLA comparison view. Added `slaComparison()` to GovernanceQueryService joining SlaCalibrationStore (estimated medians from precedent cases) with PreferenceProvider (configured SLA hours). REST endpoint at `GET /api/governance/sla-comparison`, frontend SLA Calibration table in the System view with per-capability breakdown and deviation percentage. Branch closed — squashed to 1 commit, pushed to fork and upstream. 1 garden entry submitted (IntelliJ MCP import optimization gotcha).

## Known Issues

- **Quinoa npm install** fails during Quarkus augmentation — pre-existing frontend build issue
- **CaseMemoryObserver** binary incompatibility with neocortex SNAPSHOT — resolves when engine is rebuilt
- **reviewerId always "unknown"** in ReviewOutcomeObserver — PlanItemStateChangedEvent doesn't carry executor identity (accepted scope reduction; needs PlanItemRecord query to resolve)
