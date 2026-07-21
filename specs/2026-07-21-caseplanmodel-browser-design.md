# CasePlanModel Browser — Design Spec

**Issue:** devtown#119
**Date:** 2026-07-21
**Repos:** devtown (primary), engine, blocks-ui
**Slot:** 17

## Overview

Two UI surfaces sharing a conceptual model — case definitions (what can happen) and live case state (what is happening) — built across three repos with clear ownership boundaries.

**Static definitions tab** — new top-level "Definitions" tab using blocks-ui `CaseDefinitionBrowser` via `hostPanel()`. Consumes engine-rest `GET /api/v1/case-definitions` directly. Summary list with drill-down to definition detail showing capabilities, bindings, goals, and completion criteria.

**Live case state** — integrated into the existing Reviews tab detail view. Three new sections (Plan Items, Blackboard Context, Goal Progress) consuming new engine-rest endpoints via `dataset()`. Devtown adds only PR-specific enrichment.

## Architecture

### Data flow

```
Static:   engine-rest /api/v1/case-definitions
            → EntityReader (blocks-ui#90)
            → CaseDefinitionBrowser
            → hostPanel()
            → Definitions tab

Live:     engine-rest /api/v1/cases/{caseId}/plan-items
          engine-rest /api/v1/cases/{caseId}/goals
          engine-rest /api/v1/cases/{caseId}/context  (existing)
            → dataset()
            → Reviews detail DSL sections
```

### Ownership boundaries

| Concern | Owner | Why |
|---------|-------|-----|
| Plan item listing | engine-rest | Generic — every CaseHub app has plan items |
| Goal evaluation against live context | engine-rest | Generic — engine owns CaseDefinition and CaseContext |
| Blackboard context query | engine-rest | Already exists: `GET /api/v1/cases/{caseId}/context` |
| EntityReader pattern | blocks-ui | Generic — any consumer of case-explorer needs to map domain shapes |
| Definition detail rendering | devtown | Domain-specific — devtown's capabilities, binding tiers, routing policies |
| PR metadata enrichment | devtown | Domain-specific — `PrPayload`, `PrReviewCaseTracker` |
| Trust/commitment context | devtown | Domain-specific — routing policies, qhorus commitments |

AML and Clinical get the engine-rest endpoints for free. blocks-ui components are reusable across all apps.

### Known limitation: CaseDefinition serialization

`CaseDefinitionResource.listAll()` returns `PagedResponse<CaseDefinition>` — the raw domain object, not a DTO. Two consequences:

1. **Lambda conditions are null in serialized output.** `LambdaExpressionEvaluator` wraps a `Predicate<CaseContext>` which is not Jackson-serializable. For DSL-built case definitions using lambda predicates, the `when` field on bindings and the `condition` field on goals serialize as null. The Definitions tab's "Condition" column will show empty for these. This is a known gap until engine#767 (serializable binding/goal conditions) lands.

2. **Internal fields exposed.** `CaseDefinition` has 30+ fields including engine internals (`contextStoreFactory`, `defaultWorkerBridge`, `cbrConfig`, `candidateMatching`, `implementationRouting`, etc.). The `definitionReader` in devtown extracts only the fields it needs; the rest are ignored. A `CaseDefinitionResponse` DTO is an engine-rest concern tracked separately from this spec.

## Engine-rest changes

New endpoints are sub-resources on `CaseInstanceResource` at `/api/v1/cases/{caseId}/...`, consistent with the existing `/context` endpoint. Both follow the engine-rest module patterns: `@RunOnVirtualThread`, blocking SPI calls, DTO records with `from()` factory methods, exception boundary translation via `EntityNotFoundException`.

### New endpoint: Plan Items

```
GET /api/v1/cases/{caseId}/plan-items
```

Returns all plan items for a case instance.

**Response DTO:**

