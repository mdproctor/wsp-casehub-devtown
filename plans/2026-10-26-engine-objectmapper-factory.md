# Engine ObjectMapper Factory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #262 — Fix Jackson Instant serialization in WorkerScheduleEventHandler
**Issue group:** #262

**Goal:** Create a shared `EngineMappers` factory in casehub-engine's `common-core`
module and replace all bare `new ObjectMapper()` instances across engine production
code with the factory, fixing the `Instant` serialization bug and preventing recurrence.

**Architecture:** Single utility class `EngineMappers` in `common-core/internal/utils/`
with a `defaultMapper()` method that returns `new ObjectMapper().registerModule(new JavaTimeModule())`.
Each existing `new ObjectMapper()` in production code becomes `EngineMappers.defaultMapper()`.
Test code is not changed. One special case: `WorkerExecutionKeys` chains `.configure(ORDER_MAP_ENTRIES_BY_KEYS, true)` after the factory call.

**Tech Stack:** Java 21, Jackson 2.x, `jackson-datatype-jsr310` (already on classpath via `api` module)

## Global Constraints

- Fix is in **casehub-engine** repo, not devtown
- `jackson-datatype-jsr310` must be an explicit dependency in `common-core/pom.xml`
- Test classes keep their existing `new ObjectMapper()` — only production code changes
- `ExchangeSerializer` 2-arg constructor is untouched; only the 1-arg fallback changes
- Build must pass: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install` in engine root

---

## Batch 1: Factory + dependency + unit test

### Task 1: Create EngineMappers utility with unit test

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/utils/EngineMappers.java`
- Create: `common-core/src/test/java/io/casehub/engine/common/internal/utils/EngineMappersTest.java`
- Modify: `common-core/pom.xml` (add explicit `jackson-datatype-jsr310` dependency)

**Interfaces:**
- Produces: `EngineMappers.defaultMapper()` returning `ObjectMapper` — used by all subsequent tasks

- [ ] **Step 1: Add `jackson-datatype-jsr310` dependency to `common-core/pom.xml`**

Add under `<dependencies>`:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

Version is managed by the parent BOM — no `<version>` tag needed.

- [ ] **Step 2: Write the failing test**

```java
package io.casehub.engine.common.internal.utils;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.time.Instant;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertNotSame;

class EngineMappersTest {

    @Test
    void defaultMapperSerializesInstant() throws JsonProcessingException {
        ObjectMapper mapper = EngineMappers.defaultMapper();
        Instant now = Instant.parse("2026-10-25T12:00:00Z");

        String json = mapper.writeValueAsString(now);
        Instant roundTripped = mapper.readValue(json, Instant.class);

        assertEquals(now, roundTripped);
    }

    @Test
    void defaultMapperReturnsNewInstanceEachCall() {
        ObjectMapper a = EngineMappers.defaultMapper();
        ObjectMapper b = EngineMappers.defaultMapper();
        assertNotSame(a, b);
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl common-core -Dtest=EngineMappersTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: compilation failure — `EngineMappers` does not exist yet.

- [ ] **Step 4: Write minimal implementation**

```java
package io.casehub.engine.common.internal.utils;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

public final class EngineMappers {

    private EngineMappers() {}

    public static ObjectMapper defaultMapper() {
        return new ObjectMapper().registerModule(new JavaTimeModule());
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl common-core -Dtest=EngineMappersTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: 2 tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common-core/pom.xml common-core/src/main/java/io/casehub/engine/common/internal/utils/EngineMappers.java common-core/src/test/java/io/casehub/engine/common/internal/utils/EngineMappersTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#262): add EngineMappers utility with JavaTimeModule

Refs casehubio/devtown#262"
```

---

## Batch 2: Replace all bare ObjectMapper instances in common-core

### Task 2: Replace `new ObjectMapper()` in common-core production classes

**Files (modify all — replace `new ObjectMapper()` with `EngineMappers.defaultMapper()` and add import):**
- `common-core/src/main/java/io/casehub/engine/common/internal/channel/DualWriteProjection.java:25`
- `common-core/src/main/java/io/casehub/engine/common/internal/channel/FullProjection.java:26`
- `common-core/src/main/java/io/casehub/engine/common/internal/channel/CustomJqProjection.java:29`
- `common-core/src/main/java/io/casehub/engine/common/internal/channel/ExchangeSerializer.java:32` (1-arg constructor only)
- `common-core/src/main/java/io/casehub/engine/common/internal/context/BridgeResolver.java:33`
- `common-core/src/main/java/io/casehub/engine/common/internal/utils/WorkerExecutionKeys.java:29-30`
- `common-core/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java:46`
- `common-core/src/main/java/io/casehub/engine/common/internal/executor/ScheduledTriggerOrchestrator.java:53`
- `common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java:63`
- `common-core/src/main/java/io/casehub/engine/common/internal/jq/JQEvaluator.java:46`

Also update `CommonBeans.java`:
- `common/src/main/java/io/casehub/engine/common/quarkus/CommonBeans.java:173`

**Interfaces:**
- Consumes: `EngineMappers.defaultMapper()` from Task 1

- [ ] **Step 1: Replace in each file**

For each file, the change is mechanical:

1. Add import: `import io.casehub.engine.common.internal.utils.EngineMappers;`
2. Replace `new ObjectMapper()` with `EngineMappers.defaultMapper()`

**Special case — `WorkerExecutionKeys.java`:**
```java
// Before:
private static final ObjectMapper OBJECT_MAPPER =
    new ObjectMapper().configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, true);
// After:
private static final ObjectMapper OBJECT_MAPPER =
    EngineMappers.defaultMapper().configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, true);
