# CaseMemoryStore Integration — devtown#43

**Date:** 2026-06-05
**Issue:** casehubio/devtown#43
**Branch:** issue-43-case-memory-store
**Dependencies:** platform#27 (CaseMemoryStore SPI ✅ CLOSED), platform#48 (consumer gaps ✅ CLOSED)

---

## Problem

Every PR review case starts cold. Facts established in prior reviews — contributor patterns, agent behaviour, module-specific risk signals — are invisible unless someone explicitly remembers to look them up. Routing weights and WorkItem priorities cannot adapt to historical context.

## Solution

Integrate `CaseMemoryStore` (platform SPI) into devtown PR review cases. Two integration points:

1. **Emission** — after each reviewer binding completes (COMPLETED, DECLINED, or FAILED), store structured facts about the contributor, the reviewer, and the code areas touched.
2. **Recall** — before a new case opens, query prior facts for the contributor and code areas, enriching the initial case context so binding conditions can use historical signals.

## Scope

**MVP scope (this issue):** COMPLETED outcomes only. DECLINED and FAILED require engine/qhorus CDI events that may not exist yet — see Follow-up Issues.

Three entity types, all in the `software-review` domain:

| Entity type | Entity ID format | Example |
|---|---|---|
| Contributor | `contributor:<github-login>` | `contributor:mdproctor` |
| Reviewer | `reviewer:<actor-id>` | `reviewer:security-agent-1` |
| Code area | `module:<repo>/<module-path>` | `module:casehubio/devtown/app` |

**Code area granularity:** Module-level, not file-level. File-level is too fine for pattern detection — two PRs touching different files in `app/` would create different entities with no shared history.

**Module normalization** (shared utility in `devtown-domain`, class `ModulePathNormalizer`):

| Input path | Normalised module |
|---|---|
| `app/src/main/java/io/casehub/.../Foo.java` | `app` |
| `app/src/test/java/io/casehub/.../FooTest.java` | `app` |
| `domain/src/main/java/.../Bar.java` | `domain` |
| `pom.xml` | _(root)_ |
| `config/application.yaml` | _(root)_ |
| `.github/workflows/ci.yml` | _(root)_ |
| `README.md` | _(root)_ |

Rule: find the first `/src/` in the path. Everything before it is the module name (e.g., `app/src/main/...` → `app`). If the path has no `/src/`, use `(root)`. `src/test/` and `src/main/` map to the same module entity. Multiple changed files in the same module produce one entity, not N.

Tradeoff: coarser granularity may merge unrelated risk signals within a large module — acceptable for initial implementation; refine if recall quality suffers.

## Domain Model

### MemoryDomain

```java
// devtown-domain
public final class DevtownMemoryDomain {
    public static final MemoryDomain SOFTWARE_REVIEW = new MemoryDomain("software-review");
    private DevtownMemoryDomain() {}
}
```

### ReviewOutcome enum

```java
// devtown-domain
public enum ReviewOutcome {
    COMPLETED, DECLINED, FAILED
}
```

Eliminates string literal comparisons throughout the emission and recall code.

### Attribute keys

**Platform reserved keys (use as-is from `MemoryAttributeKeys`):**
- `MemoryAttributeKeys.ACTOR_ID` — reviewer identity (agent ID or human ID)
- `MemoryAttributeKeys.ACTOR_ROLE` — `reviewer` or `contributor`
- `MemoryAttributeKeys.OUTCOME` — `COMPLETED`, `DECLINED`, `FAILED` (via `ReviewOutcome.name()`)
- `MemoryAttributeKeys.CONFIDENCE` — formatted via `MemoryAttributeKeys.formatConfidence()`

**Devtown-specific keys (kebab-case, defined in `DevtownMemoryKeys`):**

| Key | Purpose |
|---|---|
| `capability` | Which capability was exercised (`security-review`, `style-review`, etc.) |
| `pr-number` | The PR that triggered this memory |
| `pr-repo` | The repository |
| `lines-changed` | PR size context |
| `entity-type` | `contributor`, `reviewer`, or `code-area` — for filtering on recall |
| `outcome-detail` | Raw outcome value from case context (e.g., `APPROVED`, `FINDINGS_PRESENT`, `sla-breach`). Used by `hasRiskSignals()` to distinguish clean completions from findings. |

