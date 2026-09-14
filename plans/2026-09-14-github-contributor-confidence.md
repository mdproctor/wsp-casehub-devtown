# GitHub-Derived Contributor Confidence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #202 — GitHub-derived contributor confidence — bootstrap trust from observable PR history
**Issue group:** #202

**Goal:** Bootstrap contributor trust from GitHub PR history so contributors with existing track records don't start at zero in devtown.

**Architecture:** Hybrid TrustBootstrapSource integration — eager import via `TrustImportService` for immediate score availability on first PR, plus pull SPI (`TrustBootstrapSource`) as fallback for the trust job. Cache per-contributor per-repo signal data with adaptive TTL. Async bootstrap triggered from PR intake, classification defaults to TRIAGE until data arrives.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Flyway, MicroProfile REST Client, CDI events

## Global Constraints

- Java 21 source on Java 26 JVM
- Quarkus 3.32.2
- All casehubio SNAPSHOTs at 0.2-SNAPSHOT
- H2 MODE=PostgreSQL for tests, PostgreSQL for production
- Flyway V2–V999 for devtown domain migrations
- Identity scheme: `"github-id:" + numericId` (must match existing attestation pipeline)
- `domain/` = pure Java (no Quarkus deps); `review/` = integration logic; `github/` = REST adapters; `app/` = CDI wiring + JPA
- IntelliJ MCP for all code navigation and structural editing

---

## Batch 1: Domain Foundation

Domain types in `domain/` — pure Java, no framework deps. These are the building blocks everything else depends on.

### Task 1: CacheMaturity enum and RepoConfidenceTier enum

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/CacheMaturity.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/RepoConfidenceTier.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/trust/CacheMaturityTest.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/trust/RepoConfidenceTierTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `CacheMaturity.isStale(Instant lastRefresh, Instant now) → boolean`, `CacheMaturity.forObservationCount(int count) → CacheMaturity`, `RepoConfidenceTier.confidenceMultiplier() → double`, `RepoConfidenceTier.classify(int contributorCount, int pullRequestCount, Instant createdAt) → RepoConfidenceTier`

- [ ] **Step 1: Write failing tests for CacheMaturity**

```java
package io.casehub.devtown.domain.trust;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class CacheMaturityTest {

    private static final Instant NOW = Instant.parse("2026-09-14T12:00:00Z");

    @Test
    void forObservationCount_cold() {
        assertEquals(CacheMaturity.COLD, CacheMaturity.forObservationCount(0));
        assertEquals(CacheMaturity.COLD, CacheMaturity.forObservationCount(5));
    }

    @Test
    void forObservationCount_warm() {
        assertEquals(CacheMaturity.WARM, CacheMaturity.forObservationCount(6));
        assertEquals(CacheMaturity.WARM, CacheMaturity.forObservationCount(20));
    }

    @Test
    void forObservationCount_hot() {
        assertEquals(CacheMaturity.HOT, CacheMaturity.forObservationCount(21));
        assertEquals(CacheMaturity.HOT, CacheMaturity.forObservationCount(50));
    }

    @Test
    void forObservationCount_mature() {
        assertEquals(CacheMaturity.MATURE, CacheMaturity.forObservationCount(51));
        assertEquals(CacheMaturity.MATURE, CacheMaturity.forObservationCount(500));
    }

    @Test
    void cold_alwaysStale() {
        assertTrue(CacheMaturity.COLD.isStale(NOW.minus(Duration.ofSeconds(1)), NOW));
        assertTrue(CacheMaturity.COLD.isStale(NOW, NOW));
    }

    @Test
    void warm_staleAfterOneDay() {
        assertFalse(CacheMaturity.WARM.isStale(NOW.minus(Duration.ofHours(23)), NOW));
        assertTrue(CacheMaturity.WARM.isStale(NOW.minus(Duration.ofHours(25)), NOW));
    }

    @Test
    void hot_staleAfterThreeDays() {
        assertFalse(CacheMaturity.HOT.isStale(NOW.minus(Duration.ofDays(2)), NOW));
        assertTrue(CacheMaturity.HOT.isStale(NOW.minus(Duration.ofDays(4)), NOW));
    }

    @Test
    void mature_staleAfterSevenDays() {
        assertFalse(CacheMaturity.MATURE.isStale(NOW.minus(Duration.ofDays(6)), NOW));
        assertTrue(CacheMaturity.MATURE.isStale(NOW.minus(Duration.ofDays(8)), NOW));
    }

    @Test
    void nullLastRefresh_alwaysStale() {
        assertTrue(CacheMaturity.MATURE.isStale(null, NOW));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=CacheMaturityTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Implement CacheMaturity**

```java
package io.casehub.devtown.domain.trust;

import java.time.Duration;
import java.time.Instant;

public enum CacheMaturity {
    COLD(0, 5, Duration.ZERO),
    WARM(6, 20, Duration.ofDays(1)),
    HOT(21, 50, Duration.ofDays(3)),
    MATURE(51, Integer.MAX_VALUE, Duration.ofDays(7));

    private final int minObservations;
    private final int maxObservations;
    private final Duration staleDuration;

    CacheMaturity(int minObservations, int maxObservations, Duration staleDuration) {
        this.minObservations = minObservations;
        this.maxObservations = maxObservations;
        this.staleDuration = staleDuration;
    }

    public boolean isStale(Instant lastRefresh, Instant now) {
        if (lastRefresh == null) return true;
        if (this == COLD) return true;
        return Duration.between(lastRefresh, now).compareTo(staleDuration) > 0;
    }

    public static CacheMaturity forObservationCount(int count) {
        for (CacheMaturity m : values()) {
            if (count >= m.minObservations && count <= m.maxObservations) return m;
        }
        return MATURE;
    }
}
```

- [ ] **Step 4: Run CacheMaturity tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=CacheMaturityTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 5: Write failing tests for RepoConfidenceTier**

```java
package io.casehub.devtown.domain.trust;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import static org.junit.jupiter.api.Assertions.*;

class RepoConfidenceTierTest {

    private static final Instant NOW = Instant.parse("2026-09-14T12:00:00Z");

    @Test
    void multipliers() {
        assertEquals(0.8, RepoConfidenceTier.HIGH.confidenceMultiplier());
        assertEquals(0.5, RepoConfidenceTier.MEDIUM.confidenceMultiplier());
        assertEquals(0.3, RepoConfidenceTier.LOW.confidenceMultiplier());
    }

