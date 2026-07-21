# CasePlanModel Browser — Design Spec

**Issue:** devtown#119
**Date:** 2026-07-21
**Status:** Draft
**Work slot:** 10 (branch `issue-119-caseplanmodel-browser`)

## Problem

devtown#119 asks for a "CasePlanModel browser — view adaptive case definitions and binding conditions." The blocks-ui case-explorer (blocks-ui#87) already delivers the generic entity browsing infrastructure — tree mode, entity detail, command bar, convenience components. The question is: what does devtown need to do, and what gaps exist between the engine's CasePlanModel runtime state and what the case-explorer can render?

This spec answers that through a comprehensive audit.

## 1. What Case-Explorer Already Provides

The composable case-explorer (spec: `blocks-ui/docs/specs/2026-07-20-composable-case-explorer-design.md`) delivers:

| Component | What it renders |
|-----------|----------------|
| `<case-explorer>` | Full composed experience — entity type tabs, list/tree mode, NavigationController, breadcrumbs |
| `<entity-list>` | Paginated entity list with filters, driven by `EntityTypeRegistration` |
| `<entity-tree>` | Collapsible hierarchy with status badges, lazy loading, M-of-N groups |
| `<entity-detail>` | Three-tier polymorphic detail: sub-type → entity-type → default state table |
| `<entity-command-bar>` | Dynamic command buttons from `availableCommands`, confirmation, POST execution |
| `<case-definition-browser>` | Convenience: browse CaseDefinitions (namespace, name, version, worker count, milestones) |
| `<case-instance-list>` | Convenience: list case instances (name, status, started, active workers, SLA) |
| `<case-detail-panel>` | Convenience: case detail with tabs (Overview, Plan, Workers, Sub-cases, Timeline, Channels) |
| `<worker-list>` | Convenience: list workers (name, type, status, parent case, started, progress) |
| `<worker-detail-panel>` | Convenience: worker detail with state, commands, pluggable renderer |

The entity tree example from the spec maps directly to CasePlanModel runtime state:

```
Case #7 (PR review)                    [RUNNING]
├── Worker: code-analysis              [COMPLETED]
├── Worker: security-review            [RUNNING]
├── SubCase Group: per-repo-checks     [2/5 required, 1 completed]
│   ├── SubCase: repo-A                [COMPLETED]
│   │   ├── Worker: lint               [COMPLETED]
│   │   └── Worker: test               [COMPLETED]
│   ├── SubCase: repo-B                [RUNNING]
│   │   ├── Worker: lint               [COMPLETED]
│   │   └── Worker: test               [RUNNING]
│   └── SubCase: repo-C                [PENDING]
└── SubCase: rollback-plan             [PENDING]
    └── Worker: rollback-analyzer      [PENDING]
```

Mapping: tree root = CasePlanModel instance, worker nodes = PlanItems, sub-case nodes = SubCaseTarget activations, groups = Stages with M-of-N completion criteria, status badges = TaskStatus/StageStatus.

## 2. CasePlanModel Concept Audit

### 2.1 Engine Runtime Model

**CasePlanModel** (`io.casehub.blackboard.plan.CasePlanModel`) — the per-case-instance working memory. Key state:

| Method | Returns | What it exposes |
|--------|---------|----------------|
| `getAllPlanItems()` | `List<PlanItem>` | All binding activations with status |
| `getAgenda()` | `List<PlanItem>` | PENDING items sorted by priority |
| `getAllStages()` | `List<Stage>` | All stage containers with lifecycle |
| `getSubCases()` | `List<SubCase>` | Registered sub-cases |
| `getMilestoneStatus(name)` | `Optional<MilestoneLifecycleStatus>` | Per-milestone: PENDING/ACTIVE/COMPLETED/FAILED/CANCELLED |
| `isMilestoneAchieved(name)` | `boolean` | Whether milestone was achieved |
| `getFocus()` / `getFocusRationale()` | `Optional<String>` | Current planning focus |
| `getResourceBudget()` | `Map<String, Object>` | Resource allocations |
| `get(key, type)` | `Optional<T>` | Arbitrary blackboard context |

**PlanItem** (`io.casehub.blackboard.plan.PlanItem`) — a fired binding activation:

