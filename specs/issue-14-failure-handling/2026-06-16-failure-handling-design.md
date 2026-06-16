# Failure Handling — Declarative DECLINED vs FAILED Routing

**Epic:** casehubio/devtown#14
**Date:** 2026-06-16
**Status:** Blocked by casehubio/engine#501

---

## 1. Problem

The PR review case has bindings that fire once and check `== null` — if a worker DECLINES or FAILS, the blackboard key stays null and the binding never re-fires. There is no recovery path. The case hangs indefinitely waiting for a review that will never come.

The engine handles worker *exceptions* (RetryPolicy, backoff, WORKER_RETRIES_EXHAUSTED → FAULTED) but has no mechanism for semantic outcomes where the worker *chooses* to decline or reports failure via Qhorus speech acts.

---

## 2. Design Decisions

**Approach:** Hybrid (Approach C) — tier structure in declarative bindings, policy parameters in descriptors. The fluent DSL companion generates failure bindings programmatically per capability from the descriptor's `FailurePolicy`.

**Re-routing model:** Same capability + agent exclusion. The declined/failed agent ID is written to `excludedAgents` on the blackboard. The engine's `AgentRoutingStrategy` (which already receives `CaseContext` via `AgentRoutingContext`) reads this list and filters candidates. No separate backup capabilities.

**Terminal failure:** Failure goals produce `CaseStatus.COMPLETED` with failure outcome metadata — not `FAULTED`. FAULTED is reserved for system errors. A review that exhausted all tiers is a legitimate process conclusion.

**Scope reduction:** Generic mechanism — any capability can declare `scopeReductionAllowed: true` and provide a `reducedInputSchema` that narrows the input for tier 3 dispatch.

**Protocol:** The four-tier failure cascade is a repeatable pattern captured as `failure-cascade-pattern.md` in `casehub/garden/docs/protocols/casehub/`.

---

## 3. Blackboard Failure State Schema

When a worker DECLINES or FAILS, structured failure state is written to the blackboard under the capability's key.

**Initial dispatch (existing, unchanged):**
```
securityReview: null
```

**On success (existing, unchanged):**
```json
{ "securityReview": { "outcome": "APPROVED" } }
```

**On DECLINED or FAILED (new):**
```json
{
  "securityReview": {
    "status": "DECLINED",
    "attempts": 1,
    "history": [
      { "agent": "agent-a", "status": "DECLINED", "reason": "unsupported language", "timestamp": "2026-06-16T10:00:00Z" }
    ],
    "excludedAgents": ["agent-a"]
  }
}
```

**On successful retry after previous failure:**
```json
{
  "securityReview": {
    "status": "COMPLETED",
    "outcome": "APPROVED",
    "attempts": 2,
    "history": [
      { "agent": "agent-a", "status": "DECLINED", "reason": "unsupported language", "timestamp": "..." },
      { "agent": "agent-b", "status": "COMPLETED", "timestamp": "..." }
    ],
    "excludedAgents": ["agent-a"]
  }
}
```

**On scope reduction (tier 3):**
```json
{
  "securityReview": {
    "status": "FAILED",
    "attempts": 2,
    "reducedScope": true,
    "history": [ "..." ],
    "excludedAgents": ["agent-a", "agent-b"]
  }
}
```

**Key design choices:**
- `status` is the current state; `outcome` only appears on completion
- `history` is append-only — full audit trail
- `excludedAgents` accumulates across attempts
- `reducedScope: true` signals narrowed input for next dispatch
- Existing goal conditions (`securityReview.outcome == "APPROVED"`) still work — they check `outcome`, not `status`

---

## 4. Failure Policy and Descriptor

Each capability declares its failure handling behavior via `FailurePolicy`:

```java
public record FailurePolicy(
    int maxRerouteAttempts,
    boolean scopeReductionAllowed,
    String reducedInputSchema,
    Duration humanEscalationSla,
    String failureGoal
) {}
```

In `PrReviewCaseDescriptor`:

| Capability | maxReroute | scopeReduction | reducedInputSchema | humanSla | failureGoal |
|-----------|-----------|----------------|-------------------|---------|-------------|
| security-review | 2 | yes | `{ flaggedFiles: .codeAnalysis.flaggedFiles }` | 4h | review-blocked |
| architecture-review | 2 | yes | `{ crossingPoints: .codeAnalysis.crossingPoints }` | 4h | review-blocked |
| style-review | 1 | no | — | 2h | review-blocked |
| test-coverage | 1 | no | — | 2h | review-blocked |
| performance-analysis | 1 | no | — | 2h | review-blocked |
| code-analysis | 1 | no | — | 1h | review-blocked |
| ci-runner | 2 | no | — | 1h | review-blocked |

Security and architecture reviews get 2 re-route attempts and scope reduction — high-value, worth multiple tries. Style/test/performance get 1 attempt then human escalation. CI runner gets 2 attempts (infra flakiness) but no scope reduction (all-or-nothing).

---

## 5. Failure Tier Bindings

Four tiers per capability, generated programmatically by `PrReviewCaseDefinition.build()`:

### Tier 1 — DECLINED re-route

Agent said "I can't do this" (capability boundary). Immediately dispatch to a different agent.

```
when: status == "DECLINED" AND attempts < maxRerouteAttempts AND reducedScope == null
action: re-dispatch same capability; excludedAgents list tells routing to skip decliners
```

### Tier 2 — FAILED retry

Agent tried but couldn't complete (execution error). Same routing as tier 1, different trust scoring consequence — DECLINED feeds `scope-calibration`, FAILED feeds `review-thoroughness`.

```
when: status == "FAILED" AND attempts < maxRerouteAttempts AND reducedScope == null
action: re-dispatch same capability with exclusion
```

### Tier 3 — Scope reduction

Multiple agents have failed. Narrow the input and try once more. Only available when `scopeReductionAllowed: true`.

```
when: attempts >= maxRerouteAttempts AND scopeReductionAllowed AND reducedScope == null AND outcome == null
action: dispatch same capability with reducedInputSchema; write reducedScope: true; reset excludedAgents
```

Excluded agents list is reset to empty — a previously-declined agent might handle the narrower scope successfully. The rationale: a DECLINE for "I can't review 500 files of Rust" doesn't imply "I can't review 3 flagged Rust files."

### Tier 4 — Human escalation

Everything automated has been exhausted. Create a WorkItem.

```
when: (attempts >= maxRerouteAttempts AND NOT scopeReductionAllowed)
   OR (reducedScope == true AND outcome == null AND status in [DECLINED, FAILED])
action: create WorkItem via HumanTaskTarget with humanEscalationSla
```

The WorkItem carries the full `history` from the blackboard so the human reviewer sees what was tried and why.