```java
public record PlanItemResponse(
    String planItemId,
    String bindingName,
    String targetType,     // "capability", "human-task", "sub-case", "extension"
    String status,         // TaskStatus enum: PENDING, RUNNING, DELEGATED, SUSPENDED,
                           //   COMPLETED, FAULTED, REJECTED, OBSOLETE, CANCELLED
    String executorName,   // assigned executor (if any)
    String description,    // binding description (if any)
    Instant createdAt
) {
    public static PlanItemResponse from(PlanItemRecord record) {
        return new PlanItemResponse(
            record.planItemId(),
            record.bindingName(),
            record.targetType().name().toLowerCase().replace('_', '-'),
            record.status().name(),
            record.executorName(),
            record.description(),
            record.createdAt()
        );
    }
}
```

**Implementation:** Added as a new method on `CaseInstanceResource`. Injects `PlanItemStore` SPI (in addition to existing injections). Uses `CaseService.requireCase()` for existence + tenancy verification, then `PlanItemStore.findByCaseId()` to retrieve `List<PlanItemRecord>`.

`PlanItemRecord` is the persistence record from the `PlanItemStore` SPI — it works for both running and terminated cases. Fields mapped directly: `planItemId`, `bindingName`, `targetType` (TargetType enum → lowercase string), `status` (TaskStatus enum), `executorName`, `description`, `createdAt`.

`targetType` derived from `PlanItemRecord.targetType()`: `TargetType` is an enum with four values — `CAPABILITY`, `HUMAN_TASK`, `SUB_CASE`, `EXTENSION` — matching the `BindingTarget` sealed hierarchy.

**Exception boundary:** `PlanItemStore.findByCaseId()` does not throw for non-existent cases (returns empty list). Case existence is verified by `CaseService.requireCase()` before querying plan items — 404 if not found.

### New endpoint: Goal Status

```
GET /api/v1/cases/{caseId}/goals
```

Evaluates each goal from the case's definition against the current case context.

**Response DTOs:**

```java
public record GoalStatusResponse(
    String name,
    String kind,           // SUCCESS, FAILURE, or custom GoalKind
    boolean satisfied,
    String condition,      // always the original expression string (JQ), null if lambda
    String error           // null if evaluation succeeded, error message if it failed
) {}

public record CompletionStatus(
    boolean satisfied,
    String expressionType  // "allOf", "anyOf", "single"
) {}

public record CompletionSummary(
    String type,                                    // "goal-based" or "predicate-based"
    Boolean satisfied,                              // predicate-based: true/false; goal-based: null (use byKind)
    Map<String, CompletionStatus> byKind            // goal-based: per-kind detail; predicate-based: empty
) {}

public record GoalEvaluationResponse(
    List<GoalStatusResponse> goals,
    CompletionSummary completion                    // null if no completion configured
) {}
```

**Implementation:** Added as a new method on `CaseInstanceResource`. Additional SPI dependencies: `CaseDefinitionRegistry` and `ExpressionEngineRegistry`.

1. `CaseService.requireCase(caseId, tenancyId)` → verify existence + tenancy, get `CaseInstance`
2. `CaseDefinitionRegistry.findByIdentity(namespace, name, version)` → `CaseMetaModel` → `getCaseDefinition()` → `CaseDefinition`
3. `CaseHubRuntime.query(caseId, ".")` → live `CaseContext`
4. For each `Goal` in `definition.getGoals()`:
   a. Extract condition string via pattern matching on `goal.getCondition()`:
      - `JQExpressionEvaluator jq` → `jq.expression()`
      - `LambdaExpressionEvaluator` → `null`
   b. Try evaluate: `ExpressionEngineRegistry.evaluate(goal.getCondition(), caseContext)` → `satisfied`
   c. On success: `GoalStatusResponse(name, kind, satisfied, conditionString, null)`
   d. On evaluation failure (JQ syntax error, type mismatch, etc.): `GoalStatusResponse(name, kind, false, conditionString, errorMessage)` — `satisfied` defaults to false as a safe fallback, `condition` preserves the original expression, `error` carries the failure message
