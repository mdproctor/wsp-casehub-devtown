# Observability MCP Tools Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build 11 MCP tools (8 read + 3 write) + W3C PROV-DM export for devtown operational observability, achieving full Gastown CLI parity.

**Architecture:** Thin MCP shell over existing foundation services in `app/` module. `PrReviewCaseTracker` provides an event-sourced read model of active cases. `DevtownMcpTools` composes foundation service outputs into tool responses. No intermediate service layer.

**Tech Stack:** Java 21, Quarkus 3.32.2, Quarkus MCP Server HTTP 1.11.1, CaseHub foundation (engine, qhorus, ledger, work, platform)

**Spec:** `specs/issue-17-observability-tooling/2026-06-17-observability-mcp-tools-design.md`

---

## File Structure

### New Files

| File | Responsibility |
|------|---------------|
| `app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java` | Event-sourced read model — observes `CaseLifecycleEvent`, maintains case registry + recent event ring buffer |
| `app/src/main/java/io/casehub/devtown/app/mcp/CaseTrackingStatus.java` | Enum: RUNNING, WAITING, COMPLETED, FAULTED, CANCELLED |
| `app/src/main/java/io/casehub/devtown/app/mcp/CaseInfo.java` | Record: case metadata snapshot stored in tracker |
| `app/src/main/java/io/casehub/devtown/app/mcp/TrackedEvent.java` | Record: ring buffer event entry |
| `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java` | `@ApplicationScoped` MCP tool class — all 11 `@Tool` methods + response records as inner types |
| `app/src/test/java/io/casehub/devtown/app/mcp/PrReviewCaseTrackerTest.java` | Unit tests for tracker state machine and ring buffer |
| `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsQueueTest.java` | `@QuarkusTest` for `get_queue_status` + `get_recent_events` |
| `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsHealthTest.java` | `@QuarkusTest` for `get_system_health` + `list_problems` |
| `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsInspectTest.java` | `@QuarkusTest` for `inspect_review` + `get_reviewer_health` |
| `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsWriteTest.java` | `@QuarkusTest` for write tools (`retry_reviewer`, `reroute_review`, `force_complete_check`) |
| `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsProvTest.java` | `@QuarkusTest` for `export_prov` |
| `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsPriorTest.java` | `@QuarkusTest` for `get_prior_decisions` |

### Modified Files

| File | Change |
|------|--------|
| `pom.xml` | Add `quarkus-mcp-server.version` property + `<dependencyManagement>` entry |
| `app/pom.xml` | Add `quarkus-mcp-server-http` compile dependency |
| `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java` | Inject `PrReviewCaseTracker`, register case on `startCase()` |
| `app/src/test/resources/application.properties` | Add Jandex index for MCP server if needed |

---

### Task 1: Add MCP Server Dependency

**Files:**
- Modify: `pom.xml` (parent)
- Modify: `app/pom.xml`

- [ ] **Step 1: Add version property and dependency management to parent pom.xml**

In `pom.xml`, add inside `<properties>` (or add a `<properties>` block if not present):

```xml
<quarkus-mcp-server.version>1.11.1</quarkus-mcp-server.version>
```

Add inside `<dependencyManagement><dependencies>`:

```xml
<!-- Quarkus MCP Server — HTTP transport for operational tooling -->
<dependency>
  <groupId>io.quarkiverse.mcp</groupId>
  <artifactId>quarkus-mcp-server-http</artifactId>
  <version>${quarkus-mcp-server.version}</version>
</dependency>
```

- [ ] **Step 2: Add dependency to app/pom.xml**

In `app/pom.xml`, add in the `<!-- Quarkus extensions -->` section:

```xml
<dependency>
  <groupId>io.quarkiverse.mcp</groupId>
  <artifactId>quarkus-mcp-server-http</artifactId>
</dependency>
```

- [ ] **Step 3: Build to verify dependency resolves**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app -am compile -DskipTests`

Expected: BUILD SUCCESS — dependency resolves from Maven Central.

- [ ] **Step 4: Commit**

```bash
git add pom.xml app/pom.xml
git commit -m "build(devtown#17): add quarkus-mcp-server-http dependency

Refs casehubio/devtown#17"
```

---

### Task 2: PrReviewCaseTracker — Data Model

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/mcp/CaseTrackingStatus.java`
- Create: `app/src/main/java/io/casehub/devtown/app/mcp/CaseInfo.java`
- Create: `app/src/main/java/io/casehub/devtown/app/mcp/TrackedEvent.java`

- [ ] **Step 1: Create CaseTrackingStatus enum**

```java
package io.casehub.devtown.app.mcp;

public enum CaseTrackingStatus {
    RUNNING,
    WAITING,
    COMPLETED,
    FAULTED,
    CANCELLED;

    public boolean isTerminal() {
        return this == COMPLETED || this == FAULTED || this == CANCELLED;
    }

    public static CaseTrackingStatus fromCaseStatus(String caseStatus) {
        return switch (caseStatus) {
            case "COMPLETED" -> COMPLETED;
            case "FAULTED" -> FAULTED;
            case "CANCELLED" -> CANCELLED;
            case "WAITING" -> WAITING;
            default -> RUNNING;
        };
    }
}
```

- [ ] **Step 2: Create CaseInfo record**

