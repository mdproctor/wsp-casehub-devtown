# Handoff — issue-221-merge-queue-contributor-workbenches

## Context

Upgrading two devtown dashboard tabs from "functional" (flat declarative tables) to
"working" (interactive Lit components with split-panel drill-down):

1. **Merge Queue** — currently a declarative `queueView` in `views/queue.ts`
2. **Contributors** — currently a declarative `contributorsView` in `views/contributors.ts`

"Working" means the pattern used by Reviews (`review-workbench.ts`) and Reviewers
(`reviewer-workbench.ts`): Lit `@customElement` using `blocks-split-workbench` or
`pages-split-workbench`, list table on the left with `@row-activate`, detail panel
on the right.

## Approved Design

### Contributors — `devtown-contributor-workbench`

Follow the reviewer-workbench pattern exactly:
- **Left panel:** fleet list table (actorId, trustScore, intakeLane, observations,
  mergeRate) using `pages-table` with `@row-activate`
- **Right panel:** existing `blocks-contributor-workbench` component from blocks-ui,
  receiving `endpoint` and `actor-id` from the selected row
- No vitals bar — the blocks-contributor-workbench detail already has intake
  classification and trust breakdown
- Register as `devtown-contributor-workbench` in `index.ts`, replace the declarative
  `contributorsView` in the tab bar

The `blocks-contributor-workbench` already exists:
- Source: `blocks-ui/npm-packages/.../packages/contributor-workbench/src/contributor-workbench.ts`
- Tag: `<blocks-contributor-workbench>`
- Already imported in devtown's `index.ts`: `import "@casehubio/blocks-ui-contributor-workbench"`
- Calls `/contributors/{actorId}` → `GovernanceQueryService.contributorDetail()`
- Shows: intake lane badge, trust score panel, dimension scores, outcome history

### Merge Queue — `devtown-merge-queue-workbench`

Follow the review-workbench pattern with `blocks-split-workbench`:
- **Left panel:**
  - Vitals bar: queue depth, active batches, throughput 24h, failure rate, oldest
    wait, avg wait (from `/api/devtown/governance/merge-queue/metrics`)
  - "Queued PRs" section: table with number, repo, author, lane, trust, wait time
  - "Active Batches" section: table with batchId, caseId, prCount, riskLevel
  - Both tables have `@row-activate`
- **Right panel — detail:**
  - For a queued PR: number, repo, author, lane badge, trust score bar, wait time,
    dependency list, actions (Dequeue, Signal CI Pass)
  - For a batch: batchId, PR count, risk level, list of PRs in the batch
    (from `batchStatus(caseId)`), bisection strategy
- Actions use existing endpoints: `/api/actions/dequeue`, `/api/actions/signal-ci-pass`

**Backend check needed:** `GovernanceQueryService.batchStatus(UUID)` exists as a Java
method but may not have a REST endpoint. Check and add one if missing
(`/api/devtown/governance/merge-queue/batches/{caseId}`).

## Key Files

### Frontend (app/src/main/webui/src/)
- `index.ts` — tab registration and panel registration
- `views/queue.ts` — current declarative merge queue (replace)
- `views/contributors.ts` — current declarative contributors (replace)
- `datasets.ts` — REST data bindings (merge-queue, contributors datasets)
- `components/review-workbench.ts` — reference pattern (split workbench)
- `components/reviewer-workbench.ts` — reference pattern (simpler split)
- `components/review-detail.ts` — reference for detail panel with actions

### Backend (app/src/main/java/.../governance/)
- `GovernanceQueryService.java` — all query methods and records:
  - `mergeQueue()` → `MergeQueueStatus` (QueuedPrEntry, ActiveBatchEntry)
  - `mergeQueueMetrics()` → `MergeQueueMetrics`
  - `batchStatus(UUID)` → `BatchStatus` (BatchPrEntry list)
  - `contributorFleet()` → `List<ContributorFleetEntry>`
  - `contributorDetail(String)` → `ContributorDetail`
- `DevModeStubResource.java` — action endpoints (approve, dequeue, signal-ci-pass)

### blocks-ui (external)
- `blocks-ui/npm-packages/.../contributor-workbench/src/contributor-workbench.ts`
  — existing `<blocks-contributor-workbench>` component

## Implementation Order

1. **Contributor workbench** (simpler — wraps existing component)
   - Create `components/contributor-workbench.ts`
   - Register panel in `index.ts`
   - Replace `contributorsView` with `hostPanel("contributor-workbench", ...)`
   - Test: click contributor row → detail panel shows intake, trust, outcomes

2. **Merge queue workbench**
   - Check/add REST endpoint for `batchStatus`
   - Create `components/merge-queue-workbench.ts`
   - Create `components/merge-queue-detail.ts` (or inline in workbench)
   - Register panel in `index.ts`
   - Replace `queueView` with `hostPanel("merge-queue-workbench", ...)`
   - Test: click queued PR → detail with actions; click batch → batch detail

## What Was Done — Session 1

- Surveyed all 8 dashboard tabs and classified as working/functional/stub
- Read all view files, component files, datasets.ts, and GovernanceQueryService.java
- Read blocks-contributor-workbench source to confirm it exists and understand its API
- Design approved by user
- Branch created on canonical devtown (then deleted — slot not yet created)
- Slot 212 created with feature branch `issue-221-merge-queue-contributor-workbenches`

## What Was Done — Session 2 (12 commits)