| Field | Type | Notes |
|-------|------|-------|
| `planItemId` | `String` | Unique ID |
| `bindingName` | `String` | Which binding fired |
| `executor` | `ExecutorRef` | Who is executing |
| `priority` | `int` | Dispatch priority |
| `status` | `TaskStatus` | PENDING/RUNNING/DELEGATED/SUSPENDED/COMPLETED/FAULTED/REJECTED/OBSOLETE/CANCELLED |
| `createdAt` | `Instant` | When the binding fired |
| `parentStageId` | `String` | Containing stage (nullable) |
| `target` | `BindingTarget` | Sealed: CapabilityTarget, SubCaseTarget, HumanTaskTarget, ExtensionTarget |
| `description` | `String` | Human-readable description |

**Stage** (`io.casehub.blackboard.stage.Stage`) — grouping container:

| Field | Type | Notes |
|-------|------|-------|
| `stageId` | `String` | Unique ID |
| `name` | `String` | Stage name |
| `status` | `StageStatus` | PENDING/ACTIVE/SUSPENDED/COMPLETED/TERMINATED/FAULTED |
| `createdAt` / `activatedAt` / `completedAt` | `Instant` | Lifecycle timestamps |
| `parentStageId` | `String` | Nesting (nullable) |
| `entryCondition` / `exitCondition` | `ExpressionEvaluator` | Guards |
| `containedPlanItemIds` | `Set<String>` | PlanItems in this stage |
| `containedMilestoneIds` | `Set<String>` | Milestones in this stage |
| `containedStageIds` | `Set<String>` | Nested stages |
| `containedBindingNames` | `Set<String>` | Bindings scoped to this stage |
| `requiredItemIds` | `Set<String>` | Items required for autocomplete |
| `autocomplete` / `manualActivation` / `repeatable` | `boolean` | Behaviour flags |
| `instanceIndex` | `int` | Repetition counter |

**Milestone** (`io.casehub.api.model.Milestone`) — named checkpoint:

| Field | Type | Notes |
|-------|------|-------|
| `name` | `String` | Milestone name |
| `entryCriteria` / `completionCriteria` | `ExpressionEvaluator` | Guards |
| `slaDuration` | `Duration` | Deadline (nullable) |
| `slaStartFrom` | `SlaStartFrom` | When SLA clock starts |
| `parentStageId` | `String` | Containing stage |
| `description` | `String` | Human-readable |

Lifecycle tracked by CasePlanModel via `MilestoneLifecycleStatus`: PENDING → ACTIVE → COMPLETED (or FAILED/CANCELLED).

**Goal** (`io.casehub.api.model.Goal`) — completion condition:

| Field | Type | Notes |
|-------|------|-------|
| `name` | `String` | Goal name |
| `condition` | `ExpressionEvaluator` | When this goal is satisfied |
| `kind` | `String` | GoalKind: success or failure |
| `description` | `String` | Human-readable |

**GoalExpression** — sealed: `AllOfGoalExpression`, `AnyOfGoalExpression`, `SingleGoalExpression`. Defines case completion as a boolean tree over goal names. Has `isSatisfiedBy(Set<String> reachedGoalNames)` and `goalNames()`.

**BindingTarget** — sealed: `CapabilityTarget` (dispatch to agent), `SubCaseTarget` (spawn child case), `HumanTaskTarget` (create work item), `ExtensionTarget`.

**ExpressionEvaluator** — interface with `type()`. Two implementations: `JQExpressionEvaluator` (has `.expression()` string — serializable) and `LambdaExpressionEvaluator` (Java predicate — not serializable).

### 2.2 Gap Table

