# GitHub Combined-Status Check Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace in-memory CI aggregation with GitHub Checks API as the single source of truth for combined CI status.

**Architecture:** Domain SPI `CiStatusClient` in `domain/` (alongside `MergeClient`), implemented by `GitHubCiStatusClient` in `github/` using the existing `github-api` MP REST Client config. `PrReviewCaseService` drops the in-memory `ciSuiteResults` map and delegates aggregate status to the SPI. Individual suite results still recorded in case context for audit.

**Tech Stack:** Java 21, Quarkus 3.32.2 (MicroProfile REST Client), JUnit 5, Mockito, AssertJ

**Spec:** `specs/2026-06-23-github-combined-status-design.md` (rev 3)

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

---

## File Map

| File | Module | Action | Responsibility |
|------|--------|--------|----------------|
| `domain/src/main/java/io/casehub/devtown/domain/CombinedCiStatus.java` | domain | Create | Sealed result type — Passing, Failing, Pending, Unavailable |
| `domain/src/main/java/io/casehub/devtown/domain/CiStatusClient.java` | domain | Create | SPI interface — `getCombinedStatus(owner, repo, headSha)` |
| `github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java` | github | Create | MP REST Client interface for `GET /repos/{owner}/{repo}/commits/{ref}/check-suites` |
| `github/src/main/java/io/casehub/devtown/github/CheckSuitesResponse.java` | github | Create | API response record — `total_count`, `List<CheckSuite>` |
| `github/src/main/java/io/casehub/devtown/github/GitHubCiStatusClient.java` | github | Create | SPI implementation — queries GitHub, classifies conclusions, handles errors |
| `github/src/test/java/io/casehub/devtown/github/GitHubCiStatusClientTest.java` | github | Create | Unit tests — all conclusion classifications, pagination, errors |
| `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java` | app | Modify | Remove in-memory map, inject CiStatusClient, delegate aggregate. Fix `@Alternative @Priority(2)` anti-pattern. |
| `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java` | app | Modify | Replace in-memory aggregation tests with CiStatusClient mock tests |

---

## Task 1: Domain SPI — `CombinedCiStatus` and `CiStatusClient`

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/CombinedCiStatus.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/CiStatusClient.java`

- [ ] **Step 1: Create `CombinedCiStatus` sealed interface**

```java
package io.casehub.devtown.domain;

public sealed interface CombinedCiStatus {
    record Passing() implements CombinedCiStatus {}
    record Failing(String summary) implements CombinedCiStatus {}
    record Pending(int completed, int total) implements CombinedCiStatus {}
    record Unavailable(String reason) implements CombinedCiStatus {}
}
```

- [ ] **Step 2: Create `CiStatusClient` SPI interface**

```java
package io.casehub.devtown.domain;

public interface CiStatusClient {
    CombinedCiStatus getCombinedStatus(String owner, String repo, String headSha);
}
```

Parameters follow the `MergeClient.merge(String owner, String repo, ...)` convention.

- [ ] **Step 3: Compile domain module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl domain compile`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add domain/src/main/java/io/casehub/devtown/domain/CombinedCiStatus.java domain/src/main/java/io/casehub/devtown/domain/CiStatusClient.java
git commit -m "feat(domain): add CiStatusClient SPI and CombinedCiStatus sealed result type

Refs casehubio/devtown#89"
```

---

## Task 2: GitHub REST Client — `GitHubChecksApi` and `CheckSuitesResponse`

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java`
- Create: `github/src/main/java/io/casehub/devtown/github/CheckSuitesResponse.java`

- [ ] **Step 1: Create `CheckSuitesResponse` record**

```java
package io.casehub.devtown.github;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = true)
public record CheckSuitesResponse(
    @JsonProperty("total_count") int totalCount,
    @JsonProperty("check_suites") List<CheckSuite> checkSuites
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record CheckSuite(
        long id,
        String status,
        String conclusion
    ) {}
}
```

- [ ] **Step 2: Create `GitHubChecksApi` REST client interface**

