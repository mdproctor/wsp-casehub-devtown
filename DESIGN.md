# DevTown Design — casehub-devtown

**Role:** Cross-cutting architectural decision record. Captures the *why* behind
technical choices that affect multiple layers or don't belong in a single layer entry.
Distilled from `design/JOURNAL.md` at each epic close. Each entry is cross-referenced
from the relevant `LAYER-LOG.md` layer entry and vice versa.

**Not a design spec** (those live in `docs/specs/`). Not an ADR (those in `docs/adr/`).
Not session narrative (that is `blog/`). See `AGENTIC-HARNESS-GUIDE.md §Three-Document
Design System` for the full chain.

Sections updated as epics close via journal merge.

---

## Layer 2 SLA Breach Policy — Design

**Date:** 2026-05-22  
**Spec:** `docs/specs/2026-05-22-layer2-sla-breach-policy-design.md`

### Key decisions

**SlaBreachPolicy SPI lives in `casehub-work-api`, not `platform/apps-api`.**  
Initially planned for a new `platform/apps-api` module. Revised: every consumer of the SPI already has `casehub-work-api` on their classpath; `BreachedTask` mirrors WorkItem fields; the SPI is intrinsically tied to WorkItem expiry. `platform/apps-api` is deferred until a genuinely work-independent cross-app SPI surfaces. ADR filed in platform repo (ADR-0007).

**`BreachDecision` sealed interface with `Chained` as a separate type.**  
Alternative was embedding `thenOnBreach` on `EscalateTo` and `Extend`. Chose separate `Chained` record: each decision type stays pure (no nullable continuations), composition is explicit, and the executor handles chaining in one switch arm. `thenOnBreach()` default interface method provides fluent construction.

**Stateless multi-tier escalation via `candidateGroups`.**  
When the escalated WorkItem expires, `ExpiryLifecycleService` calls `onBreach()` again. The policy reads `ctx.task().candidateGroups()` to determine the current tier — no decision tree serialization, no state storage. If escalation group is already assigned → terminal failure; otherwise → escalate. Devtown's `DefaultSlaBreachPolicy` implements this pattern.

**Scope resolution uses `WorkItem.scope` field (new V31 column), with `Path.root()` as null fallback.**  
`buildBreachContext()` calls `Path.root()` when `WorkItem.scope` is null. `Path.root()` is not yet on platform main — blocks `work#212` compilation until platform publishes it. Devtown WorkItems created by `HumanTaskScheduleHandler` won't have scope set until engine#330 ships; the `SlaBreachPolicy` implementation in devtown-app must enrich scope from `callerRef` independently.

**`EscalateTo.deadline` controls escalated task lifetime for COMPLETION_EXPIRED only.**  
For CLAIM_EXPIRED, `computeNewClaimDeadline()` governs the new deadline regardless of `deadline` field — `ClaimSlaPolicy` manages the pool SLA. Documented in EscalateTo Javadoc.

### Deferred

- engine#325 — `HumanTaskTarget.claimDeadlineHours` (declarative claim SLA in YAML)
- engine#326 — failure goal support in CasePlanModel YAML
- engine#327 — dynamic `expiresIn` from runtime context
- engine#330 — `HumanTaskTarget.scope` for scope propagation to WorkItem
- work#211 — `WorkItemService.extend()` for `BreachDecision.Extend`
- platform#24 — `apps-api`/`apps` modules (deferred; SPI placed in work-api instead)

### Review findings

Three reviews of the casehub-work `SlaBreachPolicy` design produced these corrections applied by the work team:

1. `Path.of()` zero-arg throws → `Path.root()` needed (not yet on platform main)
2. `preferenceProvider.preferencesFor()` → `resolve()` (wrong method name)
3. `EscalateTo` missing `deadline` field → added `@Nullable Duration deadline`
4. `SlaBreachEvent` carrying `Chained` wrapper → now fires the leaf decision always
5. `BreachExecutionFailed` escape to `@Transactional` boundary → protocol PP-20260522-f08b62 + garden GE-20260522-4e806e
