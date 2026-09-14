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
per-repo signal data with adaptive TTL. Import historical scores eagerly
via `TrustImportService` and serve them on-demand via
`TrustBootstrapSource` (pull SPI). Continue using
`ContributorAttestationPolicy` for live PR observations. Classify repo
quality into confidence tiers that modulate how much weight historical data
carries.

---

## Architecture

### Data flow

```
PR webhook arrives
  → PrPayload.contributor() / PrPayload.contributorNumericId()
  → is contributor profile cached and fresh?
      YES → no action (live attestation pipeline handles scoring)
      NO  → classify PR as TRIAGE immediately (safe default)
          → fire async CDI event: BootstrapContributorEvent(login, numericId, repo)
          → ContributorBootstrapWriter observes the event (app/):
              → deduplicate (skip if bootstrap already in-flight for this login)
              → ContributorHistoryClient.fetchHistory(login, repo, Instant.MIN)
              → ContributorHistoryClient.fetchRepoMetadata(owner, repo)
              → build ContributorGitHubProfile (aggregate cache)
              → build/update RepoConfidenceProfile (tier assignment)
              → compute α/β from merge/reject ratio × repo tier multiplier
              → construct TrustExportPayload with capability + dimension scores
              → TrustImportService.importTrust(payload) — eager import
              → persist ContributorGitHubProfile and RepoConfidenceProfile
          → next PR from this contributor uses imported score

TrustScoreJob (24h schedule) discovers new actors
  → TrustBootstrapService.bootstrapIfNew(newActorIds)
  → TrustBootstrapSource.fetchPriorTrust(actorId) — pull SPI
      → devtown implementation reads cached ContributorGitHubProfile
      → returns TrustExportPayload (or empty if no cached profile)
      → TrustImportService.importTrust(payload) — idempotent

Full end-to-end path (bootstrap → intake classification):
  GitHub PR history
  → ContributorHistoryClient.fetchHistory()
  → ContributorGitHubProfile (cached)
  → TrustImportService.importTrust(TrustExportPayload)
  → ActorTrustScore (materialized: capability + dimension scores)
  → TrustGateService.allCapabilityScores(actorId)
  → ContributorIntakePolicy.classify(score, observations)
  → IntakeClassification(lane=FAST_TRACK|STANDARD|TRIAGE)
```

### Module placement

Following the hexagonal port/adapter pattern established by
`CiStatusClient`/`GitHubCiStatusClient`:

| Component | Module | Rationale |
|-----------|--------|-----------|
| `ContributorHistoryClient` (SPI interface) | `domain/` | Port — service depends on this, not the adapter |
| `ContributorGitHubProfile` (domain record) | `domain/` | Pure Java, no framework deps |
| `RepoConfidenceTier` (enum) | `domain/` | HIGH / MEDIUM / LOW with confidence multipliers |
| `BootstrapScoreComputer` (pure computation) | `domain/` | Computes α/β and dimension scores from profile + tier — no framework deps |
| `GitHubContributorHistoryClient` | `github/` | Adapter — implements the SPI via GitHub REST API |
| `GitHubRepoApi` (REST client interface) | `github/` | New interface for repo metadata endpoint |
| `ContributorGitHubProfileEntity` (JPA) | `app/` | Persistence |
| `RepoConfidenceProfileEntity` (JPA) | `app/` | Persistence |
| `ContributorBootstrapWriter` | `app/` | CDI observer, persists profiles, calls `TrustImportService` — follows `ContributorOutcomeLedgerWriter` pattern |
| `DevtownTrustBootstrapSource` | `app/` | Implements `TrustBootstrapSource` SPI, reads from cached profiles — displaces `NoOpTrustBootstrapSource @DefaultBean` via CDI |
| `NoOpContributorHistoryClient` (`@DefaultBean`) | `app/` | Fallback when GitHub not configured |
| `BootstrapContributorEvent` (CDI event) | `review/` | Event type — domain integration boundary |

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

