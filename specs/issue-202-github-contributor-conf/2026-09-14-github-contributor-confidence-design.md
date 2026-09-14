# GitHub-Derived Contributor Confidence — Design Spec

**Date:** 2026-09-14
**Issue:** casehubio/devtown#202
**Status:** Design — pending implementation plan

---

## Problem

A contributor with 500 merged PRs on GitHub starts at zero in devtown. The only
score data comes from internal attestations generated after devtown is deployed.
Every new contributor begins in TRIAGE lane regardless of their track record.
GitHub's observable PR history is a trusted, authoritative signal that the system
ignores.

## Solution

Bootstrap contributor trust from GitHub PR history. Cache per-contributor
per-repo signal data with adaptive TTL. Seed the Bayesian Beta prior via
`TrustBootstrapSource` for historical data. Continue using
`ContributorAttestationPolicy` for live PR observations. Classify repo
quality into confidence tiers that modulate how much weight historical data
carries.

---

## Architecture

### Data flow

```
PR webhook arrives
  → ContributorIntelligenceService.onPrOpened(contributorLogin, repo)
  → is contributor profile cached and fresh?
      YES → no action (live attestation pipeline handles scoring)
      NO  → classify PR as TRIAGE immediately (safe default)
          → fire async CDI event: BootstrapContributorEvent(login, repo)
          → ContributorIntelligenceService.bootstrap(login, repo)
              → ContributorHistoryClient.fetchPrHistory(login, repo)
              → ContributorHistoryClient.fetchRepoMetadata(repo)
              → build ContributorGitHubProfile (aggregate cache)
              → build/update RepoConfidenceProfile (tier assignment)
              → compute α/β from merge/reject ratio × repo tier multiplier
              → TrustBootstrapSource.seed(actorId, capability, α, β)
              → Bayesian Beta prior updated
          → next PR from this contributor uses cached score
```

### Module placement

Following the hexagonal port/adapter pattern established by
`CiStatusClient`/`GitHubCiStatusClient`:

| Component | Module | Rationale |
|-----------|--------|-----------|
| `ContributorHistoryClient` (SPI interface) | `domain/` | Port — service depends on this, not the adapter |
| `ContributorGitHubProfile` (domain record) | `domain/` | Pure Java, no framework deps |
| `RepoConfidenceTier` (enum) | `domain/` | HIGH / MEDIUM / LOW with confidence multipliers |
| `ContributorIntelligenceService` | `review/` | Integration logic, calls SPI |
| `GitHubContributorHistoryClient` | `github/` | Adapter — implements the SPI via GitHub REST API |
| `GitHubRepoApi` (REST client interface) | `github/` | New interface for repo metadata endpoint |
| `ContributorGitHubProfileEntity` (JPA) | `app/` | Persistence |
| `RepoConfidenceProfileEntity` (JPA) | `app/` | Persistence |
| `NoOpContributorHistoryClient` (`@DefaultBean`) | `app/` | Fallback when GitHub not configured |
| CDI event wiring, bootstrap observer | `app/` | Application-tier glue |

### Extensibility points

The design anticipates adding signals beyond the initial merge ratio and
revision count. Extension points:

1. **`ContributorHistoryClient` SPI** — returns a `ContributorHistorySnapshot`
   record containing signal data. New signals are new fields on this record.
   The SPI contract is: "given a contributor login and repo, return observable
   signal data." Implementations (GitHub, GitLab, etc.) populate what they can.

2. **`ContributorGitHubProfile` cache entity** — aggregate columns for each
   signal. For 2-4 signals, columns are appropriate (simple queries, typed).
   If signal count grows past 4, migrate signal details to a JSON column
   alongside the stable structural fields (observationCount, lastRefreshAt,
   cacheMaturity). This migration is safe — the entity is a cache that can
   be rebuilt from the API.

3. **α/β computation** — for v1, inline: α = mergedCount × tier multiplier,
   β = closedCount × tier multiplier. When signal 3+ arrives, extract into
   a `SignalWeightPolicy` that maps signal values to Bayesian update
   parameters. New signals register their weight contribution. The policy
   becomes the single place where "what does this signal mean for trust?"
   is answered.

4. **Repo confidence tier criteria** — the threshold rules (age >1yr, >5
   contributors, >50 PRs) are configurable via `PreferenceKey`. New criteria
   can be added without structural changes.

---

## Domain model

### ContributorGitHubProfile

Per-contributor, per-repo cached aggregate. This is a cache — always
rebuildable from the GitHub API.