| # | CasePlanModel Concept | What a user wants to see | Case-explorer coverage | Gap |
|---|---|---|---|---|
| 1 | PlanItem (binding activation) | ID, binding name, status, executor, priority, created time, description | `EntityInstance.state` — all primitives | **No gap** |
| 2 | PlanItem.target (BindingTarget) | Target type + type-specific fields (capability name, sub-case definition, human task title) | Generic `state` bag loses sealed type discrimination | **Serialization gap** — EntityStateContributor must flatten to `{ targetType: "capability", capabilityName: "..." }`. Detail renderer dispatches on `targetType`. No engine change needed. |
| 3 | Stage hierarchy | Name, status, timestamps, parent/child relationships, contained items, autocomplete flag | `EntityTreeNode` with `children`/`childrenEndpoint` | **Serialization gap** — `containedPlanItemIds` are ID sets, not objects. EntityStateContributor resolves IDs server-side. No engine change needed. |
| 4 | Stage entry/exit conditions | What expression guards this stage | `ExpressionEvaluator` — JQ strings serializable, lambdas not | **Engine gap** — see §3.1 |
| 5 | Milestone lifecycle | Name, status (PENDING/ACTIVE/COMPLETED/FAILED/CANCELLED), entry/completion criteria | Status via `getMilestoneStatus()` — serializable. Criteria are ExpressionEvaluator — same opacity issue | **Engine gap** — see §3.1 |
| 6 | Milestone SLA runtime | Deadline timestamp, time remaining, breached flag | `Milestone` stores `slaDuration` and `slaStartFrom` definition. `MilestoneSLATimeoutJob` computes deadline. CasePlanModel tracks lifecycle but **not** deadline/breach state | **Engine gap** — see §3.2 |
| 7 | Goal evaluation state | Name, kind, description, whether reached | `Goal` has name/kind/description. `GoalReachedEvent` fires when reached, but CasePlanModel has **no** `isGoalReached(name)` getter or reached-goals set | **Engine gap** — see §3.3 |
| 8 | GoalExpression (completion condition) | Tree structure: allOf/anyOf over goal names, which are satisfied | `GoalExpression.goalNames()` and `isSatisfiedBy()` exist. Serialization is straightforward (recursive allOf/anyOf/single tree) | **No gap** — serialize as `{ type: "allOf", children: [...] }` |
| 9 | Binding (design-time rules) | Name, trigger type, guard expression, target type, conflict strategy | `CaseDefinition.getBindings()` — all fields serializable except guard (ExpressionEvaluator opacity) | **Engine gap** — see §3.1 |
| 10 | Agenda (pending items by priority) | Ordered list of what's about to fire | `getAgenda()` exists, returns `List<PlanItem>` | **No gap** — serialize as a sub-section of case detail |
| 11 | Sub-cases | Registered sub-cases with status | `getSubCases()` exists | **No gap** |
| 12 | Focus / resource budget | Current planning focus and rationale, budget allocations | `getFocus()`, `getFocusRationale()`, `getResourceBudget()` — all serializable | **No gap** |
| 13 | Blackboard context (key-value) | Runtime context driving binding evaluation | `get(key, type)` — arbitrary typed entries | **Partial** — no `getAllKeys()` or snapshot method on CasePlanModel to enumerate what's in the blackboard. EntityStateContributor would need to know which keys to export. |

### 2.3 Summary

**Three engine API gaps** that need fixes before the UI can display the data:
1. ExpressionEvaluator opacity (§3.1)
2. Milestone SLA runtime state (§3.2)
3. Goal reached state (§3.3)

**Two serialization concerns** solved entirely in the EntityStateContributor — no engine changes:
4. BindingTarget sealed type flattening
5. Stage containment cross-references

**One partial gap** (blackboard context enumeration) — addressable by convention.

## 3. Engine API Changes Required

### 3.1 ExpressionEvaluator display string

**Problem:** Binding guards, milestone criteria, goal conditions, and stage entry/exit conditions are `ExpressionEvaluator`. `JQExpressionEvaluator` has `.expression()` (serializable). `LambdaExpressionEvaluator` is a Java predicate — the UI gets a class name, not a meaningful description.

**Proposed fix:** Add a default method to `ExpressionEvaluator`:

```java
public interface ExpressionEvaluator {
    String type();

    default String displayExpression() {
        return type();
    }
}
```

`JQExpressionEvaluator` overrides to return `expression()`. `LambdaExpressionEvaluator` returns a configurable label (set at construction) or falls back to `"lambda"`. CaseDefinition YAML-declared expressions always use JQ and are always displayable; lambdas are only used in programmatic Java test definitions.

**Engine issue to file:** casehubio/engine — `ExpressionEvaluator.displayExpression()` default method.

**Scope:** XS — one default method, two overrides, no behavioural change.

### 3.2 Milestone SLA runtime state