3. **α/β computation** — for v1, `BootstrapScoreComputer` in domain/ computes
   capability-level α/β (α = mergedCount × tier multiplier,
   β = closedCount × tier multiplier) and dimension-level scores
   (MERGE_RATE = mergedCount / observationCount). When signal 3+ arrives,
   extract into a `SignalWeightPolicy` that maps signal values to Bayesian
   update parameters. New signals register their weight contribution.

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
    String login,             // GitHub username (natural key)
    long contributorNumericId, // GitHub numeric user ID
    String repo,              // owner/repo (natural key)
    String actorId,           // "github-id:" + contributorNumericId (matches attestation pipeline)
    int mergedCount,          // PRs merged
    int closedCount,          // PRs closed without merge
    int observationCount,     // mergedCount + closedCount
    Instant lastRefreshAt,    // last successful API fetch
    CacheMaturity maturity    // computed from observationCount
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
    long contributorNumericId,  // GitHub user.id from PR listing
    String repo,
    int mergedCount,
    int closedCount,
    Instant oldestPrAt,
    Instant newestPrAt
    // Future signals: int forcePushCount (needs per-PR timeline API),
    //                 int ciPassCount, int ciFailCount,
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

GitHub numeric ID → devtown actor ID, matching the existing attestation
pipeline identity scheme established in `PrReviewCaseService.closePr()`:

```java
public static String actorIdFromGitHubNumericId(long numericId) {
    return "github-id:" + numericId;
}
```

This produces the same `actorId` format as the live attestation pipeline
(`"github-id:" + event.senderId()` in `PrLifecycleAttestationObserver`),
ensuring that bootstrapped scores and live attestation scores combine
under a single identity. The numeric ID is stable across GitHub login
renames. `PrPayload.contributorNumericId()` provides the value at PR
intake time; the GitHub PR listing API returns `user.id` for each PR.

---

## GitHub API integration

**Note on issue #202 deliverable 5:** The issue references "casehub-connectors
GitHub adapter." No GitHub-specific adapter exists in casehub-connectors (which
provides webhook, Slack, Discord, email, etc.). devtown already has a `github/`
module with 4 REST client interfaces (`GitHubPullRequestApi`, `GitHubChecksApi`,
`GitHubPayloadMapper`, `GitHubCiStatusClient`). This spec follows the
established pattern. Issue #202 should be updated to reflect this.

**Force-push data deferred to follow-up:** The GitHub PR list endpoint
(`/repos/{owner}/{repo}/pulls`) does not include force-push events. Force-push
data requires the timeline endpoint (`/repos/{owner}/{repo}/issues/{pr}/timeline`),
a separate API call per PR. For a contributor with 100 PRs, this means 100
additional API calls. Per decision D1 ("proves the pipeline end-to-end with
minimal API surface"), force-push is deferred. The `FIRST_ATTEMPT_QUALITY`
dimension will only be populated by live attestations, not by bootstrap.

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
   this login+repo). Source: `PrPayload.contributor()` (login) and
   `PrPayload.contributorNumericId()` (numeric ID).
2. PR classified as TRIAGE immediately (safe default)
3. `BootstrapContributorEvent(login, contributorNumericId, repo)` fired as
   async CDI event
4. `ContributorBootstrapWriter` (app/) observes the event:
   a. Deduplication check — if bootstrap already in-flight for this login
      (tracked via `ConcurrentHashMap<String, CompletableFuture>`), skip.
      Coalesce concurrent requests: return existing future, don't fire a
      second API fetch.
   b. Call `ContributorHistoryClient.fetchHistory(login, repo, Instant.MIN)`
      — full history
   c. Call `ContributorHistoryClient.fetchRepoMetadata(owner, repo)` if no
      fresh `RepoConfidenceProfile` exists
   d. Build `ContributorGitHubProfile` from snapshot using
      `actorId = "github-id:" + contributorNumericId`
   e. Assign `RepoConfidenceTier` from repo metadata (or read admin override)
   f. Compute scores via `BootstrapScoreComputer` (domain/):
      ```
      α_raw = mergedCount
      β_raw = closedCount
      multiplier = tier.confidenceMultiplier()
      α_seeded = α_raw * multiplier
      β_seeded = β_raw * multiplier

      // Dimension-level approximation
      mergeRate = observationCount > 0
          ? (double) mergedCount / observationCount
          : 0.5  // uninformative prior
      ```
   g. Construct `TrustExportPayload` containing:
      - `CapabilityScoreExport(PR_CONTRIBUTION, α_seeded, β_seeded, ...)`
      - `DimensionScoreExport(MERGE_RATE, mergeRate, observationCount, ...)`
      - `CapabilityDimensionScoreExport(PR_CONTRIBUTION, MERGE_RATE, mergeRate, observationCount, ...)`
      - No `FIRST_ATTEMPT_QUALITY` — cannot be approximated from PR list
        data alone (requires per-PR timeline API; deferred to follow-up)
   h. Call `TrustImportService.importTrust(payload)` — eager import,
      writes `ActorTrustScore` records immediately. `JpaTrustImportService`
      only seeds actors not already present (idempotent).
   i. Persist `ContributorGitHubProfile` and `RepoConfidenceProfile`
   j. Remove login from in-flight dedup map
