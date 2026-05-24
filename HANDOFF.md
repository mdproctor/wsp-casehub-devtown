# Handoff — Layer 2 shipped, devtown#40 closed, CI green, all work closed

2026-05-24

## What shipped this session

**devtown#41 closed** — Layer 2: SLA-bounded human review gate with escalation:
- `DefaultSlaBreachPolicy` — stateless two-tier escalation via `candidateGroups`
- `SlaBreachPolicyBean @ApplicationScoped` — displaces `NoOpSlaBreachPolicy`
- `SlaBreachHandler @Observes SlaBreachEvent` — signals case on terminal Fail
- `pr-review.yaml` — `candidateGroups: [pr-reviewers]`, `expiresIn: PT24H`
- Key insight: `SlaBreachEvent` fired via `Event.fire()` (synchronous) — must use `@Observes` not `@ObservesAsync` (Javadoc example was wrong; filed work#224)

**devtown#40 closed** — Engine persistence SPIs wired via `@ApplicationScoped` subclasses in `app/spi/`:
- `quarkus.arc.selected-alternatives` silently does nothing during `quarkus:build` — filed as GE-20260524-2b587e
- Non-`@Alternative` subclass in app module is `@Default`, always indexed — filed as GE-20260524-d75218

**CI green** — Missing `<repositories>` in devtown parent pom was blocking all CI since repo creation. Maven can't resolve the parent POM without knowing where to look, and the repo URL is defined in the parent. Added wildcard entry to devtown's own pom — GE-20260524-122018.

**Both remotes pushed** — mdproctor and casehubio on same HEAD.

**All workspace branches properly closed** — added missing `design/EPIC-CLOSED.md` to `epic-pr-review-case`; all three stale branches now have the work-end marker.

**All 7 blog entries published** — `2026-05-24-mdp01-four-subclasses-missing-repo.md` pushed to mdproctor.github.io.

## Immediate next step

Start Layer 3: `work-start` for casehub-qhorus typed COMMAND/RESPONSE/DONE/DECLINE per reviewer agent interaction. Check for open issues first — no #41 equivalent exists yet, will need to create one.

## What's Left

- LAYER-LOG.md Layer 5 blog reference — update pointer from placeholder to actual entry · XS · Low
- LAYER-LOG.md Layer 2 entry — write in full when layer closes (engine#326 failure goal is the remaining gap) · M · Low
- devtown#42 — SlaBreachHandlerWiringTest (focused CDI wiring test) · S · Low
- parent#63 — update casehub-devtown.md layer table (Layer 2 pending → in progress) · XS · Low (peer repo issue, not us)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| Layer 3 | casehub-qhorus typed COMMAND/RESPONSE/DONE/DECLINE per reviewer agent | L | High | Design first; read qhorus#124 status (claudony persona mapping) |
| engine#326 | Failure goal support (`kind: failure`) | M | Med | Needed to close Layer 2 fully; work on engine side |
| Layer 2 LAYER-LOG | Write Layer 2 entry in full when engine#326 done | M | Low | See LAYER-LOG.md convention — full at close only |

## Key references

- Blog: `blog/2026-05-24-mdp01-four-subclasses-missing-repo.md`
- Garden: GE-20260524-2b587e (selected-alternatives silent failure), GE-20260524-d75218 (CDI @Alternative subclass technique), GE-20260524-122018 (Maven parent bootstrap), GE-20260524-baae14 (BOM scope inheritance)
- Layer 2 spec: `docs/specs/2026-05-22-layer2-sla-breach-policy-design.md`
- Stale project branches (all < 14 days, deletion scheduled): `backup/pre-squash-main-20260523`, `epic-pr-review-case`, `issue-293-wire-work-adapter-hitl`, `issue-30-hitl-human-approval-test`, `issue-38-layer2-sla-escalation`
