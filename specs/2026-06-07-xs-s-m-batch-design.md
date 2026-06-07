# XS/S/M Batch — Issues #19, #61, #72, #64, #18, #60

**Branch:** `issue-19-xs-s-m-batch`
**Date:** 2026-06-07

---

## Scope

Six issues triaged and resolved in a single branch. Three close with no code changes (dependency satisfied, no observers to update, premature abstraction). Three require implementation.

---

## Issues Closing Without Code Changes

### #19 — CAPABILITY_DIMENSION composite trust score (XS, dependency on ledger#76)

**Decision:** Close. The wiring is already complete.

ledger#76 shipped `ScoreType.CAPABILITY_DIMENSION`. The issue anticipated needing to "extend RoutingPolicy with per-capability dimension floors" — but this was completed as part of the trust routing implementation:

- `TrustRoutingPolicyKeys` defines per-dimension floor keys (`FLOOR_REVIEW_THOROUGHNESS`, `FLOOR_PRECISION`, `FLOOR_SCOPE_CALIBRATION`) mapped to `DevtownTrustDimension` constants.
- `DevtownTrustRoutingPolicyProvider.forCapability()` builds a `qualityFloors` map from these keys and passes it to `TrustRoutingPolicy`.
- The YAML config (`trust-routing.yaml`) already declares per-dimension floors per capability.
- The engine's `TrustWeightedAgentStrategy` queries CAPABILITY_DIMENSION scores automatically when floor keys are present.

No devtown code changes needed. The domain model accommodated the extension exactly as the issue predicted.

### #61 — CaseLifecycleEvent observer for tenancyId (XS, dependency on engine#299)

**Decision:** Close. No observers exist.

engine#299 added `String tenancyId` as the second field of `CaseLifecycleEvent`. The issue said to "find all `@ObservesAsync CaseLifecycleEvent` usages." There are zero in devtown.

Devtown's only relevant observer is `ReviewOutcomeObserver`, which observes `PlanItemCompletedEvent` (not `CaseLifecycleEvent`). It already gets `tenancyId` correctly from `CaseInstance.tenancyId` via `CrossTenantCaseInstanceRepository`.

No compile break. No code to update.

### #18 — Composite capability registry (S, idea)

**Decision:** Close as premature.

The composite pattern (CDI-aggregated multi-contributor `CapabilityRegistry`) is architecturally sound but has no current consumers beyond devtown's single static registry. The issue itself notes: "worth it only when multiple simultaneous contributors are a real requirement" and "platform consolidation value only materialises when ≥3 domain repos independently implement the pattern."

The current `DevtownCapabilityRegistry` is the correct "populated default" per the SPI defaults rule in PLATFORM.md. The eidos `VocabularyRegistrar` pattern exists as a reference implementation when the time comes. Reopen when a second harness needs capability aggregation.

---

## Issues Requiring Implementation

### #72 — Fix CaseMemoryIntegrationTest CaseDefinition not found (S, Med)

#### Problem

`CaseMemoryIntegrationTest.fullRoundTrip_emitThenRecall()` is `@Disabled` with error: "CaseDefinition not found during startCase — async case definition registration race."

#### Root Cause Analysis

`DefaultCaseDefinitionRegistry` registers case definitions at startup:

```java
void onStart(@Observes @Priority(10) StartupEvent ev) {
    ReactiveUtils.runOnSafeVertxContext(this.vertx, this::registerKnownDefinitions)
        .await().atMost(Duration.ofSeconds(30L));
}
```

`registerKnownDefinitions()` iterates all `CaseHub` beans via `Instance<CaseHub>` and calls `registerCaseDefinition()` for each. This reactive chain calls `caseMetaModelRepository.findByKey(namespace, name, version, currentPrincipal.tenancyId())` followed by `.save()`, then does `registry.put()` in an `.invoke()` callback.