```java
package io.casehub.devtown.github;

import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;

@RegisterRestClient(configKey = "github-api")
@Path("/repos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface GitHubChecksApi {

    @GET
    @Path("/{owner}/{repo}/commits/{ref}/check-suites")
    CheckSuitesResponse listCheckSuites(
        @PathParam("owner") String owner,
        @PathParam("repo") String repo,
        @PathParam("ref") String ref,
        @QueryParam("per_page") int perPage);
}
```

Reuses `configKey = "github-api"` — same base URL and auth as `GitHubMergeApi`.

- [ ] **Step 3: Compile github module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl github compile`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```
git add github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java github/src/main/java/io/casehub/devtown/github/CheckSuitesResponse.java
git commit -m "feat(github): add GitHubChecksApi REST client and CheckSuitesResponse DTO

Refs casehubio/devtown#89"
```

---

## Task 3: `GitHubCiStatusClient` — tests first

**Files:**
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubCiStatusClientTest.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubCiStatusClient.java`

- [ ] **Step 1: Write all failing tests**

Follow the `GitHubMergeClientTest` pattern — mock the API interface, construct the client directly.

```java
package io.casehub.devtown.github;

import io.casehub.devtown.domain.CombinedCiStatus;
import jakarta.ws.rs.WebApplicationException;
import jakarta.ws.rs.ProcessingException;
import jakarta.ws.rs.core.Response;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class GitHubCiStatusClientTest {

    private GitHubChecksApi api;
    private GitHubCiStatusClient client;

    @BeforeEach
    void setUp() {
        api = mock(GitHubChecksApi.class);
        client = new GitHubCiStatusClient(api);
    }

    @Test
    void allSuitesSuccess_returnsPassing() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(2,
                suite(1, "completed", "success"),
                suite(2, "completed", "success")));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Passing.class);
    }

    @Test
    void successAndNeutral_returnsPassing() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(2,
                suite(1, "completed", "success"),
                suite(2, "completed", "neutral")));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Passing.class);
    }

    @Test
    void successAndSkipped_returnsPassing() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(2,
                suite(1, "completed", "success"),
                suite(2, "completed", "skipped")));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Passing.class);
    }

    @Test
    void oneFailure_returnsFailing() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(3,
                suite(1, "completed", "success"),
                suite(2, "completed", "failure"),
                suite(3, "completed", "success")));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Failing.class);
        assertThat(((CombinedCiStatus.Failing) result).summary()).contains("1");
    }

    @Test
    void cancelled_isBlocking() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(1,
                suite(1, "completed", "cancelled")));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Failing.class);
    }

    @Test
    void timedOut_isBlocking() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(1,
                suite(1, "completed", "timed_out")));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Failing.class);
    }

    @Test
    void someInProgress_returnsPending() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(3,
                suite(1, "completed", "success"),
                suite(2, "in_progress", null),
                suite(3, "queued", null)));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Pending.class);
        var pending = (CombinedCiStatus.Pending) result;
        assertThat(pending.completed()).isEqualTo(1);
        assertThat(pending.total()).isEqualTo(3);
    }

    @Test
    void zeroSuites_returnsPending() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(response(0));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Pending.class);
        var pending = (CombinedCiStatus.Pending) result;
        assertThat(pending.completed()).isEqualTo(0);
        assertThat(pending.total()).isEqualTo(0);
    }

    @Test
    void totalCountExceedsReturnedList_returnsPending() {
        when(api.listCheckSuites(eq("casehubio"), eq("devtown"), eq("sha123"), eq(100)))
            .thenReturn(new CheckSuitesResponse(105, List.of(
                suite(1, "completed", "success"))));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Pending.class);
    }

    @Test
    void webApplicationException_returnsUnavailable() {
        when(api.listCheckSuites(anyString(), anyString(), anyString(), anyInt()))
            .thenThrow(new WebApplicationException(Response.status(403).build()));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Unavailable.class);
        assertThat(((CombinedCiStatus.Unavailable) result).reason()).contains("403");
    }

    @Test
    void processingException_returnsUnavailable() {
        when(api.listCheckSuites(anyString(), anyString(), anyString(), anyInt()))
            .thenThrow(new ProcessingException("connection reset"));

        var result = client.getCombinedStatus("casehubio", "devtown", "sha123");

        assertThat(result).isInstanceOf(CombinedCiStatus.Unavailable.class);
        assertThat(((CombinedCiStatus.Unavailable) result).reason()).contains("connection reset");
    }

    private static CheckSuitesResponse response(int totalCount, CheckSuitesResponse.CheckSuite... suites) {
        return new CheckSuitesResponse(totalCount, List.of(suites));
    }

    private static CheckSuitesResponse.CheckSuite suite(long id, String status, String conclusion) {
        return new CheckSuitesResponse.CheckSuite(id, status, conclusion);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl github test -Dtest=GitHubCiStatusClientTest`