5. Collect reached goal names: goals where `satisfied == true && error == null` → `Set<String>`
6. Build `CompletionSummary` based on `definition.getCompletion()`:
   a. If `GoalBasedCompletion`:
      - For each `GoalKind → GoalExpression` entry in `completion.getGoals()`:
        - `expression.isSatisfiedBy(reachedGoalNames)` → `satisfied`
        - Expression type from class: `AllOfGoalExpression` → `"allOf"`, `AnyOfGoalExpression` → `"anyOf"`, `SingleGoalExpression` → `"single"`
      - Build `Map<String, CompletionStatus>` keyed by `goalKind.value()`
      - `satisfied` = null (per-kind status is authoritative; top-level aggregation is semantically undefined for mixed-polarity kinds — SUCCESS triggers COMPLETED, FAILURE triggers FAULTED)
      - `type` = `"goal-based"`
   b. If `PredicateBasedCompletion`:
      - Evaluate `doneWhen` via `ExpressionEngineRegistry.evaluate(predicate.getDoneWhen(), caseContext)`
      - `satisfied` = evaluation result, `type` = `"predicate-based"`, `byKind` = empty map
   c. If `completion` is null: `CompletionSummary` = null
7. Return `GoalEvaluationResponse(goals, completionSummary)`

**Goal evaluation errors vs unsatisfied goals:** A goal that threw during evaluation is semantically different from a goal that evaluated to `false`. The `error` field on `GoalStatusResponse` distinguishes these: `error == null` means clean evaluation, `error != null` means the condition is broken. Goals with errors are excluded from the reached goal set (step 5) to prevent a crashed evaluation from counting toward completion. The UI can render errored goals with a warning indicator rather than a simple "not met" state.

`GoalExpression` is a sealed hierarchy (`AllOfGoalExpression`, `AnyOfGoalExpression`, `SingleGoalExpression`) supporting nested composition. `isSatisfiedBy(Set<String>)` evaluates whether the composition is met — `anyOf` requires at least one reached goal, `allOf` requires all.