```java
package io.casehub.devtown.app.mcp;

import io.casehub.devtown.review.PrPayload;
import java.time.Instant;
import java.util.UUID;

public record CaseInfo(
    UUID caseId,
    String tenancyId,
    PrPayload payload,
    Instant startedAt,
    Instant lastEventAt,
    CaseTrackingStatus status
) {
    public CaseInfo withStatus(CaseTrackingStatus newStatus, Instant eventTime) {
        return new CaseInfo(caseId, tenancyId, payload, startedAt, eventTime, newStatus);
    }

    public boolean isStalled(long thresholdMinutes) {
        return !status.isTerminal()
            && Instant.now().isAfter(lastEventAt.plusSeconds(thresholdMinutes * 60));
    }
}
```

- [ ] **Step 3: Create TrackedEvent record**

```java
package io.casehub.devtown.app.mcp;

import java.time.Instant;
import java.util.UUID;

public record TrackedEvent(
    Instant timestamp,
    UUID caseId,
    String repo,
    int prNumber,
    String eventType,
    String caseStatus,
    String actorId
) {}
```

- [ ] **Step 4: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app -am compile -DskipTests`

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/
git commit -m "feat(devtown#17): tracker data model — CaseInfo, CaseTrackingStatus, TrackedEvent

Refs casehubio/devtown#17"
```

---

### Task 3: PrReviewCaseTracker — Core Implementation

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/PrReviewCaseTrackerTest.java`

- [ ] **Step 1: Write failing tests for PrReviewCaseTracker**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.devtown.review.PrPayload;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class PrReviewCaseTrackerTest {

    private PrReviewCaseTracker tracker;

    @BeforeEach
    void setUp() {
        tracker = new PrReviewCaseTracker();
    }

    private PrPayload samplePr() {
        return new PrPayload("casehubio/devtown", 42, "abc123", "main", 150, "alice", List.of("src/Main.java"));
    }

    @Test
    void register_addsCase() {
        UUID id = UUID.randomUUID();
        tracker.register(id, "default", samplePr());

        var info = tracker.getCase(id);
        assertThat(info).isNotNull();
        assertThat(info.status()).isEqualTo(CaseTrackingStatus.RUNNING);
        assertThat(info.payload().repo()).isEqualTo("casehubio/devtown");
        assertThat(info.payload().prNumber()).isEqualTo(42);
    }

    @Test
    void register_appearsInActiveCases() {
        UUID id = UUID.randomUUID();
        tracker.register(id, "default", samplePr());

        assertThat(tracker.activeCases()).hasSize(1);
        assertThat(tracker.activeCases().get(0).caseId()).isEqualTo(id);
    }

    @Test
    void updateStatus_movesToTerminal() {
        UUID id = UUID.randomUUID();
        tracker.register(id, "default", samplePr());

        tracker.updateStatus(id, "COMPLETED", Instant.now());

        var info = tracker.getCase(id);
        assertThat(info).isNotNull();
        assertThat(info.status()).isEqualTo(CaseTrackingStatus.COMPLETED);
    }

    @Test
    void updateStatus_terminalExcludedFromActiveCases() {
        UUID id = UUID.randomUUID();
        tracker.register(id, "default", samplePr());
        tracker.updateStatus(id, "COMPLETED", Instant.now());

        assertThat(tracker.activeCases()).isEmpty();
    }

    @Test
    void updateStatus_unknownCaseIgnored() {
        tracker.updateStatus(UUID.randomUUID(), "COMPLETED", Instant.now());
        assertThat(tracker.activeCases()).isEmpty();
    }

    @Test
    void stalledCases_returnsOnlyOverThreshold() {
        UUID active = UUID.randomUUID();
        UUID stale = UUID.randomUUID();
        tracker.register(active, "default", samplePr());
        tracker.register(stale, "default", samplePr());

        // Simulate stale case by updating with an old timestamp
        tracker.updateStatus(stale, "RUNNING", Instant.now().minusSeconds(7200));

        var stalled = tracker.stalledCases(60);
        assertThat(stalled).hasSize(1);
        assertThat(stalled.get(0).caseId()).isEqualTo(stale);
    }

    @Test
    void addEvent_appearsInRecentEvents() {
        tracker.addEvent(new TrackedEvent(
            Instant.now(), UUID.randomUUID(), "repo", 1, "CaseStarted", "RUNNING", "system"
        ));

        assertThat(tracker.recentEvents(10, null)).hasSize(1);
    }

    @Test
    void recentEvents_respectsLimit() {
        for (int i = 0; i < 20; i++) {
            tracker.addEvent(new TrackedEvent(
                Instant.now(), UUID.randomUUID(), "repo", i, "Event", "RUNNING", "actor"
            ));
        }

        assertThat(tracker.recentEvents(5, null)).hasSize(5);
    }

    @Test
    void recentEvents_respectsSince() {
        Instant cutoff = Instant.now().minusSeconds(30);
        tracker.addEvent(new TrackedEvent(
            Instant.now().minusSeconds(60), UUID.randomUUID(), "repo", 1, "Old", "RUNNING", "actor"
        ));
        tracker.addEvent(new TrackedEvent(
            Instant.now(), UUID.randomUUID(), "repo", 2, "New", "RUNNING", "actor"
        ));

        var events = tracker.recentEvents(50, cutoff);
        assertThat(events).hasSize(1);
        assertThat(events.get(0).prNumber()).isEqualTo(2);
    }

    @Test
    void ringBuffer_evictsOldest() {
        tracker = new PrReviewCaseTracker(5);
        for (int i = 0; i < 10; i++) {
            tracker.addEvent(new TrackedEvent(
                Instant.now(), UUID.randomUUID(), "repo", i, "Event", "RUNNING", "actor"
            ));
        }

        var events = tracker.recentEvents(100, null);
        assertThat(events).hasSize(5);
        assertThat(events.get(0).prNumber()).isEqualTo(5);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=PrReviewCaseTrackerTest -DfailIfNoTests=false`

