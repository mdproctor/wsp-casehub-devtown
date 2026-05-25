# Handoff — devtown#42 closed, Layer 2 wiring test shipped

2026-05-25

## What shipped this session

**devtown#42 closed** — `SlaBreachHandlerWiringTest` — two-test CDI wiring verification:
- `slaBreachPolicyBean_displaces_noOp()` — type assertion, no alternatives needed
- `slaBreachHandler_onFail_signalsCaseContext()` — `Event<SlaBreachEvent>` fired directly as test driver, bypasses `ExpiryLifecycleService`
- `CapturingBreachPolicy @Alternative` approach (from the issue spec) rejected — `selected-alternatives` is build-time, would displace `SlaBreachPolicyBean` and break the displacement check
- `linesChanged=100` (below threshold) eliminates all async binding activity after `startCase()` resolves
- `MapPreferences.empty()` does not exist despite `SlaBreachContext` Javadoc saying it does — use `new MapPreferences(Map.of())` — filed as GE-20260525-c715ae
- 15/15 tests passing; PR casehubio/devtown#44 open

**LAYER-LOG.md Layer 5 blog reference** — fixed at session start (filename was wrong placeholder)

## Immediate next step

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- LAYER-LOG.md Layer 2 entry — write in full when engine#326 (failure goal) ships · M · Low
- parent#68 — layer table update: Layer 2 code complete (devtown#41 ✅, devtown#42 ✅); LAYER-LOG pending engine#326 · XS · Low (peer repo)
- casehubio/devtown#44 — merge upstream PR · XS · Low

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Key references

- Blog: `blog/2026-05-25-mdp01-the-spec-said-capturingbreachpolicy.md`
- Garden: GE-20260525-c715ae (`MapPreferences.empty()` gotcha), GE-20260525-f04530 (`Event<T>.fire()` as `@Observes` test driver)
- Wiring test spec: `docs/specs/2026-05-25-sla-breach-handler-wiring-test-design.md`
- Stale workspace branches (all EPIC-CLOSED, deletion eligible 2026-06-08): `epic-pr-review-case`, `issue-30-hitl-human-approval-test`, `issue-38-layer2-sla-escalation`
- Stale project branches (all < 14 days): `backup/pre-squash-main-20260523`, `epic-pr-review-case`, `issue-293-wire-work-adapter-hitl`, `issue-30-hitl-human-approval-test`, `issue-38-layer2-sla-escalation`