**Problem:** `Milestone` defines SLA (`slaDuration`, `slaStartFrom`). `MilestoneSLATimeoutJob` computes the actual deadline and fires on breach. But CasePlanModel only tracks `MilestoneLifecycleStatus` (PENDING/ACTIVE/COMPLETED) — there's no queryable deadline timestamp or breach flag.

**Proposed fix:** Enrich the milestone tracking in `DefaultCasePlanModel` to record activation timestamp alongside status. The EntityStateContributor computes deadline from `activatedAt + slaDuration` and breach from `now > deadline`.

Alternatively, add a `MilestoneRuntimeState` record:

```java
public record MilestoneRuntimeState(
    MilestoneLifecycleStatus status,
    Instant activatedAt,     // null if PENDING
    Instant deadline,        // null if no SLA or PENDING
    boolean breached         // true if past deadline and not completed
) {}
```

Replace `getMilestoneStatus()` with `getMilestoneState()` returning `Optional<MilestoneRuntimeState>`.

**Engine issue to file:** casehubio/engine — `MilestoneRuntimeState` replacing status-only tracking.

**Scope:** S — new record, modify `DefaultCasePlanModel` milestone tracking, update callers of `getMilestoneStatus()`.

### 3.3 Goal reached state

**Problem:** `GoalReachedEvent` fires when a goal condition evaluates to true, but CasePlanModel has no getter for which goals have been reached. The information exists transiently in the event flow and in `GoalExpression.isSatisfiedBy()` — but there's no persistent `Set<String> reachedGoals` on the plan model.

**Proposed fix:** Add reached-goal tracking to `CasePlanModel`:

```java
void markGoalReached(String goalName);
Set<String> getReachedGoals();
boolean isGoalReached(String goalName);
```

`GoalReachedEventHandler` calls `markGoalReached()` when it fires the event. The EntityStateContributor reads `getReachedGoals()` and cross-references with the CaseDefinition's goals to produce a goal status table.

**Engine issue to file:** casehubio/engine — goal reached state tracking on CasePlanModel.

**Scope:** S — three methods on interface, implementation in `DefaultCasePlanModel` (a `ConcurrentHashMap.newKeySet()`), one call site in `GoalReachedEventHandler`.

## 4. Devtown EntityStateContributor Design

### 4.1 Entity types to register

devtown registers these entity types for the case-explorer:

| Entity type | List endpoint | Detail endpoint | Tree endpoint | Notes |
|-------------|--------------|----------------|---------------|-------|
| `case-definition` | `/api/case-definitions` | `/api/case-definitions/{id}` | — | Static CaseDefinition browsing |
| `case-instance` | `/api/cases` | `/api/cases/{id}` | `/api/cases/{id}/tree` | Runtime CasePlanModel instance |
| `worker` | `/api/workers` | `/api/workers/{id}` | — | Aggregated across sub-types |
| `worker:agent` | — | — | — | Sub-type for agent workers |
| `worker:flow` | — | — | — | Sub-type for flow workers |
| `worker:human` | — | — | — | Sub-type for human work items |
| `milestone` | `/api/cases/{caseId}/milestones` | `/api/cases/{caseId}/milestones/{name}` | — | Per-case milestone list |
| `stage` | `/api/cases/{caseId}/stages` | `/api/cases/{caseId}/stages/{id}` | — | Per-case stage list |

### 4.2 Case instance EntityInstance

The `EntityStateContributor` for `case-instance` serializes CasePlanModel state into `EntityInstance`:

```typescript
// EntityInstance.state for a case instance
{
  definitionName: string;
  definitionVersion: string;
  focus: string | null;
  focusRationale: string | null;
  resourceBudget: Record<string, unknown>;

  // Goal status
  goals: Array<{
    name: string;
    kind: string;            // "success" or "failure"
    description: string;
    condition: string;       // displayExpression() from §3.1
    reached: boolean;        // from §3.3
  }>;
  goalExpression: GoalExpressionNode;  // { type: "allOf"|"anyOf"|"single", ... }
  completionSatisfied: boolean;

  // Agenda snapshot
  agenda: Array<{
    planItemId: string;
    bindingName: string;
    priority: number;
    targetType: string;
  }>;

  // Counts
  planItemCount: number;
  activePlanItemCount: number;
  stageCount: number;
  subCaseCount: number;
  milestoneCount: number;
}
```

