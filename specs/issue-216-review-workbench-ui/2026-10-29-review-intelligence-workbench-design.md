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
| **Merge Queue** | Batches, wait times, throughput, ordering | Extracted from PR Cases "Queued" filter | `/api/devtown/governance/merge-queue`, `/api/devtown/governance/merge-queue/metrics` |
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

Filter chips carry over: All, In Review, Needs Decision.

**Right region — Intelligence Pane:**

A single `<intelligence-pane>` component that switches content based on which timeline node (or table row) is clicked. Five primary node types, each with its own data requirements and layout:

| Node type | Sections shown | Data sources |
|-----------|---------------|--------------|
| **Case header** | Contributor profile, similar past cases (CBR), code classification, policy context, SLA estimate | `GET /reviews/{caseId}`, `GET /reviews/contributors/{actorId}`, `similarCases` MCP query |
| **Dispatch** | Selection rationale (with alternatives), agent profile, routing strategy | `GET /trust/{actorId}/routing-history/{entryId}`, `GET /reviews/reviewers/{actorId}` |
| **Findings** | Finding list (severity, file, confidence), precedent comparison (CBR confirmation rate) | `GET /reviews/{caseId}` (findings data), `similarCases` MCP query (aggregate client-side) |
| **Human gate** | WorkItem status, what needs deciding, decision history for candidate group | Triage data from unified model, `GET /reviews/contributors/{candidateGroup}` |
| **CI / Lifecycle** | Event detail, timestamp, status | Inline from spine event data (no additional fetch) |

When a case is selected from the table view (not a specific node), the intelligence pane defaults to the **case header** view.

For **queue, triage, and batch** contexts, the existing `pr-detail.ts` component renders in the detail slot instead of the intelligence pane (D11). The workbench knows the context from the `UnifiedPrRow.stage` field.

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

**Collapse/expand behaviour (D14):**

- **Collapsed row:** Case header with PR number, short title, author, line count, status summary text, and status dot indicators (coloured dots: green=complete, yellow=in-progress, red=attention, grey=pending).
- **Expanded view:** Case header (clickable to collapse) followed by indented event nodes with `●/◐/○` indicators and timestamps.
- **Smart default:** Cases are expanded if they have any node in `in-progress`, `reviewing`, or `pending-decision` state. Cases where all nodes are `complete` or `approved` are collapsed.
- **Manual override:** Clicking a case header toggles expand/collapse. The user's manual choices persist until the data refreshes.

**Border colour:** Left border of collapsed rows reflects overall case status: green (all clear), yellow (in progress), red (needs attention / human gate), blue (early stage / analysing).

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

The spine data set uses the `triggerUrl + triggerFilter` pattern from `datasets.ts` (D6). WebSocket events from `/api/governance/events` trigger selective REST re-fetches of spine data. This is consistent with how `plan-items`, `goal-status`, and `case-context` datasets already work — the `pages-data` framework manages WebSocket connection, reconnection, and trigger filtering.

No manual WebSocket subscription or client-side case-ID filtering is needed.

## 4. Component Changes

### 4.1 New Files

| File | Element/Module | Lines (est.) | Purpose |
|------|---------------|-------------|---------|
| `src/components/review-spine.ts` | `<review-spine>` | ~300 | Flat-list spine with collapse/expand, smart default, filter chips, data binding |
| `src/components/spine-node-strategy.ts` | (module) | ~100 | Label synthesis, shared category icons, status colour mapping |
| `src/components/intelligence-pane.ts` | `<intelligence-pane>` | ~400 | Node-type switching, section composition, dedup cache, parallel fetches |

### 4.2 Modified Files

| File | Change | Scope |
|------|--------|-------|
| `src/views/review.ts` | Replace 6-tab definition with new tab structure (§3.1) | Tab labels, panel references, endpoints |
| `src/index.ts` | Register new panels (`review-spine`, `intelligence-pane`), remove `session-workbench` and `definitions` registrations, mount `operations-workbench` | Panel registration block |
| `src/components/pr-cases-workbench.ts` | Add spine/table toggle chip, integrate `<review-spine>` in list slot, switch detail slot between `<intelligence-pane>` (review context) and `<devtown-pr-detail>` (queue/triage/batch) | List slot content, detail slot switching, toggle state |
| `src/components/pr-detail.ts` | Remove review-mode rendering (trust section, routing, timeline, findings) — narrow to queue context (priority lane, trust score, wait time, dependencies), triage context (decision type, escalation), and batch context only | Remove ~150 lines of review-mode rendering |
| `src/components/operations-workbench.ts` | Add collapsible "Sessions" section sourcing from `/api/sessions` (previously Workers tab content) | New section at bottom |
| `src/components/system-workbench.ts` | Remove problems display and active data. Focus on SLA calibration table, merge queue configuration, fleet settings | Remove problems table, health cards refocused |
| `src/datasets.ts` | Add spine data set binding with `triggerUrl` for WebSocket-triggered re-fetch | New dataset entry |

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