The `getCaseMetaModel()` method (called from `startCase`) does a synchronous lookup in the `registry` ConcurrentHashMap. If the entry isn't present, it throws `RuntimeException("CaseMetaModel not found...")`.

Probable cause: the reactive registration chain runs on a Vert.x context thread. The `await()` on the calling thread may return before the `registry.put()` in the reactive chain's `.invoke()` callback has executed — a happens-before gap between the Vert.x context completing the Uni and the ConcurrentHashMap write being visible.

#### Fix Approach

1. Remove `@Disabled` from `fullRoundTrip_emitThenRecall()`.
2. Run the test to capture the exact exception and stack trace.
3. If the failure is the registration race: add an `awaitility` guard at the start of the test that waits for `caseHub.getDefinition()` to be registered in the `CaseDefinitionRegistry` before calling `startCase()`. This is a test-side fix — the engine's startup registration is correct for production (where no test thread races against it).
4. If the failure has a different root cause (e.g., tenancyId mismatch between startup principal and test principal), fix accordingly.

#### Verification

Test passes reliably without `@Disabled`. Run 3 times to confirm no flakiness.

---

### #64 — Set allowedWriters on pr-review oversight channel (S, Med)

#### Problem

`QhorusPrReviewService` creates three channels per PR review (work, observe, oversight). All pass `null` for `allowedWriters`. The oversight channel restricts message types (`COMMAND, DONE, DECLINE`) but not who can write — any actor can send a DONE and close a human oversight gate.

#### Design

Set `allowedWriters` on all three channels at creation time. The participants are known:

| Channel | Writers | Rationale |
|---------|---------|-----------|
| `work` | `pr-orchestrator` | Orchestrator sends COMMANDs; agent responses use the agent's identity, but the orchestrator is the channel owner. Agent writes are gated by the Commitment target, not channel ACL — the orchestrator dispatches on behalf of agents. |
| `observe` | `pr-orchestrator` | Observe is for passive monitoring events; the orchestrator publishes state. |
| `oversight` | `pr-orchestrator` | Only the orchestrator writes COMMANDs to the oversight channel. Human DONE/DECLINE responses will be added via `channelService.setAllowedWriters()` when the HITL adapter assigns a reviewer (future Layer 6 wiring — not in scope here). |

The work channel writer list is intentionally restrictive. In the current Layer 3 implementation, `QhorusPrReviewService` dispatches all messages as the orchestrator (both COMMANDs and DONE/DECLINE responses). When Layer 6 wires real agents, the agent identity will be added to `allowedWriters` at dispatch time via `channelService.setAllowedWriters()`.

#### Changes

1. **`QhorusPrReviewService`:**
   - `findOrCreateWorkChannel()` — pass `ORCHESTRATOR` as `allowedWriters` (5th param).
   - `findOrCreateObserveChannel()` — pass `ORCHESTRATOR` as `allowedWriters`.
   - `findOrCreateOversightChannel()` — pass `ORCHESTRATOR` as `allowedWriters`.
   - Add `requireAllowedWriters()` validation method, parallel to existing `requireAllowedTypes()`.
   - Introduce `ORCHESTRATOR_WRITERS` constant alongside existing type constants.

2. **`PrReviewQhorusLifecycleTest`:**
   - Add assertions for `allowedWriters` on all three channels.
   - Verify existing `allowedTypes` assertions still pass.

#### Migration

Existing channels (created before this change) will have `allowedWriters = null`. The `requireAllowedWriters()` method fails fast if an existing channel's writer contract doesn't match, same as `requireAllowedTypes()` does today. This surfaces stale channels rather than silently bypassing enforcement.

---

### #60 — Fluent Java DSL companions for devtown case definitions (M, Low)

#### Requirement

Protocol `case-definition-layers` mandates every YAML case definition has a companion fluent Java DSL builder. Currently `pr-review.yaml` (147 lines, 8 capabilities, 3 goals, 11 bindings) is the only case definition and has no DSL companion.

#### Design

