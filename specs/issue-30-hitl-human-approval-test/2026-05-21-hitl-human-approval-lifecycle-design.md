# Design: HITL Human Approval Lifecycle Test

**Issue:** devtown#30  
**Branch:** issue-30-hitl-human-approval-test  
**Date:** 2026-05-21

## Problem

The `human-approval` binding in `pr-review.yaml` uses `humanTask:` to create a
casehub-work WorkItem. `casehub-engine-work-adapter` (wired in devtown#33) bridges
WorkItem lifecycle events back to the engine, resuming the case from WAITING state
when a human completes the WorkItem. This round-trip has no end-to-end integration
test. Until one exists, the wiring is unverified below unit level.

## What the test covers

1. A PR review case with `linesChanged > humanApprovalThreshold` causes the
   `human-approval` binding to fire and create a `WorkItem`.
2. The WorkItem carries the correct `callerRef` (`case:{id}/pi:{planItemId}`) and
   title (`PR approval required`).
3. Completing the WorkItem via `WorkItemService.completeFromSystem()` with resolution
   `{"decision": "approved"}` triggers `WorkItemLifecycleAdapter`, which applies the
   `outputMapping` `{ humanApproval: { status: .decision } }` and fires `CONTEXT_CHANGED`.
4. The case context is updated: `humanApproval.status == "approved"`.
5. With all other conditions already satisfied, the `merge` binding fires (its PlanItem
   transitions to `RUNNING`).

## Test class

**`HumanApprovalLifecycleTest`** — `app/src/test/java/io/casehub/devtown/app/`

`@QuarkusTest`. No `@TestHTTPEndpoint` — exercises the internal service and engine
boundary directly, not the REST layer.

### Injections

```java
@Inject PrReviewCaseHub    caseHub;
@Inject WorkItemStore      workItemStore;
@Inject WorkItemService    workItemService;
@Inject BlackboardRegistry blackboardRegistry;
```

### Initial context strategy

All parallel checks are pre-seeded in the initial context so only `human-approval`
fires on the first engine evaluation. This avoids needing capability workers registered
in the test runtime:

| Key                | Value                                                          | Effect                             |
|--------------------|----------------------------------------------------------------|------------------------------------|
| `pr.linesChanged`  | 600                                                            | Exceeds threshold of 500           |
| `policy.humanApprovalThreshold` | 500                                           | Sets the threshold                 |
| `codeAnalysis`     | `{complete: true, securitySensitive: false, architectureCrossing: false}` | Suppresses initial-analysis, security-review, architecture-review |
| `styleCheck`       | `{outcome: "APPROVED"}`                                        | Suppresses style-check binding     |
| `testCoverage`     | `{outcome: "APPROVED"}`                                        | Suppresses test-coverage binding   |
| `performanceAnalysis` | `{outcome: "APPROVED"}`                                     | Suppresses performance-analysis binding |
| `ci`               | `{status: "passing"}`                                          | Suppresses run-ci binding          |

With this context, only `human-approval` evaluates true. `merge` evaluates false
(no `humanApproval` yet).

### Single test method: `humanApproval_fullLifecycle`

Five sequential async checkpoints via Awaitility `untilAsserted`:

**Checkpoint 1 — start case**
```
UUID caseId = caseHub.startCase(initialContext).toCompletableFuture().get(5, SECONDS);
```
`startCase()` resolves after the case is persisted. Event bus delivery (binding
evaluation, WorkItem creation) is async — hence the subsequent polls.

**Checkpoint 2 — WorkItem created**
```
await(5s).untilAsserted:
  workItemStore.scanAll() has an item whose callerRef contains caseId.toString()
  wi.title == "PR approval required"
```
`HumanTaskScheduleHandler` (blocking = true ✅) creates the WorkItem on the event
bus worker thread. The `callerRef` format is `case:{caseId}/pi:{planItemId}`.

**Checkpoint 3 — complete the WorkItem**
```
workItemService.completeFromSystem(wi.id, "system", "{\"decision\": \"approved\"}")
```
Fires `WorkItemLifecycleEvent` synchronously and asynchronously. The async observer
(`WorkItemLifecycleAdapter`) applies the `outputMapping` and fires `CONTEXT_CHANGED`.

**Checkpoint 4 — context updated**
```
await(5s).untilAsserted:
  caseHub.query(caseId, "humanApproval.status").toCompletableFuture().get(1s) == "approved"
```
`WorkItemLifecycleAdapter.applyOutputMapping()` applies JQ `{ humanApproval: { status: .decision } }`
to the resolution JSON. Two async hops from completion to context update (lifecycle
observer → CONTEXT_CHANGED → binding re-evaluation).

**Checkpoint 5 — merge binding fires**
```
await(5s).untilAsserted:
  blackboardRegistry.get(caseId)
    .flatMap(plan -> plan.getPlanItemByBindingName("merge"))
    .map(PlanItem::getStatus)
    .orElseThrow() == PlanItemStatus.RUNNING
```
With `humanApproval.status == "approved"` and all other conditions pre-seeded, the
merge binding evaluates true and the PlanItem transitions to RUNNING.

## Dependency

Add to `app/pom.xml` test scope (version from Quarkus BOM):

```xml
<dependency>
  <groupId>org.awaitility</groupId>
  <artifactId>awaitility</artifactId>
  <scope>test</scope>
</dependency>
```

## Key protocols honoured

- **hitl-runtime-assembly**: work-adapter and blackboard already indexed (devtown#33) ✅
- **yaml-humantask-binding-type**: `humanTask:` binding with `outputMapping` already in YAML ✅
- **work-adapter-test-subcase-group-repository**: `MemorySubCaseGroupRepository` in `selected-alternatives` ✅
- **quarkus-test-database**: H2, `test-port=0`, no `@TestTransaction` ✅
- **GE-20260428-a67806**: `@ConsumeEvent(blocking = true)` already on `HumanTaskScheduleHandler` ✅

## Out of scope

- Testing WorkItem rejection or cancellation paths (separate issue if needed)
- Testing the `human-approval` binding when `linesChanged <= threshold` (already
  covered by `PrReviewBindingConditionTest.HumanApproval`)
- HITL test for architecture-review or security-review gates (different binding types,
  different issue)
