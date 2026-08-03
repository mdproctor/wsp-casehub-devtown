# Binding Activation State + Dataset Enhancements — Design Spec

**Issues:** devtown#170, devtown#173
**Branch:** `issue-170-binding-activation-state`
**Date:** 2026-08-03
**Repos:** engine (runtime, common, rest), devtown (app, frontend)

## Problem

**#170 — Activation context is discarded.** When a binding fires in `CaseContextChangedEventHandler.rules()`, the `contextSnapshot` and `changedLayer` are available at evaluation time but neither is persisted. The `WORKER_SCHEDULED` event log entry includes worker/capability/binding metadata but no record of what blackboard state satisfied the binding's filter/when conditions. The plan-items endpoint reflects the same gap. Operators cannot answer "why did this binding fire?" from the audit trail.

**#173 — Dataset infrastructure is unused.** pages-data supports `refreshTime` (periodic re-fetch), `accumulate` (append mode), WebSocket sources (`ws://` prefix), and SSE sources (`sse://` prefix). devtown's `GovernanceEventBridge` already broadcasts CDI events via WebSocket at `/api/governance/events`. All 13 devtown datasets are one-shot REST fetches. The gap is purely adoption — no operational dashboard has near-real-time updates.

## Design

### Part A: Activation Context Capture (engine)

#### A1. WorkerScheduleEvent — add activationContext field

`WorkerScheduleEvent` in `engine-common/internal/event/` gains a `JsonNode activationContext` field. This carries the changed layer content from binding evaluation to the event log handler.

The record currently has: `caseInstance`, `worker`, `capability`, `bindingName`, `inputProjectionOverride`, `signalId`, `origin`, `experiences`, `lifecycleScope`, `executionMode`, `credentialToken`.

Add `activationContext` as the final field. Null when the dispatch is not triggered by a context change (e.g., manual schedule, retry).

#### A2. CaseContextChangedEventHandler — capture and thread

In `rules()`, after a binding passes filter/when evaluation and before dispatch, snapshot the changed layer:

```java
JsonNode activationSnapshot = contextSnapshot.layer(changedLayer) != null
    ? contextSnapshot.layer(changedLayer).asJsonNode()
    : null;
```

Thread through the existing call chain:
- `rules()` → pass `activationSnapshot` to `publishByTarget()`
- `publishByTarget()` → pass to `publishWorkerSchedule()` and `publishHumanTaskSchedule()`
- `publishWorkerSchedule()` → pass to `scheduleWorker()`
- `scheduleWorker()` → include in `WorkerScheduleEvent` constructor

For `publishHumanTaskSchedule()`: the human task path creates plan items via `WorkItemStore`, not via `WorkerScheduleEvent`. The activation context should be included in the `TASK_CREATED` event log entry that `publishHumanTaskSchedule()` writes. Add the same `activationContext` field to its metadata.

#### A3. WorkerScheduleEventHandler — persist in event log

In `buildEventLog()`, add the activation context to the metadata node:

```java
if (activationContext != null && !activationContext.isNull()) {
    metaNode.set("activationContext", activationContext);
}
```

The `activationContext` field name is stable — frontend queries and the plan-items endpoint rely on it.

#### A4. CDI events for SSE streaming

Two new CDI event records in `engine-common/spi/event/`:

**`PlanItemStateChangedEvent`:**
```java
public record PlanItemStateChangedEvent(
    UUID caseId,
    String planItemId,
    String bindingName,
    TaskStatus previousStatus,
    TaskStatus newStatus,
    String tenancyId) {}
```

Published from:
- `PlanItemCompletionHandler` — terminal transitions (COMPLETED, FAULTED, CANCELLED, REJECTED, OBSOLETE)
- `WorkerScheduleEventHandler` — PENDING creation (the plan item is created here)
- Existing status transition points in the planning module

**`CaseContextUpdatedEvent`:**
```java
public record CaseContextUpdatedEvent(
    UUID caseId,
    String changedLayer,
    String tenancyId) {}
```

Published from `CaseHubRuntime` at the same point where `CaseContextChangedEvent` is fired on the Vert.x event bus. Lightweight — carries case ID and layer name only.

### Part B: Extended Plan-Items Endpoint (engine-rest)

#### B1. PlanItemResponse — add activationContext

