# Unify Detail Panel Composables Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #231 — Unify Merge Queue and Reviewers detail panels with shared composable components
**Issue group:** #231

**Goal:** Extract duplicated rendering logic from three detail panels into shared render functions, unify CSS, and add review history cards to the merge-queue PR detail.

**Architecture:** Shared render functions (returning Lit `TemplateResult`) in `src/components/shared/`, consumed by three existing detail panel components. Each function has a companion CSS export in `shared-styles.ts`. The cross-reference fetch for merge-queue review history uses the existing `/api/devtown/reviews/<caseId>` endpoint.

**Tech Stack:** TypeScript, Lit 3, esbuild bundler

## Global Constraints

- TypeScript strict mode (`tsconfig.json` strict: true)
- ES2020 target, ESNext modules, bundler module resolution
- Import paths use `.js` extension (esbuild bundler convention)
- No test framework in webui — verification via `npm run typecheck` and `npm run dev:mock`
- CSS uses `--pages-*` custom properties from the design system
- Render functions are pure (data in → TemplateResult out), no side effects

---

## Batch 1: Shared composable foundation

### Task 1: Create shared helpers, styles, and render functions

**Files:**
- Create: `app/src/main/webui/src/components/shared/helpers.ts`
- Create: `app/src/main/webui/src/components/shared/shared-styles.ts`
- Create: `app/src/main/webui/src/components/shared/trust-bar.ts`
- Create: `app/src/main/webui/src/components/shared/dimension-grid.ts`
- Create: `app/src/main/webui/src/components/shared/status-card.ts`
- Create: `app/src/main/webui/src/components/shared/review-card.ts`
- Create: `app/src/main/webui/src/components/shared/badge.ts`
- Create: `app/src/main/webui/src/components/shared/types.ts`

**Interfaces:**
- Produces: `trustFillClass(score)`, `formatDimension(key)`, `feedbackClass(fb)`, `feedbackLabel(fb)` from helpers.ts
- Produces: `renderTrustBar(data)`, `renderDimensionGrid(dimensions, options?)`, `renderStatusCard(data)`, `renderReviewCard(outcome)`, `renderBadge(label, variant)` from render function files
- Produces: CSS exports `sectionTitleStyles`, `narrativeStyles`, `emptyStyles`, `trustBarStyles`, `dimensionGridStyles`, `statusCardStyles`, `reviewCardStyles`, `badgeStyles`, `hostStyles` from shared-styles.ts
- Produces: Re-exported types `TrustBarData`, `DimensionEntry`, `StatusCardData`, `StatusTier`, `ReviewOutcome`, `BadgeVariant` from types.ts

- [ ] **Step 1: Create helpers.ts**

```typescript
// app/src/main/webui/src/components/shared/helpers.ts

export function trustFillClass(score: number): 'high' | 'mid' | 'low' | 'very-low' {
  if (score >= 0.80) return 'high';
  if (score >= 0.60) return 'mid';
  if (score >= 0.40) return 'low';
  return 'very-low';
}

export function formatDimension(key: string): string {
  return key.replace(/-/g, ' ').replace(/\b\w/g, c => c.toUpperCase());
}

export function feedbackClass(fb: string): 'accepted' | 'rejected' | 'partial' {
  const lower = fb.toLowerCase();
  if (lower.includes('accept')) return 'accepted';
  if (lower.includes('reject')) return 'rejected';
  return 'partial';
}

export function feedbackLabel(fb: string): string {
  if (fb === 'ACCEPTED') return 'Findings accepted';
  if (fb === 'REJECTED') return 'Findings rejected (false positives)';
  if (fb === 'PARTIALLY_ACCEPTED') return 'Some findings accepted';
  return fb;
}
```

- [ ] **Step 2: Create shared-styles.ts**

Extract CSS from the three detail panels. Source line references:
- merge-queue-detail.ts: section-title (67), trust-bar (68-77), dim-grid (79-82), badge (59-66), empty (108), narrative (109), threshold-legend (110)
- contributor-detail.ts: section-title (34), narrative (35), badge (37-40), trust-bar (42-51), proximity-card (53-59), dim-grid (61-64)
- reviewer-detail.ts: section-title (41), narrative (42), badge (44-47), dim-grid (60-63), status-card (65-71), review-card (73-88)