### 4.3 Case definition EntityInstance

For `case-definition`, serialize the static CaseDefinition:

```typescript
{
  namespace: string;
  name: string;
  version: string;

  bindings: Array<{
    name: string;
    triggerType: string;         // "context-change" or "schedule"
    guard: string | null;        // displayExpression() — null if always-fire
    targetType: string;          // "capability" | "sub-case" | "human-task" | "extension"
    targetDetail: string;        // capability name, sub-case definition, etc.
    conflictStrategy: string;
    outcomePolicy: string;
  }>;

  milestones: Array<{
    name: string;
    description: string;
    entryCriteria: string;       // displayExpression()
    completionCriteria: string;  // displayExpression()
    slaDuration: string | null;  // ISO-8601 duration
  }>;

  goals: Array<{
    name: string;
    kind: string;
    description: string;
    condition: string;           // displayExpression()
  }>;

  goalExpression: GoalExpressionNode;

  workers: Array<{
    name: string;
    capabilityName: string;
  }>;

  capabilities: string[];
}
```

### 4.4 PlanItem EntityInstance

```typescript
{
  planItemId: string;
  bindingName: string;
  executorName: string;
  priority: number;
  status: string;          // TaskStatus enum name
  createdAt: string;       // ISO-8601
  description: string;
  parentStageId: string | null;

  // Flattened BindingTarget (§2.2 gap #2)
  targetType: string;      // "capability" | "sub-case" | "human-task" | "extension"
  targetCapability?: string;
  targetSubCaseDefinition?: string;
  targetHumanTaskTitle?: string;
}
```

### 4.5 Milestone EntityInstance

```typescript
{
  name: string;
  description: string;
  status: string;            // MilestoneLifecycleStatus enum name
  activatedAt: string | null;
  deadline: string | null;   // from §3.2 MilestoneRuntimeState
  breached: boolean;
  achieved: boolean;
  entryCriteria: string;     // displayExpression()
  completionCriteria: string;
  slaDuration: string | null;
  parentStageId: string | null;
}
```

### 4.6 Stage EntityInstance

```typescript
{
  stageId: string;
  name: string;
  status: string;            // StageStatus enum name
  createdAt: string;
  activatedAt: string | null;
  completedAt: string | null;
  parentStageId: string | null;
  entryCondition: string;    // displayExpression()
  exitCondition: string;     // displayExpression()
  autocomplete: boolean;
  manualActivation: boolean;
  repeatable: boolean;
  instanceIndex: number;

  // Resolved containment (§2.2 gap #3 — resolved server-side)
  containedPlanItems: Array<{ id: string; bindingName: string; status: string }>;
  containedMilestones: Array<{ name: string; status: string }>;
  containedStages: Array<{ id: string; name: string; status: string }>;
  requiredItems: Array<{ id: string; bindingName: string; status: string; completed: boolean }>;
  containedBindingNames: string[];

  // Completion progress
  requiredCompleted: number;
  requiredTotal: number;
}
```

### 4.7 Tree endpoint

`/api/cases/{id}/tree` returns the first two levels of the case hierarchy as `EntityTreeNode[]`:

```
Case root
├── Stage: parallel-checks        [ACTIVE]     (2/5 required, 1 completed)
│   ├── PlanItem: code-analysis   [COMPLETED]
│   ├── PlanItem: security-review [RUNNING]
│   └── SubCase: per-repo-A       [RUNNING]
├── Milestone: pr-approved        [PENDING]
├── Milestone: ci-passing         [ACTIVE]
└── Goal: security-verified       [not reached]
```

Deeper levels (e.g., sub-case internals) load lazily via `childrenEndpoint`.

Milestones and goals appear as tree nodes alongside stages and plan items — they are first-class visible entities in the case plan, not hidden metadata.

## 5. Devtown-Specific Detail Renderers

The case-explorer's three-tier detail resolution (sub-type → entity-type → default state table) handles most rendering. Devtown adds domain-specific renderers for:

### 5.1 PR Review case detail

When a case instance's definition is `devtown/pr-review`, render PR metadata:
- Repository, PR number, author, lines changed
- Capability progress pipeline: code-analysis → security → architecture → style → CI (status badges from PlanItem statuses)

