# Merge Execution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Execute PR merges via the GitHub API when the merge-executor binding fires, with tamper-evident audit trail recording the merge commit SHA.

**Architecture:** Domain-level `MergeClient` port + `MergeOutcome` sealed result in `domain/`. `GitHubMergeClient` adapter in `github/` calls the merge API. `PrReviewCaseHub` in `app/` registers a Worker in `@PostConstruct` that adapts `MergeClient → WorkerFunction.Sync`. YAML changes add the `merge-completed` goal, binding guard, outcomePolicy, and conflictResolverStrategy.

**Tech Stack:** Java 21, Quarkus 3.32.2, Quarkus REST Client, Jackson, Mockito, AssertJ

**Spec:** `specs/2026-06-21-merge-execution-design.md` (rev 4)

**Build:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml`

---

## File Structure

| File | Module | Action | Responsibility |
|------|--------|--------|---------------|
| `domain/src/main/java/io/casehub/devtown/domain/MergeOutcome.java` | domain | Create | Sealed interface: Success(mergeSha) \| Failure(reason) |
| `domain/src/main/java/io/casehub/devtown/domain/MergeClient.java` | domain | Create | Port interface: MergeOutcome merge(owner, repo, prNumber, headSha) |
| `github/src/main/java/io/casehub/devtown/github/GitHubMergeClient.java` | github | Create | REST client implementing MergeClient |
| `github/src/main/java/io/casehub/devtown/github/GitHubMergeApi.java` | github | Create | @RegisterRestClient interface for GitHub merge endpoint |
| `app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java` | app | Modify | @PostConstruct Worker registration + adaptMerge |
| `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java` | app | Modify | Prefer merge_sha over pr.headSha |
| `review/src/main/resources/devtown/pr-review.yaml` | review | Modify | YAML changes (4 sections) |
| `app/src/main/resources/application.properties` | app | Modify | Add merge config |
| `domain/src/test/java/io/casehub/devtown/domain/MergeOutcomeTest.java` | domain | Create | Sealed interface tests |
| `github/src/test/java/io/casehub/devtown/github/GitHubMergeClientTest.java` | github | Create | Mock REST client tests |
| `app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubMergeTest.java` | app | Create | Worker registration + adaptMerge tests |
| `app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java` | app | Modify | Add merge_sha preference test |

All paths relative to `/Users/mdproctor/claude/casehub/devtown/`.

---

### Task 1: Domain types — MergeOutcome + MergeClient

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/MergeOutcome.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/MergeClient.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/MergeOutcomeTest.java`

- [ ] **Step 1: Write failing tests for MergeOutcome**

Create `domain/src/test/java/io/casehub/devtown/domain/MergeOutcomeTest.java`:

```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class MergeOutcomeTest {

    @Test
    void success_carriesMergeSha() {
        var outcome = new MergeOutcome.Success("abc123def");
        assertThat(outcome.mergeSha()).isEqualTo("abc123def");
        assertThat(outcome).isInstanceOf(MergeOutcome.class);
    }

    @Test
    void failure_carriesReason() {
        var outcome = new MergeOutcome.Failure("merge conflict");
        assertThat(outcome.reason()).isEqualTo("merge conflict");
        assertThat(outcome).isInstanceOf(MergeOutcome.class);
    }

    @Test
    void switchExpression_exhaustive() {
        MergeOutcome outcome = new MergeOutcome.Success("sha");
        String result = switch (outcome) {
            case MergeOutcome.Success s -> s.mergeSha();
            case MergeOutcome.Failure f -> f.reason();
        };
        assertThat(result).isEqualTo("sha");
    }
}
```

- [ ] **Step 2: Run tests — verify FAIL**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl domain -Dtest="io.casehub.devtown.domain.MergeOutcomeTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — classes don't exist.

- [ ] **Step 3: Create MergeOutcome sealed interface**

