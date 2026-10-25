# Fix Jackson Instant Serialization — Engine ObjectMapper Factory

**Issue:** casehubio/devtown#262
**Date:** 2026-10-25
**Status:** draft

## Problem

`WorkerScheduleEventHandler.buildEventLog()` serializes `RetrievedMemory` objects
(which contain `Instant createdAt`) using a static `new ObjectMapper()` that lacks
`JavaTimeModule`. This causes `IllegalArgumentException` when the engine retries
(REROUTE) a failed worker.

The same pattern — `private static final ObjectMapper MAPPER = new ObjectMapper()` —
exists in 30+ production classes across `runtime-core`, `common-core`, and
`planning-core`. Any of these will fail if they ever serialize Java 8 date/time types.

## Solution

Create `EngineMappers` utility in `common-core` with a `defaultMapper()` factory
that returns an `ObjectMapper` pre-configured with `JavaTimeModule`. Replace all
bare `new ObjectMapper()` in engine production code with `EngineMappers.defaultMapper()`.

### EngineMappers class

Location: `common-core/src/main/java/io/casehub/engine/common/internal/utils/EngineMappers.java`

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

Each call creates a new instance because each call site stores it in a `static final`
field — the factory runs once per class load. No shared mutable state.

### Dependency change

Add `jackson-datatype-jsr310` to `common-core/pom.xml`. It's already a dependency
of the `api` module (transitive), but making it explicit in `common-core` ensures
the factory compiles without relying on transitivity.

### Affected files (production)

**common-core** (~10 classes):
- `DualWriteProjection`, `FullProjection`, `CustomJqProjection`
- `ExchangeSerializer` (constructor `new ObjectMapper()` fallback)
- `BridgeResolver`
- `WorkerExecutionKeys` (preserves `.configure(ORDER_MAP_ENTRIES_BY_KEYS, true)`)
- `RetryOrchestrator`, `ScheduledTriggerOrchestrator`, `WorkerExecutionOrchestrator`
- `JQEvaluator`, `WritableLayerImpl`, `SnapshotCaseContext`

**runtime-core** (22 classes):
- `WorkerScheduleEventHandler` (the bug trigger)
- `WorkflowExecutionCompletedHandler`, `MilestoneCompletedEventHandler`
- `CaseContextChangedEventHandler`, `ContextSignalEventHandler`
- `GoalReachedEventHandler`, `MilestoneActivatedEventHandler`
- `MilestoneSLAViolatedEventHandler`, `ActionGateApprovedHandler`
- `ScopedWorkerOutputHandler`, `CaseStatusChangedHandler`
- `CaseResumptionService`, `ImprovementOutcomeRecorder`
- `LlmGoalRevisionStrategy`, `LlmGoalFormationStrategy`
- `TopLevelContextDiffStrategy`, `OutputConvergenceMonitor`
- `SchedulerService`, `JudgmentCompletedHandler`, `JudgmentEscalationHandler`

**planning-core** (~9 classes):
- `LlmRepairStrategy`, `SubCaseCompletionService`
- `DeeperDecompositionHandler`, `ForwardReplanRevision`
- `PlanItemCompletionApplier`, `SubCaseExecutionHandler`
- `LlmDecompositionStrategy`, `DefaultPlanAdaptationEvaluator`
- `DefaultGoalDecomposer`

**annotations/runtime** (1 class):
- `AnnotationWorkerFunction`

### Special cases

- `WorkerExecutionKeys`: uses `.configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, true)`.
  Keep the configure call chained after `EngineMappers.defaultMapper()`.
- `ExchangeSerializer`: has a 2-arg constructor taking `ObjectMapper`. Only update
  the 1-arg fallback constructor that calls `new ObjectMapper()`.
- `CommonBeans`: the `new ObjectMapper()` passed to `ExchangeSerializer` — update to
  use `EngineMappers.defaultMapper()`.

### Test classes

Test classes that use `new ObjectMapper()` are not changed — they don't serialize
`Instant` and the test ObjectMapper intentionally mirrors the pre-fix behavior
(bare mapper). If a test needs `JavaTimeModule`, it can add it explicitly.

## Testing

1. **Unit test for EngineMappers**: verify `defaultMapper()` serializes/deserializes
   `Instant` correctly (round-trip test).
2. **Existing tests pass**: `mvn clean install` in casehub-engine must remain green.
3. **Devtown e2e**: after the engine fix is installed, the simulated review pipeline
   e2e test (devtown#258) should be able to exercise the case-plan-engine path
   without the `Instant` serialization error.

## Cross-Repo

This fix is in **casehub-engine**, not devtown. After the fix:
1. Commit and push to casehub-engine
2. `mvn install` in engine to publish SNAPSHOT locally
3. Rebuild devtown to pick up the fixed SNAPSHOT

## References

- `runtime-core/.../WorkerScheduleEventHandler.java:65` — the bug trigger
- `api/.../RetrievedMemory.java:26` — `Instant createdAt` field
- `api/pom.xml` — `jackson-datatype-jsr310` already declared
- casehubio/devtown#258 — the e2e test that surfaced this issue
