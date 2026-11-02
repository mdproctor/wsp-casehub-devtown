# Decisions — Per-Domain Contributor Trust Splitting (#178)

## D1: Domain source for contributor trust

**Choice:** Derive domains from the ReviewDomain capabilities the PR review case actually bound (security-review, architecture-review, etc.)
**Alternatives:**
- File-path heuristics — simpler but less semantic; misses cross-cutting changes
- Explicit contributor declaration — adds friction and can be gamed
- Code content analysis — accurate but adds complexity and latency
**Rationale:** The review case already classifies PR content by binding review capabilities. Reusing that signal avoids duplication and stays semantically accurate.
**Trade-offs:** Domains are limited to what the review case recognizes. PRs that skip review (e.g. auto-merged) won't produce domain-scoped attestations.
**Sources:** ReviewDomain.java (domain/src/main/java), DevtownCapabilityRegistry.java, routing.md (platform docs)
**Exploration:** quick
**Status:** captured

## D2: Multi-domain attestation strategy

**Choice:** One attestation per domain — a MERGED PR that triggered security-review and architecture-review produces separate attestations for each domain.
**Alternatives:**
- Single attestation for primary domain — simpler but loses signal from secondary domains
- Single attestation with weighted split — complex and mixes concerns
**Rationale:** Independent accumulation per domain preserves the Bayesian Beta model's per-capability semantics. Each domain builds its own observation count.
**Trade-offs:** More attestation entries in the ledger per PR. Contributors who touch many domains accumulate faster than single-domain specialists.
**Sources:** ContributorAttestationPolicy.java, TrustScoreRoutingPublisher (ledger), routing.md Layer 1
**Exploration:** quick
**Status:** captured

## D3: Coexistence with aggregate capability

**Choice:** Keep the aggregate `pr-contribution` alongside per-domain capabilities. Bootstrap phase uses the aggregate; per-domain scoring kicks in as observations accumulate.
**Alternatives:**
- Replace entirely — all scoring per-domain from day one; risk of thin data in rarely-triggered domains
**Rationale:** The aggregate provides a reliable fallback during the bootstrap phase where per-domain observations are sparse. Consumers can use whichever granularity has sufficient data.
**Trade-offs:** Two levels of scoring to maintain. Consumers must decide which level to query.
**Sources:** BootstrapScoreComputer.java, ContributorTrustCapability.java, issue #178 body ("Bayesian Beta needs sufficient observations per capability")
**Exploration:** quick
**Status:** captured

## D4: Data flow — domain info into attestation

**Choice:** Enrich PrLifecycleEvent with a `Set<String> reviewDomains` field. The caller (PR review case completion) already knows which review bindings fired.
**Alternatives:**
- Query the case at attestation time — adds engine runtime dependency at attestation time
- Separate event per domain — cleaner separation but more events and event types
**Rationale:** Clean data flow: the event carries all the information the attestation policy needs. No new dependencies.
**Trade-offs:** Changes to PrLifecycleEvent are a sealed interface change — all subtypes must be updated.
**Depends on:** D1 (domain source)
**Sources:** PrLifecycleEvent.java (review module)
**Exploration:** quick
**Status:** captured

## D5: Capability name modeling

**Choice:** Compound tag convention — `pr-contribution:<review-domain>` strings. ContributorTrustCapability gets a `forDomain(String)` factory method deriving from ReviewDomain constants.
**Alternatives:**
- Separate flat constants (PR_DOCS, PR_SECURITY) — discoverable but duplicates ReviewDomain's list
- Evidence metadata — domain in metadata, not capability key; doesn't support per-domain scoring
**Rationale:** The ledger keys scores by `capabilityName` — compound strings are just strings, no foundation changes. The convention is self-documenting and queryable via prefix match. The factory method avoids duplicating ReviewDomain's constant list.
**Trade-offs:** Convention-based — no compile-time enforcement that the domain part is valid. Mitigated by the factory method validating against ReviewDomain.REVIEW_CAPABILITIES.
**Depends on:** D1 (domain source), D3 (coexistence)
**Sources:** ContributorTrustCapability.java, ReviewDomain.java, DevtownCapabilityRegistry.java
**Exploration:** quick
**Status:** captured
