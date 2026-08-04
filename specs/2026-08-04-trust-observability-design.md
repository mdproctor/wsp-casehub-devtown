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

### Score proxy

Delegates to `TrustGateService`:
- `currentScore(actorId)` → globalScore
- `allCapabilityScores(actorId)` → capabilityScores map
- `allDimensionScores(actorId)` → dimensionScores map

Returns `TrustScoreResponse` matching the TypeScript interface in
`trust-score-panel/src/types.ts`.

### Routing history

Queries `LedgerEntryRepository` for `WorkerDecisionEntry` records where
`actorId` matches.

**RoutingDecisionSummary** (list item):
- `id` — entry UUID
- `capabilityTag` — which capability was routed
- `trustScoreAtRouting` — score snapshot at decision time
- `thresholdApplied` — policy threshold used
- `outcome` — DONE / DECLINED / FAILED
- `timestamp` — when the decision was made
- `caseId` — parent case reference

**RoutingDecisionDetail** (detail pane):
- `rationale` — `RoutingRationaleData` with selected candidate info and policy
  summary. Populated from the `WorkerDecisionEntry` fields. Alternatives array
  may be empty if the ledger doesn't store the full candidate set — the
  trust-workbench renders this gracefully.
- `feedback` — `List<GateDecision>` from `LedgerAttestation` records on this
  entry. Each carries decision, actor, attestation verdict, trustScoreBefore,
  trustScoreAfter, dimension.

### DTOs

Java records in `io.casehub.devtown.app.governance`:
- `TrustScoreResponse(String actorId, Double globalScore, Map<String, Double> capabilityScores, Map<String, Double> dimensionScores)`
- `RoutingDecisionSummary(UUID id, String capabilityTag, double trustScoreAtRouting, double thresholdApplied, String outcome, Instant timestamp, UUID caseId)`
- `RoutingDecisionDetail(RoutingRationaleData rationale, List<GateDecision> feedback)`
- `RoutingRationaleData(String capabilityTag, String strategyId, CandidateScore selected, List<CandidateScore> alternatives, RoutingPolicySummary policy)`
- `CandidateScore(String workerId, double trustScore, double workloadScore, String phase, int observations, double finalScore, String exclusionReason, String rationale)`
- `RoutingPolicySummary(double threshold, double borderlineMargin, double blendFactor, Map<String, Double> qualityFloors)`
- `GateDecision(String decision, String actor, String attestation, double trustScoreBefore, double trustScoreAfter, String dimension)`

---

## Frontend: Reviewers View Expansion

### Layout change

Replace the flat `dataTable` in `reviewers.ts` with a split-pane layout:

- **Left pane:** Reviewer fleet table (existing columns + new inline trust badge)
  - Add `trustScore` column using `<blocks-trust-score-panel mode="compact">`
    with pre-fetched score and trustLevel from the `/api/governance/reviewers`
    response (avoids redundant fetch)
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