```java
public record PlanItemResponse(
    String planItemId,
    String bindingName,
    String targetType,
    String status,
    String executorName,
    String description,
    Instant createdAt,
    JsonNode activationContext  // nullable — null for manually created plan items
) {
    public static PlanItemResponse from(PlanItemRecord record) {
        return from(record, null);
    }

    public static PlanItemResponse from(PlanItemRecord record, JsonNode activationContext) {
        return new PlanItemResponse(
            record.planItemId(),
            record.bindingName(),
            record.targetType().name().toLowerCase().replace('_', '-'),
            record.status().name(),
            record.executorName(),
            record.description(),
            record.createdAt(),
            activationContext);
    }
}
```

#### B2. CaseInstanceResource.getPlanItems() — server-side join

After fetching plan items, query the event log for WORKER_SCHEDULED events for this case and correlate by `bindingName`:

```java
List<EventLog> scheduledEvents = eventLogRepository.query(
    EventLogQuery.builder(caseId)
        .eventTypes(Set.of(CaseHubEventType.WORKER_SCHEDULED))
        .build(),
    tenancyId);

// Build a list per binding — repeatable bindings fire multiple times
Map<String, List<EventLog>> eventsByBinding = new HashMap<>();
for (EventLog e : scheduledEvents) {
    JsonNode meta = e.getMetadata();
    if (meta == null) continue;
    String binding = meta.path("bindingName").asText(null);
    if (binding != null && !meta.path("activationContext").isMissingNode()) {
        eventsByBinding.computeIfAbsent(binding, k -> new ArrayList<>()).add(e);
    }
}

return planItems.stream()
    .map(r -> {
        List<EventLog> candidates = eventsByBinding.get(r.bindingName());
        JsonNode activation = candidates == null ? null
            : findClosestByTimestamp(candidates, r.createdAt());
        return PlanItemResponse.from(r, activation);
    })
    .toList();
```

Correlation uses `bindingName` + closest `createdAt` timestamp. Repeatable bindings fire multiple WORKER_SCHEDULED events — each plan item matches the event whose timestamp is closest to its creation time. The `findClosestByTimestamp` method extracts `activationContext` from the event whose timestamp has minimum absolute distance from the plan item's `createdAt`.

#### B3. SSE endpoint — CaseStreamResource

```java
@Path("/api/v1/cases/{caseId}/stream")
@ApplicationScoped
public class CaseStreamResource {

    private final Map<UUID, Set<SseEventSink>> sinks = new ConcurrentHashMap<>();

    @GET
    @Produces(MediaType.SERVER_SENT_EVENTS)
    @RestSseElementType(MediaType.APPLICATION_JSON)
    @RunOnVirtualThread
    public void stream(@PathParam("caseId") UUID caseId,
                        @Context SseEventSink sink,
                        @Context Sse sse) {
        caseService.requireCaseAccess(caseId, AclAction.READ);
        sinks.computeIfAbsent(caseId, k -> ConcurrentHashMap.newKeySet()).add(sink);
        sink.onClose(() -> {
            Set<SseEventSink> set = sinks.get(caseId);
            if (set != null) { set.remove(sink); if (set.isEmpty()) sinks.remove(caseId); }
        });
    }

    void onPlanItemChanged(@ObservesAsync PlanItemStateChangedEvent event) {
        broadcast(event.caseId(), "plan-item", Map.of(
            "planItemId", event.planItemId(),
            "bindingName", event.bindingName(),
            "previousStatus", event.previousStatus().name(),
            "newStatus", event.newStatus().name()));
    }

    void onContextUpdated(@ObservesAsync CaseContextUpdatedEvent event) {
        broadcast(event.caseId(), "context", Map.of(
            "changedLayer", event.changedLayer()));
    }
}
```

Multiplexed — plan item transitions and context changes on one SSE connection per case. SSE `event:` field carries the type; `data:` carries the JSON payload.

### Part C: Dataset Enhancements (devtown)

#### C1. datasets.ts refactor