Expected: FAIL — `GitHubCiStatusClient` class does not exist.

- [ ] **Step 3: Implement `GitHubCiStatusClient`**

```java
package io.casehub.devtown.github;

import io.casehub.devtown.domain.CiStatusClient;
import io.casehub.devtown.domain.CombinedCiStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.ws.rs.WebApplicationException;
import org.eclipse.microprofile.rest.client.inject.RestClient;
import org.jboss.logging.Logger;
import java.util.Set;

@ApplicationScoped
public class GitHubCiStatusClient implements CiStatusClient {

    private static final Logger LOG = Logger.getLogger(GitHubCiStatusClient.class);
    private static final Set<String> NON_BLOCKING = Set.of("success", "neutral", "skipped");

    private final GitHubChecksApi api;

    public GitHubCiStatusClient(@RestClient GitHubChecksApi api) {
        this.api = api;
    }

    @Override
    public CombinedCiStatus getCombinedStatus(String owner, String repo, String headSha) {
        CheckSuitesResponse response;
        try {
            response = api.listCheckSuites(owner, repo, headSha, 100);
        } catch (WebApplicationException e) {
            int status = e.getResponse().getStatus();
            LOG.warnf("GitHub Checks API returned HTTP %d for %s/%s@%s", status, owner, repo, headSha);
            return new CombinedCiStatus.Unavailable("api error: HTTP " + status);
        } catch (Exception e) {
            LOG.warnf(e, "GitHub Checks API call failed for %s/%s@%s", owner, repo, headSha);
            return new CombinedCiStatus.Unavailable("api error: " + e.getMessage());
        }

        if (response.totalCount() == 0) {
            return new CombinedCiStatus.Pending(0, 0);
        }

        if (response.totalCount() > response.checkSuites().size()) {
            int completedInPage = (int) response.checkSuites().stream()
                .filter(s -> "completed".equals(s.status())).count();
            LOG.warnf("Incomplete check-suites page: %d of %d returned for %s/%s@%s",
                response.checkSuites().size(), response.totalCount(), owner, repo, headSha);
            return new CombinedCiStatus.Pending(completedInPage, response.totalCount());
        }

        int completedCount = 0;
        int blockingCount = 0;
        for (var suite : response.checkSuites()) {
            if (!"completed".equals(suite.status())) {
                return new CombinedCiStatus.Pending(completedCount, response.totalCount());
            }
            completedCount++;
            if (!NON_BLOCKING.contains(suite.conclusion())) {
                blockingCount++;
            }
        }

        if (blockingCount > 0) {
            return new CombinedCiStatus.Failing(
                blockingCount + " of " + response.totalCount() + " suites failed");
        }
        return new CombinedCiStatus.Passing();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl github test -Dtest=GitHubCiStatusClientTest`
Expected: BUILD SUCCESS — all 11 tests pass.

- [ ] **Step 5: Commit**

```
git add github/src/main/java/io/casehub/devtown/github/GitHubCiStatusClient.java github/src/test/java/io/casehub/devtown/github/GitHubCiStatusClientTest.java
git commit -m "feat(github): implement GitHubCiStatusClient with conclusion classification

NON_BLOCKING: success, neutral, skipped. Pagination returns Pending.
API errors return Unavailable. 11 tests.

Refs casehubio/devtown#89"
```

---

## Task 4: Update `PrReviewCaseService` — tests first

**Files:**
- Modify: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

- [ ] **Step 1: Rewrite test class with `CiStatusClient` mock**

Replace the entire test class. The old tests depend on the in-memory map and local aggregation — those are gone.