5. Next PR from this contributor: cache is fresh, `TrustGateService` returns
   the imported score, `ContributorIntakePolicy.classify()` uses it for
   lane assignment

### Incremental refresh

When a PR arrives and the cached profile is stale (per adaptive TTL):

1. Fire async refresh event (same dedup applies)
2. `fetchHistory(login, repo, profile.lastRefreshAt())` — incremental, new PRs only
3. Update aggregate counts: add new merged/closed counts
4. Recompute α/β and dimension scores via `BootstrapScoreComputer`
5. Construct `TrustExportPayload` and call `TrustImportService.importTrust()`
6. Update `lastRefreshAt` and potentially promote `CacheMaturity`

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
    @Column(nullable = false) long contributorNumericId;
    @Column(nullable = false) String repo;
    @Column(nullable = false) String actorId;  // "github-id:" + contributorNumericId
    int mergedCount;
    int closedCount;
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
    double mergeRatio,              // mergedCount / observationCount
    String repoConfidenceTier,      // HIGH / MEDIUM / LOW
    String cacheMaturity,           // COLD / WARM / HOT / MATURE
    Instant lastRefreshed,
    boolean bootstrapped,           // true if trust was imported from GitHub history
    String bootstrapSummary         // "bootstrapped from 47 PRs in casehubio/engine"
) {}
```

The frontend extends the existing contributor detail panel with a
"GitHub Intelligence" section showing these fields. No new navigation
target — the data appears inline in the existing contributor view.

**Known limitation:** `TrustQueryService.trustTrend()` currently returns
empty (`TrustScoreSnapshot` removed from ledger — replacement entity
pending). Bootstrapped contributors show GitHub intelligence fields
but have no trust trend history until `TrustScoreSnapshot` is restored.
This affects all contributors, not just bootstrapped ones.

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
- `BootstrapScoreComputer` — α/β and dimension score computation from
  profile + tier multiplier
- Identity mapping: `actorIdFromGitHubNumericId()` produces `"github-id:N"`
  format matching existing attestation pipeline

### Integration tests (app/)

- Bootstrap flow: mock `ContributorHistoryClient`, verify
  `TrustImportService.importTrust()` called with correct `TrustExportPayload`
  containing capability, dimension, and capability-dimension scores
- `DevtownTrustBootstrapSource.fetchPriorTrust()` — returns payload from
  cached profile, empty if no profile exists
- Incremental refresh: verify delta fetch since `lastRefreshAt`
- Adaptive TTL: verify refresh triggers at correct maturity boundaries
- Cache expiry: verify full rebuild after max-age expiry
- JPA entity round-trip: persist and retrieve profiles
- Async event: verify `BootstrapContributorEvent` fires and completes
- Deduplication: concurrent events for same login coalesced into single
  API fetch

### Edge cases

- Contributor with zero PRs (new account) — profile created with zero counts,
  no import (α/β both 0 has no effect on prior)
- Contributor with only merged PRs — β=0, α=N×multiplier, mergeRate=1.0
- Repo with admin-overridden tier — verify override respected
- Concurrent bootstrap for same contributor — dedup prevents duplicate API
  calls; persistence uses idempotent upsert
- Identity consistency: bootstrapped actorId matches actorId produced by
  `PrReviewCaseService.closePr()` for same contributor

---

## Scope and non-goals

### In scope

- Cache schema (two JPA entities)
- Historical bootstrap via GitHub API (merge ratio)
- Eager trust import via `TrustImportService` for immediate score availability
- `TrustBootstrapSource` pull SPI implementation for trust job integration
- Incremental refresh with adaptive TTL
- Repo confidence tiers (HIGH/MEDIUM/LOW) with admin override
- Dimension-level bootstrap scores (MERGE_RATE)
- UI: extend ContributorDetail with GitHub intelligence fields
- Async bootstrap trigger from PR intake with deduplication
- Bootstrap event deduplication (concurrent PR requests coalesced)

### Not in scope (follow-up candidates)

Each deferred item is tracked as a GitHub issue (see issue references below):

- Force-push count / FIRST_ATTEMPT_QUALITY bootstrap — requires per-PR
  timeline API (`/repos/{owner}/{repo}/issues/{pr}/timeline`), significant
  API cost (D3 — casehubio/devtown#TBD)
- Additional signals (CI pass rate, review comment volume, time-to-approval)
  — CI pass rate is easiest, API already exists (D1 — casehubio/devtown#TBD)
- Org-level score fallback when per-repo observations are insufficient
  (D2 — casehubio/devtown#TBD)
- Vouching system (contributor trust proposal §Vouching — casehubio/devtown#TBD)
- Score decay for dormant accounts (casehubio/devtown#TBD)
- Cross-deployment trust export/import (P2.1 — casehubio/devtown#TBD)
- Bulk pre-seed on deployment (casehubio/devtown#TBD)

---

## References

- [ContributorAttestationPolicy.java](app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java) — existing attestation mapping
- [ContributorOutcomeLedgerWriter.java](app/src/main/java/io/casehub/devtown/app/ledger/ContributorOutcomeLedgerWriter.java) — app-tier writer pattern (model for ContributorBootstrapWriter)
- [PrReviewCaseService.java:194](app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java) — `"github-id:" + numericId` identity scheme
- [ContributorIntakePolicy.java](domain/src/main/java/io/casehub/devtown/domain/ContributorIntakePolicy.java) — lane classification logic
- [ContributorTrustDimension.java](domain/src/main/java/io/casehub/devtown/domain/ContributorTrustDimension.java) — MERGE_RATE, FIRST_ATTEMPT_QUALITY
- [PrPayload.java](review/src/main/java/io/casehub/devtown/review/PrPayload.java) — `contributor()` (login), `contributorNumericId()` (ID)
- [GitHubPayloadMapper.java](github/src/main/java/io/casehub/devtown/github/GitHubPayloadMapper.java) — maps webhook to PrPayload
- [GitHubPullRequestApi.java](github/src/main/java/io/casehub/devtown/github/GitHubPullRequestApi.java) — existing REST client
- [GitHubChecksApi.java](github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java) — existing check-runs API (CI pass rate future signal)
- [GovernanceQueryService.java](app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java) — ContributorFleetEntry, ContributorDetail
- [2026-05-13-contributor-trust-open-source.md](docs/specs/2026-05-13-contributor-trust-open-source.md) — contributor trust proposal
- TrustBootstrapSource SPI (casehub-ledger) — pull SPI: `fetchPriorTrust(actorId) → Optional<TrustExportPayload>`
- TrustImportService (casehub-ledger) — `importTrust(TrustExportPayload)` writes ActorTrustScore records
- TrustBootstrapService (casehub-ledger) — called by TrustScoreJob to bootstrap new actors
- JpaTrustImportService (casehub-ledger) — seeds capability, dimension, and capability-dimension scores
- TrustWeightedImplementationRoutingStrategy (casehub-ledger) — trust-weighted routing
- [GE-20260530-fcc6c3] — ConcurrentHashMap TTL cache stale-entry gotcha
- [GE-20260607-3defda] — Per-actor computation cache with event-driven invalidation
- casehubio/devtown#202 — issue