Create `domain/src/main/java/io/casehub/devtown/domain/MergeOutcome.java`:

```java
package io.casehub.devtown.domain;

public sealed interface MergeOutcome {
    record Success(String mergeSha) implements MergeOutcome {}
    record Failure(String reason) implements MergeOutcome {}
}
```

- [ ] **Step 4: Create MergeClient port interface**

Create `domain/src/main/java/io/casehub/devtown/domain/MergeClient.java`:

```java
package io.casehub.devtown.domain;

public interface MergeClient {
    MergeOutcome merge(String owner, String repo, int prNumber, String headSha);
}
```

- [ ] **Step 5: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl domain -Dtest="io.casehub.devtown.domain.MergeOutcomeTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All 3 tests pass.

- [ ] **Step 6: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/MergeOutcome.java domain/src/main/java/io/casehub/devtown/domain/MergeClient.java domain/src/test/java/io/casehub/devtown/domain/MergeOutcomeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(domain): MergeClient port + MergeOutcome sealed interface

Pure-Java domain types with no engine dependency. Success carries
merge commit SHA, Failure carries reason. Sealed for exhaustive switch.

Refs #87"
```

---

### Task 2: GitHubMergeClient — REST client adapter

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubMergeApi.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubMergeClient.java`
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubMergeClientTest.java`

- [ ] **Step 1: Write failing tests for GitHubMergeClient**

Create `github/src/test/java/io/casehub/devtown/github/GitHubMergeClientTest.java`:

```java
package io.casehub.devtown.github;

import io.casehub.devtown.domain.MergeOutcome;
import jakarta.ws.rs.WebApplicationException;
import jakarta.ws.rs.core.Response;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class GitHubMergeClientTest {

    private GitHubMergeApi api;
    private GitHubMergeClient client;

    @BeforeEach
    void setUp() {
        api = mock(GitHubMergeApi.class);
        client = new GitHubMergeClient(api, "squash");
    }

    @Test
    void merge_success_returnsSuccessWithSha() {
        when(api.merge(eq("casehubio"), eq("devtown"), eq(42), any()))
            .thenReturn(Map.of("sha", "merge-sha-abc"));

        var result = client.merge("casehubio", "devtown", 42, "head-sha-123");

        assertThat(result).isInstanceOf(MergeOutcome.Success.class);
        assertThat(((MergeOutcome.Success) result).mergeSha()).isEqualTo("merge-sha-abc");
    }

    @Test
    void merge_success_passesMethodAndSha() {
        when(api.merge(anyString(), anyString(), anyInt(), any()))
            .thenReturn(Map.of("sha", "abc"));

        client.merge("casehubio", "devtown", 42, "head-sha-123");

        verify(api).merge(eq("casehubio"), eq("devtown"), eq(42), argThat(body ->
            "squash".equals(body.get("merge_method")) &&
            "head-sha-123".equals(body.get("sha"))
        ));
    }

    @Test
    void merge_409_returnsConflictFailure() {
        when(api.merge(anyString(), anyString(), anyInt(), any()))
            .thenThrow(new WebApplicationException(Response.status(409).build()));

        var result = client.merge("casehubio", "devtown", 42, "sha");

        assertThat(result).isInstanceOf(MergeOutcome.Failure.class);
        assertThat(((MergeOutcome.Failure) result).reason()).contains("conflict");
    }

    @Test
    void merge_405_returnsBranchProtectionFailure() {
        when(api.merge(anyString(), anyString(), anyInt(), any()))
            .thenThrow(new WebApplicationException(Response.status(405).build()));

        var result = client.merge("casehubio", "devtown", 42, "sha");

        assertThat(result).isInstanceOf(MergeOutcome.Failure.class);
        assertThat(((MergeOutcome.Failure) result).reason()).contains("branch protection");
    }

    @Test
    void merge_422_returnsNotMergeableFailure() {
        when(api.merge(anyString(), anyString(), anyInt(), any()))
            .thenThrow(new WebApplicationException(Response.status(422).build()));

        var result = client.merge("casehubio", "devtown", 42, "sha");

        assertThat(result).isInstanceOf(MergeOutcome.Failure.class);
        assertThat(((MergeOutcome.Failure) result).reason()).contains("not mergeable");
    }

    @Test
    void merge_runtimeException_returnsApiErrorFailure() {
        when(api.merge(anyString(), anyString(), anyInt(), any()))
            .thenThrow(new RuntimeException("connection reset"));

        var result = client.merge("casehubio", "devtown", 42, "sha");

        assertThat(result).isInstanceOf(MergeOutcome.Failure.class);
        assertThat(((MergeOutcome.Failure) result).reason()).contains("api error");
    }

    @Test
    void merge_customMethod_passesConfiguredMethod() {
        var rebaseClient = new GitHubMergeClient(api, "rebase");
        when(api.merge(anyString(), anyString(), anyInt(), any()))
            .thenReturn(Map.of("sha", "abc"));

        rebaseClient.merge("casehubio", "devtown", 42, "sha");

        verify(api).merge(anyString(), anyString(), anyInt(), argThat(body ->
            "rebase".equals(body.get("merge_method"))
        ));
    }
}
```

- [ ] **Step 2: Run tests — verify FAIL**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dtest="io.casehub.devtown.github.GitHubMergeClientTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — classes don't exist.

- [ ] **Step 3: Create GitHubMergeApi REST client interface**

Create `github/src/main/java/io/casehub/devtown/github/GitHubMergeApi.java`:

```java
package io.casehub.devtown.github;

