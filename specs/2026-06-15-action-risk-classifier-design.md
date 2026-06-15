# ActionRiskClassifier Oversight Gate — Design Spec

**Issue:** devtown#56
**Foundation:** engine#402 (shipped)
**Layer:** 5 extension (casehub-engine SPI implementation)
**Date:** 2026-06-15 (revised after round 3 review)

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

| Constant | Class | Value | Description |
|----------|-------|-------|-------------|
| `PR_MERGE_EXECUTE` | `DevtownActionType` | `pr-merge-execute` | Merge a PR to the target branch |
| `PR_FORCE_MERGE` | `DevtownActionType` | `pr-force-merge` | Merge overriding failed checks |
| `PR_REVIEW_OVERRIDE` | `DevtownActionType` | `pr-review-override` | Override a reviewer's decision |
| `SECURITY_ESCALATION` | `DevtownActionType` | `security-escalation` | Block PR and alert on security finding |
| `ISSUE_CLOSE_INVALID` | `DevtownActionType` | `issue-close-invalid` | Close issue as invalid |
| `DEPENDENCY_REMOVAL` | `DevtownActionType` | `dependency-removal` | Remove a flagged dependency |
| `CONTRIBUTOR_ACCESS_CHANGE` | `DevtownActionType` | `contributor-access-change` | Add/remove contributor access |
| `PRODUCTION_DEPLOY` | `DevtownActionType` | `production-deploy` | Deploy to production |

### Candidate Group Constants (separate from action types)

| Constant | Class | Value | Used by |
|----------|-------|-------|---------|
| `PR_APPROVAL` | `HumanDecision` | `human-decision:pr-approval` | pr-merge-execute, pr-force-merge, pr-review-override |
| `ROUTING_REVIEW` | `HumanOversight` | `human-oversight:routing-review` | security-escalation, production-deploy |
| `GENERAL` | `HumanOversight` | `human-oversight:general` | issue-close-invalid, dependency-removal, contributor-access-change, unknown types |

`HumanOversight.GENERAL` is a new constant in the existing `HumanOversight`
class. It follows the `human-oversight:` prefix convention alongside the
existing `ROUTING_REVIEW`. It ensures WorkItems always have group scoping —
`null` candidateGroups would allow any user to claim the item.

## Classification Categories

Four categories, mapped to `ActionGatePolicy`:

### 1. Always-gate (ALWAYS)

Always return `GateRequired` unless explicitly disabled via preference.

| Action | Reason | Reversible | candidateGroups |
|--------|--------|------------|-----------------|
| `pr-force-merge` | Bypasses safety checks — requires explicit human approval | false | `HumanDecision.PR_APPROVAL` |
| `contributor-access-change` | Access changes are consequential regardless of scope | false | `HumanOversight.GENERAL` |

### 2. Minimum-requirement (THRESHOLD, inverted comparison)

Compare a safety metric from `PlannedAction.context()` against a required
minimum. **Below minimum → `GateRequired`. At or above → `Autonomous`.**

This is the inverse of risk-threshold actions: the metric measures safety
(more = safer), not risk (more = riskier).

| Action | Context Key | Minimum Default | Reversible | candidateGroups | Rationale |
|--------|------------|-----------------|------------|-----------------|-----------|
| `pr-merge-execute` | `approvedReviewCount` | 1 | false | `HumanDecision.PR_APPROVAL` | Zero approvals must never proceed |

### 3. Risk-threshold (THRESHOLD, standard comparison)

Compare a risk metric from `PlannedAction.context()` against a configurable
threshold. **At or above threshold → `GateRequired`. Below → `Autonomous`.**

| Action | Context Key | Threshold Default | Reversible | candidateGroups | Rationale |
|--------|------------|-------------------|------------|-----------------|-----------|
| `security-escalation` | `severity` | `HIGH` (via `IncidentSeverity` ordinal) | true | `HumanOversight.ROUTING_REVIEW` | LOW/MEDIUM can auto-report |
| `issue-close-invalid` | `commentCount` | 5 | true | `HumanOversight.GENERAL` | Low engagement can auto-close |
| `dependency-removal` | `transitiveUsageCount` | 3 | false | `HumanOversight.GENERAL` | Narrow-use deps can auto-remove |
| `production-deploy` | `modulesAffected` | 3 | false | `HumanOversight.ROUTING_REVIEW` | Single-module deploys can auto-proceed |

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
| `pr-review-override` | `GateRequired` when `context.get("originalVerdict")` is `"REJECTED"` (overriding a rejection is consequential). `Autonomous` when `"APPROVED"`, `"PENDING"`, or absent (no reviewer decision being overturned) | true | `HumanDecision.PR_APPROVAL` |

### 5. Unknown action type (fail-safe)

