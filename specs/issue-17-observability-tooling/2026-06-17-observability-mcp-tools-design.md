# devtown#17 — Observability and Operational MCP Tools

**Epic:** 10 — Observability and operational tooling
**Issue:** casehubio/devtown#17
**Scope:** MCP tools (8 read + 3 write) + W3C PROV-DM export
**Deferred:** Dashboard extensions (UI), OTel spans (incremental), merge queue tooling (queue not started)

---

## 1. Architecture

### Approach

Thin MCP shell over existing foundation services. Each `@Tool` method injects foundation services (`CaseHubRuntime`, `TrustGateService`, `CommitmentStore`, `WorkItemStore`, `LedgerProvExportService`) and composes their outputs into devtown-specific response records. No intermediate service layer — the MCP class IS the composition layer.

Follows the Qhorus `QhorusMcpTools` pattern: `@ApplicationScoped` + `@Tool` annotations + `@WrapBusinessError({IllegalArgumentException.class, IllegalStateException.class})`.

### Module Placement

`app/` module, package `io.casehub.devtown.app.mcp`.

All CDI wiring is already in `app/`. MCP tools are a view layer alongside the REST resources — same tier, different package. `PrReviewCaseTracker` also lives in `app/` because it is `@ApplicationScoped` CDI infrastructure wiring (observes `CaseLifecycleEvent`, maintains an operational read model) — not domain or integration logic.

### New Dependency

`io.quarkiverse.mcp:quarkus-mcp-server-sse` in `app/pom.xml`. SSE transport, same as Qhorus uses.

### Classes

| Class | Purpose |
|-------|---------|
| `DevtownMcpTools` | `@ApplicationScoped`, all `@Tool` methods, response records as inner types |
| `PrReviewCaseTracker` | `@ApplicationScoped`, event-sourced read model of active PR review cases + recent event ring buffer |

### Tenancy Resolution

MCP tools run outside CDI request scope — no `CurrentPrincipal` available. Tools that query tenant-scoped foundation services accept an optional `tenancy_id` `@ToolArg` parameter, defaulting to `TenancyConstants.DEFAULT_TENANT_ID`.

| Needs `tenancy_id` param | Reads from tracker/buffer only |
|--------------------------|-------------------------------|
| `inspect_review` (event log, ledger) | `get_queue_status` |
| `export_prov` (LedgerProvExportService) | `get_recent_events` (ring buffer) |
| `get_prior_decisions` (CaseMemoryStore) | `list_problems` (CommitmentStore) |
| `get_reviewer_health` (EventLogRepository) | `get_system_health` (aggregated) |
| `retry_reviewer` (CaseHubRuntime.signal) | |
| `reroute_review` (CaseHubRuntime) | |
| `force_complete_check` (CaseHubRuntime.signal) | |

The tracker itself captures `tenancyId` from `CaseLifecycleEvent.tenancyId()` at observation time.

---

## 2. PrReviewCaseTracker

Application-tier read model tracking active PR review cases. Temporary solution until engine#523 ships listing methods on the `CaseInstanceRepository` SPI.

### Population

`PrReviewCaseService.review()` captures the case UUID from `caseHub.startCase()` and registers the full `PrPayload` with the tracker. The tracker stores the complete payload — not a subset of fields — so that `reroute_review` can reconstruct a case from the tracker alone without querying the case context.

### Updates — Case-Level Status

Observes `@ObservesAsync CaseLifecycleEvent`. Switches on `event.caseStatus()` for terminal transitions (matching the `MergeDecisionObserver` pattern):

| `event.caseStatus()` | Tracker action |
|----------------------|----------------|
| `"COMPLETED"` | Move to `COMPLETED` terminal state |
| `"FAULTED"` | Move to `FAULTED` terminal state |
| `"CANCELLED"` | Move to `CANCELLED` terminal state |
| `"RUNNING"` | Set `RUNNING` (non-terminal) |
| `"WAITING"` | Set `WAITING` (non-terminal — human task or parallel wait) |

Terminal cases are retained for recent history queries (configurable retention, default 1 hour), then evicted.

### Capability Status — Lazy Resolution

The tracker does NOT maintain per-capability status. `CaseLifecycleEvent` does not carry capability tags or worker IDs — only `caseId`, `tenancyId`, `commandType`, `eventType`, `caseStatus`, `actorId`, `actorRole`, `traceId`.

When an MCP tool needs capability detail for a specific case, it resolves lazily:

```java
CaseHubRuntime.eventLog(caseId, Set.of(
    WORKER_SCHEDULED,
    WORKER_EXECUTION_COMPLETED,
    WORKER_EXECUTION_FAILED,
    WORKER_OUTCOME_DECLINED
))
```