import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.HeaderParam;
import jakarta.ws.rs.PUT;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import java.util.Map;

@RegisterRestClient(configKey = "github-api")
@Path("/repos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface GitHubMergeApi {

    @PUT
    @Path("/{owner}/{repo}/pulls/{pull_number}/merge")
    Map<String, Object> merge(
        @PathParam("owner") String owner,
        @PathParam("repo") String repo,
        @PathParam("pull_number") int pullNumber,
        Map<String, Object> body
    );
}
```

- [ ] **Step 4: Create GitHubMergeClient**

Create `github/src/main/java/io/casehub/devtown/github/GitHubMergeClient.java`:

```java
package io.casehub.devtown.github;

import io.casehub.devtown.domain.MergeClient;
import io.casehub.devtown.domain.MergeOutcome;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.ws.rs.WebApplicationException;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.rest.client.inject.RestClient;
import java.util.Map;

@ApplicationScoped
public class GitHubMergeClient implements MergeClient {

    private final GitHubMergeApi api;
    private final String mergeMethod;

    public GitHubMergeClient(@RestClient GitHubMergeApi api,
                              @ConfigProperty(name = "devtown.github.merge-method", defaultValue = "squash") String mergeMethod) {
        this.api = api;
        this.mergeMethod = mergeMethod;
    }

    @Override
    public MergeOutcome merge(String owner, String repo, int prNumber, String headSha) {
        try {
            var body = Map.<String, Object>of(
                "merge_method", mergeMethod,
                "sha", headSha
            );
            var response = api.merge(owner, repo, prNumber, body);
            String mergeSha = (String) response.get("sha");
            return new MergeOutcome.Success(mergeSha);
        } catch (WebApplicationException e) {
            int status = e.getResponse().getStatus();
            return switch (status) {
                case 409 -> new MergeOutcome.Failure("merge conflict or SHA mismatch");
                case 405 -> new MergeOutcome.Failure("branch protection prevents merge");
                case 422 -> new MergeOutcome.Failure("PR not in mergeable state");
                default -> new MergeOutcome.Failure("api error: HTTP " + status);
            };
        } catch (Exception e) {
            return new MergeOutcome.Failure("api error: " + e.getMessage());
        }
    }
}
```

- [ ] **Step 5: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dtest="io.casehub.devtown.github.GitHubMergeClientTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All 7 tests pass.

- [ ] **Step 6: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add github/src/main/java/io/casehub/devtown/github/GitHubMergeApi.java github/src/main/java/io/casehub/devtown/github/GitHubMergeClient.java github/src/test/java/io/casehub/devtown/github/GitHubMergeClientTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(github): GitHubMergeClient — REST client adapter for merge API

Implements MergeClient port. Quarkus REST Client for outbound call.
Maps HTTP status codes to MergeOutcome variants. Configurable merge
method (squash default).

Refs #87"
```

