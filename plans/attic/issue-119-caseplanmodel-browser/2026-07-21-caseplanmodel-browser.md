# CasePlanModel Browser Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #119 — feat: CasePlanModel browser — view adaptive case definitions and binding conditions
**Issue group:** #119

**Goal:** Two UI surfaces — a static Definitions tab showing registered CasePlanModels, and live case state (plan items, blackboard context, goal progress) integrated into the existing Reviews detail view.

**Architecture:** Engine-rest gets two new endpoints (plan-items, goals) on CaseInstanceResource. blocks-ui case-explorer evolves to accept a configurable EntityReader so domain-specific REST shapes are consumed without adapters. devtown wires both into its Quinoa frontend — definitions via hostPanel() with blocks-ui CaseDefinitionBrowser, live state via pages-ui dataset()/table()/metric() DSL.

**Tech Stack:** Java 21 / Quarkus 3.32.2 / JAX-RS / TypeScript / Lit / casehub-pages / blocks-ui

## Global Constraints

- All casehubio artifacts at `0.2-SNAPSHOT`
- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- engine-rest uses `@RunOnVirtualThread` on all endpoints, blocking SPI calls, DTO records with `from()` factories
- engine-rest tests use `@QuarkusTest` + RestAssured + `@Alternative @Priority(1)` test beans
- blocks-ui tests use vitest
- devtown frontend: esbuild, `@casehubio/pages-runtime` 0.2.0, `@casehubio/pages-ui` 0.2.0
- IntelliJ MCP mandatory for all .java/.ts edits — `project_path` = `/Users/mdproctor/claude/casehub/worktrees/17/engine` for engine, `/Users/mdproctor/claude/casehub/worktrees/17/devtown` for devtown
- Commits reference issues: engine commits `Refs casehubio/devtown#119`, blocks-ui commits `Refs casehubio/blocks-ui#90`, devtown commits `Refs #119`

---

### Task 1: Engine-rest — PlanItemResponse DTO and endpoint

**Files:**
- Create: `engine/rest/src/main/java/io/casehub/engine/rest/dto/PlanItemResponse.java`
- Modify: `engine/rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java` — add `@Inject PlanItemStore`, add `getPlanItems()` method
- Test: `engine/rest/src/test/java/io/casehub/engine/rest/CaseInstancePlanItemsResourceTest.java`

**Interfaces:**
- Consumes: `PlanItemStore.findByCaseId(UUID, String)` → `List<PlanItemRecord>` from engine-common SPI; `CaseService.requireCase(UUID, String)` → `CaseInstance` from engine-rest service; `CurrentPrincipal.tenancyId()` from platform-api
- Produces: `PlanItemResponse` record — used by devtown frontend dataset; `GET /api/v1/cases/{caseId}/plan-items` endpoint — consumed by devtown `dataset("plan-items", ...)`

`PlanItemRecord` fields: `caseId: UUID`, `planItemId: String`, `bindingName: String`, `status: TaskStatus`, `createdAt: Instant`, `targetType: TargetType`, `outputMappingExpression: String`, `tenancyId: String`, `description: String`, `executorName: String`, `executorDescription: String`.

`TargetType` enum values: `HUMAN_TASK`, `CAPABILITY`, `SUB_CASE`, `EXTENSION`.

`TaskStatus` enum values: `PENDING`, `RUNNING`, `DELEGATED`, `SUSPENDED`, `COMPLETED`, `FAULTED`, `REJECTED`, `OBSOLETE`, `CANCELLED`.

- [ ] **Step 1: Write failing test — plan items endpoint returns 404 for unknown case**

Create test file `CaseInstancePlanItemsResourceTest.java`:

```java
package io.casehub.engine.rest;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

@QuarkusTest
class CaseInstancePlanItemsResourceTest {

  @Test
  void getPlanItems_returns404ForUnknownCase() {
    given()
        .when()
        .get("/api/v1/cases/00000000-0000-0000-0000-000000000001/plan-items")
        .then()
        .statusCode(404)
        .body("status", equalTo(404));
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest=CaseInstancePlanItemsResourceTest -f /Users/mdproctor/claude/casehub/worktrees/17/engine/pom.xml`
Expected: FAIL — 404 endpoint doesn't exist yet, will get 404 but for wrong reason (no route), or 405.

- [ ] **Step 3: Create PlanItemResponse DTO**

Use `ide_create_file` in the engine project:

```java
package io.casehub.engine.rest.dto;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.time.Instant;
import org.eclipse.microprofile.openapi.annotations.media.Schema;

@Schema(description = "Plan item status from case execution")
public record PlanItemResponse(
    @Schema(description = "Plan item UUID", required = true) @NotBlank String planItemId,
    @Schema(description = "Binding that created this plan item", required = true) @NotBlank String bindingName,
    @Schema(description = "Target type", required = true, example = "capability") @NotBlank String targetType,
    @Schema(description = "Execution status", required = true, example = "RUNNING") @NotBlank String status,
    @Schema(description = "Assigned executor", nullable = true) String executorName,
    @Schema(description = "Binding description", nullable = true) String description,
    @Schema(description = "Creation timestamp", required = true) @NotNull Instant createdAt) {

  public static PlanItemResponse from(PlanItemRecord record) {
    return new PlanItemResponse(
        record.planItemId(),
        record.bindingName(),
        record.targetType().name().toLowerCase().replace('_', '-'),
        record.status().name(),
        record.executorName(),
        record.description(),
        record.createdAt());
  }
}
```