### PrPayload enhancement

Two fields added for both recall (contributor identity) and storage (code area entities):

```java
public record PrPayload(
    String repo, int prNumber, String headSha, String baseRef, int linesChanged,
    String contributor,          // GitHub login
    List<String> changedPaths    // file paths changed in the PR
) {}
```

Existing callers updated to supply the new fields. Tests updated accordingly.

## Emission Architecture

### Two-layer design: extraction observer + memory emitter

The extraction problem — pulling structured reviewer outcome data from engine internals (untyped case context, async observer context, reactive repositories) — is the hardest part of this integration. It is isolated in ONE component (the extraction observer) and exposed via a typed intermediate event. The memory emitter is a pure function from typed input to `List<MemoryInput>`.

```
Engine events (multiple) → ReviewOutcomeObserver (extraction, one place touching engine internals)
  → fires ReviewCompletedEvent (typed, all context)
    → CaseMemoryEmitter @ObservesAsync (pure function, plain unit testable)
      → CaseMemoryStore.storeAll()
```

**Why the intermediate event is not YAGNI:** Layer 6 (trust routing) is the next layer on the roadmap and needs the same structured review outcome data. Building extraction inside the emitter means Layer 6 either duplicates it or couples to the emitter. `ReviewCompletedEvent` isolates the extraction in one place and gives both consumers a clean typed event.

**Latency profile:** Two async hops — `PlanItemCompletedEvent` (fireAsync) → observer → `ReviewCompletedEvent` (fireAsync) → emitter — with a blocking `Uni.await()` in between. For append-only memory this is acceptable: ordering is not critical, and the managed executor pool handles concurrent binding completions. If many bindings complete simultaneously on a small executor pool, memory emission queues behind other async work. This is fine — memory is enrichment, not latency-sensitive.

### ReviewCompletedEvent

Defined in `devtown-review` (integration module):

```java
public record ReviewCompletedEvent(
    UUID caseId,
    String tenantId,
    String capability,         // "security-review", "style-review", etc.
    String reviewerId,         // agent ID or human ID
    ReviewOutcome outcome,     // COMPLETED, DECLINED, FAILED
    String outcomeDetail,      // findings summary or decline reason (nullable)
    PrPayload pr               // full PR context including contributor and changedPaths
) {}
```

Carries `tenantId` explicitly — solves the `CurrentPrincipal` unavailability in async observers (request scope is not propagated to `@ObservesAsync` threads). The extraction observer extracts `tenantId` from the `CaseInstance` after lookup.

### ReviewOutcomeObserver — the extraction component

Defined in `devtown-app`. The single component that touches engine internals. This is the hardest component to implement and the most detailed section of the spec.

**Engine events observed:**

| Engine event | Outcome | Status |
|---|---|---|
| `PlanItemCompletedEvent` | COMPLETED | **MVP — this issue** |
| Engine PlanItem fault/rejection event | FAILED | **Follow-up** — file engine issue |
| Qhorus commitment DECLINED event | DECLINED | **Follow-up** — file qhorus issue |

#### Outcome extraction: the planItemId → context key mapping

This is the single hardest piece of implementation logic. `PlanItemCompletedEvent` carries `caseId`, `planItemId`, and `trackingKey`. None of these is the outcome — the outcome lives in the case context after the binding's `outputMapping` was applied.

**Mapping approach:** Hard-coded static map from `planItemId` to context key path. The PR review case definition is static YAML — the set of bindings and their output mappings are known at compile time. When a new capability is added, it requires a YAML change AND a map entry. This coupling is acceptable and documented.

```java
private static final Map<String, String> PLAN_ITEM_TO_CONTEXT_KEY = Map.of(
    "security-review",       "securityReview.outcome",
    "architecture-review",   "architectureReview.outcome",
    "style-check",           "styleCheck.outcome",
    "test-coverage",         "testCoverage.outcome",
    "performance-analysis",  "performanceAnalysis.outcome",
    "human-approval",        "humanApproval.status"
);
```

