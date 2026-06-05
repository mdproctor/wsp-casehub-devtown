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

1. **Emission** — after each reviewer binding completes, store structured facts about the contributor, the reviewer, and the code areas touched.
2. **Recall** — before a new case opens, query prior facts for the contributor and code areas, enriching the initial case context so binding conditions can use historical signals.

## Scope

Three entity types, all in the `software-review` domain:

| Entity type | Entity ID format | Example |
|---|---|---|
| Contributor | `contributor:<github-login>` | `contributor:mdproctor` |
| Reviewer | `reviewer:<actor-id>` | `reviewer:security-agent-1` |
| Code area | `path:<repo>/<relative-path>` | `path:casehubio/devtown/src/main/java/.../app/` |

## Domain Model

### MemoryDomain

```java
// devtown-domain
public final class DevtownMemoryDomain {
    public static final MemoryDomain SOFTWARE_REVIEW = new MemoryDomain("software-review");
    private DevtownMemoryDomain() {}
}
```

### Devtown-specific attribute keys

Kebab-case convention (matching `MemoryAttributeKeys` platform reserved keys).

| Key | Purpose |
|---|---|
| `capability` | Which capability was exercised (`security-review`, `style-review`, etc.) |
| `pr-number` | The PR that triggered this memory |
| `pr-repo` | The repository |
| `lines-changed` | PR size context |
| `entity-type` | `contributor`, `reviewer`, or `code-area` — for filtering on recall |

Defined as constants in `DevtownMemoryKeys` (devtown-domain, pure Java).

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

### Observation point: PlanItemCompletedEvent

Single CDI observer on `PlanItemCompletedEvent` (engine blackboard module). This event:
- Fires for every binding completion (agent and human) — universal
- Fires AFTER outputMapping has been applied to case context — timing correct
- Is a CDI `fireAsync` event with zero production observers — clean extension point
- Carries `caseId`, `planItemId`, `trackingKey`

**Dependency note:** `PlanItemCompletedEvent` is in `io.casehub.blackboard.event` (not public SPI). File engine issue to promote to `engine-common-spi`. Coupling is acceptable until then — same org, stable 3-field record.

### CaseMemoryEmitter

```java
@ApplicationScoped
public class CaseMemoryEmitter {

    @Inject CaseMemoryStore store;
    @Inject CrossTenantCaseInstanceRepository caseRepo;

    void onPlanItemCompleted(@ObservesAsync PlanItemCompletedEvent event) {
        // 1. Skip non-review bindings (run-ci, merge, initial-analysis)
        // 2. Query CaseInstance for case context
        // 3. Extract: PR metadata, contributor, changedPaths, outcome for this binding
        // 4. Store contributor fact
        // 5. Store reviewer fact
        // 6. Store code area facts (one per changedPath)
        // Failure: log and swallow — memory is enrichment, not critical path
    }
}
```

**Which bindings emit memory:** `security-review`, `architecture-review`, `style-check`, `test-coverage`, `performance-analysis`, `human-approval`. Infrastructure bindings (`initial-analysis`, `run-ci`, `merge`) skipped. Filtering via `DevtownCapabilityRegistry.MEMORY_RELEVANT_CAPABILITIES` set in devtown-domain.

**Text construction** (natural language, required for semantic adapters):
- Contributor: `"Contributor {login}: {capability} {outcome} on PR #{number} ({repo}, {linesChanged} LOC)"`
- Reviewer: `"{agentId} reviewed PR #{number} ({repo}): {outcome}"`
- Code area: `"{path}: {capability} {outcome} on PR #{number}"`

**Failure handling:** `store()` failure is logged and swallowed. Memory emission must never block case progression.

### Why this observation point

| Alternative | Why not |
|---|---|
| `WorkflowExecutionCompleted` | Engine internal (EventBus, not CDI); fires before PlanItem completion; tighter coupling |
| `CaseContextChangedEvent` | EventBus not CDI; fires on every context mutation (noisy); no binding-completion semantics |
| `CaseLifecycleEvent` | Wrong granularity — case-level transitions, not binding outcomes |
| Per-worker-type observers | Two implementations for the same concern; doesn't extend to new worker types |
| Intermediate `ReviewCompletedEvent` | YAGNI — memory is the only consumer today; add later if other concerns need it |
| Platform `MemoryStoreRequest` (Option C) | Premature abstraction; platform module doesn't exist; adds mapping layer with zero current benefit |

## Recall Architecture

### CaseMemoryRecaller

Called in `PrReviewCaseService.review()` before `caseHub.startCase()`.