```java
package io.casehub.devtown.app;

import io.casehub.devtown.app.mcp.PrReviewCaseTracker;
import io.casehub.devtown.domain.CiStatusClient;
import io.casehub.devtown.domain.CombinedCiStatus;
import io.casehub.devtown.review.LifecycleResult;
import io.casehub.devtown.review.PrPayload;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class PrReviewCaseServiceCiStatusTest {

    private PrReviewCaseTracker tracker;
    private PrReviewCaseHub caseHub;
    private CiStatusClient ciStatusClient;
    private PrReviewCaseService service;
    private UUID caseId;

    @BeforeEach
    void setUp() {
        tracker = new PrReviewCaseTracker();
        caseId = UUID.randomUUID();

        var payload = new PrPayload("casehubio/devtown", 42, "sha123", "main", 100, "octocat", List.of());
        tracker.register(caseId, "default", payload);

        caseHub = mock(PrReviewCaseHub.class);
        when(caseHub.signal(any(), any(), any())).thenReturn(CompletableFuture.completedFuture(null));
        when(caseHub.query(eq(caseId), eq("pr.headSha"), eq(String.class)))
            .thenReturn(CompletableFuture.completedFuture("sha123"));

        ciStatusClient = mock(CiStatusClient.class);

        service = new PrReviewCaseService();
        service.caseHub = caseHub;
        service.caseTracker = tracker;
        service.ciStatusClient = ciStatusClient;
        service.ciMode = "external";
    }

    @Test
    void signalCiStatus_noActiveCase_returnsNoActiveCase() {
        var result = service.signalCiStatus("casehubio/other", 99, "sha123", 1001, "success");
        assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
        verifyNoInteractions(ciStatusClient);
    }

    @Test
    void signalCiStatus_staleSha_returnsStaleEvent() {
        var result = service.signalCiStatus("casehubio/devtown", 42, "oldsha", 1001, "success");
        assertThat(result).isEqualTo(LifecycleResult.STALE_EVENT);
        verify(caseHub, never()).signal(eq(caseId), eq("ci.status"), any());
        verifyNoInteractions(ciStatusClient);
    }

    @Test
    void signalCiStatus_githubConfirmsPassing() {
        when(ciStatusClient.getCombinedStatus("casehubio", "devtown", "sha123"))
            .thenReturn(new CombinedCiStatus.Passing());

        var result = service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "success");

        assertThat(result).isEqualTo(LifecycleResult.UPDATED);
        verify(caseHub).signal(caseId, "ci.status", "passing");
    }

    @Test
    void signalCiStatus_githubSaysPending() {
        when(ciStatusClient.getCombinedStatus("casehubio", "devtown", "sha123"))
            .thenReturn(new CombinedCiStatus.Pending(1, 3));

        var result = service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "success");

        assertThat(result).isEqualTo(LifecycleResult.UPDATED);
        verify(caseHub, never()).signal(eq(caseId), eq("ci.status"), any());
    }

    @Test
    void signalCiStatus_githubConfirmsFailing() {
        when(ciStatusClient.getCombinedStatus("casehubio", "devtown", "sha123"))
            .thenReturn(new CombinedCiStatus.Failing("1 of 3 suites failed"));

        var result = service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "failure");

        assertThat(result).isEqualTo(LifecycleResult.UPDATED);
        verify(caseHub).signal(caseId, "ci.status", "failing");
    }

    @Test
    void signalCiStatus_githubUnavailable() {
        when(ciStatusClient.getCombinedStatus("casehubio", "devtown", "sha123"))
            .thenReturn(new CombinedCiStatus.Unavailable("api error: HTTP 403"));

        var result = service.signalCiStatus("casehubio/devtown", 42, "sha123", 1001, "success");

        assertThat(result).isEqualTo(LifecycleResult.UPDATED);
        verify(caseHub, never()).signal(eq(caseId), eq("ci.status"), any());
    }

    @Test
    void signalCiStatus_alwaysRecordsSuiteRegardlessOfCombinedStatus() {
        var variants = List.<CombinedCiStatus>of(
            new CombinedCiStatus.Passing(),
            new CombinedCiStatus.Pending(1, 3),
            new CombinedCiStatus.Failing("1 of 3 failed"),
            new CombinedCiStatus.Unavailable("network error")
        );

        long suiteId = 1001;
        for (var variant : variants) {
            setUp();
            when(ciStatusClient.getCombinedStatus("casehubio", "devtown", "sha123"))
                .thenReturn(variant);

            service.signalCiStatus("casehubio/devtown", 42, "sha123", suiteId, "success");

            verify(caseHub).signal(eq(caseId), eq("ci.suites." + suiteId), any(Map.class));
            suiteId++;
        }
    }

    @Test
    void signalCheckRun_noActiveCase_returnsNoActiveCase() {
        var result = service.signalCheckRun("casehubio/other", 99, "sha123", "lint", "success", java.time.Instant.now());
        assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
    }

    @Test
    void signalCheckRun_staleSha_returnsStaleEvent() {
        var result = service.signalCheckRun("casehubio/devtown", 42, "oldsha", "lint", "success", java.time.Instant.now());
        assertThat(result).isEqualTo(LifecycleResult.STALE_EVENT);
    }

    @Test
    void signalCheckRun_validSha_signalsCheckResult() {
        var completedAt = java.time.Instant.parse("2026-06-21T12:00:00Z");
        var result = service.signalCheckRun("casehubio/devtown", 42, "sha123", "lint", "success", completedAt);
        assertThat(result).isEqualTo(LifecycleResult.UPDATED);
        verify(caseHub).signal(eq(caseId), eq("ci.checks.lint"), eq(Map.of(
            "conclusion", "success",
            "completedAt", completedAt.toString()
        )));
    }

    @Test
    void signalCheckRun_doesNotChangeCiStatus() {
        service.signalCheckRun("casehubio/devtown", 42, "sha123", "lint", "failure", java.time.Instant.now());
        verify(caseHub, never()).signal(eq(caseId), eq("ci.status"), any());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=PrReviewCaseServiceCiStatusTest`
