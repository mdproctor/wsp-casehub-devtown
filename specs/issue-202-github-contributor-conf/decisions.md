# Decisions — #202 GitHub-Derived Contributor Confidence

## D1: Signal scope — core 2 signals first

**Choice:** Merge ratio + revision count (force-push count) only
**Alternatives:**
- Core 3 (add CI pass rate) — infrastructure exists (`GitHubChecksApi`, `CiStatusClient` SPI) so this is a scope choice, not a technical constraint. Excluded to prove the pipeline end-to-end first.
- All 5 signals — full signal set but significantly more pagination complexity and rate-limit pressure
**Rationale:** These two signals map directly to existing trust dimensions (MERGE_RATE, FIRST_ATTEMPT_QUALITY) and are computable from the PR list API alone. Proves the pipeline end-to-end with minimal API surface.
**Trade-offs:** Missing review comment volume, CI pass rate, and time-to-approval signals. CI pass rate is the easiest to add next — the API surface already exists.
**Sources:** ContributorTrustDimension.java:3-6, ContributorAttestationPolicy.java:33-66, GitHubChecksApi.java, issue #202 body
**Exploration:** quick
**Status:** revised (R1-02: corrected rationale — CI pass rate API already exists)

## D2: Score scope — per-repo

**Choice:** Per-repo scoping (separate contributor profile per repository)
**Alternatives:**
- Org-scoped — aggregated across repos. More forgiving but conflates different contribution contexts
- Layered (per-repo primary, org fallback) — most accurate; query is straightforward (check per-repo first, aggregate org-level if observations < minimum). Follow-up candidate.
**Rationale:** A contributor may have different quality in different repos (docs contributor vs core). Simpler bootstrap — each is one repo's PR history.
**Trade-offs:** A contributor with 500 PRs in repo A starts at zero in repo B — partially recreates the cold-start problem this feature solves. Acceptable for v1; org-level fallback is a natural follow-up that doesn't require structural changes.
**Sources:** Issue #202 body ("per-repo scoping"), ContributorIntakePolicy.java
**Exploration:** quick
**Status:** revised (R1-03: acknowledged cold-start tension, noted org-fallback as follow-up)

## D3: Integration model — hybrid TrustBootstrapSource + live attestations

**Choice:** Historical GitHub PR data seeds Bayesian Beta parameters (α/β) via the existing `TrustBootstrapSource` SPI in casehub-ledger. This creates O(1) state per contributor (the seeded prior) rather than O(N) permanent ledger entries per historical PR. Live PR outcomes continue to generate individual attestation intents via `ContributorAttestationPolicy`, producing tamper-evident ledger records. Historical data seeds the prior; live data updates it. The seeded α/β are modulated by a repo confidence tier (D4).
**Alternatives:**
- All-attestation (original D3) — both historical and live PRs enter as attestation intents. Single path, deterministic IDs prevent duplicates. But creates permanent ledger entries for events devtown never observed. Storage cost and ledger noise.
- Parallel TrustScoreSource — separate score blended at classification time. Clean separation but creates two scoring paths.
- TrustBootstrapSource only — even live PRs update bootstrap source. Simplest but loses tamper-evident individual observation records for live PRs.
**Rationale:** GitHub is a trusted data source. Both merged and rejected PRs carry signal — `CapabilityScoreExport` carries both alpha (successes) and beta (failures), so rejected PRs ARE represented in the seeded prior. Historical data belongs in the prior, not in the tamper-evident audit trail. Live observations belong in the audit trail. The platform already provides `TrustBootstrapSource` for exactly this purpose.
**Trade-offs:** Two integration paths (bootstrap vs live). But the paths serve different purposes (prior vs observations) and use existing platform SPIs. The separation is architecturally correct.
**Depends on:** D4 (repo confidence tier modulates the bootstrap α/β)
**Sources:** TrustBootstrapSource SPI (casehub-ledger), CapabilityScoreExport, ContributorAttestationPolicy.java:81-84, 2026-05-13-contributor-trust-open-source.md, GE-20260607-3defda
**Exploration:** deep-analysis
**Status:** revised (R1-06: replaced all-attestation approach with TrustBootstrapSource hybrid)

## D4: Repo confidence — tiered classification with admin override