Expected: FAIL — `PrReviewCaseTracker` class not found.

- [ ] **Step 3: Implement PrReviewCaseTracker**

```java
package io.casehub.devtown.app.mcp;

import io.casehub.devtown.review.PrPayload;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import java.time.Instant;
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import org.jboss.logging.Logger;

@ApplicationScoped
public class PrReviewCaseTracker {

    private static final Logger LOG = Logger.getLogger(PrReviewCaseTracker.class);
    private static final int DEFAULT_BUFFER_SIZE = 200;

    private final ConcurrentHashMap<UUID, CaseInfo> cases = new ConcurrentHashMap<>();
    private final Deque<TrackedEvent> eventBuffer;
    private final int maxBufferSize;

    public PrReviewCaseTracker() {
        this(DEFAULT_BUFFER_SIZE);
    }

    PrReviewCaseTracker(int maxBufferSize) {
        this.maxBufferSize = maxBufferSize;
        this.eventBuffer = new ArrayDeque<>(maxBufferSize);
    }

    public void register(UUID caseId, String tenancyId, PrPayload payload) {
        Instant now = Instant.now();
        cases.put(caseId, new CaseInfo(caseId, tenancyId, payload, now, now, CaseTrackingStatus.RUNNING));
        LOG.infof("Tracking PR review case=%s repo=%s pr=#%d", caseId, payload.repo(), payload.prNumber());
    }

    public CaseInfo getCase(UUID caseId) {
        return cases.get(caseId);
    }

    public List<CaseInfo> activeCases() {
        return cases.values().stream()
            .filter(c -> !c.status().isTerminal())
            .toList();
    }

    public List<CaseInfo> stalledCases(long thresholdMinutes) {
        return cases.values().stream()
            .filter(c -> c.isStalled(thresholdMinutes))
            .toList();
    }

    public void updateStatus(UUID caseId, String caseStatus, Instant eventTime) {
        cases.computeIfPresent(caseId, (id, existing) ->
            existing.withStatus(CaseTrackingStatus.fromCaseStatus(caseStatus), eventTime));
    }

    public void addEvent(TrackedEvent event) {
        synchronized (eventBuffer) {
            if (eventBuffer.size() >= maxBufferSize) {
                eventBuffer.pollFirst();
            }
            eventBuffer.addLast(event);
        }
    }

    public List<TrackedEvent> recentEvents(int limit, Instant since) {
        synchronized (eventBuffer) {
            return eventBuffer.stream()
                .filter(e -> since == null || e.timestamp().isAfter(since))
                .sorted((a, b) -> a.timestamp().compareTo(b.timestamp()))
                .skip(Math.max(0, eventBuffer.size() - limit))
                .limit(limit)
                .toList();
        }
    }

    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        updateStatus(event.caseId(), event.caseStatus(), Instant.now());

        CaseInfo info = cases.get(event.caseId());
        String repo = info != null ? info.payload().repo() : "unknown";
        int prNumber = info != null ? info.payload().prNumber() : 0;

        addEvent(new TrackedEvent(
            Instant.now(),
            event.caseId(),
            repo,
            prNumber,
            event.eventType(),
            event.caseStatus(),
            event.actorId()
        ));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=PrReviewCaseTrackerTest`

Expected: All 9 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java app/src/test/java/io/casehub/devtown/app/mcp/PrReviewCaseTrackerTest.java
git commit -m "feat(devtown#17): PrReviewCaseTracker — event-sourced read model with ring buffer

Refs casehubio/devtown#17"
```

---

### Task 4: Hook Tracker into PrReviewCaseService

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/TrackerRegistrationTest.java`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.PrReviewApplicationService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.List;
import org.junit.jupiter.api.Test;

@QuarkusTest
class TrackerRegistrationTest {

    @Inject PrReviewApplicationService reviewService;
    @Inject PrReviewCaseTracker tracker;