Registered via `detailRendererMap` on the case instance entity type.

### 5.2 Coordinated change case detail

When definition is `devtown/coordinated-change`, render:
- Per-repo sub-case status grid
- Merge results table
- Rollback status (if triggered)

### 5.3 Binding condition viewer

For case-definition detail, render bindings as a structured rule table rather than raw JSON:

| Binding | Trigger | Guard | Target | Conflict |
|---------|---------|-------|--------|----------|
| security-review | context-change | `.fileTypes | contains("auth")` | capability: security-review | SKIP_IF_ACTIVE |
| merge | context-change | `.allChecksPass == true` | capability: merge-executor | — |

JQ expressions rendered in monospace. Lambda guards show `"(programmatic)"`.

## 6. Frontend Entity Type Registrations

```typescript
import { caseInstanceType, workerType } from '@casehubio/blocks-ui-case-explorer';

// Case definitions — static browsing
const caseDefinitionType: EntityTypeRegistration = {
  type: 'case-definition',
  label: 'Definitions',
  listEndpoint: '/api/case-definitions',
  detailEndpoint: (id) => `/api/case-definitions/${id}`,
  columnConfig: [
    { id: 'namespace', name: 'Namespace', type: 'TEXT' },
    { id: 'name', name: 'Name', type: 'TEXT' },
    { id: 'version', name: 'Version', type: 'TEXT' },
    { id: 'bindingCount', name: 'Bindings', type: 'NUMBER' },
    { id: 'goalCount', name: 'Goals', type: 'NUMBER' },
  ],
  detailRenderer: 'devtown-case-definition-detail',  // §5.3 binding viewer
};

// Case instances — runtime browsing
const devtownCaseType = {
  ...caseInstanceType({ listEndpoint: '/api/cases' }),
  detailRendererMap: {
    'devtown/pr-review': 'devtown-pr-review-detail',           // §5.1
    'devtown/coordinated-change': 'devtown-coordinated-detail', // §5.2
  },
  treeEndpoint: (id: string) => `/api/cases/${id}/tree`,
  relationships: [
    { childType: 'worker', label: 'Workers', endpointTemplate: '/api/cases/{parentId}/workers' },
    { childType: 'milestone', label: 'Milestones', endpointTemplate: '/api/cases/{parentId}/milestones' },
    { childType: 'stage', label: 'Stages', endpointTemplate: '/api/cases/{parentId}/stages' },
  ],
};

// Workers — aggregated across sub-types
const devtownWorkerType = {
  ...workerType({ listEndpoint: '/api/workers' }),
  subTypes: ['worker:agent', 'worker:flow', 'worker:human'],
};
```

## 7. REST Endpoints Required

### 7.1 Engine-side (generic, all apps benefit)

These endpoints belong in the engine, not devtown — every casehub app needs them.

| Endpoint | Method | Returns | Notes |
|----------|--------|---------|-------|
| `/api/cases` | GET | `EntityListResponse` | List case instances with cursor pagination |
| `/api/cases/{id}` | GET | `EntityInstance` | Full case state per §4.2 |
| `/api/cases/{id}/tree` | GET | `EntityTreeNode[]` | First two levels of case hierarchy |
| `/api/cases/{id}/workers` | GET | `EntityListResponse` | Workers for a case |
| `/api/cases/{id}/milestones` | GET | `EntityListResponse` | Milestones with runtime state |
| `/api/cases/{id}/stages` | GET | `EntityListResponse` | Stages with containment |
| `/api/case-definitions` | GET | `EntityListResponse` | List registered definitions |
| `/api/case-definitions/{id}` | GET | `EntityInstance` | Full definition per §4.3 |
| `/api/workers/{id}` | GET | `EntityInstance` | Worker detail |
| `/api/cases/{id}/commands` | POST | `CommandResult` | Execute case management commands |

The case-explorer spec §5 defines `EntityStateContributor` as the SPI — engine implements contributors for `case-instance`, `case-definition`, `stage`, `milestone`, and base `worker`. Apps implement sub-type contributors.

### 7.2 Devtown-side