```typescript
import { bind, restSource } from "@casehubio/pages-ui";
import { wsSource, sseSource } from "@casehubio/pages-data";
import type { DataSetId } from "@casehubio/pages-data";

function rest(id: string, url: string, opts?: { dataPath?: string; expression?: string; refreshTime?: string }) {
  return bind(id, restSource(url, id as DataSetId, opts));
}
function ws(id: string, url: string, opts?: { dataPath?: string; accumulate?: boolean }) {
  return bind(id, wsSource(url, id as DataSetId, opts));
}
function sse(id: string, url: string, opts?: { dataPath?: string; accumulate?: boolean }) {
  return bind(id, sseSource(url, id as DataSetId, opts));
}
```

#### C2. Dataset definitions with enhancements

```typescript
export const datasets = [
  // Operational — 10s refresh
  rest("queue-status", "/api/governance/queue-status", { dataPath: "reviews", refreshTime: "10second" }),
  rest("problems", "/api/governance/problems?threshold_minutes=0", { dataPath: "items", refreshTime: "10second" }),
  rest("merge-queue", "/api/governance/merge-queue", { dataPath: "queuedPrs", refreshTime: "10second" }),
  rest("active-batches", "/api/governance/merge-queue", { dataPath: "activeBatches", refreshTime: "10second" }),
  rest("triage", "/api/governance/triage", { dataPath: "items", refreshTime: "10second" }),

  // Metrics — 30s refresh
  rest("system-health", "/api/governance/system-health", { expression: "[$]", refreshTime: "30second" }),
  rest("merge-queue-metrics", "/api/governance/merge-queue/metrics", { expression: "[$]", refreshTime: "30second" }),
  rest("reviewers", "/api/governance/reviewers", { dataPath: "items", refreshTime: "30second" }),

  // Event stream — WebSocket push with accumulate
  ws("recent-events", "ws:///api/governance/events", { accumulate: true }),

  // Static — no refresh
  rest("case-definitions", "/api/v1/case-definitions"),

  // Per-case — 5s refresh (SSE-triggered refresh deferred to pages-data enhancement)
  rest("plan-items", "/api/v1/cases/#{row.caseId}/plan-items", { refreshTime: "5second" }),
  rest("goal-status", "/api/v1/cases/#{row.caseId}/goals", { refreshTime: "5second" }),
  rest("case-context", "/api/v1/cases/#{row.caseId}/context", { refreshTime: "5second" }),
];
```

**Note on SSE for per-case datasets:** The SSE stream (`/api/v1/cases/{caseId}/stream`) sends lightweight notifications, not full dataset snapshots. pages-data does not currently support "re-fetch on SSE event" — only fixed-interval polling via `refreshTime`. The 5-second poll provides near-real-time visibility. A future pages-data enhancement could connect SSE events to immediate dataset refresh, replacing polling entirely. Filed separately.

#### C3. Platform preferences for refresh intervals

**GovernancePreferencesResource** in devtown-app:

```java
@Path("/api/governance/preferences")
@Produces(MediaType.APPLICATION_JSON)
public class GovernancePreferencesResource {

    @Inject PreferenceProvider preferenceProvider;

    @GET
    public Map<String, String> getPreferences() {
        return Map.of(
            "refresh.operational", resolve("devtown.dashboard.refresh.operational", "10second"),
            "refresh.metrics",    resolve("devtown.dashboard.refresh.metrics", "30second"),
            "refresh.caseDetail", resolve("devtown.dashboard.refresh.case-detail", "5second"),
            "events.mode",        resolve("devtown.dashboard.events.mode", "websocket"));
    }

    private String resolve(String path, String defaultValue) {
        return preferenceProvider.get(Path.of(path))
            .map(Preference::value)
            .orElse(defaultValue);
    }
}
```

**Frontend integration:** `index.ts` fetches preferences at load time, before constructing datasets. The `refreshTime` values in C2 are replaced with values from the preferences response. If the preferences endpoint is unavailable (e.g., dev mode without platform), defaults apply.

#### C4. reviews.ts — activation context column

Add `activationContext` to the plan items table:

```typescript
title("Plan Items", "h3"),
dataTable({
  lookup: lookup("plan-items", groupBy("planItemId",
    col("bindingName"), col("targetType"),
    col("status"), col("executorName"), col("createdAt"),
    col("activationContext"),
  )),
  sortable: true,
}),
```

`activationContext` renders as a JSON key-value display in the table cell. The changed layer content is typically compact — a handful of keys like `{ "pr.analysis.security.critical": 3, "pr.analysis.security.confidence": 0.92 }`.

