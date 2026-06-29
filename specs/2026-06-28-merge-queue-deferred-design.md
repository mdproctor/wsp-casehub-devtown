# Merge Queue Deferred Work — SLA, Preferences, Ledger, PR-Review Integration, Persistence

> **Date:** 2026-06-28
> **Issue:** devtown#100
> **Parent spec:** `specs/2026-06-26-merge-queue-design.md` (Epic #11)
> **Status:** Design approved in brainstorming

---

## 1. Problem Statement

The merge queue initial implementation (devtown#11) deferred five integration points to keep scope focused on the CasePlanModel, queue service, and bisection strategies. These are not optional enhancements — they are structural gaps that prevent production use:

1. **SLA WorkItem creation** — no obligation tracking for queued PRs; a PR can wait indefinitely with no visibility or escalation
2. **PreferenceProvider integration** — all configuration hardcoded as `DEFAULT_*` constants; no per-repo customization
3. **MergeDecisionLedgerEntry enrichment** — batch merges produce no batch metadata in the audit trail; the observer reads PR review context paths that don't exist on batch cases
4. **PR Review Case integration** — no path from a completed PR review to the merge queue; the only admission path is the MCP `enqueue_pr` tool
5. **Queue persistence** — in-memory `CopyOnWriteArrayList` / `ConcurrentHashMap`; queue state lost on restart

---

## 2. Module Structure

New `merge/` module for merge queue integration ports, parallel to `review/`:

```
domain/  → vocabulary, preference keys (pure Java, no Quarkus)
queue/   → merge queue domain logic (pure Java) — unchanged
merge/   → NEW: merge queue port interfaces and value types (queue/ dep only)
review/  → PR review integration (casehub-work, engine deps)
github/  → GitHub API clients, webhook handling (unchanged)
app/     → CDI wiring, JPA entities, MCP tools, REST endpoints
```

### 2.1 What `merge/` contains (new module)

- `MergeQueueStore` — port interface for queue persistence
- `QueueEntry`, `BatchRecord`, `QueueEntryStatus` — value types
- merge-batch YAML CasePlanModel (`resources/devtown/merge-batch.yaml`) — moved from `queue/src/main/resources/devtown/` to correct an existing tier violation (YAML CasePlanModel is an engine artifact, belongs in tier 2, not tier 1 pure-Java)

### 2.2 What stays in `app/`

- `MergeQueueService` — queue lifecycle orchestration (`@ApplicationScoped`)
- `MergeBatchCaseHub` — `YamlCaseHub` subclass, worker registration (`@ApplicationScoped`)
- `PrReviewMergeQueueAdapter` — registers `merge-queue-enqueue` worker on PR review case (`@ApplicationScoped`)
- `MergeQueueSlaBreachObserver` — CDI observer for `SlaBreachEvent` (`@ApplicationScoped`)
- `MergeDecisionLedgerEntry` + Flyway migration — JPA entity, persistence unit ownership
- `MergeDecisionObserver` — enhanced for batch context, stays near the entity it writes
- `MergeBatchCompletionObserver` — CDI observer for batch case terminal events, drives queue cleanup and WorkItem obsolescence (`@ApplicationScoped`)
- `JpaMergeQueueStore` — JPA implementation of the `MergeQueueStore` port
- `QueuedPrEntity`, `ActiveBatchEntity` — JPA entities for queue persistence
- MCP tools, REST endpoints

### 2.3 Dependencies

- `merge/` depends on `queue/` (only — after R1-04/R1-07 moved all CDI beans to `app/`, the remaining types in `merge/` reference only `queue/` types and `java.*`)
- `app/` depends on `merge/`, `review/`, `queue/`, `domain/`, `github/`

### 2.4 Maven coordinates

- `groupId`: `io.casehub.devtown`
- `artifactId`: `casehub-devtown-merge`
- Package: `io.casehub.devtown.merge`

---

## 3. PreferenceProvider Integration

### 3.1 Preference keys

New `MergeQueuePreferenceKeys` in `domain/` (follows `SlaPreferenceKeys` pattern):

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `devtown.merge-queue.enabled` | `BooleanPreference` | `false` | Enable merge queue for this repo |
| `devtown.merge-queue.max-batch-size` | `IntPreference` | `10` | Maximum batch size |
| `devtown.merge-queue.min-batch-size` | `IntPreference` | `1` | Floor for adaptive sizing |
| `devtown.merge-queue.bisection-strategy` | `StringPreference` | `trust-weighted` | Split strategy name |
| `devtown.merge-queue.failure-rate-window` | `IntPreference` | `20` | Recent batches for adaptive sizing |
| `devtown.merge-queue.priority.decay-rate-per-hour` | `IntPreference` | `125` | Starvation prevention coefficient |
| `devtown.merge-queue.target-branch` | `StringPreference` | `main` | Target branch |

SLA keys — individual `PreferenceKey<StringPreference>` per lane (follows `SlaPreferenceKeys` pattern — no subKey needed):

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `devtown.merge-queue.sla.CRITICAL` | `StringPreference` | `PT1H` | SLA duration for CRITICAL lane |
| `devtown.merge-queue.sla.HIGH` | `StringPreference` | `PT4H` | SLA duration for HIGH lane |
| `devtown.merge-queue.sla.NORMAL` | `StringPreference` | `PT8H` | SLA duration for NORMAL lane |

### 3.2 Scope

All merge queue preferences resolved at `Path.of("casehubio", "devtown", "merge-queue")`. Follows the PLATFORM.md convention: org first, app second, case-type third.

### 3.3 MergeQueueService changes

`MergeQueueService` injects `PreferenceProvider`. On every operation:

```java
Preferences prefs = preferenceProvider.resolve(
    new SettingsScope(Path.of("casehubio", "devtown", "merge-queue"), Instant.now()));
int maxBatch = prefs.getOrDefault(MergeQueuePreferenceKeys.MAX_BATCH_SIZE).value();
```

`BatchFormationContext` receives resolved values instead of hardcoded defaults. The `DEFAULT_*` constants are removed entirely.

**Bug fix:** The current `DEFAULT_DECAY_RATE = 0.1` is incorrect — the parent spec §3.2 defines the decay rate as `125` points/hour. The preference default of `125` is the correct value. `BatchFormationContext.decayRatePerHour` remains `double` for precision in the decay formula; the `IntPreference` value is widened to `double` at context construction.

### 3.4 YAML config file

New `app/src/main/resources/casehub/devtown/merge-queue.yaml` (parallels `trust-routing.yaml`):

```yaml
entries:
  - scope: casehubio/devtown/merge-queue
    devtown.merge-queue.enabled: "false"
    devtown.merge-queue.max-batch-size: "10"
    devtown.merge-queue.min-batch-size: "1"
    devtown.merge-queue.bisection-strategy: "trust-weighted"
    devtown.merge-queue.failure-rate-window: "20"
    devtown.merge-queue.priority.decay-rate-per-hour: "125"
    devtown.merge-queue.target-branch: "main"
    devtown.merge-queue.sla.CRITICAL: "PT1H"
    devtown.merge-queue.sla.HIGH: "PT4H"
    devtown.merge-queue.sla.NORMAL: "PT8H"
    devtown.sla.escalation-group: "merge-leads"
    devtown.sla.escalation-hours: "4"
    devtown.sla.breach-terminal-reason: "merge-queue-sla-breach"
```

---

## 4. SLA WorkItem Creation

### 4.1 Lifecycle

1. `MergeQueueService.enqueue()` → create WorkItem via `WorkItemService.create()` with `callerRef` set to `"{repository}#{prNumber}"` (e.g., `"casehubio/devtown#456"`). This encodes the domain identity needed by the SLA breach observer (§4.4) to resolve the PR without a store lookup. The `CallerRef` sealed interface is not used — merge-queue WorkItems are standalone SLA timers, not plan-item-bound (no `caseId` or `planItemId` to encode).
2. PR exits queue (merged / rejected / dequeued) → `WorkItemService.obsoleteFromSystem()`
3. SLA breach → `SlaBreachEvent` CDI event fired by casehub-work

### 4.2 WorkItemTemplate

Devtown's first `WorkItemTemplate`. Seeded via Flyway data migration for production, `@BeforeEach @Transactional` for tests (per `harness-workitem-template-test-seeding` protocol):

- `name`: `merge-queue-wait`
- `description`: "PR waiting in merge queue"
- `candidateGroups`: *(empty — this is a system-managed SLA timer, not a human task)*
- `outcomes`: *(empty — the WorkItem is always terminated by the system via `obsoleteFromSystem()`)*
- `scope`: `casehubio/devtown/merge-queue`

This WorkItem exists purely for SLA breach detection. No human claims or completes it. Queue status visibility is exposed through MCP tools and REST endpoints, not the human task inbox.

### 4.3 SLA duration

Set per PR at WorkItem creation time based on priority lane. The lane's SLA duration is resolved from the per-lane `MergeQueuePreferenceKeys`:

```java
PreferenceKey<StringPreference> slaKey = MergeQueuePreferenceKeys.slaKeyFor(lane);
String slaDuration = prefs.getOrDefault(slaKey).value();
Duration expiry = Duration.parse(slaDuration);
```

`MergeQueuePreferenceKeys.slaKeyFor(PriorityLane)` returns the corresponding `SLA_CRITICAL`, `SLA_HIGH`, or `SLA_NORMAL` key.

### 4.4 Breach handling — two layers

**Layer 1 — `DefaultSlaBreachPolicy`** (already exists): Returns standard decisions based on preferences at the WorkItem's scope (`casehubio/devtown/merge-queue`). First breach → `EscalateTo(merge-leads)` with new deadline. Already-escalated → `Fail("sla-breach")`.

**Layer 2 — `MergeQueueSlaBreachObserver`** in `app/`: Observes `SlaBreachEvent` CDI events. Filters on scope — only handles events where `context.scope()` matches `casehubio/devtown/merge-queue`; PR review breaches (scope `casehubio/devtown/pr-review`) are ignored. On any merge-queue SLA breach: reads `event.context().task().callerRef()`, parses the `"{repository}#{prNumber}"` convention (set at enqueue time, §4.1) to extract `prNumber` and `repository`, then calls `MergeQueueService.prioritize(prNumber, repository)` to force the PR into the next batch as a domain-specific side-effect. All breached PRs are prioritized regardless of lane — the SLA durations per lane (CRITICAL: 1h, HIGH: 4h, NORMAL: 8h) already encode the urgency difference. A NORMAL PR that breaches its 8-hour SLA has waited long enough to warrant immediate dispatch. The breach policy stays within its foundation contract (return a decision); the domain action is an event observer. The observer lives in `app/` because it injects `MergeQueueService`.

**`MergeQueueService.prioritize(int prNumber, String repository)` contract:**

1. Verifies the PR is in state `QUEUED` — no-op if not in state `QUEUED` (covers `IN_BATCH`, `MERGED`, `REJECTED`, `DEQUEUED`, and absent)
2. Calls `store.markPrioritized(prNumber, repository)` to flag the entry
3. Calls `formAndDispatchBatches()`, which checks for prioritized entries — see §7.4 dispatch threshold

This ensures an SLA breach forces immediate dispatch — the PR does not wait for more PRs to accumulate. Other QUEUED PRs may be included in the same batch if compatible. The method is idempotent: calling it on an already-prioritized or already-batched PR has no effect.

### 4.5 WorkItem ID tracking

The WorkItem ID is stored alongside the queued PR in `MergeQueueStore` (§7). When the PR exits the queue, the service uses the stored ID to call `obsoleteFromSystem()`.

---

## 5. MergeDecisionLedgerEntry Enrichment

### 5.1 New columns

Added directly to the existing `V2002__merge_decision_ledger_entry.sql` CREATE TABLE (no production installs):

```sql
batch_id          VARCHAR(64),
batch_size        INTEGER,
bisection_occurred BOOLEAN,
bisection_strategy VARCHAR(30),
batch_context_json TEXT
```

Index on `batch_id`.

### 5.2 Entity changes

```java
@Column(name = "batch_id", length = 64)
public String batchId;

@Column(name = "batch_size")
public Integer batchSize;

@Column(name = "bisection_occurred")
public Boolean bisectionOccurred;

@Column(name = "bisection_strategy", length = 30)
public String bisectionStrategy;

@Column(name = "batch_context_json", columnDefinition = "TEXT")
public String batchContextJson;
```

### 5.3 Domain content hash

`domainContentBytes()` updated to include all scalar fields. Null fields hash to empty string. `batchContextJson` is **excluded** — JSON serialization is non-deterministic (key ordering, whitespace, numeric representation vary across serializers), which would violate the `ledger-subclass-extension.md` requirement that `domainContentBytes()` be deterministic. The batch context is supplementary audit metadata; the decision identity is fully captured by the scalar fields.

```java
@Override
protected byte[] domainContentBytes() {
    return String.join("|",
        String.valueOf(prNumber),
        escapePipe(repository),
        escapePipe(commitSha),
        escapePipe(decision),
        caseId != null ? caseId.toString() : "",
        escapePipe(batchId),
        batchSize != null ? String.valueOf(batchSize) : "",
        bisectionOccurred != null ? String.valueOf(bisectionOccurred) : "",
        escapePipe(bisectionStrategy)
    ).getBytes(StandardCharsets.UTF_8);
}
```

### 5.4 `batch_context_json` structure

```json
{
  "prList": [
    {"number": 456, "repository": "casehubio/devtown", "author": "alice", "trustScore": 0.85},
    {"number": 457, "repository": "casehubio/devtown", "author": "bob", "trustScore": 0.72}
  ],
  "trustScoresAtDecision": {"alice": 0.85, "bob": 0.72},
  "ciRunIds": ["run-12345"],
  "rejectedPrs": [457]
}
```

### 5.5 Observer enhancement

`MergeDecisionObserver` handles two paths, distinguished by case context shape:

**PR review path** (existing): Case context has `pr.repo`, `pr.id`. Batch columns stay null. Unchanged behavior.

**Merge batch path** (new): Case context has `batch.*` AND `batch.isRootBatch == true`. The observer skips sub-case completions during bisection — sub-cases receive `{ batch: .splitResult.left }` from the bisection splitter, which does not include `isRootBatch`. Only root batch cases set `batch.isRootBatch = true` at dispatch (§7.4). Without this guard, bisection sub-case completions write duplicate ledger entries (each sub-case has `batch.*` context and fires `CaseLifecycleEvent`). The observer:
1. Reads `batch.prs` — iterates to write one entry per PR
2. Each entry carries shared batch metadata: `batchId`, `batchSize`, `bisectionOccurred`, `bisectionStrategy`
3. `batchContextJson` carries the full PR list, trust scores, CI run IDs
4. `decision` = APPROVED (if batch merged) or REJECTED (if cancelled)
5. `subjectId` = deterministic UUID derived from batch caseId + PR number (UUID v5 with `caseId` as namespace, `prNumber` as name). This ensures `findLatestBySubjectId()` returns the specific PR's entry, not an arbitrary entry from the batch.
6. `actorId` = `"system"` (correct for automated merges)
7. `ComplianceSupplement.algorithmRef` = `"casehub-devtown:merge-queue-v1"` (distinct from `pr-review-v1`; the merge queue is a different decision algorithm)

Batch context aggregation (walking the bisection tree) is `MergeQueueService`'s responsibility (per original spec §9, decision #9). The observer receives aggregated results, not the raw tree.

---

## 6. PR Review Case Integration

### 6.1 Mutually exclusive merge bindings

The existing `merge` binding in `pr-review.yaml` becomes two bindings:

```yaml
- name: enqueue-for-merge
  on: { contextChange: {} }
  when: >-
    .merge_sha == null and
    .enqueueResult == null and
    .pr.status != "merged" and
    (.codeAnalysis.securitySensitive == false or .securityReview.outcome == "APPROVED") and
    (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
    .styleCheck.outcome == "APPROVED" and
    .testCoverage.outcome == "APPROVED" and
    .performanceAnalysis.outcome == "APPROVED" and
    (.pr.linesChanged <= .policy.humanApprovalThreshold or .humanApproval.outcome == "APPROVED") and
    .ci.status == "passing" and
    .policy.mergeQueueEnabled == true
  capability: merge-queue-enqueue
  conflictResolverStrategy: DEEP_MERGE
  outcomePolicy:
    onDecline: FAULT
    onFailure: FAULT
    onExpired: FAULT

- name: merge-direct
  on: { contextChange: {} }
  when: >-
    .merge_sha == null and
    .pr.status != "merged" and
    (.codeAnalysis.securitySensitive == false or .securityReview.outcome == "APPROVED") and
    (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
    .styleCheck.outcome == "APPROVED" and
    .testCoverage.outcome == "APPROVED" and
    .performanceAnalysis.outcome == "APPROVED" and
    (.pr.linesChanged <= .policy.humanApprovalThreshold or .humanApproval.outcome == "APPROVED") and
    .ci.status == "passing" and
    .policy.mergeQueueEnabled != true
  capability: merge-executor
  conflictResolverStrategy: DEEP_MERGE
  outcomePolicy:
    onDecline: FAULT
    onFailure: FAULT
    onExpired: FAULT
```

### 6.2 New capability

```yaml
- name: merge-queue-enqueue
  description: "Enqueues the PR into the merge queue"
  inputSchema: "{ pr: .pr, codeAnalysis: .codeAnalysis }"
  outputSchema: "{ enqueueResult: . }"
```

### 6.3 Worker function

Registered via a `PrReviewMergeQueueAdapter` in `app/`. The adapter is an `@ApplicationScoped` bean that injects `PrReviewCaseHub`, `MergeQueueService`, and `TrustScoreSource` (all from `app/`). It registers the `merge-queue-enqueue` worker function on the PR review `CaseDefinition` at `@PostConstruct`. The adapter lives in `app/` because it must inject `PrReviewCaseHub` (also in `app/`) — placing it in `merge/` would create a circular dependency (`app/ → merge/ → app/`).

**QueuedPr construction from case context:**

| Field | Source |
|-------|--------|
| `number` | `pr.id` (parsed as int) |
| `headSha` | `pr.headSha` |
| `author` | `pr.contributor` |
| `trustScore` | Looked up from `TrustScoreSource.getGlobalScore(contributor)`. If unavailable (new contributor, service error), defaults to `0.5` (neutral trust — avoids penalizing unknown contributors while not granting full trust). |
| `lane` | `NORMAL` for all automated CasePlanModel enqueues. The MCP `enqueue_pr` tool accepts an explicit `priority` parameter for operator-driven lane selection. |
| `enqueuedAt` | `Instant.now()` |
| `dependsOn` | Empty set. PR dependency extraction (from description, labels, or GitHub API) is out of scope — tracked as part of devtown#101 (webhook/label integration). |

The worker calls `MergeQueueService.enqueue()` followed by `MergeQueueService.formAndDispatchBatches()`.

### 6.4 Batch formation trigger

Batch formation is triggered synchronously after every enqueue operation:

1. **PR review case path**: The `enqueue-for-merge` worker calls `MergeQueueService.enqueue()`, then `MergeQueueService.formAndDispatchBatches()`. The batch formation logic in `BatchCompositionPolicy` decides whether to form batches now (sufficient PRs, compatible lanes) or leave PRs queued.
2. **MCP `enqueue_pr` tool path**: Same call sequence — `enqueue()` then `formAndDispatchBatches()`.
3. **GitHub webhook path** (future, devtown#101): Same call sequence when implemented.

No scheduled timer is needed. Batch formation is an eager, idempotent operation — calling it with insufficient PRs queued is a no-op (returns empty list). This avoids polling overhead and ensures PRs are batched immediately when conditions are met.

### 6.5 Policy injection

The PR review case needs `policy.mergeQueueEnabled` in its context. `PrReviewApplicationService` resolves `MergeQueuePreferenceKeys.ENABLED` via `PreferenceProvider` and includes it in the case's initial context alongside the existing `policy.humanApprovalThreshold`. Zero behavior change when `mergeQueueEnabled` is false (default).

---

## 7. Queue Persistence

### 7.1 Port interface in `merge/`

**Prerequisite:** `QueuedPr` in `queue/` gains a `String repository` field (inserted after `number`). This is a domain concept — a PR belongs to a repository — and belongs in the tier 1 value type. All callers of `QueuedPr` are updated accordingly.

```java
public interface MergeQueueStore {
    void enqueue(QueuedPr pr, UUID workItemId);
    boolean dequeue(int prNumber, String repository);
    List<QueueEntry> queued();
    List<QueueEntry> queuedForUpdate();
    void markInBatch(List<Integer> prNumbers, String repository, String batchId);
    void markCompleted(int prNumber, String repository, String outcome);
    void markPrioritized(int prNumber, String repository);
    void markQueued(List<Integer> prNumbers, String repository);

    void recordBatch(String batchId, UUID caseId, List<Integer> prNumbers, String repository);
    Optional<BatchRecord> findBatchByCaseId(UUID caseId);
    List<QueueEntry> findEntriesByBatchId(String batchId);
    Map<String, BatchRecord> activeBatches();
}
```

All mutating operations use `(prNumber, repository)` as composite identifier — PR numbers are not globally unique across repositories.

`enqueue()` is idempotent: if a `(prNumber, repository)` entry already exists in state `QUEUED` or `IN_BATCH`, the call returns silently. This prevents duplicate enqueue from any admission path (CasePlanModel binding, MCP tool, future webhook).

`queuedForUpdate()` returns QUEUED entries with a `SELECT FOR UPDATE` lock, used exclusively by `formAndDispatchBatches()` to serialize concurrent batch formation at the database level (see §7.3).

### 7.2 Value types in `merge/`

```java
public record QueueEntry(
    QueuedPr pr,
    UUID workItemId,
    QueueEntryStatus status,    // QUEUED, IN_BATCH, MERGED, REJECTED, DEQUEUED
    boolean prioritized,        // true after markPrioritized(); orthogonal to status
    String batchId              // null until batched
) {}

public record BatchRecord(
    String batchId,
    UUID caseId,
    List<Integer> prNumbers,
    String repository,
    Instant dispatchedAt
) {}

public enum QueueEntryStatus {
    QUEUED, IN_BATCH, MERGED, REJECTED, DEQUEUED
}
```

### 7.3 JPA implementation in `app/`

`JpaMergeQueueStore implements MergeQueueStore` — `@ApplicationScoped`.

JPA entities:
- `QueuedPrEntity` — maps queued PR state to `merge_queue_entry` table
- `ActiveBatchEntity` — maps batch→case relationship to `merge_queue_batch` table

Flyway migration in `app/src/main/resources/db/devtown/migration/`.

**Concurrent batch formation serialization:** `queuedForUpdate()` uses `SELECT FOR UPDATE` to lock QUEUED rows during batch formation. When two threads call `formAndDispatchBatches()` concurrently, the second blocks until the first completes its transaction. The first thread transitions PRs to `IN_BATCH`; the second finds no QUEUED rows and returns an empty list. This is database-level serialization — correct in multi-node deployments without distributed application locks.

### 7.4 MergeQueueService changes

`MergeQueueService` injects `MergeQueueStore` instead of maintaining in-memory fields. All queue mutations go through the store. The service becomes stateless.

**Repository-aware batch formation:** `formAndDispatchBatches()` groups queued entries by `QueuedPr.repository()` before calling `compositionPolicy.formBatches()` for each group. A batch containing PRs from multiple repositories is invalid — the merge executor and batch CI runner target a single repository. The grouping happens at the service level in `app/`, not in the pure-Java `queue/` module; `BatchCompositionPolicy` receives only same-repository PRs.

**queue/ type changes for repository propagation:**
- `BatchFormationContext` gains `String repository` — the service constructs one context per repository group
- `Batch` gains `String repository` — `buildBatch()` in `DefaultBatchCompositionPolicy` propagates `ctx.repository()` to the batch
- `dispatchBatch()` includes `batch.repository()` and `batch.isRootBatch = true` in the case context map (alongside `batch.targetBranch`, `batch.id`, etc.). The `isRootBatch` marker is set only at dispatch — bisection sub-cases receive `{ batch: .splitResult.left }` from the splitter output, which does not include this field. This enables `MergeDecisionObserver` to distinguish root batch completions from sub-case completions (§5.5)

These are tier 1 value type changes — no framework dependencies introduced. Consistent with `QueuedPr` already gaining `repository` (R2-02).

**Dispatch threshold:** Before calling the composition policy, `formAndDispatchBatches()` applies a per-repository-group pre-check: if the number of QUEUED entries in the group is less than `minBatchSize` AND no entry in the group is prioritized, skip that group — leave entries queued for a future call. This gives `minBatchSize` dispatch-gating semantics: with `minBatchSize = 3`, PRs accumulate until 3 are queued before a batch is formed. Prioritized entries (from SLA breach, §4.4) bypass this threshold, forcing immediate dispatch even for a single PR. Note: `minBatchSize` in `BatchCompositionPolicy` is a separate concern — it is the floor for adaptive batch *sizing* (the maximum batch size never drops below it), not a dispatch gate. The dispatch threshold is a service-level concern in `formAndDispatchBatches()`.

`formAndDispatchBatches()` calls `store.queuedForUpdate()` (not `store.queued()`) to acquire the `SELECT FOR UPDATE` lock, ensuring serialized batch formation.

**Transaction boundary:** `formAndDispatchBatches()` has three phases with distinct transaction requirements:

1. **Batch formation (single transaction):** `queuedForUpdate()` → `formBatches()` → `markInBatch()` per batch. The `SELECT FOR UPDATE` lock, batch formation, and status transition to `IN_BATCH` must be atomic — if any step fails, all entries remain `QUEUED`.

2. **Case start (engine transaction):** `mergeBatchCaseHub.startCase(batchContext)` → returns `caseId`. Runs in the engine's own transaction. If this fails after markInBatch committed, the entries are `IN_BATCH` with no running case — see recovery below.

3. **Batch recording:** `recordBatch(batchId, caseId, prNumbers, repository)`. Links the batch to the running case.

**Failure recovery:** If case start fails (phase 2), a compensating action returns the affected entries to `QUEUED` via `store.markQueued(prNumbers, repository)` — a new store method that reverses `markInBatch`. If batch recording fails (phase 3, rare), the case runs but `handleBatchCompletion()` returns early (R4-02 guard — no batch record found). A startup reconciliation check detects entries stuck in `IN_BATCH` with no corresponding `activeBatch` record or running case and returns them to `QUEUED`.

### 7.5 Batch completion flow

When a merge-batch case reaches a terminal state, the queue must be cleaned up and WorkItems obsoleted. This is the return path for the forward flow described in §4.1 and §6.4.

**`MergeBatchCompletionObserver`** (new, in `app/`):
1. Observes `CaseLifecycleEvent` via `@ObservesAsync` (same pattern as `MergeDecisionObserver`)
2. Looks up `CaseInstance` via `CrossTenantCaseInstanceRepository` and reads case context
3. Filters: only acts on batch cases (context has `batch.*` key, not `pr.*`)
4. Maps case status: `COMPLETED` → `batchSucceeded = true`, `CANCELLED` → `batchSucceeded = false`
5. Extracts `rejectedPrs` from case context — the set of PR numbers rejected during bisection (may be empty if the batch merged cleanly or was cancelled before bisection)
6. Calls `MergeQueueService.handleBatchCompletion(caseId, batchSucceeded, rejectedPrNumbers)`

The observer is the adapter between engine events and domain operations — it owns the `CaseInstance` lookup and context extraction, keeping the service free of engine dependencies. This matches the existing pattern in `MergeDecisionObserver`.

**`MergeQueueService.handleBatchCompletion(UUID caseId, boolean batchSucceeded, Set<Integer> rejectedPrNumbers)`**:
1. `store.findBatchByCaseId(caseId)` → `Optional<BatchRecord>`. If empty, return — sub-case lifecycle events are expected during bisection (the `merge-batch` YAML uses recursive sub-cases for `bisect-left`/`bisect-right`) and have no batch record. Only root batch cases are registered via `recordBatch()`, so only root completions trigger queue cleanup.
2. `store.findEntriesByBatchId(batchRecord.batchId())` → list of `QueueEntry`
3. For each entry: determine outcome — if PR number in `rejectedPrNumbers` → `REJECTED`; else if `batchSucceeded` → `MERGED`; else → `DEQUEUED`. Call `store.markCompleted(entry.pr().number(), entry.pr().repository(), outcome)`.
4. For each entry: `workItemService.obsoleteFromSystem(entry.workItemId(), "system", "batch-" + outcome.toLowerCase())`

This ensures:
- Queue entries reach a terminal state (`MERGED`, `REJECTED`, or `DEQUEUED`)
- WorkItem SLA timers stop — no false breaches after batch completion
- `activeBatches()` stays bounded

---

## 8. Testing Strategy

### 8.1 Unit tests

| Test | Module | Covers |
|------|--------|--------|
| `MergeQueuePreferenceKeysTest` | `domain` | Key parsing, defaults, qualified names |
| `MergeQueueStoreContractTest` | `merge` | Store interface contract (abstract, run against JPA impl) |

### 8.2 Integration tests (`@QuarkusTest`)

| Test | Module | Covers |
|------|--------|--------|
| `MergeQueueSlaWorkItemTest` | `app` | Enqueue → WorkItem created with correct SLA. Merge → WorkItem obsoleted. Dequeue → WorkItem obsoleted. |
| `MergeQueueSlaBreachTest` | `app` | SLA breach → escalation. Any merge-queue breach → `prioritize()` marks PRIORITIZED + triggers `formAndDispatchBatches()` bypassing dispatch threshold. Already-escalated → terminal Fail. |
| `MergeQueuePreferenceIntegrationTest` | `app` | Preference resolution from YAML. Override per scope. All hardcoded defaults eliminated. |
| `MergeDecisionObserverBatchTest` | `app` | Batch case completion → N ledger entries (one per PR) with batch metadata. Bisection context in `batchContextJson`. Sub-case filtering: bisection sub-case completes → no ledger entries written (no `batch.isRootBatch`). Root batch completes → correct number of entries. |
| `MergeQueuePersistenceTest` | `app` | Enqueue/dequeue survives (simulated) restart. Batch dispatch recorded. Completion callback finds batch by caseId. Concurrent `formAndDispatchBatches()` does not produce overlapping batches. |
| `MergeBatchCompletionTest` | `app` | Batch case completion → queue entries reach terminal state (MERGED/REJECTED/DEQUEUED). WorkItems obsoleted for all PRs in batch. Cancelled batch → all PRs DEQUEUED. |
| `MergeQueueIdempotencyTest` | `app` | Duplicate enqueue for same `(prNumber, repository)` is silently ignored. Store-level and binding-level guards both verified. |
| `MergeQueueMultiRepoBatchTest` | `app` | Enqueue PRs from repo A and repo B → `formAndDispatchBatches()` → two separate batches, one per repo. No cross-repo contamination. Verifies §7.4 repository-aware grouping. |
| `PrReviewMergeQueueRoutingTest` | `app` | `mergeQueueEnabled=true` → `enqueue-for-merge` fires. `mergeQueueEnabled=false` → `merge-direct` fires. |

### 8.3 WorkItemTemplate test seeding

Per `harness-workitem-template-test-seeding` protocol: `@BeforeEach @Transactional` seeds the `merge-queue-wait` template in tests. Guard: `if (WorkItemTemplate.find("name", name).count() == 0)`.

---

## 9. Out of Scope

These items are natural follow-ups but not part of devtown#100. Each is tracked as a GitHub issue:

- **GitHub webhook receiver** — `pull_request.labeled` admission path (spec §7.2) — devtown#101
- **Batch branch management** — git operations for batch testing (spec §7.3, requires Claudony workers) — devtown#104
- **MCP tool enhancements** — `list_problems` gaining `QUEUE_SLA_BREACH`, `get_merge_queue_metrics` — devtown#102
- **Adaptive batch sizing feedback loop** — computing `recentFailureRate` from batch outcome history (enabled by persistence but not wired) — devtown#103
- **PrReviewCaseService preference migration** — migrate remaining `@ConfigProperty` fields to `PreferenceProvider` to eliminate the hybrid config model created by §6.5's `mergeQueueEnabled` injection — devtown#105

---

## 10. Platform Coherence Verification

| Concern | Protocol | Status |
|---------|----------|--------|
| Path convention | PLATFORM.md §Capability Ownership | `Path.of("casehubio", "devtown", "merge-queue")` — org/app/case-type ✅ |
| Ledger subclass | `ledger-subclass-extension.md` | JOINED, consumer migration, no shadowing ✅ |
| Typed preferences | `typed-preference-keys.md` | `PreferenceKey<T>` with parser in `domain/` ✅ |
| SLA breach | `SlaBreachPolicy` SPI | Decisions via policy, domain actions via CDI observer ✅ |
| Module tiers | `module-tier-structure.md` | `queue/` pure Java, `merge/` integration, `app/` CDI ✅ |
| Persistence | `persistence-backend-cdi-priority.md` | `@ApplicationScoped` JPA in `app/` ✅ |
| Test seeding | `harness-workitem-template-test-seeding.md` | `@BeforeEach @Transactional` ✅ |
| Ledger writer | `harness-ledger-writer.md` | Single `MergeDecisionObserver` owns all writes ✅ |
| Boundary rules | PLATFORM.md §Key Boundary Rules | No domain logic in foundation ✅ |
