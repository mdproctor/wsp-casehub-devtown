# ActionRiskClassifier Oversight Gate — Design Spec

**Issue:** devtown#56
**Foundation:** engine#402 (shipped)
**Layer:** 5 extension (casehub-engine SPI implementation)
**Date:** 2026-06-15 (revised after round 1 review)

## Problem

Devtown orchestrates software engineering workflows where agents take
consequential actions — merging PRs, escalating security findings, modifying
contributor access. The engine provides an `ActionRiskClassifier` SPI
(engine#402) that lets applications declare which actions require human
oversight before proceeding. Devtown needs domain-specific risk rules for
its action categories.

## SPI Summary

```java
// Engine SPI (casehub-engine-api)
interface ActionRiskClassifier {
    RiskDecision classify(PlannedAction action);
}

record PlannedAction(String workerId, UUID caseId,
    String description, String actionType, Map<String, Object> context) {}

sealed interface RiskDecision {
    record Autonomous() implements RiskDecision {}
    record GateRequired(String reason, boolean reversible,
        List<String> candidateGroups, Duration expiresIn, String scope)
        implements RiskDecision {}
}
```

The engine's `ChainedReactiveActionRiskClassifier` discovers all
`@RiskClassifier`-qualified beans and chains them. When two `GateRequired`
results compete, the chainer picks the most restrictive — defined as the
one with fewer candidate groups (tighter scope), breaking ties by shorter
expiration. If no classifiers are registered, actions proceed autonomously.
On exception, the chainer applies a fail-safe `GateRequired`.

### Engine Lifecycle Flow

Classifier returns `GateRequired` → engine stores `PendingActionGate` →
emits `ActionGateScheduleEvent` → `ActionGateWorkItemHandler` (in
casehub-engine-work-adapter, already a devtown dependency) creates a
`WorkItem` → human approves → `ActionGateApprovedHandler` re-fires the
worker completion.

### ActionGatePolicy Alignment

The engine defines `ActionGatePolicy { ALWAYS, THRESHOLD, CONDITIONAL }`
which maps directly to the spec's classification categories — validating
the categorisation against the engine's own taxonomy.

## Action Type Catalog

| Constant | Action Type String | Description |
|----------|-------------------|-------------|
| `PR_MERGE_EXECUTE` | `pr-merge-execute` | Merge a PR to the target branch |
| `PR_FORCE_MERGE` | `pr-force-merge` | Merge overriding failed checks |
| `PR_REVIEW_OVERRIDE` | `pr-review-override` | Override a reviewer's decision |
| `SECURITY_ESCALATION` | `security-escalation` | Block PR and alert on security finding |
| `ISSUE_CLOSE_INVALID` | `issue-close-invalid` | Close issue as invalid |
| `DEPENDENCY_REMOVAL` | `dependency-removal` | Remove a flagged dependency |
| `CONTRIBUTOR_ACCESS_CHANGE` | `contributor-access-change` | Add/remove contributor access |
| `PRODUCTION_DEPLOY` | `production-deploy` | Deploy to production |
| `GENERAL_OVERSIGHT` | `human-oversight:general` | Catch-all oversight group for actions without a specific group |

`GENERAL_OVERSIGHT` is a candidateGroup constant, not an action type. It
ensures WorkItems always have group scoping — `null` candidateGroups would
allow any user to claim the item, not just qualified humans.

## Classification Categories

Four categories, mapped to `ActionGatePolicy`:

### 1. Always-gate (ALWAYS)

Always return `GateRequired` unless explicitly disabled via preference.

| Action | Reason | Reversible | candidateGroups |
|--------|--------|------------|-----------------|
| `pr-force-merge` | Bypasses safety checks — requires explicit human approval | false | `human-decision:pr-approval` |
| `contributor-access-change` | Access changes are consequential regardless of scope | false | `human-oversight:general` |

### 2. Minimum-requirement (THRESHOLD, inverted comparison)

Compare a safety metric from `PlannedAction.context()` against a required
minimum. **Below minimum → `GateRequired`. At or above → `Autonomous`.**

This is the inverse of risk-threshold actions: the metric measures safety
(more = safer), not risk (more = riskier).

| Action | Context Key | Minimum Default | Reversible | candidateGroups | Rationale |
|--------|------------|-----------------|------------|-----------------|-----------|
| `pr-merge-execute` | `approvedReviewCount` | 1 | false | `human-decision:pr-approval` | Zero approvals must never proceed |

### 3. Risk-threshold (THRESHOLD, standard comparison)

Compare a risk metric from `PlannedAction.context()` against a configurable
threshold. **At or above threshold → `GateRequired`. Below → `Autonomous`.**

| Action | Context Key | Threshold Default | Reversible | candidateGroups | Rationale |
|--------|------------|-------------------|------------|-----------------|-----------|
| `security-escalation` | `severity` | `HIGH` (via `IncidentSeverity` ordinal) | true | `human-oversight:routing-review` | LOW/MEDIUM can auto-report |
| `issue-close-invalid` | `commentCount` | 5 | true | `human-oversight:general` | Low engagement can auto-close |
| `dependency-removal` | `transitiveUsageCount` | 3 | false | `human-oversight:general` | Narrow-use deps can auto-remove |
| `production-deploy` | `modulesAffected` | 3 | false | `human-oversight:routing-review` | Single-module deploys can auto-proceed |

The `security-escalation` threshold uses `IncidentSeverity` (domain enum
with `LOW(0.3)`, `MEDIUM(0.5)`, `HIGH(0.7)`, `CRITICAL(0.9)`) for
type-safe ordinal comparison. The context value is parsed to the enum;
the threshold preference is stored as a severity level name and also
parsed to the enum. Both sides are the same type.

### 4. Conditional (CONDITIONAL)

Classification depends on inspecting context values, not comparing against
a numeric threshold.

| Action | Rule | Reversible | candidateGroups |
|--------|------|------------|-----------------|
| `pr-review-override` | `GateRequired` when `context.get("originalVerdict")` is `"REJECTED"` (overriding a rejection is consequential). `Autonomous` when `"APPROVED"`, `"PENDING"`, or absent (no reviewer decision being overturned) | true | `human-decision:pr-approval` |

### 5. Unknown action type (fail-safe)

Unknown action types return `GateRequired("Unknown action type — manual
review required", true, List.of(GENERAL_OVERSIGHT), Duration.ofHours(24),
actionType)`. Consistent with the engine's fail-safe principle — unknown
actions must not proceed autonomously.

## PreferenceProvider Integration

### Scope/Key separation

Two separate concepts, following the established `DevtownTrustRoutingPolicyProvider` pattern:

**SettingsScope** — resolved via `PreferenceProvider.resolve()` to get a `Preferences` bag:
```java
Preferences prefs = preferenceProvider.resolve(
    SettingsScope.of("casehubio", "devtown", "risk", actionType));
```

**PreferenceKey** — looks up a specific value within the resolved `Preferences`:
```java
BooleanPreference enabled = prefs.getOrDefault(RiskPreferenceKeys.ENABLED);
```

### Preference keys per action type

Each action type's `Preferences` bag contains:

| Key name | Type | Default | Notes |
|----------|------|---------|-------|
| `enabled` | `BooleanPreference` | `true` | Escape hatch — `false` returns `Autonomous` regardless |
| `expiresInMinutes` | `IntPreference` | 240 (reversible) / 1440 (irreversible) | Per-action-type override, no separate global defaults |

Threshold-gated and minimum-requirement actions add:

| Key name | Type | Default | Notes |
|----------|------|---------|-------|
| `threshold` | `IntPreference` or `StringPreference` | Per action type (see tables above) | Each classifier method reads its own key directly |

### Namespace convention

All keys use namespace `casehubio.devtown.risk` — consistent with the
majority pattern (`casehubio.devtown.trust-gate`,
`casehubio.devtown.trust-routing`). The `devtown.sla` prefix is a legacy
inconsistency — not addressed in this issue.

### New preference types required

**`BooleanPreference`** — new record in `domain/preferences/` following
the existing pattern:

```java
public record BooleanPreference(boolean value) implements SingleValuePreference {
    public static BooleanPreference of(boolean value) { return new BooleanPreference(value); }
    public static BooleanPreference parse(String raw) {
        Objects.requireNonNull(raw, "raw must not be null");
        return new BooleanPreference(Boolean.parseBoolean(raw));
    }
}
```

No `DurationPreference` needed. Time-based preferences use `IntPreference`
representing minutes, following the convention established by
`SlaPreferenceKeys.ESCALATION_HOURS` (which uses `IntPreference` for time).
The key name (`expiresInMinutes`) encodes the unit.

## Module Structure

```
domain/
  DevtownActionType.java           — typed string constants (action types + GENERAL_OVERSIGHT group)
  preferences/BooleanPreference.java — new SingleValuePreference for boolean keys
  preferences/RiskPreferenceKeys.java — PreferenceKey constants per action type

review/
  DevtownActionRiskClassifier.java — implements ActionRiskClassifier
                                     classification logic per action type

app/
  DevtownRiskClassifierProducer.java — @ApplicationScoped @RiskClassifier CDI adapter
```

### domain/ — DevtownActionType

Typed constant class following `ReviewDomain`/`AgentQualification` pattern.
Each constant is a `public static final String`. Includes the
`GENERAL_OVERSIGHT` candidateGroup constant.

### domain/ — RiskPreferenceKeys

Preference key constants shared across all action types. Contains:
- `ENABLED` — `PreferenceKey<BooleanPreference>` (namespace `casehubio.devtown.risk`, name `enabled`)
- Per-action-type threshold keys (e.g., `MERGE_APPROVED_REVIEW_COUNT`, `SECURITY_SEVERITY_THRESHOLD`)
- Per-action-type `expiresInMinutes` keys with defaults based on reversibility

### review/ — DevtownActionRiskClassifier

Implements `ActionRiskClassifier`. Constructor takes no CDI dependencies —
pure logic. The `classify()` method:

1. Receives `PlannedAction` plus a `Preferences` bag (resolved by the CDI adapter)
2. Checks `ENABLED` — if false, returns `Autonomous`
3. Dispatches on `actionType` to a per-action-type method
4. Each method reads its own threshold from `Preferences` and applies category rules
5. Unknown action type → fail-safe `GateRequired`

No `RiskConfig` record. Each per-action-type method reads directly from
`Preferences` via its specific `PreferenceKey`, avoiding the type mismatch
between `IntPreference` and `StringPreference` thresholds.

```java
public RiskDecision classify(PlannedAction action, Preferences prefs) {
    BooleanPreference enabled = prefs.getOrDefault(RiskPreferenceKeys.ENABLED);
    if (!enabled.value()) {
        return new RiskDecision.Autonomous();
    }
    return switch (action.actionType()) {
        case DevtownActionType.PR_MERGE_EXECUTE -> classifyMergeExecute(action, prefs);
        case DevtownActionType.SECURITY_ESCALATION -> classifySecurityEscalation(action, prefs);
        // ... per action type
        default -> failSafe(action);
    };
}
```

### app/ — DevtownRiskClassifierProducer

`@ApplicationScoped @RiskClassifier` bean. Injects `PreferenceProvider`.
On each `classify()` call:
1. Resolves `SettingsScope.of("casehubio", "devtown", "risk", action.actionType())`
2. Gets `Preferences` bag
3. Delegates to `DevtownActionRiskClassifier.classify(action, prefs)`

Implements `ActionRiskClassifier` (not `ReactiveActionRiskClassifier`) —
synchronous classification is sufficient. The engine wraps it in a reactive
chain automatically via `ChainedReactiveActionRiskClassifier`.

## Testing

### Unit tests (domain + review, no CDI)

- `BooleanPreferenceTest` — of/parse/value roundtrip (~3 tests)
- `DevtownActionTypeTest` — constants exist and match expected strings (~9 assertions)
- `RiskPreferenceKeysTest` — all keys have correct namespace, name, defaults (~10 tests)
- `DevtownActionRiskClassifierTest` — per action type:
  - Always-gate: returns `GateRequired` with correct fields
  - Minimum-requirement: below minimum → `GateRequired`, at minimum → `Autonomous`
  - Risk-threshold: below threshold → `Autonomous`, at threshold → `GateRequired`
  - Security escalation: `IncidentSeverity` ordinal comparison
  - Conditional: `pr-review-override` with `REJECTED` vs `APPROVED`/`PENDING`/absent
  - Disabled (`enabled=false`) → `Autonomous` for every action type
  - Unknown action type → fail-safe `GateRequired`
  - Null/missing context keys → handled gracefully per action type
  - ~30+ test cases total

### Integration tests (app, @QuarkusTest)

- `DevtownRiskClassifierWiringTest` — `@RiskClassifier` bean discovered
- Preference override test — set preference via YAML, verify threshold change
- ~4 test cases

## Gastown Feature Parity

This maps to Gastown's "hierarchical watchdog" (Witness/Deacon/Boot) but
with formal obligation tracking: each `GateRequired` creates a `WorkItem`
through the engine's oversight channel, with SLA and escalation —
capabilities that Gastown's watchdog cannot provide.

## Non-goals

- Custom `PlannedAction` creation by devtown workers — workers declare
  actions through the engine API; devtown only classifies them
- Reactive classifier — synchronous `ActionRiskClassifier` is sufficient
- Audit logging of classification decisions — future concern for ledger
  integration (classification events as ledger entries)
- Fixing `devtown.sla` namespace inconsistency — separate cleanup concern

## ARC42STORIES.MD Attribution

This is a Layer 5 extension — an SPI implementation within the existing
engine integration layer. `ARC42STORIES.MD §9.4 Layer 5` should be updated
when this ships to note the `ActionRiskClassifier` implementation.