```java
// domain/ — pure Java record
public record ContributorGitHubProfile(
    String login,           // GitHub username (natural key)
    String repo,            // owner/repo (natural key)
    String actorId,         // deterministic UUID: github:<login>
    int mergedCount,        // PRs merged
    int closedCount,        // PRs closed without merge
    int totalForcePushes,   // total force-pushes across all PRs (revision proxy)
    int observationCount,   // mergedCount + closedCount
    Instant lastRefreshAt,  // last successful API fetch
    CacheMaturity maturity  // computed from observationCount
) {}

public enum CacheMaturity {
    COLD(0, 5),       // refresh on every PR
    WARM(5, 20),      // refresh if >1 day stale
    HOT(20, 50),      // refresh if >3 days stale
    MATURE(50, -1);   // refresh if >7 days stale

    // isStale(Instant lastRefresh, Instant now) method
}
```

### RepoConfidenceTier

```java
// domain/
public enum RepoConfidenceTier {
    HIGH(0.8),    // first-party repos or admin-designated
    MEDIUM(0.5),  // meets thresholds: age >1yr, >5 contributors, >50 PRs
    LOW(0.3);     // everything else

    private final double confidenceMultiplier;
    // The multiplier scales the α/β values seeded into the Beta prior.
    // HIGH: historical PRs carry 80% of the weight of a live devtown observation
    // LOW: historical PRs carry 30% — we trust the merge happened but not the
    //      review quality
}
```

### RepoConfidenceProfile

Per-repo cached metadata, used to assign a confidence tier.

```java
// domain/
public record RepoConfidenceProfile(
    String repo,              // owner/repo (natural key)
    int starCount,            // informational, not used in tier assignment
    int contributorCount,     // used in tier threshold
    int pullRequestCount,     // used in tier threshold
    Instant createdAt,        // used in tier threshold (age)
    Instant lastPushedAt,     // informational
    RepoConfidenceTier tier,  // computed or admin-overridden
    boolean adminOverride,    // true if tier was set by admin
    Instant lastRefreshAt     // 7-day TTL
) {}
```

### ContributorHistoryClient (SPI)

```java
// domain/ — port interface
public interface ContributorHistoryClient {

    ContributorHistorySnapshot fetchHistory(String login, String repo,
                                            Instant since);

    RepoMetadataSnapshot fetchRepoMetadata(String owner, String repo);
}

// Returned by fetchHistory — extensible record for signal data
public record ContributorHistorySnapshot(
    String login,
    String repo,
    int mergedCount,
    int closedCount,
    int forcePushCount,
    Instant oldestPrAt,
    Instant newestPrAt
    // Future signals: int ciPassCount, int ciFailCount,
    //                 int reviewCommentTotal, Duration avgTimeToApproval
) {}

public record RepoMetadataSnapshot(
    String repo,
    int starCount,
    int contributorCount,
    int pullRequestCount,
    Instant createdAt,
    Instant lastPushedAt
) {}
```

### Identity mapping

GitHub login → devtown actor ID via deterministic UUID:

```java
public static UUID actorIdFromGitHubLogin(String login) {
    return UUID.nameUUIDFromBytes(("github:" + login).getBytes(StandardCharsets.UTF_8));
}
```

Consistent across repos — the same GitHub user contributing to multiple repos
produces the same actor ID, enabling future org-level aggregation.

---

## GitHub API integration

### GitHubPullRequestApi — extended

Add author-filtered PR listing to the existing interface:

```java
@GET
@Path("/{owner}/{repo}/pulls")
List<Map<String, Object>> listPullRequestsByAuthor(
    @PathParam("owner") String owner,
    @PathParam("repo") String repo,
    @QueryParam("creator") String creator,    // GitHub login
    @QueryParam("state") String state,        // "all" for merged + closed
    @QueryParam("sort") String sort,          // "created"
    @QueryParam("direction") String direction, // "desc"
    @QueryParam("per_page") int perPage,      // 100 (max)
    @QueryParam("page") int page);
```

Pagination: follow GitHub's `Link` header for `rel="next"`. Stop when no
more pages or when PR `created_at` is before `since` parameter (for
incremental refresh).

### GitHubRepoApi — new interface

```java
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
    List<Map<String, Object>> listContributors(
        @PathParam("owner") String owner,
        @PathParam("repo") String repo,
        @QueryParam("per_page") int perPage,
        @QueryParam("anon") String anon);  // "true" to include anonymous
}
```

### Rate limiting