**Choice:** Three tiers: HIGH (first-party repos or admin-designated), MEDIUM (meets minimum thresholds: age >1 year, >5 contributors, >50 PRs), LOW (everything else — personal projects, new repos, unknown). Each tier maps to a fixed confidence multiplier applied to the bootstrap α/β. Admin override assigns a repo to a tier via preferences.
**Alternatives:**
- Composite score (original D4) — continuous 0-1 factor from normalized metrics. More granular but the input signals are unreliable: stars correlate with visibility not quality, contributor count is ambiguous, mature stable libraries look "dead" by liveness. Over-engineering a problem the admin override makes redundant.
- Deferred heuristics — admin-only assignment, auto-classification later. Safe but doesn't scale.
**Rationale:** Tiers are easy to reason about, explain to users, and configure. The tier criteria (age, contributors, PR count) are less susceptible to gaming than stars or liveness. Admin override collapses into the same mechanism: assign a repo to a tier.
**Trade-offs:** Loses nuance between 50-star and 5000-star repos. But that nuance was illusory — stars don't correlate with review quality.
**Sources:** ContributorIntakePreferenceKeys.java (preference pattern), R1-07 review finding
**Exploration:** quick
**Status:** revised (R1-07: replaced composite score with tiered classification)

## D5: Cache entity — single aggregate entity

**Choice:** `ContributorGitHubProfile` entity with computed aggregates stored directly (mergedCount, closedCount, totalForcePushes, observationCount, lastRefreshAt, cacheMaturity). On refresh, fetch new PRs since lastRefreshAt, update counts, recompute. The entity is the cache AND the aggregate.
**Alternatives:**
- Raw event log + materialized view — store individual PR records, aggregate on read. Full history retained but unnecessary when GitHub API is the authoritative source for rebuilds.
- Hybrid (columns for aggregates, JSON for extensible signal details) — avoids migrations for new signals. Premature for 2 signals; reconsider when adding signal 3+.
**Rationale:** The issue spec explicitly says "it's a cache, so it can always be rebuilt from the GitHub API if needed." Storing raw events mirrors GitHub's database unnecessarily. Aggregate entity is the right abstraction for a cache with adaptive TTL.
**Trade-offs:** Adding new signals later means adding columns. Acceptable for 2 signals — it's a cache entity, not a core domain model. If signal count grows past 3-4, migrate to JSON column for signal details.
**Sources:** Issue #202 body (cache structure section), GE-20260530-fcc6c3 (TTL cache gotcha)
**Exploration:** quick
**Status:** revised (R1-08: acknowledged JSON column as future option)

## D6: GitHub API integration — split across existing pattern

**Choice:** Author-filtered PR listing added to existing `GitHubPullRequestApi` (it IS a pull request operation). Repo metadata (stars, contributor count, age) goes in a new `GitHubRepoApi` interface, following the established multi-interface pattern (`GitHubPullRequestApi`, `GitHubChecksApi`, `GitHubGitApi`, `GitHubMergeApi`).
**Alternatives:**
- Everything in `GitHubPullRequestApi` (original D6) — repo metadata is not a PR operation, breaks naming contract
- Separate `GitHubContributorApi` — unnecessary, author-filtered PR listing is a PR operation
**Rationale:** The `github/` module already uses one interface per API domain. Repo metadata is a distinct API domain. Rate-limit handling is a shared response filter across all interfaces.
**Trade-offs:** One new interface. Minimal — it follows the existing pattern exactly.
**Sources:** GitHubPullRequestApi.java, GitHubChecksApi.java, GitHubGitApi.java, GitHubMergeApi.java
**Exploration:** quick
**Status:** revised (R1-05: split repo metadata into GitHubRepoApi)

## D7: Module placement — SPI pattern with hexagonal boundary

**Choice:** SPI interface `ContributorHistoryClient` in `domain/` (port). GitHub implementation `GitHubContributorHistoryClient` in `github/` (adapter). `@DefaultBean` NoOp in `app/`. Service logic (`ContributorIntelligenceService`) in `review/` calls the SPI, not the GitHub implementation directly. Domain records (`ContributorGitHubProfile`, `RepoConfidenceProfile`) in `domain/`. JPA entity + CDI wiring in `app/`.
**Alternatives:**
- Service in `review/` calling `github/` directly (original D7) — impossible: `review/` cannot depend on `github/` (would create cycle: `github → review → github`)
- New `intelligence` module — clean boundary but adds a module for an enrichment concern
**Rationale:** Follows the established `CiStatusClient` (domain/) → `GitHubCiStatusClient` (github/) → `NoOpCiStatusClient @DefaultBean` (app/) pattern. Hexagonal architecture — the service depends on the port, not the adapter.
**Trade-offs:** One more SPI interface in `domain/`. Minimal — it's the correct architectural pattern.
**Sources:** CiStatusClient (domain SPI pattern), GitHubCiStatusClient (github adapter), R1-01 review finding
**Exploration:** quick
**Status:** revised (R1-01: fixed module dependency violation with SPI pattern)

