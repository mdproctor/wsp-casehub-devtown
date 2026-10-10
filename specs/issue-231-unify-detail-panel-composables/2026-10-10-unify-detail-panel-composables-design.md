# Design: Unify Detail Panel Composables (#231)

## Summary

Extract duplicated rendering logic from `merge-queue-detail.ts`, `contributor-detail.ts`, and `reviewer-detail.ts` into shared render functions and a shared CSS module. Add review history cards to the merge-queue PR detail via cross-reference fetch to `reviewDetail(caseId)`.

## Decisions

- **D1:** Composables stay devtown-local in `src/components/shared/`
- **D2:** Render functions returning `TemplateResult` + shared CSS module (not custom elements)
- **D3:** Cross-reference fetch (`reviewDetail(caseId)`) is in scope

## File structure

```
src/components/shared/
  trust-bar.ts          — renderTrustBar()
  dimension-grid.ts     — renderDimensionGrid()
  status-card.ts        — renderStatusCard()
  review-card.ts        — renderReviewCard()
  badge.ts              — renderBadge()
  shared-styles.ts      — CSS fragments (sectionTitle, narrative, empty, etc.)
  helpers.ts            — trustFillClass(), formatDimension()
  types.ts              — shared interfaces
```

## Shared render functions

### renderTrustBar

**Extracted from:** merge-queue-detail, contributor-detail (near-identical trust bar).

```typescript
interface TrustBarData {
  score: number;
  thresholds?: Array<{ value: number; label: string }>;
  observationCount?: number;
}

function renderTrustBar(data: TrustBarData): TemplateResult
```

Renders: track with colored fill, optional threshold markers, optional legend. Uses `trustFillClass(score)` for coloring.

### renderDimensionGrid

**Extracted from:** merge-queue-detail and contributor-detail (identical 2-col grid), reviewer-detail (3-col variant).

```typescript
interface DimensionEntry {
  key: string;
  value: number;
}

function renderDimensionGrid(
  dimensions: DimensionEntry[],
  options?: { columns?: 2 | 3 }
): TemplateResult
```

Renders: CSS grid of dimension name/value pairs. `formatDimension(key)` converts kebab-case to title case. Value rendered as percentage.

- **2-column (default):** side-by-side label/value in each cell (merge-queue, contributor)
- **3-column:** center-aligned with label above value, uppercase 10px label, 16px value (reviewer)

### renderStatusCard

**Extracted from:** contributor-detail ("proximity-card") and reviewer-detail ("status-card") — same structure, different class names.

```typescript
type StatusTier = 'good' | 'watch' | 'concern';

interface StatusCardData {
  tier: StatusTier;
  text: string;
}

function renderStatusCard(data: StatusCardData): TemplateResult
```

Renders: colored border-left card with tier-driven color (green/yellow/red) and text content. The tier enum unifies contributor's `stable/watch/risk` and reviewer's `healthy/busy/concern` — callers map their domain tiers to `good/watch/concern` before calling.

### renderReviewCard

**Extracted from:** reviewer-detail (review history cards). Will be added to merge-queue-detail.

```typescript
interface ReviewOutcome {
  caseId: string;
  capability: string;
  outcome: 'DONE' | 'DECLINED';
  timestamp: string;
  pr: { repo: string; prNumber: number; contributor: string };
  findingCount: number;
  findingSummary: string | null;
  feedbackOutcome?: 'ACCEPTED' | 'PARTIALLY_ACCEPTED' | 'REJECTED' | null;
  declineReason?: string;
}

function renderReviewCard(outcome: ReviewOutcome): TemplateResult
```

Renders: card with PR reference, capability tag, finding summary, and feedback badge. DECLINED outcomes show decline reason instead of findings.

### renderBadge

**Extracted from:** merge-queue-detail and contributor-detail (lane badges), reviewer-detail (phase badges).

```typescript
type BadgeVariant = 'fast-track' | 'standard' | 'enhanced' | 'active' | 'emerging' | 'bootstrap';

function renderBadge(label: string, variant: BadgeVariant): TemplateResult
```

Renders: colored pill with variant-driven background/text colors.

## Shared CSS module

`shared-styles.ts` exports CSS tagged templates that detail panels import into their `static styles` array:

```typescript
export const sectionTitleStyles = css`...`;   // .section-title
export const narrativeStyles = css`...`;      // .narrative
export const emptyStyles = css`...`;          // .empty
export const trustBarStyles = css`...`;       // .trust-bar-track, .trust-bar-fill, .threshold-marker, .trust-legend
export const dimensionGridStyles = css`...`;  // .dim-grid, .dim-item, .dim-label, .dim-value
export const statusCardStyles = css`...`;     // .status-card
export const reviewCardStyles = css`...`;     // .review-card, .feedback-badge
export const badgeStyles = css`...`;          // .badge variants
```

