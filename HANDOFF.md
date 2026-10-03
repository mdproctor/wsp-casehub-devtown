# Handoff — devtown slot 208

**Date:** 2026-10-03
**Branch:** `wip/demo-readiness` (13 commits, not merged)
**Last commit:** `db3ebd8` feat: action buttons + action API

## What happened this session

### Repo sync (all 12 repos)
All canonical repos synced to both `origin` (mdproctor) and `upstream` (casehubio):
- Resolved pages 17-file merge conflict (origin #507 refactoring vs upstream CI fixes)
- Resolved eidos SHA divergence (same content, different hashes — merged)
- Resolved blocks merge conflict (social-jpa removal)
- All slot 208 repos reset to canonical main

### SNAPSHOT chain rebuild
Rebuilt from source in dependency order: platform → worker → ledger → connectors → neocortex → eidos → qhorus → work → engine. Blocks build fails (neocortex `MemoryDomain` type change) but devtown doesn't need current blocks jar.

### DevTown dev mode running
Fixed: CDI exclusion packages, Hibernate entity packages (qhorus sub-packages), Flyway migration version conflicts, eidos descriptor YAML format, connector config placeholders, artifact rename (work-engine-adapter → engine-work-adapter), `deny-unannotated-endpoints` (must be global, not `%dev.`).

### UI — 9 tabs functional
All tabs render. Custom workbench components built:
- `operations-workbench` — KPI cards, problems, active reviews, event stream. Clickable rows open detail pane with case events.
- `review-workbench` — master-detail split pane. Click PR row to see metadata + filtered event timeline.
- Action buttons: Approve, Request Changes, Add to Merge Queue (enqueue fails on H2 — composite-key IN clause needs PostgreSQL).

### Simulation script
`app/src/main/resources/demo/simulate.sh` seeds 3 PRs via REST API. Action endpoints at `/api/actions/{approve,request-changes,enqueue,dequeue}`.

## What needs to happen next

### Design-first: user story mapping
The UI works mechanically but lacks coherent user stories. Each tab should correspond to a step in the PR review workflow, not be an independent island. Specifically:

1. **Map the user journey** — what persona uses each tab, what they're trying to accomplish, what actions they can take at each stage
2. **Study Gastown UIs** — `docs/gastown-casehub-analysis-v2.md` has the Refinery (merge queue dashboard) and Deacon (oversight) flows. These are the production reference for what devtown should demonstrate.
3. **Differentiate Operations vs Reviews** — currently too similar. Operations should be the command center (system-wide awareness), Reviews should be case-centric investigation.
4. **Build comprehensive simulation scenarios** — the current 3-PR simulation doesn't exercise approvals, merge queue batching, trust scoring, SLA breaches, or CI feedback loops. Need scenarios that walk through the full lifecycle so every tab has meaningful, interconnected data.
5. **Merge queue needs PostgreSQL** — H2 doesn't support composite-key IN clause used by the merge queue JPA queries. Either switch dev-mode to PostgreSQL (Testcontainers) or fix the JPA query for H2 compatibility.

### Known issues
- `BlocksBeans` excluded from CDI — upstream blocks needs `@DefaultBean` on producers
- `WorkStrategyContributor` excluded — stale `EngineStrategyResolver` import in work-engine-adapter
- Test suite: 65+ CDI wiring failures (test-specific Jandex index config stale against new engine)
- `MergeQueueSlaWorkItemTest` deleted (used removed Panache active-record API)
- Blocks canonical repo: build fails against new neocortex (`MemoryDomain` type changed from String to class)
- Dev mode process dies when background task timeout expires — run `mvn quarkus:dev -pl app` in terminal for persistence

### Starting dev mode
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app -Dquarkus.console.enabled=false
# Then seed data:
zsh app/src/main/resources/demo/simulate.sh
```

### Branch status
`wip/demo-readiness` has 13 commits on top of `main`. Not pushed to origin. Ready for review when user stories are mapped and the simulation is comprehensive.