Unknown action types return `GateRequired("Unknown action type — manual
review required", true, List.of(HumanOversight.GENERAL),
Duration.ofHours(24), actionType)`. Consistent with the engine's fail-safe
principle — unknown actions must not proceed autonomously.

### Null/Missing Context Key Rule

**Missing or unparseable context metrics default to fail-safe
(`GateRequired`), except for conditional actions where absence has defined
non-risk semantics.**

Specifically:
- **Minimum-requirement** (`approvedReviewCount` missing): no evidence of
  approval → `GateRequired`. Zero is the safe assumption.
- **Risk-threshold** (`severity`, `commentCount`, etc. missing): unknown
  risk is not zero risk → `GateRequired`.
- **Risk-threshold** (value present but unparseable — e.g., `severity`
  string that does not map to `IncidentSeverity`): malformed input is not
  safe input → `GateRequired`.
- **Conditional** (`originalVerdict` absent): already specified — absent
  means no verdict to override → `Autonomous`. This is correct because
  you cannot override a verdict that does not exist.

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

### Preference keys

**Shared keys** (all action types):

| Key constant | Name | Type | Default | Notes |
|-------------|------|------|---------|-------|
| `ENABLED` | `enabled` | `BooleanPreference` | `true` | Escape hatch — `false` returns `Autonomous` regardless |
| `EXPIRES_IN_MINUTES_REVERSIBLE` | `expiresInMinutes` | `IntPreference` | `240` (4h) | Used by classifier methods for reversible actions |
| `EXPIRES_IN_MINUTES_IRREVERSIBLE` | `expiresInMinutes` | `IntPreference` | `1440` (24h) | Used by classifier methods for irreversible actions |

Both expiration keys share the same `name` (`expiresInMinutes`) so they
resolve to the same YAML property within a scope. If a user configures
a custom value at a specific action-type scope, it overrides both. The
distinct defaults serve only the case where no YAML value is present —
each classifier method picks the key matching its action's reversibility
(a compile-time constant per action type).

**Per-action-type threshold keys:**

| Key constant | Name | Type | Default |
|-------------|------|------|---------|
| `MERGE_MIN_APPROVED_REVIEWS` | `threshold` | `IntPreference` | `1` |
| `SECURITY_SEVERITY_THRESHOLD` | `threshold` | `StringPreference` | `"HIGH"` |
| `ISSUE_CLOSE_COMMENT_THRESHOLD` | `threshold` | `IntPreference` | `5` |
| `DEPENDENCY_USAGE_THRESHOLD` | `threshold` | `IntPreference` | `3` |
| `DEPLOY_MODULE_THRESHOLD` | `threshold` | `IntPreference` | `3` |

All threshold keys share `name` = `threshold` but each classifier method
reads its own typed key. The `SettingsScope` per action type means each
scope bag has at most one `threshold` value — no collision.

### Namespace convention

All keys use namespace `casehubio.devtown.risk` — consistent with the
majority pattern (`casehubio.devtown.trust-gate`,
`casehubio.devtown.trust-routing`). The `devtown.sla` prefix is a legacy
inconsistency — not addressed in this issue.

### New preference types required

**`BooleanPreference`** — new record in `domain/preferences/` with strict
parsing (consistent with `IntPreference`/`DoublePreference` which throw on
invalid input):

```java
public record BooleanPreference(boolean value) implements SingleValuePreference {
    public static BooleanPreference of(boolean value) {
        return new BooleanPreference(value);
    }
    public static BooleanPreference parse(String raw) {
        Objects.requireNonNull(raw, "raw must not be null");
        return switch (raw.strip().toLowerCase()) {
            case "true" -> new BooleanPreference(true);
            case "false" -> new BooleanPreference(false);
            default -> throw new IllegalArgumentException(
                "Invalid boolean preference: '" + raw
                + "' — expected 'true' or 'false'");
        };
    }
}
```

`Boolean.parseBoolean("ture")` silently returns `false`. In a system where
`enabled=false` disables an oversight gate, a YAML typo silently turning
off security review is a serious failure mode. Strict parsing fails fast
at preference resolution.

No `DurationPreference` needed. Time-based preferences use `IntPreference`
representing minutes, following the convention established by
`SlaPreferenceKeys.ESCALATION_HOURS` (which uses `IntPreference` for time).
The key name (`expiresInMinutes`) encodes the unit.

## Module Structure

