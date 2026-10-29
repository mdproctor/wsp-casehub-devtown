# Review Intelligence Workbench — Design Spec

**Issue:** casehubio/devtown#216
**Epic:** #208
**Branch:** issue-216-review-workbench-ui
**Date:** 2026-10-29

---

## 1. Problem

DevTown's UI currently presents PRs as a flat list with a detail pane — structurally identical to GitHub's PR page. The value proposition of DevTown is the intelligence layer: trust-weighted routing rationale, CBR precedents, contributor history, policy context, and SLA calibration. This intelligence exists in the backend (`GovernanceQueryService`, `TrustQueryService`, `CbrRetrievalService`, `SlaCalibrationStore`) but the UI doesn't surface it coherently.

The tab structure conflates operational monitoring (problems, event streams) with review intelligence (trust, routing, findings). Workers and Definitions tabs serve dev/debug concerns but occupy top-level navigation.

## 2. Solution

Replace the PR Cases tab with a **Review Intelligence Workbench** — a two-region layout where the left region shows case lifecycle as decision events (timeline spine) and the right region shows contextual intelligence for any clicked event node. Restructure tabs to separate daily-use views from configuration.

## 3. Architecture

### 3.1 Tab Structure

Six tabs, reordered by daily-use priority:

| Tab | Content | Component | Data endpoint |
|-----|---------|-----------|---------------|
| **Review Workbench** (default) | Spine/table + intelligence pane | `pr-cases-workbench` (modified) | `/api/devtown/governance/queue-status`, `/api/devtown/reviews` |
| **Operations** | Live: problems, active reviews, event stream, sessions | `operations-workbench` (now mounted + sessions section) | `/api/devtown/governance/problems`, `/api/devtown/governance/recent-events` |
| **Merge Queue** | Batches, wait times, throughput, ordering | `merge-queue-workbench` (new — extracted from PR Cases "Queued" filter + active batches) | `/api/devtown/governance/merge-queue`, `/api/devtown/governance/merge-queue/metrics` |
| **Reviewers** | Trust scores, routing effectiveness, decision history | `reviewer-workbench` (unchanged) | `/api/devtown/reviews/reviewers` |
| **Contributors** | Trust, intake lanes, quality, GitHub intelligence | `contributor-workbench` (unchanged) | `/api/devtown/reviews/contributors` |
| **System** | SLA calibration, merge queue config, fleet settings | `system-workbench` (refactored) | `/api/devtown/governance/sla-comparison`, `/api/devtown/governance/preferences` |

**Removed tabs:** Workers (session monitoring folds into Operations), Definitions (accessible via direct URL only).

### 3.2 Review Workbench Layout

The Review Workbench uses the existing `pages-split-workbench` split-pane layout with modified content in both slots.

**Left region — Timeline Spine (default) or Table View (toggle):**

A toggle chip in the filter bar switches between spine and table views. Both share the same data source (`UnifiedPrRow` model) and selection mechanism — clicking a case in either view populates the intelligence pane.

- **Spine view:** Vertical list of cases, each with a header card and indented event nodes. Cases with in-progress or attention-needed nodes expand automatically; completed cases collapse to a summary row with status dot indicators (D14).
- **Table view:** Dense sortable table with columns: PR#, repo, author, status, lines, wait time. Equivalent to current PR Cases list for ops scanning scenarios.

Filter chips carry over: All, In Review, Needs Decision. The Review Workbench scopes to review-lifecycle cases only — QUEUED and BATCHED PRs are excluded at the data level (filtered from the unified model before rendering) and appear exclusively in the Merge Queue tab. The "All" chip shows all review-lifecycle cases across stages, not all PRs system-wide.

**Scalability (D4):** At high volume (50+ active cases), the spine is a flat scrollable list of uniform-ish height divs. Virtual scrolling can be added as a standard windowing solution without architectural changes — the flat-div spine (D7) was chosen specifically to enable this.

**Right region — Intelligence Pane:**