| Endpoint | Method | Returns | Notes |
|----------|--------|---------|-------|
| — | — | — | No devtown-specific endpoints needed. All REST is engine-generic. devtown contributes only `EntityStateContributor` implementations for worker sub-types (agent, flow, human). |

## 8. Engine Issues to File

| Issue | Repo | Scope | Complexity | What |
|-------|------|-------|------------|------|
| `ExpressionEvaluator.displayExpression()` | casehubio/engine | XS | Low | Default method returning `type()`. JQ overrides to return `expression()`. Lambda accepts optional label at construction. |
| `MilestoneRuntimeState` | casehubio/engine | S | Low | Record with status + activatedAt + deadline + breached. Replace `getMilestoneStatus()` with `getMilestoneState()`. |
| Goal reached state tracking | casehubio/engine | S | Low | `markGoalReached()`, `getReachedGoals()`, `isGoalReached()` on CasePlanModel. Called by GoalReachedEventHandler. |
| Case entity REST endpoints | casehubio/engine | M | Med | EntityStateContributor implementations + REST resources for cases, definitions, workers, milestones, stages. Tree endpoint. |
| WebSocket entity events | casehubio/engine | S | Med | CDI `EntityStateChanged` events → WebSocket topic publishing per case-explorer spec §6 |

## 9. Devtown Implementation Tasks

With engine gaps fixed, devtown's work is:

| Task | Scale | What |
|------|-------|------|
| Worker sub-type EntityStateContributors | S | `AgentWorkerStateContributor`, `FlowWorkerStateContributor`, `HumanWorkerStateContributor` — expose runtime state and commands per worker type |
| Frontend entity type registrations | XS | Register entity types per §6 in `app/src/main/webui/` |
| PR review detail renderer | S | Lit component showing PR metadata + capability progress pipeline (§5.1) |
| Coordinated change detail renderer | S | Lit component showing per-repo grid + merge/rollback status (§5.2) |
| Binding condition viewer | XS | Table renderer for case-definition detail (§5.3) |

**Total devtown scope:** M/Med (mostly straightforward wiring, but three custom detail renderers need domain knowledge).

**Total engine scope:** M/Med (API gaps are small individually but REST endpoints + WebSocket are material).

## 10. Dependencies and Ordering

```
Engine: ExpressionEvaluator.displayExpression()  ──┐
Engine: MilestoneRuntimeState                     ──┤── Engine: Case entity REST endpoints
Engine: Goal reached state                        ──┘         │
                                                              │
                                                              ▼
                                              Devtown: EntityStateContributor SPIs
                                              Devtown: Frontend registrations
                                              Devtown: Detail renderers
```

Engine API gaps (§3.1–3.3) are independent of each other and can be done in parallel. The REST endpoints depend on all three. Devtown work depends on the REST endpoints existing.

## 11. Testing

### 11.1 Engine

- `ExpressionEvaluator.displayExpression()` — unit tests for JQ (returns expression string) and Lambda (returns label or fallback)
- `MilestoneRuntimeState` — unit test: activate milestone, verify deadline computed, verify breach detection
- Goal reached tracking — unit test: fire GoalReachedEvent, verify `isGoalReached()` returns true, verify `getReachedGoals()` accumulates
- REST endpoints — `@QuarkusTest` with in-memory engine, create case, verify EntityInstance serialization

### 11.2 Devtown

- EntityStateContributor SPIs — `@QuarkusTest`: create a PR review case, verify worker sub-type detail includes expected state and commands
- Frontend registrations — TypeScript unit test: verify entity types are registered with correct endpoints and column configs
- Detail renderers — TypeScript unit test: verify PR review renderer shows capability pipeline, coordinated change renderer shows per-repo grid

## 12. Scope Summary

| Component | Issues | Scale | Complexity |
|-----------|--------|-------|------------|
| Engine API gaps (§3.1–3.3) | 3 issues | S total | Low |
| Engine REST + WebSocket (§8) | 2 issues | M | Med |
| Devtown SPIs + frontend (§9) | devtown#119 | M | Med |
| **Total** | **6 issues** | **L** | **Med** |

This is larger than the original "just register entity types" estimate. The engine API gaps and REST endpoints are foundation work that benefits all apps — devtown is the first consumer but clinical, AML, and life will all use the same endpoints.
