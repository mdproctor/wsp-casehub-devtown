# Handoff — devtown slot 208

**Date:** 2026-10-05
**Branch:** `main` (wip/demo-readiness merged as `eb2145d`)
**Last commit:** `06e6c71` merge commit on main

## What happened this session

### Issues closed: #209, #211, #214, #215

**#209 — GitHub approvals advance the case:**
- `signalReviewSubmitted("approved")` now records approval in case context and completes matching human-decision WorkItems via `WorkItemOperations`
- Dev-mode approve button uses the same code path

**#211 — WorkItem completion API:**
- Triage claim/decide REST endpoints added to `DevtownGovernanceApi`
- `triageItems()` includes PENDING, ASSIGNED, and typeless judgment WorkItems

**#214 — Dev-mode action buttons:**
- claim-workitem and decide-workitem action endpoints on `DevModeStubResource`
- Approve button now produces visible state changes

**#215 — Simulation script + scenario controller:**
- `simulate.sh` rewritten as 7-phase lifecycle with --auto flag
- `ScenarioResource`: 11-step backend at `/scenario/*`
- Standalone scenario runner at `/scenario.html` with step/auto-run/reset
- Idempotent — reset cleans merge queue entries and stale WorkItems
- **Validated end-to-end: 11/11 steps pass, both fresh and repeated runs**

### Dev-mode infrastructure fixes
- Removed `casehub-engine-rest` (duplicate REST endpoints causing DeploymentException)
- `DevClaimSlaPolicy` + `DevWorkerSelectionStrategy` + `DevWorkStrategyContributor`
- Excluded JPA ledger repositories (SNAPSHOT drift broke @DefaultBean)
- esbuild target bumped to es2024 (TC39 decorator support for pages-aria)
- IPv4 default (127.0.0.1) for Java 26 Netty SO_KEEPALIVE bug

### Cross-repo: engine#1216 filed + reproducer test
- CasePlanModel eviction race in JudgmentWorkItemScheduler — now fixed
- Reproducer test at `work-adapter/src/test/java/.../JudgmentWorkItemSchedulerRegistryTest.java`

## What needs to happen next

### New issues filed (#226-#229) — dashboard data visibility

The simulation drives the full lifecycle but the dashboard doesn't surface the interesting data:

| Issue | Title | Scale |
|-------|-------|-------|
| #226 | Dashboard: routing decisions visible in Review detail | M |
| #227 | Dashboard: AI review findings as inline comments | L |
| #228 | Dashboard: rich event timeline with binding and agent events | M |
| #229 | Scenario: produce realistic AI review findings | M |

### Starting dev mode
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app -Dquarkus.console.enabled=false
# Then open http://127.0.0.1:8080/scenario.html
```

### Known issues
- Merge queue batch PK violation when enqueueing multiple PRs sequentially (handled gracefully — reports "batch formation deferred")
- `BlocksBeans` excluded from CDI — upstream blocks needs @DefaultBean on producers
- Test suite: CDI wiring failures in @QuarkusTest (test-specific Jandex config stale)