GitHub rate limits: 60 req/hr unauthenticated, 5000 req/hr authenticated.
Production deployments MUST use authenticated access (PAT or GitHub App).

Rate-limit handling via MicroProfile REST client response filter:
- Read `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers
- If remaining < 100: log warning, slow down (add delay between requests)
- If remaining == 0: pause until reset time, then retry
- If 429 response: read `Retry-After` header, pause, retry

Bootstrap is async (D8) so rate-limit pauses don't affect the intake path.

---

## Bootstrap and refresh flow

### Initial bootstrap (first encounter)

1. PR arrives from unknown contributor (no `ContributorGitHubProfile` for
   this login+repo)
2. PR classified as TRIAGE immediately (safe default)
3. `BootstrapContributorEvent(login, repo)` fired as async CDI event
4. `ContributorIntelligenceService` observes the event:
   a. Call `ContributorHistoryClient.fetchHistory(login, repo, Instant.MIN)`
      — full history
   b. Call `ContributorHistoryClient.fetchRepoMetadata(owner, repo)` if no
      fresh `RepoConfidenceProfile` exists
   c. Build `ContributorGitHubProfile` from snapshot
   d. Assign `RepoConfidenceTier` from repo metadata (or read admin override)
   e. Compute α/β:
      ```
      α_raw = mergedCount
      β_raw = closedCount
      multiplier = tier.confidenceMultiplier()
      α_seeded = α_raw * multiplier
      β_seeded = β_raw * multiplier
      ```
   f. Call `TrustBootstrapSource.seed(actorId, PR_CONTRIBUTION, α_seeded, β_seeded)`
   g. Persist `ContributorGitHubProfile` and `RepoConfidenceProfile`
5. Next PR from this contributor: cache is fresh, score reflects GitHub history

### Incremental refresh

When a PR arrives and the cached profile is stale (per adaptive TTL):

1. Fire async refresh event
2. `fetchHistory(login, repo, profile.lastRefreshAt())` — incremental, new PRs only
3. Update aggregate counts: add new merged/closed/force-push counts
4. Recompute α/β and re-seed via `TrustBootstrapSource`
5. Update `lastRefreshAt` and potentially promote `CacheMaturity`

### Adaptive TTL

| Maturity | Observations | Stale after | Rationale |
|----------|-------------|-------------|-----------|
| COLD | 0-5 | Every PR | Each data point significantly changes the picture |
| WARM | 5-20 | >1 day | New data still moves the needle |
| HOT | 20-50 | >3 days | Scores are stable |
| MATURE | 50+ | >7 days | Large sample, very stable |

### Cache expiry

Configurable max age (default 30 days). If fully expired (contributor absent
for months), rebuild from scratch on next PR — same as initial bootstrap.

---

## Persistence

### JPA entities

```java
// app/ — JPA entity
@Entity
@Table(name = "contributor_github_profile",
       uniqueConstraints = @UniqueConstraint(columnNames = {"login", "repo"}))
public class ContributorGitHubProfileEntity {
    @Id @GeneratedValue UUID id;
    @Column(nullable = false) String login;
    @Column(nullable = false) String repo;
    @Column(nullable = false) String actorId;  // deterministic UUID string
    int mergedCount;
    int closedCount;
    int totalForcePushes;
    int observationCount;
    Instant lastRefreshAt;
    @Enumerated(EnumType.STRING) CacheMaturity maturity;
}

@Entity
@Table(name = "repo_confidence_profile",
       uniqueConstraints = @UniqueConstraint(columnNames = {"repo"}))
public class RepoConfidenceProfileEntity {
    @Id @GeneratedValue UUID id;
    @Column(nullable = false) String repo;
    int starCount;
    int contributorCount;
    int pullRequestCount;
    Instant createdAt;
    Instant lastPushedAt;
    @Enumerated(EnumType.STRING) RepoConfidenceTier tier;
    boolean adminOverride;
    Instant lastRefreshAt;
}
```

### Flyway migrations

Two new tables in the devtown domain range (V1-V999). H2 MODE=PostgreSQL
compatible.

---

## UI integration

Extend `GovernanceQueryService.ContributorDetail` with GitHub intelligence:

```java
public record ContributorDetail(
    // ... existing fields ...
    GitHubIntelligence githubIntelligence  // nullable — absent if no profile
) {}