```typescript
// app/src/main/webui/src/components/shared/shared-styles.ts
import { css } from 'lit';

export const hostStyles = css`
  :host { display: block; height: 100%; overflow-y: auto; padding: 16px;
    font-family: var(--pages-font-family, system-ui); }
  .header { font-size: 18px; font-weight: 600; margin-bottom: 8px; }
`;

export const sectionTitleStyles = css`
  .section-title {
    font-size: 13px; font-weight: 600; margin: 16px 0 6px;
    color: var(--pages-neutral-9, #404040);
    text-transform: uppercase; letter-spacing: 0.5px;
  }
`;

export const narrativeStyles = css`
  .narrative {
    font-size: 13px; line-height: 1.5;
    color: var(--pages-neutral-8, #404040); margin: 6px 0 12px;
  }
`;

export const emptyStyles = css`
  .empty {
    display: flex; align-items: center; justify-content: center;
    height: 100%; color: var(--pages-neutral-7, #525252); font-size: 13px;
  }
`;

export const trustBarStyles = css`
  .trust-bar { display: flex; align-items: center; gap: 8px; margin: 8px 0 4px; }
  .trust-track {
    flex: 1; height: 8px;
    background: var(--pages-neutral-3, #e5e5e5);
    border-radius: 4px; overflow: hidden; position: relative;
  }
  .trust-fill { height: 100%; border-radius: 4px; transition: width 0.3s; }
  .trust-fill.high { background: var(--pages-success-9, #16a34a); }
  .trust-fill.mid { background: var(--pages-primary-9, #1d4ed8); }
  .trust-fill.low { background: var(--pages-warning-9, #d97706); }
  .trust-fill.very-low { background: var(--pages-danger-9, #dc2626); }
  .trust-threshold {
    position: absolute; top: -2px; bottom: -2px; width: 2px;
    background: var(--pages-neutral-7, #525252);
  }
  .trust-value { font-size: 13px; font-weight: 600; min-width: 40px; }
  .threshold-legend {
    display: flex; gap: 12px; font-size: 11px;
    color: var(--pages-neutral-6, #737373); margin: 4px 0 8px;
  }
`;

export const dimensionGridStyles = css`
  .dim-grid { display: grid; gap: 6px; margin-top: 6px; }
  .dim-grid.cols-2 { grid-template-columns: 1fr 1fr; }
  .dim-grid.cols-3 { grid-template-columns: 1fr 1fr 1fr; }
  .dim-item {
    font-size: 12px; padding: 4px 8px;
    background: var(--pages-neutral-2, #f5f5f5); border-radius: 3px;
  }
  .dim-grid.cols-2 .dim-item { display: flex; justify-content: space-between; }
  .dim-grid.cols-3 .dim-item { text-align: center; padding: 6px 8px; }
  .dim-label { color: var(--pages-neutral-8, #404040); }
  .dim-grid.cols-3 .dim-label {
    display: block; font-size: 10px; text-transform: uppercase;
    margin-bottom: 2px; color: var(--pages-neutral-7, #525252);
  }
  .dim-val { font-weight: 600; }
  .dim-grid.cols-3 .dim-val { font-size: 16px; }
`;

export const statusCardStyles = css`
  .status-card {
    margin: 10px 0; padding: 10px 14px; border-radius: 6px;
    font-size: 13px; line-height: 1.5; border-left: 3px solid;
  }
  .status-card.good {
    background: var(--pages-success-2, #f0fdf4);
    border-color: var(--pages-success-9, #16a34a);
  }
  .status-card.watch {
    background: var(--pages-warning-2, #fffbeb);
    border-color: var(--pages-warning-9, #d97706);
  }
  .status-card.concern {
    background: var(--pages-danger-2, #fef2f2);
    border-color: var(--pages-danger-9, #dc2626);
  }
`;

export const reviewCardStyles = css`
  .review-card {
    margin: 8px 0; padding: 10px 14px; border-radius: 6px;
    border: 1px solid var(--pages-neutral-3, #e5e5e5);
    font-size: 12px; line-height: 1.5;
  }
  .review-card-header {
    display: flex; justify-content: space-between;
    align-items: center; margin-bottom: 4px;
  }
  .review-pr { font-weight: 600; font-size: 13px; }
  .review-meta { color: var(--pages-neutral-6, #737373); font-size: 11px; }
  .review-capability { color: var(--pages-neutral-7, #525252); }
  .review-findings { margin-top: 6px; color: var(--pages-neutral-8, #404040); }
  .review-feedback { margin-top: 4px; font-size: 11px; }
  .feedback-badge {
    display: inline-block; padding: 1px 6px; border-radius: 3px;
    font-size: 10px; font-weight: 600;
  }
  .feedback-badge.accepted {
    background: var(--pages-success-3, #dcfce7);
    color: var(--pages-success-9, #166534);
  }
  .feedback-badge.rejected {
    background: var(--pages-danger-3, #fee2e2);
    color: var(--pages-danger-9, #dc2626);
  }
  .feedback-badge.partial {
    background: var(--pages-warning-3, #fef3c7);
    color: var(--pages-warning-9, #92400e);
  }
  .decline-reason {
    margin-top: 4px; font-style: italic;
    color: var(--pages-neutral-7, #525252);
  }
`;

export const badgeStyles = css`
  .badge {
    display: inline-block; padding: 2px 8px; border-radius: 3px;
    font-size: 11px; font-weight: 600;
  }
  .badge.fast-track { background: var(--pages-success-3, #dcfce7); color: var(--pages-success-9, #166534); }
  .badge.standard { background: var(--pages-primary-3, #dbeafe); color: var(--pages-primary-9, #1d4ed8); }
  .badge.enhanced { background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-9, #92400e); }
  .badge.active { background: var(--pages-success-3, #dcfce7); color: var(--pages-success-9, #166534); }
  .badge.emerging { background: var(--pages-primary-3, #dbeafe); color: var(--pages-primary-9, #1d4ed8); }
  .badge.bootstrap { background: var(--pages-neutral-3, #e5e5e5); color: var(--pages-neutral-7, #525252); }
`;
```

- [ ] **Step 3: Create trust-bar.ts**

```typescript
// app/src/main/webui/src/components/shared/trust-bar.ts
import { html, nothing, type TemplateResult } from 'lit';
import { trustFillClass } from './helpers.js';

export interface TrustBarData {
  score: number;
  thresholds?: Array<{ value: number; label: string }>;
  observationCount?: number;
}

export function renderTrustBar(data: TrustBarData): TemplateResult {
  const pct = Math.round(data.score * 100);
  return html`
    <div class="trust-bar">
      <div class="trust-track">
        <div class="trust-fill ${trustFillClass(data.score)}" style="width:${pct}%"></div>
        ${(data.thresholds ?? []).map(t => html`
          <div class="trust-threshold" style="left:${Math.round(t.value * 100)}%"
               title="${t.label} (${Math.round(t.value * 100)}%)"></div>
        `)}
      </div>
      <div class="trust-value">${pct}%</div>
    </div>
    ${data.thresholds || data.observationCount != null ? html`
      <div class="threshold-legend">
        ${(data.thresholds ?? []).map(t => html`
          <span>▮ ${Math.round(t.value * 100)}% ${t.label}</span>
        `)}
        ${data.observationCount != null ? html`
          <span>${data.observationCount} observations</span>
        ` : nothing}
      </div>
    ` : nothing}
  `;
}
```

- [ ] **Step 4: Create dimension-grid.ts**

```typescript
// app/src/main/webui/src/components/shared/dimension-grid.ts
import { html, type TemplateResult } from 'lit';
import { formatDimension } from './helpers.js';

export interface DimensionEntry {
  key: string;
  value: number;
}

export function renderDimensionGrid(
  dimensions: DimensionEntry[],
  options?: { columns?: 2 | 3 },
): TemplateResult {
  const cols = options?.columns ?? 2;
  return html`
    <div class="dim-grid cols-${cols}">
      ${dimensions.map(d => html`
        <div class="dim-item">
          <span class="dim-label">${formatDimension(d.key)}</span>
          <span class="dim-val">${typeof d.value === 'number' ? Math.round(d.value * 100) + '%' : '—'}</span>
        </div>
      `)}
    </div>
  `;
}
```

- [ ] **Step 5: Create status-card.ts**

```typescript
// app/src/main/webui/src/components/shared/status-card.ts
import { html, type TemplateResult } from 'lit';

export type StatusTier = 'good' | 'watch' | 'concern';

export interface StatusCardData {
  tier: StatusTier;
  text: string;
}

export function renderStatusCard(data: StatusCardData): TemplateResult {
  return html`<div class="status-card ${data.tier}">${data.text}</div>`;
}
```

- [ ] **Step 6: Create review-card.ts**

```typescript
// app/src/main/webui/src/components/shared/review-card.ts
import { html, nothing, type TemplateResult } from 'lit';
import { feedbackClass, feedbackLabel } from './helpers.js';

export interface ReviewOutcome {
  caseId: string;
  capability: string;
  outcome: string;
  timestamp: string;
  pr?: { repo: string; prNumber: number; contributor: string };
  findingCount?: number;
  findingSummary?: string | null;
  feedbackOutcome?: string | null;
  declineReason?: string | null;
}

export function renderReviewCard(o: ReviewOutcome): TemplateResult {
  const pr = o.pr;
  return html`
    <div class="review-card">
      <div class="review-card-header">
        <span class="review-pr">${pr ? `PR #${pr.prNumber} — ${pr.repo}` : o.caseId.substring(0, 8)}</span>
        <span class="review-meta">${new Date(o.timestamp).toLocaleDateString()}</span>
      </div>
      ${pr ? html`<div class="review-capability">${o.capability.replace(/-/g, ' ')} · by ${pr.contributor}</div>` : nothing}
      ${o.outcome === 'DECLINED' && o.declineReason ? html`
        <div class="decline-reason">Declined: ${o.declineReason}</div>
      ` : nothing}
      ${o.findingCount != null && o.findingCount > 0 ? html`
        <div class="review-findings">${o.findingCount} finding${o.findingCount > 1 ? 's' : ''}: ${o.findingSummary}</div>
      ` : o.outcome === 'DONE' ? html`
        <div class="review-findings" style="color:var(--pages-neutral-6,#737373)">No findings — clean review</div>
      ` : nothing}
      ${o.feedbackOutcome ? html`
        <div class="review-feedback">
          <span class="feedback-badge ${feedbackClass(o.feedbackOutcome)}">${feedbackLabel(o.feedbackOutcome)}</span>
        </div>
      ` : nothing}
    </div>
  `;
}
```

- [ ] **Step 7: Create badge.ts**

```typescript
// app/src/main/webui/src/components/shared/badge.ts
import { html, type TemplateResult } from 'lit';

export type BadgeVariant = 'fast-track' | 'standard' | 'enhanced' | 'active' | 'emerging' | 'bootstrap';

export function renderBadge(label: string, variant: BadgeVariant): TemplateResult {
  return html`<span class="badge ${variant}">${label}</span>`;
}
```

- [ ] **Step 8: Create types.ts barrel**

```typescript
// app/src/main/webui/src/components/shared/types.ts
export type { TrustBarData } from './trust-bar.js';
export type { DimensionEntry } from './dimension-grid.js';
export type { StatusCardData, StatusTier } from './status-card.js';
export type { ReviewOutcome } from './review-card.js';
export type { BadgeVariant } from './badge.js';
```

- [ ] **Step 9: Typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: PASS (no errors — new files are self-contained, not yet imported)

- [ ] **Step 10: Commit**

```bash
git add app/src/main/webui/src/components/shared/
git commit -m "feat: add shared composable render functions for detail panels

Extract trust-bar, dimension-grid, status-card, review-card, badge
as render functions with shared CSS module and helpers.

Refs #231"
```

---

## Batch 2: Rewire contributor-detail

### Task 2: Replace duplicated code in contributor-detail.ts with shared composables

**Files:**
- Modify: `app/src/main/webui/src/components/contributor-detail.ts` (245 lines)

**Interfaces:**
- Consumes: `renderTrustBar(TrustBarData)` from `./shared/trust-bar.js`
- Consumes: `renderDimensionGrid(DimensionEntry[])` from `./shared/dimension-grid.js`
- Consumes: `renderStatusCard(StatusCardData)` from `./shared/status-card.js`
- Consumes: `renderBadge(label, variant)` from `./shared/badge.js`
- Consumes: CSS exports from `./shared/shared-styles.js`

- [ ] **Step 1: Add shared imports**

Add to the top of contributor-detail.ts (after existing lit imports):

```typescript
import { renderTrustBar } from './shared/trust-bar.js';
import { renderDimensionGrid } from './shared/dimension-grid.js';
import { renderStatusCard } from './shared/status-card.js';
import { renderBadge } from './shared/badge.js';
import type { DimensionEntry } from './shared/types.js';
import { hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles, trustBarStyles, dimensionGridStyles, statusCardStyles, badgeStyles } from './shared/shared-styles.js';
```

- [ ] **Step 2: Replace static styles**

Replace the `static override styles = css\`...\`` block. Remove all CSS that moved to shared-styles (section-title line 34, narrative 35, badge 37-40, trust-bar 42-51, proximity-card 53-59, dim-grid 61-64). Keep panel-specific CSS (host overrides, outcome-list, outcome-item, outcome-result, outcome-date).

```typescript
static override styles = [
  hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles,
  trustBarStyles, dimensionGridStyles, statusCardStyles, badgeStyles,
  css`
    /* panel-specific styles that remain */
    .sub-header { font-size: 13px; color: var(--pages-neutral-7, #525252); margin-bottom: 16px; }
    .outcome-list { list-style: none; padding: 0; margin: 0; }
    .outcome-item { padding: 6px 0; border-bottom: 1px solid var(--pages-neutral-3, #e5e5e5); font-size: 12px; display: flex; justify-content: space-between; }
    .outcome-result { font-weight: 600; }
    .outcome-date { color: var(--pages-neutral-6, #737373); font-size: 11px; }
  `,
];
```

- [ ] **Step 3: Remove helper methods**

Delete these methods from the class (they're now in shared helpers):
- `_trustFillClass` (lines 92-97)
- `_laneBadgeClass` (lines 99-104)

- [ ] **Step 4: Replace trust bar call site**

Replace the inline trust bar HTML (lines 199-211) with:

```typescript
${renderTrustBar({
  score: c.intakeClassification.trustScore,
  thresholds: [
    { value: c.intakeClassification.standardThreshold, label: 'standard' },
    { value: c.intakeClassification.fastTrackThreshold, label: 'fast-track' },
  ],
  observationCount: c.intakeClassification.observationCount,
})}
```

- [ ] **Step 5: Replace proximity card call site**

Replace the inline proximity card (line 213) with:

```typescript
${renderStatusCard({
  tier: proximity.level === 'stable' ? 'good' : proximity.level === 'watch' ? 'watch' : 'concern',
  text: proximity.text,
})}
```

- [ ] **Step 6: Replace dimension grid call site**

Replace the inline dimension grid (lines 215-225) with:

```typescript
${renderDimensionGrid(
  Object.entries(c.dimensionScores).map(([key, value]): DimensionEntry => ({ key, value: value as number }))
)}
```

- [ ] **Step 7: Replace badge call site**

Replace the inline badge HTML (line 194) with:

```typescript
${renderBadge(c.intakeClassification.lane.replace(/_/g, ' ').toLowerCase(),
  c.intakeClassification.lane === 'FAST_TRACK' ? 'fast-track'
  : c.intakeClassification.lane === 'ENHANCED_REVIEW' ? 'enhanced' : 'standard')}
```

- [ ] **Step 8: Typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add app/src/main/webui/src/components/contributor-detail.ts
git commit -m "refactor: rewire contributor-detail to use shared composables

Replace inline trust bar, dimension grid, status card, and badge
with shared render functions and CSS imports.

Refs #231"
```

---

## Batch 3: Rewire reviewer-detail

### Task 3: Replace duplicated code in reviewer-detail.ts with shared composables

**Files:**
- Modify: `app/src/main/webui/src/components/reviewer-detail.ts` (247 lines)

**Interfaces:**
- Consumes: `renderDimensionGrid(DimensionEntry[], { columns: 3 })` from `./shared/dimension-grid.js`
- Consumes: `renderStatusCard(StatusCardData)` from `./shared/status-card.js`
- Consumes: `renderReviewCard(ReviewOutcome)` from `./shared/review-card.js`
- Consumes: `renderBadge(label, variant)` from `./shared/badge.js`
- Consumes: CSS exports from `./shared/shared-styles.js`

- [ ] **Step 1: Add shared imports**

Add to the top of reviewer-detail.ts (after existing lit imports):

```typescript
import { renderDimensionGrid } from './shared/dimension-grid.js';
import { renderStatusCard } from './shared/status-card.js';
import { renderReviewCard } from './shared/review-card.js';
import { renderBadge } from './shared/badge.js';
import type { DimensionEntry } from './shared/types.js';
import { hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles, dimensionGridStyles, statusCardStyles, reviewCardStyles, badgeStyles } from './shared/shared-styles.js';
```

- [ ] **Step 2: Replace static styles**

Remove all CSS that moved to shared-styles (section-title 41, narrative 42, badge 44-47, dim-grid 60-63, status-card 65-71, review-card 73-88). Keep panel-specific CSS (host overrides, sub-header, cap-list/cap-item/cap-bar/cap-fill/cap-score).

```typescript
static override styles = [
  hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles,
  dimensionGridStyles, statusCardStyles, reviewCardStyles, badgeStyles,
  css`
    /* panel-specific: capability bars */
    .sub-header { font-size: 13px; color: var(--pages-neutral-7, #525252); margin-bottom: 8px; }
    .cap-list { display: flex; flex-direction: column; gap: 4px; margin-top: 6px; }
    .cap-item { display: flex; align-items: center; gap: 8px; font-size: 12px; }
    .cap-label { min-width: 120px; color: var(--pages-neutral-8, #404040); }
    .cap-bar { flex: 1; height: 6px; background: var(--pages-neutral-3, #e5e5e5); border-radius: 3px; overflow: hidden; }
    .cap-fill { height: 100%; border-radius: 3px; transition: width 0.3s; }
    .cap-fill.high { background: var(--pages-success-9, #16a34a); }
    .cap-fill.mid { background: var(--pages-primary-9, #1d4ed8); }
    .cap-fill.low { background: var(--pages-warning-9, #d97706); }
    .cap-fill.very-low { background: var(--pages-danger-9, #dc2626); }
    .cap-score { font-weight: 600; min-width: 36px; }
  `,
];
```

- [ ] **Step 3: Remove helper methods**

Delete these methods from the class:
- `_fillClass` (lines 109-113) — replaced by `trustFillClass` from helpers (add `import { trustFillClass } from './shared/helpers.js';` if capability bars still need it)
- `_phaseBadgeClass` (lines 115-120)
- `_feedbackClass` (lines 163-168)
- `_feedbackLabel` (lines 170-175)
- `_renderReviewCard` method (lines 177-199)

For the capability bar rendering that remains panel-specific, import `trustFillClass` from helpers:
```typescript
import { trustFillClass } from './shared/helpers.js';
```

Replace any `this._fillClass(score)` calls in the capability bar section with `trustFillClass(score)`.

- [ ] **Step 4: Replace status card call site**

Replace the inline status card (line 215) with:

```typescript
${renderStatusCard({
  tier: status.level === 'healthy' ? 'good' : status.level === 'busy' ? 'watch' : 'concern',
  text: status.text,
})}
```

- [ ] **Step 5: Replace dimension grid call site**

Replace the inline dimension grid (lines 228-238) with:

```typescript
${renderDimensionGrid(
  Object.entries(h.trustByDimension).map(([key, value]): DimensionEntry => ({ key, value: value as number })),
  { columns: 3 },
)}
```

- [ ] **Step 6: Replace review card loop**

Replace the inline `_renderReviewCard` calls in the template with:

```typescript
${h.recentOutcomes.map(o => renderReviewCard(o))}
```

- [ ] **Step 7: Replace badge call site**

Replace the inline badge HTML (line 210) with:

```typescript
${renderBadge(h.maturityPhase,
  h.maturityPhase.toLowerCase() === 'active' ? 'active'
  : h.maturityPhase.toLowerCase() === 'emerging' ? 'emerging' : 'bootstrap')}
```

- [ ] **Step 8: Typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add app/src/main/webui/src/components/reviewer-detail.ts
git commit -m "refactor: rewire reviewer-detail to use shared composables

Replace inline dimension grid, status card, review cards, and badge
with shared render functions and CSS imports.

Refs #231"
```

---

## Batch 4: Rewire merge-queue-detail + review history cross-reference

### Task 4: Replace duplicated code in merge-queue-detail.ts and add review history section

**Files:**
- Modify: `app/src/main/webui/src/components/merge-queue-detail.ts` (376 lines)

**Interfaces:**
- Consumes: `renderTrustBar(TrustBarData)` from `./shared/trust-bar.js`
- Consumes: `renderDimensionGrid(DimensionEntry[])` from `./shared/dimension-grid.js`
- Consumes: `renderBadge(label, variant)` from `./shared/badge.js`
- Consumes: `renderReviewCard(ReviewOutcome)` from `./shared/review-card.js`
- Consumes: CSS exports from `./shared/shared-styles.js`
- Consumes: `/api/devtown/reviews/<caseId>` endpoint (existing mock, review detail data with capabilities and findings)

- [ ] **Step 1: Add shared imports**

Add to the top of merge-queue-detail.ts (after existing lit imports):

```typescript
import { renderTrustBar } from './shared/trust-bar.js';
import { renderDimensionGrid } from './shared/dimension-grid.js';
import { renderBadge } from './shared/badge.js';
import { renderReviewCard } from './shared/review-card.js';
import type { DimensionEntry } from './shared/types.js';
import type { ReviewOutcome } from './shared/review-card.js';
import { hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles, trustBarStyles, dimensionGridStyles, badgeStyles, reviewCardStyles } from './shared/shared-styles.js';
```

- [ ] **Step 2: Replace static styles**

Remove CSS that moved to shared-styles (badge 59-66, section-title 67, trust-bar 68-77, dim-grid 79-82, empty 108, narrative 109, threshold-legend 110). Keep panel-specific CSS (meta-grid, check-list, outcome-row, dep-list, actions, action-result, outcome-chip).

```typescript
static override styles = [
  hostStyles, sectionTitleStyles, narrativeStyles, emptyStyles,
  trustBarStyles, dimensionGridStyles, badgeStyles, reviewCardStyles,
  css`
    /* panel-specific styles that remain */
    .sub-header { font-size: 13px; color: var(--pages-neutral-7, #525252); }
    .meta-grid { display: grid; grid-template-columns: auto 1fr; gap: 4px 12px; margin: 8px 0 16px; font-size: 12px; }
    .meta-label { color: var(--pages-neutral-6, #737373); }
    .check-list { display: flex; flex-direction: column; gap: 4px; margin-top: 6px; }
    .check-item { display: flex; align-items: center; gap: 8px; font-size: 12px; padding: 4px 8px; background: var(--pages-neutral-2, #f5f5f5); border-radius: 3px; }
    .outcome-row { display: flex; align-items: center; gap: 6px; }
    .outcome-chip { display: inline-block; padding: 1px 6px; border-radius: 3px; font-size: 10px; font-weight: 600; }
    .outcome-chip.approved { background: var(--pages-success-3, #dcfce7); color: var(--pages-success-9, #166534); }
    .outcome-chip.rejected { background: var(--pages-danger-3, #fee2e2); color: var(--pages-danger-9, #dc2626); }
    .outcome-chip.pending { background: var(--pages-warning-3, #fef3c7); color: var(--pages-warning-9, #92400e); }
    .dep-list { font-size: 12px; color: var(--pages-neutral-7, #525252); margin-top: 4px; }
    .actions { display: flex; gap: 8px; margin-top: 16px; }
    .actions button { padding: 6px 14px; border-radius: 4px; font-size: 12px; font-weight: 600; cursor: pointer; border: 1px solid var(--pages-neutral-4, #d4d4d4); background: var(--pages-neutral-1, #fafafa); }
    .actions button.primary { background: var(--pages-primary-9, #1d4ed8); color: white; border-color: var(--pages-primary-9, #1d4ed8); }
    .action-result { margin-top: 8px; padding: 8px 12px; border-radius: 4px; font-size: 12px; }
    .action-result.success { background: var(--pages-success-2, #f0fdf4); color: var(--pages-success-9, #166534); }
    .action-result.error { background: var(--pages-danger-2, #fef2f2); color: var(--pages-danger-9, #dc2626); }
  `,
];
```

- [ ] **Step 3: Remove helper methods**

Delete these methods from the class:
- `_laneBadgeClass` (lines 160-165)
- `_trustFillClass` (lines 167-172)
- `_formatDimension` (lines 181-183)

- [ ] **Step 4: Add review detail state and fetch**

Add a new state property and fetch method to the class:

```typescript
@state() private _reviewOutcomes: ReviewOutcome[] = [];
```

Add a fetch method that retrieves review detail for the current PR's caseId:

```typescript
private async _fetchReviewDetail(caseId: string): Promise<void> {
  try {
    const resp = await fetch(`${this.endpoint}/reviews/${caseId}`);
    if (!resp.ok) return;
    const data = await resp.json();
    const capabilities = data.capabilities ?? [];
    this._reviewOutcomes = capabilities
      .filter((c: any) => c.status === 'COMPLETED')
      .map((c: any) => ({
        caseId,
        capability: c.name,
        outcome: c.outcome === 'APPROVED' ? 'DONE' : 'DECLINED',
        timestamp: c.completedAt ?? new Date().toISOString(),
        findingCount: (data.findings?.[c.name] ?? []).length,
        findingSummary: (data.findings?.[c.name] ?? []).map((f: any) => f.message).join('; ') || null,
        feedbackOutcome: null,
      } satisfies ReviewOutcome));
  } catch {
    this._reviewOutcomes = [];
  }
}
```

Call this from the existing PR selection handler (wherever `_selectedPr` or similar is set), alongside the existing contributor profile fetch:

```typescript
this._fetchReviewDetail(selectedEntry.caseId);
```

- [ ] **Step 5: Replace trust bar call site**

Replace the inline trust bar HTML (lines 265-282) with:

```typescript
${c?.intakeClassification ? renderTrustBar({
  score: c.intakeClassification.trustScore,
  thresholds: [
    { value: c.intakeClassification.standardThreshold, label: 'standard' },
    { value: c.intakeClassification.fastTrackThreshold, label: 'fast-track' },
  ],
  observationCount: c.intakeClassification.observationCount,
}) : nothing}
```

- [ ] **Step 6: Replace dimension grid call site**

Replace the inline dimension grid (lines 284-293) with:

```typescript
${c?.dimensionScores ? renderDimensionGrid(
  Object.entries(c.dimensionScores).map(([key, value]): DimensionEntry => ({ key, value: value as number }))
) : nothing}
```

- [ ] **Step 7: Replace badge call site**

Replace the inline badge HTML (line 258) with:

```typescript
${renderBadge(entry.priorityLane.replace(/_/g, ' ').toLowerCase(),
  entry.priorityLane === 'FAST_TRACK' ? 'fast-track'
  : entry.priorityLane === 'ENHANCED_REVIEW' ? 'enhanced' : 'standard')}
```

- [ ] **Step 8: Add review history section to render**

Add a new section after the existing content (after the dimension grid, before actions):

```typescript
${this._reviewOutcomes.length > 0 ? html`
  <div class="section-title">Review Activity</div>
  ${this._reviewOutcomes.map(o => renderReviewCard(o))}
` : nothing}
```

- [ ] **Step 9: Typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: PASS

- [ ] **Step 10: Visual verification**

Run: `npm run dev:mock` from `app/src/main/webui/`
Open `http://localhost:8280` in browser.

Verify:
1. **Merge Queue tab:** Click a queued PR → detail panel shows trust bar, dimension grid, badge, AND review activity cards
2. **Contributors tab:** Click a contributor → detail panel renders identically to before (trust bar, dimension grid, status card, badge)
3. **Reviewers tab:** Click a reviewer → detail panel renders identically to before (capability bars, dimension grid, status card, review history, badge)
4. No visual regressions in any panel

- [ ] **Step 11: Commit**

```bash
git add app/src/main/webui/src/components/merge-queue-detail.ts
git commit -m "feat: rewire merge-queue-detail to shared composables, add review history

Replace inline trust bar, dimension grid, and badge with shared
render functions. Add review activity section showing which agents
reviewed the PR and what they found.

Refs #231"
```

---

## References

- [2026-10-10-unify-detail-panel-composables-design.md] — design spec this plan implements
- `app/src/main/webui/src/components/merge-queue-detail.ts` — 376 lines, trust bar (265-282), dim-grid (284-293), badge (258), helpers (160-183)
- `app/src/main/webui/src/components/contributor-detail.ts` — 245 lines, trust bar (199-211), proximity (213), dim-grid (215-225), badge (194), helpers (92-104)
- `app/src/main/webui/src/components/reviewer-detail.ts` — 247 lines, status (215), dim-grid (228-238), review cards (177-199), badge (210), helpers (109-175)
- `app/src/main/webui/mock-server.mjs:38-44` — review detail endpoint
- `blocks-ui-core renderSparkline` — prior art for render function pattern
- GitHub #231 — focal issue