`CaseCompletion` has two implementations: `GoalBasedCompletion` (maps `GoalKind → GoalExpression`, used by devtown's PR review case) and `PredicateBasedCompletion` (single `ExpressionEvaluator doneWhen`, used for simple cases). The `CompletionSummary` wrapper handles both with a `type` discriminator. For goal-based completion, `satisfied` is null because each `GoalKind` maps to a different terminal `CaseStatus` — `SUCCESS` triggers `COMPLETED`, `FAILURE` triggers `FAULTED` — so aggregating across kinds with AND or OR is semantically undefined. The frontend reads `completion.byKind.SUCCESS.satisfied` and `completion.byKind.FAILURE.satisfied` independently. For predicate-based completion, `satisfied` carries the single boolean result; `byKind` is empty.

**Exception boundary:** `ExpressionEngineRegistry.evaluate()` throws `IllegalArgumentException` for missing engines — the resource catches this and re-throws as `EntityNotFoundException`. Per-goal evaluation failures (JQ syntax errors, type mismatches) are caught individually and reported via the `error` field, preventing one broken goal from failing the entire response.

### Existing endpoint: Context

`GET /api/v1/cases/{caseId}/context` already returns the full blackboard state. No changes needed.

## blocks-ui changes

### EntityReader pattern (blocks-ui#90)

Full specification in the GitHub issue. Summary of what's needed for this feature:

**New interfaces in `types.ts`:**

```typescript
interface EntityReader<T = any> {
  id: (entity: T) => string;
  type?: (entity: T) => string;
  summary: (entity: T) => string;
  status: (entity: T) => string;
  createdAt?: (entity: T) => string;
  updatedAt?: (entity: T) => string;
  state?: (entity: T) => Record<string, unknown>;
  commands?: (entity: T) => readonly CommandDescriptor[];
}

interface ResponseReader<T = any> {
  entities: (response: any) => readonly T[];
  nextCursor?: (response: any) => string | undefined;
  totalCount?: (response: any) => number | undefined;
}
```

**Added to `EntityTypeRegistration`:** `reader?: EntityReader` and `responseReader?: ResponseReader`

**Built-in readers exported:** `DEFAULT_READER` (EntityInstance shape), `offsetPaginationReader` (engine `PagedResponse` shape)

**Change sites:** 5 in `entity-list.ts`, 7 in `entity-detail.ts`, convenience wrapper pass-through. Zero changes to `entity-tree.ts`, `navigation-controller.ts`, `entity-command-bar.ts`.

Full backward compatibility — omitting `reader` uses `DEFAULT_READER` which assumes `EntityInstance` shape.

## devtown changes

### Frontend: New "Definitions" tab

**File: `app/src/main/webui/src/views/definitions.ts`**

```typescript
import { page, hostPanel } from "@casehubio/pages-ui";
import { caseDefinitionType, offsetPaginationReader } from "@casehubio/blocks-ui-case-explorer";

const definitionReader = {
  id: (d) => `${d.namespace}/${d.name}/${d.version}`,
  summary: (d) => d.title || d.name,
  status: (d) => `v${d.version}`,
  state: (d) => ({
    namespace: d.namespace,
    capabilities: d.capabilities?.length ?? 0,
    bindings: d.bindings?.length ?? 0,
    goals: d.goals?.length ?? 0,
  }),
  commands: () => [],
};

export const definitionsView = page("Definitions",
  hostPanel("case-definition-browser", {
    endpoint: "/api/v1/case-definitions",
    reader: definitionReader,
    responseReader: offsetPaginationReader,
    // detailRenderer provided as a Lit template function — see detail section below
  }),
);
```

**File: `app/src/main/webui/src/index.ts`** — add `["Definitions", definitionsView]` to tabs.

**File: `app/src/main/webui/package.json`** — add `@casehubio/blocks-ui-case-explorer` dependency.

### Frontend: Definition detail renderer

Custom `detailRenderer` passed to the case-definition-browser. Renders three sections:

**Goals section** — table: Name | Kind | Condition
- Kind color-coded: green SUCCESS, red FAILURE
- Condition shows JQ expression or "lambda" placeholder

**Bindings section** — table: Name | Target Type | Target Name | Condition | Conflict Strategy | Outcome Policy
- Grouped by tier (derived from binding name prefixes): initial dispatch, content-driven, precedent-triggered, human gate, scope reduction, escalation, **other** (bindings not matching any recognized prefix)

**Capabilities section** — table: Name | Input Schema | Output Schema

### Frontend: Live case state in Reviews detail

**File: `app/src/main/webui/src/datasets.ts`** — add three new datasets:

```typescript
dataset("plan-items", "/api/v1/cases/#{row.caseId}/plan-items");
dataset("goal-status", "/api/v1/cases/#{row.caseId}/goals");
dataset("case-context", "/api/v1/cases/#{row.caseId}/context");
```

**Parameterized URL mechanism:** pages-ui supports template URL syntax (`#{path.to.value}`). The data pipeline detects template variables via `hasTemplateVars()`, defers fetch until all variables resolve via `allTemplateVarsResolved()`, and automatically re-fetches when the resolved URL changes. `#{row.caseId}` resolves from the `RuntimeContext.row` — the currently selected review row. When a different review is selected, the URL re-resolves and the datasets re-fetch.

**Prerequisite:** The reviews dataset must include `caseId` in each row's data. The `PrReviewCaseTracker` tracks the `CaseInstance` UUID — this must be mapped to a `caseId` field in the reviews dataset response.

**File: `app/src/main/webui/src/views/reviews.ts`** — add three sections to the `reviewDetail` layout after "Capability Progress":

**Plan Items section:**
```typescript
title("Plan Items", 3),
table({
  lookup: lookup("plan-items", groupBy("planItemId",
    col("bindingName"), col("targetType"),
    col("status"), col("executorName"), col("createdAt")
  )),
  sortable: true,
}),
```

**Blackboard Context section:**
```typescript
title("Case Context", 3),
table({
  lookup: lookup("case-context", groupBy("key", col("key"), col("value"))),
}),
```

**Goal Progress section:**
```typescript
title("Goal Progress", 3),
columns([70, 30],
  [table({
    lookup: lookup("goal-status", groupBy("name",
      col("name"), col("kind"), col("satisfied")
    )),
  })],
  [rows(
    metric({ title: "Success Completion", lookup: lookup("goal-status", groupBy(null, col("completion.byKind.SUCCESS.satisfied"))), subtype: "plain-text" }),
    metric({ title: "Failure Triggered", lookup: lookup("goal-status", groupBy(null, col("completion.byKind.FAILURE.satisfied"))), subtype: "plain-text" }),
  )],
),
```

### Backend: Maven dependency

**File: `app/pom.xml`** — add `casehub-engine-rest` dependency to activate the engine REST endpoints in devtown's Quarkus app:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-rest</artifactId>
</dependency>
```

Version managed by `<dependencyManagement>` in the parent pom (all casehubio artifacts are `0.2-SNAPSHOT`).

## Implementation order

```
Phase 1 — engine-rest (no dependencies)
  1. PlanItemResponse DTO with from(PlanItemRecord) factory
  2. GET /api/v1/cases/{caseId}/plan-items on CaseInstanceResource
  3. GoalStatusResponse + CompletionStatus + GoalEvaluationResponse DTOs
  4. GET /api/v1/cases/{caseId}/goals on CaseInstanceResource
  5. Tests for both endpoints

Phase 2 — blocks-ui EntityReader (parallel with Phase 1)
  1. EntityReader + ResponseReader interfaces in types.ts
  2. Wire reader into entity-list.ts (5 sites) and entity-detail.ts (7 sites)
  3. DEFAULT_READER + offsetPaginationReader exports
  4. Convenience wrapper pass-through
  5. Tests

Phase 3 — devtown frontend (depends on Phase 1 + 2)
  1. Add @casehubio/blocks-ui-case-explorer dependency
  2. views/definitions.ts with hostPanel + reader config + custom detailRenderer
  3. datasets.ts — add plan-items, goal-status, case-context datasets
  4. views/reviews.ts — add Plan Items, Case Context, Goal Progress sections
  5. index.ts — register Definitions tab

Phase 4 — devtown backend (parallel with Phase 1; must complete before Phase 3 testing)
  1. Add casehub-engine-rest Maven dependency to app/pom.xml
  2. Verify engine-rest endpoints active in devtown's Quarkus dev mode
```

## Testing strategy

**Engine-rest:**
- Unit tests for `PlanItemResponse.from(PlanItemRecord)` mapping
- Unit tests for `GoalStatusResponse` evaluation (JQ condition → satisfied, lambda condition → satisfied with null expression)
- Unit tests for `GoalEvaluationResponse` completion: `anyOf` composition satisfied with subset, `allOf` requires all
- `@QuarkusTest` integration test: start a case, signal context, verify plan-items and goals endpoints return correct state

**blocks-ui:**
- Existing tests pass unchanged (DEFAULT_READER backward compat)
- New test: entity-list with custom reader — verify columns render from domain fields
- New test: entity-detail with custom reader — verify header uses `reader.summary()`/`reader.status()`
- New test: offsetPaginationReader — verify cursor derivation from `{ items, page, totalPages }`

**devtown:**
- `@QuarkusTest` verifying engine-rest endpoints are reachable when casehub-engine-rest is on the classpath
- Manual browser verification of the Definitions tab and live state sections in dev mode

## Related issues

| Repo | # | Relationship |
|------|---|-------------|
| devtown | 119 | Primary — this spec |
| devtown | 167 | WebSocket/SSE live updates (out of scope) |
| devtown | 168 | Merge queue case definition (out of scope) |
| devtown | 169 | Coordinated change case definition (out of scope) |
| devtown | 170 | Binding activation state (deferred from #119) |
| engine | 767 | Serializable binding/goal conditions (not a blocker, enhances condition display) |
| blocks-ui | 90 | EntityReader pattern (Phase 2 of this work) |
| blocks-ui | 41 | Governance workbench migration epic (this is the first Phase 1 adoption) |
| engine | 657 | Extract engine-rest (already done — engine#762 closed) |

## Out of scope

- Merge queue case definition (devtown#168)
- Coordinated change case definition (devtown#169)
- WebSocket/SSE live updates for plan item state changes (devtown#167)
- Binding activation state — what blackboard state satisfied each binding (devtown#170, deferred from #119)
- `<routing-rationale>` and `<commitment-lifecycle>` blocks-ui components (blocks-ui#54, #55)
- Full blocks-ui migration of existing devtown views (blocks-ui#41)