A single `<intelligence-pane>` component that switches content based on which timeline node (or table row) is clicked. Five primary node types, each with its own data requirements and layout:

| Node type | Sections shown | Data sources |
|-----------|---------------|--------------|
| **Case header** | Contributor profile, similar past cases (CBR), code classification, policy context, SLA estimate | `GET /reviews/{caseId}`, `GET /reviews/contributors/{actorId}`, `GET /reviews/{caseId}/similar-cases` (§5.1) |
| **Dispatch** | Selection rationale, agent profile, routing strategy. **Two tiers:** in-progress dispatches show basic rationale (`reason`, `confidence`, `bindingName` from AGENT_ROUTED event metadata) + agent profile; completed dispatches additionally show candidate alternatives via full routing detail | In-progress: `ReviewDetail.routing` (event metadata), `GET /reviews/reviewers/{actorId}`. Completed: additionally `GET /trust/{actorId}/routing-history/{entryId}` (`entryId` from §5.2 enrichment) |
| **Findings** | Finding list (severity, file, confidence), precedent comparison (CBR confirmation rate) | `GET /reviews/{caseId}` (findings data), `GET /reviews/{caseId}/similar-cases` (aggregate confirmation rate client-side) |
| **Human gate** | WorkItem status, what needs deciding, decision history for candidate group | Triage data from unified model, `GET /reviews/contributors/{candidateGroup}` |
| **CI / Lifecycle** | Event detail, timestamp, status | Inline from spine event data (no additional fetch) |

When a case is selected from the table view (not a specific node), the intelligence pane defaults to the **case header** view.

The Review Workbench detail slot always renders the intelligence pane for the selected case. Queue/batch detail rendering has moved to the Merge Queue tab, which uses `pr-detail.ts` in its own split-pane layout (see §4.1 `merge-queue-workbench`). The D11 coexistence clause no longer applies to the Review Workbench after R1-09's scoping change.

**Known data gaps:** `TrustScoreSnapshot` has been removed from the ledger (pending replacement entity). This affects the dispatch view: trust trend data (`TrustQueryService.trustTrend()`) returns empty, and gate decision before/after scores are unavailable (`0.0` placeholders). Current trust scores remain available via `TrustGateService.currentScore()`. The intelligence pane renders "Data unavailable" for trend charts and "—" for zeroed gate scores rather than displaying misleading zeros.

### 3.3 Timeline Spine Component

**Element:** `<review-spine>`

A custom LitElement that renders a flat `<div>` list — not one `agency-timeline` per case. Each case is a header div followed by indented event node divs. This avoids N shadow DOM subtrees with unused features (pagination, filtering, detail expansion) and enables future virtual scrolling over uniform-height elements.

**Data model:** The spine consumes the same unified data model as the current `pr-cases-workbench` (merged from queue-status, merge-queue, and triage endpoints). Per-case event data comes from `GET /reviews/{caseId}` fetched on expand.

**Spine node strategy (`spineNodeStrategy`):**

A dedicated strategy module (not `reviewTimelineStrategy`) that synthesizes compact human-readable labels from event + case context + routing data:

| Event type | Synthesized label example |
|------------|--------------------------|
| Case opened | `● Opened` |
| Code analysis complete | `● Code analysis: security-sensitive` |
| Reviewer dispatched | `● Security review → agent-3` |
| Review complete (approved) | `● Style review: APPROVED` |
| Review in progress with findings | `◐ Security review: 2 findings` |
| Human gate pending | `○ Human gate` |
| CI pending | `○ CI` |

Shares category icon mapping and status colour logic with `reviewTimelineStrategy` via shared imports — the synthesis logic (combining event type + capability + actor + outcome into a label) is strategy-specific.

Labels for expanded event nodes are synthesised from the `GET /reviews/{caseId}` timeline response — event metadata includes `capabilityName` and `actorId`, which are sufficient for labels like "Security review → agent-3". Collapsed rows do not show per-event labels — they show only the coarse status indicator derived from the unified model (see collapse/expand section above).

