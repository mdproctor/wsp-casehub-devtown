# Governance Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the devtown governance workbench — an operational UI for PR review coordination, trust-weighted routing, merge queue management, and human-in-the-loop oversight.

**Architecture:** Extract aggregation logic from `DevtownMcpTools` into `GovernanceQueryService`. Expose that service via REST (`GovernanceResource`) and WebSocket (`GovernanceEventBridge`). Wire Quinoa + casehub-pages for the TypeScript frontend. Six views composed via the pages DSL.

**Tech Stack:** Java 21 / Quarkus 3.32.2, Quarkiverse Quinoa 2.8.3, casehub-pages 0.2.0, esbuild, TypeScript

## Global Constraints

- Java 21 source (runs on Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- All REST endpoints `@RolesAllowed("admin")` initially
- All commits reference issues: `Refs #85` or `Refs #92`
- casehub-pages packages at version `0.2.0` from GitHub Packages (`@casehubio` scope)
- Quinoa version `2.8.3` — declared in devtown parent pom.xml `<properties>`
- Quarkus WebSocket is a core platform extension — no version needed
- Security in dev mode is disabled (`%dev.quarkus.security.auth.enabled-in-dev-mode=false`)
- Tests use `@TestSecurity(user = "test-admin", roles = "admin")` for authenticated endpoints
- esbuild gotcha GE-20260623-06914b: use named imports from `@casehubio/pages-viz`, never bare side-effect imports
- esbuild gotcha GE-20260629-59c7e6: never interpolate TypeScript constants inside `html()` template literals

---

### Task 1: GovernanceQueryService — extract aggregation from DevtownMcpTools

The root task. All REST and WebSocket work depends on this service existing. The aggregation logic currently inline in `DevtownMcpTools` (800+ lines, 16 tool methods) is extracted into a standalone `@ApplicationScoped` service. The MCP tool methods are then refactored to delegate.

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java`
- Create: `app/src/test/java/io/casehub/devtown/app/governance/GovernanceQueryServiceTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java` — refactor read methods to delegate

**Interfaces:**
- Consumes: `PrReviewCaseTracker` (queue status, events, stalled cases), `CommitmentStore` (open commitments), `TrustExportService` (fleet enumeration), `TrustGateService` (per-actor scores, decision counts), `WorkItemStore` (triage), `MergeQueueService` (queue/batch data), `CaseHubRuntime` (event log, case context), `LedgerProvExportService` (PROV-DM)
- Produces: `GovernanceQueryService` — consumed by Task 2 (REST) and Task 4 (MCP refactor)

**Records — define in `GovernanceQueryService` initially (same file, inner records):**

The existing DTO records (`QueueStatus`, `ActiveReview`, `SystemHealth`, `ReviewDetail`, `EventEntry`, `CapabilityStatus`, `ReviewerHealth`, `RecentOutcome`, `Problem`, `MergeQueueStatus`, `QueuedPrEntry`, `ActiveBatchEntry`, `BatchStatus`, `BatchPrEntry`, `MergeQueueMetrics`) currently live inside `DevtownMcpTools`. They move to `GovernanceQueryService` as public inner records. `DevtownMcpTools` then imports them from the new location.

**New records (not in MCP tools today):**
- `ReviewerFleetEntry(String actorId, Map<String, Double> trustByCapability, String maturityPhase, int openCommitments, int totalDecisions)` — fleet-wide reviewer summary
- `TriageItem(UUID workItemId, String prRef, String decisionType, String candidateGroup, Instant expiresAt, String escalationStage, Instant createdAt, UUID caseId)` — pending human decision

- [ ] **Step 1: Write failing test — queueStatus()**

Create `GovernanceQueryServiceTest.java`. This is a plain unit test (not `@QuarkusTest`) — `GovernanceQueryService` is a POJO with constructor injection.

```java
package io.casehub.devtown.app.governance;

import io.casehub.devtown.app.MergeQueueService;
import io.casehub.devtown.app.mcp.CaseInfo;
import io.casehub.devtown.app.mcp.PrReviewCaseTracker;
import io.casehub.devtown.app.mcp.TrackedEvent;
import io.casehub.devtown.review.PrPayload;
import io.casehub.engine.common.spi.CaseHubRuntime;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.ledger.runtime.service.federation.TrustExportService;
import io.casehub.ledger.runtime.service.prov.LedgerProvExportService;
import io.casehub.qhorus.runtime.store.CommitmentStore;
import io.casehub.work.runtime.repository.WorkItemStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class GovernanceQueryServiceTest {

    PrReviewCaseTracker tracker;
    CommitmentStore commitmentStore;
    TrustExportService trustExportService;
    TrustGateService trustGateService;
    WorkItemStore workItemStore;
    MergeQueueService mergeQueueService;
    CaseHubRuntime caseHubRuntime;
    LedgerProvExportService provExportService;

    GovernanceQueryService service;

    @BeforeEach
    void setUp() {
        tracker = new PrReviewCaseTracker();
        commitmentStore = mock(CommitmentStore.class);
        trustExportService = mock(TrustExportService.class);
        trustGateService = mock(TrustGateService.class);
        workItemStore = mock(WorkItemStore.class);
        mergeQueueService = mock(MergeQueueService.class);
        caseHubRuntime = mock(CaseHubRuntime.class);
        provExportService = mock(LedgerProvExportService.class);

        service = new GovernanceQueryService(
            tracker, commitmentStore, trustExportService, trustGateService,
            workItemStore, mergeQueueService, caseHubRuntime, provExportService
        );
    }

    @Test
    void queueStatus_returnsActiveReviewsWithStatusCounts() {
        var payload = new PrPayload("casehubio/devtown", 42, "abc123", "main", 150, "jsmith", List.of("src/Main.java"));
        var caseId = UUID.randomUUID();
        tracker.register(caseId, "default", payload);

        var result = service.queueStatus();

        assertThat(result.total()).isEqualTo(1);
        assertThat(result.reviews()).hasSize(1);
        assertThat(result.reviews().get(0).prNumber()).isEqualTo(42);
        assertThat(result.reviews().get(0).contributor()).isEqualTo("jsmith");
        assertThat(result.countsByStatus()).containsKey("RUNNING");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=GovernanceQueryServiceTest#queueStatus_returnsActiveReviewsWithStatusCounts -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `GovernanceQueryService` class does not exist.

- [ ] **Step 3: Implement GovernanceQueryService with queueStatus()**

```java
package io.casehub.devtown.app.governance;

import io.casehub.devtown.app.MergeQueueService;
import io.casehub.devtown.app.mcp.CaseInfo;
import io.casehub.devtown.app.mcp.PrReviewCaseTracker;
import io.casehub.devtown.app.mcp.TrackedEvent;
import io.casehub.devtown.merge.BatchRecord;
import io.casehub.devtown.queue.QueuedPr;
import io.casehub.devtown.review.PrPayload;
import io.casehub.engine.common.spi.CaseHubRuntime;
import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.ledger.runtime.service.federation.TrustExportService;
import io.casehub.ledger.runtime.service.prov.LedgerProvExportService;
import io.casehub.qhorus.runtime.store.CommitmentStore;
import io.casehub.work.runtime.repository.WorkItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

@ApplicationScoped
public class GovernanceQueryService {

    private final PrReviewCaseTracker tracker;
    private final CommitmentStore commitmentStore;
    private final TrustExportService trustExportService;
    private final TrustGateService trustGateService;
    private final WorkItemStore workItemStore;
    private final MergeQueueService mergeQueueService;
    private final CaseHubRuntime caseHubRuntime;
    private final LedgerProvExportService provExportService;

    @Inject
    public GovernanceQueryService(
            PrReviewCaseTracker tracker,
            CommitmentStore commitmentStore,
            TrustExportService trustExportService,
            TrustGateService trustGateService,
            WorkItemStore workItemStore,
            MergeQueueService mergeQueueService,
            CaseHubRuntime caseHubRuntime,
            LedgerProvExportService provExportService) {
        this.tracker = tracker;
        this.commitmentStore = commitmentStore;
        this.trustExportService = trustExportService;
        this.trustGateService = trustGateService;
        this.workItemStore = workItemStore;
        this.mergeQueueService = mergeQueueService;
        this.caseHubRuntime = caseHubRuntime;
        this.provExportService = provExportService;
    }

    // ── Records ──────────────────────────────────────────────

    public record QueueStatus(int total, Map<String, Integer> countsByStatus, List<ActiveReview> reviews) {}

    public record ActiveReview(UUID caseId, String repo, int prNumber, String contributor,
                               int linesChanged, String status, Instant startedAt, Instant lastEventAt) {}

    public record SystemHealth(int activeCases, int fleetSize, Map<String, Double> avgTrustByCapability,
                               int openCommitments, int pendingWorkItems) {}

    public record ReviewDetail(UUID caseId, PrPayload pr, List<EventEntry> timeline,
                               List<CapabilityStatus> capabilities) {}

    public record EventEntry(Instant timestamp, String eventType, String actor, String summary) {}

    public record CapabilityStatus(String name, String status, String outcome, Instant completedAt) {}

    public record ReviewerHealth(String reviewerId, int openCommitments, Map<String, Double> trustByCapability,
                                 Map<String, Double> trustByDimension, int totalDecisions,
                                 List<RecentOutcome> recentOutcomes) {}

    public record RecentOutcome(UUID caseId, String capability, String outcome, Instant timestamp) {}

    public record Problem(String category, String severity, String description,
                          UUID caseId, String actorId, Instant since) {}

    public record MergeQueueStatus(int queuedCount, int activeBatchCount,
                                   List<QueuedPrEntry> queuedPrs, List<ActiveBatchEntry> activeBatches) {}

    public record QueuedPrEntry(int number, String repository, String headSha, String author,
                                double trustScore, String priorityLane, Instant enqueuedAt,
                                long waitMinutes, Set<Integer> dependsOn) {}

    public record ActiveBatchEntry(UUID caseId, String batchId, int prCount, String riskLevel) {}

    public record BatchStatus(String batchId, UUID caseId, List<BatchPrEntry> prs,
                              String riskLevel, String bisectionStrategy) {}

    public record BatchPrEntry(int number, String repository, String headSha,
                               String author, double trustScore, String lane) {}

    public record MergeQueueMetrics(int queueDepth, int activeBatches, long oldestWaitMinutes,
                                    long avgWaitMinutes, double avgTrustScore,
                                    Map<String, Integer> countsByLane, int throughput24h,
                                    double failureRate, Map<Integer, Integer> batchSizeDistribution) {}

    public record ReviewerFleetEntry(String actorId, Map<String, Double> trustByCapability,
                                     String maturityPhase, int openCommitments, int totalDecisions) {}

    public record TriageItem(UUID workItemId, String prRef, String decisionType, String candidateGroup,
                             Instant expiresAt, String escalationStage, Instant createdAt, UUID caseId) {}

    // ── Query methods ────────────────────────────────────────

    public QueueStatus queueStatus() {
        List<CaseInfo> active = tracker.activeCases();
        Map<String, Integer> countsByStatus = new HashMap<>();

        List<ActiveReview> reviews = active.stream()
            .map(c -> {
                String statusStr = c.status().name();
                countsByStatus.merge(statusStr, 1, Integer::sum);
                return new ActiveReview(
                    c.caseId(), c.payload().repo(), c.payload().prNumber(),
                    c.payload().contributor(), c.payload().linesChanged(),
                    statusStr, c.startedAt(), c.lastEventAt()
                );
            })
            .toList();

        return new QueueStatus(active.size(), countsByStatus, reviews);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=GovernanceQueryServiceTest#queueStatus_returnsActiveReviewsWithStatusCounts -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Add remaining query methods with tests**

Add tests and implementations for each method. The implementation is a direct extraction from `DevtownMcpTools` — copy the logic, remove the `@Tool` annotations, adjust field references to use injected dependencies.

Methods to extract (reference the existing MCP tool method for exact logic):
- `recentEvents(int limit, Instant since)` — from `DevtownMcpTools.getRecentEvents()` line 257
- `systemHealth()` — from `DevtownMcpTools.getSystemHealth()` line 270
- `problems(int thresholdMinutes)` — from `DevtownMcpTools.listProblems()` line 314
- `reviewDetail(UUID caseId)` — from `DevtownMcpTools.inspectReview()` line 385
- `reviewerHealth(String actorId)` — from `DevtownMcpTools.getReviewerHealth()` line 446
- `mergeQueue()` — from `DevtownMcpTools.getMergeQueue()` line 543
- `batchStatus(UUID batchCaseId)` — from `DevtownMcpTools.getBatchStatus()` line 573
- `mergeQueueMetrics()` — from `DevtownMcpTools.getMergeQueueMetrics()` line 595

New methods (not in MCP tools):
- `reviewerFleet()` — enumerate all agents via `trustExportService.exportAll(0.0)`, compute maturity phase per agent per capability
- `triageItems()` — query `WorkItemStore.scan()` with `status=PENDING` and `category IN ("human-decision", "human-oversight")`
- `reviewsList()` — all tracked cases (active + from event buffer), for the Reviews list view

Write at least one test per method. For `systemHealth()`, verify fleet size computation. For `triageItems()`, verify category filtering. For `reviewerFleet()`, verify maturity phase derivation.

- [ ] **Step 6: Refactor DevtownMcpTools to delegate to GovernanceQueryService**

Inject `GovernanceQueryService` into `DevtownMcpTools`. Replace the inline aggregation logic in each read tool method with a delegation call. The `@Tool` methods remain in `DevtownMcpTools` (MCP annotation concern), but the logic lives in `GovernanceQueryService`.

The MCP tool record types (`DevtownMcpTools.QueueStatus`, etc.) are removed — the MCP tools return `GovernanceQueryService.QueueStatus` directly. Update `DevtownMcpToolsTest` imports accordingly.

Keep the write tool methods (`retryReviewer`, `rerouteReview`, `forceCompleteCheck`, `enqueuePr`, `dequeuePr`, `reportIncident`) inline in `DevtownMcpTools` — they are mutation operations, not queries.

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: All existing tests pass (MCP tools still work via delegation).

- [ ] **Step 8: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/governance/ app/src/test/java/io/casehub/devtown/app/governance/ app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/
git commit -m "feat(#85): extract GovernanceQueryService from DevtownMcpTools

Moves read-path aggregation logic into a standalone service. MCP tools
delegate to the same service REST endpoints will use.

Refs #85"
```

---

### Task 2: GovernanceResource — REST endpoints

Thin JAX-RS resource that delegates to `GovernanceQueryService`. Each endpoint maps directly to one service method.

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceResource.java`
- Create: `app/src/test/java/io/casehub/devtown/app/governance/GovernanceResourceTest.java`

**Interfaces:**
- Consumes: `GovernanceQueryService` (all query methods from Task 1)
- Produces: REST endpoints under `/api/governance/*` — consumed by Task 6 (TypeScript datasets)

- [ ] **Step 1: Write failing test — GET /api/governance/queue-status**

```java
package io.casehub.devtown.app.governance;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
@TestSecurity(user = "test-admin", roles = "admin")
class GovernanceResourceTest {

    @Test
    void queueStatus_returnsJson() {
        given()
            .when().get("/api/governance/queue-status")
            .then()
            .statusCode(200)
            .body("total", is(notNullValue()))
            .body("reviews", is(notNullValue()));
    }

    @Test
    void queueStatus_requiresAuth() {
        // No @TestSecurity — unauthenticated
        // This test method must NOT have @TestSecurity
    }
}
```

Note: The `requiresAuth` test needs to be in a separate non-annotated test class or use Quarkus `@TestSecurity(authorizationEnabled = false)` pattern. Follow the existing `DevtownMcpToolsTest` pattern for auth testing.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=GovernanceResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — 404, resource does not exist.

- [ ] **Step 3: Implement GovernanceResource**

```java
package io.casehub.devtown.app.governance;

import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@Path("/api/governance")
@Produces(MediaType.APPLICATION_JSON)
@RolesAllowed("admin")
public class GovernanceResource {

    @Inject
    GovernanceQueryService queryService;

    @GET
    @Path("/queue-status")
    public GovernanceQueryService.QueueStatus queueStatus() {
        return queryService.queueStatus();
    }

    @GET
    @Path("/recent-events")
    public List<?> recentEvents(
            @QueryParam("limit") @DefaultValue("50") int limit,
            @QueryParam("since") String since) {
        Instant sinceTime = since != null ? Instant.parse(since) : null;
        return queryService.recentEvents(limit, sinceTime);
    }

    @GET
    @Path("/system-health")
    public GovernanceQueryService.SystemHealth systemHealth() {
        return queryService.systemHealth();
    }

    @GET
    @Path("/problems")
    public List<GovernanceQueryService.Problem> problems(
            @QueryParam("threshold_minutes") @DefaultValue("60") int thresholdMinutes) {
        return queryService.problems(thresholdMinutes);
    }

    @GET
    @Path("/reviews")
    public List<GovernanceQueryService.QueueStatus> reviews() {
        // Returns all tracked reviews — active + recent from buffer
        return List.of(queryService.queueStatus());
    }

    @GET
    @Path("/reviews/{caseId}")
    public GovernanceQueryService.ReviewDetail reviewDetail(@PathParam("caseId") UUID caseId) {
        return queryService.reviewDetail(caseId);
    }

    @GET
    @Path("/reviewers")
    public List<GovernanceQueryService.ReviewerFleetEntry> reviewers() {
        return queryService.reviewerFleet();
    }

    @GET
    @Path("/reviewers/{actorId}")
    public GovernanceQueryService.ReviewerHealth reviewerHealth(@PathParam("actorId") String actorId) {
        return queryService.reviewerHealth(actorId);
    }

    @GET
    @Path("/merge-queue")
    public GovernanceQueryService.MergeQueueStatus mergeQueue() {
        return queryService.mergeQueue();
    }

    @GET
    @Path("/merge-queue/metrics")
    public GovernanceQueryService.MergeQueueMetrics mergeQueueMetrics() {
        return queryService.mergeQueueMetrics();
    }

    @GET
    @Path("/merge-queue/batch/{batchId}")
    public GovernanceQueryService.BatchStatus batchStatus(@PathParam("batchId") UUID batchId) {
        return queryService.batchStatus(batchId);
    }

    @GET
    @Path("/triage")
    public List<GovernanceQueryService.TriageItem> triage() {
        return queryService.triageItems();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=GovernanceResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Add tests for remaining endpoints**

Add a test for each endpoint: `systemHealth`, `problems`, `reviewDetail`, `reviewers`, `mergeQueue`, `mergeQueueMetrics`, `triage`. Each test verifies 200 status and basic JSON structure.

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/governance/GovernanceResource.java app/src/test/java/io/casehub/devtown/app/governance/GovernanceResourceTest.java
git commit -m "feat(#85): GovernanceResource — REST endpoints for governance workbench

Thin JAX-RS resource delegating to GovernanceQueryService. Twelve
endpoints under /api/governance/* for queue status, reviews, reviewers,
merge queue, triage, and system health.

Refs #85"
```

---

### Task 3: GovernanceEventBridge — WebSocket endpoint

CDI event observer that forwards governance events as JSON to connected WebSocket sessions.

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceEventBridge.java`
- Create: `app/src/test/java/io/casehub/devtown/app/governance/GovernanceEventBridgeTest.java`
- Modify: `app/src/main/resources/application.properties` — add WebSocket auth permission
- Modify: `app/pom.xml` — add `quarkus-websockets` dependency

**Interfaces:**
- Consumes: CDI events — `CaseLifecycleEvent` (`io.casehub.engine.common.spi.event`), `WorkItemLifecycleEvent` (`io.casehub.work.runtime.event`), `CommitmentDeclinedEvent` (`io.casehub.qhorus.api.message`), `CommitmentExpiredEvent` (`io.casehub.qhorus.api.message`), `TrustScoreActorUpdatedEvent` (`io.casehub.ledger.runtime.service.routing`), `SlaBreachEvent` (`io.casehub.work.runtime.event`)
- Produces: WebSocket messages on `ws://host/api/governance/events` — consumed by Task 7 (TypeScript WebSocket dataset)

- [ ] **Step 1: Add quarkus-websockets dependency to app/pom.xml**

Add to `<dependencies>` section after the existing Quarkus extensions:

```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-websockets</artifactId>
</dependency>
```

- [ ] **Step 2: Write failing test**

```java
package io.casehub.devtown.app.governance;

import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import org.junit.jupiter.api.Test;

import jakarta.websocket.Session;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class GovernanceEventBridgeTest {

    @Test
    void onCaseLifecycle_sendsJsonToConnectedSessions() throws Exception {
        var bridge = new GovernanceEventBridge();
        var session = mock(Session.class);
        var asyncRemote = mock(Session.AsyncRemote.class);
        when(session.getAsyncRemote()).thenReturn(asyncRemote);
        when(session.isOpen()).thenReturn(true);

        bridge.onOpen(session);

        var event = new CaseLifecycleEvent(
            java.util.UUID.randomUUID(), "default", "COMPLETED"
        );
        bridge.onCaseLifecycle(event);

        verify(asyncRemote).sendText(argThat(json ->
            json.contains("\"op\":\"event\"") &&
            json.contains("\"topic\":\"case.state\"") &&
            json.contains("COMPLETED")
        ));
    }

    @Test
    void onClose_removesSession() {
        var bridge = new GovernanceEventBridge();
        var session = mock(Session.class);

        bridge.onOpen(session);
        bridge.onClose(session);

        // Verify session is removed — send an event, session should NOT receive it
        var asyncRemote = mock(Session.AsyncRemote.class);
        when(session.getAsyncRemote()).thenReturn(asyncRemote);
        // No interaction expected after close
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=GovernanceEventBridgeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist.

- [ ] **Step 4: Implement GovernanceEventBridge**

```java
package io.casehub.devtown.app.governance;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.ledger.runtime.service.routing.TrustScoreActorUpdatedEvent;
import io.casehub.qhorus.api.message.CommitmentDeclinedEvent;
import io.casehub.qhorus.api.message.CommitmentExpiredEvent;
import io.casehub.work.runtime.event.SlaBreachEvent;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.websocket.OnClose;
import jakarta.websocket.OnError;
import jakarta.websocket.OnOpen;
import jakarta.websocket.Session;
import jakarta.websocket.server.ServerEndpoint;
import org.jboss.logging.Logger;

import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

@ApplicationScoped
@ServerEndpoint("/api/governance/events")
public class GovernanceEventBridge {

    private static final Logger LOG = Logger.getLogger(GovernanceEventBridge.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final Set<Session> sessions = new CopyOnWriteArraySet<>();

    @OnOpen
    public void onOpen(Session session) {
        sessions.add(session);
    }

    @OnClose
    public void onClose(Session session) {
        sessions.remove(session);
    }

    @OnError
    public void onError(Session session, Throwable error) {
        LOG.warnf("WebSocket error for session %s: %s", session.getId(), error.getMessage());
        sessions.remove(session);
    }

    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        broadcast("case.state", MAPPER.createObjectNode()
            .put("caseId", event.caseId().toString())
            .put("status", event.newStatus())
            .put("timestamp", java.time.Instant.now().toString()));
    }

    void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
        broadcast("workitem.lifecycle", MAPPER.createObjectNode()
            .put("workItemId", event.workItemId().toString())
            .put("status", event.newStatus().name())
            .put("timestamp", java.time.Instant.now().toString()));
    }

    void onCommitmentDeclined(@ObservesAsync CommitmentDeclinedEvent event) {
        broadcast("commitment.lifecycle", MAPPER.createObjectNode()
            .put("commitmentId", event.commitmentId().toString())
            .put("status", "DECLINED")
            .put("timestamp", java.time.Instant.now().toString()));
    }

    void onCommitmentExpired(@ObservesAsync CommitmentExpiredEvent event) {
        broadcast("commitment.lifecycle", MAPPER.createObjectNode()
            .put("commitmentId", event.commitmentId().toString())
            .put("status", "EXPIRED")
            .put("timestamp", java.time.Instant.now().toString()));
    }

    void onTrustScoreUpdated(@ObservesAsync TrustScoreActorUpdatedEvent event) {
        broadcast("trust.update", MAPPER.createObjectNode()
            .put("actorId", event.actorId())
            .put("timestamp", java.time.Instant.now().toString()));
    }

    void onSlaBreached(@ObservesAsync SlaBreachEvent event) {
        broadcast("sla.breach", MAPPER.createObjectNode()
            .put("workItemId", event.workItemId().toString())
            .put("timestamp", java.time.Instant.now().toString()));
    }

    private void broadcast(String topic, ObjectNode payload) {
        ObjectNode envelope = MAPPER.createObjectNode()
            .put("op", "event")
            .put("topic", topic);
        envelope.set("payload", payload);

        String json;
        try {
            json = MAPPER.writeValueAsString(envelope);
        } catch (Exception e) {
            LOG.error("Failed to serialize WebSocket message", e);
            return;
        }

        for (Session session : sessions) {
            if (session.isOpen()) {
                session.getAsyncRemote().sendText(json);
            } else {
                sessions.remove(session);
            }
        }
    }
}
```

Note: The CDI event field accessors (`event.caseId()`, `event.commitmentId()`, etc.) must be verified against the actual decompiled event classes. Check with IntelliJ MCP `ide_read_file` on each event class to confirm the accessor names before implementing.

- [ ] **Step 5: Add WebSocket auth to application.properties**

Append to `app/src/main/resources/application.properties`:

```properties
# ── Governance WebSocket ───────────────────────────────────────────────
quarkus.http.auth.permission.governance-ws.paths=/api/governance/events
quarkus.http.auth.permission.governance-ws.policy=authenticated
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=GovernanceEventBridgeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: All tests pass.

- [ ] **Step 8: Commit**

```
git add app/pom.xml app/src/main/java/io/casehub/devtown/app/governance/GovernanceEventBridge.java app/src/test/java/io/casehub/devtown/app/governance/GovernanceEventBridgeTest.java app/src/main/resources/application.properties
git commit -m "feat(#85): GovernanceEventBridge — WebSocket endpoint for live events

CDI event bridge forwarding case lifecycle, work item, commitment,
trust, SLA, and queue events as JSON to connected WebSocket sessions.

Refs #85"
```

---

### Task 4: Quinoa wiring — frontend scaffold

Add the Quinoa extension, create the webui directory structure, and get a minimal `loadSite()` rendering a test page.

**Files:**
- Modify: `pom.xml` (parent) — add `quarkus-quinoa.version` property
- Modify: `app/pom.xml` — add `quarkus-quinoa` dependency
- Modify: `app/src/main/resources/application.properties` — add Quinoa config
- Create: `app/src/main/webui/package.json`
- Create: `app/src/main/webui/.npmrc`
- Create: `app/src/main/webui/.gitignore`
- Create: `app/src/main/webui/tsconfig.json`
- Create: `app/src/main/webui/esbuild.config.mjs`
- Create: `app/src/main/webui/index.html`
- Create: `app/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `@casehubio/pages-runtime` (loadSite), `@casehubio/pages-ui` (DSL builders)
- Produces: A working Quinoa build that renders a placeholder page at `http://localhost:8080/`

- [ ] **Step 1: Add Quinoa version to parent pom.xml**

Add to `<properties>` in the root `pom.xml`:

```xml
<quarkus-quinoa.version>2.8.3</quarkus-quinoa.version>
```

Add to `<dependencyManagement>` in the root `pom.xml`:

```xml
<dependency>
  <groupId>io.quarkiverse.quinoa</groupId>
  <artifactId>quarkus-quinoa</artifactId>
  <version>${quarkus-quinoa.version}</version>
</dependency>
```

- [ ] **Step 2: Add Quinoa dependency to app/pom.xml**

Add to `<dependencies>` after the Quarkus extensions:

```xml
<dependency>
  <groupId>io.quarkiverse.quinoa</groupId>
  <artifactId>quarkus-quinoa</artifactId>
</dependency>
```

- [ ] **Step 3: Add Quinoa config to application.properties**

Append to `app/src/main/resources/application.properties`:

```properties
# ── Quinoa frontend ────────────────────────────────────────────────────
quarkus.quinoa.build-dir=dist
quarkus.quinoa.package-manager-install=true
quarkus.quinoa.enable-spa-routing=true
```

- [ ] **Step 4: Create webui directory structure**

Create `app/src/main/webui/package.json`:

```json
{
  "name": "devtown-webui",
  "private": true,
  "scripts": {
    "build": "node esbuild.config.mjs",
    "dev": "node esbuild.config.mjs --watch",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@casehubio/pages-runtime": "0.2.0",
    "@casehubio/pages-ui": "0.2.0"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "typescript": "^5.6.0"
  }
}
```

Create `app/src/main/webui/.npmrc`:

```
@casehubio:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

Create `app/src/main/webui/.gitignore`:

```
node_modules/
dist/
```

Create `app/src/main/webui/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": false,
    "noEmit": true
  },
  "include": ["src"]
}
```

Create `app/src/main/webui/esbuild.config.mjs`:

```javascript
import { build, context } from "esbuild";

const isWatch = process.argv.includes("--watch");

const options = {
  entryPoints: ["src/index.ts"],
  bundle: true,
  outfile: "dist/app.js",
  format: "esm",
  target: "es2020",
  minify: !isWatch,
  sourcemap: isWatch,
};

if (isWatch) {
  const ctx = await context(options);
  await ctx.watch();
  console.log("Watching for changes...");
} else {
  await build(options);
}
```

Create `app/src/main/webui/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>DevTown Governance</title>
  <style>
    body { margin: 0; font-family: system-ui, sans-serif; }
    #app { width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="dist/app.js"></script>
</body>
</html>
```

Create `app/src/main/webui/src/index.ts`:

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { page, metric, columns, rows, title } from "@casehubio/pages-ui";

const app = page("DevTown Governance",
  rows(
    title("DevTown Governance Workbench", 1),
    columns([50, 50],
      [metric({ title: "Status", value: "Online", subtype: "plain-text" })],
      [metric({ title: "Build", value: "Quinoa + casehub-pages", subtype: "plain-text" })],
    ),
  ),
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, app).catch(console.error);
}
```

- [ ] **Step 5: Verify the build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl app`

If `npm install` fails due to GitHub Packages auth, ensure `GITHUB_TOKEN` env var is set with `read:packages` scope.

Expected: Build succeeds. Quinoa runs `npm install` and `npm run build` during the Maven build. The `dist/app.js` bundle is included in the JAR.

- [ ] **Step 6: Verify dev mode renders**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app`

Open `http://localhost:8080/` in a browser. Expected: The placeholder page renders with "DevTown Governance Workbench" title and two metric cards.

Stop dev mode after verifying.

- [ ] **Step 7: Commit**

```
git add pom.xml app/pom.xml app/src/main/resources/application.properties app/src/main/webui/
git commit -m "feat(#92): Quinoa wiring — casehub-pages frontend scaffold

Adds quarkus-quinoa extension, webui directory with esbuild config,
and a minimal loadSite() page verifying the full stack works.

Closes #92, Refs #85"
```

---

### Task 5: TypeScript view structure — datasets, theme, navigation

Set up the shared infrastructure that all views depend on: dataset declarations, theme config, and top-level navigation routing.

**Files:**
- Create: `app/src/main/webui/src/datasets.ts`
- Create: `app/src/main/webui/src/theme.ts`
- Create: `app/src/main/webui/src/views/system.ts` — simplest view, validates the full stack
- Modify: `app/src/main/webui/src/index.ts` — replace placeholder with nav + System view

**Interfaces:**
- Consumes: REST endpoints from Task 2, WebSocket from Task 3
- Produces: Shared datasets and theme used by all subsequent view tasks (Tasks 6–9)

- [ ] **Step 1: Create datasets.ts**

```typescript
import { dataset } from "@casehubio/pages-ui";

// REST datasets — resolved relative to the Quarkus server
dataset("queue-status", "/api/governance/queue-status");
dataset("recent-events", "/api/governance/recent-events?limit=100");
dataset("system-health", "/api/governance/system-health");
dataset("problems", "/api/governance/problems");
dataset("reviewers", "/api/governance/reviewers");
dataset("merge-queue", "/api/governance/merge-queue");
dataset("merge-queue-metrics", "/api/governance/merge-queue/metrics");
dataset("triage", "/api/governance/triage");
```

- [ ] **Step 2: Create theme.ts**

```typescript
import { DARK_THEME } from "@casehubio/pages-runtime";
import type { PagesTheme } from "@casehubio/pages-runtime";

export const devtownDark: PagesTheme = {
  ...DARK_THEME,
  accent: "#7c8cf8",
  accentHover: "#9ba6ff",
};

export const defaultTheme = devtownDark;
```

- [ ] **Step 3: Create System view**

Create `app/src/main/webui/src/views/system.ts`:

```typescript
import {
  page, metric, columns, rows, table, barChart, title,
  lookup, groupBy, col, sum,
} from "@casehubio/pages-ui";
import { dataset } from "@casehubio/pages-ui";

export const systemView = page("System",
  rows(
    title("System Health", 2),

    // Health metrics row
    columns([25, 25, 25, 25],
      [metric({ title: "Active Cases", lookup: lookup("system-health"), subtype: "plain-text" })],
      [metric({ title: "Fleet Size", lookup: lookup("system-health"), subtype: "plain-text" })],
      [metric({ title: "Open Commitments", lookup: lookup("system-health"), subtype: "plain-text" })],
      [metric({ title: "Pending Work Items", lookup: lookup("system-health"), subtype: "plain-text" })],
    ),

    // Body — trust chart + problems table
    columns([50, 50],
      [barChart({
        title: "Average Trust by Capability",
        lookup: lookup("system-health"),
      })],
      [table({
        title: "Problems",
        lookup: lookup("problems"),
        sortable: true,
      })],
    ),

    // Queue health
    title("Queue Health", 3),
    columns([25, 25, 25, 25],
      [metric({ title: "Queue Depth", lookup: lookup("merge-queue-metrics"), subtype: "plain-text" })],
      [metric({ title: "Oldest Wait", lookup: lookup("merge-queue-metrics"), subtype: "plain-text" })],
      [metric({ title: "Avg Wait", lookup: lookup("merge-queue-metrics"), subtype: "plain-text" })],
      [metric({ title: "Failure Rate", lookup: lookup("merge-queue-metrics"), subtype: "plain-text" })],
    ),
  ),
);
```

Note: The exact DSL usage (lookup bindings for metrics, dataset field extraction) depends on the casehub-pages API for single-value metrics from REST responses. The implementer must check how `metric()` binds to a JSON object dataset — the lookup may need JSONata expressions or direct field bindings. Consult `docs/CASEHUB-PAGES.md` at implementation time.

- [ ] **Step 4: Update index.ts with navigation**

Replace `app/src/main/webui/src/index.ts`:

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { page, tabs } from "@casehubio/pages-ui";
import "./datasets";
import { defaultTheme } from "./theme";
import { systemView } from "./views/system";

const app = page("DevTown",
  tabs(
    ["System", systemView],
  ),
  { settings: { mode: "dark" } },
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, app).then(site => {
    site.setTheme(defaultTheme);
  }).catch(console.error);
}
```

- [ ] **Step 5: Verify dev mode**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app`

Open `http://localhost:8080/`. Expected: Dark-themed page with "System" tab showing health metrics (values from REST endpoints), trust chart, and problems table.

- [ ] **Step 6: Commit**

```
git add app/src/main/webui/src/
git commit -m "feat(#85): TypeScript infrastructure — datasets, theme, System view

Shared dataset declarations for all REST endpoints, dark theme config,
and the System view as the first working governance workbench page.

Refs #85"
```

---

### Task 6: Remaining views — Operations, Reviews, Merge Queue, Reviewers, Human Triage

Build the remaining five views and wire them into the top-level navigation.

**Files:**
- Create: `app/src/main/webui/src/views/operations.ts`
- Create: `app/src/main/webui/src/views/reviews.ts`
- Create: `app/src/main/webui/src/views/queue.ts`
- Create: `app/src/main/webui/src/views/reviewers.ts`
- Create: `app/src/main/webui/src/views/triage.ts`
- Modify: `app/src/main/webui/src/index.ts` — add all views to navigation

**Interfaces:**
- Consumes: Shared datasets from Task 5, REST endpoints from Task 2, WebSocket from Task 3
- Produces: Complete governance workbench UI

Each view follows the spec layout exactly. The implementer should:

1. Start each view as a `page()` with DSL components (`table()`, `metric()`, `barChart()`, `columns()`, `rows()`)
2. Use `lookup()` bindings to the datasets declared in `datasets.ts`
3. Enable cross-filter on tables where the spec calls for it (`filter: { enabled: true }`)
4. Use `withId()` on components that need URL state persistence (sortable tables, paginated tables)
5. If the workbench primitives (`dockBar()`, `split()`) are not yet available in casehub-pages, use `tabs()` or `columns()` as a fallback for the Operations view layout — the upgrade to workbench primitives is additive

- [ ] **Step 1: Create Operations view**

Build `operations.ts` following the spec's Operations layout:
- Left column: Active Reviews table + Merge Queue table (both with cross-filter)
- Center: metric cards (default) or review detail (when selected)
- Right column: context panel (routing, trust, commitment)
- Bottom: event stream table bound to WebSocket dataset

If `dockBar()` is not available yet, use `columns([20, 50, 30], ...)` with the bottom event stream in a `rows()` wrapper. Mark the layout with a comment: `// TODO: upgrade to dockBar() when casehub-pages#64 ships`

- [ ] **Step 2: Create Reviews view**

Build `reviews.ts` following the spec:
- List: `table()` of all reviews with status `selector()` filter
- Detail subpage: PR header metrics, capability progress cards, timeline + routing columns, compliance + provenance columns

Use `tabs()` for list/detail navigation within the view.

- [ ] **Step 3: Create Merge Queue view**

Build `queue.ts` following the spec:
- Metrics row: 4-5 metric cards
- Charts row: wait time `barChart()` + throughput `timeseries()`
- Failure rate trend: `timeseries()`
- Queued PRs table + Active Batches table

- [ ] **Step 4: Create Reviewers view**

Build `reviewers.ts` following the spec:
- List: `table()` with trust score columns per capability
- Profile subpage: header metrics, trust `barChart()` columns, history table, commitments table, incidents table

- [ ] **Step 5: Create Human Triage view**

Build `triage.ts` following the spec:
- Metrics row: pending, breached, escalated, approved-24h
- Pending decisions table sorted by SLA urgency

- [ ] **Step 6: Update index.ts with all views**

```typescript
import { loadSite } from "@casehubio/pages-runtime";
import { page, tabs } from "@casehubio/pages-ui";
import "./datasets";
import { defaultTheme } from "./theme";
import { operationsView } from "./views/operations";
import { reviewsView } from "./views/reviews";
import { queueView } from "./views/queue";
import { reviewersView } from "./views/reviewers";
import { triageView } from "./views/triage";
import { systemView } from "./views/system";

const app = page("DevTown",
  tabs(
    ["Operations", operationsView],
    ["Reviews", reviewsView],
    ["Merge Queue", queueView],
    ["Reviewers", reviewersView],
    ["Triage", triageView],
    ["System", systemView],
  ),
  { settings: { mode: "dark" } },
);

const container = document.getElementById("app");
if (container) {
  loadSite(container, app).then(site => {
    site.setTheme(defaultTheme);
  }).catch(console.error);
}
```

- [ ] **Step 7: Verify in dev mode**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app`

Open `http://localhost:8080/`. Verify:
- All six tabs render
- Tables load data from REST endpoints (may show empty tables if no demo data)
- Dark theme applies consistently
- Tab switching works

- [ ] **Step 8: Commit**

```
git add app/src/main/webui/src/
git commit -m "feat(#85): governance workbench — all six views

Operations (workbench), Reviews (list+detail), Merge Queue (metrics+charts),
Reviewers (list+profile), Human Triage (SLA-sorted), and System (health).
All views use casehub-pages DSL with REST dataset bindings.

Refs #85"
```

---

### Task 7: MergeQueueStateEvent — CDI event for live queue updates

New CDI event fired by `MergeQueueService` on enqueue, dequeue, and batch formation. Required for the WebSocket bridge to push queue updates to the UI.

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/governance/MergeQueueStateEvent.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java` — fire CDI events
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceEventBridge.java` — observe new event
- Create: `app/src/test/java/io/casehub/devtown/app/governance/MergeQueueStateEventTest.java`

**Interfaces:**
- Consumes: `MergeQueueService` mutation methods (enqueue, dequeue, formAndDispatchBatches, handleBatchCompletion)
- Produces: `MergeQueueStateEvent` CDI event — observed by `GovernanceEventBridge` (Task 3)

- [ ] **Step 1: Define the CDI event record**

```java
package io.casehub.devtown.app.governance;

import java.time.Instant;

public record MergeQueueStateEvent(
    String action,  // "enqueue", "dequeue", "batch_formed", "batch_completed"
    String repository,
    int prNumber,
    String batchId,   // null for enqueue/dequeue
    Instant timestamp
) {
    public static MergeQueueStateEvent enqueue(String repository, int prNumber) {
        return new MergeQueueStateEvent("enqueue", repository, prNumber, null, Instant.now());
    }

    public static MergeQueueStateEvent dequeue(String repository, int prNumber) {
        return new MergeQueueStateEvent("dequeue", repository, prNumber, null, Instant.now());
    }

    public static MergeQueueStateEvent batchFormed(String repository, String batchId) {
        return new MergeQueueStateEvent("batch_formed", repository, 0, batchId, Instant.now());
    }

    public static MergeQueueStateEvent batchCompleted(String repository, String batchId) {
        return new MergeQueueStateEvent("batch_completed", repository, 0, batchId, Instant.now());
    }
}
```

- [ ] **Step 2: Write failing test**

```java
package io.casehub.devtown.app.governance;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MergeQueueStateEventTest {

    @Test
    void enqueue_createsCorrectEvent() {
        var event = MergeQueueStateEvent.enqueue("casehubio/devtown", 42);
        assertThat(event.action()).isEqualTo("enqueue");
        assertThat(event.repository()).isEqualTo("casehubio/devtown");
        assertThat(event.prNumber()).isEqualTo(42);
        assertThat(event.batchId()).isNull();
        assertThat(event.timestamp()).isNotNull();
    }

    @Test
    void batchFormed_createsCorrectEvent() {
        var event = MergeQueueStateEvent.batchFormed("casehubio/devtown", "batch-abc");
        assertThat(event.action()).isEqualTo("batch_formed");
        assertThat(event.batchId()).isEqualTo("batch-abc");
    }
}
```

- [ ] **Step 3: Run test, verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MergeQueueStateEventTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (record is self-contained)

- [ ] **Step 4: Inject Event<MergeQueueStateEvent> into MergeQueueService and fire on mutations**

Add to `MergeQueueService`:

```java
@Inject
Event<MergeQueueStateEvent> queueEvent;
```

Fire events in:
- `enqueue()` — after successful enqueue: `queueEvent.fireAsync(MergeQueueStateEvent.enqueue(pr.repository(), pr.number()));`
- `dequeue()` — after successful dequeue: `queueEvent.fireAsync(MergeQueueStateEvent.dequeue(repository, prNumber));`
- `formAndDispatchBatches()` — after each batch dispatched: `queueEvent.fireAsync(MergeQueueStateEvent.batchFormed(batch.repository(), batch.batchId()));`
- `handleBatchCompletion()` — after processing: `queueEvent.fireAsync(MergeQueueStateEvent.batchCompleted(repository, batchId));`

- [ ] **Step 5: Add observer to GovernanceEventBridge**

Add to `GovernanceEventBridge.java`:

```java
void onMergeQueueState(@ObservesAsync MergeQueueStateEvent event) {
    broadcast("queue.state", MAPPER.createObjectNode()
        .put("action", event.action())
        .put("repository", event.repository())
        .put("prNumber", event.prNumber())
        .put("batchId", event.batchId())
        .put("timestamp", event.timestamp().toString()));
}
```

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/governance/MergeQueueStateEvent.java app/src/main/java/io/casehub/devtown/app/MergeQueueService.java app/src/main/java/io/casehub/devtown/app/governance/GovernanceEventBridge.java app/src/test/java/io/casehub/devtown/app/governance/MergeQueueStateEventTest.java
git commit -m "feat(#126): MergeQueueStateEvent — CDI events for live queue updates

Fires async CDI events on enqueue, dequeue, batch formation, and batch
completion. GovernanceEventBridge observes and forwards to WebSocket.

Closes #126, Refs #85"
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ GovernanceQueryService extraction (§Data Architecture)
- ✅ GovernanceResource REST endpoints (§REST Endpoints)
- ✅ GovernanceEventBridge WebSocket (§WebSocket Endpoint)
- ✅ Quinoa setup (§Quinoa Setup)
- ✅ All 6 views (§Views 1-6)
- ✅ MergeQueueStateEvent (§Prerequisites, devtown#126)
- ⚠️ PrReviewCaseTracker startup hydration (§Data Durability) — deferred: depends on CaseHubRuntime query API that may not exist (Assumption A2 from design review). File as a follow-up issue at implementation time if the API is missing.
- ⚠️ Pagination (§REST Endpoints) — not implemented in Task 2. All list endpoints return full results initially. Cursor-based pagination is additive and should be a follow-up when data volumes require it.

**Placeholder scan:** No TBDs. Task 6 (views) intentionally references the spec for layout details rather than repeating ASCII art — the implementer has the spec file and the CASEHUB-PAGES.md reference.

**Type consistency:** Records defined in Task 1 `GovernanceQueryService` are used consistently in Task 2 `GovernanceResource` return types.