---

### Task 3: YAML changes — outputSchema, binding, goal, completion

**Files:**
- Modify: `review/src/main/resources/devtown/pr-review.yaml`

- [ ] **Step 1: Remove outputSchema from merge-executor capability**

In `pr-review.yaml`, replace (around line 46-49):

```yaml
    - name: merge-executor
      description: "Executes the merge when all goals are satisfied"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{}"
```

With:

```yaml
    - name: merge-executor
      description: "Executes the merge when all goals are satisfied"
      inputSchema: "{ pr: .pr }"
```

- [ ] **Step 2: Add merge-completed goal**

After the `review-abandoned` goal (around line 90), add:

```yaml
    - name: merge-completed
      kind: success
      condition: '.pr.status == "merged" or .merge_sha != null'
```

- [ ] **Step 3: Update completion to include merge-completed**

Replace the completion section (around line 92-102):

```yaml
  completion:
    success:
      allOf:
        - pr-approved
        - security-verified
        - ci-passing
        - merge-completed
    failure:
      anyOf:
        - review-blocked
        - review-rejected
        - review-abandoned
```

- [ ] **Step 4: Update merge binding**

Replace the merge binding (around line 328-341):

```yaml
    ## ── Merge — fires when all success conditions satisfied ──

    - name: merge
      on: { contextChange: {} }
      when: >-
        .merge_sha == null and
        .pr.status != "merged" and
        (.codeAnalysis.securitySensitive == false or .securityReview.outcome == "APPROVED") and
        (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
        .styleCheck.outcome == "APPROVED" and
        .testCoverage.outcome == "APPROVED" and
        .performanceAnalysis.outcome == "APPROVED" and
        (.pr.linesChanged <= .policy.humanApprovalThreshold or .humanApproval.outcome == "APPROVED") and
        .ci.status == "passing"
      capability: merge-executor
      conflictResolverStrategy: DEEP_MERGE
      outcomePolicy:
        onDecline: FAULT
        onFailure: FAULT
        onExpired: FAULT
```

- [ ] **Step 5: Compile review module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml compile -pl review --batch-mode`

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add review/src/main/resources/devtown/pr-review.yaml
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(review): YAML — merge-completed goal, binding guard, FAULT outcomePolicy

Remove merge-executor outputSchema (JQ {} discards output).
Add .merge_sha == null guard on merge binding.
Add outcomePolicy: FAULT on all failure modes (not REROUTE×3 default).
Add conflictResolverStrategy: DEEP_MERGE for consistency.
Add merge-completed goal to prevent premature case completion.
Update completion allOf to include merge-completed.

Refs #87"
```

---

### Task 4: PrReviewCaseHub — @PostConstruct Worker registration

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java`
- Create: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubMergeTest.java`

- [ ] **Step 1: Write failing tests for adaptMerge**

Create `app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubMergeTest.java`:

```java
package io.casehub.devtown.app;

import io.casehub.api.model.WorkerResult;
import io.casehub.devtown.domain.MergeClient;
import io.casehub.devtown.domain.MergeOutcome;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class PrReviewCaseHubMergeTest {

    private MergeClient mergeClient;
    private PrReviewCaseHub hub;

    @BeforeEach
    void setUp() {
        mergeClient = mock(MergeClient.class);
        hub = new PrReviewCaseHub();
        hub.mergeClient = mergeClient;
    }

    private Map<String, Object> prInput(String repo, String id, String headSha) {
        return Map.of("pr", Map.of("repo", repo, "id", id, "headSha", headSha));
    }

    @Test
    void adaptMerge_success_returnsWorkerResultWithMergeSha() {
        when(mergeClient.merge("casehubio", "devtown", 42, "sha123"))
            .thenReturn(new MergeOutcome.Success("merge-sha-abc"));

        WorkerResult result = hub.adaptMerge(prInput("casehubio/devtown", "42", "sha123"));

        assertThat(result.output()).containsEntry("merge_sha", "merge-sha-abc");
        assertThat(result.outcome()).isInstanceOf(io.casehub.api.model.WorkerOutcome.Success.class);
    }

    @Test
    void adaptMerge_failure_returnsFailedWorkerResult() {
        when(mergeClient.merge("casehubio", "devtown", 42, "sha123"))
            .thenReturn(new MergeOutcome.Failure("merge conflict"));

        WorkerResult result = hub.adaptMerge(prInput("casehubio/devtown", "42", "sha123"));

        assertThat(result.outcome()).isInstanceOf(io.casehub.api.model.WorkerOutcome.Failed.class);
    }

    @Test
    void adaptMerge_parsesRepoIntoOwnerAndName() {
        when(mergeClient.merge(anyString(), anyString(), anyInt(), anyString()))
            .thenReturn(new MergeOutcome.Success("sha"));

        hub.adaptMerge(prInput("myorg/myrepo", "7", "headsha"));

        verify(mergeClient).merge("myorg", "myrepo", 7, "headsha");
    }
}
```

- [ ] **Step 2: Run tests — verify FAIL**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseHubMergeTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: FAIL — `adaptMerge` method and `mergeClient` field don't exist.

- [ ] **Step 3: Implement PrReviewCaseHub with Worker registration**

Replace `app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java`:

```java
package io.casehub.devtown.app;

import io.casehub.api.engine.YamlCaseHub;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.Worker;
import io.casehub.api.model.WorkerResult;
import io.casehub.devtown.domain.MergeClient;
import io.casehub.devtown.domain.MergeOutcome;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class PrReviewCaseHub extends YamlCaseHub {

    @Inject
    MergeClient mergeClient;

    public PrReviewCaseHub() {
        super("devtown/pr-review.yaml");
    }

    @PostConstruct
    void registerWorkers() {
        CaseDefinition def = super.getDefinition();
        var mergeCap = def.getCapabilities().stream()
            .filter(c -> "merge-executor".equals(c.getName()))
            .findFirst().orElseThrow();
        def.getWorkers().add(Worker.builder()
            .name("merge-executor")
            .capabilities(mergeCap)
            .function(this::adaptMerge)
            .build());
    }

    WorkerResult adaptMerge(Map<String, Object> input) {
        @SuppressWarnings("unchecked")
        Map<String, Object> pr = (Map<String, Object>) input.get("pr");
        String repo = (String) pr.get("repo");
        String[] parts = repo.split("/");
        int prNumber = Integer.parseInt((String) pr.get("id"));
        String headSha = (String) pr.get("headSha");

        return switch (mergeClient.merge(parts[0], parts[1], prNumber, headSha)) {
            case MergeOutcome.Success s -> WorkerResult.of(Map.of("merge_sha", s.mergeSha()));
            case MergeOutcome.Failure f -> WorkerResult.failed(f.reason());
        };
    }
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.PrReviewCaseHubMergeTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All 3 tests pass.

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/PrReviewCaseHub.java app/src/test/java/io/casehub/devtown/app/PrReviewCaseHubMergeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): PrReviewCaseHub — @PostConstruct Worker registration for merge-executor

Registers merge-executor Worker with WorkerFunction.Sync in CaseDefinition.
adaptMerge adapts MergeClient (domain port) to WorkerResult (engine type).
Input extraction (repo parsing, id→int) is adapter logic in app/.

Refs #87"
```

