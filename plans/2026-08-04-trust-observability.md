# Trust Observability Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #98 — Trust visibility UI — reviewer scores, routing history, incidents
**Issue group:** #98, #179

**Goal:** Wire existing trust-workbench, trust-score-panel, and
trust-feedback-display components into the devtown reviewers view,
backed by a routing history REST endpoint serving real ledger data.

**Architecture:** Thin adapter layer — `TrustObservabilityResource` in
`app/governance/` queries `WorkerDecisionEntry` records from the ledger
and reshapes them to match the TypeScript interfaces in casehub-packages.
The reviewers view expands to a split-pane layout with the existing
trust-workbench component in the detail pane.

**Tech Stack:** Java 21, Quarkus 3.32.2, JAX-RS, Lit web components,
casehub-packages (blocks-ui), casehub-ledger

## Global Constraints

- Java DTOs must produce JSON matching TypeScript interfaces exactly
  (see `trust-workbench/src/types.ts`, `routing-rationale/src/types.ts`,
  `trust-feedback-display/src/types.ts`, `trust-score-panel/src/types.ts`)
- Security: `@PermitAll` matching `GovernanceResource` pattern
- Module placement: DTOs and resource in `app/governance/` (same package
  as `GovernanceResource` — these are app-layer REST concerns)
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- All commits reference #98 or #179

---

### Task 1: Score Proxy Endpoint + ReviewerFleetEntry GlobalScore

Add globalScore to the reviewer fleet response and create a trust score
proxy endpoint that matches the `TrustScoreResponse` TypeScript interface.

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java`
- Create: `app/src/main/java/io/casehub/devtown/app/governance/TrustObservabilityResource.java`
- Test: `app/src/test/java/io/casehub/devtown/app/governance/TrustObservabilityResourceTest.java`

**Interfaces:**
- Consumes: `TrustGateService.currentScore(actorId)`, `TrustGateService.allCapabilityScores(actorId)`, `TrustGateService.allDimensionScores(actorId)`
- Produces: `GET /api/governance/trust/{actorId}` returning JSON matching `TrustScoreResponse` interface; `ReviewerFleetEntry` with added `globalScore` field

- [ ] **Step 1: Write failing test for score proxy endpoint**

```java
@QuarkusTest
class TrustObservabilityResourceTest {

    @Test
    void getActorTrust_knownActor_returnsScoreResponse() {
        // Seed trust score via ActorTrustScoreRepository.upsert()
        // in QuarkusTransaction.requiringNew()

        given()
            .when().get("/api/governance/trust/test-agent")
            .then()
            .statusCode(200)
            .body("actorId", equalTo("test-agent"))
            .body("capabilityScores", notNullValue());
    }

