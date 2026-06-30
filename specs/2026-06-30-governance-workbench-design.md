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

### REST Endpoints — GovernanceResource

Thin JAX-RS layer calling the same backing services as MCP tools. No bridge — shared service layer, two access protocols.

| Endpoint | Backing Service | MCP Equivalent |
|----------|----------------|----------------|
| `GET /api/governance/queue-status` | `PrReviewCaseTracker` | `get_queue_status` |
| `GET /api/governance/recent-events` | `PrReviewCaseTracker` | `get_recent_events` |
| `GET /api/governance/system-health` | Multiple (aggregation) | `get_system_health` |
| `GET /api/governance/problems` | `PrReviewCaseTracker` | `list_problems` |
| `GET /api/governance/reviews/{caseId}` | `PrReviewCaseTracker` | `inspect_review` |
| `GET /api/governance/reviewers` | `TrustGateService` + aggregation | New (fleet-wide) |
| `GET /api/governance/reviewers/{actorId}` | `TrustGateService` | `get_reviewer_health` |
| `GET /api/governance/merge-queue` | `MergeQueueService` | `get_merge_queue` |
| `GET /api/governance/merge-queue/metrics` | `MergeQueueService` | `get_merge_queue_metrics` |
| `GET /api/governance/merge-queue/batch/{batchId}` | `MergeQueueService` | `get_batch_status` |
| `GET /api/governance/triage` | `WorkItemService` | New (pending human decisions) |

All endpoints `@RolesAllowed(ADMIN)` initially. Contributor-scoped endpoints (filtered by PR author) are a future concern.

### WebSocket Endpoint

`ws://host/api/governance/events` — live stream of:
- Case state changes (RUNNING → WAITING → COMPLETED)
- Commitment lifecycle events (COMMAND → ACKNOWLEDGED → DONE)
- Trust score updates (agent, capability, old score, new score)
- SLA breach notifications
- Merge queue state changes

The WebSocket connection feeds a casehub-pages WebSocket dataset. Multiple panels subscribe to the same connection via the pages data pipeline — no duplicate connections.

---

## Views

Six top-level views, navigated via topbar. Each view picks the layout that suits it — workbench or web page.

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

**Metrics row:** `metric()` cards — queue depth, active batches, 24h throughput, failure rate.

**Charts row:** `columns([50, 50], ...)`
- Left: `barChart()` — wait time distribution by lane (NORMAL/HIGH/CRITICAL)
- Right: `timeseries()` — merge throughput over 24h

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
- Data: `GET /api/governance/reviewers` (new fleet-wide endpoint)

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

Decision types shown:
- `human-decision:pr-approval` — formal PR approval gate
- Action gates — `ActionRiskClassifier` flagged consequential action
- `human-oversight:routing-review` — low-trust/borderline routing flagged for human confirmation

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
