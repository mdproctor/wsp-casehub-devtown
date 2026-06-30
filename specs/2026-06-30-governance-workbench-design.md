# Governance Workbench — Design Spec

> **Date:** 2026-06-30
> **Issues:** #92 (Quinoa adoption), #85 (PR governance dashboard), #98 (trust visibility), #108 (batch failure rate trends)
> **Branch:** issue-85-pr-governance-dashboard

---

## Overview

A governance workbench for devtown — the operational UI for PR review coordination, trust-weighted routing, merge queue management, and human-in-the-loop oversight. Built on Quarkus Quinoa + casehub-pages.

**Audiences:** Operators (review queue management, system health, incident response) and contributors (PR status, review progress, routing transparency).

**Approach:** Progressive — DSL-first for all panels, promote to custom `hostPanel()` Web Components only when the DSL can't express the visualization. Designed against casehub-pages workbench primitives (`split()`, `dockBar()`, `hostPanel()`, event bus) as available (casehub-pages #64).

---

## Stack

| Layer | Technology |
|-------|-----------|
| Backend | Quarkus 3.32.2 + JAX-RS REST endpoints |
| Frontend build | Quinoa + esbuild (single `mvn quarkus:dev`) |
| UI framework | casehub-pages — TypeScript DSL, Web Components |
| Live data | WebSocket for event streams |
| On-demand data | REST polling via `dataset()` URLs |
| Theming | Pages theming API — dark default, light/dark toggle in topbar |

---

## Data Architecture

### Service Layer — GovernanceQueryService

The aggregation logic currently inline in `DevtownMcpTools` (queue status assembly, system health aggregation, reviewer health, problem detection) is extracted into `GovernanceQueryService` in `devtown-app`. This is the first implementation step — both MCP tools and REST endpoints consume the same service.

Methods:
- `queueStatus()` → builds `QueueStatus` from `PrReviewCaseTracker.activeCases()` with status counts
- `systemHealth()` → aggregates fleet size via `TrustExportService.exportAll()`, average trust per capability via `TrustGateService.allCapabilityScores()`, commitment count, work item count
- `reviewerHealth(actorId)` → open commitments, trust by capability/dimension, decision count, recent outcomes
- `reviewerFleet()` → all actors via `TrustExportService.exportAll()`, per-actor capability scores and maturity phase (derived from `minimumObservations` and decision count per `RoutingPolicy`)
- `problems(thresholdMinutes)` → stalled cases, expired commitments, worker failures, queue SLA breaches
- `reviewDetail(caseId)` → timeline from `CaseHubRuntime.eventLog()`, capability progress from case context
- `triageItems()` → pending human decisions filtered by `category` (see §Triage Filtering below)
- `mergeQueue()` → queued PRs and active batches from `MergeQueueService`
- `mergeQueueMetrics()` → queue depth, throughput, failure rate, failure rate timeseries (bucketed by hour from `MergeQueueStore.completedBatchesSince()`), adaptive sizing (`adaptiveMax` vs `maxBatchSize` per repository)
- `batchStatus(batchId)` → batch detail from `MergeQueueService`

Fleet enumeration uses `TrustExportService.exportAll(0.0)` — already proven in the MCP tools' `getSystemHealth()`. Maturity phase is computed in `GovernanceQueryService`, not the view: compare `TrustGateService.decisionCount(actorId, capability)` against `RoutingPolicy.minimumObservations()` to derive Bootstrap/Emerging/Active/Adaptive.

### REST Endpoints — GovernanceResource

Thin JAX-RS layer delegating to `GovernanceQueryService`. The MCP tool methods in `DevtownMcpTools` are refactored to delegate to the same service.

| Endpoint | Backing Service | MCP Equivalent |
|----------|----------------|----------------|
| `GET /api/governance/queue-status` | `GovernanceQueryService` | `get_queue_status` |
| `GET /api/governance/recent-events` | `GovernanceQueryService` | `get_recent_events` |
| `GET /api/governance/system-health` | `GovernanceQueryService` | `get_system_health` |
| `GET /api/governance/problems` | `GovernanceQueryService` | `list_problems` |
| `GET /api/governance/reviews` | `GovernanceQueryService` | (list) |
| `GET /api/governance/reviews/{caseId}` | `GovernanceQueryService` | `inspect_review` |
| `GET /api/governance/reviewers` | `GovernanceQueryService` | (fleet-wide) |
| `GET /api/governance/reviewers/{actorId}` | `GovernanceQueryService` | `get_reviewer_health` |
| `GET /api/governance/merge-queue` | `GovernanceQueryService` | `get_merge_queue` |
| `GET /api/governance/merge-queue/metrics` | `GovernanceQueryService` | `get_merge_queue_metrics` |
| `GET /api/governance/merge-queue/batch/{batchId}` | `GovernanceQueryService` | `get_batch_status` |
| `GET /api/governance/triage` | `GovernanceQueryService` | (pending human decisions) |