**Collapse/expand behaviour (D14):**

- **Collapsed row:** Case header with PR number, short title, author, line count, and a single status indicator derived from the unified model's `stage` and `status` fields. No per-event dots — collapsed rows use only data available from the initial unified model fetch (no `GET /reviews/{caseId}` needed). Status indicator mapping: REVIEW+ANALYSING → blue (early stage), REVIEW+IN_REVIEW → yellow (in progress), HUMAN_GATE → red (needs attention), APPROVED/COMPLETED → green (done).
- **Expanded view:** Case header (clickable to collapse) followed by indented event nodes with `●/◐/○` indicators and timestamps. Per-event data comes from `GET /reviews/{caseId}` fetched on expand.
- **Smart default:** Cases are expanded if their unified model `stage` is `REVIEW` (with `status` not `APPROVED`/`COMPLETED`) or `HUMAN_GATE`. Cases with `stage` in `QUEUED`, `BATCHED`, or with `status` `APPROVED`/`COMPLETED` are collapsed. This uses only data from the initial unified model fetch — no per-event data is required for expand/collapse decisions.
- **Manual override:** Clicking a case header toggles expand/collapse. The user's manual choices persist until the data refreshes.
- **Edge case:** A case with all review nodes complete but not yet merged (awaiting queue entry) will be collapsed by the smart default. The "Needs Decision" filter chip surfaces these if the user is looking for them.

**Border colour:** Left border of collapsed rows reflects overall case status derived from the unified model's `stage` and `status` fields: green (`status` = APPROVED/COMPLETED), yellow (`stage` = REVIEW, `status` = IN_REVIEW), red (`stage` = HUMAN_GATE), blue (`stage` = REVIEW, `status` = ANALYSING).

### 3.4 Intelligence Pane Component

**Element:** `<intelligence-pane>`

A single LitElement with a `nodeType` property and a `nodeData` property. Internally switches on `nodeType` to render the appropriate sections using existing shared sub-components.

**Shared primitives reused:**
- `renderTrustBar()` — trust score bar with threshold markers
- `renderDimensionGrid()` — quality dimension grid
- `renderBadge()` — intake lane, maturity badges
- `renderReviewCard()` — review outcome cards
- `renderStatusCard()` — tier status (good/watch/concern)
- `<devtown-routing-summary>` — routing decisions + feature vector
- `<devtown-findings-panel>` — findings grouped by capability
- `renderWorkItemAlert()` — human decision required alert

No higher-level section extraction from `pr-detail.ts` is needed (D9) — the shared primitives are already the right reuse level. The intelligence pane composes them into node-type-specific layouts.

**Fetch pattern (D3, D12):**

Each section fires its own `fetch()` on mount. A lightweight dedup cache (`Map<string, {promise: Promise, timestamp: number}>`) with 30-second TTL eliminates duplicate concurrent requests and caches responses during click-around exploration. All fetches for a node type fire in parallel — no waterfall.

### 3.5 Real-Time Updates

The spine data refreshes via WebSocket-triggered re-fetch using the `triggerUrl + triggerFilter` pattern from `datasets.ts` (D6). The existing `queue-status` dataset entry in `createDatasets()` gains `triggerUrl` and `triggerFilter` properties:

| Property | Value |
|----------|-------|
| Dataset ID | `queue-status` (existing entry, enhanced) |
| REST URL | `/api/devtown/governance/queue-status` |
| Trigger URL | `ws[s]://{host}/api/governance/events` (same WebSocket used by `plan-items`, `goal-status`, `case-context`) |
| Trigger filter | `(e) => e.topic === "case.state" \|\| e.topic === "planitem.state" \|\| e.topic === "workitem.lifecycle"` |
| Data path | `reviews` |

This is consistent with how `plan-items`, `goal-status`, and `case-context` datasets already work — the `pages-data` framework manages WebSocket connection, reconnection, and trigger filtering. The spine component consumes the `queue-status` dataset via framework-managed data binding.

