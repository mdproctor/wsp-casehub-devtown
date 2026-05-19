# Handoff — Epic 3 shipped: PR review CasePlanModel

2026-05-19

## What shipped this session

**Layer 1 Part B (devtown#27)** — `NaivePrReviewService @DefaultBean`, `PrReviewResource`, 5 unit tests. Teaching baseline that Layer 5 displaces.

**Epic 3 / Layer 5 (devtown#10)** — PR review CasePlanModel fully shipped and pushed to main:
- `review/src/main/resources/devtown/pr-review.yaml` — 9 bindings, 3 goals, 9 capabilities
- `PrReviewCaseHub extends YamlCaseHub` + `PrReviewCaseService @ApplicationScoped` (displaces NaivePrReviewService via @DefaultBean pattern)
- 38 unit tests (28 binding condition + 10 goal condition) — pure Java, no Quarkus
- 5 `@QuarkusTest` YAML round-trip tests
- `InMemoryLedgerEntryRepository` stub in test sources (needed for CDI — see GE-20260519-e13b01)
- Critical bug found and fixed in code review: `pr-approved` goal was missing `securitySensitive == false` guard — non-security PRs could never complete

**Platform protocols added to casehub-parent:**
- `docs/protocols/casehub/case-definition-layers.md` — three-layer YAML/schema/canonical architecture (inherited from SW 1.0)
- `docs/PLATFORM.md` — upstream consistency principle (SW 1.0 / quarkus-flow)
- Engine issue engine#289 opened for pluggable ExpressionEvaluatorFactory

## Branch state (⚠️ mismatch — known)

- **Project repo** (`casehub/devtown`): on `main` — all Epic 3 work committed and pushed
- **Workspace** (`public/casehub/devtown`): on `epic-pr-review-case` — has `design/.meta` (issue: 10), plan, and blog

The mismatch is intentional (merged early, continued on main). Workspace epic branch needs formal `/epic close` before next epic begins.

## Open issues from this session

| Issue | What | Priority |
|-------|------|----------|
| devtown#28 | Layer 1 API stability (PrReviewOutcome.findings type + PrVerdict constant) | Before Layer 2 |
| devtown#29 | Layer 1 test improvements (fixture + REST integration test) | Low |
| devtown#30 | HITL end-to-end test once casehub-work-adapter wiring lands | After P1.2 |
| devtown#31 | quarkus:build fails — engine SPIs unsatisfied without claudony | After claudony integration |
| devtown#32 | Minor test improvements (parallel check coverage + method reference style) | Low |
| parent#26 | casehub-platform-api scoped preferences SPI | Pending |
| engine#289 | CaseDefinitionYamlMapper pluggable ExpressionEvaluatorFactory | Pending |

## Next

**Workspace housekeeping first:** run `/epic` (close mode) on workspace `epic-pr-review-case` to promote specs, archive plans, and clean up the branch. Then start the next epic.

**Next layer candidates:**
- Layer 2 (casehub-work): PR review WorkItem with SLA — formal human task lifecycle
- Layer 6 (trust routing): TrustWeightedSelectionStrategy — needs P1.3

## Key references

- Blog: `blog/2026-05-19-mdp01-layer-5-case-definition-lands.md`
- Spec: project `docs/specs/2026-05-19-epic3-pr-review-caseplanmodel-design.md`
- Plan: workspace `plans/2026-05-19-epic3-pr-review-caseplanmodel.md`
- LAYER-LOG: project `LAYER-LOG.md` §Layer 5 (complete)
- Garden: GE-20260519-e13b01 — InMemoryLedgerEntryRepository CDI gotcha
