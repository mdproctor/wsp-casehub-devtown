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

#### What We Know

`DefaultCaseDefinitionRegistry` registers case definitions at startup:

```java
void onStart(@Observes @Priority(10) StartupEvent ev) {
    ReactiveUtils.runOnSafeVertxContext(this.vertx, this::registerKnownDefinitions)
        .await().atMost(Duration.ofSeconds(30L));
}
```

Registration iterates all `CaseHub` beans via `Instance<CaseHub>` and for each calls `registerCaseDefinition()`. This reactive chain calls `caseMetaModelRepository.findByKey(namespace, name, version, currentPrincipal.tenancyId())`, then `.save()`, then `registry.put()` via `.invoke()`.

`getCaseMetaModel()` (called by `startCase`) does a synchronous lookup in the `registry` ConcurrentHashMap. If the entry is absent, it throws `RuntimeException("CaseMetaModel not found...")`.

**Key facts:**

1. **`await()` guarantees visibility.** Mutiny's `await()` uses a `CountDownLatch` internally, which establishes a happens-before relationship per JMM. By the time `await().atMost(30s)` returns, all `.invoke()` callbacks in the reactive pipeline have completed and their side effects (the `ConcurrentHashMap.put()`) are visible to the calling thread. A timing/visibility race between the Vert.x context and the startup thread is not the cause.

2. **`CaseKey` is tenancy-agnostic.** `CaseKey(namespace, name, version)` — no tenancyId component. Both `CaseKey.of(CaseDefinition)` and `CaseKey.of(CaseMetaModel)` extract the same three fields. A key mismatch between registration and lookup is not the cause.

3. **Startup-only registration (GE-20260414-f4f539).** Since engine PR #28, `startCase()` no longer registers definitions at runtime — it only looks up pre-registered definitions. If registration fails during startup for any reason, `startCase()` will always fail.

4. **tenancyId at startup vs test.** During startup, `currentPrincipal.tenancyId()` returns the `FixedCurrentPrincipal` default (before `@BeforeEach` sets it to `DEFAULT_TENANT_ID`). The tenancyId is used in `caseMetaModelRepository.findByKey()` and `.save()` — these store the `CaseMetaModel` in the in-memory map keyed by `tenancyId:namespace:name:version`. While this doesn't affect the registry ConcurrentHashMap (tenancy-agnostic), a startup failure in the repository path (e.g., null tenancyId causing an NPE) could abort registration silently.

5. **`registerKnownDefinitions()` concatenates, doesn't fail-fast.** It uses `transformToUniAndConcatenate` — if the first `CaseHub` registration fails, the exception propagates to `await()` and the startup handler throws. But if the Uni completes successfully with a null result (e.g., `findByKey` returns null, `save` swallows an error), the `registry.put()` would be skipped.

#### Fix Approach

The root cause is not established — the analysis above narrows the search space but does not identify the failure. The fix is diagnostic-first:

1. **Remove `@Disabled`** from `fullRoundTrip_emitThenRecall()`.
2. **Run the test** and capture the exact exception, stack trace, and any startup warnings/errors in the Quarkus log.
3. **Let the actual failure drive the fix.** Candidate root causes to investigate in order:
   - Startup registration failure (exception swallowed or Uni completing empty) — check Quarkus startup logs for `CaseDefinitionRegistry` registration messages.
   - tenancyId null/mismatch during startup causing `InMemoryCaseMetaModelRepository` to store under a different key than expected.
   - `FixedCurrentPrincipal` state at startup vs `@BeforeEach` — does `reset()` undo a successful registration?
4. **If the root cause is in the engine's startup path**, the fix belongs in the engine (or in the devtown SPI configuration), not in a test-side workaround. A startup registration bug affects production, not just tests.
5. **If the root cause is test configuration** (e.g., principal state, CDI ordering), fix the test configuration.

#### Verification

Test passes reliably without `@Disabled`. Run 3 times to confirm no flakiness.

#### Outcome

Neither candidate root cause from the spec's list was the actual failure. The diagnostic-first approach proved its value — the real bugs were discovered empirically:

**Bug 1:** `SchedulerService.registerScheduledTriggers()` calls `getCaseDefinition(caseInstance.getCaseMetaModel())` which returns null in the `CaseStartedEventHandler` event bus handler, then throws `IllegalStateException`. This blocks `CONTEXT_CHANGED` publication and prevents binding evaluation. Filed as **engine#444**. The pr-review case has zero `ScheduleTrigger` bindings — the failure occurs before SchedulerService discovers there's nothing to schedule. This is an engine-side bug, not a devtown test issue.

**Bug 2:** Even after bypassing Bug 1, `CaseMemoryEmitter` `@ObservesAsync ReviewCompletedEvent` handler doesn't store facts to `CaseMemoryStore` — despite `ReviewCompletedEventCapture` confirming the event IS fired (Phase 3.5 passes). No error or warning logged from the emitter's catch block. CDI async observer delivery issue requiring further investigation.