No manual WebSocket subscription or client-side case-ID filtering is needed.

## 4. Component Changes

### 4.1 New Files

| File | Element/Module | Lines (est.) | Purpose |
|------|---------------|-------------|---------|
| `src/components/review-spine.ts` | `<review-spine>` | ~300 | Flat-list spine with collapse/expand, smart default, filter chips, data binding |
| `src/components/spine-node-strategy.ts` | (module) | ~100 | Label synthesis, shared category icons, status colour mapping |
| `src/components/intelligence-pane.ts` | `<intelligence-pane>` | ~400 | Node-type switching, section composition, dedup cache, parallel fetches |
| `src/components/merge-queue-workbench.ts` | `<merge-queue-workbench>` | ~300 | Merge queue tab: `pages-split-workbench` layout with queued PRs table + active batches in the list slot, `<devtown-pr-detail>` in queue/batch mode in the detail slot. Extracted from PR Cases "Queued" filter + active batches section. Sole consumer of `pr-detail`'s queue and batch modes after D11 scoping change |

### 4.2 Modified Files

| File | Change | Scope |
|------|--------|-------|
| `src/views/review.ts` | Replace 6-tab definition with new tab structure (§3.1) | Tab labels, panel references, endpoints |
| `src/index.ts` | Register new panels (`review-spine`, `intelligence-pane`), remove `session-workbench` and `definitions` registrations, mount `operations-workbench` | Panel registration block |
| `src/components/pr-cases-workbench.ts` | Add spine/table toggle chip, integrate `<review-spine>` in list slot, detail slot always renders `<intelligence-pane>` (review-lifecycle cases only after R1-09 scoping) | List slot content, detail slot simplified, toggle state |
| `src/components/pr-detail.ts` | Remove review-mode rendering (trust section, routing, timeline, findings) — narrow to queue context (priority lane, trust score, wait time, dependencies) and batch context only. Consumer is now `merge-queue-workbench` (not `pr-cases-workbench`). Triage/human-gate rendering removed — handled by intelligence pane's human gate node type | Remove ~150 lines of review-mode rendering, consumer change |
| `src/components/operations-workbench.ts` | (1) Add collapsible "Sessions" section sourcing from `/api/sessions` — currently a dev-mode stub (`DevModeStubResource.sessions()`) returning empty data; section renders "No session data" when empty. (2) Fix `_navigateToReview()`: replace brittle `textContent?.trim() === 'Reviews'` tab selector with `data-panel-id` attribute matching (current selector is already broken against the `'PR Cases'` tab label). (3) `setInterval` polling migration to WebSocket triggers deferred — acknowledged as inconsistent with §3.5 pattern | Sessions section, navigation fix, scope note |
| `src/components/system-workbench.ts` | Remove problems display and active data. Focus on SLA calibration table, merge queue configuration, fleet settings | Remove problems table, health cards refocused |
| `src/datasets.ts` | Enhance existing `queue-status` dataset entry with `triggerUrl` and `triggerFilter` for WebSocket-triggered re-fetch (§3.5). No new dataset entry needed — the existing binding is augmented | WebSocket trigger properties on existing entry |

### 4.3 Deleted/Retired Files

| File | Reason |
|------|--------|
| `src/views/operations.ts` | Superseded by `operations-workbench.ts` (already unused) |
| `src/views/system.ts` | Superseded by `system-workbench.ts` (already unused) |

### 4.4 Unchanged Files

- `src/components/reviewer-workbench.ts` — no changes
- `src/components/reviewer-detail.ts` — no changes
- `src/components/contributor-workbench.ts` — no changes
- `src/components/contributor-detail.ts` — no changes
- `src/components/review-timeline-strategy.ts` — no changes (spine uses its own strategy; detail view in intelligence pane continues using this)
- `src/components/shared/*` — no changes (already extracted to the right reuse level)
- `src/components/routing-summary.ts` — no changes
- `src/components/findings-panel.ts` — no changes