Each detail panel imports only what it uses:

```typescript
import { sectionTitleStyles, narrativeStyles, trustBarStyles, dimensionGridStyles } from './shared/shared-styles.js';

static override styles = [sectionTitleStyles, narrativeStyles, trustBarStyles, dimensionGridStyles, css`/* panel-specific */`];
```

## Shared helpers

```typescript
// helpers.ts
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

## Shared types

```typescript
// types.ts — re-exports interfaces used by render functions
export type { TrustBarData } from './trust-bar.js';
export type { DimensionEntry } from './dimension-grid.js';
export type { StatusCardData, StatusTier } from './status-card.js';
export type { ReviewOutcome } from './review-card.js';
export type { BadgeVariant } from './badge.js';
```

## Detail panel changes

### merge-queue-detail.ts

1. Replace inline trust bar with `renderTrustBar()`
2. Replace inline dimension grid with `renderDimensionGrid()`
3. Replace inline badge rendering with `renderBadge()`
4. **Add review history section:** Fetch `reviewDetail(caseId)` when a PR is selected, extract `recentOutcomes` from the matched reviewer endpoints, render each with `renderReviewCard()`
5. Replace duplicated CSS with shared style imports

The cross-reference flow:
- When a queued PR is selected, the detail already fetches contributor profile data
- Additionally fetch the review detail for the PR's `caseId` (from queue-status data)
- Extract which reviewers were assigned and their outcomes
- Render review cards showing what each agent found on THIS PR

### contributor-detail.ts

1. Replace inline trust bar with `renderTrustBar()`
2. Replace inline dimension grid with `renderDimensionGrid()`
3. Replace inline proximity card with `renderStatusCard()`
4. Replace inline badge rendering with `renderBadge()`
5. Replace duplicated CSS with shared style imports
6. Narrative generation stays panel-specific (contributor-centric language)

### reviewer-detail.ts

1. Replace inline dimension grid with `renderDimensionGrid({ columns: 3 })`
2. Replace inline status card with `renderStatusCard()`
3. Replace inline review cards with `renderReviewCard()` loop
4. Replace inline badge rendering with `renderBadge()`
5. Replace duplicated CSS with shared style imports
6. Narrative generation stays panel-specific (agent-centric language)

## Mock server changes

The mock server already serves review detail by caseId (mock-server.mjs lines 38-44). No changes needed — the merge-queue detail component can fetch `/api/devtown/reviews/<caseId>` and get the `_review_detail_template` data including capabilities, routing, and findings.

For the reviewer outcomes cross-reference (which agents reviewed a specific PR), the existing per-reviewer endpoints (`/api/devtown/reviews/reviewers/<actorId>`) include `recentOutcomes` with PR references. The merge-queue detail can match outcomes by `caseId`.

## CSS normalization

Minor differences between the three detail panels will be normalized to consistent values:

- `.header` margin-bottom: 4px → 8px (split the difference)
- `.section-title` margin: normalize to `16px 0 6px`
- `.narrative` line-height: normalize to `1.5`
- `.trust-bar` margin: normalize to `8px 0 4px`
- `.threshold-legend` margin: normalize to `4px 0 8px`

## Testing

- `npm run typecheck` verifies all imports and types
- Visual verification via `npm run dev:mock` — each detail panel should render identically before and after extraction
- Verify merge-queue PR detail shows review history cards when a PR is selected
- Each shared render function is a pure function of its data — unit-testable but not gated on this issue

## Out of scope

- Moving composables to blocks-ui (D1)
- Narrative generation unification (domain-specific per panel)
- review-detail.ts changes (structurally different — delegates to child components)

## References

- `src/components/merge-queue-detail.ts` — trust bar, dimension grid, badge duplication source
- `src/components/contributor-detail.ts` — trust bar, dimension grid, status card, badge duplication source
- `src/components/reviewer-detail.ts` — status card, review card, dimension grid, badge duplication source
- `src/components/review-detail.ts` — excluded, structurally different
- `blocks-ui-core/renderSparkline` — prior art for render function pattern
- `blocks-trust-score-panel` — existing trust component in blocks-ui (not replaced, different scope)
- `mock-server.mjs` lines 11-19, 38-44 — review detail mock endpoint
- Issue #231 body — component extraction table and page compositions
