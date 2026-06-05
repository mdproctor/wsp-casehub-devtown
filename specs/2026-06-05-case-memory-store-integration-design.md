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

Three entity types, all in the `software-review` domain:

| Entity type | Entity ID format | Example |
|---|---|---|
| Contributor | `contributor:<github-login>` | `contributor:mdproctor` |
| Reviewer | `reviewer:<actor-id>` | `reviewer:security-agent-1` |
| Code area | `module:<repo>/<module-path>` | `module:casehubio/devtown/app` |

**Code area granularity:** Module-level (Maven module or top-level package directory), not file-level. File-level is too fine for pattern detection — two PRs touching different files in `app/` would create different entities with no shared history. Normalization: strip file names, keep directory path up to the first `src/` boundary (e.g., `app/src/main/java/io/casehub/devtown/app/Foo.java` → `app`). Multiple changed files in the same module produce one entity, not N. Tradeoff: coarser granularity may merge unrelated risk signals within a large module — acceptable for initial implementation; refine if recall quality suffers.

## Domain Model

### MemoryDomain

```java
// devtown-domain
public final class DevtownMemoryDomain {
    public static final MemoryDomain SOFTWARE_REVIEW = new MemoryDomain("software-review");
    private DevtownMemoryDomain() {}
}
```

### Attribute keys

**Platform reserved keys (use as-is from `MemoryAttributeKeys`):**
- `actor-id` — reviewer identity (agent ID or human ID)
- `actor-role` — `reviewer` or `contributor`
- `outcome` — `COMPLETED`, `DECLINED`, `FAILED`
- `confidence` — formatted via `MemoryAttributeKeys.formatConfidence()`

**Devtown-specific keys (kebab-case, defined in `DevtownMemoryKeys`):**

| Key | Purpose |
|---|---|
| `capability` | Which capability was exercised (`security-review`, `style-review`, etc.) |
| `pr-number` | The PR that triggered this memory |
| `pr-repo` | The repository |
| `lines-changed` | PR size context |
| `entity-type` | `contributor`, `reviewer`, or `code-area` — for filtering on recall |

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

### ReviewCompletedEvent

Defined in `devtown-review` (integration module):

```java
public record ReviewCompletedEvent(
    UUID caseId,
    String tenantId,
    String capability,         // "security-review", "style-review", etc.
    String reviewerId,         // agent ID or human ID
    String outcome,            // "COMPLETED", "DECLINED", "FAILED"
    String outcomeDetail,      // findings summary or decline reason
    PrPayload pr               // full PR context including contributor and changedPaths
) {}
```

Carries `tenantId` explicitly — solves the `CurrentPrincipal` unavailability in async observers (request scope is not propagated to `@ObservesAsync` threads). The extraction observer extracts `tenantId` from the `CaseInstance` after lookup.

### ReviewOutcomeObserver

Defined in `devtown-app`. The single component that touches engine internals. Observes multiple engine events to capture all outcome types:

| Engine event | Outcome | How tenantId is obtained |
|---|---|---|
| `PlanItemCompletedEvent` | COMPLETED | `caseInstance.getTenancyId()` after lookup |
| Engine PlanItem fault/rejection event (TBD — investigate engine event model) | FAILED | Same |
| Qhorus commitment DECLINED event (TBD — investigate qhorus event model) | DECLINED | Same |

**For each observed event:**
1. Look up CaseInstance via repository — use `CrossTenantCaseInstanceRepository.findByUuid()` with explicit acknowledgement that this is a cross-tenant lookup in an async context (no request-scoped tenant available). The `findByUuid()` call returns `Uni<CaseInstance>` — block with `.await().atMost(Duration.ofSeconds(5))` on the managed executor thread (Quarkus virtual threads support this).
2. Extract `tenantId` from `CaseInstance.getTenancyId()`.
3. Extract PR metadata and contributor from case context (`pr.*` keys).
4. Map `planItemId` to capability name via case context key convention (the outcome key in case context matches the binding name, e.g., `securityReview.outcome`).
5. Extract outcome value from case context — the specific key depends on the capability (e.g., `securityReview.outcome`, `styleCheck.outcome`, `humanApproval.status`). The observer maintains a mapping of `planItemId → context key` derived from the case definition's outputMappings.
6. Skip if capability is not review-relevant (infrastructure bindings: `initial-analysis`, `run-ci`, `merge`). Filter via a private constant set on the observer — not in the capability registry.
7. Fire `ReviewCompletedEvent` via CDI `Event.fireAsync()`.

**Failure handling:** If CaseInstance lookup fails or case context extraction fails, log the error and do not fire the event. The memory emitter never sees the failure.

**PlanItem fault/rejection and qhorus DECLINED events:** These require investigation of the engine and qhorus event models to identify the correct CDI events to observe. If no suitable CDI event exists for DECLINED/FAILED, file engine/qhorus issues to add them. The extraction observer registers additional `@ObservesAsync` methods as events become available. The initial implementation may ship with COMPLETED-only and tracked issues for the remaining outcomes.

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

**Text construction** — natural language for semantic embedding quality (not structured log lines):
- Contributor: `"Security review of a 342-line pull request by mdproctor in casehubio/devtown found issues requiring attention."`
- Reviewer: `"security-agent-1 completed a security review of pull request #45 in casehubio/devtown. Outcome: approved with no findings."`
- Code area: `"The app module in casehubio/devtown received a security review on pull request #45. No issues found."`

