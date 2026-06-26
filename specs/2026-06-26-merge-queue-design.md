# Merge Queue Design — CasePlanModel Batch-Then-Bisect

> **Date:** 2026-06-26
> **Epic:** devtown#11 (Epic 4: Merge queue)
> **Foundation gates:** engine#573 (recursive sub-case depth limit), engine#574 (M-of-N in YAML + per-child outputMapping)
> **Status:** Design approved, revised after schema alignment and adversarial reviews

---

## 1. Problem Statement

Multiple PRs are ready to merge concurrently. Each was tested against main at some point in the past, but main moves with each merge. Testing every PR against latest main serially is correct but slow. Batching is the optimization — test N PRs together, merge all if green, find the culprit if red.

Gastown's Refinery solves this with a Go state machine: FIFO batching, mechanical binary bisection, hardcoded strategy. Changing the strategy requires changing code and redeploying. There is no priority, no trust-awareness, no dependency ordering, no SLA tracking, and no cryptographic audit of merge decisions.

CaseHub's merge queue solves the same problem with structural advantages at every layer.

---

## 2. Architecture — Two-Tier Decomposition

### Why two tiers?

Queue management (admission, ordering, batch formation) is **imperative scheduling** — "select the next N satisfying these constraints." Batch processing (test, bisect, merge, reject) is **reactive state management** — "if test failed and batch > 1, bisect." Any design that forces both into the same paradigm is awkward in one direction.

```
┌─────────────────────────────────────────────────────┐
│  MergeQueueService  (@ApplicationScoped)            │
│                                                     │
│  Receives: PR merge-ready signal (webhook/MCP/case) │
│  Manages:  Priority queue, dependency DAG           │
│  Forms:    Batches via BatchCompositionPolicy        │
│  Creates:  WorkItem per queued PR (SLA tracking)    │
│  Spawns:   Batch Case per formed batch              │
│  Handles:  Batch completion (merged/rejected)       │
│                                                     │
│  Configuration: PreferenceProvider (per-repo scope)  │
└──────────────────────┬──────────────────────────────┘
                       │ spawns
                       ▼
┌─────────────────────────────────────────────────────┐
│  merge-batch CasePlanModel  (YAML)                  │
│                                                     │
│  Goals: batch-merged, single-pr-rejected (success)  │
│         merge/tip-test terminal failure (failure)    │
│  Bindings:                                          │
│    test-tip → batch-ci-runner                       │
│    merge-batch → merge-executor (trust ≥ 0.80)      │
│    compute-split → bisection-splitter (pluggable)   │
│    bisect-left/right → recursive sub-case (M-of-N)  │
│    reject-single → pr-reject-and-notify             │
│    human-oversight → WorkItem (high-risk batches)   │
│    merge-escalation → WorkItem (reroutes exhausted) │
│    tip-test-escalation → WorkItem (CI exhausted)    │
│    *-after-escalation → contextWrite reset + retry  │
│                                                     │
│  Recursive: bisect spawns merge-batch sub-cases     │
│  Ledger: MergeDecisionLedgerEntry on every merge    │
└─────────────────────────────────────────────────────┘
```

### Module placement

Following the existing three-tier rule:

- `devtown-domain` — new vocabulary types: `MergeQueueCapability`, `MergeRiskDimension`
- `devtown-merge` — new module alongside `review/` — queue service, batch composition policy, dependency resolver, bisection split strategies
- `devtown-app` — CDI wiring, CasePlanModel registration, MCP tools

The merge queue is a distinct domain concern from PR review. A PR goes through review (Layer 5 case), then enters the merge queue (this case). Separate modules, separate case definitions, shared domain vocabulary.

---

## 3. Queue Service Design

### 3.1 Admission

Three paths into the queue:

1. **From PR Review Case** (primary) — when pr-review case completes successfully, the `enqueue-for-merge` binding fires instead of the direct merge binding. Controlled by `policy.mergeQueueEnabled` preference.
2. **GitHub webhook** — for PRs not managed by CaseHub. Webhook receiver validates approval + CI status before enqueuing.
3. **MCP tool** — `enqueue_pr` for manual override.