Test remains `@Disabled` with updated tracking note. Fix belongs in the engine (Bug 1) and in the async observer chain (Bug 2).

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
   - Combine `requireAllowedTypes()` and `requireAllowedWriters()` into a single `requireContract()` method — validates both constraints in one call.
   - Use the existing `ORCHESTRATOR` constant directly — no separate `ORCHESTRATOR_WRITERS` constant. All three channels have the same writer set right now; when Layer 6 differentiates them, the constant structure will be revisited.
   - DONE/DECLINE dispatches changed to use `ORCHESTRATOR` as sender and `ActorType.SYSTEM` — in Layer 3 the orchestrator acts on behalf of agents; when Layer 6 wires real agents with their own identity, they will be added to `allowedWriters` and dispatch with their own sender.

2. **`PrReviewQhorusLifecycleTest`:**
   - Add assertions for `allowedWriters` on all three channels.
   - Add migration guard test for existing channels with null `allowedWriters`.
   - Update dispatch test senders to `ORCHESTRATOR` (consistent with Layer 3).
   - Fix pre-existing EVENT content violations (qhorus API change: EVENT messages must not carry content).

#### Migration

Existing channels (created before this change) will have `allowedWriters = null`. The `requireAllowedWriters()` method fails fast if an existing channel's writer contract doesn't match, same as `requireAllowedTypes()` does today. This surfaces stale channels rather than silently bypassing enforcement.

---

### #60 — Fluent Java DSL companions for devtown case definitions (M, Low)

#### Requirement

Protocol `case-definition-layers` mandates every YAML case definition has a companion fluent Java DSL builder. PLATFORM.md specifies: "Fluent DSL builders target the same canonical model and additionally support `LambdaExpressionEvaluator` (not expressible in YAML). All YAML definitions ⊂ fluent DSL; reverse is not true. Tests: build `CaseDefinition` directly via builders."

#### Existing Code

There is already a fluent DSL factory at `review/src/test/java/io/casehub/devtown/review/PrReviewCaseDefinition.java` — 177 lines, package-private, builds a complete `CaseDefinition` with 9 capabilities, 3 goals, 9 bindings, and uses `LambdaExpressionEvaluator` for binding conditions. It is used by `PrReviewBindingConditionTest` (28 unit tests covering all binding condition logic).

This existing class is already 90% of the protocol-mandated companion. Creating a second parallel class would produce naming confusion (`PrReviewCaseDefinition` vs `PrReviewCaseDefinitions`), duplicated structure, and drift risk. The right move is to promote and fix the existing class.

#### Design: Promote and Fix

**Promote** `PrReviewCaseDefinition` from `review/src/test/` to `review/src/main/`:

- Move to `review/src/main/java/io/casehub/devtown/review/PrReviewCaseDefinition.java`
- Change visibility from package-private to `public`
- Change `build(int humanApprovalThreshold)` to `public`

**Fix three divergences from the YAML:**

1. **`human-approval` binding uses wrong target type.** The existing class uses `.capability(humanDecisionCap)` but the YAML uses `humanTask:` with `title`, `candidateGroups`, `expiresIn`, and `outputMapping`. Fix to use `HumanTaskTarget.inline()`:

   ```java
   def.getBindings().add(Binding.builder().name("human-approval").on(trigger)
       .when(new LambdaExpressionEvaluator(ctx -> { ... }))
       .humanTask(HumanTaskTarget.inline()
           .title("PR approval required")
           .candidateGroups(Set.of("pr-reviewers"))
           .expiresIn(Duration.ofHours(24))
           .outputMapping("{ humanApproval: . }")
           .build())
       .build());
   ```

2. **Capability `inputSchema`/`outputSchema` are all `"{}"`**. The YAML defines specific schemas (e.g., code-analysis has `inputSchema: "{ pr: .pr }"`, `outputSchema: "{ codeAnalysis: . }"`). Fix each capability to match the YAML schemas exactly.

