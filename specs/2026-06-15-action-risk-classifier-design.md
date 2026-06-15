# ActionRiskClassifier Oversight Gate — Design Spec

**Issue:** devtown#56
**Foundation:** engine#402 (shipped)
**Date:** 2026-06-15

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
`@RiskClassifier`-qualified beans and picks the most restrictive verdict.
If no classifiers are registered, actions proceed autonomously.

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

## Classification Rules

### Always-gate actions

These always return `GateRequired` unless explicitly disabled via preference:

| Action | Reason | Reversible |
|--------|--------|------------|
| `pr-force-merge` | Bypasses safety checks — requires explicit human approval | false |
| `contributor-access-change` | Access changes are consequential regardless of scope | false |

### Threshold-gated actions

These compare a metric from `PlannedAction.context()` against a configurable
threshold. Below threshold → `Autonomous`. At or above → `GateRequired`.

| Action | Context Key | Threshold Default | Reversible | Rationale |
|--------|------------|-------------------|------------|-----------|
| `pr-merge-execute` | `approvalCount` | 1 (require at least 1 approval) | false | Standard merge requires approval |
| `security-escalation` | `severity` | `"HIGH"` (gate HIGH and CRITICAL) | true | Low/medium findings can auto-report |
| `issue-close-invalid` | `commentCount` | 5 (gate high-engagement issues) | true | Low engagement can auto-close |
| `dependency-removal` | `transitiveUsageCount` | 3 (gate widely-used deps) | false | Narrow-use deps can auto-remove |
| `production-deploy` | `modulesAffected` | 3 (gate multi-module deploys) | false | Single-module deploys can auto-proceed |

### Override-specific

| Action | Rule | Reversible |
|--------|------|------------|
| `pr-review-override` | Always `GateRequired` when overriding a rejection; `Autonomous` when overriding an approval hold | true |

The `pr-review-override` rule inspects `context.get("originalVerdict")`:
- `"REJECTED"` → `GateRequired` (overriding a rejection is consequential)
- `"APPROVED"`, `"PENDING"`, or absent → `Autonomous` (no reviewer decision being overturned)

## PreferenceProvider Integration

Each action type maps to a preference scope path:

```
devtown/risk/<action-type>/enabled    → boolean (default: true)
devtown/risk/<action-type>/threshold  → type-specific (int, string)
```

Examples:
- `devtown/risk/production-deploy/enabled` = `true`
- `devtown/risk/production-deploy/threshold` = `3`
- `devtown/risk/security-escalation/threshold` = `"HIGH"`

Preference keys defined in `DevtownRiskPreferenceKeys` (domain module).
Defaults hardcoded as constants. `PreferenceProvider` lookup uses scope
from `PlannedAction.context().get("scope")` if present, falling back to
`Path.root()`.

When `enabled=false` for an action type, the classifier returns `Autonomous`
regardless of context — this is an escape hatch for trusted environments.

## GateRequired Details

| Field | Derivation |
|-------|-----------|
| `reason` | Action-type-specific message including the triggering metric |
| `reversible` | From action type definition (see tables above) |
| `candidateGroups` | `pr-merge-execute`, `pr-force-merge`, `pr-review-override` → `["human-decision:pr-approval"]`; `security-escalation`, `production-deploy` → `["human-oversight:routing-review"]`; others → `null` (any qualified human) |
| `expiresIn` | Reversible: 4h default. Irreversible: 24h default. Configurable via `devtown/risk/<type>/expiresIn` |
| `scope` | The action type string |

## Module Structure

```
domain/
  DevtownActionType.java         — typed string constants
  DevtownRiskPreferenceKeys.java — preference key constants + defaults

review/
  DevtownActionRiskClassifier.java — implements ActionRiskClassifier
                                     classification logic per action type

app/
  DevtownRiskClassifierProducer.java — @ApplicationScoped @RiskClassifier
                                        CDI adapter wiring PreferenceProvider
```

### domain/ — DevtownActionType

Typed constant class following `ReviewDomain`/`AgentQualification` pattern.
Each constant is a `public static final String`. No enum — action types
are open for extension by downstream consumers.

### domain/ — DevtownRiskPreferenceKeys

Preference key constants for each action type. Follows
`TrustGatePreferenceKeys` pattern:
- `PREFIX = "devtown.risk."`
- Per action type: `enabled` key, `threshold` key, `expiresIn` key
- Default values as constants

### review/ — DevtownActionRiskClassifier

Implements `ActionRiskClassifier`. Constructor takes no CDI dependencies —
pure logic. Methods:

```java
public RiskDecision classify(PlannedAction action) {
    // dispatch on action.actionType()
}

// Per action type:
RiskDecision classifyMergeExecute(PlannedAction action, RiskConfig config)
RiskDecision classifySecurityEscalation(PlannedAction action, RiskConfig config)
// ... etc
```

`RiskConfig` is a simple record holding the resolved preferences (enabled,
threshold, expiresIn) for one action type. Built by the CDI adapter from
`PreferenceProvider`.

### app/ — DevtownRiskClassifierProducer

`@ApplicationScoped` bean annotated `@RiskClassifier`. Injects
`PreferenceProvider`, resolves preferences per action type, constructs
`RiskConfig`, delegates to `DevtownActionRiskClassifier`.

## Testing

### Unit tests (domain + review, no CDI)

- `DevtownActionTypeTest` — constants exist and match expected strings (~8 assertions)
- `DevtownActionRiskClassifierTest` — per action type:
  - Always-gate actions return `GateRequired` with correct fields
  - Threshold-gated actions: below threshold → `Autonomous`, at/above → `GateRequired`
  - Disabled action → `Autonomous`
  - Unknown action type → `Autonomous` (fallback)
  - `pr-review-override` with `REJECTED` vs non-rejected verdict
  - ~24 test cases total

### Integration tests (app, @QuarkusTest)

- `DevtownRiskClassifierWiringTest` — `@RiskClassifier` bean discovered, classifier invoked via engine
- Preference override test — set preference, verify threshold change takes effect
- ~4 test cases

## Gastown Feature Parity

This maps to Gastown's "hierarchical watchdog" (Witness/Deacon/Boot) but with
formal obligation tracking: each `GateRequired` creates a `WorkItem` through
the engine's oversight channel, with SLA and escalation — capabilities that
Gastown's watchdog cannot provide.

## Non-goals

- Custom `PlannedAction` creation by devtown workers — workers declare actions
  through the engine API; devtown only classifies them
- Reactive classifier — synchronous `ActionRiskClassifier` is sufficient; the
  engine wraps it in a reactive chain automatically
- Audit logging of classification decisions — future concern for Layer 4 ledger
  integration (classification events as ledger entries)
