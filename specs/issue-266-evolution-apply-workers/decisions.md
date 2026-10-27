# Design Decisions — Issue #266: Evolution Apply-Stage Workers

## D1: Apply execution model — direct preference writes

**Choice:** Workers write to PreferenceProvider directly in the apply stage
**Alternatives:**
- PR-based config changes — produces YAML diffs as PRs with git history audit, but slower and fights the self-improvement template's operational flow assumptions
- Devtown-specific case template — custom template skipping PR stages, but adds template maintenance overhead without adding value over the existing conductor gating
**Rationale:** The conductor's gated propose and review stages already provide human oversight before apply fires. The case lifecycle (EventLog, conductor inbox, worker output with old/new values, CaseLedgerEntry Merkle audit) provides richer auditability than git history alone — it links each change to the health signal that triggered it, the human approval that authorized it, and the regression evaluator that monitors impact.
**Trade-offs:** No git-diffable history of preference changes; audit trail lives in the case system rather than version control. Mitigated by worker output recording before/after values in the case context.
**Sources:** DevtownCategoryProvider stages (analyze→propose[gated]→review[gated]→apply→observe), self-improvement.yaml operational branch, PreferenceProvider JPA persistence (platform#24), CaseLedgerEntry (devtown#73)
**Exploration:** quick
**Status:** captured

## D2: Hardcoded value migration scope — migrate as needed

**Choice:** Migrate hardcoded tuning values to PreferenceProvider incrementally, as each worker needs them
**Alternatives:**
- Preferences only — workers only adjust existing PreferenceProvider values; hardcoded values tracked as separate issue. Reduces scope but leaves some workers unable to tune their primary targets.
- Migrate all upfront — migrate all hardcoded tuning values before building workers. Clean but adds significant scope and delays worker delivery.
**Rationale:** Each worker knows exactly which values it needs. Migrating per-worker keeps the change focused and avoids migrating values no worker actually adjusts. The migration is mechanical (add PreferenceKeys constant, read from PreferenceProvider with hardcoded default as fallback).
**Trade-offs:** Migration happens across multiple worker implementations rather than as a clean preparatory step. Acceptable because each migration is small and self-contained.
**Depends on:** D1 (direct preference writes — migration only needed because workers write preferences)
**Sources:** DevtownCapabilityRegistry.POLICIES (domain module), DefaultBatchCompositionPolicy.HIGH_RISK_THRESHOLD (queue module), MergeQueuePreferenceKeys (already preference-based)
**Exploration:** quick
**Status:** captured

## D3: Worker wiring — one worker per category

**Choice:** 5 dedicated workers, each registered for its category's capabilities
**Alternatives:**
- Generic dispatcher + strategy — one CDI bean reads category from input and dispatches to strategy classes. Fewer beans but adds a routing layer on top of the engine's existing capability-based routing.
- Introspect/apply split — 5 introspection classes paired with a generic apply worker. Separates analysis from execution but creates artificial coupling through the shared output format.
**Rationale:** Each category has distinct tuning targets (different preference keys, different health metrics, different adjustment algorithms). Dedicated workers keep each category's logic self-contained and testable in isolation. The engine's capability-based dispatch already handles routing — no need for another routing layer.
**Trade-offs:** More CDI beans (5 vs 1). Acceptable given each worker is focused and small.
**Depends on:** D1 (workers write preferences directly)
**Sources:** DependencyUpdateWorker pattern (engine), DevtownCategoryProvider 5 categories, self-improvement.yaml capability bindings
**Exploration:** quick
**Status:** captured