## 5. Backend Changes

Two targeted backend changes are required.

### 5.1 REST endpoint for CBR similar cases

`MemoryQueryResolver.similarCases()` is exposed via GraphQL and MCP protocol only — it uses `@GraphQLApi` + `@Query` without `basePath`/`@PlatformQuery`/`@RestPath`. The frontend intelligence pane cannot call it via `fetch()`.

**Change:** Add a REST endpoint to `DevtownReviewApi`:

| Annotation | Value |
|------------|-------|
| `@PlatformQuery` | `"Find similar cases by CBR"` |
| `@RestPath` | `"/{caseId}/similar-cases"` |

This delegates to `CbrRetrievalService.findSimilar()`, extracting PR parameters (`repo`, `prNumber`, `contributor`, `linesChanged`, `changedPaths`) from the case's `PrPayload`. The frontend calls `GET /api/devtown/reviews/{caseId}/similar-cases` with only the case ID — no need to pass CBR parameters explicitly.

### 5.2 Routing entry ID in ReviewDetail

The intelligence pane's dispatch view needs `GET /trust/{actorId}/routing-history/{entryId}` for full routing rationale with alternatives. Currently, `ReviewDetail.RoutingDecision` contains `(capability, reason, confidence, bindingName)` but no `entryId` — the `WorkerDecisionEntry.id` is not in the AGENT_ROUTED event metadata.

**Change:** Enrich `GovernanceQueryService.reviewDetail()` to correlate routing decisions with `WorkerDecisionEntry` records by `caseId` + `capabilityTag`, adding the `entryId` to each `RoutingDecision`. Update the record: `RoutingDecision(capability, reason, confidence, bindingName, entryId)`.

**Timing constraint:** `WorkerDecisionEntry` is written by `WorkerDecisionEventCapture` on worker *completion* (fired from `WorkflowExecutionCompletedHandler`), not at dispatch time. For in-progress dispatches, `entryId` is null and the correlation query returns empty. The intelligence pane handles this with two-tier dispatch intelligence:

- **In-progress dispatches** (`entryId` = null): Show basic rationale from AGENT_ROUTED event metadata (`reason`, `confidence`, `bindingName`) plus agent profile from `GET /reviews/reviewers/{actorId}`. No candidate alternatives. Display "Full rationale available after review completes."
- **Completed dispatches** (`entryId` present): Full rationale with candidate alternatives, trust scores, and selection strategy via `GET /trust/{actorId}/routing-history/{entryId}`.

### Existing endpoints (no changes)

- `GET /api/devtown/reviews/{caseId}` — review detail with timeline, capabilities, routing (now enriched with `entryId`), findings
- `GET /api/devtown/reviews/contributors/{actorId}` — contributor profile with GitHub intelligence
- `GET /api/devtown/reviews/reviewers/{actorId}` — reviewer health with trust scores
- `GET /api/devtown/trust/{actorId}/routing-history/{entryId}` — routing decision rationale with candidates
- `GET /api/devtown/governance/sla-comparison` — SLA calibration data

CBR confirmation rate is computed client-side from the `capabilityOutcomes` in the similar cases response (D5). If the precedent pool grows beyond ~20 results, server-side aggregation becomes a separate decision.

## 6. Testing Strategy

### 6.1 Unit Tests

- `spine-node-strategy.ts` — label synthesis for each event type, status colour mapping, category icon selection
- `intelligence-pane.ts` — node-type switching renders correct sections, dedup cache TTL behaviour
- `review-spine.ts` — smart default expand logic (expand active, collapse done), manual toggle override

### 6.2 Integration Tests

- Tab restructure — all 6 tabs render without errors, correct panel content
- Spine ↔ intelligence pane interaction — clicking spine node populates correct intelligence pane content
- Table ↔ intelligence pane interaction — clicking table row shows case header intelligence
- Detail slot — Review Workbench always shows intelligence pane; Merge Queue tab shows pr-detail for queue/batch selection
- WebSocket trigger — simulated event triggers spine data re-fetch

