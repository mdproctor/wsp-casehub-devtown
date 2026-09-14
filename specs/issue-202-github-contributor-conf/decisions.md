# Decisions — #202 GitHub-Derived Contributor Confidence

## D1: Signal scope — core 2 signals first

**Choice:** Merge ratio + revision count (force-push count) only
**Alternatives:**
- Core 3 (add CI pass rate) — useful but requires check-runs API, separate endpoint, more complex data model
- All 5 signals — full signal set but significantly more API surface, pagination complexity, and rate-limit pressure
**Rationale:** These two signals map directly to existing trust dimensions (MERGE_RATE, FIRST_ATTEMPT_QUALITY) and are computable from the PR list API alone. Proves the pipeline end-to-end with minimal API surface.
**Trade-offs:** Missing review comment volume, CI pass rate, and time-to-approval signals. Can be added incrementally once the pipeline is proven.
**Sources:** ContributorTrustDimension.java:3-6, ContributorAttestationPolicy.java:33-66, issue #202 body
**Exploration:** quick
**Status:** captured

## D2: Score scope — per-repo

**Choice:** Per-repo scoping (separate contributor profile per repository)
**Alternatives:**
- Org-scoped — aggregated across repos. More forgiving but conflates different contribution contexts
- Layered (per-repo primary, org fallback) — most accurate but significantly more complex query and cache logic
**Rationale:** A contributor may have different quality in different repos (docs contributor vs core). Simpler bootstrap — each is one repo's PR history.
**Trade-offs:** A contributor with 500 PRs in repo A starts at zero in repo B. Acceptable for initial implementation.
**Sources:** Issue #202 body ("per-repo scoping"), ContributorIntakePolicy.java
**Exploration:** quick
**Status:** captured

## D3: Integration model — attestations with repo-modulated confidence

**Choice:** GitHub historical PRs generate attestation intents via the existing ContributorAttestationPolicy, feeding directly into the ledger's Bayesian Beta model. Historical attestations carry a confidence weight modulated by a repo confidence factor derived from observable repo metrics (stars, contributors, age, liveness). Admins can override the repo confidence factor for specific repos.
**Alternatives:**
- Parallel TrustScoreSource — separate score blended at classification time. Clean separation but creates two scoring paths
- Cache replaces ledger for cold contributors — simple but creates a hard transition point
- Pre-seed Bayesian prior only — mathematically clean but doesn't capture rejected PRs as individual observations
**Rationale:** GitHub is a trusted data source. Both merged and rejected PRs are legitimate observations. Using the existing attestation pipeline maintains a single scoring path. Deterministic entry IDs (already in ContributorAttestationPolicy) make historical attestations idempotent. The repo confidence factor addresses the quality gap between devtown-observed PRs (with reviewer scores) and GitHub-historical PRs (merge/reject only).
**Trade-offs:** Historical attestations are permanent ledger records. The repo confidence factor adds complexity but is necessary — a merge in a 10k-star repo means something different from a merge in a personal project.
**Sources:** ContributorAttestationPolicy.java:81-84 (deterministic entry IDs), TrustScoreSource SPI, 2026-05-13-contributor-trust-open-source.md, GE-20260607-3defda (per-actor cache pattern)
**Exploration:** deep-analysis
**Status:** captured

## D4: Repo confidence — composite score with admin override

**Choice:** Compute a composite repo confidence factor from normalized metrics (stars, contributor count, age, liveness), weighted-averaged into a single 0-1 factor. Admin override sets the factor directly for specific repos via preferences. Third-party repos dampened by default; first-party repos start at a higher baseline.
**Alternatives:**
- Tiered classification (HIGH/MEDIUM/LOW) — simpler to reason about but loses nuance between 50-star and 5000-star repos
**Rationale:** Continuous score preserves the signal gradient. Admin override handles cases where heuristics are wrong. Per-project preference via existing PreferenceKey mechanism.
**Trade-offs:** Composite score formula needs tuning — initial weights are a best guess. But the admin override provides an escape hatch.
**Sources:** ContributorIntakePreferenceKeys.java (preference pattern), GitHub REST API repo endpoint
**Exploration:** quick
**Status:** captured