### 3.2 Priority Scoring

Composite score determines processing order:

```
score = lanePriority × 1000 + authorTrustScore × 100 + waitTimeDecayFactor
```

| Component | Source | Range |
|-----------|--------|-------|
| `lanePriority` | PR label: CRITICAL=3, HIGH=2, NORMAL=1 | 1000–3000 |
| `authorTrustScore` | Aggregate trust from ledger (0.0–1.0) | 0–100 |
| `waitTimeDecayFactor` | `waitTimeMinutes × (decayRatePerHour / 60)` | 0–∞ |

**Decay rate:** Default 125 points/hour. NORMAL PR overtakes fresh HIGH at ~8 hours, fresh CRITICAL at ~16 hours. Configurable via `devtown.merge-queue.priority.decay-rate-per-hour`.

### 3.3 Dependency Resolution

The queue service builds a DAG before forming batches:

- **Explicit**: `depends-on: #456` label on the PR
- **Implicit**: if PR B's base branch is PR A's head branch, B depends on A (stacked PRs)
- **Constraints**:
  - A dependent PR never enters a batch without its dependency
  - If a dependency is rejected, all dependents are automatically dequeued and authors notified
  - Cycle detection prevents circular dependencies from blocking the queue

### 3.4 Batch Formation

`BatchCompositionPolicy` interface with a default implementation:

```java
public interface BatchCompositionPolicy {
    List<Batch> formBatches(List<QueuedPr> queue, BatchFormationContext ctx);
}
```

Default policy algorithm:
1. Take PRs from the priority queue in score order
2. Respect dependency ordering (topological sort within the batch)
3. Size the batch based on trust: `maxSize = floor(minTrustInBatch × MAX_BATCH_SIZE)`
4. Apply adaptive sizing: `effectiveMax = max(MIN_BATCH_SIZE, floor(maxSize × (1 - recentFailureRate)))`
5. All parameters configurable via PreferenceProvider

### 3.5 SLA Tracking

Each queued PR gets a WorkItem via casehub-work:

| Priority | SLA | First breach | Second breach |
|----------|-----|-------------|--------------|
| CRITICAL | 1 hour | Notify repo maintainers | Force into next batch |
| HIGH | 4 hours | Notify PR author | Escalate to team lead |
| NORMAL | 8 hours | Notify PR author | Escalate to team lead |

---

## 4. Batch CasePlanModel — `merge-batch.yaml`

### 4.1 Capabilities

Each capability declares `inputSchema` (JQ over case context → worker input) and `outputSchema` (JQ over worker response → context update). This is the established pattern from `pr-review.yaml`.

```yaml
capabilities:
  - name: batch-ci-runner
    description: "Tests the tip-of-batch against the target branch"
    inputSchema: "{ batch: .batch }"
    outputSchema: "{ tipTest: . }"

  - name: merge-executor
    description: "Executes the merge when all goals are satisfied"
    inputSchema: "{ batch: .batch }"
    outputSchema: "{ mergeResult: . }"

  - name: bisection-splitter
    description: "Computes the trust-weighted bisection split"
    inputSchema: "{ prs: .batch.prs, strategy: (.batch.bisectionStrategy // \"trust-weighted\") }"
    outputSchema: "{ splitResult: . }"

  - name: pr-reject-and-notify
    description: "Rejects a single PR and notifies the author"
    inputSchema: "{ pr: .batch.prs[0], reason: .tipTest.failureReason, batchId: .batch.id }"
    outputSchema: "{ rejectedPrs: [.pr] }"
```

### 4.2 Context Schema

```json
{
  "batch": {
    "id": "batch-20260626-001",
    "targetBranch": "main",
    "prs": [
      { "number": 456, "headSha": "abc123", "author": "alice", "trustScore": 0.85 },
      { "number": 457, "headSha": "def456", "author": "bob", "trustScore": 0.72 }
    ],
    "size": 2,
    "parentBatchId": null,
    "bisectionDepth": 0,
    "bisectionStrategy": "trust-weighted",
    "riskLevel": "ROUTINE"
  },
  "tipTest": null,
  "splitResult": null,
  "mergeResult": null,
  "bisectLeft": null,
  "bisectRight": null,
  "mergeApproval": null,
  "mergeEscalation": null,
  "mergeEscalatedRetry": null,
  "tipTestEscalation": null,
  "tipTestEscalatedRetry": null,
  "rejectedPrs": [],
  "mergedPrs": []
}
```