This avoids the `CrossTenantCaseInstanceRepository` tech debt that `MergeDecisionObserver` and `ReviewOutcomeObserver` already carry.

### Stalled Detection

A case is `STALLED` when `Instant.now() - lastEventTimestamp > threshold`. The tracker updates `lastEventTimestamp` on every `CaseLifecycleEvent` for that case. `STALLED` is a derived state computed at query time, not stored.

### Recent Event Ring Buffer

The tracker maintains a bounded `ArrayDeque<TrackedEvent>` (configurable size, default 200, evicts oldest). Every observed `CaseLifecycleEvent` is added to the buffer with the PR metadata from the case registry (if known). `get_recent_events` reads directly from this buffer — no per-case event log queries.

### Restart Recovery

Logs a warning on startup that the tracker is empty. Populates as new cases arrive. The ring buffer also rebuilds from the first post-restart events. When engine#523 ships, replace the tracker with `caseInstanceRepository.findByNamespaceAndName("devtown", "pr-review", tenancyId)`.

### Deletion Plan

Delete `PrReviewCaseTracker` when devtown#80 (production persistence) + engine#523 (listing SPI) both ship. MCP tools then query the repository directly.

### Data Model

```java
record CaseInfo(
    UUID caseId,
    String tenancyId,
    PrPayload payload,
    Instant startedAt,
    Instant lastEventAt,
    CaseTrackingStatus status
)

enum CaseTrackingStatus {
    RUNNING,    // case active — maps from caseStatus "RUNNING"
    WAITING,    // case waiting — maps from caseStatus "WAITING"
    COMPLETED,  // terminal
    FAULTED,    // terminal
    CANCELLED   // terminal
}
// STALLED is derived at query time: RUNNING or WAITING with
// (now - lastEventAt) > threshold — not a stored state.

record TrackedEvent(
    Instant timestamp,
    UUID caseId,
    String repo,
    int prNumber,
    String eventType,   // raw CaseLifecycleEvent.eventType() value
    String caseStatus,  // raw CaseLifecycleEvent.caseStatus() value
    String actorId
)
```

---

## 3. Read Tools

### 3.1 `get_queue_status` — PR review queue overview

| | |
|---|---|
| Gastown equivalent | None (additive) |
| Source | `PrReviewCaseTracker` |
| Returns | Counts by status, list of active PRs with repo, PR number, contributor, current status, time since last event |

### 3.2 `inspect_review` — Full case inspection

| | |
|---|---|
| Gastown equivalent | `gt peek` |
| Params | `case_id` (UUID), `tenancy_id` (optional, default `TenancyConstants.DEFAULT_TENANT_ID`) |
| Source | `CaseHubRuntime.query()` for live context, `CaseHubRuntime.eventLog()` for timeline, `MergeDecisionLedgerEntry` for ledger records, `WorkItemStore` for human tasks, `CommitmentStore` for obligations |
| Returns | PR metadata (`PrPayload`), goal status (per goal: reached/pending), ordered event timeline, active/completed capabilities (resolved lazily from event log), associated work items, ledger entries with chain verification, active commitments |

### 3.3 `get_reviewer_health` — Per-reviewer operational state

| | |
|---|---|
| Gastown equivalent | None (additive — Gastown has no trust) |
| Params | `reviewer_id` (agent instance ID), `tenancy_id` (optional) |
| Source | `CommitmentStore.findOpenByObligor()`, `TrustGateService` for all capability/dimension scores, `EventLogRepository.findByWorkerAndType()` for recent outcomes |
| Returns | Open commitment count with details, trust scores by capability and dimension, decision counts, last N outcomes with capability and result |

### 3.4 `get_recent_events` — Event feed

| | |
|---|---|
| Gastown equivalent | `gt feed` |
| Params | `limit` (default 50), `since` (optional ISO timestamp) |
| Source | `PrReviewCaseTracker` ring buffer — single read, no per-case queries |
| Returns | Chronological event list with case ID, PR reference, event type, case status, actor, timestamp |

### 3.5 `list_problems` — Operational problem surface

| | |
|---|---|
| Gastown equivalent | `gt problems` + `gt stale` |
| Params | `threshold_minutes` (default 60) |
| Source | `CommitmentStore.findAllOpen()` + `findExpiredBefore()`, `PrReviewCaseTracker` for stalled cases (derived from `lastEventAt`), `TrustGateService` for below-threshold agents, ring buffer for recent worker failures (filtered by `eventType` containing `"Failed"`) |
| Returns | Categorised problem list — stalled reviewers, failed workers, SLA-breached work items, below-threshold agents, stalled cases |