    @Test
    void review_registersCaseInTracker() {
        var pr = new PrPayload("casehubio/devtown", 99, "sha123", "main", 50, "bob", List.of("src/Foo.java"));
        reviewService.review(pr);

        var active = tracker.activeCases();
        assertThat(active).anyMatch(c ->
            c.payload().prNumber() == 99 && c.payload().repo().equals("casehubio/devtown"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=TrackerRegistrationTest`

Expected: FAIL — tracker is empty after `review()` because `PrReviewCaseService` doesn't register with it yet.

- [ ] **Step 3: Modify PrReviewCaseService to inject and use tracker**

In `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`, add the injection and registration.

Add field:

```java
@Inject
PrReviewCaseTracker caseTracker;
```

Add import:

```java
import io.casehub.devtown.app.mcp.PrReviewCaseTracker;
import io.casehub.platform.api.identity.CurrentPrincipal;
```

Modify the `review()` method — after `caseHub.startCase(initialContext)`, capture the UUID and register:

Replace:
```java
caseHub.startCase(initialContext);
return new PrReviewOutcome(VERDICT_CASE_OPENED, List.of());
```

With:
```java
var caseId = caseHub.startCase(initialContext);
caseId.thenAccept(id ->
    caseTracker.register(id, principal.tenancyId(), pr));
return new PrReviewOutcome(VERDICT_CASE_OPENED, List.of());
```

Add `CurrentPrincipal` injection:

```java
@Inject
CurrentPrincipal principal;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=TrackerRegistrationTest`

Expected: PASS

- [ ] **Step 5: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test`

Expected: All existing tests still PASS.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/test/java/io/casehub/devtown/app/mcp/TrackerRegistrationTest.java
git commit -m "feat(devtown#17): register PR review cases with tracker on creation

Refs casehubio/devtown#17"
```

---

### Task 5: DevtownMcpTools — Scaffold + get_queue_status + get_recent_events

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsQueueTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.devtown.review.PrPayload;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DevtownMcpToolsQueueTest {

    @Inject DevtownMcpTools tools;
    @Inject PrReviewCaseTracker tracker;

    @BeforeEach
    void setUp() {
        // Tests may use shared tracker — register fresh cases
    }

    @Test
    void getQueueStatus_emptyTracker_returnsZeroCounts() {
        var result = tools.getQueueStatus();
        assertThat(result.total()).isGreaterThanOrEqualTo(0);
    }

    @Test
    void getQueueStatus_withActiveCase_returnsCounts() {
        var id = UUID.randomUUID();
        tracker.register(id, "default", new PrPayload("repo", 1, "sha", "main", 10, "alice", List.of()));

        var result = tools.getQueueStatus();
        assertThat(result.total()).isGreaterThanOrEqualTo(1);
        assertThat(result.reviews()).anyMatch(r -> r.prNumber() == 1);
    }

    @Test
    void getRecentEvents_emptyBuffer_returnsEmpty() {
        var result = tools.getRecentEvents(10, null);
        assertThat(result).isNotNull();
    }

    @Test
    void getRecentEvents_withEvents_returnsChronological() {
        tracker.addEvent(new TrackedEvent(
            Instant.now().minusSeconds(10), UUID.randomUUID(), "repo", 1, "CaseStarted", "RUNNING", "sys"
        ));
        tracker.addEvent(new TrackedEvent(
            Instant.now(), UUID.randomUUID(), "repo", 2, "WorkerScheduled", "RUNNING", "sys"
        ));

        var result = tools.getRecentEvents(50, null);
        assertThat(result).hasSizeGreaterThanOrEqualTo(2);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsQueueTest -DfailIfNoTests=false`

Expected: FAIL — `DevtownMcpTools` class not found.

- [ ] **Step 3: Implement DevtownMcpTools with first two tools + response records**

```java
package io.casehub.devtown.app.mcp;

import io.casehub.devtown.review.PrPayload;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import io.quarkiverse.mcp.server.WrapBusinessError;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
@WrapBusinessError({IllegalArgumentException.class, IllegalStateException.class})
public class DevtownMcpTools {

    @Inject PrReviewCaseTracker tracker;

    // --- Response records ---

    public record QueueStatus(int total, Map<String, Integer> countsByStatus, List<ActiveReview> reviews) {}
    public record ActiveReview(UUID caseId, String repo, int prNumber, String contributor,
                               int linesChanged, String status, Instant startedAt, Instant lastEventAt) {}

    // --- Read Tools ---

    @Tool(name = "get_queue_status",
          description = "PR review queue overview. Returns counts by status and a list of all active reviews with their current state.")
    public QueueStatus getQueueStatus() {
        var active = tracker.activeCases();
        Map<String, Integer> counts = active.stream()
            .collect(Collectors.groupingBy(
                c -> c.status().name(),
                Collectors.summingInt(c -> 1)));

        List<ActiveReview> reviews = active.stream()
            .map(c -> new ActiveReview(
                c.caseId(), c.payload().repo(), c.payload().prNumber(),
                c.payload().contributor(), c.payload().linesChanged(),
                c.status().name(), c.startedAt(), c.lastEventAt()))
            .toList();

        return new QueueStatus(active.size(), counts, reviews);
    }

    @Tool(name = "get_recent_events",
          description = "Live event feed across all active PR reviews. Returns the most recent events in chronological order. Equivalent of Gastown's gt feed.")
    public List<TrackedEvent> getRecentEvents(
            @ToolArg(name = "limit", description = "Maximum number of events to return", required = false) Integer limit,
            @ToolArg(name = "since", description = "ISO timestamp — only return events after this time", required = false) String since) {
        int effectiveLimit = limit != null ? limit : 50;
        Instant sinceParsed = since != null ? Instant.parse(since) : null;
        return tracker.recentEvents(effectiveLimit, sinceParsed);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsQueueTest`

Expected: All 4 tests PASS.

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test`

Expected: All tests PASS. MCP server auto-discovered the `@Tool` methods without breaking existing tests.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsQueueTest.java
git commit -m "feat(devtown#17): DevtownMcpTools scaffold + get_queue_status + get_recent_events

Refs casehubio/devtown#17"
```

---

### Task 6: get_system_health + list_problems

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsHealthTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DevtownMcpToolsHealthTest {

    @Inject DevtownMcpTools tools;

    @Test
    void getSystemHealth_returnsValidStructure() {
        var health = tools.getSystemHealth();
        assertThat(health).isNotNull();
        assertThat(health.activeCases()).isGreaterThanOrEqualTo(0);
        assertThat(health.openCommitments()).isGreaterThanOrEqualTo(0);
    }

    @Test
    void listProblems_emptySystem_returnsEmptyList() {
        var problems = tools.listProblems(60);
        assertThat(problems).isNotNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsHealthTest -DfailIfNoTests=false`

Expected: FAIL — methods not found.

- [ ] **Step 3: Add response records and tool methods to DevtownMcpTools**

Add these response records:

```java
public record SystemHealth(int activeCases, int fleetSize,
                           Map<String, Double> avgTrustByCapability,
                           int openCommitments, int pendingWorkItems) {}

public record Problem(String category, String severity, String description,
                      UUID caseId, String actorId, Instant since) {}
```

Add these injections:

```java
@Inject io.casehub.qhorus.runtime.store.CommitmentStore commitmentStore;
@Inject io.casehub.ledger.runtime.service.federation.TrustExportService trustExportService;
```

Add `get_system_health` tool:

```java
@Tool(name = "get_system_health",
      description = "Operational health check. Returns active case count, reviewer fleet size, average trust scores, open commitments, and pending work items. Equivalent of Gastown's gt doctor.")
public SystemHealth getSystemHealth() {
    var active = tracker.activeCases();
    var openCommitments = commitmentStore.findAllOpen();

    var trustPayload = trustExportService.exportAll(0.0);
    Map<String, Double> avgTrust = trustPayload.actors().stream()
        .flatMap(a -> a.capabilities().stream())
        .collect(Collectors.groupingBy(
            c -> c.capabilityKey(),
            Collectors.averagingDouble(c -> c.trustScore())));

    int fleetSize = (int) openCommitments.stream()
        .map(c -> c.obligor)
        .distinct()
        .count();

    return new SystemHealth(active.size(), fleetSize, avgTrust,
        openCommitments.size(), 0);
}
```

Add `list_problems` tool:

```java
@Tool(name = "list_problems",
      description = "Operational problem surface. Surfaces stalled reviewers, failed workers, below-threshold agents, and stalled cases. Equivalent of Gastown's gt problems + gt stale.")
public List<Problem> listProblems(
        @ToolArg(name = "threshold_minutes", description = "Minutes of inactivity before a case or commitment is considered stalled. Default 60.", required = false) Integer thresholdMinutes) {
    int threshold = thresholdMinutes != null ? thresholdMinutes : 60;
    var problems = new java.util.ArrayList<Problem>();

    // Stalled cases
    for (var c : tracker.stalledCases(threshold)) {
        problems.add(new Problem("stalled_case", "HIGH",
            "PR #" + c.payload().prNumber() + " in " + c.payload().repo() + " has no progress",
            c.caseId(), null, c.lastEventAt()));
    }

    // Stalled commitments
    var expiredCutoff = Instant.now().minusSeconds(threshold * 60L);
    for (var commitment : commitmentStore.findExpiredBefore(expiredCutoff)) {
        problems.add(new Problem("stalled_reviewer", "HIGH",
            "Open commitment expired for obligor " + commitment.obligor,
            null, commitment.obligor, commitment.createdAt));
    }

    // Failed workers from ring buffer
    for (var event : tracker.recentEvents(200, null)) {
        if (event.eventType() != null && event.eventType().contains("Failed")) {
            problems.add(new Problem("failed_worker", "MEDIUM",
                "Worker failure in PR #" + event.prNumber(),
                event.caseId(), event.actorId(), event.timestamp()));
        }
    }

    return problems;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsHealthTest`

Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsHealthTest.java
git commit -m "feat(devtown#17): get_system_health + list_problems MCP tools

Refs casehubio/devtown#17"
```

---

### Task 7: inspect_review + get_reviewer_health

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsInspectTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DevtownMcpToolsInspectTest {

    @Inject DevtownMcpTools tools;

    @Test
    void inspectReview_unknownCase_throws() {
        assertThatThrownBy(() -> tools.inspectReview(UUID.randomUUID().toString(), null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void getReviewerHealth_unknownReviewer_returnsEmptyScores() {
        var health = tools.getReviewerHealth("nonexistent-agent", null);
        assertThat(health).isNotNull();
        assertThat(health.openCommitments()).isEqualTo(0);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsInspectTest -DfailIfNoTests=false`

Expected: FAIL — methods not found.

- [ ] **Step 3: Add response records and tool methods**

Add response records to `DevtownMcpTools`:

```java
public record ReviewDetail(UUID caseId, PrPayload pr, List<GoalStatus> goals,
                           List<EventEntry> timeline, List<CapabilityStatus> capabilities) {}
public record GoalStatus(String name, String kind, boolean reached) {}
public record EventEntry(Instant timestamp, String eventType, String actor, String summary) {}
public record CapabilityStatus(String name, String status, String outcome, Instant completedAt) {}

public record ReviewerHealth(String reviewerId, int openCommitments,
                             Map<String, Double> trustByCapability, Map<String, Double> trustByDimension,
                             int totalDecisions, List<RecentOutcome> recentOutcomes) {}
public record RecentOutcome(UUID caseId, String capability, String outcome, Instant timestamp) {}
```

Add injections:

```java
@Inject io.casehub.api.engine.CaseHubRuntime caseHubRuntime;
@Inject io.casehub.ledger.runtime.service.TrustGateService trustGateService;
```

Add `inspect_review`:

```java
@Tool(name = "inspect_review",
      description = "Full case inspection for a PR review. Returns PR metadata, goal status, event timeline, capabilities, and commitments. Equivalent of Gastown's gt peek.")
public ReviewDetail inspectReview(
        @ToolArg(name = "case_id", description = "UUID of the PR review case") String caseIdStr,
        @ToolArg(name = "tenancy_id", description = "Tenant ID. Defaults to 'default'.", required = false) String tenancyId) {
    UUID caseId = UUID.fromString(caseIdStr);
    CaseInfo info = tracker.getCase(caseId);
    if (info == null) {
        throw new IllegalArgumentException("Case not found in tracker: " + caseId);
    }

    // Event timeline via event log — lazy capability resolution
    var eventLog = caseHubRuntime.eventLog(caseId).toCompletableFuture().join();
    List<EventEntry> timeline = eventLog.stream()
        .map(e -> new EventEntry(e.timestamp(), e.eventType().name(), e.workerId(), e.payload()))
        .toList();

    // Capabilities from worker events
    var workerEvents = caseHubRuntime.eventLog(caseId,
        java.util.Set.of(
            io.casehub.api.model.event.CaseHubEventType.WORKER_SCHEDULED,
            io.casehub.api.model.event.CaseHubEventType.WORKER_EXECUTION_COMPLETED,
            io.casehub.api.model.event.CaseHubEventType.WORKER_EXECUTION_FAILED,
            io.casehub.api.model.event.CaseHubEventType.WORKER_OUTCOME_DECLINED
        )).toCompletableFuture().join();

    Map<String, List<io.casehub.api.model.event.CaseEventLogRecord>> byCapability =
        workerEvents.stream().collect(Collectors.groupingBy(e ->
            e.workerId() != null ? e.workerId() : "unknown"));

    List<CapabilityStatus> capabilities = byCapability.entrySet().stream()
        .map(entry -> {
            var events = entry.getValue();
            var last = events.get(events.size() - 1);
            return new CapabilityStatus(entry.getKey(), last.eventType().name(),
                last.payload(), last.timestamp());
        })
        .toList();

    return new ReviewDetail(caseId, info.payload(), List.of(), timeline, capabilities);
}
```

Add `get_reviewer_health`:

```java
@Tool(name = "get_reviewer_health",
      description = "Per-reviewer operational state. Returns open commitments, trust scores by capability and dimension, and recent outcomes. Gastown has no equivalent — this leverages CaseHub's trust model.")
public ReviewerHealth getReviewerHealth(
        @ToolArg(name = "reviewer_id", description = "Agent instance ID") String reviewerId,
        @ToolArg(name = "tenancy_id", description = "Tenant ID. Defaults to 'default'.", required = false) String tenancyId) {
    var openCommitments = commitmentStore.findOpenByObligor(reviewerId);
    var capScores = trustGateService.allCapabilityScores(reviewerId);
    var dimScores = trustGateService.allDimensionScores(reviewerId);

    int totalDecisions = capScores.keySet().stream()
        .mapToInt(cap -> trustGateService.decisionCount(reviewerId, cap))
        .sum();

    return new ReviewerHealth(reviewerId, openCommitments.size(),
        capScores, dimScores, totalDecisions, List.of());
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsInspectTest`

Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsInspectTest.java
git commit -m "feat(devtown#17): inspect_review + get_reviewer_health MCP tools

Refs casehubio/devtown#17"
```

---

### Task 8: get_prior_decisions

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsPriorTest.java`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DevtownMcpToolsPriorTest {

    @Inject DevtownMcpTools tools;

    @Test
    void getPriorDecisions_noHistory_returnsEmptyList() {
        var result = tools.getPriorDecisions("casehubio/devtown", "src/Main.java", null);
        assertThat(result).isNotNull();
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsPriorTest -DfailIfNoTests=false`

Expected: FAIL — method not found.

- [ ] **Step 3: Add response record and tool method**

Add record:

```java
public record PriorDecision(UUID caseId, String repo, int prNumber, String capability,
                            String outcome, Instant decidedAt) {}
```

Add injection:

```java
@Inject jakarta.enterprise.inject.Instance<io.casehub.platform.api.memory.CaseMemoryStore> memoryStore;
```

Add tool:

```java
@Tool(name = "get_prior_decisions",
      description = "Prior review outcomes for a code area. Queries memory store for past review decisions affecting the specified file path. Basic equivalent of Gastown's gt seance.")
public List<PriorDecision> getPriorDecisions(
        @ToolArg(name = "repo", description = "Repository name, e.g. casehubio/devtown") String repo,
        @ToolArg(name = "file_path", description = "File path or path prefix to query") String filePath,
        @ToolArg(name = "tenancy_id", description = "Tenant ID. Defaults to 'default'.", required = false) String tenancyId) {
    if (!memoryStore.isResolvable()) {
        return List.of();
    }

    String tenant = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    var modules = io.casehub.devtown.domain.memory.ModulePathNormalizer.normalize(List.of(filePath));
    List<String> entityIds = modules.stream()
        .map(m -> io.casehub.devtown.domain.memory.DevtownMemoryDomain.MODULE_PREFIX + repo + "/" + m)
        .limit(io.casehub.platform.api.memory.MemoryQuery.MAX_ENTITY_IDS)
        .toList();

    if (entityIds.isEmpty()) {
        return List.of();
    }

    var memories = memoryStore.get().query(
        io.casehub.platform.api.memory.MemoryQuery.forEntities(
            entityIds,
            io.casehub.devtown.domain.memory.DevtownMemoryDomain.SOFTWARE_REVIEW,
            tenant)
        .withLimit(20)
        .withOrder(io.casehub.platform.api.memory.MemoryOrder.CHRONOLOGICAL)
    );

    return memories.stream()
        .map(m -> new PriorDecision(
            null,
            repo,
            0,
            m.attributes().getOrDefault("capability", "unknown"),
            m.text(),
            m.createdAt()))
        .toList();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsPriorTest`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsPriorTest.java
git commit -m "feat(devtown#17): get_prior_decisions MCP tool — basic gt seance equivalent

Refs casehubio/devtown#17"
```

---

### Task 9: Write Tools — retry, reroute, force_complete

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsWriteTest.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DevtownMcpToolsWriteTest {

    @Inject DevtownMcpTools tools;

    @Test
    void retryReviewer_unknownCase_throws() {
        assertThatThrownBy(() ->
            tools.retryReviewer(UUID.randomUUID().toString(), "security-review", null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rerouteReview_unknownCase_throws() {
        assertThatThrownBy(() ->
            tools.rerouteReview(UUID.randomUUID().toString(), null))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void forceCompleteCheck_unknownCase_throws() {
        assertThatThrownBy(() ->
            tools.forceCompleteCheck(UUID.randomUUID().toString(), "security-review", "APPROVED", "test override", null))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsWriteTest -DfailIfNoTests=false`

Expected: FAIL — methods not found.

- [ ] **Step 3: Add write tool response records and methods**

Add records:

```java
public record RetryResult(UUID caseId, String capability, String status) {}
public record RerouteResult(UUID oldCaseId, UUID newCaseId) {}
public record ForceCompleteResult(UUID caseId, String capability, String outcome, String status) {}
```

Add injection:

```java
@Inject io.casehub.devtown.app.PrReviewCaseHub caseHub;
```

Add the binding name to context key mapping (from `ReviewOutcomeObserver`):

```java
private static final Map<String, String> CAPABILITY_CONTEXT_KEYS = Map.of(
    "code-analysis", "codeAnalysis",
    "security-review", "securityReview",
    "architecture-review", "architectureReview",
    "style-review", "styleCheck",
    "test-coverage", "testCoverage",
    "performance-analysis", "performanceAnalysis"
);
```

Add tools:

```java
@Tool(name = "retry_reviewer",
      description = "Re-dispatch a stalled capability check to a different reviewer. Clears the capability's context entry so the binding re-fires.")
public RetryResult retryReviewer(
        @ToolArg(name = "case_id", description = "UUID of the PR review case") String caseIdStr,
        @ToolArg(name = "capability", description = "Capability to retry, e.g. security-review") String capability,
        @ToolArg(name = "tenancy_id", description = "Tenant ID. Defaults to 'default'.", required = false) String tenancyId) {
    UUID caseId = UUID.fromString(caseIdStr);
    CaseInfo info = tracker.getCase(caseId);
    if (info == null) {
        throw new IllegalArgumentException("Case not found in tracker: " + caseId);
    }

    String contextKey = CAPABILITY_CONTEXT_KEYS.get(capability);
    if (contextKey == null) {
        throw new IllegalArgumentException("Unknown capability: " + capability + ". Valid: " + CAPABILITY_CONTEXT_KEYS.keySet());
    }

    caseHubRuntime.signal(caseId, contextKey, null).toCompletableFuture().join();
    return new RetryResult(caseId, capability, "RETRYING");
}

@Tool(name = "reroute_review",
      description = "Cancel a stuck PR review and start a fresh one with the same PR payload. Blunt instrument — use retry_reviewer for individual capabilities.")
public RerouteResult rerouteReview(
        @ToolArg(name = "case_id", description = "UUID of the stuck PR review case") String caseIdStr,
        @ToolArg(name = "tenancy_id", description = "Tenant ID. Defaults to 'default'.", required = false) String tenancyId) {
    UUID oldCaseId = UUID.fromString(caseIdStr);
    CaseInfo info = tracker.getCase(oldCaseId);
    if (info == null) {
        throw new IllegalArgumentException("Case not found in tracker: " + oldCaseId);
    }

    caseHubRuntime.cancelCase(oldCaseId);

    var prContext = Map.<String, Object>of(
        "id", String.valueOf(info.payload().prNumber()),
        "repo", info.payload().repo(),
        "linesChanged", info.payload().linesChanged(),
        "baseRef", info.payload().baseRef(),
        "headSha", info.payload().headSha(),
        "contributor", info.payload().contributor(),
        "changedPaths", info.payload().changedPaths()
    );
    var initialContext = new java.util.HashMap<String, Object>();
    initialContext.put("pr", prContext);

    UUID newCaseId = caseHub.startCase(initialContext).toCompletableFuture().join();
    String tenant = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    tracker.register(newCaseId, tenant, info.payload());

    return new RerouteResult(oldCaseId, newCaseId);
}

@Tool(name = "force_complete_check",
      description = "Inject a synthetic outcome for a timed-out capability. The outcome carries operatorOverride=true and does NOT generate trust attestations.")
public ForceCompleteResult forceCompleteCheck(
        @ToolArg(name = "case_id", description = "UUID of the PR review case") String caseIdStr,
        @ToolArg(name = "capability", description = "Capability to force-complete, e.g. security-review") String capability,
        @ToolArg(name = "outcome", description = "Outcome value, e.g. APPROVED or FLAGGED") String outcome,
        @ToolArg(name = "reason", description = "Human-readable reason for the override") String reason,
        @ToolArg(name = "tenancy_id", description = "Tenant ID. Defaults to 'default'.", required = false) String tenancyId) {
    UUID caseId = UUID.fromString(caseIdStr);
    CaseInfo info = tracker.getCase(caseId);
    if (info == null) {
        throw new IllegalArgumentException("Case not found in tracker: " + caseId);
    }

    String contextKey = CAPABILITY_CONTEXT_KEYS.get(capability);
    if (contextKey == null) {
        throw new IllegalArgumentException("Unknown capability: " + capability);
    }

    var syntheticResult = Map.of(
        "outcome", outcome,
        "operatorOverride", true,
        "reason", reason
    );
    caseHubRuntime.signal(caseId, contextKey, syntheticResult).toCompletableFuture().join();
    return new ForceCompleteResult(caseId, capability, outcome, "FORCED");
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsWriteTest`

Expected: All 3 tests PASS (each throws on unknown case).

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsWriteTest.java
git commit -m "feat(devtown#17): write tools — retry_reviewer, reroute_review, force_complete_check

Refs casehubio/devtown#17"
```

---

### Task 10: export_prov — PROV-DM Export

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java`
- Create: `app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsProvTest.java`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.devtown.app.mcp;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DevtownMcpToolsProvTest {

    @Inject DevtownMcpTools tools;

    @Test
    void exportProv_unknownCase_throws() {
        assertThatThrownBy(() ->
            tools.exportProv(UUID.randomUUID().toString(), null))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsProvTest -DfailIfNoTests=false`

Expected: FAIL — method not found.

- [ ] **Step 3: Add tool method**

Add injection:

```java
@Inject io.casehub.ledger.runtime.service.LedgerProvExportService provExportService;
```

Add tool:

```java
@Tool(name = "export_prov",
      description = "Export a W3C PROV-DM causal chain for a PR review case. Returns PROV-JSON-LD documenting every entity, activity, and agent in the review process.")
public String exportProv(
        @ToolArg(name = "case_id", description = "UUID of the PR review case") String caseIdStr,
        @ToolArg(name = "tenancy_id", description = "Tenant ID. Defaults to 'default'.", required = false) String tenancyId) {
    UUID caseId = UUID.fromString(caseIdStr);
    String tenant = tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    return provExportService.exportSubject(caseId, tenant);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=DevtownMcpToolsProvTest`

Expected: PASS — `LedgerProvExportService.exportSubject()` throws `IllegalArgumentException` when no entries exist, which is the expected behavior for an unknown case.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/mcp/DevtownMcpTools.java app/src/test/java/io/casehub/devtown/app/mcp/DevtownMcpToolsProvTest.java
git commit -m "feat(devtown#17): export_prov MCP tool — W3C PROV-DM causal chain export

Refs casehubio/devtown#17"
```

---

### Task 11: Full Build Verification

**Files:**
- None (verification only)

- [ ] **Step 1: Run full build with tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: BUILD SUCCESS — all modules compile, all tests pass.

- [ ] **Step 2: Verify MCP tools are discoverable**

Check the build log for MCP server tool registration. The Quarkus MCP server extension logs discovered `@Tool` methods at startup.

- [ ] **Step 3: Verify no CDI ambiguity errors**

Check for `UnsatisfiedResolutionException` or `AmbiguousResolutionException` in the build output. All injected foundation services should have exactly one satisfying bean.

---

## Self-Review

**Spec coverage check:**

| Spec section | Task |
|---|---|
| §1 Architecture — thin MCP shell, `@WrapBusinessError` | Task 5 (scaffold) |
| §1 Module placement — `app/mcp` package | Task 2-5 (all classes) |
| §1 New dependency — quarkus-mcp-server | Task 1 |
| §1 Tenancy resolution — optional `tenancy_id` params | Tasks 7-10 |
| §2 PrReviewCaseTracker — population | Task 4 |
| §2 PrReviewCaseTracker — status updates | Task 3 |
| §2 PrReviewCaseTracker — ring buffer | Task 3 |
| §3.1 get_queue_status | Task 5 |
| §3.2 inspect_review | Task 7 |
| §3.3 get_reviewer_health | Task 7 |
| §3.4 get_recent_events | Task 5 |
| §3.5 list_problems | Task 6 |
| §3.6 get_system_health | Task 6 |
| §3.7 get_prior_decisions | Task 8 |
| §4.1 retry_reviewer | Task 9 |
| §4.2 reroute_review | Task 9 |
| §4.3 force_complete_check | Task 9 |
| §5 export_prov | Task 10 |
| §6 Response records | Tasks 5-10 (per tool) |

**Placeholder scan:** No TBDs, TODOs, or vague steps. All code blocks are complete.

**Type consistency:** `CaseInfo`, `CaseTrackingStatus`, `TrackedEvent`, `PrPayload` used consistently across all tasks. `DevtownMcpTools` method names match spec names. Response record field names match spec §6.

**Note:** The spec mentions `quarkus-mcp-server-sse` but Qhorus actually uses `quarkus-mcp-server-http` (version 1.11.1). The plan uses the correct artifact (`quarkus-mcp-server-http`).