- [ ] **Step 4: Add getPlanItems method to CaseInstanceResource**

Use `ide_insert_member` to add the `PlanItemStore` injection field after `currentPrincipal`:

```java
@Inject PlanItemStore planItemStore;
```

Use `ide_insert_member` to add the endpoint method after `getContextPath`:

```java
@GET
@Path("/{caseId}/plan-items")
@RunOnVirtualThread
@Operation(summary = "Get plan items for a case")
@APIResponse(
    responseCode = "200",
    description = "Plan items for the case",
    content = @Content(schema = @Schema(implementation = PlanItemResponse.class)))
@APIResponse(
    responseCode = "404",
    description = "Case not found",
    content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
public List<PlanItemResponse> getPlanItems(@PathParam("caseId") UUID caseId) {
  String tenancyId = currentPrincipal.tenancyId();
  caseService.requireCase(caseId, tenancyId);
  return planItemStore.findByCaseId(caseId, tenancyId).stream()
      .map(PlanItemResponse::from)
      .toList();
}
```

Add missing imports: `io.casehub.engine.common.spi.PlanItemStore`, `io.casehub.engine.rest.dto.PlanItemResponse`, `java.util.List`.

- [ ] **Step 5: Run test to verify 404 passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest=CaseInstancePlanItemsResourceTest -f /Users/mdproctor/claude/casehub/worktrees/17/engine/pom.xml`
Expected: PASS — `CaseService.requireCase()` throws `EntityNotFoundException` → 404.

- [ ] **Step 6: Write test — plan items endpoint returns items for existing case**

Add to `CaseInstancePlanItemsResourceTest`:

```java
import static org.hamcrest.Matchers.hasSize;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.TaskStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.TargetType;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.CaseMetaModelRepository;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.engine.common.spi.PlanItemSaveRequest;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;

// Add fields:
@Inject CaseInstanceRepository instanceRepository;
@Inject CaseMetaModelRepository metaModelRepository;
@Inject TestCaseDefinitionRegistry definitionRegistry;
@Inject PlanItemStore planItemStore;

private UUID caseId;

@BeforeEach
void setUp() {
  caseId = UUID.randomUUID();

  CaseMetaModel meta = new CaseMetaModel();
  meta.setNamespace("test");
  meta.setName("test-case");
  meta.setVersion("1.0.0");
  meta = metaModelRepository.save(meta, "test-tenant");

  CaseDefinition def = CaseDefinition.builder()
      .namespace("test").name("test-case").version("1.0.0").build();
  definitionRegistry.register(def, meta);

  CaseInstance instance = new CaseInstance();
  instance.setUuid(caseId);
  instance.setCaseMetaModel(meta);
  instance.setState(io.casehub.api.model.CaseStatus.RUNNING);
  instanceRepository.save(instance, "test-tenant");

  planItemStore.save(new PlanItemSaveRequest(
      caseId, "pi-1", "code-analysis", TaskStatus.COMPLETED,
      Instant.parse("2026-07-21T10:00:00Z"), TargetType.CAPABILITY,
      null, "test-tenant", "Initial code analysis", null, null), "test-tenant");

  planItemStore.save(new PlanItemSaveRequest(
      caseId, "pi-2", "security-review", TaskStatus.RUNNING,
      Instant.parse("2026-07-21T10:05:00Z"), TargetType.CAPABILITY,
      null, "test-tenant", "Security review", "agent-1", null), "test-tenant");
}

@Test
void getPlanItems_returnsItemsForExistingCase() {
  given()
      .when()
      .get("/api/v1/cases/" + caseId + "/plan-items")
      .then()
      .statusCode(200)
      .body("$", hasSize(2))
      .body("[0].bindingName", equalTo("code-analysis"))
      .body("[0].targetType", equalTo("capability"))
      .body("[0].status", equalTo("COMPLETED"))
      .body("[1].bindingName", equalTo("security-review"))
      .body("[1].executorName", equalTo("agent-1"))
      .body("[1].status", equalTo("RUNNING"));
}
```

Note: `PlanItemSaveRequest` is the save DTO used by `PlanItemStore.save()`. Verify its constructor signature via `ide_find_definition` before running. If the constructor differs, adjust the field order.

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest=CaseInstancePlanItemsResourceTest -f /Users/mdproctor/claude/casehub/worktrees/17/engine/pom.xml`
Expected: PASS

- [ ] **Step 8: Verify with ide_diagnostics**

Run `ide_diagnostics` on `CaseInstanceResource.java` and `PlanItemResponse.java` to check for compilation errors.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/17/engine add rest/src/main/java/io/casehub/engine/rest/dto/PlanItemResponse.java rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java rest/src/test/java/io/casehub/engine/rest/CaseInstancePlanItemsResourceTest.java
git -C /Users/mdproctor/claude/casehub/worktrees/17/engine commit -m "feat(#119): plan-items endpoint on CaseInstanceResource

