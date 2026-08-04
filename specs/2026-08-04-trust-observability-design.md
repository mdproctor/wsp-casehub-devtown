# Trust Observability — Design Spec

**Epic:** Trust Observability (#98 + #179)
**Date:** 2026-08-04
**Status:** Approved

## Context

Trust scoring infrastructure is mature — TrustGateService, TrustExportService,
TrustCandidateClassifier, IncidentFeedbackResource, and compliance evidence all
exist in the foundation. UI components exist in casehub-packages: trust-workbench,
trust-score-panel, trust-feedback-display, routing-rationale. None are wired into
devtown views. The reviewers view shows a flat table with no trust data.

## Goals

1. Surface per-reviewer trust scores, routing history, and incident feedback in
   the devtown UI by wiring existing components (#98)
2. Investigate and configure score decay for dormant contributors (#179)

## Non-Goals

- New UI components (all exist in casehub-packages)
- Fleet-wide trust comparison charts
- Contributor trust UI (separate from reviewer trust)
- Storing full routing rationale (use what the ledger has)

---

## Issue Structure

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #98 | Trust visibility UI — routing history endpoint + reviewers view expansion | M | Med |
| #179 | Score decay — investigate DecayFunction, configure or file upstream | S | Med |

The routing history endpoint is folded into #98 — it exists solely to serve
the trust-workbench component.

---

## Backend: TrustObservabilityResource

**Package:** `io.casehub.devtown.app.governance`

### Endpoints

| Method | Path | Returns |
|--------|------|---------|
| GET | `/api/governance/trust/{actorId}` | Score proxy — globalScore, capabilityScores, dimensionScores |
| GET | `/api/governance/trust/{actorId}/routing-history` | `List<RoutingDecisionSummary>` |
| GET | `/api/governance/trust/{actorId}/routing-history/{entryId}` | `RoutingDecisionDetail` |

All endpoints use `@RolesAllowed` matching `GovernanceResource` (read-only
governance data — no admin restriction).

The routing history endpoint accepts an optional `?capability=X` query
parameter to filter by capability tag (the trust-workbench sends this when
a capability is selected in the score panel). Pagination: `?limit=N&offset=M`
with a default limit of 50.

### Score proxy

Delegates to `TrustGateService`:
- `currentScore(actorId)` → globalScore (nullable — `OptionalDouble.empty()` → JSON `null`)
- `allCapabilityScores(actorId)` → capabilityScores map
- `allDimensionScores(actorId)` → dimensionScores map

Returns 200 with nullable `globalScore` when the actor exists but has no
global score computed. Returns 404 only when the actor is entirely unknown.

### Routing history

Queries `LedgerEntryRepository` for `WorkerDecisionEntry` records where
`actorId` matches. The query must filter by entry type — `findByActorId`
returns all ledger entry types, not just `WorkerDecisionEntry`.

### DTOs — TypeScript types are authoritative

The Java DTOs must produce JSON matching the TypeScript interfaces in
casehub-packages exactly. The authoritative type definitions are:

- `trust-score-panel/src/types.ts` → `TrustScoreResponse`
- `trust-workbench/src/types.ts` → `RoutingDecisionSummary`, `RoutingDecisionDetail`
- `routing-rationale/src/types.ts` → `RoutingRationaleData`, `CandidateScore`, `RoutingPolicySummary`
- `trust-feedback-display/src/types.ts` → `GateDecision`

Key fields that diverge from naive ledger mapping:

**RoutingDecisionSummary** — the TS interface expects:
`id (string)`, `timestamp (string)`, `capabilityTag`, `selectedWorkerId`,
`finalScore`, `phase` (BOOTSTRAP|QUALIFIED|BORDERLINE|EXCLUDED_PHASE2B|EXCLUDED_PHASE3).
Not: trustScoreAtRouting, thresholdApplied, outcome, caseId. Derive
`selectedWorkerId`, `finalScore`, and `phase` from `WorkerDecisionEntry` fields.

**RoutingPolicySummary** — the TS interface requires 7 fields:
`threshold`, `borderlineMargin`, `blendFactor`, `minimumObservations`,
`qualityFloors`, `cbrWeight`, `bootstrapEscalationRequired`.
Populate from `DevtownTrustRoutingPolicyProvider.forCapability()` at query
time (the policy is deterministic for a given capability tag).

**CandidateScore** — `trustScore` is `number | null` (not primitive double).
`phase` is a string enum, not free-form. `exclusionReason`, `rationale`,
and `additionalScores` are optional. Alternatives array may be empty if the
ledger doesn't store the full candidate set — trust-workbench renders
"selected" without alternatives gracefully.

**GateDecision** — `trustScoreBefore` and `trustScoreAfter` are not stored
on `LedgerAttestation` directly. Compute by querying the actor's trust score
at the attestation timestamp (from materialized score snapshots) or
approximate from the attestation's `dimensionScore` and confidence fields.
If not computable, use the current score for both (no delta shown).

---

## Frontend: Reviewers View Expansion

### Layout change

Replace the flat `dataTable` in `reviewers.ts` with a split-pane layout:

- **Left pane:** Reviewer fleet table (existing columns + new inline trust badge)
  - Add `trustScore` column using `<blocks-trust-score-panel mode="compact">`
  - The `/api/governance/reviewers` response already includes per-reviewer
    trust data (trust-by-capability from `TrustExportService` and
    `TrustGateService`). Pass `score` and `trust-level` attributes directly
    from the reviewer row data — trust-score-panel's compact mode renders
    from pre-fetched attributes without an additional fetch when both
    `score` and `trustLevel` properties are set (see `_hasPreFetchedData()`)
  - `GovernanceQueryService.reviewerFleet()` must include a `globalScore`
    field in its response for this to work. If missing, add it — it's a
    single `TrustGateService.currentScore()` call per reviewer
  - Row selection emits pages event with actorId

- **Right pane:** `<blocks-trust-workbench>` bound to selected actorId
  - `endpoint="/api/governance"` resolves to the new trust endpoints
  - Shows: full trust-score-panel (global + per-capability + trend), routing
    history list, routing rationale detail, feedback entries

### Data flow

1. `reviewers.ts` fetches `/api/governance/reviewers` (existing dataset)
2. Table renders with inline trust badges (pre-fetched data, no extra request)
3. User clicks a row → pages event `reviewer:selected` with actorId
4. Trust-workbench fetches `/api/governance/trust/{actorId}` (full scores)
5. Trust-workbench fetches `/api/governance/trust/{actorId}/routing-history`
6. User clicks a routing decision → workbench fetches detail endpoint

### Endpoint alignment

The trust-workbench component uses `${endpoint}/trust/${actorId}` pattern.
With `endpoint="/api/governance"`, this resolves to:
- Scores: `/api/governance/trust/{actorId}` ✓ (score proxy)
- History: `/api/governance/trust/{actorId}/routing-history` ✓
- Detail: `/api/governance/trust/{actorId}/routing-history/{entryId}` ✓

---

## #179: Score Decay Investigation

### Phase 1: Investigate

1. Search casehub-ledger for `DecayFunction`, `decay`, `time-weight` in
   `TrustScoreCalculator` and `TrustScoreComputer`
2. If found: document behaviour, check if active, determine applicability
   to contributor trust (`PR_CONTRIBUTION` capability)
3. If not found: document the gap

### Phase 2: Act (outcome-dependent)

- **Decay exists and works:** Configure for contributors via preference key
  `casehubio.devtown.contributor-trust.decay-half-life`. Close #179.
- **Decay exists but incomplete:** File ledger enhancement. Close #179 with
  "blocked by ledger#N".
- **No decay mechanism:** File ledger issue for `DecayFunction` SPI. Close
  #179 as deferred. The issue already recommends "evaluate after 6+ months."

---

## Testing

### Integration tests (QuarkusTest)

- Seed `WorkerDecisionEntry` + `LedgerAttestation` records, hit routing
  history endpoints, verify response shape matches trust-workbench types
- Empty history → empty list (not 404)
- Decision with no attestation feedback → detail with empty feedback array
- Score proxy returns consistent data with `TrustGateService` methods
- L1 cache gotcha (GE-20260625-aaf3d4): wrap score reads after
  `requiringNew()` writes

### Frontend verification

- Reviewer row click → trust-workbench renders in detail pane
- Inline trust badges render with correct level colors
- Capability filter in trust-score-panel filters routing history list
- Manual browser test via `quarkus:dev`

### Decay (#179)

- If configured: test that old attestations produce lower effective scores
  than recent attestations of equal quality

---

## Known Risks

- **Routing rationale completeness:** `WorkerDecisionEntry` stores
  `trustScoreAtRouting` and `thresholdApplied` but may not store the full
  candidate set. The alternatives array in `RoutingRationaleData` may be
  empty. Trust-workbench handles this gracefully — it renders "selected"
  without alternatives.

- **Quinoa npm install:** Pre-existing build issue. Frontend changes
  require the build to pass — may need workaround.

- **CaseMemoryObserver binary incompatibility:** Pre-existing SNAPSHOT
  issue. Unrelated to this epic but may affect `mvn install`.