**Key naming rationale:** The context keys are camelCase because that's what the JQ outputMappings produce (e.g., `"{ securityReview: { outcome: . } }"`). The `humanApproval` binding uses `.status` instead of `.outcome` because it was established in Layer 2 before the outcome convention was set. This inconsistency is real and documented — not a placeholder.

**Extraction steps (for each PlanItemCompletedEvent):**

1. **Filter:** If `event.planItemId()` is not in `PLAN_ITEM_TO_CONTEXT_KEY`, skip silently. This filters infrastructure bindings (`initial-analysis`, `run-ci`, `merge`) and any unknown planItemIds. No logging for expected skips.

2. **Lookup CaseInstance:** `caseRepo.findByUuid(event.caseId()).await().atMost(Duration.ofSeconds(5))`. Returns `Uni<CaseInstance>` — block on the managed executor thread. If lookup fails or times out, log at WARN and return (do not fire event).

3. **Extract tenantId:** `caseInstance.getTenancyId()`.

4. **Extract PR metadata:** Navigate case context: `caseInstance.getCaseContext().getPath("pr.repo")`, `getPath("pr.id")`, `getPath("pr.linesChanged")`, `getPath("pr.baseRef")`, `getPath("pr.headSha")`, `getPath("pr.contributor")`, `getPath("pr.changedPaths")`. Reconstruct `PrPayload` from these values. If any required field is missing, log at WARN and return.

5. **Extract outcome:** Navigate case context using the mapped key path: `caseInstance.getCaseContext().getPath(contextKeyPath)`. The value is a string (e.g., `"APPROVED"`, `"FINDINGS_PRESENT"`, `"approved"`, `"sla-breach"`). This is the raw outcome from the binding's outputMapping. Map to `ReviewOutcome.COMPLETED` (since this observer only handles PlanItemCompletedEvent).

6. **Extract reviewerId:** `event.trackingKey()` — this is the workerName or equivalent identifier for the actor that completed the PlanItem. If null, use `"unknown"`.

7. **Extract outcomeDetail:** For COMPLETED outcomes, the detail is the raw outcome value from the context (e.g., `"FINDINGS_PRESENT"` carries more meaning than `ReviewOutcome.COMPLETED` alone). For future FAILED/DECLINED observers, the detail is the failure reason or decline reason.

8. **Fire event:** `reviewCompletedEvents.fireAsync(new ReviewCompletedEvent(...))`.

**When the mapping can't resolve a planItemId:** Silent skip. This is the expected path for infrastructure bindings — not an error condition. An unknown planItemId that IS a review binding (i.e., a new capability was added to the YAML but not the map) is a maintenance oversight. It fails silently (no memory emitted), which is safe — memory is enrichment. A follow-up improvement could log at DEBUG for unknown planItemIds that aren't in a known-infrastructure set.