Create `PrReviewCaseDefinitions` in the `review/` module — a utility class with a single static factory method that builds the same `CaseDefinition` as `pr-review.yaml`.

**Module placement:** `review/src/main/java/io/casehub/devtown/review/PrReviewCaseDefinitions.java`

The `review/` module is correct because:
- The DSL builder is domain logic (defines what a PR review case is), not application wiring.
- It sits alongside `PrReviewApplicationService` and other domain contracts.
- The `app/` module's `PrReviewCaseHub` continues to load from YAML — the DSL companion exists for programmatic construction and testing, not as a replacement.

**Structure:**

```java
public final class PrReviewCaseDefinitions {

    public static CaseDefinition prReview() {
        return CaseDefinition.builder()
            .name("pr-review")
            .namespace("devtown")
            .version("1.0.0")
            .dsl("0.1")
            // ... capabilities, goals, completion, bindings
            .build();
    }

    private PrReviewCaseDefinitions() {}
}
```

The method mirrors the YAML structure exactly:
- 8 capabilities with inputSchema/outputSchema
- 3 goals with JQ conditions
- Completion criteria (allOf: pr-approved, security-verified, ci-passing)
- 11 bindings organized in 4 groups (entry, content-driven, human gate, merge)

**Dependencies:** The `review/` module already depends on `casehub-engine-api` (which provides `CaseDefinition`, `CaseDefinition.builder()`, and all model types). No new dependencies needed.

#### Test

Create `PrReviewCaseDefinitionTest` in `review/src/test/`:

```java
class PrReviewCaseDefinitionTest {

    @Test
    void dslMatchesYaml() {
        CaseDefinition fromYaml = CaseDefinitionYamlMapper.load(
            getClass().getClassLoader().getResourceAsStream("devtown/pr-review.yaml"));
        CaseDefinition fromDsl = PrReviewCaseDefinitions.prReview();

        // Structural equivalence checks
        assertThat(fromDsl.getName()).isEqualTo(fromYaml.getName());
        assertThat(fromDsl.getNamespace()).isEqualTo(fromYaml.getNamespace());
        assertThat(fromDsl.getCapabilities()).hasSameSizeAs(fromYaml.getCapabilities());
        assertThat(fromDsl.getGoals()).hasSameSizeAs(fromYaml.getGoals());
        assertThat(fromDsl.getBindings()).hasSameSizeAs(fromYaml.getBindings());

        // Per-capability checks
        for (int i = 0; i < fromYaml.getCapabilities().size(); i++) {
            assertThat(fromDsl.getCapabilities().get(i).getName())
                .isEqualTo(fromYaml.getCapabilities().get(i).getName());
        }

        // Per-binding checks (name, capability, when condition)
        for (int i = 0; i < fromYaml.getBindings().size(); i++) {
            assertThat(fromDsl.getBindings().get(i).getName())
                .isEqualTo(fromYaml.getBindings().get(i).getName());
        }
    }
}
```

This is a plain JUnit test (no `@QuarkusTest` needed) — it compares two `CaseDefinition` objects structurally.

#### Garden Gotcha

GE-20260531-d896bf: "SubCase M-of-N fields (groupId, totalInGroup, requiredCount, onThresholdReached) are DSL-only — not supported in YAML." Not relevant here since `pr-review` doesn't use subcases, but confirms the DSL can express things YAML cannot.

---

## Implementation Order

1. **#19, #61, #18** — close issues on GitHub (no code)
2. **#72** — fix CaseMemoryIntegrationTest
3. **#64** — set allowedWriters on channels
4. **#60** — create fluent DSL companion

This order handles the smallest/diagnostic work first, then the structural channel change, then the larger additive DSL work.

---

## Out of Scope

- Layer 6 trust routing wiring (future — updates `allowedWriters` dynamically when agents are assigned)
- Composite `CapabilityRegistry` (deferred until ≥2 harnesses need it)
- `docs/PROGRESS.md` DT-003 update (file does not exist; tracked in issue description only)