## D8: Bootstrap trigger — async on-demand at PR intake

**Choice:** When a PR arrives from a contributor with no cached profile: classify immediately as TRIAGE (the safe default — zero-history contributors go to TRIAGE anyway), fire an async event to bootstrap their GitHub history. When bootstrap completes, the cached profile is available for the next PR. Subsequent PRs use the cache with adaptive TTL refresh.
**Alternatives:**
- Synchronous block (original D8) — blocks PR intake on GitHub API call. First PR gets accurate classification but adds latency, vulnerable to GitHub rate limits and outages. A contributor with 500 PRs means paginated API calls taking seconds.
- Sync with timeout fallback — tries synchronous with a 2s timeout, falls through to TRIAGE on error. More complex error handling for marginal benefit.
- Bulk pre-seed on deployment — no first-PR latency but large upfront API cost for contributors who may never submit again
**Rationale:** The zero-history default is TRIAGE — exactly where a new contributor should land. Async bootstrap adds no cost to the first PR's classification (it would have been TRIAGE regardless). The system is resilient to GitHub API failures — bootstrap failure means TRIAGE continues, which is correct. GitHub API errors, rate limits, and timeouts affect background processing, not the intake path.
**Trade-offs:** First PR is always TRIAGE regardless of contributor history. Acceptable — by the time review starts, the async bootstrap has likely completed and subsequent routing decisions benefit.
**Sources:** Issue #202 body, ContributorIntakePolicy.classify() default behavior, R1-04 review finding
**Exploration:** quick
**Status:** revised (R1-04: changed from synchronous to async)

## D9: UI placement — extend existing contributor views

**Choice:** Add GitHub signal data (merge ratio, revision count, repo confidence tier, data freshness, bootstrap status) to the existing `ContributorDetail` record in `GovernanceQueryService`. Extend classification reason to include "bootstrapped from N PRs across M repos."
**Alternatives:**
- Separate intelligence panel — more detailed but adds UI complexity and a separate navigation target
**Rationale:** The governance UI already has ContributorFleetEntry and ContributorDetail views. Adding fields to existing records avoids navigation changes and keeps the contributor view as a single coherent panel.
**Trade-offs:** ContributorDetail record grows. Acceptable — it's a query projection, not a domain entity.
**Sources:** GovernanceQueryService.java:144-154 (ContributorFleetEntry, ContributorDetail)
**Exploration:** quick
**Status:** captured

## D10: Identity mapping — GitHub login to devtown actor ID

**Choice:** Map GitHub username to devtown actor ID via deterministic UUID: `UUID.nameUUIDFromBytes(("github:" + login).getBytes())`. The `ContributorGitHubProfile` stores the GitHub login as the natural key alongside the deterministic actor UUID. `TrustBootstrapSource` and `ContributorAttestationPolicy` both use this actor ID as the subject.
**Alternatives:**
- Actor registry lookup — requires a pre-populated registry mapping GitHub logins to devtown actors. More accurate for orgs that maintain identity mapping, but adds a hard dependency on registry state before bootstrap can run.
- PR-based UUID (existing fallback) — `resolveSubjectId()` falls back to `UUID.nameUUIDFromBytes("pr:" + repo + ":" + prNumber)`. Wrong granularity — generates a UUID per PR, not per contributor.
**Rationale:** Deterministic UUID from login is simple, stable, and consistent. If the same GitHub user contributes to multiple repos, their actor ID is the same — enabling future org-level aggregation (D2 follow-up). No registry dependency.
**Trade-offs:** If a GitHub user renames their account, the old and new logins produce different actor IDs. Edge case — GitHub login renames are rare, and the old profile would simply age out via cache expiry.
**Sources:** ContributorAttestationPolicy.java:86-92 (resolveSubjectId), R1-09 review finding
**Exploration:** quick
**Status:** captured (R1-09: new decision for previously implicit choice)

## D11: GitHub API authentication — config-driven, documented

**Choice:** GitHub API authentication uses the existing `@RegisterRestClient(configKey = "github-api")` configuration. Authentication mechanism (PAT via header injection or GitHub App JWT) is a deployment configuration concern, not a code design decision. The spec will document the rate-limit implications (60 req/hr unauthenticated vs 5000 req/hr authenticated) and require authenticated access for production deployment.
**Rationale:** The REST client infrastructure already exists. Authentication is configuration, not architecture.
**Trade-offs:** None — this is documenting an operational requirement.
**Sources:** GitHubPullRequestApi.java:16 (configKey), R1-10 review finding
**Exploration:** quick
**Status:** captured (R1-10: documenting previously implicit operational requirement)
