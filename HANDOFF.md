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

## What Was Done This Session

- Surveyed all 8 dashboard tabs and classified as working/functional/stub
- Read all view files, component files, datasets.ts, and GovernanceQueryService.java
- Read blocks-contributor-workbench source to confirm it exists and understand its API
- Design approved by user
- Branch created on canonical devtown (then deleted — slot not yet created)
- Slot 212 created with feature branch `issue-221-merge-queue-contributor-workbenches`

## What Was NOT Done

- No code written
- No REST endpoint added
- No tests written
- Build not verified (`mvn clean install` not run in slot)