**CrossTenantCaseInstanceRepository:** Used because no request-scoped tenant is available in the async observer. This is a contract violation (the repository's Javadoc says "for startup recovery services only"). Documented as accepted technical debt — see Follow-up Issues for the resolution path.

### CaseMemoryEmitter

Defined in `devtown-app`. Receives typed `ReviewCompletedEvent` — no engine internals, no case context parsing, no reactive types.

```java
@ApplicationScoped
public class CaseMemoryEmitter {

    @Inject Instance<CaseMemoryStore> store;

    void onReviewCompleted(@ObservesAsync ReviewCompletedEvent event) {
        if (!store.isResolvable()) return;
        var s = store.get();
        try {
            List<MemoryInput> facts = buildFacts(event);
            s.storeAll(facts);
        } catch (Exception e) {
            LOG.warnf(e, "Memory emission failed for case=%s capability=%s — non-critical, continuing",
                event.caseId(), event.capability());
        } finally {
            store.destroy(s);
        }
    }
}
```

**Key design decisions:**
- `Instance<CaseMemoryStore>` — matches engine's `CaseMemoryObserver` pattern; handles edge cases where bean can't be resolved; properly manages dependent-scoped instances via `destroy()`.
- `storeAll()` — single batch call for 1 contributor + 1 reviewer + N module entities. Avoids 2+N network round-trips.
- `caseId` populated on every `MemoryInput` — free provenance linking memory to the specific case.
- Failure: logged at WARN and swallowed. Memory is enrichment, not critical path.

**Text construction** — natural language for semantic embedding quality (not structured log lines). Code area text does NOT include contributor login (avoids GDPR cross-entity scrubbing — see GDPR section):
- Contributor: `"Security review of a 342-line pull request by mdproctor in casehubio/devtown found issues requiring attention."`
- Reviewer: `"security-agent-1 completed a security review of pull request #45 in casehubio/devtown. Outcome: approved with no findings."`
- Code area: `"The app module in casehubio/devtown received a security review on pull request #45. No issues found."`

Structured metadata goes in `attributes` (platform reserved keys + devtown-specific keys). The `text` field is specifically for vector embedding — it must be a readable sentence, not a template-filled log line.

**Module normalization for code area entities:** `changedPaths` are deduplicated to module-level via `ModulePathNormalizer` (shared utility in `devtown-domain`) before constructing code area `MemoryInput` entries. E.g., if a PR touches `app/src/main/.../Foo.java` and `app/src/main/.../Bar.java`, only one code area entity (`module:casehubio/devtown/app`) is created.

## Recall Architecture

### CaseMemoryRecaller

Called in `PrReviewCaseService.review()` before `caseHub.startCase()`. Runs in request scope — `CurrentPrincipal` is available.

```java
@ApplicationScoped
public class CaseMemoryRecaller {

    @Inject Instance<CaseMemoryStore> store;
    @Inject CurrentPrincipal principal;

    public MemoryContext recall(PrPayload pr) {
        if (!store.isResolvable()) return MemoryContext.EMPTY;
        var s = store.get();
        try {
            String tenantId = principal.tenancyId();

            List<Memory> contributorHistory = s.query(
                MemoryQuery.forEntity(
                        "contributor:" + pr.contributor(),
                        SOFTWARE_REVIEW, tenantId)
                    .withLimit(10)
                    .withSince(Instant.now().minus(Duration.ofDays(90)))
                    .withOrder(MemoryOrder.CHRONOLOGICAL));

            List<String> moduleIds = ModulePathNormalizer.normalize(pr.changedPaths()).stream()
                .map(m -> "module:" + pr.repo() + "/" + m)
                .limit(MemoryQuery.MAX_ENTITY_IDS)
                .toList();

            List<Memory> codeAreaHistory = moduleIds.isEmpty()
                ? List.of()
                : s.query(
                    MemoryQuery.forEntities(moduleIds, SOFTWARE_REVIEW, tenantId)
                        .withLimit(15)
                        .withSince(Instant.now().minus(Duration.ofDays(90)))
                        .withOrder(MemoryOrder.RELEVANCE)
                        .withQuestion("review history for " + String.join(", ", ModulePathNormalizer.normalize(pr.changedPaths())) + " in " + pr.repo()));

            return new MemoryContext(contributorHistory, codeAreaHistory);
        } catch (Exception e) {
            LOG.warnf(e, "Memory recall failed for contributor=%s — proceeding without memory",
                pr.contributor());
            return MemoryContext.EMPTY;
        } finally {
            store.destroy(s);
        }
    }
}
```

**Design decisions:**
- `Instance<CaseMemoryStore>` — matches emitter pattern and engine convention.
- **90-day time window** (`withSince`) — bounds recall to recent context. Prevents noise from stale facts accumulated over months. Configurable via preferences in a follow-up if needed.
- **Contributor: limit 10, CHRONOLOGICAL** — last 10 reviews by this contributor within 90 days. Rationale: case-open enrichment needs a snapshot of recent contributor patterns, not exhaustive history. 10 is roughly 2–3 months of weekly PRs.
- **Code areas: limit 15, RELEVANCE** — top 15 most relevant module-level facts. RELEVANCE with a question parameter lets semantic adapters surface topically relevant history (security findings, review issues) rather than just most recent. Non-semantic adapters fall back to CHRONOLOGICAL silently.
- **Query failure: fail-open** — log at WARN, return `MemoryContext.EMPTY`, case opens without memory. Memory is enrichment; a failed query must never prevent case creation.

**No reviewer history on recall** — reviewer agent history is stored but not recalled at case open. It is useful for trust-weighted routing (Layer 6), not for case-open enrichment. When Layer 6 integrates trust routing with memory, it will query `reviewer:*` entities directly.

### MemoryContext

Pure Java record in `devtown-review`:

```java
public record MemoryContext(
    List<Memory> contributorHistory,
    List<Memory> codeAreaHistory
) {
    public static final MemoryContext EMPTY = new MemoryContext(List.of(), List.of());

    public Map<String, Object> toContextMap() { ... }
    public boolean hasRiskSignals() { ... }
}
```

`toContextMap()` produces a map with two keys:
- `contributorHistory` — list of maps, each with `text` (from `Memory.text()`), `outcome` (from `Memory.attributes().get(MemoryAttributeKeys.OUTCOME)`), `capability` (from attributes), `createdAt` (from `Memory.createdAt()`)
- `codeAreaHistory` — list of maps, same structure

Injected as the `memory` key in the initial case context. Binding conditions can reference `memory.contributorHistory` and `memory.codeAreaHistory`.

`hasRiskSignals()` returns true if any recalled fact has `outcome == ReviewOutcome.FAILED.name()`, or has `outcome == ReviewOutcome.COMPLETED.name()` with an `outcome-detail` attribute value NOT in the known-safe set. Known-safe values: `APPROVED`, `passed`, `approved`. Everything else (e.g., `FINDINGS_PRESENT`, `sla-breach`, any new findings-type value) is treated as a risk signal — fail-closed, so unrecognised outcome details default to risk rather than slipping through. A DECLINED review is a routing signal (scope boundary), not a risk signal — it does not indicate contributor risk.

### Integration into PrReviewCaseService

```java
var memoryContext = recaller.recall(pr);
var initialContext = Map.of(
    "pr", prContext,
    "policy", policy,
    "memory", memoryContext.toContextMap()
);
caseHub.startCase(initialContext);
```

### Graceful degradation

If `CaseMemoryStore` is the no-op (no adapter installed), `Instance.isResolvable()` returns true (the `@DefaultBean` no-op is always present), `query()` returns empty lists, `MemoryContext` is empty. Case opens normally with no memory enrichment. Zero overhead.

## Relationship to engine CaseMemoryObserver

The engine already provides a `CaseMemoryObserver` that stores case-level memories (CaseCompleted/Cancelled/Failed) in a `case-lifecycle` domain. Devtown's integration is a different concern:

| | Engine CaseMemoryObserver | Devtown CaseMemoryEmitter |
|---|---|---|
| Domain | `case-lifecycle` | `software-review` |
| Granularity | Case-level (one fact per case completion) | Binding-level (one fact set per reviewer outcome) |
| Entity model | Case ID as entity | Contributor, reviewer, code area entities |
| Attributes | Generic case metadata | Structured review attributes (capability, PR number, etc.) |

No conflict — different domains, different granularity, different entity models. The engine observer provides generic case lifecycle memory; devtown provides domain-specific review outcome memory.

## GDPR Erasure

Contributor facts are keyed by `contributor:<github-login>`. Under GDPR Art.17, a contributor can request deletion.

**Deletion mechanism:** `CaseMemoryStore.eraseEntity("contributor:mdproctor", tenantId)` — hard-deletes all contributor facts across the `software-review` domain.

**Cross-entity impact:** Code area facts do NOT include the contributor login in the `text` field (see text construction examples — code area text omits contributor identity by design). Reviewer facts reference the reviewer identity, not the contributor. Therefore, `eraseEntity()` on the contributor entity is sufficient — no cross-entity scrubbing required.

**Not in scope for this issue:** Building a GDPR erasure endpoint. The `eraseEntity()` mechanism exists in the SPI; this spec documents how to use it. An admin endpoint or scheduled erasure job is a separate concern.

## Module Placement

| Module | New classes | Rationale |
|---|---|---|
| `devtown-domain` | `DevtownMemoryDomain`, `DevtownMemoryKeys`, `ReviewOutcome`, `ModulePathNormalizer` | Pure Java, zero framework deps |
| `devtown-review` | `PrPayload` enhanced, `MemoryContext`, `ReviewCompletedEvent` | Integration logic; `casehub-platform-api` on classpath for `Memory` type |
| `devtown-app` | `ReviewOutcomeObserver`, `CaseMemoryEmitter`, `CaseMemoryRecaller` | CDI wiring — observe engine events, inject `CaseMemoryStore` |

**Dependencies added:**
- `devtown-review` pom: `casehub-platform-api` — make explicit (likely already transitive)
- `devtown-app` pom: no new deps — `casehub-engine` (blackboard events) and `casehub-platform` (no-op bean) already present

## Testing

| Test | Type | Verifies |
|---|---|---|
| `ModulePathNormalizerTest` | Unit (domain) | All normalization cases from the table above; deduplication; edge cases (empty list, root-only files) |
| `DevtownMemoryDomainTest` | Unit (domain) | Domain constant, entity ID formatting, `ReviewOutcome` enum values, attribute key conventions |
| `MemoryContextTest` | Unit (review) | `toContextMap()` serialisation (text from `Memory.text()`, outcome/capability from attributes, createdAt from `Memory.createdAt()`), `hasRiskSignals()` logic (FAILED = risk, COMPLETED + unknown detail = risk, COMPLETED + APPROVED = safe, DECLINED = not risk), `EMPTY` constant, empty-list handling |
| `CaseMemoryEmitterTest` | Unit (app) | Mock `ReviewCompletedEvent` → verify `storeAll()` called with correct `MemoryInput` list; verify batch includes contributor + reviewer + deduplicated modules; verify `caseId` set on each `MemoryInput`; verify platform reserved keys used (`MemoryAttributeKeys.OUTCOME`, `.ACTOR_ID`); verify `Instance.isResolvable()` guard; verify failure swallowed |
| `ReviewOutcomeObserverTest` | `@QuarkusTest` (app) | Fire `PlanItemCompletedEvent` → verify `ReviewCompletedEvent` fired with correct fields extracted from case context; verify `PLAN_ITEM_TO_CONTEXT_KEY` mapping resolves correctly; verify infrastructure bindings (unknown planItemId) skipped silently; verify CaseInstance lookup failure swallowed; verify timeout on `Uni.await()` |
| `CaseMemoryRecallerTest` | `@QuarkusTest` (app) | Pre-populate store → `recall()` → verify `MemoryContext` populated with correct entity IDs, domain, time window (90 days), ordering; verify empty store → `MemoryContext.EMPTY`; verify query failure → `MemoryContext.EMPTY` (fail-open) |
| `CaseMemoryIntegrationTest` | `@QuarkusTest` (app) | Round-trip: start case → binding completes → observer fires → emitter stores → start second case for same contributor → recall returns prior facts in initial context |

Pre-seeding note: tests that start cases must pre-seed parallel check keys with non-null values (PP-20260521-134c38).

`CaseMemoryEmitterTest` is a plain unit test (no `@QuarkusTest`) — the emitter receives a typed event and produces a `List<MemoryInput>`. No engine, no case context, no reactive types. This is the primary benefit of the `ReviewCompletedEvent` separation.

## Follow-up Issues

- **Engine:** promote `PlanItemCompletedEvent` to `engine-common-spi` as a stable extension point
- **Engine:** add `tenantId` to `PlanItemCompletedEvent` — eliminates the need for `CrossTenantCaseInstanceRepository` in the observer (resolves the contract violation)
- **Engine:** add CDI event for PlanItem fault/rejection (needed for FAILED outcome emission)
- **Qhorus:** add CDI event for commitment DECLINED (needed for DECLINED outcome emission)
- **Enriched outputMappings:** if compressed outcomes prove insufficient for `ReviewCompletedEvent.outcomeDetail`, enrich the YAML outputMappings to preserve more agent output in case context (declarative change, no code)
- **Recall limits as preferences:** make contributor limit (10), code area limit (15), and time window (90 days) configurable via `PreferenceKey` definitions
- **GDPR erasure endpoint:** admin endpoint or scheduled job to handle Art.17 requests using the documented `eraseEntity()` mechanism
