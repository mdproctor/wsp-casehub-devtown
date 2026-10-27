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

## D4: Self-improvement template compatibility — signal completion directly

**Choice:** The implement worker applies the preference change and emits the `improvement:pr-submitted` signal with status `MERGED` so downstream bindings resolve cleanly
**Alternatives:**
- Skip bindings via context flag — set `improvementType` to `"preference"` and add when-guards. Requires engine template modification, which is cross-repo scope.
- No-op passthrough workers — register stubs for submit-pr, review, integrate that do nothing. All stages fire but meaninglessly, adding noise to the case EventLog.
**Rationale:** The implement worker is the right place to apply the change (D1) and signal completion. By emitting the expected signal with `MERGED` status, the template's existing bindings fire naturally — `review` sees `MERGED` and `integrate` proceeds. The conductor gates already provided the human oversight equivalent to code review. No engine template changes needed.
**Trade-offs:** The review and integrate bindings still fire, but they process a pre-approved outcome rather than doing substantive work. This is acceptable — the EventLog accurately reflects that the change was applied, and outcome recording works correctly.
**Depends on:** D1 (direct preference writes), D3 (per-category workers)
**Sources:** self-improvement.yaml binding chain (introspect→implement→submit-pr→review→integrate→record-outcome), ImprovementOutcomeWorkerFunction (record-outcome stage)
**Exploration:** quick
**Status:** captured

## D5: Module placement — app/evolution/

**Choice:** Workers live in `app/src/main/java/io/casehub/devtown/app/evolution/worker/` alongside existing evolution infrastructure
**Alternatives:**
- review/ module — alongside PrReviewApplicationService. But review/ doesn't own evolution today, and workers need CDI injection of PreferenceProvider which is wired at app level.
- New evolution/ module — separate build module for evolution workers. Clean boundary but adds build complexity for 5 small classes that belong with their existing evolution peers (DevtownProposalSource, capability areas, etc.).
**Rationale:** Follows the existing pattern. All evolution infrastructure (DevtownProposalSource, 5 capability areas, DevtownCategoryProvider, DevtownConflictStrategy, DevtownDenyPatternProvider, DevtownRegressionEvaluator) already lives in `app/evolution/`. Workers are the natural companion — they execute what the proposal source proposes. A `worker/` subdirectory keeps them visually distinct.
**Trade-offs:** App module grows slightly. Acceptable — the evolution package is already the largest logical group in app/ and adding 5 focused workers doesn't change its cohesion.
**Depends on:** D3 (one worker per category)
**Sources:** app/src/main/java/io/casehub/devtown/app/evolution/ (22 existing files), three-tier module rule in CLAUDE.md
**Exploration:** quick
**Status:** captured

## D6: Adjustment algorithm — proportional with CBR auto-promotion

**Choice:** `AdjustmentStrategy` interface with two implementations. `ProportionalAdjustmentStrategy` is the default — adjusts values proportionally to the health gap with configurable bounds. `CbrInformedAdjustmentStrategy` activates automatically when precedent count >= configurable minimum (e.g. 5 per category). CBR queries past adjustment outcomes (same category, similar health gap) and recommends the value that produced the best health improvement. CBR recommendations are still constrained by proportional bounds (safety guardrail).
**Alternatives:**
- Weighted blend — always compute both, weight shifts as precedents grow. More complex (two algorithms always run) with marginal benefit over a clean threshold switch.
- CBR with proportional fallback — always try CBR first, fall back when insufficient precedents. Unnecessary CBR queries during bootstrap when they'll always miss.
**Rationale:** Mirrors devtown's trust maturity model — Phase 0 (no observations) uses simple routing, quality improves as attestations accumulate. Each completed improvement cycle records structured data (parameter, oldValue, newValue, healthBefore, healthAfter) via ImprovementOutcomeEventCapture, which bootstraps the CBR store. The engine already has ImprovementCbrProjector for this. SlaEstimator in devtown already uses the same CBR-from-precedents pattern.
**Trade-offs:** Early adjustments use a simpler algorithm that may not be optimal. Acceptable — the observe stage and regression evaluator catch poor outcomes, and each outcome improves the CBR store for next time.
**Depends on:** D1 (preference writes), D3 (per-category workers)
**Sources:** ImprovementCbrProjector (engine), ImprovementOutcomeEventCapture (engine), SlaEstimator CBR pattern (devtown domain), trust maturity model (CLAUDE.md)
**Exploration:** quick
**Status:** captured