```java
@ApplicationScoped
public class CaseMemoryRecaller {

    @Inject CaseMemoryStore store;
    @Inject CurrentPrincipal principal;

    public MemoryContext recall(PrPayload pr) {
        String tenantId = principal.tenancyId();

        // Contributor history (single-entity query)
        List<Memory> contributorHistory = store.query(
            MemoryQuery.forEntity("contributor:" + pr.contributor(), SOFTWARE_REVIEW, tenantId)
                .withLimit(10)
                .withOrder(MemoryOrder.CHRONOLOGICAL));

        // Code area history (multi-entity query, max 25)
        List<String> areaIds = pr.changedPaths().stream()
            .map(p -> "path:" + pr.repo() + "/" + p)
            .limit(MemoryQuery.MAX_ENTITY_IDS)
            .toList();

        List<Memory> codeAreaHistory = areaIds.isEmpty()
            ? List.of()
            : store.query(
                MemoryQuery.forEntities(areaIds, SOFTWARE_REVIEW, tenantId)
                    .withLimit(15)
                    .withOrder(MemoryOrder.CHRONOLOGICAL));

        return new MemoryContext(contributorHistory, codeAreaHistory);
    }
}
```

### MemoryContext

Pure Java record in `devtown-review`:

```java
public record MemoryContext(
    List<Memory> contributorHistory,
    List<Memory> codeAreaHistory
) {
    public Map<String, Object> toContextMap() { ... }
    public boolean hasRiskSignals() { ... }
}
```

`toContextMap()` produces a map with two keys:
- `contributorHistory` — list of maps, each with `text`, `outcome`, `capability`, `createdAt` (extracted from `Memory` attributes)
- `codeAreaHistory` — list of maps, same structure

Injected as the `memory` key in the initial case context. Binding conditions can reference `memory.contributorHistory` and `memory.codeAreaHistory`. `hasRiskSignals()` returns true if any recalled fact has `outcome` != `APPROVED` — used to flag elevated-risk PRs.

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

### What is not recalled

Reviewer agent history is stored but not recalled at case open. It is useful for trust-weighted routing (Layer 6), not for case-open enrichment. When Layer 6 integrates trust routing with memory, it will query `reviewer:*` entities directly.

### Graceful degradation

If `CaseMemoryStore` is the no-op (no adapter installed), `query()` returns empty lists. `MemoryContext` is empty. The case opens normally with no memory enrichment. Zero overhead.

## Module Placement

| Module | New classes | Rationale |
|---|---|---|
| `devtown-domain` | `DevtownMemoryDomain`, `DevtownMemoryKeys`, memory-relevant capability set in `DevtownCapabilityRegistry` | Pure Java, zero framework deps |
| `devtown-review` | `PrPayload` enhanced, `MemoryContext` record | Integration logic; `casehub-platform-api` on classpath for `Memory` type |
| `devtown-app` | `CaseMemoryEmitter`, `CaseMemoryRecaller` | CDI wiring — inject `CaseMemoryStore`, observe engine events |

**Dependencies added:**
- `devtown-review` pom: `casehub-platform-api` — make explicit (likely already transitive)
- `devtown-app` pom: no new deps — `casehub-engine` (blackboard events) and `casehub-platform` (no-op bean) already present

## Testing

| Test | Type | Verifies |
|---|---|---|
| `DevtownMemoryDomainTest` | Unit (domain) | Domain constant, entity ID formatting, attribute key conventions |
| `MemoryContextTest` | Unit (review) | `toContextMap()` serialisation, `hasRiskSignals()` logic, empty-list graceful handling |
| `CaseMemoryEmitterTest` | `@QuarkusTest` (app) | PlanItemCompletedEvent → store() called with correct MemoryInput per entity type; infrastructure bindings skipped; store() failure swallowed |
| `CaseMemoryRecallerTest` | `@QuarkusTest` (app) | Pre-populate store → recall() → MemoryContext populated; empty store → empty context |
| `CaseMemoryIntegrationTest` | `@QuarkusTest` (app) | Round-trip: start case → binding completes → emitter fires → facts stored → start second case for same contributor → recall returns prior facts |

Pre-seeding note: tests that start cases must pre-seed parallel check keys with non-null values (PP-20260521-134c38).

## Follow-up Issues

- **Engine:** promote `PlanItemCompletedEvent` to `engine-common-spi` as a stable extension point
- **Enriched outputMappings:** if compressed outcomes prove insufficient for memory fact quality, enrich the YAML outputMappings to preserve more agent output in case context (declarative change, no code)
- **Intermediate domain event:** if other concerns (notifications, dashboards, trust scoring) need reviewer outcome events, introduce `ReviewCompletedEvent` between PlanItemCompletedEvent and CaseMemoryEmitter
