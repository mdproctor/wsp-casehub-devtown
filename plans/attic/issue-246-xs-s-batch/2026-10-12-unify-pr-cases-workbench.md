# Unify PR Cases Workbench — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #233 — Unify Reviews, Merge Queue, and Triage into a single PR Cases workbench
**Issue group:** #233

**Goal:** Replace three separate tabs (Reviews, Merge Queue, Triage) with a single "PR Cases" tab that shows all PR cases with filter chips, adaptive vitals, and a unified detail panel with lifecycle-appropriate actions and work-item alerts.

**Architecture:** A new `pr-cases-workbench.ts` LitElement web component replaces the existing `review-workbench.ts` and `merge-queue-workbench.ts`. It uses `blocks-split-workbench` for the list/detail layout, filter chips to switch between All/In Review/Needs Decision/Queued views, and a renamed `pr-detail.ts` (from `review-detail.ts`) that renders lifecycle-appropriate actions and work-item alerts. Data comes from three existing API endpoints merged client-side, with deduplication by caseId.

**Tech Stack:** TypeScript, Lit 3 (LitElement + decorators), `@casehubio/pages-table`, `@casehubio/blocks-ui-split-workbench`, `@casehubio/blocks-ui-blocks-timeline`, esbuild, mock-server.mjs

## Global Constraints

- All components use Lit 3 with `@customElement` decorator pattern
- CSS uses `var(--pages-*)` design tokens throughout — never hardcode colors
- `TypedDataSet` via `fromRows()` from `@casehubio/pages-data` for table data
- `columnId()` from `@casehubio/pages-data` for type-safe column identifiers
- No new npm dependencies — everything needed is already installed
- Verification: `npm run typecheck` from `app/src/main/webui/`
- Visual verification: `npm run dev:mock` from `app/src/main/webui/`
- IntelliJ MCP preferred for code navigation if available; Read/Edit/Write acceptable for TypeScript

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `src/components/pr-cases-workbench.ts` | Create | Unified workbench: filter chips, adaptive vitals, combined PR list, batch context panel, three-endpoint data merge |
| `src/components/pr-detail.ts` | Rename from `review-detail.ts` | Unified detail: lifecycle-adaptive actions, work-item alert banner, handles both review and queue contexts |
| `src/components/shared/work-item-alert.ts` | Create | Render function for human gate alert banner in detail panel |
| `src/components/shared/vitals.ts` | Create | Render functions for the four vitals row configurations (All, In Review, Needs Decision, Queued) |
| `src/components/shared/shared-styles.ts` | Modify | Add filter chip styles, work-item alert styles, vitals styles |
| `src/index.ts` | Modify | Replace Reviews/Merge Queue/Triage tabs with single PR Cases tab; update panel registration |
| `mock-fixtures.json` | Modify | Add `/api/devtown/governance/triage` work-item detail fields for the alert banner |
| `mock-server.mjs` | Modify | Add triage item lookup by caseId for work-item alert fetch |
| `src/components/merge-queue-detail.ts` | Delete (end) | Retired — pr-detail handles all stages |
| `src/components/review-workbench.ts` | Delete (end) | Retired — pr-cases-workbench replaces it |
| `src/components/merge-queue-workbench.ts` | Delete (end) | Retired — pr-cases-workbench replaces it |
| `src/views/reviews.ts` | Delete (end) | Unused (was already imported but not wired) |
| `src/views/triage.ts` | Delete (end) | Retired — Needs Decision filter replaces it |

---

## Batch 1: Shared infrastructure — vitals render functions, work-item alert, filter chip styles

After this batch: new shared composables exist, typecheck passes, existing tabs still work unchanged.

### Task 1: Work-item alert composable and vitals render functions

**Files:**
- Create: `app/src/main/webui/src/components/shared/work-item-alert.ts`
- Create: `app/src/main/webui/src/components/shared/vitals.ts`
- Modify: `app/src/main/webui/src/components/shared/shared-styles.ts`
- Modify: `app/src/main/webui/src/components/shared/types.ts`

**Interfaces:**
- Produces: `renderWorkItemAlert(item: WorkItemData): TemplateResult | typeof nothing` — renders the human gate alert banner
- Produces: `renderVitals(config: VitalEntry[]): TemplateResult` — renders a row of vital stat tiles
- Produces: `VitalEntry` type: `{ label: string; value: string | number }`
- Produces: `WorkItemData` type: `{ workItemId: string; decisionType: string; candidateGroup: string; expiresAt: string; escalationStage: string; status: string }`
- Produces: CSS exports: `workItemAlertStyles`, `vitalsStyles`, `filterChipStyles`

- [ ] **Step 1: Create work-item-alert.ts with the render function**

```typescript
// app/src/main/webui/src/components/shared/work-item-alert.ts
import { html, nothing } from 'lit';
import type { TemplateResult } from 'lit';

export interface WorkItemData {
  workItemId: string;
  decisionType: string;
  candidateGroup: string;
  expiresAt: string;
  escalationStage: string;
  status: string;
}

function formatTimeRemaining(expiresAt: string): string {
  const diff = new Date(expiresAt).getTime() - Date.now();
  if (diff <= 0) return 'EXPIRED';
  const hours = Math.floor(diff / 3600000);
  const mins = Math.round((diff % 3600000) / 60000);
  if (hours > 0) return `${hours}h ${mins}m remaining`;
  return `${mins}m remaining`;
}

export function renderWorkItemAlert(item: WorkItemData | null): TemplateResult | typeof nothing {
  if (!item) return nothing;
  const isEscalated = item.escalationStage !== 'initial';
  return html`
    <div class="work-item-alert ${isEscalated ? 'escalated' : ''}">
      <div class="wia-header">
        <span class="wia-icon">${isEscalated ? '🔴' : '⚠️'}</span>
        <span class="wia-title">Human decision required</span>
      </div>
      <div class="wia-details">
        <div class="wia-row"><span class="wia-label">Type</span><span>${item.decisionType}</span></div>
        <div class="wia-row"><span class="wia-label">Assigned to</span><span>${item.candidateGroup}</span></div>
        <div class="wia-row"><span class="wia-label">Expires</span><span>${formatTimeRemaining(item.expiresAt)}</span></div>
        <div class="wia-row"><span class="wia-label">Escalation</span><span>${item.escalationStage}</span></div>
      </div>
    </div>
  `;
}
```

- [ ] **Step 2: Create vitals.ts with the render function**

```typescript
// app/src/main/webui/src/components/shared/vitals.ts
import { html } from 'lit';
import type { TemplateResult } from 'lit';

export interface VitalEntry {
  label: string;
  value: string | number;
}

export function renderVitals(entries: VitalEntry[]): TemplateResult {
  return html`
    <div class="vitals" style="grid-template-columns: repeat(${Math.min(entries.length, 6)}, 1fr)">
      ${entries.map(e => html`
        <div class="vital">
          <div class="vital-value">${e.value}</div>
          <div class="vital-label">${e.label}</div>
        </div>
      `)}
    </div>
  `;
}
```

- [ ] **Step 3: Add work-item-alert, vitals, and filter-chip styles to shared-styles.ts**

Append to `app/src/main/webui/src/components/shared/shared-styles.ts`:

```typescript
export const workItemAlertStyles = css`
  .work-item-alert {
    padding: 10px 14px; margin: 0 0 16px; border-radius: 6px;
    background: var(--pages-warning-2, #fffbeb);
    border: 1px solid var(--pages-warning-6, #d97706);
    font-size: 12px;
  }
  .work-item-alert.escalated {
    background: var(--pages-danger-2, #fef2f2);
    border-color: var(--pages-danger-6, #dc2626);
  }
  .wia-header { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; font-weight: 600; font-size: 13px; }
  .wia-details { display: grid; grid-template-columns: auto 1fr; gap: 2px 12px; }
  .wia-row { display: contents; }
  .wia-label { font-weight: 500; color: var(--pages-neutral-7, #525252); }
`;

export const vitalsStyles = css`
  .vitals {
    display: grid; gap: 6px;
    padding: 10px 0; border-bottom: 1px solid var(--pages-neutral-4, #d4d4d4);
  }
  .vital { text-align: center; padding: 6px; }
  .vital-value { font-size: 20px; font-weight: 700; }
  .vital-label { font-size: 10px; color: var(--pages-neutral-7, #525252); text-transform: uppercase; }
`;

export const filterChipStyles = css`
  .filter-chips {
    display: flex; gap: 6px; padding: 10px 0;
    border-bottom: 1px solid var(--pages-neutral-4, #d4d4d4);
  }
  .filter-chip {
    padding: 4px 12px; border-radius: 16px; font-size: 12px; font-weight: 500;
    cursor: pointer; border: 1px solid var(--pages-neutral-4, #d4d4d4);
    background: white; color: var(--pages-neutral-9, #171717);
    transition: all 0.15s;
  }
  .filter-chip:hover { background: var(--pages-neutral-2, #f5f5f5); }
  .filter-chip.active {
    background: var(--pages-primary-9, #1d4ed8); color: white;
    border-color: var(--pages-primary-9, #1d4ed8);
  }
  .filter-count {
    margin-left: 4px; font-size: 11px; opacity: 0.8;
  }
`;
```

- [ ] **Step 4: Update types.ts barrel export**

Add to `app/src/main/webui/src/components/shared/types.ts`:

```typescript
export type { WorkItemData } from './work-item-alert.js';
export type { VitalEntry } from './vitals.js';
```

- [ ] **Step 5: Run typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: PASS — no errors. Existing components unchanged.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/webui/src/components/shared/
git commit -m "feat(#233): add work-item-alert, vitals, and filter-chip shared composables

Refs #233"
```

---

## Batch 2: Unified workbench — pr-cases-workbench with filter chips, data merge, and adaptive vitals

After this batch: new PR Cases workbench renders alongside existing tabs (registered as additional tab for testing). Filter chips switch views, vitals adapt, data merges from three endpoints, batch context shows under Queued filter.

### Task 2: Create pr-cases-workbench.ts with data merge and filter chip logic

**Files:**
- Create: `app/src/main/webui/src/components/pr-cases-workbench.ts`

**Interfaces:**
- Consumes: `renderVitals(entries: VitalEntry[]): TemplateResult` from `shared/vitals.ts`
- Consumes: `filterChipStyles`, `vitalsStyles` from `shared/shared-styles.ts`
- Consumes: `fromRows(rows, columns): TypedDataSet` from `@casehubio/pages-data`
- Consumes: `columnId(name): ColumnId` from `@casehubio/pages-data`
- Produces: `<devtown-pr-cases-workbench endpoint="...">` web component
- Produces: Selection events via `emitPagesEvent(document, 'review:selected', { caseId })` — reuses existing review topic

- [ ] **Step 1: Create pr-cases-workbench.ts**

This is the core file. It fetches from three endpoints, merges/deduplicates by caseId, and renders filter chips, adaptive vitals, the PR list table (with filter-driven column config), the problems banner, and a collapsible batch context panel under the Queued filter.

```typescript
// app/src/main/webui/src/components/pr-cases-workbench.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { columnId, ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';
import type { TypedDataSet } from '@casehubio/pages-data/dist/dataset/types.js';
import { fromRows } from '@casehubio/pages-data/dist/dataset/conversion.js';
import type { TableColumnConfig } from '@casehubio/pages-table';
import { emitPagesEvent, onPagesEvent } from '@casehubio/blocks-ui-core';
import { renderVitals } from './shared/vitals.js';
import type { VitalEntry } from './shared/vitals.js';
import '@casehubio/pages-table';
import '@casehubio/blocks-ui-split-workbench';
import './pr-detail.js';

// --- Interfaces ---

interface ReviewEntry {
  caseId: string;
  prNumber: number;
  repo: string;
  contributor: string;
  status: string;
  linesChanged: number;
  startedAt: string;
  lastEventAt: string;
}

interface QueuedPrEntry {
  number: number;
  repository: string;
  author: string;
  headSha: string;
  priorityLane: string;
  trustScore: number;
  waitMinutes: number;
  enqueuedAt: string;
  dependsOn: number[];
}

interface TriageItem {
  workItemId: string;
  prRef: string;
  decisionType: string;
  candidateGroup: string;
  expiresAt: string;
  escalationStage: string;
  status: string;
  createdAt: string;
  caseId: string;
}

interface ActiveBatchEntry {
  caseId: string;
  batchId: string;
  prCount: number;
  riskLevel: string;
  ciStatus?: string;
  prNumbers?: number[];
  startedAt?: string;
  suspectedPr?: number;
}

interface SystemHealth {
  activeCases: number;
  fleetSize: number;
  openCommitments: number;
  pendingWorkItems: number;
}

interface MergeQueueMetrics {
  queueDepth: number;
  activeBatches: number;
  throughput24h: number;
  failureRate: number;
  oldestWaitMinutes: number;
  avgWaitMinutes: number;
}

interface Problem {
  category: string;
  severity: string;
  description: string;
  caseId: string | null;
}

// --- Unified row for the combined list ---

interface UnifiedPrRow {
  caseId: string;
  prNumber: number;
  repo: string;
  author: string;
  stage: string; // REVIEW | HUMAN_GATE | QUEUED | BATCHED
  status: string;
  linesChanged: number;
  // Queue-specific
  priorityLane: string;
  trustScore: number;
  waitMinutes: number;
  headSha: string;
  dependsOn: number[];
  // Triage-specific
  decisionType: string;
  candidateGroup: string;
  expiresAt: string;
  escalationStage: string;
}

type FilterKey = 'all' | 'review' | 'decision' | 'queued';

// --- Column definitions per filter ---

const COL_PR = columnId('prNumber');
const COL_REPO = columnId('repo');
const COL_AUTHOR = columnId('author');
const COL_STAGE = columnId('stage');
const COL_STATUS = columnId('status');
const COL_LINES = columnId('linesChanged');
const COL_LANE = columnId('priorityLane');
const COL_TRUST = columnId('trustScore');
const COL_WAIT = columnId('waitMinutes');
const COL_DECISION_TYPE = columnId('decisionType');
const COL_CANDIDATE = columnId('candidateGroup');
const COL_EXPIRES = columnId('expiresAt');
const COL_ESCALATION = columnId('escalationStage');

const ALL_COLUMN_DEFS = [
  { id: COL_PR, name: 'PR', type: ColumnType.NUMBER, getValue: (r: UnifiedPrRow) => r.prNumber },
  { id: COL_REPO, name: 'Repo', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.repo },
  { id: COL_AUTHOR, name: 'Author', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.author },
  { id: COL_STAGE, name: 'Stage', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.stage },
  { id: COL_STATUS, name: 'Status', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.status },
  { id: COL_LINES, name: 'Lines', type: ColumnType.NUMBER, getValue: (r: UnifiedPrRow) => r.linesChanged },
  { id: COL_LANE, name: 'Lane', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.priorityLane },
  { id: COL_TRUST, name: 'Trust', type: ColumnType.NUMBER, getValue: (r: UnifiedPrRow) => r.trustScore },
  { id: COL_WAIT, name: 'Wait (min)', type: ColumnType.NUMBER, getValue: (r: UnifiedPrRow) => r.waitMinutes },
  { id: COL_DECISION_TYPE, name: 'Decision', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.decisionType },
  { id: COL_CANDIDATE, name: 'Candidate Group', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.candidateGroup },
  { id: COL_EXPIRES, name: 'Expires', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.expiresAt },
  { id: COL_ESCALATION, name: 'Escalation', type: ColumnType.TEXT, getValue: (r: UnifiedPrRow) => r.escalationStage },
];

const FILTER_COLUMNS: Record<FilterKey, readonly TableColumnConfig[]> = {
  all: [
    { id: COL_PR, sortable: true },
    { id: COL_REPO, sortable: true },
    { id: COL_AUTHOR, sortable: true },
    { id: COL_STAGE, sortable: true },
    { id: COL_STATUS, sortable: true },
  ],
  review: [
    { id: COL_PR, sortable: true },
    { id: COL_REPO, sortable: true },
    { id: COL_AUTHOR, sortable: true },
    { id: COL_STATUS, sortable: true },
    { id: COL_LINES, sortable: true },
  ],
  decision: [
    { id: COL_PR, sortable: true },
    { id: COL_REPO, sortable: true },
    { id: COL_AUTHOR, sortable: true },
    { id: COL_DECISION_TYPE, sortable: true },
    { id: COL_CANDIDATE, sortable: true },
    { id: COL_EXPIRES, sortable: true },
    { id: COL_ESCALATION, sortable: true },
  ],
  queued: [
    { id: COL_PR, sortable: true },
    { id: COL_REPO, sortable: true },
    { id: COL_AUTHOR, sortable: true },
    { id: COL_LANE, sortable: true },
    { id: COL_TRUST, sortable: true },
    { id: COL_WAIT, sortable: true },
  ],
};

// --- Batch table columns ---

const BATCH_ID_COL = columnId('batchId');
const BATCH_PRS_COL = columnId('prList');
const BATCH_CI_COL = columnId('ciStatus');
const BATCH_RISK_COL = columnId('riskLevel');

const BATCH_COLUMNS = [
  { id: BATCH_ID_COL, name: 'ID', type: ColumnType.TEXT, getValue: (r: ActiveBatchEntry) => r.batchId },
  { id: BATCH_PRS_COL, name: 'PRs', type: ColumnType.TEXT, getValue: (r: ActiveBatchEntry) => (r.prNumbers ?? []).map(n => `#${n}`).join(', ') || `${r.prCount} PRs` },
  { id: BATCH_CI_COL, name: 'CI', type: ColumnType.TEXT, getValue: (r: ActiveBatchEntry) => r.ciStatus ?? '—' },
  { id: BATCH_RISK_COL, name: 'Risk', type: ColumnType.TEXT, getValue: (r: ActiveBatchEntry) => r.riskLevel },
];

const BATCH_TABLE_CONFIG: readonly TableColumnConfig[] = [
  { id: BATCH_ID_COL, sortable: false, visible: false },
  { id: BATCH_PRS_COL, sortable: false },
  { id: BATCH_CI_COL, sortable: true },
  { id: BATCH_RISK_COL, sortable: true },
];

@customElement('devtown-pr-cases-workbench')
export class PrCasesWorkbench extends LitElement {
  @property({ type: String }) endpoint = '';

  @state() private _filter: FilterKey = 'all';
  @state() private _selectedCaseId = '';
  @state() private _selectionKind: 'pr' | 'batch' = 'pr';

  // Raw data
  @state() private _reviews: ReviewEntry[] = [];
  @state() private _queuedPrs: QueuedPrEntry[] = [];
  @state() private _triageItems: TriageItem[] = [];
  @state() private _batches: ActiveBatchEntry[] = [];
  @state() private _health: SystemHealth = { activeCases: 0, fleetSize: 0, openCommitments: 0, pendingWorkItems: 0 };
  @state() private _metrics: MergeQueueMetrics = { queueDepth: 0, activeBatches: 0, throughput24h: 0, failureRate: 0, oldestWaitMinutes: 0, avgWaitMinutes: 0 };
  @state() private _problems: Problem[] = [];

  // Derived
  @state() private _unified: UnifiedPrRow[] = [];
  @state() private _listData: TypedDataSet | undefined;
  @state() private _batchData: TypedDataSet | undefined;
  @state() private _batchesExpanded = true;

  // For merge-queue detail: pass selected batch
  @state() private _selectedBatch: ActiveBatchEntry | null = null;

  private _unsubs: Array<() => void> = [];

  static override styles = css`
    :host { display: block; height: 100%; font-family: var(--pages-font-family, system-ui); }
    blocks-split-workbench { height: 100%; }
    .list-panel { height: 100%; overflow: auto; padding: 0 12px; }
    .detail-panel { height: 100%; }
    .filter-chips {
      display: flex; gap: 6px; padding: 10px 0;
      border-bottom: 1px solid var(--pages-neutral-4, #d4d4d4);
    }
    .filter-chip {
      padding: 4px 12px; border-radius: 16px; font-size: 12px; font-weight: 500;
      cursor: pointer; border: 1px solid var(--pages-neutral-4, #d4d4d4);
      background: white; color: var(--pages-neutral-9, #171717);
      transition: all 0.15s;
    }
    .filter-chip:hover { background: var(--pages-neutral-2, #f5f5f5); }
    .filter-chip.active {
      background: var(--pages-primary-9, #1d4ed8); color: white;
      border-color: var(--pages-primary-9, #1d4ed8);
    }
    .filter-count { margin-left: 4px; font-size: 11px; opacity: 0.8; }
    .vitals {
      display: grid; gap: 6px;
      padding: 10px 0; border-bottom: 1px solid var(--pages-neutral-4, #d4d4d4);
    }
    .vital { text-align: center; padding: 6px; }
    .vital-value { font-size: 20px; font-weight: 700; }
    .vital-label { font-size: 10px; color: var(--pages-neutral-7, #525252); text-transform: uppercase; }
    .problems-banner {
      padding: 8px 12px; margin: 8px 0;
      background: var(--pages-warning-3, #fef3c7); border: 1px solid var(--pages-warning-6, #d97706);
      border-radius: 6px; font-size: 12px;
    }
    .problem-item { padding: 3px 0; display: flex; gap: 8px; align-items: center; cursor: pointer; }
    .problem-item:hover { text-decoration: underline; }
    .problem-severity {
      font-size: 10px; font-weight: 600; padding: 1px 5px; border-radius: 3px;
      background: var(--pages-warning-9, #92400e); color: white;
    }
    .problem-severity.error { background: var(--pages-danger-9, #dc2626); }
    h3 { font-size: 13px; font-weight: 600; margin: 12px 0 6px; color: var(--pages-neutral-9, #404040); }
    h3.section-first { margin-top: 4px; }
    .batch-toggle {
      font-size: 12px; cursor: pointer; border: none; background: none;
      color: var(--pages-neutral-9, #404040); font-weight: 600; padding: 0;
    }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this._fetchAll();
    this._unsubs.push(
      onPagesEvent(document, 'review:deselected', () => {
        this._selectedCaseId = '';
        this._selectedBatch = null;
      }),
    );
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._unsubs.forEach(u => u());
    this._unsubs = [];
  }

  configure(props: Record<string, unknown>): void {
    if (props.endpoint) this.endpoint = String(props.endpoint);
  }

  // --- Data fetching ---

  private async _fetchAll(): Promise<void> {
    try {
      const [qsRes, mqRes, trRes, hRes, metRes, pRes] = await Promise.all([
        fetch(`${this.endpoint}/queue-status`).catch(() => null),
        fetch(`${this.endpoint}/merge-queue`).catch(() => null),
        fetch(`${this.endpoint}/triage`).catch(() => null),
        fetch(`${this.endpoint}/system-health`).catch(() => null),
        fetch(`${this.endpoint}/merge-queue/metrics`).catch(() => null),
        fetch(`${this.endpoint}/problems?threshold_minutes=0`).catch(() => null),
      ]);

      if (qsRes?.ok) {
        const json = await qsRes.json();
        this._reviews = json.reviews ?? [];
      }
      if (mqRes?.ok) {
        const json = await mqRes.json();
        this._queuedPrs = json.queuedPrs ?? [];
        this._batches = json.activeBatches ?? [];
        this._batchData = fromRows([...this._batches], BATCH_COLUMNS);
      }
      if (trRes?.ok) {
        const json = await trRes.json();
        this._triageItems = json.items ?? [];
      }
      if (hRes?.ok) this._health = await hRes.json();
      if (metRes?.ok) this._metrics = await metRes.json();
      if (pRes?.ok) this._problems = (await pRes.json()).items ?? [];

      this._mergeData();
    } catch (err) {
      console.warn('Failed to fetch PR cases data', err);
    }
  }

  private _mergeData(): void {
    const byCase = new Map<string, UnifiedPrRow>();
    const triageByCaseId = new Map<string, TriageItem>();
    for (const t of this._triageItems) {
      triageByCaseId.set(t.caseId, t);
    }

    // Queued PRs — need a caseId. Map by prNumber from reviews.
    const reviewByCaseId = new Map<string, ReviewEntry>();
    for (const r of this._reviews) {
      reviewByCaseId.set(r.caseId, r);
    }

    // Build from reviews first
    for (const r of this._reviews) {
      const triage = triageByCaseId.get(r.caseId);
      const isQueued = this._queuedPrs.some(q => q.number === r.prNumber && q.repository === r.repo);
      byCase.set(r.caseId, {
        caseId: r.caseId,
        prNumber: r.prNumber,
        repo: r.repo,
        author: r.contributor,
        stage: triage ? 'HUMAN_GATE' : isQueued ? 'QUEUED' : 'REVIEW',
        status: r.status,
        linesChanged: r.linesChanged,
        priorityLane: '',
        trustScore: 0,
        waitMinutes: 0,
        headSha: '',
        dependsOn: [],
        decisionType: triage?.decisionType ?? '',
        candidateGroup: triage?.candidateGroup ?? '',
        expiresAt: triage?.expiresAt ?? '',
        escalationStage: triage?.escalationStage ?? '',
      });
    }

    // Enrich from queued PRs (or add if not in reviews)
    for (const q of this._queuedPrs) {
      const existing = [...byCase.values()].find(r => r.prNumber === q.number && r.repo === q.repository);
      if (existing) {
        existing.stage = 'QUEUED';
        existing.priorityLane = q.priorityLane;
        existing.trustScore = q.trustScore;
        existing.waitMinutes = q.waitMinutes;
        existing.headSha = q.headSha;
        existing.dependsOn = q.dependsOn;
      } else {
        const fakeId = `queue-${q.repository}-${q.number}`;
        byCase.set(fakeId, {
          caseId: fakeId,
          prNumber: q.number,
          repo: q.repository,
          author: q.author,
          stage: 'QUEUED',
          status: 'QUEUED',
          linesChanged: 0,
          priorityLane: q.priorityLane,
          trustScore: q.trustScore,
          waitMinutes: q.waitMinutes,
          headSha: q.headSha,
          dependsOn: q.dependsOn,
          decisionType: '',
          candidateGroup: '',
          expiresAt: '',
          escalationStage: '',
        });
      }
    }

    // Mark batched PRs
    const batchedPrs = new Set<number>();
    for (const b of this._batches) {
      for (const n of (b.prNumbers ?? [])) batchedPrs.add(n);
    }
    for (const row of byCase.values()) {
      if (batchedPrs.has(row.prNumber) && row.stage === 'QUEUED') {
        row.stage = 'BATCHED';
      }
    }

    // Handle triage items not matched to reviews
    for (const t of this._triageItems) {
      if (!byCase.has(t.caseId)) {
        const parts = t.prRef.split('#');
        const repo = parts[0] ?? '';
        const prNum = parseInt(parts[1] ?? '0', 10);
        byCase.set(t.caseId, {
          caseId: t.caseId,
          prNumber: prNum,
          repo,
          author: '',
          stage: 'HUMAN_GATE',
          status: 'WAITING',
          linesChanged: 0,
          priorityLane: '',
          trustScore: 0,
          waitMinutes: 0,
          headSha: '',
          dependsOn: [],
          decisionType: t.decisionType,
          candidateGroup: t.candidateGroup,
          expiresAt: t.expiresAt,
          escalationStage: t.escalationStage,
        });
      }
    }

    this._unified = [...byCase.values()];
    this._rebuildTable();
  }

  // --- Filtering ---

  private _filteredRows(): UnifiedPrRow[] {
    switch (this._filter) {
      case 'review': return this._unified.filter(r => r.stage === 'REVIEW' || r.stage === 'HUMAN_GATE');
      case 'decision': return this._unified.filter(r => r.stage === 'HUMAN_GATE');
      case 'queued': return this._unified.filter(r => r.stage === 'QUEUED' || r.stage === 'BATCHED');
      default: return this._unified;
    }
  }

  private _rebuildTable(): void {
    const rows = this._filteredRows();
    this._listData = fromRows([...rows], ALL_COLUMN_DEFS);
  }

  private _filterCounts(): Record<FilterKey, number> {
    return {
      all: this._unified.length,
      review: this._unified.filter(r => r.stage === 'REVIEW' || r.stage === 'HUMAN_GATE').length,
      decision: this._unified.filter(r => r.stage === 'HUMAN_GATE').length,
      queued: this._unified.filter(r => r.stage === 'QUEUED' || r.stage === 'BATCHED').length,
    };
  }

  private _setFilter(f: FilterKey): void {
    this._filter = f;
    this._rebuildTable();
  }

  // --- Vitals ---

  private _vitals(): VitalEntry[] {
    const h = this._health;
    const m = this._metrics;
    switch (this._filter) {
      case 'review':
        return [
          { label: 'Active', value: h.activeCases },
          { label: 'Fleet', value: h.fleetSize },
          { label: 'Commits', value: h.openCommitments },
          { label: 'Work Items', value: h.pendingWorkItems },
        ];
      case 'decision': {
        const escalated = this._triageItems.filter(t => t.escalationStage !== 'initial').length;
        return [
          { label: 'Pending', value: this._triageItems.length },
          { label: 'Escalated', value: escalated },
        ];
      }
      case 'queued':
        return [
          { label: 'Depth', value: m.queueDepth },
          { label: 'Batches', value: m.activeBatches },
          { label: '24h Thru', value: m.throughput24h },
          { label: 'Fail Rate', value: `${Math.round(m.failureRate * 100)}%` },
          { label: 'Oldest (m)', value: m.oldestWaitMinutes },
          { label: 'Avg (m)', value: m.avgWaitMinutes },
        ];
      default:
        return [
          { label: 'Active', value: h.activeCases },
          { label: 'Queued', value: m.queueDepth },
          { label: 'Decisions', value: this._triageItems.length },
          { label: 'Fleet', value: h.fleetSize },
        ];
    }
  }

  // --- Event handlers ---

  private _handleRowActivate = (e: Event): void => {
    const detail = (e as CustomEvent).detail;
    if (detail?.row) {
      const prNum = detail.row.number(COL_PR);
      const entry = this._filteredRows().find(r => r.prNumber === prNum);
      if (entry) {
        this._selectionKind = 'pr';
        this._selectedCaseId = entry.caseId;
        this._selectedBatch = null;
        emitPagesEvent(document, 'review:selected', { caseId: entry.caseId });
      }
    }
  };

  private _handleBatchRowActivate = (e: Event): void => {
    const detail = (e as CustomEvent).detail;
    if (detail?.row) {
      const batchId = detail.row.text(BATCH_ID_COL);
      const entry = this._batches.find(b => b.batchId === batchId);
      if (entry) {
        this._selectionKind = 'batch';
        this._selectedBatch = entry;
        this._selectedCaseId = '';
      }
    }
  };

  private _handleProblemClick(p: Problem): void {
    if (p.caseId) {
      this._selectionKind = 'pr';
      this._selectedCaseId = p.caseId;
      this._selectedBatch = null;
      emitPagesEvent(document, 'review:selected', { caseId: p.caseId });
    }
  }

  // --- Render ---

  override render() {
    const counts = this._filterCounts();
    const vitals = this._vitals();
    const selectedRow = this._unified.find(r => r.caseId === this._selectedCaseId) ?? null;

    return html`
      <blocks-split-workbench selection-topic="review" title="PR Cases">
        <div slot="list" class="list-panel">
          <div class="filter-chips">
            ${this._renderChip('all', 'All', counts.all)}
            ${this._renderChip('review', 'In Review', counts.review)}
            ${this._renderChip('decision', 'Needs Decision', counts.decision)}
            ${this._renderChip('queued', 'Queued', counts.queued)}
          </div>

          ${renderVitals(vitals)}

          ${this._problems.length > 0 ? html`
            <div class="problems-banner">
              ${this._problems.map(p => html`
                <div class="problem-item" @click=${() => this._handleProblemClick(p)}>
                  <span class="problem-severity ${p.severity === 'error' ? 'error' : ''}">${p.severity}</span>
                  <span>${p.description}</span>
                </div>
              `)}
            </div>
          ` : nothing}

          <h3>${this._filterLabel()} (${this._filteredRows().length})</h3>
          ${this._listData ? html`
            <pages-table
              .dataSet=${this._listData}
              .columnConfig=${FILTER_COLUMNS[this._filter]}
              @row-activate=${this._handleRowActivate}
            ></pages-table>
          ` : nothing}

          ${this._filter === 'queued' && this._batches.length > 0 ? html`
            <h3>
              <button class="batch-toggle" @click=${() => { this._batchesExpanded = !this._batchesExpanded; }}>
                ${this._batchesExpanded ? '▾' : '▸'} Active Batches (${this._batches.length})
              </button>
            </h3>
            ${this._batchesExpanded && this._batchData ? html`
              <pages-table
                .dataSet=${this._batchData}
                .columnConfig=${BATCH_TABLE_CONFIG}
                @row-activate=${this._handleBatchRowActivate}
              ></pages-table>
            ` : nothing}
          ` : nothing}
        </div>
        <div slot="detail" class="detail-panel">
          ${this._selectionKind === 'batch' && this._selectedBatch
            ? html`<devtown-pr-detail
                mode="batch"
                endpoint="${this.endpoint}"
                .batch=${this._selectedBatch}
              ></devtown-pr-detail>`
            : html`<devtown-pr-detail
                case-id=${this._selectedCaseId}
                endpoint="/api/devtown/reviews"
                .stage=${selectedRow?.stage ?? ''}
                .queueContext=${{ priorityLane: selectedRow?.priorityLane ?? '', trustScore: selectedRow?.trustScore ?? 0, waitMinutes: selectedRow?.waitMinutes ?? 0, headSha: selectedRow?.headSha ?? '', dependsOn: selectedRow?.dependsOn ?? [] }}
                .triageContext=${{ caseId: selectedRow?.caseId ?? '', decisionType: selectedRow?.decisionType ?? '', candidateGroup: selectedRow?.candidateGroup ?? '', expiresAt: selectedRow?.expiresAt ?? '', escalationStage: selectedRow?.escalationStage ?? '' }}
              ></devtown-pr-detail>`
          }
        </div>
      </blocks-split-workbench>
    `;
  }

  private _renderChip(key: FilterKey, label: string, count: number) {
    return html`
      <button class="filter-chip ${this._filter === key ? 'active' : ''}"
              @click=${() => this._setFilter(key)}>
        ${label}<span class="filter-count">(${count})</span>
      </button>
    `;
  }

  private _filterLabel(): string {
    switch (this._filter) {
      case 'review': return 'In Review';
      case 'decision': return 'Needs Decision';
      case 'queued': return 'Queued PRs';
      default: return 'All PR Cases';
    }
  }
}
```

- [ ] **Step 2: Run typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: FAIL — `devtown-pr-detail` does not yet exist (it's `devtown-review-detail` currently). That's expected — Task 3 creates it.

- [ ] **Step 3: Commit**

```bash
git add app/src/main/webui/src/components/pr-cases-workbench.ts
git commit -m "feat(#233): create pr-cases-workbench with filter chips, data merge, adaptive vitals

Three-endpoint merge with deduplication by caseId. Filter chips:
All, In Review, Needs Decision, Queued. Adaptive vitals per filter.
Batch context panel under Queued filter.

Refs #233"
```

---

## Batch 3: Unified detail panel — rename review-detail to pr-detail, add lifecycle-adaptive actions and work-item alert

After this batch: pr-detail.ts handles all lifecycle stages with appropriate actions and work-item alert banners. Typecheck passes. The old merge-queue-detail is no longer needed.

### Task 3: Rename review-detail to pr-detail and add lifecycle-adaptive actions + work-item alert

**Files:**
- Modify: `app/src/main/webui/src/components/review-detail.ts` — rename to `pr-detail.ts`, add stage-aware actions, work-item alert, queue context, batch rendering
- Create: `app/src/main/webui/src/components/pr-detail.ts` (the renamed file — create new, then delete old in Task 5)

**Interfaces:**
- Consumes: `renderWorkItemAlert(item: WorkItemData): TemplateResult | typeof nothing` from `shared/work-item-alert.ts`
- Consumes: `WorkItemData` type from `shared/work-item-alert.ts`
- Consumes: All existing shared composables (trust-bar, dimension-grid, review-card, badge, routing-summary, findings-panel, timeline)
- Produces: `<devtown-pr-detail case-id="..." endpoint="..." .stage .queueContext .triageContext .batch mode>` web component

- [ ] **Step 1: Create pr-detail.ts**

This is the unified detail panel. It incorporates everything from both `review-detail.ts` and `merge-queue-detail.ts`: trust narrative, routing summary, capability checks, completed reviews, findings panel, event timeline, lifecycle-adaptive actions, work-item alert, queue context (dependencies, wait time), and batch rendering.

```typescript
// app/src/main/webui/src/components/pr-detail.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { reviewTimelineStrategy } from './review-timeline-strategy.js';
import { renderTrustBar } from './shared/trust-bar.js';
import { renderDimensionGrid } from './shared/dimension-grid.js';
import { renderReviewCard } from './shared/review-card.js';
import { renderBadge } from './shared/badge.js';
import { renderWorkItemAlert } from './shared/work-item-alert.js';
import type { WorkItemData } from './shared/work-item-alert.js';
import type { DimensionEntry } from './shared/types.js';
import type { ReviewOutcome } from './shared/review-card.js';
import { hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles, trustBarStyles, dimensionGridStyles, reviewCardStyles, badgeStyles, workItemAlertStyles } from './shared/shared-styles.js';
import './routing-summary.js';
import './findings-panel.js';
import '@casehubio/blocks-ui-blocks-timeline';

interface ContributorProfile {
  actorId: string;
  globalScore: number | null;
  intakeClassification: { lane: string; trustScore: number; observationCount: number; classificationReason: string; fastTrackThreshold: number; standardThreshold: number };
  dimensionScores: Record<string, number>;
  recentOutcomes: Array<{ outcome: string; timestamp: string }>;
  githubIntelligence: { mergedCount: number; closedCount: number; mergeRatio: number; tier: string; maturity: string; summary: string } | null;
}

interface ReviewDetailData {
  caseId: string;
  pr: { repo: string; prNumber: number; contributor: string; linesChanged: number; headSha: string };
  timeline: Array<{ timestamp: string; category: string; eventType: string; actor: string; summary: string; metadata: unknown }>;
  capabilities: Array<{ name: string; status: string; outcome: string | null; completedAt: string }>;
  routing: { decisions: Array<{ capability: string; reason: string; confidence: number; bindingName: string }>; featureVector: string | null };
  findings: Record<string, Array<{ severity: string; category: string; filePath: string; message: string; confidence: number; startLine: number | null; endLine: number | null }>>;
}

interface QueueContext {
  priorityLane: string;
  trustScore: number;
  waitMinutes: number;
  headSha: string;
  dependsOn: number[];
}

interface TriageContext {
  caseId: string;
  decisionType: string;
  candidateGroup: string;
  expiresAt: string;
  escalationStage: string;
}

interface BatchData {
  batchId: string;
  caseId: string;
  prCount: number;
  riskLevel: string;
  ciStatus?: string;
  prNumbers?: number[];
  startedAt?: string;
  suspectedPr?: number;
}

@customElement('devtown-pr-detail')
export class PrDetail extends LitElement {
  @property({ type: String, attribute: 'case-id' }) caseId = '';
  @property({ type: String }) endpoint = '';
  @property({ type: String }) stage = '';
  @property({ type: String }) mode = ''; // 'batch' for batch detail
  @property({ type: Object }) queueContext: QueueContext = { priorityLane: '', trustScore: 0, waitMinutes: 0, headSha: '', dependsOn: [] };
  @property({ type: Object }) triageContext: TriageContext = { caseId: '', decisionType: '', candidateGroup: '', expiresAt: '', escalationStage: '' };
  @property({ type: Object }) batch: BatchData | null = null;

  @state() private _data: ReviewDetailData | null = null;
  @state() private _loading = false;
  @state() private _error = '';
  @state() private _actionResult = '';
  @state() private _contributor: ContributorProfile | null = null;
  @state() private _prevContributor = '';

  static override styles = [
    hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles,
    trustBarStyles, dimensionGridStyles, reviewCardStyles, badgeStyles,
    workItemAlertStyles,
    css`
      .meta { display: grid; grid-template-columns: auto 1fr; gap: 4px 12px; margin-bottom: 16px; font-size: 13px; }
      .meta dt { font-weight: 600; color: var(--pages-neutral-8, #404040); }
      .meta dd { margin: 0; }
      .sub-header { font-size: 12px; color: var(--pages-neutral-7, #525252); margin-bottom: 16px; }
      .actions { display: flex; gap: 8px; margin: 12px 0; }
      .actions button {
        padding: 6px 14px; border-radius: 4px; font-size: 13px; font-weight: 500;
        cursor: pointer; border: 1px solid var(--pages-neutral-5, #a3a3a3);
        background: white; color: var(--pages-neutral-9, #171717);
      }
      .actions button:hover { background: var(--pages-neutral-2, #f5f5f5); }
      .actions button.primary {
        background: var(--pages-primary-9, #1d4ed8); color: white;
        border-color: var(--pages-primary-9, #1d4ed8);
      }
      .actions button.primary:hover { background: var(--pages-primary-10, #1e40af); }
      .actions button.danger {
        background: var(--pages-danger-9, #dc2626); color: white;
        border-color: var(--pages-danger-9, #dc2626);
      }
      .actions button.danger:hover { background: var(--pages-danger-10, #b91c1c); }
      .action-result { font-size: 12px; padding: 6px 10px; margin: 4px 0 8px; background: var(--pages-neutral-2, #f5f5f5); border-radius: 3px; }
      .error { color: var(--pages-danger-9, #dc2626); padding: 16px; }
      .check-list { list-style: none; padding: 0; margin: 6px 0 0 0; }
      .check-item { display: flex; align-items: center; gap: 8px; padding: 4px 0; font-size: 12px; }
      .check-icon { width: 16px; text-align: center; font-size: 14px; }
      .check-name { flex: 1; }
      .check-status { font-size: 11px; font-weight: 600; }
      .dep-list { list-style: none; padding: 0; margin: 0; }
      .dep-list li { padding: 3px 0; font-size: 13px; color: var(--pages-neutral-8, #404040); }
      .badge.routine { background: var(--pages-neutral-3, #e5e5e5); color: var(--pages-neutral-9, #171717); }
      .badge.elevated { background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-9, #92400e); }
    `,
  ];

  override willUpdate(changed: Map<PropertyKey, unknown>): void {
    if (changed.has('caseId') && this.caseId) {
      this._fetchDetail();
    }
  }

  private async _fetchDetail(): Promise<void> {
    if (!this.caseId) return;
    this._loading = true;
    this._error = '';
    try {
      const res = await fetch(`${this.endpoint}/${this.caseId}`);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      this._data = await res.json();
      if (this._data && this._data.pr.contributor !== this._prevContributor) {
        this._prevContributor = this._data.pr.contributor;
        this._fetchContributor(this._data.pr.contributor);
      }
    } catch (err) {
      this._error = `Failed to load: ${err}`;
    } finally {
      this._loading = false;
    }
  }

  private async _fetchContributor(actorId: string): Promise<void> {
    this._contributor = null;
    try {
      const res = await fetch(`${this.endpoint}/contributors/${actorId}`);
      if (res.ok) this._contributor = await res.json();
    } catch { /* contributor data optional */ }
  }

  private _checkIcon(status: string): string {
    if (status === 'COMPLETED') return '✅';
    if (status === 'FAILED' || status === 'DECLINED') return '❌';
    if (status === 'SCHEDULED' || status === 'IN_PROGRESS') return '⏳';
    return '⭕';
  }

  private _buildTrustNarrative(contributor: string, c: ContributorProfile): string {
    const ic = c.intakeClassification;
    const gh = c.githubIntelligence;
    const dims = c.dimensionScores;
    const totalPrs = gh ? gh.mergedCount + gh.closedCount : ic.observationCount;
    const parts: string[] = [];

    if (gh && totalPrs > 0) {
      parts.push(`${contributor} has submitted ${totalPrs} PRs — ${gh.mergedCount} merged, ${gh.closedCount} closed (${Math.round(gh.mergeRatio * 100)}% merge rate).`);
    } else {
      parts.push(`${contributor} has ${ic.observationCount} observed PRs.`);
    }

    const lane = ic.lane.toLowerCase();
    const trustPct = Math.round(ic.trustScore * 100);
    const ftPct = Math.round(ic.fastTrackThreshold * 100);
    const stdPct = Math.round(ic.standardThreshold * 100);
    if (lane.includes('fast')) {
      parts.push(`Trust score ${trustPct}% exceeds the fast-track threshold (${ftPct}%), so this PR skips enhanced scrutiny.`);
    } else if (lane.includes('enhanced')) {
      parts.push(`Trust score ${trustPct}% is below the standard threshold (${stdPct}%), so this PR gets additional review scrutiny.`);
    } else {
      parts.push(`Trust score ${trustPct}% is between standard (${stdPct}%) and fast-track (${ftPct}%) thresholds — standard review applies.`);
    }

    const mergeRate = dims['merge-rate'];
    const faq = dims['first-attempt-quality'];
    if (mergeRate != null && faq != null) {
      const mr = Math.round(mergeRate * 100);
      const fa = Math.round(faq * 100);
      if (mr >= 90 && fa >= 80) {
        parts.push(`Strong track record: ${mr}% of PRs merge successfully and ${fa}% pass on first attempt.`);
      } else if (mr >= 70) {
        parts.push(`Merge rate is ${mr}%, but first-attempt quality is ${fa}% — some PRs need revision cycles.`);
      } else {
        parts.push(`Merge rate (${mr}%) and first-attempt quality (${fa}%) indicate this contributor's PRs frequently need rework.`);
      }
    }

    if (ic.observationCount < 10) {
      parts.push(`Only ${ic.observationCount} observations — trust score may shift significantly as more data accumulates.`);
    }

    return parts.join(' ');
  }

  private _buildRecentPattern(outcomes: Array<{ outcome: string }>): string | null {
    if (outcomes.length === 0) return null;
    const recent = outcomes.slice(0, 5);
    const merged = recent.filter(o => o.outcome === 'MERGED').length;
    if (merged === recent.length) return `Last ${recent.length} PRs all merged successfully.`;
    const closed = recent.length - merged;
    return `Last ${recent.length} PRs: ${merged} merged, ${closed} closed/rejected.`;
  }

  private async _doAction(action: string): Promise<void> {
    if (!this._data) return;
    try {
      const res = await fetch(`/api/actions/${action}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          repo: this._data.pr.repo,
          prNumber: this._data.pr.prNumber,
          contributor: this._data.pr.contributor,
          headSha: this._data.pr.headSha,
        }),
      });
      const json = await res.json();
      this._actionResult = `${json.action}: ${json.result}`;
      this._fetchDetail();
    } catch (err) {
      this._actionResult = `Error: ${err}`;
    }
  }

  override render() {
    if (this.mode === 'batch' && this.batch) return this._renderBatch(this.batch);
    if (!this.caseId) return html`<div class="empty">Select a PR case to see details</div>`;
    if (this._loading && !this._data) return html`<div class="empty">Loading...</div>`;
    if (this._error) return html`<div class="error">${this._error}</div>`;
    if (!this._data) return html`<div class="empty">No data</div>`;

    const d = this._data;
    const c = this._contributor;
    const completedCaps = d.capabilities.filter(cap => cap.status === 'COMPLETED');
    const pendingCaps = d.capabilities.filter(cap => cap.status !== 'COMPLETED');
    const isQueued = this.stage === 'QUEUED' || this.stage === 'BATCHED';
    const hasWorkItem = this.triageContext.decisionType !== '';

    const workItem: WorkItemData | null = hasWorkItem ? {
      workItemId: '',
      decisionType: this.triageContext.decisionType,
      candidateGroup: this.triageContext.candidateGroup,
      expiresAt: this.triageContext.expiresAt,
      escalationStage: this.triageContext.escalationStage,
      status: 'PENDING',
    } : null;

    return html`
      <div class="header">PR #${d.pr.prNumber} — ${d.pr.repo}</div>
      ${isQueued ? html`
        <div class="sub-header">by ${d.pr.contributor} · waiting ${this.queueContext.waitMinutes} min · ${(this.queueContext.headSha || d.pr.headSha).substring(0, 7)}</div>
      ` : html`
        <dl class="meta">
          <dt>Author</dt><dd>${d.pr.contributor}</dd>
          <dt>Lines Changed</dt><dd>${d.pr.linesChanged}</dd>
          <dt>Case ID</dt><dd style="font-size:11px">${d.caseId}</dd>
        </dl>
      `}

      ${renderWorkItemAlert(workItem)}

      ${c ? html`
        <div class="section-title">Contributor Trust</div>
        <div style="margin-bottom:6px">
          ${renderBadge(c.intakeClassification.lane, c.intakeClassification.lane === 'FAST_TRACK' ? 'fast-track' : c.intakeClassification.lane === 'ENHANCED_REVIEW' ? 'enhanced' : 'standard')}
        </div>
        <div class="narrative">${this._buildTrustNarrative(d.pr.contributor, c)}</div>
        ${renderTrustBar({
          score: c.intakeClassification.trustScore,
          thresholds: [
            { value: c.intakeClassification.standardThreshold, label: 'standard' },
            { value: c.intakeClassification.fastTrackThreshold, label: 'fast-track' },
          ],
          observationCount: c.intakeClassification.observationCount,
        })}
        ${Object.keys(c.dimensionScores).length > 0 ? renderDimensionGrid(
          Object.entries(c.dimensionScores).map(([key, value]): DimensionEntry => ({ key, value: value as number }))
        ) : nothing}
        ${c.recentOutcomes.length > 0 ? html`
          <div class="section-title">Recent Activity</div>
          <div class="narrative">${this._buildRecentPattern(c.recentOutcomes)}</div>
        ` : nothing}
      ` : nothing}

      <devtown-routing-summary .routing=${d.routing}></devtown-routing-summary>

      ${pendingCaps.length > 0 ? html`
        <div class="section-title">Checks In Progress</div>
        <ul class="check-list">
          ${pendingCaps.map(cap => html`
            <li class="check-item">
              <span class="check-icon">${this._checkIcon(cap.status)}</span>
              <span class="check-name">${cap.name}</span>
              <span class="check-status">${cap.status}</span>
            </li>
          `)}
        </ul>
      ` : nothing}

      ${completedCaps.length > 0 ? html`
        <div class="section-title">Completed Reviews</div>
        ${completedCaps.map(cap => {
          const capFindings = d.findings[cap.name] ?? [];
          return renderReviewCard({
            caseId: d.caseId,
            capability: cap.name,
            outcome: cap.outcome === 'APPROVED' ? 'DONE' : 'DECLINED',
            timestamp: cap.completedAt,
            findingCount: capFindings.length,
            findingSummary: capFindings.map(f => f.message).join('; ') || null,
            feedbackOutcome: null,
          } satisfies ReviewOutcome);
        })}
      ` : nothing}

      <devtown-findings-panel .findings=${d.findings}></devtown-findings-panel>

      ${isQueued && this.queueContext.dependsOn.length > 0 ? html`
        <div class="section-title">Dependencies</div>
        <ul class="dep-list">
          ${this.queueContext.dependsOn.map(d => html`<li>Blocked by PR #${d}</li>`)}
        </ul>
      ` : nothing}

      ${this._renderActions()}
      ${this._actionResult ? html`<div class="action-result">${this._actionResult}</div>` : nothing}

      <div class="section-title">Event Timeline</div>
      <blocks-timeline
        .strategy=${reviewTimelineStrategy}
        .data=${d.timeline}
        layout="vertical"
      ></blocks-timeline>
    `;
  }

  private _renderActions() {
    if (this.stage === 'QUEUED' || this.stage === 'BATCHED') {
      return html`
        <div class="actions">
          <button class="danger" @click=${() => this._doAction('dequeue')}>Dequeue</button>
          <button class="primary" @click=${() => this._doAction('signal-ci-pass')}>Signal CI Pass</button>
        </div>
      `;
    }
    return html`
      <div class="actions">
        <button @click=${() => this._doAction('approve')}>Approve</button>
        <button @click=${() => this._doAction('request-changes')}>Request Changes</button>
        <button class="primary" @click=${() => this._doAction('enqueue')}>Add to Merge Queue</button>
      </div>
    `;
  }

  // --- Batch rendering (from merge-queue-detail) ---

  private _batchElapsed(startedAt?: string): string {
    if (!startedAt) return '—';
    const mins = Math.round((Date.now() - new Date(startedAt).getTime()) / 60000);
    if (mins < 60) return `${mins} min`;
    return `${Math.floor(mins / 60)}h ${mins % 60}m`;
  }

  private _buildBatchNarrative(batch: BatchData): string {
    const prs = batch.prNumbers ?? [];
    const prList = prs.map(n => `#${n}`).join(', ');
    const ci = batch.ciStatus ?? 'UNKNOWN';

    if (ci === 'RUNNING') {
      return `Testing ${prs.length} PRs (${prList}) together against main. If CI passes, all merge as one atomic operation. If it fails, bisection will identify the faulty PR.`;
    }
    if (ci === 'BISECTING') {
      const suspect = batch.suspectedPr ? ` PR #${batch.suspectedPr} is the current suspect.` : '';
      return `CI failed for this batch of ${prs.length} PRs (${prList}). Bisection is running to isolate which PR caused the failure.${suspect} The faulty PR will be ejected and the rest retried.`;
    }
    if (ci === 'PASSED') {
      return `All ${prs.length} PRs (${prList}) passed CI together and are ready to merge.`;
    }
    return `Batch contains ${prs.length} PRs (${prList}). CI status: ${ci}.`;
  }

  private _renderBatch(batch: BatchData) {
    const riskClass = batch.riskLevel.toLowerCase().includes('elevated') ? 'elevated' : 'routine';
    const prs = batch.prNumbers ?? [];
    const ci = batch.ciStatus ?? '—';

    return html`
      <div class="header">Merge Batch — ${prs.length} PRs</div>
      <div class="sub-header">${ci === 'BISECTING' ? 'Bisecting — CI failure detected' : ci === 'RUNNING' ? 'CI running' : ci} · ${this._batchElapsed(batch.startedAt)} elapsed</div>
      <div class="narrative">${this._buildBatchNarrative(batch)}</div>
      <div class="section-title">PRs in Batch</div>
      <ul class="dep-list">
        ${prs.map(n => html`<li style="${batch.suspectedPr === n ? 'color:var(--pages-danger-9,#dc2626);font-weight:600' : ''}">PR #${n}${batch.suspectedPr === n ? ' — suspected cause' : ''}</li>`)}
      </ul>
      <dl class="meta" style="margin-top:12px">
        <dt>Risk</dt><dd><span class="badge ${riskClass}">${batch.riskLevel}</span></dd>
        <dt>CI Status</dt><dd>${ci}</dd>
      </dl>
    `;
  }
}
```

- [ ] **Step 2: Run typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: MAY FAIL if old review-detail.ts has duplicate custom element registration. If so, proceed to Task 4 (wiring) which resolves imports.

- [ ] **Step 3: Commit**

```bash
git add app/src/main/webui/src/components/pr-detail.ts
git commit -m "feat(#233): create pr-detail with lifecycle-adaptive actions and work-item alert

Unified detail panel handles REVIEW, HUMAN_GATE, QUEUED, and BATCHED
stages. Renders work-item alert when a human gate is pending. Includes
batch detail rendering from merge-queue-detail.

Refs #233"
```

---

## Batch 4: Wiring and cleanup — connect unified workbench to index.ts, remove old tabs and components

After this batch: six tabs instead of eight. PR Cases replaces Reviews + Merge Queue + Triage. Old components deleted. Typecheck passes. Mock server serves all data.

### Task 4: Rewire index.ts to use PR Cases tab and clean up old components

**Files:**
- Modify: `app/src/main/webui/src/index.ts` — replace three tabs with one, update imports and panel registration
- Delete: `app/src/main/webui/src/components/review-workbench.ts`
- Delete: `app/src/main/webui/src/components/merge-queue-workbench.ts`
- Delete: `app/src/main/webui/src/components/review-detail.ts`
- Delete: `app/src/main/webui/src/components/merge-queue-detail.ts`
- Delete: `app/src/main/webui/src/views/reviews.ts`
- Delete: `app/src/main/webui/src/views/triage.ts`

**Interfaces:**
- Consumes: `<devtown-pr-cases-workbench>` from `pr-cases-workbench.ts`
- Produces: Updated tab layout: PR Cases, Reviewers, Contributors, Workers, System, Definitions

- [ ] **Step 1: Update index.ts**

Replace the content of `app/src/main/webui/src/index.ts`:

```typescript
import { loadSite, registerPanel } from "@casehubio/pages-runtime";
import { page, tabs, hostPanel } from "@casehubio/pages-ui";
import "@casehubio/blocks-ui-session-workbench";
import "./components/reviewer-workbench";
import "./components/contributor-workbench";
import "./components/pr-cases-workbench";
import { createDatasets } from "./datasets";
import { reviewersView } from "./views/reviewers";
import { systemView } from "./views/system";
import { definitionsView } from "./views/definitions";

const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

registerPanel("session-workbench", "blocks-session-workbench");
registerPanel("reviewer-workbench", "devtown-reviewer-workbench");
registerPanel("contributor-workbench", "devtown-contributor-workbench");
registerPanel("pr-cases-workbench", "devtown-pr-cases-workbench");

async function start() {
  const prefs = await fetch("/api/devtown/governance/preferences")
    .then(r => r.ok ? r.json() : {})
    .then(json => {
      const flat: Record<string, string> = {};
      for (const [section, values] of Object.entries(json)) {
        for (const [key, val] of Object.entries(values as Record<string, string>)) {
          flat[`${section}.${key}`] = val;
        }
      }
      return flat;
    })
    .catch(() => ({}));

  const app = page("DevTown",
    tabs(
      ["PR Cases", hostPanel("pr-cases-workbench", { endpoint: "/api/devtown/governance" })],
      ["Reviewers", reviewersView],
      ["Contributors", hostPanel("contributor-workbench", { endpoint: "/api/devtown/reviews" })],
      ["Workers", hostPanel("session-workbench", { endpoint: "/api/sessions" })],
      ["System", systemView],
      ["Definitions", definitionsView],
    ),
    { settings: { mode: prefersDark ? "dark" : "light" }, datasets: createDatasets(prefs) },
  );

  const container = document.getElementById("app");
  if (container) {
    const site = await loadSite(container, app);
    site.setTheme(prefersDark ? "dark" : "light");

    window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", (e) => {
      site.setTheme(e.matches ? "dark" : "light");
    });
  }
}

start().catch(console.error);
```

- [ ] **Step 2: Delete old components**

Delete these files:
- `app/src/main/webui/src/components/review-workbench.ts`
- `app/src/main/webui/src/components/merge-queue-workbench.ts`
- `app/src/main/webui/src/components/review-detail.ts`
- `app/src/main/webui/src/components/merge-queue-detail.ts`
- `app/src/main/webui/src/views/reviews.ts`
- `app/src/main/webui/src/views/triage.ts`

- [ ] **Step 3: Run typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: PASS — all imports resolved, no references to deleted files.

- [ ] **Step 4: Run mock server and verify visually**

Run: `npm run dev:mock` from `app/src/main/webui/`

Verify in browser at `http://localhost:8280`:
1. Six tabs visible: PR Cases, Reviewers, Contributors, Workers, System, Definitions
2. PR Cases tab opens with filter chips: All (showing combined count), In Review, Needs Decision, Queued
3. Clicking "In Review" shows review vitals (Active, Fleet, Commits, Work Items) and review entries
4. Clicking "Needs Decision" shows triage items with Decision Type, Candidate Group, Expires columns
5. Clicking "Queued" shows queue vitals and queued PRs with Lane/Trust/Wait columns, plus collapsible Active Batches
6. Selecting a PR shows the unified detail with trust narrative, routing, checks, reviews, findings, timeline
7. Selecting a batch shows batch detail with narrative, PR list, risk, CI status
8. Work-item alert banner appears for PRs with pending human gates
9. Actions adapt: Approve/Request Changes/Enqueue for reviews, Dequeue/Signal CI Pass for queued

- [ ] **Step 5: Commit**

```bash
git add app/src/main/webui/src/
git commit -m "feat(#233): wire PR Cases tab, remove Reviews/Merge Queue/Triage tabs

Six tabs: PR Cases, Reviewers, Contributors, Workers, System, Definitions.
Deleted: review-workbench.ts, merge-queue-workbench.ts, review-detail.ts,
merge-queue-detail.ts, views/reviews.ts, views/triage.ts.

Refs #233"
```

---

## Self-Review

**1. Spec coverage:**
- Filter chips (All, In Review, Needs Decision, Queued) — Task 2
- Adaptive vitals per filter — Task 2 `_vitals()` method
- Column switching per filter — Task 2 `FILTER_COLUMNS` config
- Problems banner (system-wide) — Task 2, shown regardless of filter
- Batch context under Queued filter — Task 2, collapsible section
- Work-item alert in detail — Task 3, `renderWorkItemAlert()`
- Lifecycle-adaptive actions — Task 3, `_renderActions()`
- Batch detail rendering — Task 3, `_renderBatch()`
- Three-endpoint data merge with dedup — Task 2, `_mergeData()`
- Tab consolidation (8 → 6) — Task 4
- URL/deep-linking (filter as query param) — **NOT IMPLEMENTED** — deferred; the spec notes this as a resolution but it requires router integration not present in the current app
- Performance (fetch only active filter's data) — **NOT IMPLEMENTED** — the spec proposes lazy fetching but the current codebase fetches all data eagerly; switching to lazy fetch mid-stream adds complexity without the backend `/governance/cases` endpoint. All three fetches are lightweight JSON.

**2. Placeholder scan:** No TBD, TODO, "similar to", or placeholder steps found.

**3. Type consistency:** `UnifiedPrRow`, `FilterKey`, `WorkItemData`, `VitalEntry`, `QueueContext`, `TriageContext`, `BatchData` used consistently across tasks. `renderWorkItemAlert` signature matches import in pr-detail.ts. `renderVitals` signature matches import in pr-cases-workbench.ts.

**4. Tooling safety scan:** No bash cp/rm/mv on source files. File deletions in Task 4 are listed as "Delete" operations — executor should use `rm` (acceptable for deletion) or `ide_refactor_safe_delete` if IntelliJ is available.

## References

- [GitHub #233] — design spec (issue body)
- [GitHub #231] — composable unification (prerequisite, completed)
- `app/src/main/webui/src/components/review-workbench.ts` — existing Reviews tab implementation
- `app/src/main/webui/src/components/merge-queue-workbench.ts` — existing Merge Queue tab implementation
- `app/src/main/webui/src/components/review-detail.ts` — existing review detail panel
- `app/src/main/webui/src/components/merge-queue-detail.ts` — existing merge queue detail panel
- `app/src/main/webui/src/components/shared/` — shared composables from #231
- `app/src/main/webui/src/views/triage.ts` — existing triage view
- `app/src/main/webui/src/index.ts` — tab registration
- `app/src/main/webui/src/datasets.ts` — dataset bindings
- `app/src/main/webui/mock-server.mjs` — mock development server
- `app/src/main/webui/mock-fixtures.json` — mock API fixtures