### 4.3 Goals

```yaml
goals:
  # ── Success goals ──
  - name: batch-merged
    kind: success
    condition: '.mergeResult.status == "success"'

  - name: all-culprits-isolated
    kind: success
    condition: '.bisectLeft != null and .bisectRight != null'

  - name: single-pr-rejected
    kind: success
    condition: '.batch.size == 1 and (.rejectedPrs | length) > 0'

  # ── Failure goals ──
  - name: merge-approval-rejected
    kind: failure
    condition: '.mergeApproval.outcome == "REJECTED" or .mergeApproval.outcome == "BLOCKED"'

  - name: merge-terminal-failure
    kind: failure
    condition: '.mergeEscalation.outcome == "REJECTED" or .mergeEscalation.outcome == "BLOCKED"'

  - name: tip-test-terminal-failure
    kind: failure
    condition: '.tipTestEscalation.outcome == "REJECT_BATCH" or .tipTestEscalation.outcome == "BLOCKED"'

completion:
  success:
    anyOf: [batch-merged, all-culprits-isolated, single-pr-rejected]
  failure:
    anyOf: [merge-approval-rejected, merge-terminal-failure, tip-test-terminal-failure]
```

Three success paths: whole batch merges clean; bisection fully resolves both halves; or a single-PR batch identifies and rejects the culprit. `single-pr-rejected` is critical — every leaf node of the bisection tree is a batch of 1, and without this goal those sub-cases hang indefinitely, blocking the entire bisection tree.

