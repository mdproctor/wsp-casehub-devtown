# devtown#17 — Observability and Operational MCP Tools

**Epic:** 10 — Observability and operational tooling
**Issue:** casehubio/devtown#17
**Scope:** MCP tools (8 read + 3 write) + W3C PROV-DM export
**Deferred:** Dashboard extensions (UI), OTel spans (incremental), merge queue tooling (queue not started)

---

## 1. Architecture

### Approach

Thin MCP shell over existing foundation services. Each `@Tool` method injects foundation services (`CaseHubRuntime`, `TrustGateService`, `CommitmentStore`, `WorkItemStore`, `LedgerProvExportService`) and composes their outputs into devtown-specific response records. No intermediate service layer — the MCP class IS the composition layer.

Follows the Qhorus `QhorusMcpTools` pattern: `@ApplicationScoped` + `@Tool` annotations + `@WrapBusinessError`.

### Module Placement

`app/` module, package `io.casehub.devtown.app.mcp`.

All CDI wiring is already in `app/`. MCP tools are another view layer alongside the REST resources — same module, different package.

### New Dependency

`io.quarkiverse.mcp:quarkus-mcp-server-sse` in `app/pom.xml`. SSE transport, same as Qhorus uses.

### Classes

| Class | Purpose |
|-------|---------|
| `DevtownMcpTools` | `@ApplicationScoped`, all `@Tool` methods, response records as inner types |
| `PrReviewCaseTracker` | `@ApplicationScoped`, event-sourced read model of active PR review cases |

---

## 2. PrReviewCaseTracker

Application-tier read model tracking active PR review cases. Temporary solution until engine#523 ships listing methods on the `CaseInstanceRepository` SPI.

### Population

`PrReviewCaseService.review()` captures the case UUID from `caseHub.startCase()` and registers it with the tracker alongside PR metadata (repo, PR number, contributor, lines changed).

### Updates

Observes `@ObservesAsync CaseLifecycleEvent`:
- `CASE_COMPLETED` / `CASE_FAULTED` / `CASE_CANCELLED` → moves to terminal state (retained for recent history)
- `WORKER_SCHEDULED` / `WORKER_EXECUTION_COMPLETED` / `WORKER_EXECUTION_FAILED` → updates capability status within the case entry

### Restart Recovery

Logs a warning on startup that the tracker is empty. Populates as new cases arrive. When engine#523 ships, replace the tracker with `caseInstanceRepository.findByNamespaceAndName("devtown", "pr-review", tenancyId)`.

### Deletion Plan

Delete `PrReviewCaseTracker` when devtown#80 (production persistence) + engine#523 (listing SPI) both ship. MCP tools then query the repository directly.

### Data Model

```java
record CaseInfo(
    UUID caseId,
    String repo,
    int prNumber,
    String contributor,
    Instant startedAt,
    ReviewStatus currentStatus,
    Set<String> activeCapabilities,
    Set<String> completedCapabilities
)

enum ReviewStatus {
    ANALYSING,      // initial code analysis running
    IN_REVIEW,      // content-driven capability bindings fired
    AWAITING_HUMAN, // human task binding fired
    BLOCKED,        // no progress past threshold
    MERGING,        // merge executor scheduled
    COMPLETED,      // terminal — case completed
    FAULTED,        // terminal — case faulted
    CANCELLED       // terminal — case cancelled
}
```

---

## 3. Read Tools

### 3.1 `get_queue_status` — PR review queue overview

| | |
|---|---|
| Gastown equivalent | None (additive) |
| Source | `PrReviewCaseTracker` |
| Returns | Counts by status, list of active PRs with repo, PR number, contributor, current status, active capabilities |

### 3.2 `inspect_review` — Full case inspection

| | |
|---|---|
| Gastown equivalent | `gt peek` |
| Params | `case_id` (UUID) |
| Source | `CaseHubRuntime.query()` for live context, `CaseHubRuntime.eventLog()` for timeline, `MergeDecisionLedgerEntry` for ledger records, `WorkItemStore` for human tasks, `CommitmentStore` for obligations |
| Returns | PR metadata, goal status (per goal: reached/pending), ordered event timeline, active/completed capabilities, associated work items, ledger entries with chain verification, active commitments |

### 3.3 `get_reviewer_health` — Per-reviewer operational state

| | |
|---|---|
| Gastown equivalent | None (additive — Gastown has no trust) |
| Params | `reviewer_id` (agent instance ID) |
| Source | `CommitmentStore.findOpenByObligor()`, `TrustGateService` for all capability/dimension scores, `EventLogRepository` filtered to `WORKER_EXECUTION_COMPLETED`/`WORKER_EXECUTION_FAILED` |
| Returns | Open commitment count with details, trust scores by capability and dimension, decision counts, last N outcomes with capability and result |

### 3.4 `get_recent_events` — Event feed

| | |
|---|---|
| Gastown equivalent | `gt feed` |
| Params | `limit` (default 50), `since` (optional ISO timestamp) |
| Source | `PrReviewCaseTracker` for case IDs, `CaseHubRuntime.eventLog()` per case, merged and sorted by timestamp |
| Returns | Chronological event list with case ID, PR reference, event type, actor, timestamp, payload summary |