## D5: Cache entity — single aggregate entity

**Choice:** `ContributorGitHubProfile` entity with computed aggregates stored directly (mergedCount, closedCount, totalForcePushes, observationCount, lastRefreshAt, cacheMaturity). On refresh, fetch new PRs since lastRefreshAt, update counts, recompute. The entity is the cache AND the aggregate.
**Alternatives:**
- Raw event log + materialized view — store individual PR records, aggregate on read. Full history retained but unnecessary when GitHub API is the authoritative source for rebuilds.
**Rationale:** The issue spec explicitly says "it's a cache, so it can always be rebuilt from the GitHub API if needed." Storing raw events mirrors GitHub's database unnecessarily. Aggregate entity is the right abstraction for a cache with adaptive TTL.
**Trade-offs:** Adding new signals later means adding columns. Acceptable — it's a cache entity, not a core domain model.
**Sources:** Issue #202 body (cache structure section), GE-20260530-fcc6c3 (TTL cache gotcha)
**Exploration:** quick
**Status:** captured

## D6: GitHub API integration — extend existing REST client

**Choice:** Add methods to existing `GitHubPullRequestApi` interface: author-filtered PR listing + repo metadata endpoint. Rate-limit handling as a shared response filter.
**Alternatives:**
- Separate `GitHubContributorApi` REST client — cleaner separation but duplicated config and rate-limit handling
**Rationale:** The `github` module is already the single surface for GitHub API interaction. These are standard REST endpoints. Consistent with existing patterns.
**Trade-offs:** REST client interface grows, but it's the canonical GitHub API surface.
**Sources:** GitHubPullRequestApi.java
**Exploration:** quick
**Status:** captured

## D7: Module placement — extend existing modules

**Choice:** Domain types (`ContributorGitHubProfile`, `RepoConfidenceProfile`) in `domain/`. Service logic (`ContributorIntelligenceService`) in `review/`. JPA entity, REST client extension, CDI wiring in `app/`.
**Alternatives:**
- New `intelligence` module — clean boundary but adds a module for an enrichment concern
**Rationale:** This is an extension of existing contributor trust, not a new subsystem. Follows the ContributorIntakePolicy (domain) + ContributorAttestationPolicy (app) pattern.
**Trade-offs:** `review/` module grows. Acceptable — it's the integration logic module.
**Sources:** Module tier structure in CLAUDE.md, ContributorIntakePolicy.java (domain/), ContributorAttestationPolicy.java (app/)
**Exploration:** quick
**Status:** captured

## D8: Bootstrap trigger — on-demand at PR intake

**Choice:** When a PR arrives and the contributor has no cached profile (or cache is stale per adaptive TTL), fetch their history before classification. Lazy — only pays API cost for contributors who actually submit.
**Alternatives:**
- Bulk pre-seed on deployment — no first-PR latency but large upfront API cost for contributors who may never submit again
- Both — most complete but more complex orchestration
**Rationale:** Lazy bootstrap avoids wasted API calls. First PR has a small latency hit (GitHub API call) but subsequent ones use the cache. Matches the issue spec's "on first encounter" language.
**Trade-offs:** First PR from each contributor has a latency hit for the API call. Acceptable — it's a one-time cost per contributor.
**Sources:** Issue #202 body (historical bootstrap section)
**Exploration:** quick
**Status:** captured

## D9: UI placement — extend existing contributor views

**Choice:** Add GitHub signal data (merge ratio, revision count, repo confidence, data freshness) to the existing `ContributorDetail` record in `GovernanceQueryService`. Extend classification reason to include "bootstrapped from N PRs across M repos."
**Alternatives:**
- Separate intelligence panel — more detailed but adds UI complexity and a separate navigation target
**Rationale:** The governance UI already has ContributorFleetEntry and ContributorDetail views. Adding fields to existing records avoids navigation changes and keeps the contributor view as a single coherent panel.
**Trade-offs:** ContributorDetail record grows. Acceptable — it's a query projection, not a domain entity.
**Sources:** GovernanceQueryService.java:144-154 (ContributorFleetEntry, ContributorDetail)
**Exploration:** quick
**Status:** captured
