# HANDOFF — devtown

## Last Session

Code review audit of 28 commits (week of 2026-10-20). Created epic #269 with 15 findings across evolution, auth, agent pipeline, and rename areas. Fixed #270 (missing `@POST /watch-patterns` endpoint) — landed as `9118627` on main. Remaining 14 issues organized into 4 batches below.

## Immediate Next Step

Start with **Batch 4** (XS — rename comment cleanup), then work through batches 3→2→1 in order.

## Batched Plan — Epic #269

### Batch 1: Evolution workers (M1 + M2 + L6 + L7 + L8)

**Branch:** `issue-269-evolution-worker-fixes`
**Scale:** M | **Complexity:** Med

| Issue | Summary | Notes |
|-------|---------|-------|
| #271 (M1) | Workers hardcode current preference values | Design `PreferenceReader` or pass current value through accumulated state. Acceptable as-is in Phase 0 while `PreferenceWriter` is a logging stub — defer if preference persistence isn't imminent. |
| #274 (M2) | Missing failure-path tests for evolution workers | Add tests: missing `tenancyId` (all 5), missing `target` (2 workers), unknown category dispatch, boundary at `TARGET_HEALTH=0.6`. |
| #280 (L6) | Duplicated `assess()` across 5 capability areas | Extract base class or shared method. Intentional scaffold — defer if areas will diverge soon. |
| #281 (L7) | Custom duration format in `SlaThresholdWorker` | Switch to `java.time.Duration.toString()` if consumers expect ISO-8601. |
| #282 (L8) | Workers return `status=MERGED` unconditionally | Rename to reflect actual semantics (e.g. `APPLIED` or `PROPOSED`). |

**Decision needed for M1:** Is preference persistence imminent? If yes, implement `PreferenceReader`. If no, defer M1 and document the gap.

### Batch 2: Auth hardening (M4 + L3 + L4 + L5)

**Branch:** `issue-269-auth-hardening`
**Scale:** S | **Complexity:** Low

| Issue | Summary | Notes |
|-------|---------|-------|
| #273 (M4) | Non-404 GitHub errors silently deny access | Throw on non-404 errors. Add cold-cache failure test. |
| #277 (L3) | Hardcoded `per_page=100` | Add pagination or document limitation. |
| #278 (L4) | Untyped `List<Map<String, Object>>` in `GitHubOrgApi` | Create `GitHubTeamMember` record. |
| #279 (L5) | No `@RolesAllowed` on `UserInfoResource` | Add `@RolesAllowed("**")` annotation. |

### Batch 3: Agent pipeline config (M3 + L1 + L2)

**Branch:** `issue-269-agent-config-cleanup`
**Scale:** XS | **Complexity:** Low

| Issue | Summary | Notes |
|-------|---------|-------|
| #272 (M3) | `defaults.backend: claude` references absent backend | Change to `simulation` or a tier query. |
| #275 (L1) | `cheap-fast` uses explicit `backend-key: ollama` | Align with tier-based direction. |
| #276 (L2) | Stale TODO for `agent-router-core` version | File upstream platform issue, update or remove TODO. |

### Batch 4: Rename comment cleanup (L9 + L10)

**Branch:** `issue-269-rename-comment-cleanup`
**Scale:** XS | **Complexity:** Low

| Issue | Summary | Notes |
|-------|---------|-------|
| #283 (L9) | 6 stale "blocks" comments in properties + CI | Find-and-replace in comments. |
| #284 (L10) | `contributor-guide.md` says "blocks framework" | Same fix session as L9. |

## Recommended Order

1. **Batch 4** (XS) — warmup, zero-risk comment fixes
2. **Batch 3** (XS) — config alignment, low risk
3. **Batch 2** (S) — auth hardening with one genuine behavioral change (M4)
4. **Batch 1** (M) — evolution worker improvements, includes a design decision (M1)

## References

- Epic: casehubio/devtown#269
- Landed: #270 as `9118627`