## Implementation Order

```
Phase 1 — Engine runtime (no dependencies)
  1. Add activationContext to WorkerScheduleEvent record
  2. Thread activationContext through CaseContextChangedEventHandler call chain
  3. Persist activationContext in WorkerScheduleEventHandler.buildEventLog() metadata
  4. Add PlanItemStateChangedEvent and CaseContextUpdatedEvent CDI events
  5. Publish CDI events from PlanItemCompletionHandler, WorkerScheduleEventHandler, CaseHubRuntime
  6. Tests for all of the above

Phase 2 — Engine REST (depends on Phase 1)
  1. Extend PlanItemResponse with activationContext field
  2. Server-side join in CaseInstanceResource.getPlanItems()
  3. CaseStreamResource SSE endpoint
  4. Tests for plan-items enrichment and SSE streaming

Phase 3 — Devtown backend (parallel with Phase 2)
  1. GovernancePreferencesResource
  2. Tests for preferences endpoint

Phase 4 — Devtown frontend (depends on Phase 2 + 3)
  1. datasets.ts refactor — add ws(), sse() helpers
  2. Apply refreshTime to operational/metrics/case-detail datasets
  3. Convert recent-events to WebSocket source
  4. Add activationContext column to plan items table
  5. Wire preferences to dataset construction
  6. TypeScript type-check + build
  7. Visual verification in dev mode
```

## Testing Strategy

**Engine runtime:**
- Unit: `WorkerScheduleEventHandler.buildEventLog()` includes `activationContext` in metadata
- Unit: `PlanItemStateChangedEvent` published on terminal transitions
- Unit: `CaseContextUpdatedEvent` published on context signal
- Integration (`@QuarkusTest`): start case → signal context → verify WORKER_SCHEDULED event has activationContext containing changed layer content

**Engine REST:**
- Unit: `PlanItemResponse.from(record, activationContext)` maps correctly; `from(record)` returns null activation
- Integration: start case → trigger binding → verify plan-items endpoint returns activationContext per item
- Integration: SSE — connect to `/api/v1/cases/{caseId}/stream`, trigger plan item transition, verify SSE event; uses `reconnectingEvery(Long.MAX_VALUE, MILLISECONDS)` (GE-20260617-0c1498); filters comment-only frames (GE-20260617-cb0731)

**Devtown backend:**
- Integration: `GET /api/governance/preferences` returns defaults
- Integration: preference override applies to response

**Devtown frontend:**
- `npm run typecheck` passes
- `npm run build` passes
- Visual: activation context visible in plan items table
- Visual: operational datasets refresh at configured intervals
- Visual: recent-events streams via WebSocket with accumulation
- Visual: preferences control refresh rates

## Garden Entries Referenced

| GE ID | Relevance |
|-------|-----------|
| GE-20260526-fa0e3e | `getPlanItemByBindingName()` excludes terminal PlanItems — affects activation context correlation |
| GE-20260607-245588 | Event fan-out overload — activation context capture must not introduce similar fan-out issues |
| GE-20260617-0c1498 | SSE reconnect disable for integration tests |
| GE-20260617-cb0731 | RESTEasy SseEventSource fires on comment-only frames |

## Related Issues

| Repo | # | Relationship |
|------|---|-------------|
| devtown | 170 | Primary — binding activation state |
| devtown | 173 | Primary — dataset enhancements |
| devtown | 119 | Parent — CasePlanModel browser (activation state was deferred from here) |
| devtown | 167 | Related — WebSocket/SSE live updates (this spec partially addresses) |
| devtown | 172 | Prerequisite — pages upgrade (completed, base for dataset refactor) |
| pages | — | Future — SSE-triggered dataset refresh (pages-data enhancement, not filed yet) |

## Out of Scope

- SSE-triggered immediate dataset refresh (pages-data feature, not available — using refreshTime polling)
- Full context snapshot as activation context (changed layer is sufficient and bounded)
- Condition-referenced key extraction from JQ expressions (complex, diminishing returns over changed layer)
- Per-case SSE topic filtering in pages-data (SSE endpoint multiplexes, frontend receives all events for subscribed case)
- Preference persistence UI (preferences are set via API or platform admin, not governance dashboard)