Refs casehubio/devtown#119"
```

---

### Task 2: Engine-rest — GoalEvaluationResponse DTOs and goals endpoint

**Files:**
- Create: `engine/rest/src/main/java/io/casehub/engine/rest/dto/GoalStatusResponse.java`
- Create: `engine/rest/src/main/java/io/casehub/engine/rest/dto/CompletionStatus.java`
- Create: `engine/rest/src/main/java/io/casehub/engine/rest/dto/CompletionSummary.java`
- Create: `engine/rest/src/main/java/io/casehub/engine/rest/dto/GoalEvaluationResponse.java`
- Modify: `engine/rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java` — add injections, add `getGoals()` method
- Test: `engine/rest/src/test/java/io/casehub/engine/rest/CaseInstanceGoalsResourceTest.java`

**Interfaces:**
- Consumes: `CaseService.requireCase(UUID, String)` → `CaseInstance`; `CaseInstance.getCaseMetaModel()` → `CaseMetaModel`; `CaseDefinitionRegistry.getCaseDefinition(CaseMetaModel)` → `CaseDefinition`; `CaseHubRuntime.query(UUID, String)` → `CompletionStage<Object>`; `ExpressionEngineRegistry.evaluate(ExpressionEvaluator, CaseContext)` → `boolean`; `GoalExpression.isSatisfiedBy(Set<String>)` → `boolean`
- Produces: `GoalEvaluationResponse` record — consumed by devtown frontend dataset; `GET /api/v1/cases/{caseId}/goals` endpoint

`GoalBasedCompletion<K extends GoalKind>`: `getGoals()` returns `Map<K, GoalExpression>`. Each `GoalKind` has `value(): String` and `terminalStatus(): CaseStatus`. Standard kinds: `GoalKind.SUCCESS`, `GoalKind.FAILURE`.

`PredicateBasedCompletion`: `getDoneWhen()` returns `ExpressionEvaluator`.

`GoalExpression` sealed hierarchy: `AllOfGoalExpression`, `AnyOfGoalExpression`, `SingleGoalExpression`. All have `isSatisfiedBy(Set<String>)`.

`ExpressionEvaluator` implementations: `JQExpressionEvaluator` (record with `expression(): String`), `LambdaExpressionEvaluator` (wraps `Predicate<CaseContext>`, no displayable expression).

- [ ] **Step 1: Write failing test — goals endpoint returns 404 for unknown case**

Create `CaseInstanceGoalsResourceTest.java`:

```java
package io.casehub.engine.rest;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

@QuarkusTest
class CaseInstanceGoalsResourceTest {