    @Test
    void getActorTrust_unknownActor_returns404() {
        given()
            .when().get("/api/governance/trust/nonexistent")
            .then()
            .statusCode(404);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TrustObservabilityResourceTest -Dquarkus.test.flat-class-path=true`
Expected: FAIL — no endpoint registered

- [ ] **Step 3: Create TrustObservabilityResource with score proxy**

```java
package io.casehub.devtown.app.governance;

import io.casehub.ledger.runtime.service.TrustGateService;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.Map;
import java.util.OptionalDouble;

@Path("/api/governance/trust")
@Produces(MediaType.APPLICATION_JSON)
@jakarta.annotation.security.PermitAll
public class TrustObservabilityResource {

    @Inject TrustGateService trustGateService;

    public record TrustScoreResponse(
        String actorId,
        Double globalScore,
        Map<String, Double> capabilityScores,
        Map<String, Double> dimensionScores) {}

    @GET
    @Path("/{actorId}")
    public Response getActorTrust(@PathParam("actorId") String actorId) {
        Map<String, Double> capScores = trustGateService.allCapabilityScores(actorId);
        Map<String, Double> dimScores = trustGateService.allDimensionScores(actorId);
        if (capScores.isEmpty() && dimScores.isEmpty()) {
            return Response.status(404)
                .entity(Map.of("error", "No trust data for: " + actorId))
                .build();
        }
        OptionalDouble global = trustGateService.currentScore(actorId);
        return Response.ok(new TrustScoreResponse(
            actorId,
            global.isPresent() ? global.getAsDouble() : null,
            capScores,
            dimScores
        )).build();
    }
}
```

- [ ] **Step 4: Add globalScore to ReviewerFleetEntry**

In `GovernanceQueryService.java`, change the record:

```java
public record ReviewerFleetEntry(String actorId, Map<String, Double> trustByCapability,
                                 Double globalScore,
                                 String maturityPhase, int openCommitments, int totalDecisions) {}
```

Update `reviewerFleet()` method to populate it:

```java
OptionalDouble globalScore = trustGateService.currentScore(actor.actorId());
return new ReviewerFleetEntry(actor.actorId(), capScores,
    globalScore.isPresent() ? globalScore.getAsDouble() : null,
    phase, openCommitments, totalDecisions);
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TrustObservabilityResourceTest,GovernanceResourceTest -Dquarkus.test.flat-class-path=true`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/governance/TrustObservabilityResource.java \
       app/src/test/java/io/casehub/devtown/app/governance/TrustObservabilityResourceTest.java \
       app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java
git commit -m "feat(#98): trust score proxy endpoint and globalScore in reviewer fleet"
```

---

### Task 2: Routing History Endpoint

Add routing history list and detail endpoints serving `WorkerDecisionEntry`
data from the ledger, shaped to match the trust-workbench TypeScript types.

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/TrustObservabilityResource.java`
- Test: `app/src/test/java/io/casehub/devtown/app/governance/TrustObservabilityResourceTest.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository` (or `CaseLedgerEntryRepository`) for `WorkerDecisionEntry` records; `DevtownTrustRoutingPolicyProvider.forCapability()` for policy summary
- Produces: `GET /api/governance/trust/{actorId}/routing-history` returning JSON matching `RoutingDecisionSummary[]`; `GET /api/governance/trust/{actorId}/routing-history/{entryId}` returning JSON matching `RoutingDecisionDetail`

- [ ] **Step 1: Write failing test for routing history list**

```java
@Test
void routingHistory_knownActor_returnsList() {
    // Seed a WorkerDecisionEntry via the ledger's event capture path
    // or direct repository insertion

    given()
        .when().get("/api/governance/trust/test-agent/routing-history")
        .then()
        .statusCode(200)
        .body("size()", greaterThanOrEqualTo(0));
}

@Test
void routingHistory_withCapabilityFilter_filtersResults() {
    given()
        .queryParam("capability", "code-analysis")
        .when().get("/api/governance/trust/test-agent/routing-history")
        .then()
        .statusCode(200);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TrustObservabilityResourceTest#routingHistory* -Dquarkus.test.flat-class-path=true`
Expected: FAIL — no endpoint

- [ ] **Step 3: Add DTO records matching TypeScript interfaces**

Add to `TrustObservabilityResource.java` (inner records):

```java
// Matches trust-workbench/src/types.ts RoutingDecisionSummary
public record RoutingDecisionSummary(
    String id,
    String timestamp,
    String capabilityTag,
    String selectedWorkerId,
    double finalScore,
    String phase) {}

// Matches trust-workbench/src/types.ts RoutingDecisionDetail
public record RoutingDecisionDetail(
    RoutingRationaleData rationale,
    List<GateDecision> feedback) {}

// Matches routing-rationale/src/types.ts
public record RoutingRationaleData(
    String capabilityTag,
    String strategyId,
    CandidateScore selected,
    List<CandidateScore> alternatives,
    RoutingPolicySummary policy) {}

public record CandidateScore(
    String workerId,
    Double trustScore,          // number | null in TS
    double workloadScore,
    String phase,               // enum string
    int observations,
    double finalScore,
    String exclusionReason,     // optional
    String rationale,           // optional
    Map<String, Double> additionalScores) {} // optional

public record RoutingPolicySummary(
    double threshold,
    double borderlineMargin,
    double blendFactor,
    int minimumObservations,
    Map<String, Double> qualityFloors,
    double cbrWeight,
    boolean bootstrapEscalationRequired) {}

// Matches trust-feedback-display/src/types.ts
public record GateDecision(
    String decision,
    String actor,
    String attestation,
    double trustScoreBefore,
    double trustScoreAfter,
    String dimension) {}
```

- [ ] **Step 4: Implement routing history list endpoint**

```java
@Inject CaseLedgerEntryRepository ledgerEntryRepository;
@Inject DevtownTrustRoutingPolicyProvider policyProvider;

@GET
@Path("/{actorId}/routing-history")
public List<RoutingDecisionSummary> routingHistory(
        @PathParam("actorId") String actorId,
        @QueryParam("capability") String capability,
        @QueryParam("limit") @DefaultValue("50") int limit,
        @QueryParam("offset") @DefaultValue("0") int offset) {

    // Query WorkerDecisionEntry records for this actor
    // Filter by type (WorkerDecisionEntry only), capability if provided
    // Map to RoutingDecisionSummary using fields from the entry:
    //   id = entry.id.toString()
    //   timestamp = entry.timestamp.toString()
    //   capabilityTag = entry.capabilityTag
    //   selectedWorkerId = actorId (this IS the selected worker)
    //   finalScore = entry.trustScoreAtRouting (or 0.0 if null)
    //   phase = derive from score vs policy threshold
    // Apply offset/limit pagination
}
```

The exact query method depends on what `CaseLedgerEntryRepository` or
`LedgerEntryRepository` exposes. During implementation, use
`ide_find_class` → `ide_file_structure` on the repository to discover
available query methods. If no `findByActorId` method exists that
filters by entry type, add a JPQL query.

- [ ] **Step 5: Implement routing history detail endpoint**

```java
@GET
@Path("/{actorId}/routing-history/{entryId}")
public Response routingDetail(
        @PathParam("actorId") String actorId,
        @PathParam("entryId") UUID entryId) {

    // Look up the WorkerDecisionEntry by ID
    // Build RoutingRationaleData:
    //   selected = CandidateScore from entry fields
    //   alternatives = empty list (ledger doesn't store candidates)
    //   policy = from policyProvider.forCapability(entry.capabilityTag)
    // Build feedback list from LedgerAttestation records on this entry:
    //   Map each attestation to GateDecision
    //   trustScoreBefore/After: use current score for both if historical
    //   snapshots unavailable (no delta shown — acceptable degradation)
    // Return RoutingDecisionDetail
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=TrustObservabilityResourceTest -Dquarkus.test.flat-class-path=true`
Expected: PASS

- [ ] **Step 7: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/governance/TrustObservabilityResource.java \
       app/src/test/java/io/casehub/devtown/app/governance/TrustObservabilityResourceTest.java
git commit -m "feat(#98): routing history endpoints with ledger data"
```

---

### Task 3: Reviewers View — Split-Pane with Trust Workbench

Expand the reviewers view from a flat table to a split-pane layout.
Left: reviewer table with inline trust badges. Right: trust-workbench
for the selected reviewer.

**Files:**
- Modify: `app/src/main/webui/src/views/reviewers.ts`
- Modify: `app/src/main/webui/src/index.ts` (if trust-workbench import needed)

**Interfaces:**
- Consumes: `GET /api/governance/reviewers` (existing, now with `globalScore`); `GET /api/governance/trust/{actorId}` (Task 1); `GET /api/governance/trust/{actorId}/routing-history` (Task 2)
- Produces: Reviewers view with split-pane layout and trust-workbench detail

- [ ] **Step 1: Understand the pages-ui DSL**

Before writing code, check how other views use split-pane layouts. Use
`ide_search_text` to find `split` or `splitWorkbench` usage in `.ts`
files under `src/views/`. Also check what `pages-ui` exports:

```
ide_search_text("split", filePattern="*.ts", project_path="...")
```

The trust-workbench internally uses `<blocks-split-workbench>`. For the
reviewers view, wrap the existing table + a detail pane using the
pages-ui `detail()` or `splitView()` DSL function if available. If no
DSL function exists, use raw Lit HTML with the `<blocks-split-workbench>`
element directly.

- [ ] **Step 2: Add trust-workbench import and expand layout**

Replace the current `reviewersView` in `reviewers.ts`. The current view:

```typescript
export const reviewersView = page("Reviewers",
  rows(
    title("Reviewer Fleet", "h2"),
    dataTable({
      lookup: lookup("reviewers", groupBy("actorId",
        col("actorId"),
        col("maturityPhase"),
        col("openCommitments"),
        col("totalDecisions")
      )),
      sortable: true,
      filter: { enabled: true },
    }),
  ),
);
```

The new view needs to:
1. Keep the table with an added `globalScore` column
2. Add row selection that sets the active `actorId`
3. Render `<blocks-trust-workbench>` in a detail pane
4. Pass `endpoint="/api/governance"` and `actor-id` to the workbench

The exact DSL depends on what `pages-ui` provides for split layouts.
Explore `@casehubio/pages-ui` exports during implementation. If the DSL
doesn't support split panes, create a thin wrapper Lit element in
`reviewers.ts` that renders both the table and the workbench.

- [ ] **Step 3: Add inline trust badge column**

Add `globalScore` to the `col()` list in the lookup. Use a custom column
renderer to display `<blocks-trust-score-panel mode="compact">` with
the pre-fetched `score` and `trust-level` attributes.

The `trust-score-panel` component accepts `score` and `trustLevel`
properties. When both are set, its `_hasPreFetchedData()` returns true
and it renders from attributes without fetching.

```typescript
col("globalScore"),
```

With a custom renderer that renders the compact trust badge.

- [ ] **Step 4: Test in browser**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app`
Navigate to the Reviewers view. Verify:
- Trust badges appear inline in the table
- Clicking a reviewer row shows trust-workbench in the detail pane
- Trust-workbench loads scores and routing history
- Capability filtering works (click capability → history filters)

- [ ] **Step 5: Commit**

```
git add app/src/main/webui/src/views/reviewers.ts
git commit -m "feat(#98): reviewers view with trust-workbench split pane"
```

---

### Task 4: Investigate Score Decay (#179)

Investigate whether `DecayFunction` exists in casehub-ledger. Configure
it or file an upstream issue.

**Files:**
- No devtown source changes expected — investigation task

**Interfaces:**
- Consumes: casehub-ledger source code (via IntelliJ MCP on workspace)
- Produces: Either a configuration change or a filed upstream issue

- [ ] **Step 1: Search casehub-ledger for decay mechanisms**

Use IntelliJ MCP to search the ledger codebase:

```
ide_search_text("DecayFunction", project_path="/path/to/ledger")
ide_search_text("decay", filePattern="*.java", project_path="/path/to/ledger")
ide_search_text("time-weight", filePattern="*.java", project_path="/path/to/ledger")
ide_find_class("Decay", project_path="/path/to/ledger")
```

Also check `TrustScoreCalculator` and `TrustScoreComputer` for any
time-based weighting in the Bayesian Beta computation.

- [ ] **Step 2: Document findings**

Record what was found:
- Does `DecayFunction` exist? What interface does it expose?
- Is it wired into the trust computation pipeline?
- Is it active by default or opt-in?
- Does the existing decay (if any) apply to contributor trust
  (`PR_CONTRIBUTION` capability)?

- [ ] **Step 3: Act on findings**

**If decay exists and is configurable:**
- Add a preference key via `PreferenceProvider` for contributor decay
- Test that the configuration takes effect
- Close #179

**If decay doesn't exist or is insufficient:**
- File a casehub-ledger issue proposing `DecayFunction` SPI
- Close #179 with "deferred — prerequisite filed as ledger#N"
- Update the devtown issue with the upstream reference

- [ ] **Step 4: Commit (if configuration changes were made)**

```
git commit -m "feat(#179): configure contributor trust score decay"
```

Or if only an upstream issue was filed:

```
# No commit needed — close #179 on GitHub with a note
gh issue close 179 --repo casehubio/devtown --comment "Deferred — filed casehubio/ledger#N for DecayFunction SPI"
```

---

## Dependency Graph

```
Task 1 (score proxy + globalScore)
  ↓
Task 2 (routing history endpoints)
  ↓
Task 3 (frontend wiring)

Task 4 (decay investigation) — independent, can run in parallel
```

Tasks 1-3 are sequential. Task 4 is independent.