public record GitHubIntelligence(
    int mergedCount,
    int closedCount,
    int forcePushCount,
    double mergeRatio,              // mergedCount / observationCount
    String repoConfidenceTier,      // HIGH / MEDIUM / LOW
    String cacheMaturity,           // COLD / WARM / HOT / MATURE
    Instant lastRefreshed,
    boolean bootstrapped,           // true if TrustBootstrapSource was seeded
    String bootstrapSummary         // "bootstrapped from 47 PRs in casehubio/engine"
) {}
```

The frontend extends the existing contributor detail panel with a
"GitHub Intelligence" section showing these fields. No new navigation
target — the data appears inline in the existing contributor view.

---

## Error handling

- **GitHub API failure during bootstrap:** Bootstrap event is retried with
  exponential backoff (3 attempts). If all retries fail, the contributor
  remains in TRIAGE with no cached profile. Next PR triggers another
  bootstrap attempt.
- **GitHub API rate limit:** Async bootstrap respects rate-limit headers.
  If rate-limited, delays until reset. Does not affect PR intake path.
- **Stale cache during API outage:** If refresh fails, the existing cached
  profile remains valid (stale data is better than no data). Classification
  uses the last-known score.
- **GitHub login rename:** Old and new logins produce different actor IDs.
  The old profile ages out via cache expiry. The new login triggers a fresh
  bootstrap. No data corruption — two independent profiles, one becomes
  inactive.

---

## Testing strategy

### Unit tests (domain/)

- `CacheMaturity.isStale()` — boundary conditions for each tier
- `RepoConfidenceTier` assignment from metadata thresholds
- α/β computation from profile + tier multiplier
- Identity mapping determinism (same login → same UUID)

### Integration tests (app/)

- Bootstrap flow: mock `ContributorHistoryClient`, verify
  `TrustBootstrapSource.seed()` called with correct α/β
- Incremental refresh: verify delta fetch since `lastRefreshAt`
- Adaptive TTL: verify refresh triggers at correct maturity boundaries
- Cache expiry: verify full rebuild after max-age expiry
- JPA entity round-trip: persist and retrieve profiles
- Async event: verify `BootstrapContributorEvent` fires and completes

### Edge cases

- Contributor with zero PRs (new account) — profile created with zero counts,
  no bootstrap seed (α/β both 0 has no effect on prior)
- Contributor with only merged PRs — β=0, α=N×multiplier
- Repo with admin-overridden tier — verify override respected
- Concurrent bootstrap for same contributor — idempotent (upsert)

---

## Scope and non-goals

### In scope

- Cache schema (two JPA entities)
- Historical bootstrap via GitHub API (merge ratio + revision count)
- Incremental refresh with adaptive TTL
- Repo confidence tiers (HIGH/MEDIUM/LOW) with admin override
- TrustBootstrapSource integration for seeding Bayesian Beta prior
- UI: extend ContributorDetail with GitHub intelligence fields
- Async bootstrap trigger from PR intake

### Not in scope (follow-up candidates)

- Additional signals (CI pass rate, review comment volume, time-to-approval)
  — CI pass rate is easiest, API already exists (D1)
- Org-level score fallback when per-repo observations are insufficient (D2)
- Vouching system (contributor trust proposal §Vouching)
- Score decay for dormant accounts
- Cross-deployment trust export/import (P2.1)
- Bulk pre-seed on deployment

---

## References

- [ContributorAttestationPolicy.java](app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java) — existing attestation mapping
- [ContributorIntakePolicy.java](domain/src/main/java/io/casehub/devtown/domain/ContributorIntakePolicy.java) — lane classification logic
- [ContributorTrustDimension.java](domain/src/main/java/io/casehub/devtown/domain/ContributorTrustDimension.java) — MERGE_RATE, FIRST_ATTEMPT_QUALITY
- [GitHubPullRequestApi.java](github/src/main/java/io/casehub/devtown/github/GitHubPullRequestApi.java) — existing REST client
- [GitHubChecksApi.java](github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java) — existing check-runs API (CI pass rate future signal)
- [GovernanceQueryService.java:144-154](app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java) — ContributorFleetEntry, ContributorDetail
- [2026-05-13-contributor-trust-open-source.md](docs/specs/2026-05-13-contributor-trust-open-source.md) — contributor trust proposal
- TrustBootstrapSource SPI (casehub-ledger-api) — platform SPI for seeding Beta priors
- TrustWeightedImplementationRoutingStrategy (casehub-ledger) — trust-weighted routing
- [GE-20260530-fcc6c3] — ConcurrentHashMap TTL cache stale-entry gotcha
- [GE-20260607-3defda] — Per-actor computation cache with event-driven invalidation
- casehubio/devtown#202 — issue