**Security:** All REST endpoints `@RolesAllowed(ADMIN)` initially. Contributor-scoped endpoints (filtered by PR author) are a future concern.

**Pagination:** List endpoints (`/reviews`, `/reviewers`, `/triage`, `/merge-queue`, `/problems`) use cursor-based pagination: `?cursor={opaqueToken}&limit={n}` (default limit 50, max 200). Response includes `nextCursor` (null when exhausted). Consistent with casehub-work's `WorkItemBulkResource` pagination convention.

### WebSocket Endpoint

`ws://host/api/governance/events` — live stream of governance events.

**Security:** WebSocket auth uses Quarkus HTTP path-based permissions — `quarkus.http.auth.permission.governance-ws.paths=/api/governance/events` with `policy=authenticated`. Auth happens at the HTTP upgrade handshake; `@RolesAllowed` (JAX-RS) does not apply to `@ServerEndpoint` classes.

**Bridge component:** `GovernanceEventBridge` (`@ApplicationScoped @ServerEndpoint("/api/governance/events")`) observes CDI events and forwards them as JSON to connected WebSocket sessions:

| CDI Event | WebSocket Topic | Source |
|-----------|----------------|--------|
| `CaseLifecycleEvent` | `case.state` | `casehub-engine` (`io.casehub.engine.common.spi.event`) — case state transitions |
| `WorkItemLifecycleEvent` | `workitem.lifecycle` | `casehub-work` (`io.casehub.work.runtime.event`) — work item state changes (PENDING → CLAIMED → COMPLETED) |
| `CommitmentDeclinedEvent` | `commitment.lifecycle` | `casehub-qhorus` (`io.casehub.qhorus.api.message`) — agent DECLINED a commitment |
| `CommitmentExpiredEvent` | `commitment.lifecycle` | `casehub-qhorus` (`io.casehub.qhorus.api.message`) — commitment expired without fulfilment |
| `TrustScoreActorUpdatedEvent` | `trust.update` | `casehub-ledger` (`io.casehub.ledger.runtime.service.routing`) — trust score materialisation |
| `SlaBreachEvent` | `sla.breach` | `casehub-work` (`io.casehub.work.runtime.event`) — SLA expiry |
| `MergeQueueStateEvent` (new) | `queue.state` | `MergeQueueService` — enqueue/dequeue/batch events (devtown#126) |

**Note on commitment events:** Qhorus publishes CDI events only for terminal/failure commitment transitions (`CommitmentDeclinedEvent`, `CommitmentExpiredEvent`). Happy-path transitions (ACKNOWLEDGED, FULFILLED) are reflected in `CaseLifecycleEvent` via case context updates. The bridge observes both topics — commitment failures are surfaced immediately; happy-path commitment progress is visible through case state changes.

**Note on merge queue events:** `MergeQueueService` currently uses imperative calls without CDI event publishing. `MergeQueueStateEvent` is a new CDI event to be fired on enqueue, dequeue, batch formation, and batch completion. Filed as devtown#126 — required for live merge queue updates in the Operations and Queue views.

**Message format:** JSON envelope compatible with casehub-pages `WebSocketSource` event dispatch:

```json
{ "op": "event", "topic": "case.state", "payload": { "caseId": "...", "status": "COMPLETED", "timestamp": "..." } }
```

The `op: "event"` field routes through the pages `processMessage` handler, which dispatches a `pages-event` DOM custom event on the container's `eventTarget`. Panels subscribe via the pages data pipeline — no duplicate connections.

**Session management:** `GovernanceEventBridge` maintains a `Set<Session>` (Quarkus WebSocket sessions). `@OnOpen` adds, `@OnClose`/`@OnError` removes. CDI observer methods (`@ObservesAsync`) iterate the session set and send JSON; failed sends trigger session removal. No backpressure beyond Quarkus WebSocket's built-in send buffering.

**Prerequisite:** casehub-pages#81 (Wire eventTarget from loadSite to WebSocketSource) must be closed for the `event` op to dispatch DOM events in production. The WebSocket endpoint and bridge work independently of #81 — the gap is on the pages consumer side. pages#81 is XS/Low complexity (one-line wiring fix).

### Data Durability

`PrReviewCaseTracker` is an in-memory materialised view (`ConcurrentHashMap` + bounded `Deque`). It loses state on restart. This is acceptable for the MCP tool use case but not for an operational dashboard.

**Startup hydration:** On application startup, `PrReviewCaseTracker` rebuilds from durable state:
- Active cases: query `CaseHubRuntime` for non-terminal `CaseInstance` records, reconstruct `CaseInfo` from each instance's `CaseContext` (contains `pr.*` fields)
- Event history: query `CaseLedgerEntry` records for recent cases to populate the event buffer

The durable source of truth is `casehub-engine` (case instances, event log) and `casehub-ledger` (audit entries). `PrReviewCaseTracker` is a read-optimised cache, not a persistence layer. Rebuilding from durable state is a `@Startup` CDI observer — no manual intervention needed.

---

## Views

Six top-level views, navigated via topbar. Each view picks the layout that suits it — workbench or web page.

**Navigation model:** `loadSite()` composes all views under a shared `topbar()` element with route entries for each view (`/ops`, `/reviews`, `/queue`, `/reviewers`, `/triage`, `/system`). Clean URL routing via History API — `quarkus.quinoa.enable-spa-routing=true` ensures the server returns `index.html` for all unmatched paths, and the client-side router resolves the view. The `dockBar()` (Operations) and `page()` (other views) are different component trees rendered under the same `topbar()` container — switching between them is a component swap, not a page load. `loadSite()` manages the lifecycle: unmount the current view tree, mount the new one. No multi-page loads, no iframe isolation.

### 1. Operations (`/ops`)

**Layout:** Workbench — `dockBar()` with left, right, bottom docks + center content.

**Left dock — Active Reviews:**
- `table()` with cross-filter enabled
- Columns: PR (`repo#number`), contributor, status, capability progress, started, last event
- Data: `GET /api/governance/queue-status`
- Click → populates center and right dock

**Left dock — Merge Queue** (below reviews):
- `table()` with cross-filter
- Columns: PR number, lane, author, trust score, wait time, status
- Data: `GET /api/governance/merge-queue`

**Center — default (nothing selected):**
- `metric()` cards: active cases, waiting on human, completed today, problems count
- `barChart()` or `timeseries()`: review throughput over 24h

**Center — review selected:**
- Review timeline — ordered events for the case. Start as `table()` of events with timestamp, type, actor, detail. Promote to `hostPanel()` if a vertical timeline with connecting lines is needed.
- Capability progress — `metric()` cards per capability, colour-coded by state (COMPLETED/IN_PROGRESS/SCHEDULED/FAILED).

**Right dock — context panel (review selected):**
- Routing rationale: `metric()` cards — reviewer trust score, threshold, phase, observations count
- Trust scores: `barChart()` — selected reviewer's trust by capability
- Commitment state: `metric()` — current obligation status

**Bottom dock — Event Stream:**
- WebSocket dataset rendered as scrolling `table()`
- Columns: timestamp, case ID, event type, actor, status
- Cross-filter listening — filters to selected case's events when a review is selected

### 2. Reviews (`/reviews`)

**Layout:** Web page — clean list with drill-down.

**List page:**
- `table()` — all reviews (active + completed). Sortable, filterable, paginated.
- Columns: PR, contributor, status, capabilities completed/total, started, duration, reviewer(s)
- `selector()` status filter: Active / Waiting / Completed / Faulted / Cancelled

**Detail page (`/reviews/{caseId}`):**

Full-page layout for understanding one review completely.

- **PR header:** `metric()` cards — repo, PR number, author, lines changed, status, duration
- **Capability progress:** `metric()` cards in `columns()` layout — one per capability, colour-coded by state. Threshold line visible.
- **Timeline + Routing:** `columns([60, 40], ...)`
  - Left: event timeline — `table()` or `hostPanel()` if visual timeline needed
  - Right: routing rationale — `metric()` cards (score, threshold, phase) + `barChart()` of alternatives considered
- **Compliance + Provenance:** `columns([30, 70], ...)`
  - Left: four `metric()` cards — audit chain, SLA, trust routing, GDPR. Green/red.
  - Right: PROV-DM causal chain. Candidate for `hostPanel()` (directed graph of linked entries). Fallback: `table()` flat list.

This page is where all three demo paths converge.

### 3. Merge Queue (`/queue`)

**Layout:** Web page — metrics, charts, tables.

**Metrics row:** `metric()` cards — queue depth, active batches, 24h throughput, failure rate, adaptive max vs max batch size (per repository).

**Charts row:** `columns([50, 50], ...)`
- Left: `barChart()` — wait time distribution by lane (NORMAL/HIGH/CRITICAL)
- Right: `timeseries()` — merge throughput over 24h

**Failure rate trend:** `timeseries()` — batch failure rate over time (computed from `MergeQueueStore.completedBatchesSince()` bucketed by hour). Fulfils #108 requirement for time-series failure rate visibility.

**Adaptive sizing:** `metric()` cards per repository — current `adaptiveMax` (from `recentBatchFailureRate` × decay) vs configured `maxBatchSize`. Shows how the system is auto-tuning batch sizes based on failure history.

**Queued PRs:** `table()` — lane-styled rows (CRITICAL red, HIGH amber). Columns: PR, author, lane, trust score, wait time, depends on.

**Active Batches:** `table()` — batch ID, PR count, risk level, dispatched time, status. Click → batch detail.

**Batch detail (`/queue/batch/{batchId}`):**
- PRs in batch: `table()` with per-PR trust score and status
- Batch timeline: `table()` of events
- Bisection state (if failed): candidate for `hostPanel()` later

### 4. Reviewers (`/reviewers`)

**Layout:** Web page — list with profile drill-down.

**List page:**
- `table()` — one row per reviewer agent
- Columns: agent ID, open commitments, trust score per capability (colour-coded: green above threshold, amber borderline, red below, grey bootstrap), total decisions, maturity phase
- Data: `GET /api/governance/reviewers` — fleet enumeration via `GovernanceQueryService.reviewerFleet()` (backed by `TrustExportService.exportAll()`; maturity phase derived from decision count vs `RoutingPolicy.minimumObservations()`)

**Profile page (`/reviewers/{actorId}`):**

- **Header:** `metric()` cards — agent name, phase, total decisions, open commitments
- **Trust charts:** `columns([50, 50], ...)`
  - Left: `barChart()` — trust per capability with threshold overlay
  - Right: `barChart()` — trust per dimension (thoroughness, false-positive-rate, scope-calibration)
- **Review history:** `table()` — recent outcomes with trust delta (+0.02, -0.15). Shows the feedback loop.
- **Active commitments:** `table()` — current obligations with SLA countdown (live-updating via WebSocket)
- **Incidents:** `table()` — FLAGGED attestations, severity, capability, trust impact

This page is the Demo Path A money shot — watch Agent B's trust score cross the threshold between two reviews.

### 5. Human Triage (`/triage`)

**Layout:** Web page — urgency-sorted list with SLA visibility.

**Metrics row:** `metric()` cards — pending decisions, SLA-breached (red), escalated (amber), approved in 24h.

**Pending decisions:** `table()` sorted by SLA urgency (most urgent first).
- Columns: PR, decision type (PR approval / action gate / routing oversight), candidate group, SLA remaining (colour-coded: red breached, amber <25%, green), escalation stage (Tier 1 / Tier 2 / —), age
- Click → navigates to `/reviews/{caseId}`

**Triage filtering:** Agent work is dispatched via qhorus COMMAND, not through `WorkItemStore` — scanning for `PENDING` work items already excludes agent tasks by design. The three decision types are distinguished by `WorkItem.category`:

- `human-decision` → `human-decision:pr-approval` — formal PR approval gate. Created by `HumanTaskScheduleHandler` with `category = "human-decision"`.
- `human-oversight` → `human-oversight:routing-review` — low-trust/borderline routing flagged for human confirmation. Created by the routing policy with `category = "human-oversight"`.
- `action-gate` → `ActionRiskClassifier`-flagged consequential actions. These are engine-level gates (`pendingActionGate` in-memory state, engine#433); the triage endpoint queries them via `GovernanceQueryService.pendingActionGates()` alongside WorkItem results and merges both into the triage list. Action gates are NOT WorkItems — they are engine state with a different lifecycle.

`GovernanceQueryService.triageItems()` combines: `WorkItemStore.scan(status=PENDING, category IN ("human-decision", "human-oversight"))` + engine pending action gates → unified triage list sorted by SLA urgency.

Demo Path B drives this view — SLA countdown, breach, escalation, resolution.

### 6. System (`/system`)

**Layout:** Simple web page — health overview.

**Health metrics:** `metric()` cards — active cases, fleet size, open commitments, pending work items.

**Body:** `columns([50, 50], ...)`
- Left: `barChart()` — fleet-wide average trust by capability with threshold overlay
- Right: `table()` — problems list sorted by severity (stalled, expired, breached, failed)

**Queue health:** `metric()` cards — queue depth, oldest wait, avg wait, failure rate.

---

## Quinoa Setup (#92)

Follows the casehub-pages quinoa-convention exactly:

1. Add `quarkus-quinoa` extension to `app/pom.xml`
2. Create `app/src/main/webui/` from `templates/quinoa-host/`
3. Configure `.npmrc` for `@casehubio` GitHub Packages scope
4. Configure Quinoa in `application.properties`:
   ```properties
   quarkus.quinoa.build-dir=dist
   quarkus.quinoa.package-manager-install=true
   quarkus.quinoa.enable-spa-routing=true
   ```
5. `package.json` dependencies:
   - `@casehubio/pages-runtime` (transitively provides pages-viz, pages-component, pages-data)
   - `@casehubio/pages-ui` (TypeScript DSL)
6. esbuild config: single entry point `src/index.ts` → `dist/app.js`
7. `index.html` in `webui/` with `<div id="app">` and `<script type="module" src="dist/app.js">`

### Garden Gotchas (carry forward)

- **GE-20260623-06914b:** esbuild drops `customElements.define()` from bare side-effect imports. Must use named imports from `@casehubio/pages-viz` — not bare `import '@casehubio/pages-viz'`.
- **GE-20260629-59c7e6:** esbuild minifies constants inside template literal strings. Avoid interpolating constants in `html()` component template literals — use runtime variables.

---

## TypeScript Structure

```
app/src/main/webui/src/
├── index.ts              # loadSite() entry point
├── views/
│   ├── operations.ts     # Workbench view — docks, splits, live stream
│   ├── reviews.ts        # Reviews list + detail
│   ├── queue.ts          # Merge queue list + batch detail
│   ├── reviewers.ts      # Reviewer list + profile
│   ├── triage.ts         # Human triage
│   └── system.ts         # System health
├── datasets.ts           # All dataset() and WebSocket source declarations
├── components/           # Custom hostPanel() Web Components (when needed)
│   ├── review-timeline.ts
│   └── provenance-chain.ts
└── theme.ts              # Dark/light theme configuration
```

Each view file exports a `page()` or component tree. `index.ts` composes them under top-level navigation.

---

## Custom Components (hostPanel candidates)

DSL-first for everything. Only these are identified as likely needing custom rendering:

| Component | Why DSL may not suffice | Fallback |
|-----------|------------------------|----------|
| Review timeline | Vertical timeline with connecting lines, milestone markers, branching events | `table()` of events works but is less visual |
| Provenance chain | Directed graph of PROV-DM entries linked by `causedByEntryId` | `table()` flat list works but loses the causal chain visual |
| Capability progress pipeline | Visual pipeline diagram showing flow from analysis → specialist checks → CI → merge | `metric()` cards work but a pipeline graphic is more compelling |

Decision on each is made during implementation — start with DSL, promote if needed.

---

## Demo Integration

The three demo paths from gastown-casehub-analysis-v4.md §8 drive the workbench:

| Demo Path | Primary View | What the audience sees |
|-----------|-------------|----------------------|
| A — Closed Feedback Loop | Operations → Reviewer profile | Trust score updating live. Agent B crosses threshold. Routing changes. |
| B — Compliance & Governance | Human Triage → Review detail | SLA countdown, breach, escalation, compliance report, GDPR erasure receipt. |
| C — AI-as-Operator | Operations (event stream) | MCP tool calls visible as events. AI monitoring and intervening in real time. |

The workbench turns the demo from "watch curl output" into "watch the system operate."

---

## Prerequisites

| Dependency | Status | Impact |
|-----------|--------|--------|
| casehub-pages#64 (workbench primitives) | ✅ Closed | `split()`, `dockBar()`, `hostPanel()`, event bus |
| casehub-pages#81 (Wire eventTarget from loadSite to WebSocketSource) | ⚠️ Open (XS/Low) | WebSocket event dispatch to DOM — required for live event stream panels |
| devtown#92 (Quinoa adoption) | ⚠️ Open | Foundational wiring for any UI |
| devtown#126 (CDI events from MergeQueueService) | ⚠️ Open (S/Low) | Required for live merge queue updates via WebSocket bridge |

## Scope Boundaries

**Supersede/relink (from #85):** Issue #85 has two requirements: (1) supersede/relink actions — backend case state transitions, audit trail links, and (2) dashboard visibility — case lifecycle views. This spec addresses requirement 2. Supersede/relink is a backend concern (new case state transition `SUPERSEDED`, `supersededBy`/`supersedes` audit trail links, new REST endpoint or MCP tool) that belongs in its own spec. Filed as devtown#124. The Reviews list and detail views are designed to display supersede/relink relationships when the backend supports them — the `predecessor` column and linked-case navigation are additive. Issue #85 cannot be fully closed without the backend work.

**GitHub integration (#15):** Listed as a prerequisite in #85. The governance workbench does not depend on #15 — it shows cases, not raw PRs. The "raw PR list" gap is noted in the Gastown Parity table and deferred to #15.

## Relationship to devtown-ui-requirements.md

`docs/devtown-ui-requirements.md` defines a four-phase panel taxonomy (Phase 1: demo-ready, Phase 2: operational depth, Phase 3: parity + depth, Phase 4: beyond Gastown). This spec implements the concrete design for Phases 1–2 and parts of Phase 3:

| This spec's view | UI requirements phase | Corresponding panels |
|------------------|----------------------|---------------------|
| Operations | Phase 1 + 2 | PR review timeline, trust score card, routing explanation, commitment timeline |
| Reviews | Phase 1 + 3 | PR review case, compliance report, PROV-DM provenance |
| Merge Queue | Phase 3 | Batch progress |
| Reviewers | Phase 2 | Reviewer profile, trust score card |
| Human Triage | Phase 2 | WorkItem inbox, action gate |
| System | Phase 2 + 3 | Fleet overview, fleet review capacity |

Issues #119–#123 (in the Extensibility section below) are Phase 2–3 features per that document. The two documents are complementary: the UI requirements doc defines the panel taxonomy and phasing; this spec is the concrete design for the first delivery.

## Extensibility

Future views and panels add to the navigation without restructuring:

| Issue | What it adds | Where |
|-------|-------------|-------|
| #119 | CasePlanModel browser | New view: `/models` |
| #120 | Case dependency graph (D3) | Panel in review detail or new view |
| #121 | Case memory browser | Panel in review detail or new view: `/memory` |
| #122 | Agent channel message inbox | New view: `/messages` |
| #123 | Worker session management | New view: `/workers` |

Each is a new `page()` export + navigation entry + REST endpoint. The architecture is additive.

---

## Gastown UI Parity

| gastown-gui panel | Covered | Notes |
|---|---|---|
| Dashboard | ✅ System view | |
| Rig list | ⚠️ Partial | Agent list in Reviewers; worker lifecycle deferred (#123) |
| Work list | ✅ Operations + Reviews | |
| PR list | ⚠️ Partial | Only PRs with active cases; raw PR list blocked on #15 |
| Mail inbox | ⚠️ Event stream | Dedicated inbox deferred (#122) |
| Health | ✅ System view | |
| Crew management | ❌ Deferred | #123 |
| Formula operations | ❌ Deferred | #119 |

| gastown-viewer-intent panel | Covered | Notes |
|---|---|---|
| Board (Kanban) | ✅ Reviews list | Status-filterable table (kanban layout possible later) |
| Graph | ❌ Deferred | #120 |
| Dashboard | ✅ Distributed | System + Queue + Reviewers |
| Memory panel | ❌ Deferred | #121 |
| Triage tab | ✅ Human Triage view | Formal obligations, not just labels |
| Sync pill | N/A | Doltgres P1.5 |

**Structural advantages shown that Gastown cannot:**
- Trust scores per agent per capability with live updates
- Commitment lifecycle with SLA and escalation
- Routing rationale (why agent A, not agent B)
- Compliance report (four regulatory dimensions)
- PROV-DM provenance chain
- Human triage with formal obligations (not notifications)
