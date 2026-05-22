# Handoff — Layer 2 SLA design complete; blocked on platform publish

2026-05-22

## What shipped this session

**CLAUDE.md + LAYER-LOG.md aligned** with AML tutorial-layer discipline: Design Phase References table added, Tutorial Structure with layer status, stale Foundation Gates corrected, LAYER-LOG.md header cleaned of placeholder mechanism.

**Layer 2 design complete (devtown#38)** — spec at `docs/specs/2026-05-22-layer2-sla-breach-policy-design.md`. `SlaBreachPolicy` SPI in `casehub-work-api` (not platform), sealed `BreachDecision` with `Chained` as separate type, stateless multi-tier escalation via `candidateGroups`, `SlaBreachHandler` in devtown-app, two-tier `SlaBreachLifecycleTest` strategy.

**Cross-session code review of casehub-work design (3 rounds)** — surfaced: `Path.root()` not on platform main (compile blocker), `BreachExecutionFailed` escape to `@Transactional` boundary (silent infinite retry), `EscalateTo` missing `deadline` field. All incorporated by work team.

**devtown#39** — `casehub-platform-api` explicit compile dep + `casehub-platform-testing` test dep added to `devtown-app/pom.xml`.

**Garden:** GE-20260522-9cd6d5 (Path.root compile blocker), GE-20260522-4e806e (BreachExecutionFailed retry), GE-20260522-f7db12 (stateless tier escalation), GE-20260522-de5ee3 (Chained as separate sealed type).

**Protocols:** PP-20260522-fe93b6 (cross-repo design doc state verification), PP-20260522-f08b62 (transactional loop exception safety).

## Immediate next step

Watch `casehub-platform` for `Path.root()` landing on main and being published. That unblocks `work#212` (SlaBreachPolicy wiring), which unblocks `devtown#38` implementation. Until then, nothing compiles.

When it lands: start Layer 2 implementation with `work-start` on this branch — spec is approved, plan not yet written (invoke `superpowers:writing-plans` from the spec).

## Cross-Module

**Blocked by:**
- `platform` — `Path.root()` not on any branch; must publish before work#212 compiles · S · Low
- `work` — work#213 (SlaBreachPolicy SPI), work#212 (wiring) blocked on platform publish · M · Med
- `engine` — engine#325 (claimDeadlineHours), engine#326 (failure goals), engine#327 (dynamic expiresIn), engine#330 (scope propagation) · M · Med

## What's Left

- LAYER-LOG.md Layer 5 blog reference still has `🔲 (not yet drafted)` — blog was written 2026-05-19; update the pointer · XS · Low
- devtown#35 — duplicate `GET /.well-known` endpoint in `@QuarkusTest` (ADR-0007 not applied) · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #38 | Layer 2 implementation — `SlaBreachPolicy`, `SlaBreachHandler`, YAML SLA fields | L | Med | Blocked on platform publish of Path.root() + work#212/213 shipping |
| #38 | Write implementation plan (invoke `superpowers:writing-plans` from the approved spec) | S | Low | Can do now while blocked on compile dep |
| Layer 3 | casehub-qhorus — typed COMMAND/RESPONSE per reviewer agent | L | High | After Layer 2 ships |

## Key references

- Spec: `docs/specs/2026-05-22-layer2-sla-breach-policy-design.md`
- Blog: `blog/2026-05-22-mdp01-three-reviews-one-contract.md`
- Garden: GE-20260522-4e806e — BreachExecutionFailed silent retry (most important for implementation review)