```
domain/
  DevtownActionType.java             — typed string constants (action types only)
  HumanOversight.java                — + GENERAL constant (new, alongside existing ROUTING_REVIEW)
  preferences/BooleanPreference.java — new SingleValuePreference for boolean keys
  preferences/RiskPreferenceKeys.java — PreferenceKey constants per action type

review/
  DevtownActionRiskClassifier.java   — classification logic per action type
                                       (does NOT implement ActionRiskClassifier —
                                       own method signature takes Preferences)

app/
  DevtownRiskClassifierProducer.java — @ApplicationScoped @RiskClassifier
                                       implements ActionRiskClassifier
                                       CDI adapter: resolves Preferences, delegates
```

### domain/ — DevtownActionType

Typed constant class following `ReviewDomain`/`AgentQualification` pattern.
Each constant is a `public static final String`. Contains only action type
constants — no candidate group constants (those belong in `HumanDecision`
and `HumanOversight`).

### domain/ — HumanOversight (existing, extended)

Add `GENERAL = "human-oversight:general"` alongside the existing
`ROUTING_REVIEW = "human-oversight:routing-review"`. Follows the
`human-oversight:` prefix convention.

### domain/ — RiskPreferenceKeys

Preference key constants. Contains:
- `ENABLED` — `PreferenceKey<BooleanPreference>` (namespace `casehubio.devtown.risk`, name `enabled`)
- `EXPIRES_IN_MINUTES_REVERSIBLE` / `EXPIRES_IN_MINUTES_IRREVERSIBLE` — both name `expiresInMinutes`, different defaults
- Per-action-type threshold keys (see table above)

### review/ — DevtownActionRiskClassifier

Pure logic class. Does **not** implement `ActionRiskClassifier` — its
`classify()` method takes an extra `Preferences` parameter that the SPI
signature does not have. The CDI adapter in app/ bridges the gap.

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

Each per-action-type method:
1. Reads its own threshold from `Preferences` via its specific `PreferenceKey`
2. Extracts the relevant metric from `PlannedAction.context()`
3. If the context key is missing or the value is unparseable → `GateRequired` (fail-safe)
4. Compares and returns the appropriate `RiskDecision`

**Context value extraction:** `PlannedAction.context()` is
`Map<String, Object>`. JSON deserialization typically produces `Long` for
integer values, not `Integer`. Integer metrics must be extracted via
`Number.intValue()` (not `(Integer)` cast). String metrics are extracted
via `(String)` cast or `Objects.toString()`.

### app/ — DevtownRiskClassifierProducer

`@ApplicationScoped @RiskClassifier` bean that **implements
`ActionRiskClassifier`** (the SPI). Injects `PreferenceProvider`. On each
`classify(PlannedAction action)` call:
1. Resolves `SettingsScope.of("casehubio", "devtown", "risk", action.actionType())`
2. Gets `Preferences` bag
3. Delegates to `DevtownActionRiskClassifier.classify(action, prefs)`

Synchronous classification — the engine wraps it reactively via
`ChainedReactiveActionRiskClassifier`.

## Testing

### Unit tests (domain + review, no CDI)

- `BooleanPreferenceTest` — of/parse/value roundtrip, strict parsing
  rejects typos like "ture" (~5 tests)
- `DevtownActionTypeTest` — constants exist and match expected strings
  (~8 assertions)
- `HumanOversightTest` — update existing test (4 assertions for
  ROUTING_REVIEW) to add 4 matching assertions for GENERAL:
  constantNonBlank, valueMatchesSpec (`human-oversight:general`),
  prefixedCorrectly (`human-oversight:`), noOverlapWithHumanDecision
- `RiskPreferenceKeysTest` — all keys have correct namespace, name,
  defaults; both expiration keys share name but differ in defaults (~12 tests)
- `DevtownActionRiskClassifierTest` — per action type:
  - Always-gate: returns `GateRequired` with correct fields (reason,
    reversible, candidateGroups, scope)
  - Minimum-requirement: below minimum → `GateRequired`, at minimum →
    `Autonomous`, above → `Autonomous`
  - Risk-threshold: below threshold → `Autonomous`, at threshold →
    `GateRequired`, above → `GateRequired`
  - Security escalation: `IncidentSeverity` ordinal comparison (LOW →
    auto, MEDIUM → auto, HIGH → gate, CRITICAL → gate)
  - Conditional: `pr-review-override` with `REJECTED` vs
    `APPROVED`/`PENDING`/absent
  - Disabled (`enabled=false`) → `Autonomous` for every action type
  - Unknown action type → fail-safe `GateRequired`
  - Missing context key → fail-safe `GateRequired` (for min-req and
    risk-threshold categories)
  - Unparseable context value (e.g., invalid severity string) → fail-safe
    `GateRequired`
  - ~30+ test cases total

### Integration tests (app, @QuarkusTest)

- `DevtownRiskClassifierWiringTest` — `@RiskClassifier` bean discovered
  by `ChainedReactiveActionRiskClassifier`
- Preference override test — set preference via YAML, verify threshold
  change takes effect
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