Human outcomes: APPROVED (override), REJECTED (PR should not merge), BLOCKED (can't resolve → triggers failure goal).

---

## 6. Failure Goals

Three failure goals added to the case definition:

### review-blocked

At least one required capability exhausted all tiers. The tier 4 human WorkItem completed with BLOCKED outcome.

### review-timed-out

Case-level SLA expired before all goals were reached. Written to blackboard as `caseSla.expired: true` by `SlaBreachPolicyBean`.

### review-abandoned

PR was closed or superseded externally. Written as `pr.status: "closed"` via external signal (REST call for now; GitHub webhook in Epic #15).

### Completion expression

```java
.completion(
    GoalExpression.allOf(prApproved, securityVerified, ciPassing),
    GoalExpression.anyOf(reviewBlocked, reviewTimedOut, reviewAbandoned)
)
```

Uses the existing two-argument `CaseDefinition.builder().completion(success, failure)` form. Engine evaluates failure before success — if both are satisfied simultaneously, failure wins.

---

## 7. ReviewerOutcome Changes

```java
public sealed interface ReviewerOutcome
        permits ReviewerOutcome.Completed, ReviewerOutcome.Declined, ReviewerOutcome.Failed {

    record Completed(List<String> findings) implements ReviewerOutcome {}
    record Declined(String reason)          implements ReviewerOutcome {}
    record Failed(String reason, boolean transient_) implements ReviewerOutcome {}
}
```

`transient_` distinguishes retryable failures (network timeout) from permanent failures (corrupt file). Maps to `OutcomePolicy`: `transient_=true` → `RETRY_SAME`, `transient_=false` → `REROUTE`.

`QhorusPrReviewService` handles the new variant with `MessageType.FAILURE`.

---

## 8. Engine Dependencies (Blocking)

**Epic:** casehubio/engine#501 — Semantic failure routing

| Issue | What | Scale | Depends on |
|-------|------|-------|------------|
| engine#502 | Agent exclusion — read `excludedAgents` from case context in routing | S / Low | — |
| engine#503 | Semantic DECLINE/FAIL — write structured failure state to blackboard | L / High | — |
| engine#504 | OutcomePolicy — retry/re-route for speech-act outcomes | L / High | #503 |
| engine#506 | Failure goals → COMPLETED not FAULTED | S / Med | — |

**Already exists (no engine change needed):**
- `GoalKind.FAILURE` enum value
- `GoalBasedCompletion(success, failure)` constructor
- `CaseDefinition.builder().completion(success, failure)`
- `AgentRoutingContext.caseContext()` carries blackboard to routing strategy

---

## 9. Adaptive Learning (Future, Not Blocking)

**Epic:** casehubio/parent#258 — Adaptive agent routing from DECLINE/FAIL patterns

| Issue | What | Repo |
|-------|------|------|
| eidos#55 | Capability sub-specialization metadata | casehubio/eidos |
| engine#505 | Routing with eidos metadata + CBR patterns | casehubio/engine |

Cross-case learning: Agent A declines Rust security reviews three times → system learns not to route Rust security reviews to Agent A. Current case-level exclusion (engine#502) is the prerequisite; adaptive learning extends it across cases.

---

## 10. Protocol: Failure Cascade Pattern

To be written as `casehub/garden/docs/protocols/casehub/failure-cascade-pattern.md`. Captures:

- DECLINED vs FAILED as distinct outcomes with different routing responses
- Four-tier failure cascade as the standard pattern
- FailurePolicy on the descriptor as the configuration mechanism
- Blackboard failure state schema
- Programmatic binding generation via fluent DSL
- Failure goals as legitimate case outcomes distinct from FAULTED

---

## 11. Goal Semantics (Platform Coherence Note)

A CaseHub Goal is a **named predicate over the blackboard that recognises an outcome**. Goals are passive observers — they evaluate state, they don't cause state changes.

| Concept | What it does |
|---------|-------------|
| Binding | Watches the blackboard, triggers action (dispatch worker, create WorkItem) |
| Goal | Watches the blackboard, recognises an outcome (success or failure) |
| Completion | Combines goals — determines when the case is done |

Goals don't care how the blackboard reached a state. Bindings fire adaptively based on content analysis, agent availability, and failure history. Goals evaluate the result. This is the ACM distinction from workflow: you reach the end when goals are satisfied, not when steps are completed.

The `GoalKind` enum (SUCCESS, FAILURE) distinguishes positive from negative outcomes. Both result in `CaseStatus.COMPLETED` — the goal kind is outcome metadata, not a different terminal state. `FAULTED` is reserved for system errors (infrastructure failures, exhausted retries at the engine level).

All Goal instances are type-safe — searchable by type across the codebase. Coherence review of Goal usage across the platform is deferred but tractable.

---

## 12. What devtown Implements (after engine#501 ships)

**Domain model:**
- `ReviewerOutcome.Failed` sealed variant
- `FailurePolicy` record
- `PrReviewCaseDescriptor` with `FAILURE_POLICIES` map

**Case definition (`PrReviewCaseDefinition`):**
- Three failure goals
- Two-arg completion expression
- Programmatic failure tier bindings per capability

**Qhorus integration (`QhorusPrReviewService`):**
- Handle `ReviewerOutcome.Failed` → `MessageType.FAILURE`

**Protocol:**
- `failure-cascade-pattern.md`