### 6.3 Visual Verification

- Collapsed spine row renders single status indicator correctly (blue/yellow/red/green mapped from `stage` + `status`)
- Expanded spine shows event nodes with correct `●/◐/○` indicators
- Intelligence pane sections render shared primitives (trust bars, dimension grids, badges) correctly
- Spine/table toggle preserves selection state

## 7. Acceptance Criteria Mapping

| Criterion | How addressed |
|-----------|---------------|
| Timeline renders case lifecycle as clickable decision nodes | `review-spine.ts` with `spineNodeStrategy` — each event is a clickable node |
| Intelligence pane shows contributor, code, reviewer, policy context | `intelligence-pane.ts` case header view — parallel fetches for ContributorDetail, CodeAnalysisResult, routing policy |
| Dispatch nodes show trust-based selection rationale with alternatives | `intelligence-pane.ts` dispatch view — two tiers: in-progress shows basic rationale from event metadata + agent profile; completed shows full CandidateScore list via `GET /trust/{actorId}/routing-history/{entryId}` (§5.2) |
| Findings nodes show precedent comparison from CBR | `intelligence-pane.ts` findings view — `GET /reviews/{caseId}/similar-cases` (§5.1), client-side confirmation rate |
| Human gate nodes show WorkItem status, what needs deciding, decision history | `intelligence-pane.ts` human gate view — triage data + contributor decision history |
| Contributor click-through shows full ContributorDetail including GitHubIntelligence | `GET /reviews/contributors/{actorId}` already returns full profile with `githubIntelligence` |
| Timeline updates in real-time via WebSocket events | `triggerUrl + triggerFilter` pattern in `datasets.ts` — existing WebSocket triggers re-fetch |
| Data from existing services — minimal backend | Two targeted backend additions (§5): REST exposure for `similarCases` via `DevtownReviewApi`, `entryId` enrichment in `RoutingDecision`. All other intelligence data served by existing endpoints |

## 8. Decisions

14 decisions captured in `decisions.md` in this spec directory. Key decisions:

- **D1:** Spine is default view with table toggle (not spine-only)
- **D3:** Inline fetch with 30s dedup cache (not a data service layer)
- **D7:** Custom flat-div spine (not N agency-timeline instances)
- **D8:** Single intelligence pane with conditional render (not separate pane per node type)
- **D11:** pr-detail moves to Merge Queue tab for queue/batch modes (no longer in Review Workbench after R1-09 scoping)
- **D14:** Smart default expand — expand active cases, collapse completed

## References

- `app/src/main/webui/src/views/review.ts` — current tab structure (6 tabs)
- `app/src/main/webui/src/components/pr-cases-workbench.ts` — current PR list + detail split (649 lines)
- `app/src/main/webui/src/components/pr-detail.ts` — current PR detail with trust, routing, timeline, findings
- `app/src/main/webui/src/components/review-timeline-strategy.ts` — event categorisation and rendering strategy
- `app/src/main/webui/src/components/operations-workbench.ts` — existing but unmounted operations view
- `app/src/main/webui/src/components/system-workbench.ts` — current system health view
- `app/src/main/webui/src/datasets.ts` — data binding with triggerUrl/triggerFilter pattern
- `app/src/main/webui/src/components/shared/` — shared rendering primitives (trust-bar, dimension-grid, badge, etc.)
- `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java` — central query service
- `app/src/main/java/io/casehub/devtown/app/governance/TrustQueryService.java` — trust scores and routing history
- `app/src/main/java/io/casehub/devtown/app/mcp/MemoryQueryResolver.java` — similarCases CBR query (GraphQL/MCP only; REST access via `DevtownReviewApi` §5.1)
- Issue #216 — full requirements and acceptance criteria
- Visual companion mockups: `before-after-tabs.html`, `spine-collapse-expand.html`
