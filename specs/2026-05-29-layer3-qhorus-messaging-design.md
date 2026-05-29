# Layer 3 — casehub-qhorus Typed Messaging Design

**Date:** 2026-05-29
**Issue:** casehubio/devtown#52
**Branch:** issue-52-layer3-qhorus-messaging
**Builds on:** Layer 2 (devtown#41, casehub-work SLA gate), Layer 5 (devtown#10, engine CasePlanModel)

---

## Context

Layer 1 (`NaivePrReviewService`) calls specialist logic directly — no formal obligation, no audit, no scope refusal. Layer 3 replaces that with typed speech-act messaging through casehub-qhorus channels: every specialist review request becomes a COMMAND that creates a Commitment; DONE fulfils it; DECLINE is a formal scope boundary with a recorded reason.

Layer 5 (`PrReviewCaseService`) was built before Layers 2–4 for architectural reasons. This spec describes Layer 3; the CDI priority ordering ensures Layer 5 remains the active bean in the full build while Layer 3 exists for tutorial reading.

---

## What Layer 3 Closes

| Gap (from Layer 1) | What changes |
|---|---|
| No formal obligation per reviewer | COMMAND creates a tracked Commitment; DONE/DECLINE discharge it |
| No formal scope refusal | DECLINE is a first-class typed outcome with a recorded reason |
| No audit of inter-agent communication | Every message recorded as `MessageLedgerEntry` |

**Not closed in Layer 3:**
- Fixed routing (no adaptive binding — Layer 5)
- SLA enforcement per reviewer (Layer 2 / casehub-work)
- Tamper-evident review record chained to incidents (Layer 4)

---

## Architecture

### CDI Displacement Chain

Three beans implement `PrReviewApplicationService`. Priority ordering is stable across all layers:

```
PrReviewService          @DefaultBean                    Layer 1   fallback; never wins
QhorusPrReviewService    @Alternative @Priority(1)       Layer 3   present; CDI-inactive in full build
PrReviewCaseService      @Alternative @Priority(2)       Layer 5   ACTIVE in full build
```

`PrReviewCaseService` is unchanged in logic; it receives `@Alternative @Priority(2)` to resolve the ambiguity that would otherwise arise from two non-`@DefaultBean @ApplicationScoped` implementations.

`QhorusPrReviewService` is injected directly by type in tests — CDI resolves it regardless of priority ordering, so no test profile is needed.

### New Types (both in `review/`)

**`ReviewerOutcome`** — sealed interface, two variants. No `Failed` — Layer 3 stubs don't exercise the FAILURE state.

```java
public sealed interface ReviewerOutcome
    permits ReviewerOutcome.Completed, ReviewerOutcome.Declined {
    record Completed(List<String> findings) implements ReviewerOutcome {}
    record Declined(String reason)          implements ReviewerOutcome {}
}
```

**`ReviewerAgent`** — driven-port interface. Implementations in `app/`.

```java
public interface ReviewerAgent {
    String capability(); // ReviewDomain constant
    ReviewerOutcome handle(PrPayload pr);
}
```

`review/pom.xml` already carries `casehub-qhorus-api` — no new module dependencies.

### Channel Structure

One normative 3-channel set per PR review. All specialist interactions share the `/work` channel. Channel creation is idempotent (`findByName` first):

| Channel | Semantic | Used in Layer 3 |
|---|---|---|
| `pr-review-{prNumber}/work` | APPEND | Yes — COMMAND/DONE/DECLINE per specialist |
| `pr-review-{prNumber}/observe` | APPEND | Created but empty; Layer 4 posts ledger events here |
| `pr-review-{prNumber}/oversight` | APPEND | Created but empty; Layer 2 oversight integration uses this |

**Rationale for shared `/work`:** Oversight and observe are PR-level concerns, not specialist-level. One channel set per PR aligns with Layer 5's one-case-per-PR model — when both layers are active, `subjectId=caseId` on qhorus messages links the ledger entries to the engine case. A per-specialist channel layout would require 9 channels per PR (3 × 3) and incoherently implies three separate oversight gates.

### `QhorusPrReviewService` (in `app/`)

```
@ApplicationScoped @Alternative @Priority(1)
implements PrReviewApplicationService
injects: ChannelService, MessageService, Instance<ReviewerAgent>
```

`review(PrPayload pr)` flow:
1. Create (or retrieve) `pr-review-{n}/work`, `/observe`, `/oversight`
2. For each `ReviewerAgent` in CDI instance set:
   a. Send `COMMAND` on `/work` with `target=agent.capability()`, fresh `correlationId`, `actorType=SYSTEM`
   b. Call `agent.handle(pr)` synchronously
   c. Pattern-match `ReviewerOutcome`:
      - `Completed` → send `DONE` on `/work` with same correlationId, `inReplyTo=commandMessageId`, `actorType=AGENT`; collect findings
      - `Declined` → send `DECLINE` with reason as content; same link fields
3. Return `new PrReviewOutcome("qhorus-reviewed", allFindings)`

**Commitment semantics:** `target=capability` on the COMMAND sets `obligor=capability` in the Commitment. After DONE/DECLINE, `commitmentStore.findOpenByObligor(capability, workChannelId)` returns empty — all obligations are discharged before `review()` returns.

### Agent Stubs (in `app/`)

Three `@ApplicationScoped` beans:

| Class | `capability()` | `handle()` return |
|---|---|---|
| `SecurityReviewAgent` | `ReviewDomain.SECURITY_REVIEW` | `Completed(["rate-limiting absent on /payment"])` |
| `ArchitectureReviewAgent` | `ReviewDomain.ARCHITECTURE_REVIEW` | `Declined("distributed transaction outside scope")` |
| `TestCoverageReviewAgent` | `ReviewDomain.TEST_COVERAGE` | `Completed(["coverage 67%; payment path untested"])` |

`ArchitectureReviewAgent` unconditionally DECLINEs — deterministic for tests; the tutorial narrative provides the business reason. All three are `@ApplicationScoped` (main-scope) production beans, present as tutorial artefacts when Layer 3 is CDI-inactive in the full build.

---

## Testing

### `PrReviewQhorusLifecycleTest @QuarkusTest`

In `app/src/test/`. Injects `QhorusPrReviewService` directly by type. Also injects `ChannelService` and in-memory message/commitment stores from `casehub-qhorus-testing` (already on the test classpath).

| Test | What it verifies |
|---|---|
| `channelsCreated` | After `review()`, all three channels exist by name |
| `commandsDispatched` | `/work` has 3 COMMAND messages; each has `target` set to the respective capability |
| `doneDischargesCommitment` | `findOpenByObligor(SECURITY_REVIEW, workId)` is empty; same for TEST_COVERAGE |
| `declineRecorded` | `/work` has exactly one DECLINE message; `findOpenByObligor(ARCHITECTURE_REVIEW, workId)` is empty |

**Commitment query:** `findOpenByObligor(capability, channelId)` is correct here — `target=capability` makes the agent the obligor (not the requester). This is consistent with qhorus deontic semantics: the obligor is the party that owes a response, and we set the target explicitly to make that binding. Garden GE-20260521-e39ad1 covers the confusion that arises when `target` is absent (obligor=null) — we avoid it by always setting `target`.

All existing `@QuarkusTest` runs are unaffected — `PrReviewCaseService @Alternative @Priority(2)` remains the selected bean for the `PrReviewApplicationService` injection point.

---

## Module Changes Summary

| Module | Change |
|---|---|
| `review/` | Add `ReviewerOutcome` (sealed), `ReviewerAgent` (interface) |
| `app/main` | Add `QhorusPrReviewService`, `SecurityReviewAgent`, `ArchitectureReviewAgent`, `TestCoverageReviewAgent` |
| `app/main` | `PrReviewCaseService`: add `@Alternative @Priority(2)` |
| `app/test` | Add `PrReviewQhorusLifecycleTest` |
| `app/pom.xml` | No changes — `casehub-qhorus` and `casehub-qhorus-testing` already present |

---

## No-Go Decisions

| Rejected | Reason |
|---|---|
| Per-specialist channel sets | Incoherent oversight model; 9 channels per PR vs 3; misaligns with Layer 5 case model |
| Flat capability channels (AML pattern) | Mixes all PR reviews on one channel; loses "channel = review workspace" teaching point |
| `AgentDispatchMechanism` / `PushAgentDispatch` | No architectural benefit in Layer 3; agents are synchronous CDI beans |
| `ReviewerOutcome.Failed` | Layer 3 stubs don't exercise FAILURE; adding it now is YAGNI |
