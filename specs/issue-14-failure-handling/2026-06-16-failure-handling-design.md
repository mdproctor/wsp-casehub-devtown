# Failure Handling — Declarative DECLINED vs FAILED Routing

**Epic:** casehubio/devtown#14
**Date:** 2026-06-16 (revised 2026-06-17, rev4)
**Status:** Blocked by casehubio/engine#501

---

## 1. Problem

The PR review case has bindings that fire once and check `== null` — if a worker DECLINES, FAILS, or goes silent (EXPIRED), the blackboard key stays null and the binding never re-fires. There is no recovery path. The case hangs indefinitely waiting for a review that will never come.

The engine handles worker *exceptions* (RetryPolicy, backoff, WORKER_RETRIES_EXHAUSTED → FAULTED) but has no mechanism for semantic outcomes where the worker *chooses* to decline, reports failure via Qhorus speech acts, or simply goes silent until the commitment deadline passes.

Three distinct outcomes require different routing:

| Outcome | Meaning | Routing response |
|---------|---------|-----------------|
| DECLINED | Capability boundary — agent is healthy, cannot do this work | Immediate re-route to different agent |
| FAILED | Execution error — agent tried but could not complete | Re-route to different agent |
| EXPIRED | Agent went silent — commitment deadline passed | Re-route + investigation side-effect |

---

## 2. Design Decisions

**Approach:** Hybrid (Approach C) — engine owns the reroute loop (Tiers 1-2) via OutcomePolicy; application owns domain-specific responses (Tiers 3-4) via declarative bindings. The fluent DSL companion generates Tier 3/4 bindings programmatically per capability from the descriptor's `FailurePolicy`.

**Engine vs application boundary:** The engine handles the mechanical reroute loop — incrementing attempts, accumulating excludedAgents, writing failure state to the blackboard, re-dispatching with agent exclusion. This is generic infrastructure every harness needs. The application handles what happens when reroutes are exhausted — scope reduction and human escalation are domain-specific.

**Re-routing model:** Same capability + agent exclusion. The engine writes `excludedAgents` to the blackboard. `AgentRoutingStrategy` already receives `AgentRoutingContext.caseContext()` as a JsonNode — the routing strategy reads `excludedAgents` from the capability's blackboard key and filters candidates. No SPI signature change needed.

**Terminal failure:** Failure goals produce `CaseStatus.COMPLETED` with failure outcome metadata — not `FAULTED`. FAULTED is reserved for system errors. A review that exhausted all tiers is a legitimate process conclusion.