Expected: FAIL — `PrReviewCaseService` has no `ciStatusClient` field.

- [ ] **Step 3: Modify `PrReviewCaseService`**

Three changes to `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`:

**3a. Fix the `@Alternative @Priority(2)` anti-pattern (lines 21-23).**

Replace:
```java
@ApplicationScoped
@Alternative
@Priority(2)
```
With:
```java
@ApplicationScoped
```

Remove the now-unused imports for `jakarta.annotation.Priority` and `jakarta.enterprise.inject.Alternative`.

**3b. Replace the in-memory map with `CiStatusClient` injection.**

Remove the `ciSuiteResults` field (line 52):
```java
private final java.util.concurrent.ConcurrentHashMap<UUID, java.util.concurrent.ConcurrentHashMap<Long, String>> ciSuiteResults = new java.util.concurrent.ConcurrentHashMap<>();
```

Add import and field:
```java
import io.casehub.devtown.domain.CiStatusClient;
import io.casehub.devtown.domain.CombinedCiStatus;
```

```java
@Inject
CiStatusClient ciStatusClient;
```

**3c. Rewrite `signalCiStatus` method (lines 137-164).**

Replace:
```java
@Override
public LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion) {
    var active = caseTracker.findActiveCaseByPr(repo, prNumber);
    if (active.isEmpty()) return LifecycleResult.NO_ACTIVE_CASE;

    UUID caseId = active.get().caseId();

    String currentSha = caseHub.query(caseId, "pr.headSha", String.class).toCompletableFuture().join();
    if (!headSha.equals(currentSha)) return LifecycleResult.STALE_EVENT;

    var suites = ciSuiteResults.computeIfAbsent(caseId, k -> new java.util.concurrent.ConcurrentHashMap<>());
    suites.put(suiteId, conclusion);

    caseHub.signal(caseId, "ci.suites." + suiteId, Map.of(
        "conclusion", conclusion,
        "completedAt", java.time.Instant.now().toString()
    ));

    boolean anyFailed = suites.values().stream().anyMatch(c -> !"success".equals(c));
    boolean allSuccess = suites.values().stream().allMatch("success"::equals);

    if (anyFailed) {
        caseHub.signal(caseId, "ci.status", "failing");
    } else if (allSuccess) {
        caseHub.signal(caseId, "ci.status", "passing");
    }

    return LifecycleResult.UPDATED;
}
```