```

**Special case — `ExchangeSerializer.java`:**
```java
// Before (1-arg constructor):
this(bridgeResolver, new ObjectMapper());
// After:
this(bridgeResolver, EngineMappers.defaultMapper());
```
The 2-arg constructor is unchanged.

**Special case — `CommonBeans.java`:**
```java
// Before:
bridgeResolver, new com.fasterxml.jackson.databind.ObjectMapper());
// After:
bridgeResolver, EngineMappers.defaultMapper());
```
Add import for `EngineMappers`.

- [ ] **Step 2: Run common-core and common tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl common-core,common -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all tests PASS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common-core/ common/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#262): replace bare ObjectMapper in common-core with EngineMappers

Refs casehubio/devtown#262"
```

---

## Batch 3: Replace all bare ObjectMapper instances in runtime-core

### Task 3: Replace `new ObjectMapper()` in runtime-core production classes

**Files (modify all — same mechanical replacement):**
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/WorkerScheduleEventHandler.java:65`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/WorkflowExecutionCompletedHandler.java:83`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/MilestoneCompletedEventHandler.java:46`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java:102`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/ContextSignalEventHandler.java:32`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/GoalReachedEventHandler.java:48`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/MilestoneActivatedEventHandler.java:51`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/MilestoneSLAViolatedEventHandler.java:37`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/ActionGateApprovedHandler.java:41`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/ScopedWorkerOutputHandler.java:41`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseStatusChangedHandler.java:56`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/JudgmentCompletedHandler.java:45`
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/JudgmentEscalationHandler.java:41`
- `runtime-core/src/main/java/io/casehub/engine/runtime/work/CaseResumptionService.java:34`
- `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementOutcomeRecorder.java:29`
- `runtime-core/src/main/java/io/casehub/engine/runtime/routing/LlmGoalRevisionStrategy.java:38`
- `runtime-core/src/main/java/io/casehub/engine/runtime/routing/LlmGoalFormationStrategy.java:38`
- `runtime-core/src/main/java/io/casehub/engine/runtime/diff/TopLevelContextDiffStrategy.java:44`
- `runtime-core/src/main/java/io/casehub/engine/runtime/convergence/OutputConvergenceMonitor.java:38`
- `runtime-core/src/main/java/io/casehub/engine/runtime/scheduler/SchedulerService.java:46`
- `runtime-core/src/main/java/io/casehub/engine/runtime/memory/SnapshotCaseContext.java:43`
- `runtime-core/src/main/java/io/casehub/engine/runtime/context/WritableLayerImpl.java:41`

**Interfaces:**
- Consumes: `EngineMappers.defaultMapper()` from Task 1

- [ ] **Step 1: Replace in each file**

For each file:
1. Add import: `import io.casehub.engine.common.internal.utils.EngineMappers;`
2. Replace `new ObjectMapper()` with `EngineMappers.defaultMapper()`

All 22 are straightforward — none have special `.configure()` chains.

- [ ] **Step 2: Run runtime-core tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime-core -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all tests PASS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime-core/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#262): replace bare ObjectMapper in runtime-core with EngineMappers

Refs casehubio/devtown#262"
```

---

## Batch 4: Replace in planning-core + annotations + full build

### Task 4: Replace `new ObjectMapper()` in planning-core and annotations

**Files (modify all):**
- `planning-core/src/main/java/io/casehub/engine/planning/adaptation/LlmRepairStrategy.java:35`
- `planning-core/src/main/java/io/casehub/engine/planning/subcase/SubCaseCompletionService.java:66`
- `planning-core/src/main/java/io/casehub/engine/planning/adaptation/DeeperDecompositionHandler.java:58`
- `planning-core/src/main/java/io/casehub/engine/planning/adaptation/ForwardReplanRevision.java:36`
- `planning-core/src/main/java/io/casehub/engine/planning/completion/PlanItemCompletionApplier.java:50`
- `planning-core/src/main/java/io/casehub/engine/planning/subcase/SubCaseExecutionHandler.java:46`
- `planning-core/src/main/java/io/casehub/engine/planning/decomposition/LlmDecompositionStrategy.java:41`
- `planning-core/src/main/java/io/casehub/engine/planning/adaptation/DefaultPlanAdaptationEvaluator.java:66`
- `planning-core/src/main/java/io/casehub/engine/planning/decomposition/DefaultGoalDecomposer.java:56`
- `annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/AnnotationWorkerFunction.java:28`

**Interfaces:**
- Consumes: `EngineMappers.defaultMapper()` from Task 1

- [ ] **Step 1: Replace in each file**

Same mechanical replacement: add import, replace `new ObjectMapper()` with `EngineMappers.defaultMapper()`.

- [ ] **Step 2: Run full engine build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS — all modules compile, all tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add planning-core/ annotations/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#262): replace bare ObjectMapper in planning-core and annotations with EngineMappers

Closes casehubio/devtown#262"
```

- [ ] **Step 4: Rebuild devtown to pick up fixed engine SNAPSHOT**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS — devtown compiles against the updated engine SNAPSHOT.

## References

- [2026-10-25-engine-objectmapper-factory-design.md] — design spec this plan implements
- [runtime-core/.../WorkerScheduleEventHandler.java:65] — the bug trigger
- [api/.../RetrievedMemory.java:26] — `Instant createdAt` field
- [common-core/.../WorkerExecutionKeys.java:29-30] — special case with `.configure()` chain
- [common/.../CommonBeans.java:173] — CDI wiring that passes bare ObjectMapper
- [GitHub casehubio/devtown#262] — focal issue
- [GitHub casehubio/devtown#258] — upstream e2e test that surfaced this