3. **Capability count mismatch.** The existing class includes 9 capabilities (including `human-decision:pr-approval`). The YAML lists 8 (no `human-decision:pr-approval` since it's a `humanTask` binding, not a `capability` binding). After fixing the `human-approval` binding to use `HumanTaskTarget`, remove `humanDecisionCap` from the capabilities list to match the YAML.

**Why lambdas, not JQ:** The protocol explicitly says "Fluent DSL builders additionally support `LambdaExpressionEvaluator` (not expressible in YAML)." The DSL companion's architectural value is precisely that it uses lambdas — enabling pure unit tests of binding conditions without a JQ runtime or Quarkus context. `PrReviewBindingConditionTest` (28 tests) depends on this. JQ conditions would make the DSL a string-for-string duplicate of the YAML with no additional value.

**No `dsl` field on the builder.** `CaseDefinition.Builder` has no `dsl()` method. The `dsl` field is set via `setDsl()` post-build. The DSL companion should call `def.setDsl("0.1")` after `build()` if the field is needed, or omit it — the `dsl` field is YAML metadata that has no runtime effect in the canonical model.

#### Test: Deep Structural Equivalence

Rename existing `PrReviewBindingConditionTest` references to use the new production path (import changes only — the class is the same, just moved).

Create `PrReviewCaseDefinitionEquivalenceTest` in `review/src/test/` — a plain JUnit test (no `@QuarkusTest`) verifying structural parity between the DSL companion and the YAML:

```java
class PrReviewCaseDefinitionEquivalenceTest {

    @Test
    void dslMatchesYaml() throws IOException {
        CaseDefinition fromYaml = CaseDefinitionYamlMapper.load(
            getClass().getClassLoader().getResourceAsStream("devtown/pr-review.yaml"));
        CaseDefinition fromDsl = PrReviewCaseDefinition.build(500);

        // Identity
        assertThat(fromDsl.getName()).isEqualTo(fromYaml.getName());
        assertThat(fromDsl.getNamespace()).isEqualTo(fromYaml.getNamespace());
        assertThat(fromDsl.getVersion()).isEqualTo(fromYaml.getVersion());

        // Capabilities — count, names, schemas
        assertThat(fromDsl.getCapabilities()).hasSameSizeAs(fromYaml.getCapabilities());
        for (int i = 0; i < fromYaml.getCapabilities().size(); i++) {
            var yamlCap = fromYaml.getCapabilities().get(i);
            var dslCap = fromDsl.getCapabilities().get(i);
            assertThat(dslCap.getName()).isEqualTo(yamlCap.getName());
            assertThat(dslCap.getInputSchema()).isEqualTo(yamlCap.getInputSchema());
            assertThat(dslCap.getOutputSchema()).isEqualTo(yamlCap.getOutputSchema());
        }

        // Goals — count, names, kinds
        assertThat(fromDsl.getGoals()).hasSameSizeAs(fromYaml.getGoals());
        for (int i = 0; i < fromYaml.getGoals().size(); i++) {
            assertThat(fromDsl.getGoals().get(i).getName())
                .isEqualTo(fromYaml.getGoals().get(i).getName());
            assertThat(fromDsl.getGoals().get(i).getKind())
                .isEqualTo(fromYaml.getGoals().get(i).getKind());
        }

        // Bindings — count, names, target types, non-null conditions
        assertThat(fromDsl.getBindings()).hasSameSizeAs(fromYaml.getBindings());
        for (int i = 0; i < fromYaml.getBindings().size(); i++) {
            var yamlBinding = fromYaml.getBindings().get(i);
            var dslBinding = fromDsl.getBindings().get(i);
            assertThat(dslBinding.getName()).isEqualTo(yamlBinding.getName());
            assertThat(dslBinding.target().getClass())
                .as("binding '%s' target type", yamlBinding.getName())
                .isEqualTo(yamlBinding.target().getClass());
            if (yamlBinding.getWhen() != null) {
                assertThat(dslBinding.getWhen())
                    .as("binding '%s' should have a when condition", yamlBinding.getName())
                    .isNotNull();
            }
            if (yamlBinding.target() instanceof HumanTaskTarget yamlHT
                    && dslBinding.target() instanceof HumanTaskTarget dslHT) {
                assertThat(dslHT.title()).isEqualTo(yamlHT.title());
                assertThat(dslHT.expiresIn()).isEqualTo(yamlHT.expiresIn());
            }
        }

        // Completion structure — same goal references
        assertThat(fromDsl.getCompletion()).isNotNull();
        assertThat(fromDsl.getCompletion().getClass())
            .isEqualTo(fromYaml.getCompletion().getClass());
    }
}
```

This test verifies everything that CAN be compared across evaluator types: names, counts, schemas, goal kinds, binding target types (CapabilityTarget vs HumanTaskTarget), condition presence, and completion structure. It intentionally does not compare evaluator equality (JQ vs lambda are structurally different objects).

#### Garden Gotcha

GE-20260531-d896bf: "SubCase M-of-N fields (groupId, totalInGroup, requiredCount, onThresholdReached) are DSL-only — not supported in YAML." Not relevant here since `pr-review` doesn't use subcases, but confirms the DSL can express things YAML cannot.

---

## Implementation Order

1. **#19, #61, #18** — close issues on GitHub (no code)
2. **#72** — fix CaseMemoryIntegrationTest (diagnostic-first)
3. **#64** — set allowedWriters on channels
4. **#60** — promote and fix fluent DSL companion

This order handles the smallest/diagnostic work first, then the structural channel change, then the larger DSL work.

---

## Out of Scope

- Layer 6 trust routing wiring (future — updates `allowedWriters` dynamically when agents are assigned)
- Composite `CapabilityRegistry` (deferred until ≥2 harnesses need it)
- `docs/PROGRESS.md` DT-003 update (file does not exist; tracked in issue description only)