With:
```java
@Override
public LifecycleResult signalCiStatus(String repo, int prNumber, String headSha, long suiteId, String conclusion) {
    var active = caseTracker.findActiveCaseByPr(repo, prNumber);
    if (active.isEmpty()) return LifecycleResult.NO_ACTIVE_CASE;

    UUID caseId = active.get().caseId();

    String currentSha = caseHub.query(caseId, "pr.headSha", String.class).toCompletableFuture().join();
    if (!headSha.equals(currentSha)) return LifecycleResult.STALE_EVENT;

    caseHub.signal(caseId, "ci.suites." + suiteId, Map.of(
        "conclusion", conclusion,
        "completedAt", java.time.Instant.now().toString()
    ));

    String[] parts = repo.split("/");
    switch (ciStatusClient.getCombinedStatus(parts[0], parts[1], headSha)) {
        case CombinedCiStatus.Passing() -> caseHub.signal(caseId, "ci.status", "passing");
        case CombinedCiStatus.Failing(var summary) -> caseHub.signal(caseId, "ci.status", "failing");
        case CombinedCiStatus.Pending(var completed, var total) -> { }
        case CombinedCiStatus.Unavailable(var reason) -> { }
    }

    return LifecycleResult.UPDATED;
}
```

**3d. Remove `ciSuiteResults.remove(caseId)` from `revisePr()` (line 113).**

Delete the line:
```java
ciSuiteResults.remove(caseId);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=PrReviewCaseServiceCiStatusTest`
Expected: BUILD SUCCESS — all 12 tests pass.

- [ ] **Step 5: Run full build to verify nothing else broke**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceCiStatusTest.java
git commit -m "feat(app): replace in-memory CI aggregation with CiStatusClient SPI

Remove ciSuiteResults ConcurrentHashMap — GitHub is the single source of truth.
Fix @Alternative @Priority(2) anti-pattern (ARC42STORIES §8).
12 tests covering all CombinedCiStatus variants including audit trail invariant.

Closes casehubio/devtown#89"
```

---

## Self-Review

**Spec coverage:**
- ✅ `CiStatusClient` SPI in domain (Task 1)
- ✅ `CombinedCiStatus` sealed interface with four variants (Task 1)
- ✅ `GitHubChecksApi` REST client (Task 2)
- ✅ `CheckSuitesResponse` DTO (Task 2)
- ✅ `GitHubCiStatusClient` with conclusion classification, pagination, errors (Task 3)
- ✅ `PrReviewCaseService` removal of in-memory map (Task 4, step 3b/3d)
- ✅ `PrReviewCaseService` CiStatusClient injection and delegation (Task 4, step 3c)
- ✅ `@Alternative @Priority(2)` drive-by fix (Task 4, step 3a)
- ✅ NON_BLOCKING set: success, neutral, skipped (Task 3)
- ✅ Pagination conservative fallback (Task 3)
- ✅ per_page=100 (Task 2, Task 3)
- ✅ Unavailable variant for API errors (Task 1, Task 3)
- ✅ owner/repo separate parameters matching MergeClient (Task 1)
- ✅ Audit trail invariant test (Task 4 — `alwaysRecordsSuiteRegardlessOfCombinedStatus`)
- ✅ timed_out explicit test (Task 3 — `timedOut_isBlocking`)
- ✅ signalCheckRun tests preserved unchanged (Task 4)

**Placeholder scan:** No TBD/TODO/placeholders. All code blocks are complete.

**Type consistency:**
- `CombinedCiStatus` variants: Passing, Failing, Pending, Unavailable — consistent across Tasks 1, 3, 4
- `CiStatusClient.getCombinedStatus(String owner, String repo, String headSha)` — consistent across Tasks 1, 3, 4
- `GitHubChecksApi.listCheckSuites` takes `(owner, repo, ref, perPage)` — consistent between Task 2 and Task 3
- `CheckSuitesResponse.CheckSuite` record fields: `id`, `status`, `conclusion` — consistent between Task 2 and Task 3
- `NON_BLOCKING` set: `{"success", "neutral", "skipped"}` — consistent between spec and Task 3