### 3.5 `list_problems` — Operational problem surface

| | |
|---|---|
| Gastown equivalent | `gt problems` + `gt stale` |
| Params | `threshold_minutes` (default 60) |
| Source | `CommitmentStore.findAllOpen()` + `findExpiredBefore()`, `PrReviewCaseTracker` for stuck cases, `TrustGateService` for below-threshold agents, event log for failed workers |
| Returns | Categorised problem list — stalled reviewers, failed workers, SLA-breached work items, below-threshold agents, cases with no progress past threshold |

### 3.6 `get_system_health` — Operational health check

| | |
|---|---|
| Gastown equivalent | `gt doctor` |
| Source | `PrReviewCaseTracker` for case counts, `CommitmentStore.findAllOpen()` for obligation counts, `TrustExportService.exportAll()` for fleet trust overview, `WorkItemStore` for pending human tasks |
| Returns | Active case count, reviewer fleet size (distinct obligors), average trust score by capability, open commitment count, pending work item count, SLA compliance rate |

### 3.7 `get_prior_decisions` — Prior review outcomes by code area

| | |
|---|---|
| Gastown equivalent | `gt seance` (basic version) |
| Params | `repo`, `file_path` (or path prefix) |
| Source | `CaseMemoryStore` (devtown memory domain), ledger entries filtered by code area metadata |
| Returns | Prior review outcomes for the code area — capabilities exercised, findings raised, trust scores resulted, links to case IDs |
| Full version | devtown#81 — Doltgres-powered time-travel |

---

## 4. Write Tools

Targeted operator actions — what you do when `list_problems` surfaces an issue.

All three are audit-visible — they appear in the event log and (where applicable) as ledger entries with the operator's identity.

### 4.1 `retry_reviewer` — Re-dispatch a stalled capability check

| | |
|---|---|
| Params | `case_id` (UUID), `capability` (e.g. `security-review`) |
| Action | Signal the case to clear the stalled capability's context entry (`null`), causing the binding condition to re-evaluate and fire. `TrustWeightedAgentStrategy` picks a different reviewer if the stalled one's trust has degraded. |
| Source | `CaseHubRuntime.signal(caseId, capabilityContextPath, null)` |
| Returns | Confirmation with new binding evaluation status |

### 4.2 `reroute_review` — Cancel and restart a stuck PR review

| | |
|---|---|
| Params | `case_id` (UUID) |
| Action | Cancels the current case and starts a fresh one with the same PR payload. Blunt instrument — use `retry_reviewer` for individual capabilities. |
| Source | `CaseHubRuntime.cancelCase()` + `PrReviewCaseService.review()` with original `PrPayload` |
| Returns | Old case ID (cancelled) + new case ID |

### 4.3 `force_complete_check` — Inject a synthetic outcome

| | |
|---|---|
| Params | `case_id` (UUID), `capability`, `outcome` (e.g. `APPROVED`, `FLAGGED`), `reason` |
| Action | Signals the case with a synthetic result. Recorded with `operator-override` actor ID. Does NOT generate a trust attestation — forced outcomes must not pollute trust scores. |
| Source | `CaseHubRuntime.signal(caseId, capabilityOutputPath, syntheticResult)` |
| Returns | Confirmation with updated goal status |

---

## 5. PROV-DM Export

### `export_prov` — W3C PROV-DM causal chain

| | |
|---|---|
| Params | `case_id` (UUID), `format` (default `json`, also `turtle`) |
| Action | Delegates to `LedgerProvExportService` for tamper-evident PROV-DM graph, enriches with engine event log for full causal chain |
| Source | `LedgerProvExportService`, `CaseHubRuntime.eventLog()` |
| Returns | W3C PROV-DM document with `prov:Entity` (PR, review outcomes, merge decision), `prov:Activity` (each capability execution), `prov:Agent` (reviewers, CI runner, merge executor), causal relationships (`wasGeneratedBy`, `used`, `wasAssociatedWith`) |

---

## 6. Response Records

Java records as inner types of `DevtownMcpTools`, following the Qhorus `QhorusMcpToolsBase` pattern.

```java
record QueueStatus(int total, Map<String, Integer> countsByStatus, List<ActiveReview> reviews)
record ActiveReview(UUID caseId, String repo, int prNumber, String contributor,
                    String status, Set<String> activeCapabilities, Instant startedAt)

record ReviewDetail(UUID caseId, Map<String, Object> prMetadata, List<GoalStatus> goals,
                    List<EventEntry> timeline, List<LedgerRecord> ledgerEntries,
                    List<CommitmentSummary> commitments)
record GoalStatus(String name, String kind, boolean reached)
record EventEntry(Instant timestamp, String eventType, String actor, String summary)
record LedgerRecord(UUID entryId, String type, Instant timestamp, boolean chainVerified)
record CommitmentSummary(UUID id, String obligor, String capability, String state, Instant createdAt)

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