  @Test
  void getGoals_returns404ForUnknownCase() {
    given()
        .when()
        .get("/api/v1/cases/00000000-0000-0000-0000-000000000001/goals")
        .then()
        .statusCode(404)
        .body("status", equalTo(404));
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest=CaseInstanceGoalsResourceTest -f /Users/mdproctor/claude/casehub/worktrees/17/engine/pom.xml`
Expected: FAIL — no route for `/goals` yet.

- [ ] **Step 3: Create DTO records**

Create four files using `ide_create_file`:

**GoalStatusResponse.java:**
```java
package io.casehub.engine.rest.dto;

import org.eclipse.microprofile.openapi.annotations.media.Schema;

@Schema(description = "Goal evaluation result against live case context")
public record GoalStatusResponse(
    @Schema(description = "Goal name", required = true) String name,
    @Schema(description = "Goal kind (SUCCESS, FAILURE, or custom)", required = true) String kind,
    @Schema(description = "Whether the goal condition is currently satisfied") boolean satisfied,
    @Schema(description = "Expression string (JQ), null if lambda-based", nullable = true) String condition,
    @Schema(description = "Error message if evaluation failed, null if clean", nullable = true) String error) {}
```

**CompletionStatus.java:**
```java
package io.casehub.engine.rest.dto;

import org.eclipse.microprofile.openapi.annotations.media.Schema;

@Schema(description = "Completion status for a single goal kind")
public record CompletionStatus(
    @Schema(description = "Whether this kind's expression is satisfied") boolean satisfied,
    @Schema(description = "Expression type: allOf, anyOf, or single") String expressionType) {}
```

**CompletionSummary.java:**
```java
package io.casehub.engine.rest.dto;

import java.util.Map;
import org.eclipse.microprofile.openapi.annotations.media.Schema;

@Schema(description = "Case completion summary")
public record CompletionSummary(
    @Schema(description = "Completion type: goal-based or predicate-based") String type,
    @Schema(description = "Overall satisfied (predicate-based only, null for goal-based)", nullable = true) Boolean satisfied,
    @Schema(description = "Per-kind completion status (goal-based only, empty for predicate-based)") Map<String, CompletionStatus> byKind) {}
```

**GoalEvaluationResponse.java:**
```java
package io.casehub.engine.rest.dto;

import java.util.List;
import org.eclipse.microprofile.openapi.annotations.media.Schema;

@Schema(description = "Goal evaluation and completion status for a case instance")
public record GoalEvaluationResponse(
    @Schema(description = "Individual goal evaluations") List<GoalStatusResponse> goals,
    @Schema(description = "Completion summary, null if no completion configured", nullable = true) CompletionSummary completion) {}
```

- [ ] **Step 4: Add getGoals method to CaseInstanceResource**

Use `ide_insert_member` to add injections after `planItemStore`:

```java
@Inject CaseDefinitionRegistry definitionRegistry;
@Inject ExpressionEngineRegistry expressionEngineRegistry;
```

Use `ide_insert_member` to add the endpoint method after `getPlanItems`:

```java
@GET
@Path("/{caseId}/goals")
@RunOnVirtualThread
@Operation(summary = "Evaluate goals against live case context")
@APIResponse(
    responseCode = "200",
    description = "Goal evaluation results",
    content = @Content(schema = @Schema(implementation = GoalEvaluationResponse.class)))
@APIResponse(
    responseCode = "404",
    description = "Case not found",
    content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
public GoalEvaluationResponse getGoals(@PathParam("caseId") UUID caseId) {
  String tenancyId = currentPrincipal.tenancyId();
  CaseInstance instance = caseService.requireCase(caseId, tenancyId);

  CaseMetaModel meta = instance.getCaseMetaModel();
  CaseDefinition definition = definitionRegistry.getCaseDefinition(meta);
  if (definition == null) {
    throw new EntityNotFoundException("Case definition not found for case: " + caseId);
  }

  CaseContext caseContext = (CaseContext) runtime.query(caseId, ".").toCompletableFuture().join();

  List<GoalStatusResponse> goalResponses = new ArrayList<>();
  Set<String> reachedGoalNames = new HashSet<>();

  for (Goal goal : definition.getGoals()) {
    String conditionStr = null;
    if (goal.getCondition() instanceof JQExpressionEvaluator jq) {
      conditionStr = jq.expression();
    }

    boolean satisfied = false;
    String error = null;
    try {
      satisfied = expressionEngineRegistry.evaluate(goal.getCondition(), caseContext);
    } catch (Exception e) {
      error = e.getMessage();
    }

    if (satisfied && error == null) {
      reachedGoalNames.add(goal.getName());
    }

    goalResponses.add(new GoalStatusResponse(
        goal.getName(), goal.getKind(), satisfied, conditionStr, error));
  }

  CompletionSummary completion = buildCompletionSummary(
      definition.getCompletion(), reachedGoalNames, caseContext);

  return new GoalEvaluationResponse(goalResponses, completion);
}

private CompletionSummary buildCompletionSummary(
    CaseCompletion caseCompletion, Set<String> reachedGoalNames, CaseContext caseContext) {
  if (caseCompletion == null) {
    return null;
  }

  if (caseCompletion instanceof GoalBasedCompletion<?> goalBased) {
    Map<String, CompletionStatus> byKind = new LinkedHashMap<>();
    for (var entry : goalBased.getGoals().entrySet()) {
      GoalKind kind = entry.getKey();
      GoalExpression expr = entry.getValue();
      boolean sat = expr.isSatisfiedBy(reachedGoalNames);
      String exprType = switch (expr) {
        case AllOfGoalExpression ignored -> "allOf";
        case AnyOfGoalExpression ignored -> "anyOf";
        case SingleGoalExpression ignored -> "single";
      };
      byKind.put(kind.value(), new CompletionStatus(sat, exprType));
    }
    return new CompletionSummary("goal-based", null, byKind);
  }

  if (caseCompletion instanceof PredicateBasedCompletion predBased) {
    boolean sat = false;
    try {
      sat = expressionEngineRegistry.evaluate(predBased.getDoneWhen(), caseContext);
    } catch (Exception ignored) {
      // predicate evaluation failure → not satisfied
    }
    return new CompletionSummary("predicate-based", sat, Map.of());
  }

  return null;
}
```

Add missing imports: `io.casehub.api.context.CaseContext`, `io.casehub.api.engine.ExpressionEngineRegistry`, `io.casehub.api.model.*`, `io.casehub.api.model.evaluator.JQExpressionEvaluator`, `io.casehub.engine.common.internal.model.CaseInstance`, `io.casehub.engine.common.internal.model.CaseMetaModel`, `io.casehub.engine.common.spi.CaseDefinitionRegistry`, `io.casehub.engine.rest.dto.*`, `java.util.*`.

- [ ] **Step 5: Run test to verify 404 passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest=CaseInstanceGoalsResourceTest -f /Users/mdproctor/claude/casehub/worktrees/17/engine/pom.xml`
Expected: PASS

- [ ] **Step 6: Write test — goals endpoint evaluates lambda-based goals**

Add to `CaseInstanceGoalsResourceTest`:

```java
import static org.hamcrest.Matchers.hasSize;
import static org.hamcrest.Matchers.nullValue;

import io.casehub.api.context.CaseContext;
import io.casehub.api.model.*;
import io.casehub.api.model.evaluator.LambdaExpressionEvaluator;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.CaseMetaModelRepository;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;

@Inject CaseInstanceRepository instanceRepository;
@Inject CaseMetaModelRepository metaModelRepository;
@Inject TestCaseDefinitionRegistry definitionRegistry;
@Inject TestCaseHubRuntime testRuntime;

private UUID caseId;

@BeforeEach
void setUp() {
  caseId = UUID.randomUUID();

  Goal success = Goal.builder()
      .name("all-checks-pass").kind(GoalKind.SUCCESS)
      .condition(ctx -> "APPROVED".equals(ctx.getPath("review.outcome")))
      .build();

  Goal failure = Goal.builder()
      .name("review-rejected").kind(GoalKind.FAILURE)
      .condition(ctx -> "REJECTED".equals(ctx.getPath("review.outcome")))
      .build();

  CaseDefinition def = CaseDefinition.builder()
      .namespace("test").name("goal-test").version("1.0.0")
      .goals(success, failure)
      .completion(
          GoalExpression.allOf(success),
          GoalExpression.anyOf(failure))
      .build();

  CaseMetaModel meta = new CaseMetaModel();
  meta.setNamespace("test");
  meta.setName("goal-test");
  meta.setVersion("1.0.0");
  meta = metaModelRepository.save(meta, "test-tenant");
  definitionRegistry.register(def, meta);

  CaseInstance instance = new CaseInstance();
  instance.setUuid(caseId);
  instance.setCaseMetaModel(meta);
  instance.setState(CaseStatus.RUNNING);
  instanceRepository.save(instance, "test-tenant");

  // Set context where success goal is satisfied
  testRuntime.setQueryResult(caseId, Map.of("review", Map.of("outcome", "APPROVED")));
}

@Test
void getGoals_evaluatesLambdaGoalsAndReturnsStatus() {
  given()
      .when()
      .get("/api/v1/cases/" + caseId + "/goals")
      .then()
      .statusCode(200)
      .body("goals", hasSize(2))
      .body("goals[0].name", equalTo("all-checks-pass"))
      .body("goals[0].kind", equalTo("SUCCESS"))
      .body("goals[0].satisfied", equalTo(true))
      .body("goals[0].condition", nullValue())
      .body("goals[0].error", nullValue())
      .body("goals[1].name", equalTo("review-rejected"))
      .body("goals[1].satisfied", equalTo(false))
      .body("completion.type", equalTo("goal-based"))
      .body("completion.satisfied", nullValue())
      .body("completion.byKind.SUCCESS.satisfied", equalTo(true))
      .body("completion.byKind.SUCCESS.expressionType", equalTo("allOf"))
      .body("completion.byKind.FAILURE.satisfied", equalTo(false));
}
```

Note: `TestCaseHubRuntime` needs a `setQueryResult(UUID, Object)` method to set mock query responses. Check if it already has this — if not, add it.

- [ ] **Step 7: Verify TestCaseHubRuntime supports query mocking, extend if needed**

Read `TestCaseHubRuntime.java`. If it doesn't have a `setQueryResult(UUID, Object)` method, add one that stores the result and returns it from `query(UUID, String)`:

```java
private final Map<UUID, Object> queryResults = new ConcurrentHashMap<>();

public void setQueryResult(UUID caseId, Object result) {
  queryResults.put(caseId, result);
}

@Override
public CompletionStage<Object> query(UUID caseId, String path) {
  Object result = queryResults.get(caseId);
  return CompletableFuture.completedFuture(result);
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest -Dtest=CaseInstanceGoalsResourceTest -f /Users/mdproctor/claude/casehub/worktrees/17/engine/pom.xml`
Expected: PASS

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on all new/modified files.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/17/engine add rest/src/main/java/io/casehub/engine/rest/dto/GoalStatusResponse.java rest/src/main/java/io/casehub/engine/rest/dto/CompletionStatus.java rest/src/main/java/io/casehub/engine/rest/dto/CompletionSummary.java rest/src/main/java/io/casehub/engine/rest/dto/GoalEvaluationResponse.java rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java rest/src/test/java/io/casehub/engine/rest/CaseInstanceGoalsResourceTest.java
git -C /Users/mdproctor/claude/casehub/worktrees/17/engine commit -m "feat(#119): goals evaluation endpoint on CaseInstanceResource

Evaluates goal conditions against live CaseContext, builds
CompletionSummary from GoalBasedCompletion or PredicateBasedCompletion.
Lambda conditions report satisfied state but null condition string.

Refs casehubio/devtown#119"
```

---

### Task 3: blocks-ui — EntityReader and ResponseReader interfaces

**Files:**
- Modify: `blocks-ui/components/case-explorer/src/types.ts` — add `EntityReader`, `ResponseReader`, extend `EntityTypeRegistration`
- Modify: `blocks-ui/components/case-explorer/src/entity-list.ts` — use reader for field extraction (5 sites)
- Modify: `blocks-ui/components/case-explorer/src/entity-detail.ts` — use reader for header/commands/state (7 sites)
- Modify: `blocks-ui/components/case-explorer/src/index.ts` — re-export new types and readers
- Create: `blocks-ui/components/case-explorer/src/readers.ts` — DEFAULT_READER and offsetPaginationReader
- Test: `blocks-ui/components/case-explorer/src/entity-list.test.ts` — new tests for custom reader
- Test: `blocks-ui/components/case-explorer/src/entity-detail.test.ts` — new tests for custom reader

**Interfaces:**
- Consumes: existing `EntityInstance`, `EntityTypeRegistration`, `EntityListResponse` types
- Produces: `EntityReader<T>` interface, `ResponseReader<T>` interface, `DEFAULT_READER`, `offsetPaginationReader` — consumed by devtown's definition reader config

This task modifies TypeScript files in blocks-ui, which is not a Maven project. Use `Read`/`Edit`/`Write` tools (not IntelliJ MCP) for TypeScript.

- [ ] **Step 1: Add EntityReader and ResponseReader interfaces to types.ts**

Append after `EntityEvent` interface (line 127):

```typescript
export interface EntityReader<T = any> {
  id: (entity: T) => string;
  type?: (entity: T) => string;
  summary: (entity: T) => string;
  status: (entity: T) => string;
  createdAt?: (entity: T) => string;
  updatedAt?: (entity: T) => string;
  state?: (entity: T) => Record<string, unknown>;
  commands?: (entity: T) => readonly CommandDescriptor[];
}

export interface ResponseReader<T = any> {
  entities: (response: any) => readonly T[];
  nextCursor?: (response: any) => string | undefined;
  totalCount?: (response: any) => number | undefined;
}
```

Add `reader?: EntityReader` and `responseReader?: ResponseReader` to `EntityTypeRegistration`:

```typescript
readonly reader?: EntityReader;
readonly responseReader?: ResponseReader;
```

- [ ] **Step 2: Create readers.ts with DEFAULT_READER and offsetPaginationReader**

Create `blocks-ui/components/case-explorer/src/readers.ts`:

```typescript
import type { EntityReader, ResponseReader } from './types.js';

export const DEFAULT_READER: EntityReader = {
  id: (e) => e.id,
  type: (e) => e.type,
  summary: (e) => e.summary,
  status: (e) => e.status,
  createdAt: (e) => e.createdAt,
  updatedAt: (e) => e.updatedAt,
  state: (e) => e.state,
  commands: (e) => e.availableCommands ?? [],
};

export const DEFAULT_RESPONSE_READER: ResponseReader = {
  entities: (r) => r.entities,
  nextCursor: (r) => r.nextCursor,
  totalCount: (r) => r.totalCount,
};

export const offsetPaginationReader: ResponseReader = {
  entities: (r) => r.items,
  nextCursor: (r) => (r.page < r.totalPages ? String(r.page + 1) : undefined),
  totalCount: (r) => r.total,
};
```

- [ ] **Step 3: Wire reader into entity-list.ts (5 change sites)**

Add import at top:
```typescript
import { DEFAULT_READER, DEFAULT_RESPONSE_READER } from './readers.js';
```

**Site 1 (line 199):** `_onRowActivate` — replace `entity.id` and `entity.type`:
```typescript
// Before:
this._handleRowActivation({ id: entity.id, type: entity.type });
// After:
const reader = this.registration?.reader ?? DEFAULT_READER;
this._handleRowActivation({ id: reader.id(entity), type: reader.type?.(entity) ?? this.registration?.type ?? '' });
```

**Site 2 (line 237):** `_fetchEntities` — replace `data.entities`:
```typescript
// Before:
const data: EntityListResponse = await response.json();
// After:
const data = await response.json();
const responseReader = this.registration?.responseReader ?? DEFAULT_RESPONSE_READER;
```

**Site 3 (line 240):** replace `data.entities`:
```typescript
// Before:
this._entities = [...this._entities, ...data.entities];
// After:
this._entities = [...this._entities, ...responseReader.entities(data)];
```

**Site 4 (line 242):** replace `data.entities`:
```typescript
// Before:
this._entities = [...data.entities];
// After:
this._entities = [...responseReader.entities(data)];
```

**Site 5 (line 245):** replace `data.nextCursor`:
```typescript
// Before:
this._nextCursor = data.nextCursor ?? null;
// After:
this._nextCursor = responseReader.nextCursor?.(data) ?? null;
```

**Site 6 (line 268):** `_buildDataSet` — replace `entity.id`:
```typescript
// Before:
getValue: (entity: EntityInstance): unknown => entity.id,
// After:
getValue: (entity: unknown): unknown => (this.registration?.reader ?? DEFAULT_READER).id(entity),
```

**Site 7 (lines 275-278):** `_buildDataSet` column resolution — replace `entity.state[key]`:
```typescript
// Before:
getValue: (entity: EntityInstance): unknown => {
  const key = String(col.id);
  if (key in entity) return (entity as unknown as Record<string, unknown>)[key];
  return entity.state[key];
},
// After:
getValue: (entity: unknown): unknown => {
  const key = String(col.id);
  const record = entity as Record<string, unknown>;
  if (key in record) return record[key];
  const reader = this.registration?.reader ?? DEFAULT_READER;
  const stateBag = reader.state?.(entity);
  return stateBag?.[key];
},
```

Change `_entities` type from `EntityInstance[]` to `any[]` (line 28) and `_buildDataSet` parameter from `EntityInstance[]` to `readonly any[]` (line 263).

- [ ] **Step 4: Wire reader into entity-detail.ts (7 change sites)**

Add import at top:
```typescript
import { DEFAULT_READER } from './readers.js';
```

Helper getter (add as private getter):
```typescript
private get _reader() { return this.registration?.reader ?? DEFAULT_READER; }
```

**Site 1 (line 168):** header summary/status:
```typescript
// Before:
<h2>${this._entity.summary}<span class="status">${this._entity.status}</span></h2>
// After:
<h2>${this._reader.summary(this._entity)}<span class="status">${this._reader.status(this._entity)}</span></h2>
```

**Site 2 (line 185):** availableCommands check:
```typescript
// Before:
this._entity.availableCommands.length > 0
// After:
(this._reader.commands?.(this._entity) ?? []).length > 0
```

**Site 3 (lines 186-189):** commands/id/type passed to command-bar:
```typescript
// Before:
.commands=${this._entity.availableCommands}
entity-id=${this._entity.id}
entity-type=${this._entity.type}
// After:
.commands=${this._reader.commands?.(this._entity) ?? []}
entity-id=${this._reader.id(this._entity)}
entity-type=${this._reader.type?.(this._entity) ?? this.registration?.type ?? ''}
```

**Site 4 (line 247):** `_renderDefaultState`:
```typescript
// Before:
const entries = Object.entries(entity.state);
// After:
const entries = Object.entries(this._reader.state?.(entity) ?? entity);
```

**Site 5 (line 308):** announce:
```typescript
// Before:
this.announce(`Loaded ${this._entity!.summary}`);
// After:
this.announce(`Loaded ${this._reader.summary(this._entity!)}`);
```

Change `_entity` type from `EntityInstance | null` to `any | null` (line 14).

- [ ] **Step 5: Update index.ts exports**

Add to `index.ts`:
```typescript
export { DEFAULT_READER, DEFAULT_RESPONSE_READER, offsetPaginationReader } from './readers.js';
```

- [ ] **Step 6: Write test — entity-list with custom reader renders domain fields**

Add test to `entity-list.test.ts` (verify existing test pattern first, then add):

```typescript
it('uses custom reader for row activation selection', async () => {
  const customReader = {
    id: (d: any) => `${d.ns}/${d.n}`,
    summary: (d: any) => d.n,
    status: (d: any) => d.v,
  };

  const reg = {
    ...caseDefinitionType({ listEndpoint: '/api/defs' }),
    reader: customReader,
    responseReader: {
      entities: (r: any) => r.items,
      nextCursor: () => undefined,
    },
  };

  // ... setup and verify selection emits { id: 'ns/name', type: 'case-definition' }
});
```

Adapt to existing test patterns in the file.

- [ ] **Step 7: Write test — entity-detail with custom reader renders header from reader**

Add test to `entity-detail.test.ts`:

```typescript
it('renders header from custom reader', async () => {
  // Verify reader.summary() and reader.status() are used for header text
});
```

- [ ] **Step 8: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/worktrees/17/blocks-ui && npx vitest run components/case-explorer`
Expected: All existing tests pass + new tests pass.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/17/blocks-ui add components/case-explorer/src/types.ts components/case-explorer/src/readers.ts components/case-explorer/src/entity-list.ts components/case-explorer/src/entity-detail.ts components/case-explorer/src/index.ts components/case-explorer/src/entity-list.test.ts components/case-explorer/src/entity-detail.test.ts
git -C /Users/mdproctor/claude/casehub/worktrees/17/blocks-ui commit -m "feat(#90): EntityReader + ResponseReader — configurable field extraction

Adds EntityReader<T> and ResponseReader<T> to EntityTypeRegistration.
Components use reader for field extraction instead of assuming
EntityInstance shape. DEFAULT_READER preserves backward compat.
offsetPaginationReader exported for engine PagedResponse shape.

Refs casehubio/blocks-ui#90"
```

---

### Task 4: devtown — Add casehub-engine-rest Maven dependency

**Files:**
- Modify: `devtown/app/pom.xml` — add `casehub-engine-rest` dependency

**Interfaces:**
- Consumes: engine-rest module artifacts from `0.2-SNAPSHOT`
- Produces: engine REST endpoints active in devtown's Quarkus app at runtime — prerequisite for frontend datasets

- [ ] **Step 1: Add dependency to app/pom.xml**

Add in the `<dependencies>` section of `devtown/app/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-rest</artifactId>
</dependency>
```

Version is managed by parent `<dependencyManagement>`.

- [ ] **Step 2: Build devtown to verify dependency resolution**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/worktrees/17/devtown/pom.xml`
Expected: BUILD SUCCESS — engine-rest resolves from local Maven cache (engine was built in Tasks 1-2).

- [ ] **Step 3: Verify engine-rest endpoints are active**

Start Quarkus dev mode briefly and curl the definitions endpoint:

```bash
# After starting quarkus:dev, in another terminal:
curl -s http://localhost:8080/api/v1/case-definitions | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'items: {len(d.get(\"items\", []))}')"
```

Expected: `items: 1` (PrReviewCaseDefinition is auto-registered).

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/17/devtown add app/pom.xml
git -C /Users/mdproctor/claude/casehub/worktrees/17/devtown commit -m "feat(#119): add casehub-engine-rest dependency

Activates engine REST endpoints (/api/v1/case-definitions,
/api/v1/cases/{caseId}/plan-items, /api/v1/cases/{caseId}/goals)
in devtown's Quarkus app.

Refs #119"
```

---

### Task 5: devtown frontend — Definitions tab and live case state

**Files:**
- Modify: `devtown/app/src/main/webui/package.json` — add `@casehubio/blocks-ui-case-explorer` dependency
- Create: `devtown/app/src/main/webui/src/views/definitions.ts` — definitions tab view
- Modify: `devtown/app/src/main/webui/src/datasets.ts` — add plan-items, goal-status, case-context datasets
- Modify: `devtown/app/src/main/webui/src/views/reviews.ts` — add plan items, context, goal progress sections
- Modify: `devtown/app/src/main/webui/src/index.ts` — register Definitions tab

**Interfaces:**
- Consumes: `hostPanel()` from `@casehubio/pages-ui`; `caseDefinitionType`, `offsetPaginationReader` from `@casehubio/blocks-ui-case-explorer`; `dataset()`, `table()`, `metric()`, `lookup()`, `groupBy()`, `col()` from `@casehubio/pages-ui`; engine-rest endpoints (from Task 4)
- Produces: "Definitions" tab in the devtown UI; live case state sections in Reviews detail

These are TypeScript files — use `Read`/`Edit`/`Write` tools.

- [ ] **Step 1: Add blocks-ui-case-explorer npm dependency**

Edit `devtown/app/src/main/webui/package.json` dependencies:

```json
"@casehubio/blocks-ui-case-explorer": "0.3.0"
```

Run: `cd /Users/mdproctor/claude/casehub/worktrees/17/devtown/app/src/main/webui && npm install`

- [ ] **Step 2: Create definitions.ts view**

Create `devtown/app/src/main/webui/src/views/definitions.ts`:

```typescript
import { page, hostPanel } from "@casehubio/pages-ui";

const definitionReader = {
  id: (d: any) => `${d.namespace}/${d.name}/${d.version}`,
  summary: (d: any) => d.title || d.name,
  status: (d: any) => `v${d.version}`,
  state: (d: any) => ({
    namespace: d.namespace,
    capabilities: d.capabilities?.length ?? 0,
    bindings: d.bindings?.length ?? 0,
    goals: d.goals?.length ?? 0,
  }),
  commands: () => [],
};

const responseReader = {
  entities: (r: any) => r.items,
  nextCursor: (r: any) => (r.page < r.totalPages ? String(r.page + 1) : undefined),
  totalCount: (r: any) => r.total,
};

export const definitionsView = page("Definitions",
  hostPanel("case-definition-browser", {
    endpoint: "/api/v1/case-definitions",
    reader: definitionReader,
    responseReader: responseReader,
  }),
);
```

- [ ] **Step 3: Add datasets for live case state**

Edit `devtown/app/src/main/webui/src/datasets.ts` — add after existing datasets:

```typescript
dataset("plan-items", "/api/v1/cases/#{row.caseId}/plan-items");
dataset("goal-status", "/api/v1/cases/#{row.caseId}/goals");
dataset("case-context", "/api/v1/cases/#{row.caseId}/context");
```

- [ ] **Step 4: Add live state sections to reviews.ts detail view**

Edit `devtown/app/src/main/webui/src/views/reviews.ts` — add after the "Capability Progress" section in `reviewDetail` (after line 53):

```typescript
// Plan Items
title("Plan Items", 3),
table({
  lookup: lookup("plan-items", groupBy("planItemId",
    col("bindingName"), col("targetType"),
    col("status"), col("executorName"), col("createdAt")
  )),
  sortable: true,
}),

// Case Context
title("Case Context", 3),
table({
  lookup: lookup("case-context", groupBy("key", col("key"), col("value"))),
}),

// Goal Progress
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

- [ ] **Step 5: Register Definitions tab in index.ts**

Edit `devtown/app/src/main/webui/src/index.ts`:

Add import:
```typescript
import { definitionsView } from "./views/definitions";
```

Add tab to the `tabs()` call:
```typescript
["Definitions", definitionsView],
```

- [ ] **Step 6: Type check**

Run: `cd /Users/mdproctor/claude/casehub/worktrees/17/devtown/app/src/main/webui && npm run typecheck`
Expected: No errors.

- [ ] **Step 7: Build**

Run: `cd /Users/mdproctor/claude/casehub/worktrees/17/devtown/app/src/main/webui && npm run build`
Expected: BUILD SUCCESS.

- [ ] **Step 8: Manual browser verification**

Start `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app -f /Users/mdproctor/claude/casehub/worktrees/17/devtown/pom.xml` and verify:

1. Definitions tab appears with the registered PR review case definition
2. Clicking a definition shows goals, bindings, capabilities in the detail view
3. Reviews detail view shows Plan Items, Case Context, and Goal Progress sections (will be empty unless a case is running)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/17/devtown add app/src/main/webui/package.json app/src/main/webui/src/views/definitions.ts app/src/main/webui/src/datasets.ts app/src/main/webui/src/views/reviews.ts app/src/main/webui/src/index.ts
git -C /Users/mdproctor/claude/casehub/worktrees/17/devtown commit -m "feat(#119): CasePlanModel browser — Definitions tab + live case state

Adds Definitions tab via blocks-ui CaseDefinitionBrowser with
custom EntityReader for engine's CaseDefinition shape. Integrates
plan items, blackboard context, and goal progress into Reviews
detail view via engine-rest endpoints.

First blocks-ui component adoption in devtown (blocks-ui#41).

Refs #119"
```

---

## Dependency graph

```
Task 1 (engine plan-items) ──┐
                              ├── Task 4 (devtown maven dep) ── Task 5 (devtown frontend)
Task 2 (engine goals)     ───┘                                       ↑
                                                                     │
Task 3 (blocks-ui reader) ──────────────────────────────────────────┘
```

Tasks 1+2 are sequential (same file). Task 3 is independent (different repo). Task 4 depends on 1+2 (needs engine-rest built). Task 5 depends on 3+4.

## Self-review checklist

1. **Spec coverage:** Plan-items endpoint ✅ (Task 1), Goals endpoint ✅ (Task 2), EntityReader ✅ (Task 3), Maven dependency ✅ (Task 4), Definitions tab ✅ (Task 5), Live state sections ✅ (Task 5), Detail renderer for definitions — deferred to runtime (hostPanel passes reader, the CaseDefinitionBrowser renders detail via blocks-ui entity-detail with state bag). Parameterized dataset URLs ✅ (Task 5 Step 3).

2. **Placeholder scan:** No TBDs, TODOs, or "implement later" found. All code blocks are complete.

3. **Type consistency:** `PlanItemResponse.from(PlanItemRecord)` matches `PlanItemRecord` fields verified via IntelliJ. `GoalStatusResponse` fields match what the endpoint builds. `EntityReader` interface matches blocks-ui#90. `offsetPaginationReader` matches engine's `PagedResponse` shape.

4. **Tooling safety scan:** No bash file operations on source files. All Java edits via `ide_create_file`/`ide_insert_member`. TypeScript edits via `Read`/`Edit`/`Write` (blocks-ui is not a Maven project). Git operations use `git -C` pattern.