---

### Task 5: MergeDecisionObserver — prefer merge_sha

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java`

- [ ] **Step 1: Write failing test for merge_sha preference**

Add to `app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java`. First read the existing test to understand setup patterns — it uses `@QuarkusTest` with `LedgerEnabledTestProfile`. Add:

```java
@Test
void completedCase_prefersContextMergeSha_overHeadSha() {
    // Seed PR payload via startReview, then signal merge_sha
    var pr = new PrPayload("casehubio/devtown", prNumber, "head-sha-aaa", "main", 50, "tester", List.of());
    reviewService.startReview(pr);

    // Signal the merge SHA to the case context
    var active = caseTracker.findActiveCaseByPr("casehubio/devtown", prNumber);
    assertThat(active).isPresent();
    UUID caseId = active.get().caseId();
    caseHub.signal(caseId, "merge_sha", "merge-sha-bbb");

    // Signal case completion
    caseHub.signal(caseId, "pr.status", "merged");

    // Wait for async observer
    // ... (existing test pattern)

    // Verify the ledger entry uses merge_sha, not pr.headSha
    var entries = findMergeDecisions(caseId, tenancyId);
    assertThat(entries).isNotEmpty();
    assertThat(entries.get(0).commitSha).isEqualTo("merge-sha-bbb");
}
```

Note: The exact test setup depends on the existing test patterns. Read `MergeDecisionObserverTest.java` to match the pattern. If the existing tests don't support signaling merge_sha, a unit-level approach (mocking CaseContext) may be needed instead.

- [ ] **Step 2: Implement merge_sha preference**

In `MergeDecisionObserver.java`, change line 75 area from:

```java
String headSha = ctx.getPathAsString("pr.headSha");
```

To:

```java
String headSha = ctx.getPathAsString("pr.headSha");
String mergeSha = ctx.getPathAsString("merge_sha");
```

And change line 92 area from:

```java
entry.commitSha = headSha;
```

To:

```java
entry.commitSha = mergeSha != null ? mergeSha : headSha;
```

- [ ] **Step 3: Run MergeDecisionObserver tests — verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dtest="io.casehub.devtown.app.ledger.MergeDecisionObserverTest" -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All tests pass (existing + new).

- [ ] **Step 4: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/ledger/MergeDecisionObserver.java app/src/test/java/io/casehub/devtown/app/ledger/MergeDecisionObserverTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "fix(app): MergeDecisionObserver — prefer merge_sha over pr.headSha

Records the actual merge commit SHA (from GitHub's merge API response)
instead of the PR's HEAD SHA. Falls back to pr.headSha for REJECTED
decisions where no merge occurred.

Refs #87"
```

---

### Task 6: Config + full integration test

**Files:**
- Modify: `app/src/main/resources/application.properties`

- [ ] **Step 1: Add merge config to application.properties**

After the existing `devtown.github.webhook-secret` line, add:

```properties
devtown.github.token=${GITHUB_TOKEN:}
devtown.github.merge-method=squash
```

Also add the REST client base URL config:

```properties
quarkus.rest-client.github-api.url=https://api.github.com
```

- [ ] **Step 2: Run full app test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl app -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: Same 5 pre-existing failures from #88. No new failures.

- [ ] **Step 3: Run full github module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl github -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All tests pass.

- [ ] **Step 4: Run full domain module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/devtown/pom.xml test -pl domain -Dsurefire.rerunFailingTestsCount=0 --batch-mode`

Expected: All tests pass.

- [ ] **Step 5: Commit**

```
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): merge execution config — token, method, REST client base URL

devtown.github.token for merge API auth.
devtown.github.merge-method=squash (default).
github-api REST client base URL for Quarkus REST Client.

Refs #87"
```