Three failure paths: human rejects the high-risk merge approval; merge executor exhausted + human rejected/expired; or CI test exhausted + human rejected/expired. All humanTask SLA expirations write `BLOCKED` (per the failure-cascade protocol's `SlaBreachHandler`), so every failure goal checks both the explicit rejection outcome AND `BLOCKED`. M-of-N group rejection (bisection sub-case faults) is handled by the engine directly (`SubCaseGroupPolicy.REJECTED` → parent cancelled) — no goal needed.

### 4.4 Bindings

```yaml
bindings:

  ## ── Tier 1: Entry — test the tip of the batch ──

  - name: test-batch-tip
    on: { contextChange: {} }
    when: ".batch.size > 0 and .tipTest == null and .tipTestEscalatedRetry != true"
    capability: batch-ci-runner
    outcomePolicy:
      onDecline: REROUTE
      onFailure: REROUTE
      maxRerouteAttempts: 2

  ## ── Tier 1: Tip-test escalation — CI runners exhausted ──

  - name: tip-test-escalation
    on: { contextChange: {} }
    when: '.tipTest.status == "REROUTES_EXHAUSTED" and .tipTestEscalation == null'
    humanTask:
      title: "Batch CI test failed — all runners exhausted"
      candidateGroups: [repo-maintainers]
      expiresIn: PT2H
      outputMapping: "{ tipTestEscalation: . }"
      outcomes: [RETRY, REJECT_BATCH, BLOCKED]

  - name: tip-test-after-escalation
    on: { contextChange: {} }
    when: '.tipTestEscalation.outcome == "RETRY" and .tipTest.status == "REROUTES_EXHAUSTED"'
    contextWrite:
      tipTest: null
      tipTestEscalatedRetry: true
    capability: batch-ci-runner
    outcomePolicy:
      onDecline: FAULT
      onFailure: FAULT

  ## ── Tier 2: Success path — merge ──

  - name: merge-batch
    on: { contextChange: {} }
    when: >-
      .tipTest.status == "passing" and
      .mergeResult == null and
      .mergeEscalatedRetry != true and
      (.batch.riskLevel == "ROUTINE" or .batch.riskLevel == "ELEVATED" or
       .mergeApproval.outcome == "APPROVED")
    capability: merge-executor
    outcomePolicy:
      onDecline: REROUTE
      onFailure: REROUTE
      maxRerouteAttempts: 3

  ## ── Tier 2: Human oversight for high-risk batches ──

  - name: human-merge-approval
    on: { contextChange: {} }
    when: >-
      .tipTest.status == "passing" and
      .mergeResult == null and
      .mergeApproval == null and
      (.batch.riskLevel == "HIGH" or .batch.riskLevel == "CRITICAL")
    humanTask:
      title: "High-risk batch merge approval required"
      candidateGroups: [repo-maintainers]
      expiresIn: PT2H
      outputMapping: "{ mergeApproval: . }"
      outcomes: [APPROVED, REJECTED, BLOCKED]

  ## ── Tier 2: Merge escalation — reroutes exhausted ──

  - name: merge-escalation
    on: { contextChange: {} }
    when: '.mergeResult.status == "REROUTES_EXHAUSTED" and .mergeEscalation == null'
    humanTask:
      title: "Merge execution failed after 3 retries"
      candidateGroups: [repo-maintainers]
      expiresIn: PT1H
      outputMapping: "{ mergeEscalation: . }"
      outcomes: [APPROVED, REJECTED, BLOCKED]

  - name: merge-after-escalation
    on: { contextChange: {} }
    when: '.mergeEscalation.outcome == "APPROVED" and .mergeResult.status == "REROUTES_EXHAUSTED"'
    contextWrite:
      mergeResult: null
      mergeEscalatedRetry: true
    capability: merge-executor
    outcomePolicy:
      onDecline: FAULT
      onFailure: FAULT

  ## ── Tier 3: Failure path — bisection ──

  - name: compute-bisection-split
    on: { contextChange: {} }
    when: '.tipTest.status == "failing" and .batch.size > 1 and .splitResult == null'
    capability: bisection-splitter
    outcomePolicy:
      onDecline: FAULT
      onFailure: FAULT

  - name: bisect-left
    on: { contextChange: {} }
    when: '.splitResult != null and .bisectLeft == null'
    subCase:
      namespace: devtown
      name: merge-batch
      version: "1.0.0"
      maxRecursionDepth: 10
      waitForCompletion: true
      inputMapping: "{ batch: .splitResult.left }"
      outputMapping: "{ bisectLeft: . }"
      groupId: "bisect"
      totalInGroup: 2
      requiredCount: 2
      onThresholdReached: KEEP

  - name: bisect-right
    on: { contextChange: {} }
    when: '.splitResult != null and .bisectRight == null'
    subCase:
      namespace: devtown
      name: merge-batch
      version: "1.0.0"
      maxRecursionDepth: 10
      waitForCompletion: true
      inputMapping: "{ batch: .splitResult.right }"
      outputMapping: "{ bisectRight: . }"
      groupId: "bisect"
      totalInGroup: 2
      requiredCount: 2
      onThresholdReached: KEEP

  ## ── Tier 3: Single PR failed → reject ──

  - name: reject-single-pr
    on: { contextChange: {} }
    when: '.tipTest.status == "failing" and .batch.size == 1 and (.rejectedPrs | length) == 0'
    capability: pr-reject-and-notify
```

**Key design decisions:**

1. **No `workerContext`** — workers receive context through the capability's `inputSchema` (JQ over case context). This is the established engine pattern.

2. **Separate output keys for bisection halves** — `bisectLeft` and `bisectRight` avoid the LAST_WRITER_WINS collision that would occur with a shared key. Goal condition checks both.

3. **Merge retry via `outcomePolicy`** — `onFailure: REROUTE, maxRerouteAttempts: 3` handles retries natively. No manual retry counter. After 3 failures, `REROUTES_EXHAUSTED` fires the `merge-escalation` humanTask binding. Same pattern as pr-review.yaml.

4. **`onDecline: REROUTE` on merge-batch** — the merge queue operates across repos with different access models. An executor might DECLINE because it lacks credentials for a specific repo; REROUTE tries another executor. Safety preserved: `TrustWeightedAgentStrategy` enforces the 0.80 threshold regardless.

5. **Escalation recovery via `contextWrite` reset** — when a human approves after reroute exhaustion (merge or tip-test), a dedicated `*-after-escalation` binding resets the exhausted result via `contextWrite` and re-dispatches with `onFailure: FAULT`. One post-escalation chance — if it still fails, the problem is systemic. Guard flags (`mergeEscalatedRetry`, `tipTestEscalatedRetry`) prevent the original bindings from re-firing. This follows the pr-review.yaml `security-review-reduced-scope` pattern.

6. **`single-pr-rejected` success goal** — every leaf node of the bisection tree is a batch of 1. Without this goal, leaf sub-cases that reject a PR hang indefinitely, blocking the M-of-N group, which blocks the parent, which blocks the entire bisection tree.

7. **M-of-N grouped sub-cases for parallel bisection** — `groupId: "bisect"`, `totalInGroup: 2`, `requiredCount: 2`. Both halves spawn simultaneously, parent enters WAITING on the group, resumes when both complete. Each child's `outputMapping` writes to its own key (requires engine#574 per-child outputMapping fix).

8. **No individual-test-each-PR optimization** — bisection handles all batch sizes ≥ 2. For batch size 2, bisection degenerates to individual testing (split into [1] and [1], 1 round). The IsolateOutlierStrategy achieves the same culprit-isolation effect for the most likely case. The architectural cost of array fan-out (new engine primitive) is not justified by saving ~1 CI round on rare small-batch failures.

9. **Bisection outcome aggregation is a queue-service responsibility** — the bisection tree produces nested results (`bisectLeft.mergeResult`, `bisectRight.bisectLeft.rejectedPrs`, etc.). `MergeQueueService` walks the tree on case completion to produce aggregated `mergedPrs`/`rejectedPrs` for the `MergeDecisionLedgerEntry` and MCP tools (`get_batch_status`). Aggregation in JQ outputMappings would be fragile at recursive depth.

10. **All bindings use only schema-valid fields** — verified against `CaseDefinition.yaml` Binding schema (`unevaluatedProperties: false`). Bindings carry: `name`, `on`, `when`, `capability`/`subCase`/`humanTask`, `conflictResolverStrategy`, `outcomePolicy`, `inputSchemaOverride`, `contextWrite`.

---

## 5. Trust-Weighted Bisection

### 5.1 The Problem with Mechanical Bisection

Bors/Gastown splits the batch in half mechanically — positions 1-5 in one half, 6-10 in the other. The split uses no information about which half is more likely to contain the culprit. Expected CI runs to isolate: exactly log₂(N) rounds regardless.

### 5.2 Trust-Weighted Splitting

Every PR author has a trust score from Layer 6 (Bayesian Beta model, per-capability). A low-trust author's PR is statistically more likely to be the culprit. By sorting PRs by trust before splitting, the low-trust PRs cluster in one sub-batch.

**Worked example — batch of 8, culprit is PR-C (trust 0.35):**

```
PRs sorted by trust:
  PR-C (0.35), PR-F (0.52), PR-A (0.71), PR-H (0.74),
  PR-B (0.82), PR-D (0.85), PR-E (0.91), PR-G (0.95)

Round 1: split → [C, F, A, H] vs [B, D, E, G]
  Left fails → bisect further
  Right passes → merge [B, D, E, G] immediately

Round 2: split → [C, F] vs [A, H]
  Left fails → bisect further
  Right passes → merge [A, H]

Round 3: [C] fails → rejected. [F] passes → merged.
```

Trust-weighted bisection has better **expected** performance because the culprit is statistically more likely to be in the low-trust half. With multiple culprits, trust-weighted bisection clusters them in the same sub-batch, avoiding the scenario where mechanical bisection splits culprits across halves and requires both sub-trees to fully recurse.

### 5.3 Pluggable Split Strategies

The split is computed by a `bisection-splitter` capability that delegates to a `BisectionSplitStrategy` SPI:

```java
public interface BisectionSplitStrategy {
    SplitResult split(List<QueuedPr> failingBatch, BisectionContext ctx);
}

public record SplitResult(BatchSlice left, BatchSlice right) {}
```

Three implementations, selected per repo via PreferenceProvider:

| Strategy | Behaviour | When to use |
|----------|-----------|-------------|
| `TrustWeightedSplitStrategy` | Sort by trust, split at midpoint | Default — clusters likely culprits |
| `IsolateOutlierStrategy` | If one PR's trust is >2σ below mean, isolate solo; otherwise trust-weighted | Repos with a mix of trusted and untrusted contributors |
| `BinarySplitStrategy` | Positional split, no trust | Benchmarking, or repos where trust data is sparse |

The split result goes on the context — fully auditable. `get_causal_chain` shows which strategy was used and exactly how the batch was divided at each level.

---

## 6. Foundation Gates

### 6.1 engine#573: Bounded Recursive Sub-Case Spawning

The current engine blocks a case from spawning sub-cases with the same namespace/name/version (circular guard in `SubCaseExecutionHandler`). This prevents the recursive bisection model.

**Change:** Replace the hard block with a bounded depth limit via `maxRecursionDepth` field on `SubCase`:

- `maxRecursionDepth: 0` (default) — preserves current behaviour, hard block on self-reference
- `maxRecursionDepth: N` — allows N levels of self-referencing, faults at the limit

Depth is computed by walking the parent chain. ~30 lines across 5 files, fully backward compatible.

### 6.2 engine#574: M-of-N Sub-Case Groups in YAML + Per-Child OutputMapping

Two gaps in a single issue:

**Part 1 — Schema/mapper:** Add `groupId`, `totalInGroup`, `requiredCount`, `onThresholdReached` to the YAML `SubCase` schema and `CaseDefinitionYamlMapper.convertSubCase()`. The runtime already fully supports M-of-N groups via the Java API. ~25 lines.

**Part 2 — Per-child outputMapping:** Fix `SubCaseCompletionService.handleGroupedCompletion()` to apply each child's `outputMapping` when it completes, not only the threshold-triggering child's. The threshold still controls parent resumption (WAITING → RUNNING), but data flows incrementally. ~10 lines.

Without this fix, `bisect-left` and `bisect-right` writing to separate keys (`bisectLeft`, `bisectRight`) would lose the first child's result.

---

## 7. Integration Points

### 7.1 PR Review Case → Merge Queue

The `merge` binding in `pr-review.yaml` gains a queue-aware variant:

```yaml
- name: enqueue-for-merge
  on: { contextChange: {} }
  when: "... all conditions met ... and .policy.mergeQueueEnabled == true"
  capability: merge-queue-enqueue

- name: merge-direct
  on: { contextChange: {} }
  when: "... all conditions met ... and .policy.mergeQueueEnabled != true"
  capability: merge-executor
```

Mutually exclusive via `policy.mergeQueueEnabled` preference. Zero behaviour change for repos without a merge queue.

### 7.2 GitHub Webhooks

| Event | Action |
|-------|--------|
| `pull_request.labeled` (`merge-ready`) | Enqueue if not managed by PR Review Case |
| `check_suite.completed` | Report CI result to batch case context |

### 7.3 Batch Branch Management

The `batch-ci-runner` worker creates a temporary merge branch for batch testing:

1. Start from `targetBranch` HEAD
2. Merge each PR's head SHA in batch order (dependency-respecting)
3. Push as `merge-queue/batch-{id}`
4. CI runs against that branch
5. On merge success: fast-forward `targetBranch`
6. Clean up temporary branch

### 7.4 Ledger Integration

Every merge writes a `MergeDecisionLedgerEntry` enriched with batch metadata:

- Batch ID and PR list
- Whether bisection occurred and which strategy was used
- Trust scores at time of decision
- CI run IDs
- Rejected PRs (if any)

### 7.5 Post-Merge Trust Feedback

Incident linked to merged PR → find the batch → identify reviewers → write FLAGGED attestation → trust score degrades → future batch formation uses updated score. The existing Layer 6 loop handles this without changes.

---

## 8. MCP Tools

### 8.1 New Tools

**Read:**

| Tool | Returns |
|------|---------|
| `get_merge_queue` | Queued PRs with priority scores, wait times, dependencies, SLA status |
| `get_batch_status` | Batch PRs, test result, bisection tree, merge result |
| `get_merge_queue_metrics` | Throughput, failure rate, average wait time, batch size distribution |

**Write:**

| Tool | Guards |
|------|--------|
| `enqueue_pr` | Validates PR exists, not already queued, approval status |
| `dequeue_pr` | Notifies PR author, removes SLA WorkItem, cascades to dependents |

### 8.2 Existing Tool Enhancements

- `list_problems` gains `QUEUE_SLA_BREACH` problem type
- `get_recent_events` gains merge queue events: `BATCH_FORMED`, `BATCH_MERGED`, `BATCH_BISECTED`, `PR_REJECTED`, `PR_ENQUEUED`

---

## 9. Testing Strategy

### 9.1 Unit Tests (devtown-merge)

| Test | Covers |
|------|--------|
| `QueuePriorityCalculatorTest` | Composite scoring, decay coefficient, starvation prevention |
| `BatchCompositionPolicyTest` | Trust-weighted sizing, adaptive sizing with floor, dependency ordering |
| `DependencyResolverTest` | DAG construction, cycle detection, cascade dequeue |
| `BisectionSplitStrategyTest` | Trust-weighted, outlier isolation, binary split |
| `MergeBatchBindingConditionTest` | All binding conditions from the CasePlanModel |

### 9.2 Integration Tests (@QuarkusTest, devtown-app)

| Test | Covers |
|------|--------|
| `MergeQueueBatchLifecycleTest` | Happy path: enqueue → batch → tip test → merge → ledger entry |
| `MergeQueueBisectionTest` | Failure path: batch fails → split → recursive sub-cases → culprit isolated → innocent merged. Separate output keys (`bisectLeft`, `bisectRight`) both populated. Full causal chain verification. |
| `MergeQueueSlaBreachTest` | SLA breach → notification → escalation |
| `MergeQueueDependencyTest` | Dependency ordering, cascade dequeue on rejection |
| `MergeQueuePriorityTest` | CRITICAL preempts NORMAL. Starvation: NORMAL overtakes stale HIGH. |
| `MergeQueueHumanOversightTest` | High-risk batch → human approval → merge |
| `MergeQueueMergeEscalationTest` | Merge reroutes exhausted → human escalation → approve → contextWrite reset → retry → success/fault |
| `MergeQueueTipTestEscalationTest` | CI reroutes exhausted → human escalation → RETRY (contextWrite reset, re-dispatch, FAULT on failure) or REJECT_BATCH (terminal failure goal) |
| `MergeQueueSinglePrRejectionTest` | Batch of 1 fails tip test → reject-single-pr → single-pr-rejected goal fires → sub-case completes → parent M-of-N group sees completion |

### 9.3 Not in Scope

- GitHub webhook integration (separate connector test)
- Actual git operations (Claudony worker integration tests)
- Cross-repo coordination (Epic #12)

---

## 10. Structural Improvements Over Gastown

Each requires a structural rewrite of Gastown to match — not bolt-on features.

| # | Capability | Gastown | CaseHub | Why it can't be bolted on |
|---|-----------|---------|---------|--------------------------|
| 1 | Strategy as data | Go code, deploy to change | CasePlanModel YAML, change per repo at runtime | Requires separating strategy from implementation |
| 2 | Trust-weighted batch composition | FIFO, batch blindly | Batch size ∝ min trust in batch; low-trust solo or small batch | Requires a trust model feeding into batch formation |
| 3 | Trust-weighted bisection | Mechanical binary split | Split by trust score — isolate likely culprits first | Requires trust scores at bisection time |
| 4 | Priority lanes with starvation prevention | FIFO only | Composite score: lane × 1000 + trust × 100 + time-decay | Requires a priority queue with decay |
| 5 | Dependency-aware ordering | Manual author coordination | DAG from explicit labels + git base-branch analysis | Requires a dependency resolver in the queue |
| 6 | SLA-bounded queue wait | No concept of queue wait time | WorkItem per queued PR, tiered SLA, escalation | Requires casehub-work's obligation model |
| 7 | Adaptive batch sizing | Static config | Batch size × (1 - recentFailureRate) × trustFactor, floor of 1 | Requires a feedback loop from batch outcomes |
| 8 | Cryptographic merge audit | Logs | MergeDecisionLedgerEntry in Merkle chain | Requires casehub-ledger |
| 9 | Human oversight for high-risk merges | Fire-and-forget | ActionRiskClassifier gates ELEVATED/HIGH/CRITICAL | Requires risk classification + WorkItem |
| 10 | Recursive auditable bisection | Opaque binary search | Every bisection level in EventLog with causal chain | Requires recursive CasePlanModel sub-cases |

---

## 11. Configuration Summary

All configuration via PreferenceProvider, scoped per repo via `Path`-based hierarchy:

| Key | Default | Description |
|-----|---------|-------------|
| `devtown.merge-queue.enabled` | `false` | Enable merge queue for this repo |
| `devtown.merge-queue.max-batch-size` | `10` | Maximum batch size before trust/adaptive reduction |
| `devtown.merge-queue.min-batch-size` | `1` | Floor for adaptive sizing — never drops below this |
| `devtown.merge-queue.bisection-strategy` | `trust-weighted` | Split strategy: `trust-weighted`, `isolate-outlier`, `binary` |
| `devtown.merge-queue.failure-rate-window` | `20` | Number of recent batches for adaptive sizing |
| `devtown.merge-queue.priority.decay-rate-per-hour` | `125` | Starvation prevention: NORMAL overtakes fresh HIGH at ~8h |
| `devtown.merge-queue.sla.critical` | `PT1H` | SLA for CRITICAL priority PRs |
| `devtown.merge-queue.sla.high` | `PT4H` | SLA for HIGH priority PRs |
| `devtown.merge-queue.sla.normal` | `PT8H` | SLA for NORMAL priority PRs |
| `devtown.merge-queue.max-recursion-depth` | `10` | Maximum bisection depth |

---

## Appendix: Review History

- **v1 (2026-06-26):** Initial design — 8 sections approved in brainstorming walkthrough.
- **v2 (2026-06-26):** Schema alignment review. Fixes: replaced invented `workerContext` with capability `inputSchema`/`outputSchema`; separated bisection output keys (`bisectLeft`/`bisectRight`); replaced manual retry counter with `outcomePolicy` reroute; dropped individual-test optimization (bisection handles all sizes); added foundation gate engine#574 (M-of-N YAML + per-child outputMapping); specified priority decay coefficient (125/hr); added adaptive batch sizing floor (min 1).
- **v3 (2026-06-26):** Adversarial review. Fixes: added `single-pr-rejected` success goal (leaf nodes of bisection tree hang without it); added `merge-after-escalation` and `tip-test-after-escalation` bindings with `contextWrite` reset (dead-end after human approval); added `tip-test-escalation` humanTask for CI reroute exhaustion; expanded failure goals (`merge-terminal-failure`, `tip-test-terminal-failure`); changed `onDecline: FAULT` to `onDecline: REROUTE` on merge-batch for multi-repo flexibility; documented bisection outcome aggregation as queue-service responsibility.
- **v4 (2026-06-26):** HumanTask outcome coverage review. Fixes: added `merge-approval-rejected` failure goal for REJECTED/BLOCKED on high-risk merge approval; expanded `merge-terminal-failure` and `tip-test-terminal-failure` to include BLOCKED (SLA expiration); added BLOCKED to all humanTask outcomes arrays; added `== null` guards on `tip-test-escalation` and `merge-escalation` bindings to prevent re-fire; added `(.rejectedPrs | length) == 0` guard on `reject-single-pr`.
- **v5 (2026-06-26):** Exhaustive state transition analysis (15 paths verified). Fix: added `onDecline: FAULT, onFailure: FAULT` on `compute-bisection-split` — local deterministic computation has nothing to reroute to; system error should fault the case.