**Scope reduction:** Generic mechanism — any capability can declare `scopeReductionAllowed: true` and provide a `reducedInputSchema`. Uses `Binding.inputSchemaOverride` (engine#509) to dispatch the same capability with narrowed input — same agent qualification, same trust scoring path. Uses `Binding.contextWrite` (engine#511) to write `reducedScope: true` and reset `status` to `PENDING` before dispatch, preventing infinite re-fire and Tier 4 race conditions. `contextWrite` uses JSON Merge Patch semantics (RFC 7396) — object keys are merged recursively; scalars and arrays are replaced. This is the engine#511 contract: `excludedAgents: []` replaces the array (clears it), while `history` and `attempts` (not mentioned in contextWrite) are preserved.

**Output merge:** Success output merges into existing failure tracking state via `DEEP_MERGE` conflict resolver strategy (engine#508). Applies to both worker output (`WorkflowExecutionCompletedHandler`) and humanTask output (`PlanItemCompletionApplier`). Prevents successful retry or human resolution from destroying attempt history and audit trail.

**YAML canonical:** YAML remains the canonical case definition. OutcomePolicy is expressed per binding in YAML. Tier 3/4 application-level bindings are explicit YAML entries.

**Protocol:** The four-tier failure cascade is a repeatable pattern captured as `failure-cascade-pattern.md` in `casehub/garden/docs/protocols/casehub/`.

---

## 3. Blackboard Failure State Schema

The engine (via engine#503 + engine#504) writes structured failure state to the blackboard under the capability's key. This is the contract that bindings react to.

### State transitions

**Initial dispatch (existing, unchanged):**
```
securityReview: null                    ← binding fires, dispatches worker
```

**On DECLINED (engine writes via OutcomePolicy REROUTE):**
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

The engine immediately re-dispatches (same capability, excludedAgents filters agent-a). No application-level binding fires for Tiers 1-2.

**On reroute success (engine writes via DEEP_MERGE):**
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

`DEEP_MERGE` (engine#508) preserves `history`, `attempts`, `excludedAgents` when the success output `{outcome: "APPROVED"}` is applied. Applies to both worker and humanTask output paths.

**On reroutes exhausted (engine writes terminal state):**
```json
{
  "securityReview": {
    "status": "REROUTES_EXHAUSTED",
    "attempts": 2,
    "history": [ "..." ],
    "excludedAgents": ["agent-a", "agent-b"]
  }
}
```

`REROUTES_EXHAUSTED` is the terminal state the engine writes when `OutcomePolicy.maxRerouteAttempts` is reached. This triggers application-level bindings (Tier 3 or Tier 4).

**On scope reduction (Tier 3 — binding's contextWrite applies before dispatch):**
```json
{
  "securityReview": {
    "status": "PENDING",
    "attempts": 2,
    "reducedScope": true,
    "history": [ "..." ],
    "excludedAgents": []
  }
}
```

`Binding.contextWrite` (engine#511) writes `status: PENDING`, `reducedScope: true`, and clears `excludedAgents` before the capability dispatch. This prevents Tier 3 re-fire (condition `reducedScope == null` is now false) and Tier 4 race (status is no longer `REROUTES_EXHAUSTED`). Excluded agents reset because a previously-declined agent might handle the narrower scope — a DECLINE for "I can't review 500 files of Rust" doesn't imply "I can't review 3 flagged Rust files."

**On scope reduction also exhausted:**
```json
{
  "securityReview": {
    "status": "REROUTES_EXHAUSTED",
    "attempts": 4,
    "reducedScope": true,
    "history": [ "..." ],
    "excludedAgents": ["agent-c"]
  }
}
```

Tier 4 fires: `REROUTES_EXHAUSTED AND reducedScope == true`.

### Key design choices

- `status` is the engine-managed dispatch state: `PENDING`, `DECLINED`, `FAILED`, `EXPIRED`, `COMPLETED`, `REROUTES_EXHAUSTED`
- `outcome` only appears on successful completion (APPROVED/REJECTED) — domain semantics, not dispatch state
- `history` is append-only — full audit trail of every attempt across all tiers
- `excludedAgents` accumulates within a reroute cycle, resets on scope reduction
- `DEEP_MERGE` on output application preserves tracking state when success or humanTask output is written
- Existing goal conditions (`securityReview.outcome == "APPROVED"`) still work — they check `outcome`, not `status`

---

## 4. Failure Tiers — Engine vs Application Boundary

### Tiers 1-2: Engine-owned (OutcomePolicy)

The engine handles the reroute loop via `OutcomePolicy` on each binding:

```yaml
bindings:
  - name: security-review
    capability: security-review
    when: ".codeAnalysis.complete == true and .codeAnalysis.securitySensitive == true and .securityReview == null"
    outcomePolicy:
      onDecline: REROUTE
      onFailure: REROUTE
      onExpired: REROUTE
      maxRerouteAttempts: 2
    conflictResolverStrategy: DEEP_MERGE
```

On DECLINED/FAILED/EXPIRED, the engine:
1. Writes structured failure state to the blackboard (engine#503)
2. Adds the agent to `excludedAgents`
3. Increments `attempts`
4. Re-dispatches the same capability with agent exclusion (engine#502)
5. When `attempts >= maxRerouteAttempts`, writes `status: "REROUTES_EXHAUSTED"`

This is generic infrastructure — every harness gets it for free.

### Tier 3: Application-owned (scope reduction)

Fires when reroutes are exhausted and scope reduction is allowed for this capability. Uses `Binding.contextWrite` (engine#511) to update blackboard state before dispatch, and `Binding.inputSchemaOverride` (engine#509) to narrow the input.

```yaml
  - name: security-review-reduced-scope
    on: { contextChange: {} }
    when: >-
      .securityReview.status == "REROUTES_EXHAUSTED" and
      .securityReview.reducedScope == null
    contextWrite:
      securityReview:
        status: PENDING
        reducedScope: true
        excludedAgents: []
    capability: security-review
    inputSchemaOverride: "{ flaggedFiles: .codeAnalysis.flaggedFiles }"
    outcomePolicy:
      onDecline: REROUTE
      onFailure: REROUTE
      onExpired: REROUTE
      maxRerouteAttempts: 2
    conflictResolverStrategy: DEEP_MERGE
```

The engine's OutcomePolicy runs a fresh reroute loop for the reduced-scope dispatch.

### Tier 4: Application-owned (human escalation)

Fires when all automated tiers are exhausted.

```yaml
  - name: security-review-human-escalation
    on: { contextChange: {} }
    when: >-
      .securityReview.status == "REROUTES_EXHAUSTED" and
      (.securityReview.reducedScope == true or
       .policy.failurePolicy["security-review"].scopeReductionAllowed == false)
    conflictResolverStrategy: DEEP_MERGE
    humanTask:
      title: "Security review escalation — all automated reviewers exhausted"
      candidateGroups: [security-reviewers]
      expiresIn: PT4H
      outputMapping: "{ securityReview: { outcome: . } }"
      outcomes: [APPROVED, REJECTED, BLOCKED]
```

`conflictResolverStrategy: DEEP_MERGE` applies to the humanTask output path — `PlanItemCompletionApplier` must read the merge strategy (engine#508 expanded scope).

`outcomes: [APPROVED, REJECTED, BLOCKED]` requires `HumanTaskTarget.outcomes` (engine#512) — engine-enforced valid outcomes, not convention-only.

Human outcomes:
- **APPROVED** — human did the review themselves (override) → success goals can be satisfied
- **REJECTED** — PR should not merge → `review-rejected` failure goal fires
- **BLOCKED** — human can't resolve it either → `review-blocked` failure goal fires

---

## 5. Failure Policy (Application-Level Descriptor)

`FailurePolicy` configures only what the application owns — scope reduction and human escalation. Reroute parameters (`maxRerouteAttempts`) are on the OutcomePolicy per binding.

```java
public record FailurePolicy(
    boolean scopeReductionAllowed,
    String reducedInputSchema,
    Duration humanEscalationSla
) {}
```

In `PrReviewCaseDescriptor`:

| Capability | scopeReduction | reducedInputSchema | humanSla |
|-----------|----------------|-------------------|---------|
| security-review | yes | `{ flaggedFiles: .codeAnalysis.flaggedFiles }` | 4h |
| architecture-review | yes | `{ crossingPoints: .codeAnalysis.crossingPoints }` | 4h |
| style-review | no | — | 2h |
| test-coverage | no | — | 2h |
| performance-analysis | no | — | 2h |
| code-analysis | no | — | 1h |
| ci-runner | no | — | 1h |

Capabilities without scope reduction skip Tier 3 — Tier 4 fires directly on `REROUTES_EXHAUSTED`.

---

## 6. Failure Goals

Three failure goals added to the case definition:

### review-blocked

At least one required capability exhausted all tiers and the Tier 4 human WorkItem completed with BLOCKED outcome.

```java
Goal.builder()
    .name("review-blocked")
    .kind(GoalKind.FAILURE)
    .condition(ctx -> {
        // Checks each required capability key for outcome == "BLOCKED"
        for (String cap : REQUIRED_CAPABILITIES) {
            String outcome = ctx.getPathAsString(cap + ".outcome");
            if ("BLOCKED".equals(outcome)) return true;
        }
        return false;
    })
    .build();
```

### review-rejected

A human reviewer explicitly rejected the PR — the review completed with a negative verdict.

```java
Goal.builder()
    .name("review-rejected")
    .kind(GoalKind.FAILURE)
    .condition(ctx -> {
        for (String cap : REQUIRED_CAPABILITIES) {
            String outcome = ctx.getPathAsString(cap + ".outcome");
            if ("REJECTED".equals(outcome)) return true;
        }
        // Also check humanApproval for direct rejection
        return "REJECTED".equals(ctx.getPathAsString("humanApproval.outcome"));
    })
    .build();
```

### review-abandoned

PR was closed or superseded externally.

```java
Goal.builder()
    .name("review-abandoned")
    .kind(GoalKind.FAILURE)
    .condition(ctx ->
        "closed".equals(ctx.getPath("pr.status")) ||
        "superseded".equals(ctx.getPath("pr.status")))
    .build();
```

Written via external signal (REST call for now; GitHub webhook in Epic #15).

### Case-level SLA (deferred)

`review-timed-out` is removed from this spec. Case-level SLA requires a time-triggered binding mechanism the engine doesn't have today. Tracked separately as casehubio/engine#510.

### Completion expression

```java
.completion(
    GoalExpression.allOf(prApproved, securityVerified, ciPassing),
    GoalExpression.anyOf(reviewBlocked, reviewRejected, reviewAbandoned)
)
```

Uses the existing two-argument `CaseDefinition.builder().completion(success, failure)` form. Engine evaluates failure before success — if both are satisfied simultaneously, failure wins. Engine#506 fixes the FAULTED→COMPLETED transition for failure goals.

---

## 7. ReviewerOutcome Changes

```java
public sealed interface ReviewerOutcome
        permits ReviewerOutcome.Completed, ReviewerOutcome.Declined, ReviewerOutcome.Failed {

    record Completed(List<String> findings) implements ReviewerOutcome {}
    record Declined(String reason)          implements ReviewerOutcome {}
    record Failed(String reason)            implements ReviewerOutcome {}
}
```

`Failed` carries a reason only. RetryPolicy (existing) handles infrastructure transients (worker threw an exception — retry same agent with backoff). OutcomePolicy handles semantic failures (worker explicitly reported failure — reroute to different agent). These are different levels and should not be conflated.

`QhorusPrReviewService` handles the new variant with `MessageType.FAILURE`:

```java
case ReviewerOutcome.Failed failed ->
    messageService.dispatch(MessageDispatch.builder()
        .channelId(work.id)
        .sender(ORCHESTRATOR)
        .type(MessageType.FAILURE)
        .content(failed.reason())
        .correlationId(correlationId)
        .inReplyTo(commandResult.messageId())
        .actorType(ActorType.SYSTEM)
        .build());
```

---

## 8. Engine Dependencies (Blocking)

**Epic:** casehubio/engine#501 — Semantic failure routing

| Issue | What | Scale | Depends on |
|-------|------|-------|------------|
| engine#502 | Agent exclusion — read `excludedAgents` from case context | S / Low | — |
| engine#503 | Semantic DECLINE/FAIL/EXPIRED — structured blackboard write | L / High | — |
| engine#504 | OutcomePolicy — REROUTE/FAULT for speech-act outcomes incl. EXPIRED | L / High | #503, qhorus#281 |
| engine#506 | Failure goals → COMPLETED not FAULTED | S / Med | — |
| engine#508 | DEEP_MERGE conflict resolver (worker + humanTask paths) | M / Med | — |
| engine#509 | Binding.inputSchemaOverride for scope-reduced dispatch | S / Low | — |
| engine#511 | Binding.contextWrite — pre-dispatch blackboard update | S / Med | — |
| engine#512 | HumanTaskTarget.outcomes — propagate to WorkItem | XS / Low | — |

**Cross-repo:**

| Issue | What | Repo |
|-------|------|------|
| qhorus#281 | CommitmentExpiredEvent CDI event from expireOverdue() | casehubio/qhorus |

**Deferred (not blocking):**

| Issue | What | Repo |
|-------|------|------|
| engine#510 | Case-level SLA — time-triggered binding | casehubio/engine |

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

## 10. Trust Scoring (Deferred)

The trust dimension mapping for failure outcomes is:

| Outcome | Trust dimension | Signal |
|---------|----------------|--------|
| DECLINED | scope-calibration | Agent correctly identified its capability boundary — positive signal |
| FAILED | review-thoroughness | Agent attempted but couldn't complete — negative signal |
| EXPIRED | responsiveness (new) | Agent went silent — negative signal |

`responsiveness` is a new trust dimension — requires adding a constant to `DevtownTrustDimension` (domain model change, noted in §14).

The mechanism: commitment terminal state (`CommitmentState.DECLINED/FAILED/EXPIRED`) already flows to the ledger via P0.2 wiring (qhorus#123). The trust dimension mapping is application-layer configuration in devtown.

Full attestation wiring is deferred to casehubio/devtown#13 (trust-weighted reviewer routing epic). This spec defines the failure states that feed trust scoring; #13 defines how the scoring consumes them.

---

## 11. Protocol: Failure Cascade Pattern

To be written as `casehub/garden/docs/protocols/casehub/failure-cascade-pattern.md`. Captures:

- DECLINED vs FAILED vs EXPIRED as distinct outcomes with different trust scoring implications but same routing response (REROUTE)
- Four-tier failure cascade: engine reroute loop (Tiers 1-2) → application scope reduction (Tier 3) → application human escalation (Tier 4)
- Engine/application boundary: OutcomePolicy owns reroute loop, application owns domain-specific responses
- FailurePolicy on the descriptor as the application-level configuration
- Blackboard failure state schema including `REROUTES_EXHAUSTED` terminal state
- `Binding.contextWrite` for pre-dispatch state marking (prevents infinite re-fire and race conditions)
- `DEEP_MERGE` conflict resolver on all failure-tracking bindings (worker and humanTask paths)
- `Binding.inputSchemaOverride` for scope-reduced dispatch
- `HumanTaskTarget.outcomes` for engine-enforced human decision options
- Failure goals as legitimate case outcomes via `COMPLETED` (not `FAULTED`)
- Tier 3 resets `status` to `PENDING` and clears `excludedAgents` via contextWrite

---

## 12. Goal Semantics (Platform Coherence Note)

A CaseHub Goal is a **named predicate over the blackboard that recognises an outcome**. Goals are passive observers — they evaluate state, they don't cause state changes.

| Concept | What it does |
|---------|-------------|
| Binding | Watches the blackboard, triggers action (dispatch worker, create WorkItem) |
| Goal | Watches the blackboard, recognises an outcome (success or failure) |
| Completion | Combines goals — determines when the case is done |

Goals don't care how the blackboard reached a state. Bindings fire adaptively based on content analysis, agent availability, and failure history. Goals evaluate the result. This is the ACM distinction from workflow: you reach the end when goals are satisfied, not when steps are completed.

The `GoalKind` enum (SUCCESS, FAILURE) distinguishes positive from negative outcomes. Both result in `CaseStatus.COMPLETED` — the goal kind is outcome metadata, not a different terminal state. `FAULTED` is reserved for system errors (infrastructure failures, exhausted retries at the engine level).

All Goal instances are type-safe — searchable by type across the codebase. Platform-wide coherence review of Goal usage is deferred but tractable.

---

## 13. YAML Representation

YAML remains the canonical case definition. The DSL is the testing companion.

### OutcomePolicy on existing bindings (engine#504)

```yaml
bindings:
  - name: security-review
    on: { contextChange: {} }
    when: >-
      .codeAnalysis.complete == true and
      .codeAnalysis.securitySensitive == true and
      .securityReview == null
    capability: security-review
    conflictResolverStrategy: DEEP_MERGE
    outcomePolicy:
      onDecline: REROUTE
      onFailure: REROUTE
      onExpired: REROUTE
      maxRerouteAttempts: 2
```

### Tier 3 scope-reduction bindings (~2 per case definition)

```yaml
  - name: security-review-reduced-scope
    on: { contextChange: {} }
    when: >-
      .securityReview.status == "REROUTES_EXHAUSTED" and
      .securityReview.reducedScope == null
    contextWrite:
      securityReview:
        status: PENDING
        reducedScope: true
        excludedAgents: []
    capability: security-review
    inputSchemaOverride: "{ flaggedFiles: .codeAnalysis.flaggedFiles }"
    conflictResolverStrategy: DEEP_MERGE
    outcomePolicy:
      onDecline: REROUTE
      onFailure: REROUTE
      onExpired: REROUTE
      maxRerouteAttempts: 2
```

### Tier 4 human-escalation bindings (7 — one per capability)

For capabilities with scope reduction:
```yaml
  - name: security-review-human-escalation
    on: { contextChange: {} }
    when: >-
      .securityReview.status == "REROUTES_EXHAUSTED" and
      .securityReview.reducedScope == true
    conflictResolverStrategy: DEEP_MERGE
    humanTask:
      title: "Security review escalation — all automated reviewers exhausted"
      candidateGroups: [security-reviewers]
      expiresIn: PT4H
      outputMapping: "{ securityReview: { outcome: . } }"
      outcomes: [APPROVED, REJECTED, BLOCKED]
```

For capabilities without scope reduction:
```yaml
  - name: style-check-human-escalation
    on: { contextChange: {} }
    when: '.styleCheck.status == "REROUTES_EXHAUSTED"'
    conflictResolverStrategy: DEEP_MERGE
    humanTask:
      title: "Style review escalation"
      candidateGroups: [pr-reviewers]
      expiresIn: PT2H
      outputMapping: "{ styleCheck: { outcome: . } }"
      outcomes: [APPROVED, REJECTED, BLOCKED]
```

### Failure goals in completion

```yaml
completion:
  success:
    allOf:
      - pr-approved
      - security-verified
      - ci-passing
  failure:
    anyOf:
      - review-blocked
      - review-rejected
      - review-abandoned
```

### Total binding count

Current: 9 bindings. After failure handling: 18 bindings (9 existing with added `outcomePolicy` + 2 scope reduction + 7 human escalation). Each existing binding gains an `outcomePolicy` and `conflictResolverStrategy` section. Readable and maintainable.

---

## 14. What devtown Implements (after engine#501 ships)

**Domain model:**
- `ReviewerOutcome.Failed` sealed variant
- `FailurePolicy` record (scope reduction + human escalation only)
- `PrReviewCaseDescriptor` with `FAILURE_POLICIES` map
- `DevtownTrustDimension.RESPONSIVENESS` — new trust dimension constant for EXPIRED outcomes

**Case definition (YAML + DSL companion):**
- Three failure goals (`review-blocked`, `review-rejected`, `review-abandoned`)
- Two-arg completion expression (success + failure)
- `outcomePolicy` + `conflictResolverStrategy: DEEP_MERGE` on all existing capability bindings
- Tier 3 scope-reduction bindings with `contextWrite` for security-review and architecture-review
- Tier 4 human-escalation bindings with `outcomes` for all 7 capabilities

**Qhorus integration (`QhorusPrReviewService`):**
- Handle `ReviewerOutcome.Failed` → `MessageType.FAILURE`

**SLA breach handler generalization (`SlaBreachHandler`):**
- Current implementation hardcodes `"humanApproval"` as the signal path. Must be generalized to route breach signals to the correct blackboard path based on the binding's output mapping. The handler reads `planItemId` from `CallerRef`, looks up the binding in the case definition, extracts the output mapping, and applies the breach outcome through it — same path as `PlanItemCompletionApplier` uses for happy-path output. Without this, a Tier 4 human reviewer who misses the SLA deadline produces a case that never completes.

**Protocol:**
- `failure-cascade-pattern.md` in `casehub/garden/docs/protocols/casehub/`

---

## Appendix: Review Issues Addressed

### Revision 2 (10 issues)

| # | Issue | Resolution |
|---|-------|-----------|
| 1 | Tiers 1-2 duplicate engine#504 | **Agreed.** Removed application-level Tier 1/2 bindings. Engine owns reroute loop via OutcomePolicy. |
| 2 | Successful retry overwrites failure history | **Agreed.** Filed engine#508 (DEEP_MERGE). Bindings declare `conflictResolverStrategy: DEEP_MERGE`. |
| 3 | Scope reduction needs input schema override | **Agreed.** Filed engine#509 (Binding.inputSchemaOverride). |
| 4 | Tier 3→4 race on status | **Agreed.** Tier 3 resets `status` to `PENDING` via contextWrite. Tier 4 fires on `REROUTES_EXHAUSTED AND (reducedScope == true OR NOT scopeReductionAllowed)`. |
| 5 | EXPIRED outcome missing | **Agreed.** Filed qhorus#281 (CommitmentExpiredEvent). Updated engine#504 to include `onExpired`. |
| 6 | review-timed-out has no mechanism | **Agreed.** Removed from spec. Filed engine#510 (case-level SLA) separately. |
| 7 | transient_ conflicts with static OutcomePolicy | **Agreed.** Dropped `transient_`. `Failed(String reason)` only. |
| 8 | YAML representation unaddressed | **Agreed.** Added §13 with full YAML examples. |
| 9 | failureGoal always same value | **Agreed.** Removed from FailurePolicy. |
| 10 | Trust scoring unspecified | **Agreed.** Added §10 with dimension mapping and explicit deferral to devtown#13. |

### Revision 3 (3 issues + 4 minor)

| # | Issue | Resolution |
|---|-------|-----------|
| 1 | Tier 3 pre-dispatch context write has no engine mechanism | **Agreed.** Filed engine#511 (Binding.contextWrite). All Tier 3 bindings use `contextWrite` for pre-dispatch state marking. |
| 2 | Tier 4 humanTask missing DEEP_MERGE + outcomes gap | **Agreed.** Added `conflictResolverStrategy: DEEP_MERGE` to all Tier 4 bindings. Expanded engine#508 scope to cover PlanItemCompletionApplier. Filed engine#512 (HumanTaskTarget.outcomes). |
| 3 | REJECTED human outcome creates dead case | **Agreed.** Added `review-rejected` failure goal. Completion expression: `anyOf(reviewBlocked, reviewRejected, reviewAbandoned)`. |
| M1 | §4/§13 Tier 4 condition inconsistency | **Fixed.** Standardized to use `policy.failurePolicy` path in §4, DSL-substituted boolean in §13 YAML. |
| M2 | hasUnresolvableCapability undefined | **Fixed.** Goal condition body now explicit with loop over REQUIRED_CAPABILITIES. |
| M3 | Binding count "~6" should be "7" | **Fixed.** 7 human escalation bindings (one per capability). |
| M4 | EXPIRED trust dimension is domain model change | **Fixed.** `DevtownTrustDimension.RESPONSIVENESS` noted as domain model change in §14. |

### Revision 4 (2 issues)

| # | Issue | Resolution |
|---|-------|-----------|
| 1 | SlaBreachHandler hardcodes humanApproval — Tier 4 breach misses capability key | **Agreed.** SlaBreachHandler generalization added to §14 as devtown implementation requirement. Handler reads planItemId from CallerRef and routes breach signal through binding's output mapping. |
| 2 | contextWrite merge semantics unspecified | **Agreed.** Added JSON Merge Patch (RFC 7396) contract to §2: objects merge recursively, scalars and arrays replace. Locks down engine#511 contract. |