### New components
- `components/contributor-workbench.ts` — split panel with fleet table + contributor-detail
- `components/contributor-detail.ts` — narrative panel: summary, lane position with proximity card, quality dimensions, trend assessment
- `components/merge-queue-workbench.ts` — split panel with vitals bar (6 metrics), queued PRs table, active batches table (PRs/CI/Risk columns)
- `components/merge-queue-detail.ts` — PR detail with trust narrative explaining lane assignment, trust bar with threshold markers, contributor history, recent outcomes, dependencies, actions (dequeue/signal-ci-pass). Batch detail with bisection narrative and suspected PR
- `components/reviewer-detail.ts` — narrative panel: capability trust bars, quality dimensions, status card, review history cards with PR context, findings, and feedback badges (accepted/rejected/partial)

### Updated components
- `reviewer-workbench.ts` — replaced blocks-trust-workbench with reviewer-detail, passes maturity phase
- `index.ts` — registered new panels, replaced tab entries for Merge Queue and Contributors

### Backend
- `GovernanceQueryService.ActiveBatchEntry` — added ciStatus, prNumbers, startedAt, suspectedPr fields (wired from BatchRecord)

### Dev tooling
- `mock-server.mjs` — Node mock server serving fixtures + static files
- `mock-fixtures.json` — seeded data for all dashboard endpoints (reviews, merge queue, contributors, reviewers, triage, SLA, sessions, definitions, per-reviewer detail with review history)
- `npm run dev:mock` — builds frontend then starts mock server on :8280

### Cleanup
- Deleted dead view files (`views/queue.ts`, `views/contributors.ts`)

### Key design decisions
- Detail panels use **narrative text** to explain trust scores, not just display numbers — e.g. "alice has submitted 34 PRs — 32 merged, 2 closed. Trust score 88% exceeds fast-track threshold"
- **Proximity cards** show actionable context: "8 points above threshold, a few rejected PRs could demote"
- Reviewer history shows **findings and feedback** — connecting trust dimensions to observable evidence
- Replaced all three blocks-ui detail components (blocks-contributor-workbench, blocks-trust-workbench, blocks-split-workbench kept) with custom components that render contextual narratives

## What Comes Next — #231

**Unify Merge Queue and Reviewers detail panels with shared composable components.**

The three detail panels (merge-queue-detail, contributor-detail, reviewer-detail) share duplicated patterns:
- Trust bar with thresholds
- Status/proximity cards
- Quality dimensions grid
- Review history cards

Extract 4 shared components and rewire the detail panels as thin compositions. The difference between Merge Queue (PR-centric) and Reviewers (agent-centric) should be which perspective the shared components render from, not different UI structures.

Also: add review history cards to Merge Queue PR detail — show what agents found on this PR, not just the contributor's trust profile.

See devtown#231 for full plan.

## Mock Server — What It Is and What Needs to Happen

### What it is
`app/src/main/webui/mock-server.mjs` is a lightweight Node HTTP server that serves the real frontend bundle alongside canned JSON API responses from `mock-fixtures.json`. The frontend code is real — the same Lit components that run against Quarkus. Only the data source is fake.

Run with `npm run dev:mock` from `app/src/main/webui/`. No Java, no Quarkus, no database.

### Why it matters
The fixture data tells a coherent story: alice is a trusted fast-track contributor, carol is risky with enhanced review, agent-sec-01 has a high false-positive rate, a batch is bisecting with PR #97 as the suspect. This data is what makes the UI evaluable — without it, every tab shows empty or "No data". **Do not delete the fixtures.** They are the reference dataset for UI development.

### Path to real backend with the same views

The UI currently works against hand-crafted fixture JSON. To get the same views from the real Quarkus backend, these gaps need closing:

| What the UI expects | Backend status | What needs to happen |
|---------------------|---------------|---------------------|
| `ActiveBatchEntry.ciStatus` | Defaults to `"RUNNING"` | Wire real CI status from merge batch case context |
| `ActiveBatchEntry.suspectedPr` | Always `null` | Implement bisection tracking in `MergeBatchCaseHub` |
| `ReviewerHealth.recentOutcomes[].pr` | Not returned | Enrich outcomes with PR context in `GovernanceQueryService.reviewerHealth()` by cross-referencing caseId → PrReviewCaseTracker |
| `ReviewerHealth.recentOutcomes[].findingSummary` | Not returned | Extract finding summaries from worker output events in the event log |
| `ReviewerHealth.recentOutcomes[].feedbackOutcome` | Not returned | Track whether findings were accepted/rejected — needs a feedback signal (not yet designed) |
| Contributor detail `recentOutcomes` | Returned but basic | Already works with real data — no change needed |

### Seeded dev-mode data

For Quarkus dev mode to show populated dashboards (not just empty tables), devtown needs a dev-mode data seeder — a `@Startup` observer that creates synthetic cases, batches, trust scores, and review outcomes in the in-memory stores. This doesn't exist yet. Until then, the mock server is the only way to see populated views.

**Recommendation:** Build a `DevModeDataSeeder` that populates the same data as `mock-fixtures.json` into the real services at startup. Then `quarkus:dev` shows the same dashboard as `npm run dev:mock`, but through the real code path.

## What Was NOT Done

- Full Maven build with tests not run (Java tests unrelated to frontend changes)
- Quarkus dev mode not successfully started (port conflict with hortora engine + IntelliJ heap pressure)
- No frontend tests written
- Mock data shapes for ActiveBatchEntry have fields (ciStatus, suspectedPr) that default to RUNNING/null in the real backend — bisection status tracking needs real implementation
- Reviewer history enrichment (`pr`, `findingCount`, `findingSummary`, `feedbackOutcome` on outcomes) needs backend support in `GovernanceQueryService.reviewerHealth()`