    @Test
    void classify_medium_meetsAllThresholds() {
        Instant twoYearsAgo = NOW.minus(730, ChronoUnit.DAYS);
        assertEquals(RepoConfidenceTier.MEDIUM,
            RepoConfidenceTier.classify(10, 100, twoYearsAgo, NOW));
    }

    @Test
    void classify_low_tooFewContributors() {
        Instant twoYearsAgo = NOW.minus(730, ChronoUnit.DAYS);
        assertEquals(RepoConfidenceTier.LOW,
            RepoConfidenceTier.classify(3, 100, twoYearsAgo, NOW));
    }

    @Test
    void classify_low_tooYoung() {
        Instant sixMonthsAgo = NOW.minus(180, ChronoUnit.DAYS);
        assertEquals(RepoConfidenceTier.LOW,
            RepoConfidenceTier.classify(10, 100, sixMonthsAgo, NOW));
    }

    @Test
    void classify_low_tooFewPrs() {
        Instant twoYearsAgo = NOW.minus(730, ChronoUnit.DAYS);
        assertEquals(RepoConfidenceTier.LOW,
            RepoConfidenceTier.classify(10, 30, twoYearsAgo, NOW));
    }

    @Test
    void classify_boundary_exactThresholds() {
        Instant exactlyOneYearAgo = NOW.minus(365, ChronoUnit.DAYS);
        assertEquals(RepoConfidenceTier.MEDIUM,
            RepoConfidenceTier.classify(5, 50, exactlyOneYearAgo, NOW));
    }
}
```

- [ ] **Step 6: Implement RepoConfidenceTier**

```java
package io.casehub.devtown.domain.trust;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

public enum RepoConfidenceTier {
    HIGH(0.8),
    MEDIUM(0.5),
    LOW(0.3);

    private final double confidenceMultiplier;

    RepoConfidenceTier(double confidenceMultiplier) {
        this.confidenceMultiplier = confidenceMultiplier;
    }

    public double confidenceMultiplier() {
        return confidenceMultiplier;
    }

    public static RepoConfidenceTier classify(int contributorCount, int pullRequestCount,
                                               Instant createdAt, Instant now) {
        long ageInDays = ChronoUnit.DAYS.between(createdAt, now);
        if (contributorCount >= 5 && pullRequestCount >= 50 && ageInDays >= 365) {
            return MEDIUM;
        }
        return LOW;
    }
}
```

- [ ] **Step 7: Run RepoConfidenceTier tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=RepoConfidenceTierTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/trust/CacheMaturity.java domain/src/main/java/io/casehub/devtown/domain/trust/RepoConfidenceTier.java domain/src/test/java/io/casehub/devtown/domain/trust/CacheMaturityTest.java domain/src/test/java/io/casehub/devtown/domain/trust/RepoConfidenceTierTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add CacheMaturity and RepoConfidenceTier domain types Refs #202"
```