Structured metadata goes in `attributes` (platform reserved keys + devtown-specific keys). The `text` field is specifically for vector embedding — it must be a readable sentence, not a template-filled log line.

**Module normalization for code area entities:** `changedPaths` are deduplicated to module-level before constructing code area `MemoryInput` entries. E.g., if a PR touches `app/src/main/.../Foo.java` and `app/src/main/.../Bar.java`, only one code area entity (`module:casehubio/devtown/app`) is created.

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

            List<String> moduleIds = normalizeToModules(pr.changedPaths()).stream()
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
                        .withQuestion("security findings and review issues in " + pr.repo()));

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
- `contributorHistory` — list of maps, each with `text`, `outcome`, `capability`, `createdAt` (extracted from `Memory` attributes)
- `codeAreaHistory` — list of maps, same structure

Injected as the `memory` key in the initial case context. Binding conditions can reference `memory.contributorHistory` and `memory.codeAreaHistory`. `hasRiskSignals()` returns true if any recalled fact has `outcome` != `COMPLETED` — used to flag elevated-risk PRs.

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

**Cross-entity scrubbing:** Code area facts include the contributor login in the natural-language `text` field (e.g., "...pull request by mdproctor..."). On contributor erasure, code area facts referencing the contributor must also be scrubbed. Implementation: query code area facts in the domain, filter by attribute `pr-contributor` (new devtown-specific key added for this purpose), erase matching code area facts. Reviewer facts reference the reviewer identity, not the contributor — no scrubbing needed.

**Not in scope for this issue:** Building a GDPR erasure endpoint. The `eraseEntity()` mechanism exists in the SPI; this spec documents how to use it. An admin endpoint or scheduled erasure job is a separate concern.

## Module Placement

| Module | New classes | Rationale |
|---|---|---|
| `devtown-domain` | `DevtownMemoryDomain`, `DevtownMemoryKeys` | Pure Java, zero framework deps |
| `devtown-review` | `PrPayload` enhanced, `MemoryContext`, `ReviewCompletedEvent` | Integration logic; `casehub-platform-api` on classpath for `Memory` type |
| `devtown-app` | `ReviewOutcomeObserver`, `CaseMemoryEmitter`, `CaseMemoryRecaller` | CDI wiring — observe engine events, inject `CaseMemoryStore` |

**Dependencies added:**
- `devtown-review` pom: `casehub-platform-api` — make explicit (likely already transitive)
- `devtown-app` pom: no new deps — `casehub-engine` (blackboard events) and `casehub-platform` (no-op bean) already present

## Testing

| Test | Type | Verifies |
|---|---|---|
| `DevtownMemoryDomainTest` | Unit (domain) | Domain constant, entity ID formatting, module normalization, attribute key conventions |
| `MemoryContextTest` | Unit (review) | `toContextMap()` serialisation, `hasRiskSignals()` logic, `EMPTY` constant, empty-list handling |
| `CaseMemoryEmitterTest` | Unit (app) | Mock `ReviewCompletedEvent` → verify `storeAll()` called with correct `MemoryInput` list; verify batch includes contributor + reviewer + deduplicated modules; verify `Instance.isResolvable()` guard; verify failure swallowed |
| `ReviewOutcomeObserverTest` | `@QuarkusTest` (app) | Fire `PlanItemCompletedEvent` → verify `ReviewCompletedEvent` fired with correct fields extracted from case context; verify infrastructure bindings skipped; verify CaseInstance lookup failure swallowed |
| `CaseMemoryRecallerTest` | `@QuarkusTest` (app) | Pre-populate store → `recall()` → verify `MemoryContext` populated with correct entity IDs, domain, time window; verify empty store → `MemoryContext.EMPTY`; verify query failure → `MemoryContext.EMPTY` |
| `CaseMemoryIntegrationTest` | `@QuarkusTest` (app) | Round-trip: start case → binding completes → observer fires → emitter stores → start second case for same contributor → recall returns prior facts in initial context |

Pre-seeding note: tests that start cases must pre-seed parallel check keys with non-null values (PP-20260521-134c38).

`CaseMemoryEmitterTest` is a plain unit test (no `@QuarkusTest`) — the emitter receives a typed event and produces a `List<MemoryInput>`. No engine, no case context, no reactive types. This is the primary benefit of the `ReviewCompletedEvent` separation.

## Follow-up Issues

- **Engine:** promote `PlanItemCompletedEvent` to `engine-common-spi` as a stable extension point
- **Engine:** add CDI event for PlanItem fault/rejection (needed for FAILED outcome emission)
- **Qhorus:** add CDI event for commitment DECLINED (needed for DECLINED outcome emission)
- **Enriched outputMappings:** if compressed outcomes prove insufficient for `ReviewCompletedEvent.outcomeDetail`, enrich the YAML outputMappings to preserve more agent output in case context (declarative change, no code)
- **Recall limits as preferences:** make contributor limit (10), code area limit (15), and time window (90 days) configurable via `PreferenceKey` definitions
- **GDPR erasure endpoint:** admin endpoint or scheduled job to handle Art.17 requests using the documented `eraseEntity()` mechanism