None required. All intelligence data is already served by existing endpoints:

- `GET /api/devtown/reviews/{caseId}` — review detail with timeline, capabilities, routing, findings
- `GET /api/devtown/reviews/contributors/{actorId}` — contributor profile with GitHub intelligence
- `GET /api/devtown/reviews/reviewers/{actorId}` — reviewer health with trust scores
- `GET /api/devtown/trust/{actorId}/routing-history/{entryId}` — routing decision rationale with candidates
- `similarCases` query via `MemoryQueryResolver` (`@McpDomain`) — CBR precedents with capability outcomes
- `GET /api/devtown/governance/sla-comparison` — SLA calibration data

CBR confirmation rate is computed client-side from the `capabilityOutcomes` in the `similarCases` response (D5). If the precedent pool grows beyond ~20 results, server-side aggregation becomes a separate decision.

## 6. Testing Strategy

### 6.1 Unit Tests

- `spine-node-strategy.ts` — label synthesis for each event type, status colour mapping, category icon selection
- `intelligence-pane.ts` — node-type switching renders correct sections, dedup cache TTL behaviour
- `review-spine.ts` — smart default expand logic (expand active, collapse done), manual toggle override

### 6.2 Integration Tests

- Tab restructure — all 6 tabs render without errors, correct panel content
- Spine ↔ intelligence pane interaction — clicking spine node populates correct intelligence pane content
- Table ↔ intelligence pane interaction — clicking table row shows case header intelligence
- Detail slot switching — review context shows intelligence pane, queue/triage context shows pr-detail
- WebSocket trigger — simulated event triggers spine data re-fetch

### 6.3 Visual Verification

- Collapsed spine row renders status dots correctly
- Expanded spine shows event nodes with correct `●/◐/○` indicators
- Intelligence pane sections render shared primitives (trust bars, dimension grids, badges) correctly
- Spine/table toggle preserves selection state

## 7. Acceptance Criteria Mapping

| Criterion | How addressed |
|-----------|---------------|
| Timeline renders case lifecycle as clickable decision nodes | `review-spine.ts` with `spineNodeStrategy` — each event is a clickable node |
| Intelligence pane shows contributor, code, reviewer, policy context | `intelligence-pane.ts` case header view — parallel fetches for ContributorDetail, CodeAnalysisResult, routing policy |
| Dispatch nodes show trust-based selection rationale with alternatives | `intelligence-pane.ts` dispatch view — `GET /trust/{actorId}/routing-history/{entryId}` returns CandidateScore list |
| Findings nodes show precedent comparison from CBR | `intelligence-pane.ts` findings view — `similarCases` query, client-side confirmation rate |
| Human gate nodes show WorkItem status, what needs deciding, decision history | `intelligence-pane.ts` human gate view — triage data + contributor decision history |
| Contributor click-through shows full ContributorDetail including GitHubIntelligence | `GET /reviews/contributors/{actorId}` already returns full profile with `githubIntelligence` |
| Timeline updates in real-time via WebSocket events | `triggerUrl + triggerFilter` pattern in `datasets.ts` — existing WebSocket triggers re-fetch |
| Data from existing services — no new backend | All endpoints exist; no backend changes |

## 8. Decisions

14 decisions captured in `decisions.md` in this spec directory. Key decisions:

- **D1:** Spine is default view with table toggle (not spine-only)
- **D3:** Inline fetch with 30s dedup cache (not a data service layer)
- **D7:** Custom flat-div spine (not N agency-timeline instances)
- **D8:** Single intelligence pane with conditional render (not separate pane per node type)
- **D11:** pr-detail coexists for queue/triage/batch modes
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
- `app/src/main/java/io/casehub/devtown/app/mcp/MemoryQueryResolver.java` — similarCases CBR query
- Issue #216 — full requirements and acceptance criteria
- Visual companion mockups: `before-after-tabs.html`, `spine-collapse-expand.html`