### Task 2: Domain records, SPI interface, and BootstrapScoreComputer

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/ContributorGitHubProfile.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/RepoConfidenceProfile.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/ContributorHistoryClient.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/ContributorHistorySnapshot.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/RepoMetadataSnapshot.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/trust/BootstrapScoreComputer.java`
- Test: `domain/src/test/java/io/casehub/devtown/domain/trust/BootstrapScoreComputerTest.java`

**Interfaces:**
- Consumes: `CacheMaturity`, `RepoConfidenceTier` (Task 1), `ContributorTrustDimension.MERGE_RATE`, `ContributorTrustCapability.PR_CONTRIBUTION`
- Produces: `ContributorHistoryClient.fetchHistory(String login, String repo, Instant since) → ContributorHistorySnapshot`, `ContributorHistoryClient.fetchRepoMetadata(String owner, String repo) → RepoMetadataSnapshot`, `BootstrapScoreComputer.compute(ContributorGitHubProfile profile, RepoConfidenceTier tier) → BootstrapScoreResult`, `ContributorGitHubProfile.actorIdFromGitHubNumericId(long numericId) → String`

- [ ] **Step 1: Write failing tests for BootstrapScoreComputer**

```java
package io.casehub.devtown.domain.trust;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class BootstrapScoreComputerTest {

    @Test
    void compute_mergedOnly_highTier() {
        var profile = new ContributorGitHubProfile(
            "alice", 123L, "org/repo", "github-id:123",
            10, 0, 10, Instant.now(), CacheMaturity.WARM);

        var result = BootstrapScoreComputer.compute(profile, RepoConfidenceTier.HIGH);

        assertEquals(8.0, result.alpha(), 0.001);   // 10 * 0.8
        assertEquals(0.0, result.beta(), 0.001);     // 0 * 0.8
        assertEquals(1.0, result.mergeRate(), 0.001); // 10/10
        assertEquals(10, result.observationCount());
    }

    @Test
    void compute_mixedOutcomes_mediumTier() {
        var profile = new ContributorGitHubProfile(
            "bob", 456L, "org/repo", "github-id:456",
            7, 3, 10, Instant.now(), CacheMaturity.WARM);

        var result = BootstrapScoreComputer.compute(profile, RepoConfidenceTier.MEDIUM);

        assertEquals(3.5, result.alpha(), 0.001);    // 7 * 0.5
        assertEquals(1.5, result.beta(), 0.001);     // 3 * 0.5
        assertEquals(0.7, result.mergeRate(), 0.001); // 7/10
    }

    @Test
    void compute_zeroObservations_uninformativePrior() {
        var profile = new ContributorGitHubProfile(
            "new", 789L, "org/repo", "github-id:789",
            0, 0, 0, Instant.now(), CacheMaturity.COLD);

        var result = BootstrapScoreComputer.compute(profile, RepoConfidenceTier.LOW);

        assertEquals(0.0, result.alpha(), 0.001);
        assertEquals(0.0, result.beta(), 0.001);
        assertEquals(0.5, result.mergeRate(), 0.001); // uninformative prior
        assertEquals(0, result.observationCount());
        assertTrue(result.isEmpty());
    }

    @Test
    void compute_lowTier_dampensSignificantly() {
        var profile = new ContributorGitHubProfile(
            "eve", 101L, "personal/repo", "github-id:101",
            20, 5, 25, Instant.now(), CacheMaturity.HOT);

        var result = BootstrapScoreComputer.compute(profile, RepoConfidenceTier.LOW);

        assertEquals(6.0, result.alpha(), 0.001);  // 20 * 0.3
        assertEquals(1.5, result.beta(), 0.001);   // 5 * 0.3
    }

    @Test
    void actorId_deterministicFromNumericId() {
        assertEquals("github-id:42", ContributorGitHubProfile.actorIdFromGitHubNumericId(42));
        assertEquals("github-id:123456", ContributorGitHubProfile.actorIdFromGitHubNumericId(123456));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=BootstrapScoreComputerTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 3: Create domain records and SPI**

`ContributorGitHubProfile.java`:
```java
package io.casehub.devtown.domain.trust;

import java.time.Instant;

public record ContributorGitHubProfile(
    String login,
    long contributorNumericId,
    String repo,
    String actorId,
    int mergedCount,
    int closedCount,
    int observationCount,
    Instant lastRefreshAt,
    CacheMaturity maturity
) {
    public static String actorIdFromGitHubNumericId(long numericId) {
        return "github-id:" + numericId;
    }
}
```

`RepoConfidenceProfile.java`:
```java
package io.casehub.devtown.domain.trust;

import java.time.Instant;

public record RepoConfidenceProfile(
    String repo,
    int starCount,
    int contributorCount,
    int pullRequestCount,
    Instant createdAt,
    Instant lastPushedAt,
    RepoConfidenceTier tier,
    boolean adminOverride,
    Instant lastRefreshAt
) {}
```

`ContributorHistorySnapshot.java`:
```java
package io.casehub.devtown.domain.trust;

import java.time.Instant;

public record ContributorHistorySnapshot(
    String login,
    long contributorNumericId,
    String repo,
    int mergedCount,
    int closedCount,
    Instant oldestPrAt,
    Instant newestPrAt
) {}
```

`RepoMetadataSnapshot.java`:
```java
package io.casehub.devtown.domain.trust;

import java.time.Instant;

public record RepoMetadataSnapshot(
    String repo,
    int starCount,
    int contributorCount,
    int pullRequestCount,
    Instant createdAt,
    Instant lastPushedAt
) {}
```

`ContributorHistoryClient.java`:
```java
package io.casehub.devtown.domain.trust;

import java.time.Instant;

public interface ContributorHistoryClient {
    ContributorHistorySnapshot fetchHistory(String login, String repo, Instant since);
    RepoMetadataSnapshot fetchRepoMetadata(String owner, String repo);
}
```

- [ ] **Step 4: Implement BootstrapScoreComputer**

```java
package io.casehub.devtown.domain.trust;

import io.casehub.devtown.domain.ContributorTrustCapability;
import io.casehub.devtown.domain.ContributorTrustDimension;

public final class BootstrapScoreComputer {

    private BootstrapScoreComputer() {}

    public static BootstrapScoreResult compute(ContributorGitHubProfile profile,
                                                RepoConfidenceTier tier) {
        double multiplier = tier.confidenceMultiplier();
        double alpha = profile.mergedCount() * multiplier;
        double beta = profile.closedCount() * multiplier;
        double mergeRate = profile.observationCount() > 0
            ? (double) profile.mergedCount() / profile.observationCount()
            : 0.5;

        return new BootstrapScoreResult(
            alpha, beta, mergeRate, profile.observationCount(),
            profile.observationCount() == 0);
    }

    public record BootstrapScoreResult(
        double alpha,
        double beta,
        double mergeRate,
        int observationCount,
        boolean isEmpty
    ) {
        public String capabilityTag() {
            return ContributorTrustCapability.PR_CONTRIBUTION;
        }

        public String mergeRateDimension() {
            return ContributorTrustDimension.MERGE_RATE;
        }

        public double trustScore() {
            if (alpha + beta == 0) return 0.5;
            return alpha / (alpha + beta);
        }
    }
}
```

- [ ] **Step 5: Run BootstrapScoreComputer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=BootstrapScoreComputerTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/trust/ domain/src/test/java/io/casehub/devtown/domain/trust/BootstrapScoreComputerTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add ContributorGitHubProfile, SPI, and BootstrapScoreComputer Refs #202"
```

---

## Batch 2: GitHub API Adapter + Persistence

REST client extensions in `github/`, JPA entities and Flyway migrations in `app/`.

### Task 3: GitHubRepoApi and GitHubPullRequestApi extension

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubRepoApi.java`
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubPullRequestApi.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubContributorHistoryClient.java`
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubContributorHistoryClientTest.java`

**Interfaces:**
- Consumes: `ContributorHistoryClient` SPI (Task 2), `ContributorHistorySnapshot`, `RepoMetadataSnapshot`
- Produces: `GitHubContributorHistoryClient implements ContributorHistoryClient` — registered as `@ApplicationScoped`, provides GitHub-backed implementation of the SPI

- [ ] **Step 1: Create GitHubRepoApi interface**

```java
package io.casehub.devtown.github;

import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import java.util.List;
import java.util.Map;

@RegisterRestClient(configKey = "github-api")
@Path("/repos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface GitHubRepoApi {

    @GET
    @Path("/{owner}/{repo}")
    Map<String, Object> getRepository(@PathParam("owner") String owner,
                                       @PathParam("repo") String repo);

    @GET
    @Path("/{owner}/{repo}/contributors")
    List<Map<String, Object>> listContributors(@PathParam("owner") String owner,
                                                @PathParam("repo") String repo,
                                                @QueryParam("per_page") int perPage,
                                                @QueryParam("anon") String anon);
}
```

- [ ] **Step 2: Extend GitHubPullRequestApi with author-filtered listing**

Add to existing `GitHubPullRequestApi.java`:

```java
@GET
@Path("/{owner}/{repo}/pulls")
List<Map<String, Object>> listPullRequestsByAuthor(
    @PathParam("owner") String owner,
    @PathParam("repo") String repo,
    @QueryParam("creator") String creator,
    @QueryParam("state") String state,
    @QueryParam("sort") String sort,
    @QueryParam("direction") String direction,
    @QueryParam("per_page") int perPage,
    @QueryParam("page") int page);
```

- [ ] **Step 3: Write failing test for GitHubContributorHistoryClient**

```java
package io.casehub.devtown.github;

import io.casehub.devtown.domain.trust.ContributorHistorySnapshot;
import io.casehub.devtown.domain.trust.RepoMetadataSnapshot;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class GitHubContributorHistoryClientTest {

    private GitHubContributorHistoryClient client;

    @BeforeEach
    void setUp() {
        var prApi = new StubPullRequestApi();
        var repoApi = new StubRepoApi();
        client = new GitHubContributorHistoryClient(prApi, repoApi);
    }

    @Test
    void fetchHistory_countsMergedAndClosed() {
        ContributorHistorySnapshot snapshot = client.fetchHistory("alice", "org/repo", Instant.MIN);
        assertEquals("alice", snapshot.login());
        assertEquals(2, snapshot.mergedCount());
        assertEquals(1, snapshot.closedCount());
    }

    @Test
    void fetchRepoMetadata_extractsFields() {
        RepoMetadataSnapshot meta = client.fetchRepoMetadata("org", "repo");
        assertEquals("org/repo", meta.repo());
        assertEquals(100, meta.starCount());
        assertEquals(2, meta.contributorCount());
    }

    // Stub implementations inline in test class
    static class StubPullRequestApi implements GitHubPullRequestApi {
        @Override
        public List<Map<String, Object>> listPullRequests(String o, String r, String h, String b, String s) {
            return List.of();
        }
        @Override
        public Map<String, Object> createPullRequest(String o, String r, Map<String, Object> body) {
            return Map.of();
        }
        @Override
        public List<Map<String, Object>> listPullRequestsByAuthor(String o, String r, String creator,
                String state, String sort, String direction, int perPage, int page) {
            if (page > 1) return List.of();
            return List.of(
                Map.of("state", "closed", "merged_at", "2026-01-01T00:00:00Z",
                       "created_at", "2025-12-30T00:00:00Z",
                       "user", Map.of("login", "alice", "id", 123)),
                Map.of("state", "closed", "merged_at", "2026-02-01T00:00:00Z",
                       "created_at", "2026-01-15T00:00:00Z",
                       "user", Map.of("login", "alice", "id", 123)),
                Map.of("state", "closed", "created_at", "2026-03-01T00:00:00Z",
                       "user", Map.of("login", "alice", "id", 123))
            );
        }
    }

    static class StubRepoApi implements GitHubRepoApi {
        @Override
        public Map<String, Object> getRepository(String o, String r) {
            return Map.of("full_name", "org/repo", "stargazers_count", 100,
                          "created_at", "2020-01-01T00:00:00Z",
                          "pushed_at", "2026-09-01T00:00:00Z");
        }
        @Override
        public List<Map<String, Object>> listContributors(String o, String r, int perPage, String anon) {
            return List.of(Map.of("login", "alice"), Map.of("login", "bob"));
        }
    }
}
```

- [ ] **Step 4: Implement GitHubContributorHistoryClient**

```java
package io.casehub.devtown.github;

import io.casehub.devtown.domain.trust.ContributorHistoryClient;
import io.casehub.devtown.domain.trust.ContributorHistorySnapshot;
import io.casehub.devtown.domain.trust.RepoMetadataSnapshot;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.rest.client.inject.RestClient;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class GitHubContributorHistoryClient implements ContributorHistoryClient {

    private final GitHubPullRequestApi prApi;
    private final GitHubRepoApi repoApi;

    @Inject
    public GitHubContributorHistoryClient(@RestClient GitHubPullRequestApi prApi,
                                           @RestClient GitHubRepoApi repoApi) {
        this.prApi = prApi;
        this.repoApi = repoApi;
    }

    GitHubContributorHistoryClient(GitHubPullRequestApi prApi, GitHubRepoApi repoApi) {
        this.prApi = prApi;
        this.repoApi = repoApi;
    }

    @Override
    public ContributorHistorySnapshot fetchHistory(String login, String repo, Instant since) {
        String[] parts = repo.split("/");
        int merged = 0, closed = 0;
        long numericId = 0;
        Instant oldest = null, newest = null;

        for (int page = 1; ; page++) {
            List<Map<String, Object>> prs = prApi.listPullRequestsByAuthor(
                parts[0], parts[1], login, "all", "created", "desc", 100, page);
            if (prs.isEmpty()) break;

            for (Map<String, Object> pr : prs) {
                Instant createdAt = Instant.parse((String) pr.get("created_at"));
                if (since != Instant.MIN && createdAt.isBefore(since)) {
                    return new ContributorHistorySnapshot(login, numericId, repo,
                        merged, closed, oldest, newest);
                }

                @SuppressWarnings("unchecked")
                Map<String, Object> user = (Map<String, Object>) pr.get("user");
                if (numericId == 0) numericId = ((Number) user.get("id")).longValue();
                if (newest == null) newest = createdAt;
                oldest = createdAt;

                if (pr.get("merged_at") != null) {
                    merged++;
                } else {
                    closed++;
                }
            }
            if (prs.size() < 100) break;
        }
        return new ContributorHistorySnapshot(login, numericId, repo,
            merged, closed, oldest, newest);
    }

    @Override
    public RepoMetadataSnapshot fetchRepoMetadata(String owner, String repo) {
        Map<String, Object> data = repoApi.getRepository(owner, repo);
        List<Map<String, Object>> contributors = repoApi.listContributors(owner, repo, 1, "false");
        int contributorCount = contributors.size();
        // GitHub returns paginated — for tier classification, approximate count is sufficient
        return new RepoMetadataSnapshot(
            owner + "/" + repo,
            ((Number) data.get("stargazers_count")).intValue(),
            contributorCount,
            0, // PR count derived from contributor history, not needed separately
            Instant.parse((String) data.get("created_at")),
            Instant.parse((String) data.get("pushed_at")));
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=GitHubContributorHistoryClientTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add github/src/main/java/io/casehub/devtown/github/GitHubRepoApi.java github/src/main/java/io/casehub/devtown/github/GitHubContributorHistoryClient.java github/src/main/java/io/casehub/devtown/github/GitHubPullRequestApi.java github/src/test/java/io/casehub/devtown/github/GitHubContributorHistoryClientTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add GitHub contributor history adapter and repo metadata API Refs #202"
```

### Task 4: JPA entities, Flyway migration, and BootstrapContributorEvent

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/trust/ContributorGitHubProfileEntity.java`
- Create: `app/src/main/java/io/casehub/devtown/app/trust/RepoConfidenceProfileEntity.java`
- Create: `app/src/main/java/io/casehub/devtown/app/trust/NoOpContributorHistoryClient.java`
- Create: `app/src/main/resources/db/devtown/migration/V2__contributor_github_profile.sql`
- Create: `review/src/main/java/io/casehub/devtown/review/BootstrapContributorEvent.java`
- Test: `app/src/test/java/io/casehub/devtown/app/trust/ContributorGitHubProfileEntityTest.java`

**Interfaces:**
- Consumes: `ContributorGitHubProfile`, `RepoConfidenceProfile`, `CacheMaturity`, `RepoConfidenceTier` (Task 1-2), `ContributorHistoryClient` (Task 2)
- Produces: `ContributorGitHubProfileEntity` (JPA managed), `RepoConfidenceProfileEntity` (JPA managed), `NoOpContributorHistoryClient @DefaultBean`, `BootstrapContributorEvent(String login, long contributorNumericId, String repo)`

- [ ] **Step 1: Create Flyway migration**

```sql
-- V2__contributor_github_profile.sql
-- GitHub-derived contributor intelligence cache (#202)

CREATE TABLE contributor_github_profile (
    id                      UUID         NOT NULL PRIMARY KEY,
    login                   VARCHAR(255) NOT NULL,
    contributor_numeric_id  BIGINT       NOT NULL,
    repo                    VARCHAR(255) NOT NULL,
    actor_id                VARCHAR(255) NOT NULL,
    merged_count            INTEGER      NOT NULL DEFAULT 0,
    closed_count            INTEGER      NOT NULL DEFAULT 0,
    observation_count       INTEGER      NOT NULL DEFAULT 0,
    last_refresh_at         TIMESTAMP,
    maturity                VARCHAR(20)  NOT NULL DEFAULT 'COLD',
    CONSTRAINT uq_contributor_github_profile UNIQUE (login, repo)
);

CREATE INDEX idx_cgp_actor_id ON contributor_github_profile(actor_id);

CREATE TABLE repo_confidence_profile (
    id                  UUID         NOT NULL PRIMARY KEY,
    repo                VARCHAR(255) NOT NULL,
    star_count          INTEGER      NOT NULL DEFAULT 0,
    contributor_count   INTEGER      NOT NULL DEFAULT 0,
    pull_request_count  INTEGER      NOT NULL DEFAULT 0,
    created_at          TIMESTAMP,
    last_pushed_at      TIMESTAMP,
    tier                VARCHAR(20)  NOT NULL DEFAULT 'LOW',
    admin_override      BOOLEAN      NOT NULL DEFAULT false,
    last_refresh_at     TIMESTAMP,
    CONSTRAINT uq_repo_confidence_profile UNIQUE (repo)
);
```

- [ ] **Step 2: Create JPA entities**

`ContributorGitHubProfileEntity.java`:
```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.trust.CacheMaturity;
import io.casehub.devtown.domain.trust.ContributorGitHubProfile;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "contributor_github_profile")
public class ContributorGitHubProfileEntity {
    @Id public UUID id;
    @Column(nullable = false) public String login;
    @Column(name = "contributor_numeric_id", nullable = false) public long contributorNumericId;
    @Column(nullable = false) public String repo;
    @Column(name = "actor_id", nullable = false) public String actorId;
    @Column(name = "merged_count") public int mergedCount;
    @Column(name = "closed_count") public int closedCount;
    @Column(name = "observation_count") public int observationCount;
    @Column(name = "last_refresh_at") public Instant lastRefreshAt;
    @Enumerated(EnumType.STRING) public CacheMaturity maturity;

    public ContributorGitHubProfile toDomain() {
        return new ContributorGitHubProfile(login, contributorNumericId, repo, actorId,
            mergedCount, closedCount, observationCount, lastRefreshAt, maturity);
    }

    public static ContributorGitHubProfileEntity fromDomain(ContributorGitHubProfile p) {
        var e = new ContributorGitHubProfileEntity();
        e.id = UUID.randomUUID();
        e.login = p.login();
        e.contributorNumericId = p.contributorNumericId();
        e.repo = p.repo();
        e.actorId = p.actorId();
        e.mergedCount = p.mergedCount();
        e.closedCount = p.closedCount();
        e.observationCount = p.observationCount();
        e.lastRefreshAt = p.lastRefreshAt();
        e.maturity = p.maturity();
        return e;
    }
}
```

`RepoConfidenceProfileEntity.java` — same pattern, mapping `RepoConfidenceProfile`.

`NoOpContributorHistoryClient.java`:
```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.trust.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;

@ApplicationScoped
@DefaultBean
public class NoOpContributorHistoryClient implements ContributorHistoryClient {
    @Override
    public ContributorHistorySnapshot fetchHistory(String login, String repo, Instant since) {
        return new ContributorHistorySnapshot(login, 0, repo, 0, 0, null, null);
    }
    @Override
    public RepoMetadataSnapshot fetchRepoMetadata(String owner, String repo) {
        return new RepoMetadataSnapshot(owner + "/" + repo, 0, 0, 0, Instant.now(), Instant.now());
    }
}
```

- [ ] **Step 3: Create BootstrapContributorEvent**

```java
package io.casehub.devtown.review;

public record BootstrapContributorEvent(
    String login,
    long contributorNumericId,
    String repo
) {}
```

- [ ] **Step 4: Write JPA round-trip test**

```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.trust.CacheMaturity;
import io.casehub.devtown.domain.trust.ContributorGitHubProfile;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class ContributorGitHubProfileEntityTest {

    @Inject EntityManager em;

    @Test
    @Transactional
    void persistAndRetrieve() {
        var profile = new ContributorGitHubProfile("alice", 123L, "org/repo",
            "github-id:123", 10, 3, 13, Instant.now(), CacheMaturity.WARM);
        var entity = ContributorGitHubProfileEntity.fromDomain(profile);
        em.persist(entity);
        em.flush();
        em.clear();

        var found = em.find(ContributorGitHubProfileEntity.class, entity.id);
        assertNotNull(found);
        var domain = found.toDomain();
        assertEquals("alice", domain.login());
        assertEquals(123L, domain.contributorNumericId());
        assertEquals(10, domain.mergedCount());
        assertEquals(CacheMaturity.WARM, domain.maturity());
    }
}
```

- [ ] **Step 5: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContributorGitHubProfileEntityTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/trust/ app/src/main/resources/db/devtown/migration/V2__contributor_github_profile.sql app/src/test/java/io/casehub/devtown/app/trust/ContributorGitHubProfileEntityTest.java review/src/main/java/io/casehub/devtown/review/BootstrapContributorEvent.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add JPA entities, Flyway migration, and bootstrap event for contributor intelligence Refs #202"
```

---

## Batch 3: Bootstrap Writer + TrustBootstrapSource Integration

The core bootstrap pipeline — CDI observer, trust import, and pull SPI.

### Task 5: ContributorBootstrapWriter and DevtownTrustBootstrapSource

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/trust/ContributorBootstrapWriter.java`
- Create: `app/src/main/java/io/casehub/devtown/app/trust/DevtownTrustBootstrapSource.java`
- Test: `app/src/test/java/io/casehub/devtown/app/trust/ContributorBootstrapWriterTest.java`
- Test: `app/src/test/java/io/casehub/devtown/app/trust/DevtownTrustBootstrapSourceTest.java`

**Interfaces:**
- Consumes: `ContributorHistoryClient` (Task 2), `BootstrapScoreComputer` (Task 2), `ContributorGitHubProfileEntity` (Task 4), `RepoConfidenceProfileEntity` (Task 4), `BootstrapContributorEvent` (Task 4), `TrustImportService` (casehub-ledger), `TrustBootstrapSource` (casehub-ledger), `TrustExportPayload`, `ActorExport`, `CapabilityScoreExport`, `DimensionScoreExport`, `CapabilityDimensionScoreExport`
- Produces: `ContributorBootstrapWriter` observes `BootstrapContributorEvent`, orchestrates bootstrap with dedup, persists profiles, calls `TrustImportService`. `DevtownTrustBootstrapSource` implements `TrustBootstrapSource`, reads from cached profiles.

- [ ] **Step 1: Write failing test for ContributorBootstrapWriter**

```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.trust.*;
import io.casehub.devtown.review.BootstrapContributorEvent;
import io.casehub.ledger.runtime.service.federation.*;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.concurrent.atomic.AtomicReference;
import static org.junit.jupiter.api.Assertions.*;

class ContributorBootstrapWriterTest {

    private AtomicReference<TrustExportPayload> importedPayload;
    private ContributorBootstrapWriter writer;

    @BeforeEach
    void setUp() {
        importedPayload = new AtomicReference<>();
        var historyClient = new StubHistoryClient();
        var importService = (TrustImportService) payload -> importedPayload.set(payload);
        writer = new ContributorBootstrapWriter(historyClient, importService);
    }

    @Test
    void onBootstrap_importsTrustWithCorrectAlphaBeta() {
        var event = new BootstrapContributorEvent("alice", 123L, "org/repo");
        writer.onBootstrap(event);

        var payload = importedPayload.get();
        assertNotNull(payload);
        assertEquals(1, payload.actors().size());

        var actor = payload.actors().get(0);
        assertEquals("github-id:123", actor.actorId());
        assertEquals(ActorType.HUMAN, actor.actorType());

        var cap = actor.capabilityScores().get(0);
        assertEquals("pr-contribution", cap.capabilityTag());
        assertEquals(1.5, cap.alpha(), 0.001);  // 5 merged * 0.3 (LOW tier)
        assertEquals(0.6, cap.beta(), 0.001);   // 2 closed * 0.3

        var dim = actor.dimensionScores().get(0);
        assertEquals("merge-rate", dim.dimension());
    }

    static class StubHistoryClient implements ContributorHistoryClient {
        @Override
        public ContributorHistorySnapshot fetchHistory(String login, String repo, Instant since) {
            return new ContributorHistorySnapshot(login, 123L, repo, 5, 2,
                Instant.parse("2024-01-01T00:00:00Z"), Instant.parse("2026-09-01T00:00:00Z"));
        }
        @Override
        public RepoMetadataSnapshot fetchRepoMetadata(String owner, String repo) {
            return new RepoMetadataSnapshot(owner + "/" + repo, 10, 2, 20,
                Instant.parse("2025-01-01T00:00:00Z"), Instant.parse("2026-09-01T00:00:00Z"));
        }
    }
}
```

- [ ] **Step 2: Implement ContributorBootstrapWriter**

```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.trust.*;
import io.casehub.devtown.review.BootstrapContributorEvent;
import io.casehub.ledger.runtime.service.federation.*;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ContributorBootstrapWriter {

    private static final Logger LOG = Logger.getLogger(ContributorBootstrapWriter.class);
    private final ConcurrentHashMap<String, CompletableFuture<Void>> inFlight = new ConcurrentHashMap<>();

    private final ContributorHistoryClient historyClient;
    private final TrustImportService importService;

    @Inject EntityManager em;

    @Inject
    public ContributorBootstrapWriter(ContributorHistoryClient historyClient,
                                       TrustImportService importService) {
        this.historyClient = historyClient;
        this.importService = importService;
    }

    ContributorBootstrapWriter(ContributorHistoryClient historyClient,
                                TrustImportService importService) {
        this.historyClient = historyClient;
        this.importService = importService;
    }

    public void onBootstrap(@ObservesAsync BootstrapContributorEvent event) {
        String key = event.login() + ":" + event.repo();
        var existing = inFlight.putIfAbsent(key, new CompletableFuture<>());
        if (existing != null) {
            LOG.debugf("Bootstrap already in-flight for %s — skipping", key);
            return;
        }
        try {
            doBootstrap(event);
        } finally {
            var future = inFlight.remove(key);
            if (future != null) future.complete(null);
        }
    }

    void doBootstrap(BootstrapContributorEvent event) {
        var snapshot = historyClient.fetchHistory(event.login(), event.repo(), Instant.MIN);
        String[] repoParts = event.repo().split("/");
        var meta = historyClient.fetchRepoMetadata(repoParts[0], repoParts[1]);

        var tier = RepoConfidenceTier.classify(meta.contributorCount(),
            meta.pullRequestCount(), meta.createdAt(), Instant.now());

        String actorId = ContributorGitHubProfile.actorIdFromGitHubNumericId(
            event.contributorNumericId());
        var profile = new ContributorGitHubProfile(
            event.login(), event.contributorNumericId(), event.repo(),
            actorId, snapshot.mergedCount(), snapshot.closedCount(),
            snapshot.mergedCount() + snapshot.closedCount(),
            Instant.now(), CacheMaturity.forObservationCount(
                snapshot.mergedCount() + snapshot.closedCount()));

        var result = BootstrapScoreComputer.compute(profile, tier);
        if (!result.isEmpty()) {
            importTrust(actorId, result);
        }

        persistProfile(profile);
        persistRepoMetadata(meta, tier);
        LOG.infof("Bootstrapped contributor %s in %s: %d merged, %d closed, tier=%s",
            event.login(), event.repo(), snapshot.mergedCount(), snapshot.closedCount(), tier);
    }

    private void importTrust(String actorId, BootstrapScoreComputer.BootstrapScoreResult result) {
        var now = Instant.now();
        var capScore = new CapabilityScoreExport(
            result.capabilityTag(), result.alpha(), result.beta(),
            result.trustScore(), result.observationCount(),
            (int) Math.round(result.alpha()), (int) Math.round(result.beta()), now);

        var dimScore = new DimensionScoreExport(
            result.mergeRateDimension(), result.mergeRate(),
            result.observationCount(), now);

        var capDimScore = new CapabilityDimensionScoreExport(
            result.capabilityTag(), result.mergeRateDimension(),
            result.mergeRate(), result.observationCount(), now);

        var actor = new ActorExport(actorId, ActorType.HUMAN, null,
            List.of(capScore), List.of(dimScore), List.of(capDimScore));

        var payload = new TrustExportPayload(now, "devtown-github-bootstrap", List.of(actor));
        importService.importTrust(payload);
    }

    @Transactional
    void persistProfile(ContributorGitHubProfile profile) {
        if (em == null) return;
        var entity = ContributorGitHubProfileEntity.fromDomain(profile);
        em.merge(entity);
    }

    @Transactional
    void persistRepoMetadata(RepoMetadataSnapshot meta, RepoConfidenceTier tier) {
        if (em == null) return;
        var entity = RepoConfidenceProfileEntity.fromSnapshot(meta, tier);
        em.merge(entity);
    }
}
```

- [ ] **Step 3: Run ContributorBootstrapWriter tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContributorBootstrapWriterTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 4: Write failing test for DevtownTrustBootstrapSource**

```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.trust.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

class DevtownTrustBootstrapSourceTest {

    @Test
    void fetchPriorTrust_withProfile_returnsPayload() {
        var profile = new ContributorGitHubProfile("alice", 123L, "org/repo",
            "github-id:123", 10, 3, 13, Instant.now(), CacheMaturity.WARM);
        var source = new DevtownTrustBootstrapSource(
            actorId -> actorId.equals("github-id:123")
                ? Optional.of(profile) : Optional.empty(),
            actorId -> Optional.of(RepoConfidenceTier.MEDIUM));

        var result = source.fetchPriorTrust("github-id:123");

        assertTrue(result.isPresent());
        assertEquals(1, result.get().actors().size());
        assertEquals("github-id:123", result.get().actors().get(0).actorId());
    }

    @Test
    void fetchPriorTrust_noProfile_empty() {
        var source = new DevtownTrustBootstrapSource(
            actorId -> Optional.empty(),
            actorId -> Optional.empty());

        var result = source.fetchPriorTrust("github-id:999");

        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 5: Implement DevtownTrustBootstrapSource**

```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.trust.*;
import io.casehub.ledger.runtime.service.federation.*;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.NoResultException;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.function.Function;

@ApplicationScoped
public class DevtownTrustBootstrapSource implements TrustBootstrapSource {

    private final Function<String, Optional<ContributorGitHubProfile>> profileLookup;
    private final Function<String, Optional<RepoConfidenceTier>> tierLookup;

    @Inject
    public DevtownTrustBootstrapSource(EntityManager em) {
        this.profileLookup = actorId -> {
            try {
                var entity = em.createQuery(
                    "SELECT p FROM ContributorGitHubProfileEntity p WHERE p.actorId = :actorId",
                    ContributorGitHubProfileEntity.class)
                    .setParameter("actorId", actorId)
                    .setMaxResults(1)
                    .getSingleResult();
                return Optional.of(entity.toDomain());
            } catch (NoResultException e) {
                return Optional.empty();
            }
        };
        this.tierLookup = actorId -> {
            try {
                var profile = em.createQuery(
                    "SELECT p FROM ContributorGitHubProfileEntity p WHERE p.actorId = :actorId",
                    ContributorGitHubProfileEntity.class)
                    .setParameter("actorId", actorId)
                    .setMaxResults(1)
                    .getSingleResult();
                var repo = em.createQuery(
                    "SELECT r FROM RepoConfidenceProfileEntity r WHERE r.repo = :repo",
                    RepoConfidenceProfileEntity.class)
                    .setParameter("repo", profile.repo)
                    .setMaxResults(1)
                    .getSingleResult();
                return Optional.of(repo.tier);
            } catch (NoResultException e) {
                return Optional.of(RepoConfidenceTier.LOW);
            }
        };
    }

    DevtownTrustBootstrapSource(Function<String, Optional<ContributorGitHubProfile>> profileLookup,
                                 Function<String, Optional<RepoConfidenceTier>> tierLookup) {
        this.profileLookup = profileLookup;
        this.tierLookup = tierLookup;
    }

    @Override
    public Optional<TrustExportPayload> fetchPriorTrust(String actorId) {
        return profileLookup.apply(actorId).map(profile -> {
            var tier = tierLookup.apply(actorId).orElse(RepoConfidenceTier.LOW);
            var result = BootstrapScoreComputer.compute(profile, tier);
            if (result.isEmpty()) return null;

            var now = Instant.now();
            var capScore = new CapabilityScoreExport(
                result.capabilityTag(), result.alpha(), result.beta(),
                result.trustScore(), result.observationCount(),
                (int) Math.round(result.alpha()), (int) Math.round(result.beta()), now);
            var dimScore = new DimensionScoreExport(
                result.mergeRateDimension(), result.mergeRate(),
                result.observationCount(), now);
            var capDimScore = new CapabilityDimensionScoreExport(
                result.capabilityTag(), result.mergeRateDimension(),
                result.mergeRate(), result.observationCount(), now);

            var actor = new ActorExport(actorId, ActorType.HUMAN, null,
                List.of(capScore), List.of(dimScore), List.of(capDimScore));
            return new TrustExportPayload(now, "devtown-github-bootstrap", List.of(actor));
        });
    }
}
```

- [ ] **Step 6: Run DevtownTrustBootstrapSource tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownTrustBootstrapSourceTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/trust/ContributorBootstrapWriter.java app/src/main/java/io/casehub/devtown/app/trust/DevtownTrustBootstrapSource.java app/src/test/java/io/casehub/devtown/app/trust/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: add ContributorBootstrapWriter and DevtownTrustBootstrapSource Refs #202"
```

---

## Batch 4: PR Intake Wiring + UI Integration

Wire the bootstrap trigger into the PR intake path. Extend the governance UI.

### Task 6: Bootstrap trigger from PR intake + GovernanceQueryService extension

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java` (or wherever PR intake dispatches)
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java`
- Create: `app/src/main/java/io/casehub/devtown/app/governance/GitHubIntelligence.java`
- Test: `app/src/test/java/io/casehub/devtown/app/trust/ContributorBootstrapIntegrationTest.java`

**Interfaces:**
- Consumes: `BootstrapContributorEvent` (Task 4), `PrPayload.contributor()`, `PrPayload.contributorNumericId()`, `ContributorGitHubProfileEntity` (Task 4), `GovernanceQueryService.ContributorDetail` (existing)
- Produces: Fires `BootstrapContributorEvent` async on PR intake when no fresh profile exists. `GovernanceQueryService.ContributorDetail` extended with `GitHubIntelligence` field. `GitHubIntelligence` record for UI consumption.

- [ ] **Step 1: Create GitHubIntelligence record**

```java
package io.casehub.devtown.app.governance;

import java.time.Instant;

public record GitHubIntelligence(
    int mergedCount,
    int closedCount,
    double mergeRatio,
    String repoConfidenceTier,
    String cacheMaturity,
    Instant lastRefreshed,
    boolean bootstrapped,
    String bootstrapSummary
) {}
```

- [ ] **Step 2: Wire bootstrap trigger into PR intake**

In `PrReviewCaseService` (or the appropriate intake handler), after receiving a `PrPayload`, check for an existing fresh profile. If none exists or stale, fire async event:

```java
// Add to the intake method, after PrPayload is constructed:
@Inject jakarta.enterprise.event.Event<BootstrapContributorEvent> bootstrapEvent;
@Inject EntityManager em;

// Check for existing fresh profile
private boolean needsBootstrap(String login, String repo) {
    try {
        var entity = em.createQuery(
            "SELECT p FROM ContributorGitHubProfileEntity p WHERE p.login = :login AND p.repo = :repo",
            ContributorGitHubProfileEntity.class)
            .setParameter("login", login).setParameter("repo", repo)
            .getSingleResult();
        return entity.maturity.isStale(entity.lastRefreshAt, Instant.now());
    } catch (NoResultException e) {
        return true;
    }
}

// In the intake method:
if (needsBootstrap(payload.contributor(), payload.repo())) {
    bootstrapEvent.fireAsync(new BootstrapContributorEvent(
        payload.contributor(), payload.contributorNumericId(), payload.repo()));
}
```

- [ ] **Step 3: Extend GovernanceQueryService.contributorDetail()**

Add `GitHubIntelligence` to the `ContributorDetail` record and populate it from the cached profile entity when available. Modify `contributorDetail()` to query `ContributorGitHubProfileEntity` by actorId and build the `GitHubIntelligence` record.

- [ ] **Step 4: Write integration test**

Test the full flow: fire `BootstrapContributorEvent` → verify profile persisted → verify `TrustImportService` called → verify `GovernanceQueryService.contributorDetail()` returns `GitHubIntelligence`.

- [ ] **Step 5: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (all existing + new tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/ app/src/test/java/io/casehub/devtown/app/trust/ContributorBootstrapIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat: wire bootstrap trigger from PR intake, extend governance UI with GitHub intelligence Refs #202"
```

### Task 7: Full build verification and docs

**Files:**
- Modify: `docs/guides/consumer-guide.md` (if contributor trust section exists)
- Modify: `docs/guides/contributor-guide.md` (document new SPIs and module additions)

- [ ] **Step 1: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 2: Update contributor-guide.md**

Document:
- New SPI: `ContributorHistoryClient` in `domain/` — port interface for contributor history fetching
- New adapter: `GitHubContributorHistoryClient` in `github/` — GitHub REST implementation
- New persistence: `contributor_github_profile` and `repo_confidence_profile` tables
- New trust integration: `DevtownTrustBootstrapSource` displaces `NoOpTrustBootstrapSource`
- Configuration: GitHub API authentication required for production (`github-api` REST client config key)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add docs/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "docs: document GitHub contributor intelligence SPIs and configuration Refs #202"
```

---

## References

- [2026-09-14-github-contributor-confidence-design.md] — design spec this plan implements
- [ContributorAttestationPolicy.java](app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java) — existing attestation mapping pattern
- [ContributorOutcomeLedgerWriter.java](app/src/main/java/io/casehub/devtown/app/ledger/ContributorOutcomeLedgerWriter.java) — app-tier writer pattern
- [ContributorIntakePolicy.java](domain/src/main/java/io/casehub/devtown/domain/ContributorIntakePolicy.java) — lane classification
- [PrPayload.java](review/src/main/java/io/casehub/devtown/review/PrPayload.java) — PR intake record
- [GitHubPullRequestApi.java](github/src/main/java/io/casehub/devtown/github/GitHubPullRequestApi.java) — existing REST client
- TrustBootstrapSource (casehub-ledger) — `fetchPriorTrust(actorId) → Optional<TrustExportPayload>`
- TrustImportService (casehub-ledger) — `importTrust(TrustExportPayload)`
- ActorExport, CapabilityScoreExport, DimensionScoreExport, CapabilityDimensionScoreExport (casehub-ledger federation)
- [GE-20260530-fcc6c3] — ConcurrentHashMap TTL cache stale-entry gotcha
- [GE-20260607-3defda] — Per-actor computation cache with event-driven invalidation
- casehubio/devtown#202 — issue