### 3.6 `get_system_health` — Operational health check

| | |
|---|---|
| Gastown equivalent | `gt doctor` |
| Source | `PrReviewCaseTracker` for case counts, `CommitmentStore.findAllOpen()` for obligation counts, `TrustExportService.exportAll(0.0)` for fleet trust overview, `WorkItemStore` for pending human tasks |
| Returns | Active case count, reviewer fleet size (distinct obligors), average trust score by capability, open commitment count, pending work item count, SLA compliance rate |

### 3.7 `get_prior_decisions` — Prior review outcomes by code area

| | |
|---|---|
| Gastown equivalent | `gt seance` (basic version) |
| Params | `repo`, `file_path` (or path prefix), `tenancy_id` (optional) |
| Source | `CaseMemoryStore` via `MemoryQuery.forEntities()` |
| Returns | Prior review outcomes for the code area — capabilities exercised, findings raised, trust scores resulted, links to case IDs |
| Full version | devtown#81 — Doltgres-powered time-travel |

**Query construction:** Follows the existing `CaseMemoryRecaller` pattern:

```java
var modules = ModulePathNormalizer.normalize(List.of(filePath));
List<String> entityIds = modules.stream()
    .map(m -> DevtownMemoryDomain.MODULE_PREFIX + repo + "/" + m)
    .limit(MemoryQuery.MAX_ENTITY_IDS)
    .toList();

store.query(
    MemoryQuery.forEntities(entityIds, DevtownMemoryDomain.SOFTWARE_REVIEW, tenantId)
        .withLimit(limit)
        .withOrder(MemoryOrder.CHRONOLOGICAL)
);
```

No `withQuestion()` — entity IDs scope to specific modules. Semantic search adds no value for structured module queries.

---

## 4. Write Tools

Targeted operator actions — what you do when `list_problems` surfaces an issue.

All three are audit-visible — they appear in the event log with the operator's identity.

### 4.1 `retry_reviewer` — Re-dispatch a stalled capability check

| | |
|---|---|
| Params | `case_id` (UUID), `capability` (e.g. `security-review`), `tenancy_id` (optional) |
| Action | Signal the case to clear the stalled capability's context entry (`null`), causing the binding condition to re-evaluate and fire. `TrustWeightedAgentStrategy` picks a different reviewer if the stalled one's trust has degraded. |
| Source | `CaseHubRuntime.signal(caseId, capabilityContextPath, null)` |
| Returns | Confirmation with new binding evaluation status |

**Signal-null semantics:** `WritablePanelImpl.applyAndDiff()` puts `null` into the context map — it does not remove the key. This works because binding conditions use `ctx.get("securityReview") == null`, which returns `true` for both absent keys and keys with null values. The `applyAndDiff` method detects the value change (previous non-null → null), fires `CaseContextChangedEvent`, and triggers binding re-evaluation.

### 4.2 `reroute_review` — Cancel and restart a stuck PR review

| | |
|---|---|
| Params | `case_id` (UUID), `tenancy_id` (optional) |
| Action | Cancels the current case and starts a fresh one with the same PR payload. Blunt instrument — use `retry_reviewer` for individual capabilities. |
| Source | `CaseHubRuntime.cancelCase()` + `PrReviewCaseHub.startCase()` directly, building initial context from `CaseInfo.payload()` |
| Returns | Old case ID (cancelled) + new case ID |

**Request scope constraint:** The MCP tool does NOT call `PrReviewCaseService.review()`. That method injects `CaseMemoryRecaller` → `CurrentPrincipal`, which requires CDI request scope unavailable in MCP context. Instead, the tool builds the initial context itself (mirroring the context-building logic in `PrReviewCaseService.review()`) and calls `caseHub.startCase(initialContext)` directly. Memory recall is skipped during reroute — an acceptable trade-off for an emergency operator action. The new case is registered with the tracker normally.

### 4.3 `force_complete_check` — Inject a synthetic outcome

| | |
|---|---|
| Params | `case_id` (UUID), `capability`, `outcome` (e.g. `APPROVED`, `FLAGGED`), `reason`, `tenancy_id` (optional) |
| Action | Signals the case with a synthetic result that includes `operatorOverride: true` in the context map. |
| Source | `CaseHubRuntime.signal(caseId, capabilityOutputPath, syntheticResult)` |
| Returns | Confirmation with updated goal status |

**Trust attestation suppression:** The synthetic result includes `operatorOverride: true` in the signalled context value. This matters at two levels:

1. **`ReviewOutcomeObserver` does NOT fire.** It observes `PlanItemCompletedEvent` (fired by the blackboard's `PlanItemCompletionHandler`), not `CaseLifecycleEvent`. A forced signal goes through `SignalReceivedEventHandler` → `CaseContextChangedEvent` → binding re-evaluation. The PlanItem completion path is not triggered — so no `ReviewCompletedEvent` is fired and no trust attestation is generated.

2. **`MergeDecisionObserver` DOES fire** if all goals are met and the case reaches `COMPLETED`. The merge decision ledger entry should carry the `operatorOverride` marker so the audit trail is honest about which checks were operator-forced vs genuine agent outcomes. The observer reads the case context — it can check for and propagate the marker.

---

## 5. PROV-DM Export

### `export_prov` — W3C PROV-DM causal chain

| | |
|---|---|
| Params | `case_id` (UUID), `tenancy_id` (optional) |
| Action | Delegates to `LedgerProvExportService.exportSubject(caseId, tenancyId)` for tamper-evident PROV-DM graph, enriches with engine event log for full causal chain |
| Source | `LedgerProvExportService`, `CaseHubRuntime.eventLog()` |
| Returns | PROV-JSON-LD string — W3C PROV-DM document with `prov:Entity` (PR, review outcomes, merge decision), `prov:Activity` (each capability execution), `prov:Agent` (reviewers, CI runner, merge executor), causal relationships (`wasGeneratedBy`, `used`, `wasAssociatedWith`) |

Output format is PROV-JSON-LD only — `LedgerProvExportService` returns JSON-LD via `LedgerProvSerializer.toProvJsonLd()`. No Turtle serializer exists in the foundation.

---

## 6. Response Records

Java records as inner types of `DevtownMcpTools`, following the Qhorus `QhorusMcpToolsBase` pattern.

```java
record QueueStatus(int total, Map<String, Integer> countsByStatus, List<ActiveReview> reviews)
record ActiveReview(UUID caseId, String repo, int prNumber, String contributor,
                    int linesChanged, String status, Instant startedAt, Instant lastEventAt)

record ReviewDetail(UUID caseId, PrPayload pr, List<GoalStatus> goals,
                    List<EventEntry> timeline, List<LedgerRecord> ledgerEntries,
                    List<CommitmentSummary> commitments, List<CapabilityStatus> capabilities)
record GoalStatus(String name, String kind, boolean reached)
record EventEntry(Instant timestamp, String eventType, String actor, String summary)
record LedgerRecord(UUID entryId, String type, Instant timestamp, boolean chainVerified)
record CommitmentSummary(UUID id, String obligor, String capability, String state, Instant createdAt)
record CapabilityStatus(String name, String status, String outcome, Instant completedAt)

record ReviewerHealth(String reviewerId, int openCommitments,
                      Map<String, Double> trustByCapability, Map<String, Double> trustByDimension,
                      int totalDecisions, List<RecentOutcome> recentOutcomes)
record RecentOutcome(UUID caseId, String capability, String outcome, Instant timestamp)

record Problem(String category, String severity, String description,
               UUID caseId, String actorId, Instant since)

record SystemHealth(int activeCases, int fleetSize,
                    Map<String, Double> avgTrustByCapability,
                    int openCommitments, int pendingWorkItems)

record PriorDecision(UUID caseId, String repo, int prNumber, String capability,
                     String outcome, Instant decidedAt)
```

---

## 7. Gastown Parity Mapping

| Gastown CLI | devtown MCP tool | Coverage |
|-------------|-----------------|----------|
| `gt feed` | `get_recent_events` | ✅ Full |
| `gt problems` | `list_problems` | ✅ Full |
| `gt doctor` | `get_system_health` | ✅ Full |
| `gt seance` | `get_prior_decisions` | ⚠️ Basic — full in devtown#81 |
| `gt stale` | `list_problems` | ✅ Folded in |
| `gt peek` | `inspect_review` | ✅ Full |

Additive beyond Gastown:
- `get_queue_status` — aggregate queue overview
- `get_reviewer_health` — per-reviewer trust/obligation deep dive (Gastown has no trust model)
- `export_prov` — W3C PROV-DM compliance export (Gastown has no compliance story)
- Write tools — `retry_reviewer`, `reroute_review`, `force_complete_check`

---

## 8. Foundation Gaps

| Issue | What | Blocks this epic? |
|-------|------|-------------------|
| devtown#80 | Activate production persistence backend — remove compile-scope `persistence-memory` | No |
| engine#523 | Add listing methods to `CaseInstanceRepository` SPI | No — `PrReviewCaseTracker` is interim |
| devtown#81 | Full `gt seance` equivalent with Doltgres time-travel | No — basic version ships here |

No new foundation work required. All tools compose over existing foundation services.
