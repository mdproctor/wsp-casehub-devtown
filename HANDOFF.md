# Handoff — devtown#30 shipped: HITL integration test

2026-05-21

## What shipped this session

**Fruit-picking (devtown#28, #29, #32)** — all closed in the same session start:
- `PrFinding` + `PrVerdict` types added to `review` module (Layer 1 API stability)
- Missing `doesNotFire_whenAnalysisNotComplete` tests for parallel checks
- Method references + fixture extraction in test classes
- REST integration test for `PrReviewResource`

**devtown#30 (HITL end-to-end test)** — closed. `HumanApprovalLifecycleTest` verifies:
- `humanTask` binding creates a WorkItem when `linesChanged > threshold`
- CDI `@ObservesAsync` delivery works in this runtime (via `WorkItemCompletionCapture`)
- `WorkItemLifecycleAdapter` applies `outputMapping { humanApproval: . }` → case context updated
- Case completes when all goals satisfied after signalling remaining checks

**Local engine fixes applied (not committed to engine repo):**
- `HumanTaskScheduleHandler` — PENDING guard relaxed (accepts RUNNING, engine#312)
- `WorkItemLifecycleAdapter` — `runOnSafeVertxContext` removed, direct `.await()` (engine#316)
- `LedgerProcessor` — Quarkus 3.32.2 build-step incompatibility fixed

**Garden (3 entries):** GE-20260521-a0f5a6, GE-20260521-9188c1 (YAML `when:` never evaluated), GE-20260521-87daa0 (@ObservesAsync external jar)

**Protocols (2):** PP-20260521-134c38 (HITL test context pre-seeding), PP-20260521-a36692 (MemoryPlanItemStore in selected-alternatives)

## Immediate next step

Get the engine team to commit the local engine fixes to the engine repo:
- `HumanTaskScheduleHandler` PENDING guard (engine#312)
- `WorkItemLifecycleAdapter` direct await (engine#316)
- `LedgerProcessor` Quarkus 3.32.2 fix

These changes are in `/Users/mdproctor/claude/casehub/engine/` locally but NOT pushed.

## Cross-Module

**Blocked by:**
- `engine` — engine#312 (double WorkItem from PlanningStrategyLoopControl), engine#314 (nested evalObjectTemplate), engine#315 (@ObservesAsync external jar), engine#316 (runOnSafeVertxContext) all need engine fixes · M · Med

## What's Left

- devtown#31 — `quarkus:build` fails without claudony (engine SPIs unsatisfied) · XS · Low — blocked by claudony integration

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| Layer 2 | HITL WorkItem SLA — human review gate SLA and escalation | L | High | devtown#30 done, Layer 2 is next logical step |
| Layer 6 | Trust-weighted routing | L | High | Blocked by P1.3 (TrustWeightedSelectionStrategy) |

## Key references

- Blog: `blog/2026-05-21-mdp01-one-test-five-discoveries.md` (published — all 159 entries across 9 workspaces verified published)
- Spec: project `docs/specs/2026-05-21-hitl-human-approval-lifecycle-design.md`
- Garden: GE-20260521-9188c1 — YAML `when:` conditions never evaluated at runtime
