# Per-Domain Contributor Trust Splitting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #178 — Per-domain contributor trust splitting
**Issue group:** #178

**Goal:** Split the single `pr-contribution` capability into per-domain capabilities (`pr-contribution:security-review`, etc.) so contributor trust scores reflect domain-specific quality.

**Architecture:** Compound capability tags (`pr-contribution:<domain>`) piggyback on the existing ledger's per-capability scoring. `PrLifecycleEvent` is enriched with `reviewDomains` so the attestation policy can produce one attestation per activated domain plus the aggregate. A `ContributorTrustResolver` helper implements domain-vs-aggregate fallback based on observation count thresholds.

**Tech Stack:** Java 21, Quarkus 3.39.3, JUnit 5, AssertJ, Mockito

## Global Constraints

- No foundation (ledger, engine) changes — compound capability strings are just strings
- `code-analysis` excluded from domain splitting — fires for every PR, no discriminative signal
- Use `ReviewDomain.FINDINGS_CAPABILITIES` as the canonical domain set
- All new code in devtown modules only: `domain`, `review`, `app`

---

## Batch 1: Domain Model Foundation

### Task 1: ContributorTrustCapability — forDomain and isDomainScoped

**Files:**
- Modify: `domain/src/main/java/io/casehub/devtown/domain/ContributorTrustCapability.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/ContributorTrustCapabilityTest.java`

**Interfaces:**
- Consumes: `ReviewDomain.FINDINGS_CAPABILITIES` (existing `Set<String>`)
- Produces: `ContributorTrustCapability.forDomain(String) → String`, `ContributorTrustCapability.isDomainScoped(String) → boolean`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.domain;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ContributorTrustCapabilityTest {

    @ParameterizedTest
    @ValueSource(strings = {
        "security-review", "architecture-review", "style-review",
        "test-coverage", "performance-analysis"
    })
    void forDomain_producesCompoundTag(String domain) {
        assertThat(ContributorTrustCapability.forDomain(domain))
            .isEqualTo("pr-contribution:" + domain);
    }

    @Test
    void forDomain_rejectsCodeAnalysis() {
        assertThatThrownBy(() -> ContributorTrustCapability.forDomain("code-analysis"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void forDomain_rejectsUnknownDomain() {
        assertThatThrownBy(() -> ContributorTrustCapability.forDomain("unknown"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void forDomain_rejectsNull() {
        assertThatThrownBy(() -> ContributorTrustCapability.forDomain(null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void isDomainScoped_trueForCompoundTag() {
        assertThat(ContributorTrustCapability.isDomainScoped("pr-contribution:security-review"))
            .isTrue();
    }

    @Test
    void isDomainScoped_falseForAggregate() {
        assertThat(ContributorTrustCapability.isDomainScoped("pr-contribution"))
            .isFalse();
    }

    @Test
    void isDomainScoped_falseForUnrelatedCapability() {
        assertThat(ContributorTrustCapability.isDomainScoped("security-review"))
            .isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ContributorTrustCapabilityTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `forDomain` method does not exist

- [ ] **Step 3: Write minimal implementation**

Use `ide_insert_member` to add to `ContributorTrustCapability`:

```java
public static final String DOMAIN_SEPARATOR = ":";

public static String forDomain(String reviewDomain) {
    java.util.Objects.requireNonNull(reviewDomain, "reviewDomain must not be null");
    if (!ReviewDomain.FINDINGS_CAPABILITIES.contains(reviewDomain)) {
        throw new IllegalArgumentException("Unknown review domain: " + reviewDomain);
    }
    return PR_CONTRIBUTION + DOMAIN_SEPARATOR + reviewDomain;
}

public static boolean isDomainScoped(String capabilityTag) {
    return capabilityTag != null && capabilityTag.startsWith(PR_CONTRIBUTION + DOMAIN_SEPARATOR);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=ContributorTrustCapabilityTest`
Expected: PASS — all 8 tests green

- [ ] **Step 5: Commit**

```bash
git add domain/src/main/java/io/casehub/devtown/domain/ContributorTrustCapability.java
git add domain/src/test/java/io/casehub/devtown/domain/ContributorTrustCapabilityTest.java
git commit -m "feat(#178): add ContributorTrustCapability.forDomain and isDomainScoped

Compound capability tags (pr-contribution:<domain>) for per-domain
contributor trust scoring. Validates against FINDINGS_CAPABILITIES.

Refs #178"
```

### Task 2: PrLifecycleEvent — add reviewDomains to all subtypes

**Files:**
- Modify: `review/src/main/java/io/casehub/devtown/review/PrLifecycleEvent.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/trust/ContributorAttestationPolicyTest.java` (update constructors)
- Modify: `app/src/test/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserverTest.java` (update constructors)
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java:209,212,241` (update constructors)

**Interfaces:**
- Consumes: nothing new
- Produces: `PrLifecycleEvent.reviewDomains() → Set<String>` on the sealed interface

- [ ] **Step 1: Add reviewDomains to PrLifecycleEvent sealed interface and all subtypes**

Use `ide_replace_member` on `PrLifecycleEvent`:

```java
public sealed interface PrLifecycleEvent {
    String repo();
    int prNumber();
    String contributorId();
    Set<String> reviewDomains();

    record Merged(String repo, int prNumber, String contributorId,
                  UUID caseId, int reviewRounds,
                  Set<String> reviewDomains) implements PrLifecycleEvent {}

    record Rejected(String repo, int prNumber, String contributorId,
                    UUID caseId, String senderLogin, long senderId,
                    String senderType, int reviewRounds,
                    Set<String> reviewDomains) implements PrLifecycleEvent {}

    record ChangesRequested(String repo, int prNumber, String contributorId,
                            UUID caseId,
                            Set<String> reviewDomains) implements PrLifecycleEvent {}

    record TriageRejected(String repo, int prNumber, String contributorId,
                          String reason) implements PrLifecycleEvent {
        @Override public Set<String> reviewDomains() { return Set.of(); }
    }
}
```

Note: `TriageRejected` does not carry `reviewDomains` as a record field — it overrides the interface method to return `Set.of()`.

- [ ] **Step 2: Fix compilation errors in production code**

Use `ide_diagnostics` to find all compilation errors. Update `PrReviewCaseService` constructor calls:

At line 209 (`closePr` — Merged):
```java
mergedEvent.fire(new PrLifecycleEvent.Merged(
        close.repo(), close.prNumber(), contributorId, caseId, reviewRounds,
        Set.of()));
```

At line 212 (`closePr` — Rejected):
```java
rejectedEvent.fire(new PrLifecycleEvent.Rejected(
        close.repo(), close.prNumber(), contributorId, caseId,
        close.senderLogin(), close.senderNumericId(), close.senderType(), reviewRounds,
        Set.of()));
```

At line 241 (`signalReviewSubmitted` — ChangesRequested):
```java
changesRequestedEvent.fire(new PrLifecycleEvent.ChangesRequested(
        review.repo(), review.prNumber(), contributorId, caseId,
        Set.of()));
```

Pass `Set.of()` for now — Task 4 in Batch 2 will populate with actual domains.

- [ ] **Step 3: Fix compilation errors in test code**

Update all `PrLifecycleEvent` constructor calls in test files to include `Set.of()` as the new `reviewDomains` parameter. Use `ide_diagnostics` to find them all.

`ContributorAttestationPolicyTest.java` — add `Set.of()` to every event construction (lines 22, 36, 48, 59-60, 71, 82-83, 94).

`PrLifecycleAttestationObserverTest.java` — add `Set.of()` to every event construction (lines 50, 59, 68-69, 77-78, 87-88, 96-97, 105-106, 114, 122, 131).

- [ ] **Step 4: Run full test suite to verify compilation and existing tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain,review,app`
Expected: PASS — all existing tests green with the new field defaulted to `Set.of()`

- [ ] **Step 5: Commit**

```bash
git add review/src/main/java/io/casehub/devtown/review/PrLifecycleEvent.java
git add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java
git add app/src/test/java/io/casehub/devtown/app/trust/ContributorAttestationPolicyTest.java
git add app/src/test/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserverTest.java
git commit -m "feat(#178): enrich PrLifecycleEvent with reviewDomains

All subtypes gain Set<String> reviewDomains. TriageRejected overrides
to return Set.of(). Event producers pass Set.of() for now — actual
domain resolution in a subsequent commit.

Refs #178"
```

## Batch 2: Attestation Production

### Task 3: ContributorAttestationPolicy — parameterize for per-domain attestations

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/trust/ContributorAttestationPolicyTest.java`

**Interfaces:**
- Consumes: `ContributorTrustCapability.forDomain(String)` (Task 1)
- Produces: `ContributorAttestationPolicy.evaluate(PrLifecycleEvent, String, Integer, Set<String>)` — new overload that returns aggregate + per-domain attestations

- [ ] **Step 1: Write the failing test — per-domain attestation production**

Add to `ContributorAttestationPolicyTest`:

```java
@Test
void merged_withTwoDomains_producesThreeAttestations() {
    var domains = Set.of(ReviewDomain.SECURITY_REVIEW, ReviewDomain.ARCHITECTURE_REVIEW);
    var event = new PrLifecycleEvent.Merged("repo/name", 42, "github-id:123",
            UUID.randomUUID(), 0, domains);
    List<AttestationIntent> intents = policy.evaluate(event, "MERGED", 0, domains);

    assertThat(intents).hasSize(3);
    assertThat(intents.stream().map(AttestationIntent::capabilityTag))
        .containsExactlyInAnyOrder(
            ContributorTrustCapability.PR_CONTRIBUTION,
            ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW),
            ContributorTrustCapability.forDomain(ReviewDomain.ARCHITECTURE_REVIEW));
}

@Test
void merged_emptyDomains_producesAggregateOnly() {
    var event = new PrLifecycleEvent.Merged("repo/name", 42, "github-id:123",
            UUID.randomUUID(), 0, Set.of());
    List<AttestationIntent> intents = policy.evaluate(event, "MERGED", 0, Set.of());

    assertThat(intents).hasSize(1);
    assertThat(intents.get(0).capabilityTag())
        .isEqualTo(ContributorTrustCapability.PR_CONTRIBUTION);
}

@Test
void perDomain_entryIds_differFromAggregate() {
    var domains = Set.of(ReviewDomain.SECURITY_REVIEW);
    var event = new PrLifecycleEvent.Merged("repo/name", 42, "github-id:123",
            UUID.randomUUID(), 0, domains);
    List<AttestationIntent> intents = policy.evaluate(event, "MERGED", 0, domains);

    UUID aggregateId = intents.stream()
        .filter(i -> i.capabilityTag().equals(ContributorTrustCapability.PR_CONTRIBUTION))
        .findFirst().orElseThrow().entryId();
    UUID domainId = intents.stream()
        .filter(i -> i.capabilityTag().contains("security"))
        .findFirst().orElseThrow().entryId();
    assertThat(aggregateId).isNotEqualTo(domainId);
}

@Test
void perDomain_sameVerdictAndDimensionsAsAggregate() {
    var domains = Set.of(ReviewDomain.SECURITY_REVIEW);
    var event = new PrLifecycleEvent.Merged("repo/name", 42, "github-id:123",
            UUID.randomUUID(), 0, domains);
    List<AttestationIntent> intents = policy.evaluate(event, "MERGED", 0, domains);

    AttestationIntent aggregate = intents.stream()
        .filter(i -> i.capabilityTag().equals(ContributorTrustCapability.PR_CONTRIBUTION))
        .findFirst().orElseThrow();
    AttestationIntent domain = intents.stream()
        .filter(i -> i.capabilityTag().contains("security"))
        .findFirst().orElseThrow();

    assertThat(domain.verdict()).isEqualTo(aggregate.verdict());
    assertThat(domain.confidence()).isEqualTo(aggregate.confidence());
    assertThat(domain.dimensions()).isEqualTo(aggregate.dimensions());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContributorAttestationPolicyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — no `evaluate` overload with 4 parameters

- [ ] **Step 3: Implement per-domain attestation logic**

Modify `ContributorAttestationPolicy`:

1. Change `intent()` to accept `String capabilityTag` parameter instead of hardcoding `ContributorTrustCapability.PR_CONTRIBUTION`
2. Add static `deterministicDomainEntryId(String repo, int prNumber, String outcome, String domain)` method
3. Add new `evaluate(PrLifecycleEvent, String, Integer, Set<String>)` overload that:
   - Calls the existing switch to produce the aggregate attestation
   - For each domain in the set, produces a domain-scoped attestation with the same verdict/confidence/dimensions but tagged with `forDomain(domain)` and a domain-specific entry ID
4. Update the existing 3-parameter `evaluate` to delegate to the new overload with `Set.of()`

```java
public List<AttestationIntent> evaluate(PrLifecycleEvent event,
                                         String outcome,
                                         Integer reviewRounds,
                                         Set<String> reviewDomains) {
    List<AttestationIntent> result = new ArrayList<>();
    UUID subjectId = resolveSubjectId(event);

    // Aggregate attestation (same as before)
    UUID aggregateEntryId = deterministicEntryId(event.repo(), event.prNumber(), outcome);
    addOutcomeIntents(result, aggregateEntryId, subjectId,
            ContributorTrustCapability.PR_CONTRIBUTION,
            outcome, event, reviewRounds);

    // Per-domain attestations
    for (String domain : reviewDomains) {
        UUID domainEntryId = deterministicDomainEntryId(
                event.repo(), event.prNumber(), outcome, domain);
        addOutcomeIntents(result, domainEntryId, subjectId,
                ContributorTrustCapability.forDomain(domain),
                outcome, event, reviewRounds);
    }
    return result;
}

public List<AttestationIntent> evaluate(PrLifecycleEvent event,
                                         String outcome,
                                         Integer reviewRounds) {
    return evaluate(event, outcome, reviewRounds, Set.of());
}

public static UUID deterministicDomainEntryId(String repo, int prNumber,
                                               String outcome, String domain) {
    String key = CONTRIBUTOR_OUTCOME_NS + "|" + repo + ":" + prNumber
            + ":" + outcome + ":" + domain;
    return UUID.nameUUIDFromBytes(key.getBytes(StandardCharsets.UTF_8));
}
```

Extract the switch body into a private `addOutcomeIntents(List, UUID, UUID, String, String, PrLifecycleEvent, Integer)` method that handles each outcome and adds to the list. The `intent()` private method takes `capabilityTag` as a parameter.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContributorAttestationPolicyTest`
Expected: PASS — all existing + new tests green

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java
git add app/src/test/java/io/casehub/devtown/app/trust/ContributorAttestationPolicyTest.java
git commit -m "feat(#178): parameterize ContributorAttestationPolicy for per-domain attestations

evaluate() gains reviewDomains parameter. Produces aggregate +
one attestation per activated domain. Domain entry IDs are deterministic
and distinct from the aggregate.

Refs #178"
```

### Task 4: PrReviewCaseService — resolve activated domains and pass to events

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`
- Modify: existing test for `PrReviewCaseService` (if exists, else create integration test)

**Interfaces:**
- Consumes: `PlanItemStore.findByCaseId(UUID, String)` → `List<PlanItemRecord>` (engine SPI, already on classpath), `ReviewDomain.FINDINGS_CAPABILITIES`
- Produces: `PrLifecycleEvent` instances with populated `reviewDomains`

- [ ] **Step 1: Write the failing test — domain resolution helper**

Create or extend test for the domain resolution:

```java
@Test
void resolveActivatedDomains_filtersToFindingsCapabilities() {
    // Given plan items with various capability names
    // When resolved
    // Then only FINDINGS_CAPABILITIES are returned
}
```

- [ ] **Step 2: Implement domain resolution in PrReviewCaseService**

Add a private method to `PrReviewCaseService`:

```java
private Set<String> resolveActivatedDomains(UUID caseId) {
    return planItemStore.findByCaseId(caseId, tenancyId())
        .stream()
        .map(PlanItemRecord::capabilityName)
        .filter(java.util.Objects::nonNull)
        .filter(ReviewDomain.FINDINGS_CAPABILITIES::contains)
        .collect(java.util.stream.Collectors.toUnmodifiableSet());
}
```

Inject `PlanItemStore` into `PrReviewCaseService` (it is already available on the classpath via the engine module).

- [ ] **Step 3: Update event construction to pass resolved domains**

In `closePr()`:
```java
Set<String> domains = resolveActivatedDomains(caseId);
if (close.merged()) {
    mergedEvent.fire(new PrLifecycleEvent.Merged(
            close.repo(), close.prNumber(), contributorId, caseId, reviewRounds, domains));
} else {
    rejectedEvent.fire(new PrLifecycleEvent.Rejected(
            close.repo(), close.prNumber(), contributorId, caseId,
            close.senderLogin(), close.senderNumericId(), close.senderType(), reviewRounds, domains));
}
```

In `signalReviewSubmitted()`:
```java
Set<String> domains = resolveActivatedDomains(caseId);
changesRequestedEvent.fire(new PrLifecycleEvent.ChangesRequested(
        review.repo(), review.prNumber(), contributorId, caseId, domains));
```

- [ ] **Step 4: Run tests to verify compilation and existing tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java
git commit -m "feat(#178): resolve activated review domains in PrReviewCaseService

PrLifecycleEvents now carry the set of FINDINGS_CAPABILITIES bindings
that fired on the case. Resolved via PlanItemStore.findByCaseId().

Refs #178"
```

### Task 5: PrLifecycleAttestationObserver — pass reviewDomains to policy

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserver.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserverTest.java`

**Interfaces:**
- Consumes: `ContributorAttestationPolicy.evaluate(event, outcome, reviewRounds, reviewDomains)` (Task 3)
- Produces: Per-domain attestations written to ledger via writer

- [ ] **Step 1: Write failing tests for per-domain attestation flow**

Add to `PrLifecycleAttestationObserverTest`:

```java
@Test
void merged_withDomains_passesDomainsToPolicy() {
    var domains = Set.of(ReviewDomain.SECURITY_REVIEW, ReviewDomain.ARCHITECTURE_REVIEW);
    var event = new PrLifecycleEvent.Merged("repo/name", 42, "github-id:123",
            UUID.randomUUID(), 0, domains);

    observer.onMerged(event);

    verify(writer).writeOutcome(eq(event), eq("MERGED"), eq(0), argThat(intents ->
        intents.size() == 3 &&
        intents.stream().map(AttestationIntent::capabilityTag)
            .collect(java.util.stream.Collectors.toSet())
            .containsAll(Set.of(
                ContributorTrustCapability.PR_CONTRIBUTION,
                ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW),
                ContributorTrustCapability.forDomain(ReviewDomain.ARCHITECTURE_REVIEW)))));
}

@Test
void triageRejected_emptyDomains_producesAggregateOnly() {
    var event = new PrLifecycleEvent.TriageRejected("repo/name", 42, "github-id:123", "spam");
    observer.onTriageRejected(event);

    verify(writer).writeOutcome(eq(event), eq("TRIAGE_REJECTED"), any(), argThat(intents ->
        intents.size() == 1 &&
        intents.get(0).capabilityTag().equals(ContributorTrustCapability.PR_CONTRIBUTION)));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PrLifecycleAttestationObserverTest`
Expected: FAIL — observer still calls 3-parameter evaluate (no domains passed)

- [ ] **Step 3: Update observer to pass reviewDomains**

Change each observer method to call the 4-parameter `evaluate` overload:

```java
void onMerged(@Observes @Priority(100) PrLifecycleEvent.Merged event) {
    String outcome = event.reviewRounds() > 0 ? "MERGED_WITH_REWORK" : "MERGED";
    List<AttestationIntent> intents = policy.evaluate(
            event, outcome, event.reviewRounds(), event.reviewDomains());
    writer.writeOutcome(event, outcome, event.reviewRounds(), intents);
}
```

Apply the same pattern to `onRejected`, `onChangesRequested`, `onTriageRejected`, and `handleBotOrAuthorClosure`. For each: replace `policy.evaluate(event, outcome, rounds)` with `policy.evaluate(event, outcome, rounds, event.reviewDomains())`.

- [ ] **Step 4: Run tests to verify all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=PrLifecycleAttestationObserverTest`
Expected: PASS — all tests green including new per-domain tests

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserver.java
git add app/src/test/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserverTest.java
git commit -m "feat(#178): wire per-domain attestations through observer

PrLifecycleAttestationObserver passes event.reviewDomains() to
the 4-parameter evaluate overload. Multi-domain PRs produce
aggregate + per-domain attestations.

Refs #178"
```

## Batch 3: Consumer Layer

### Task 6: DevtownCapabilityRegistry — add RoutingPolicy entries for compound capabilities

**Files:**
- Modify: `domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java`
- Modify: `domain/src/test/java/io/casehub/devtown/domain/DevtownCapabilityRegistryTest.java`

**Interfaces:**
- Consumes: `ContributorTrustCapability.forDomain(String)` (Task 1), `ReviewDomain.FINDINGS_CAPABILITIES`
- Produces: `CapabilityRegistry.policy(ContributorTrustCapability.forDomain(domain))` → `Optional<RoutingPolicy>` with minimum observation thresholds

- [ ] **Step 1: Write the failing test — compound capability policies**

Add to `DevtownCapabilityRegistryTest`:

```java
@Test
void securityReviewContributorDomain_minimumObservationsIs10() {
    RoutingPolicy policy = registry.policy(
        ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW)).orElseThrow();
    assertThat(policy.minimumObservations().getAsInt()).isEqualTo(10);
}

@Test
void architectureReviewContributorDomain_minimumObservationsIs8() {
    RoutingPolicy policy = registry.policy(
        ContributorTrustCapability.forDomain(ReviewDomain.ARCHITECTURE_REVIEW)).orElseThrow();
    assertThat(policy.minimumObservations().getAsInt()).isEqualTo(8);
}

@Test
void styleReviewContributorDomain_minimumObservationsIs5() {
    RoutingPolicy policy = registry.policy(
        ContributorTrustCapability.forDomain(ReviewDomain.STYLE_REVIEW)).orElseThrow();
    assertThat(policy.minimumObservations().getAsInt()).isEqualTo(5);
}

@Test
void contributorDomainPolicies_haveNoThreshold() {
    RoutingPolicy policy = registry.policy(
        ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW)).orElseThrow();
    assertThat(policy.threshold()).isEmpty();
}

@Test
void contributorDomainPolicies_haveNoFallback() {
    RoutingPolicy policy = registry.policy(
        ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW)).orElseThrow();
    assertThat(policy.fallbackType()).isEmpty();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=DevtownCapabilityRegistryTest`
Expected: FAIL — `policy()` returns empty for compound capabilities

- [ ] **Step 3: Add RoutingPolicy entries for compound capabilities**

Use `ide_replace_member` on `DevtownCapabilityRegistry.POLICIES` to add entries:

```java
private static final Map<String, RoutingPolicy> POLICIES;

static {
    Map<String, RoutingPolicy> m = new java.util.HashMap<>(Map.of(
        ReviewDomain.SECURITY_REVIEW, new RoutingPolicy(
            OptionalDouble.of(0.70), OptionalInt.of(10), OptionalDouble.of(0.05),
            Optional.of(HumanOversight.ROUTING_REVIEW),
            "Security mistakes reach production; 10 observations required for credible score"),
        ReviewDomain.ARCHITECTURE_REVIEW, new RoutingPolicy(
            OptionalDouble.of(0.65), OptionalInt.of(8), OptionalDouble.of(0.05),
            Optional.of(HumanOversight.ROUTING_REVIEW),
            "Design mistakes are expensive to reverse; 8 observations required"),
        ReviewDomain.STYLE_REVIEW, new RoutingPolicy(
            OptionalDouble.of(0.50), OptionalInt.of(5), OptionalDouble.empty(),
            Optional.empty(),
            "Baseline — any competent agent; 5 observations sufficient"),
        AgentQualification.MERGE_EXECUTOR, new RoutingPolicy(
            OptionalDouble.of(0.80), OptionalInt.of(15), OptionalDouble.of(0.05),
            Optional.of(HumanOversight.ROUTING_REVIEW),
            "Merge is irreversible; highest observation requirement")
    ));

    // Per-domain contributor trust thresholds (minimum observations only)
    Map<String, Integer> domainMinObs = Map.of(
        ReviewDomain.SECURITY_REVIEW, 10,
        ReviewDomain.ARCHITECTURE_REVIEW, 8,
        ReviewDomain.STYLE_REVIEW, 5,
        ReviewDomain.TEST_COVERAGE, 5,
        ReviewDomain.PERFORMANCE_ANALYSIS, 5
    );
    domainMinObs.forEach((domain, minObs) ->
        m.put(ContributorTrustCapability.forDomain(domain), new RoutingPolicy(
            OptionalDouble.empty(), OptionalInt.of(minObs), OptionalDouble.empty(),
            Optional.empty(),
            "Per-domain contributor trust — " + minObs + " observations before domain score preferred")));

    POLICIES = Map.copyOf(m);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=DevtownCapabilityRegistryTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java
git add domain/src/test/java/io/casehub/devtown/domain/DevtownCapabilityRegistryTest.java
git commit -m "feat(#178): add RoutingPolicy entries for per-domain contributor capabilities

Compound capabilities get minimum observation thresholds:
security-review=10, architecture-review=8, others=5.
No routing threshold or fallback — these are trust tags, not routing caps.

Refs #178"
```

### Task 7: ContributorTrustResolver — domain-vs-aggregate fallback helper

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/trust/ContributorTrustResolver.java`
- Create: `app/src/test/java/io/casehub/devtown/app/trust/ContributorTrustResolverTest.java`

**Interfaces:**
- Consumes: `CapabilityRegistry.policy(String)` (existing), `TrustGateService.decisionCount(String, String)` (existing), `TrustGateService.capabilityScore(String, String)` (existing via `TrustScoreSource`)
- Produces: `ContributorTrustResolver.resolveScore(String contributorId, String domain) → OptionalDouble`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.ContributorTrustCapability;
import io.casehub.devtown.domain.DevtownCapabilityRegistry;
import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.domain.spi.CapabilityRegistry;
import io.casehub.ledger.core.trust.TrustGateService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.OptionalDouble;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class ContributorTrustResolverTest {

    private TrustGateService trustGateService;
    private CapabilityRegistry capabilityRegistry;
    private ContributorTrustResolver resolver;

    @BeforeEach
    void setUp() {
        trustGateService = mock(TrustGateService.class);
        capabilityRegistry = new DevtownCapabilityRegistry();
        resolver = new ContributorTrustResolver(capabilityRegistry, trustGateService);
    }

    @Test
    void aboveThreshold_returnsDomainScore() {
        String domainCap = ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW);
        when(trustGateService.decisionCount("alice", domainCap)).thenReturn(12);
        when(trustGateService.capabilityScore("alice", domainCap))
            .thenReturn(OptionalDouble.of(0.85));

        OptionalDouble result = resolver.resolveScore("alice", ReviewDomain.SECURITY_REVIEW);
        assertThat(result).hasValue(0.85);
    }

    @Test
    void belowThreshold_fallsBackToAggregate() {
        String domainCap = ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW);
        when(trustGateService.decisionCount("alice", domainCap)).thenReturn(3);
        when(trustGateService.capabilityScore("alice", ContributorTrustCapability.PR_CONTRIBUTION))
            .thenReturn(OptionalDouble.of(0.70));

        OptionalDouble result = resolver.resolveScore("alice", ReviewDomain.SECURITY_REVIEW);
        assertThat(result).hasValue(0.70);
    }

    @Test
    void noScoreAtAll_returnsEmpty() {
        String domainCap = ContributorTrustCapability.forDomain(ReviewDomain.SECURITY_REVIEW);
        when(trustGateService.decisionCount("alice", domainCap)).thenReturn(0);
        when(trustGateService.capabilityScore("alice", ContributorTrustCapability.PR_CONTRIBUTION))
            .thenReturn(OptionalDouble.empty());

        OptionalDouble result = resolver.resolveScore("alice", ReviewDomain.SECURITY_REVIEW);
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContributorTrustResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement ContributorTrustResolver**

Create `app/src/main/java/io/casehub/devtown/app/trust/ContributorTrustResolver.java`:

```java
package io.casehub.devtown.app.trust;

import io.casehub.devtown.domain.ContributorTrustCapability;
import io.casehub.devtown.domain.RoutingPolicy;
import io.casehub.devtown.domain.spi.CapabilityRegistry;
import io.casehub.ledger.core.trust.TrustGateService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Optional;
import java.util.OptionalDouble;

@ApplicationScoped
public class ContributorTrustResolver {

    private final CapabilityRegistry capabilityRegistry;
    private final TrustGateService trustGateService;

    @Inject
    public ContributorTrustResolver(CapabilityRegistry capabilityRegistry,
                                     TrustGateService trustGateService) {
        this.capabilityRegistry = capabilityRegistry;
        this.trustGateService = trustGateService;
    }

    public OptionalDouble resolveScore(String contributorId, String domain) {
        String domainCap = ContributorTrustCapability.forDomain(domain);
        Optional<RoutingPolicy> policy = capabilityRegistry.policy(domainCap);

        if (policy.isPresent() && policy.get().minimumObservations().isPresent()) {
            int observations = trustGateService.decisionCount(contributorId, domainCap);
            if (observations >= policy.get().minimumObservations().getAsInt()) {
                OptionalDouble domainScore = trustGateService.capabilityScore(contributorId, domainCap);
                if (domainScore.isPresent()) {
                    return domainScore;
                }
            }
        }
        return trustGateService.capabilityScore(
                contributorId, ContributorTrustCapability.PR_CONTRIBUTION);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=ContributorTrustResolverTest`
Expected: PASS — all 3 tests green

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/trust/ContributorTrustResolver.java
git add app/src/test/java/io/casehub/devtown/app/trust/ContributorTrustResolverTest.java
git commit -m "feat(#178): add ContributorTrustResolver for domain-vs-aggregate fallback

Resolves contributor trust score by domain: uses per-domain score
when observation count meets threshold, falls back to aggregate
pr-contribution otherwise.

Refs #178"
```

### Task 8: GovernanceQueryService — expose per-domain scores

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/governance/GovernanceQueryServiceTest.java` (if exists)

**Interfaces:**
- Consumes: `ContributorTrustCapability.isDomainScoped(String)` (Task 1), `TrustGateService.allCapabilityScores()` (existing)
- Produces: Per-domain score breakdown in `ContributorFleetEntry` and `ContributorDetail` responses

- [ ] **Step 1: Read GovernanceQueryService contributor methods to understand current structure**

Use `ide_file_structure` on `GovernanceQueryService.java` to find `contributorFleet()` and `contributorDetail()` methods. Read the relevant sections.

- [ ] **Step 2: Write failing test for per-domain score in contributor detail**

The exact test depends on what `GovernanceQueryServiceTest` already contains. Add a test that verifies domain scores are included when querying contributor detail. Test that `ContributorTrustCapability.isDomainScoped()` is used to partition capability scores into domain vs non-domain.

- [ ] **Step 3: Extend GovernanceQueryService.contributorFleet()**

Add a `Map<String, Double> domainScores` field to the response. Filter `allCapabilityScores()` results using `ContributorTrustCapability.isDomainScoped()` to separate domain scores from other capability scores.

- [ ] **Step 4: Extend GovernanceQueryService.contributorDetail()**

Include per-domain observation counts alongside scores. Use `trustGateService.decisionCount()` for each domain-scoped capability.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java
git commit -m "feat(#178): expose per-domain contributor trust scores in governance API

contributorFleet() and contributorDetail() include domain-scoped
scores and observation counts. Uses isDomainScoped() to partition
capability scores.

Closes #178"
```

## References

- `specs/issue-178-per-domain-contributor-trust/2026-11-02-per-domain-contributor-trust-design.md` — design spec
- `specs/issue-178-per-domain-contributor-trust/decisions.md` — D1-D6 design decisions
- `domain/src/main/java/io/casehub/devtown/domain/ContributorTrustCapability.java:4` — current single capability
- `domain/src/main/java/io/casehub/devtown/domain/ReviewDomain.java:14-21` — REVIEW_CAPABILITIES and FINDINGS_CAPABILITIES
- `domain/src/main/java/io/casehub/devtown/domain/DevtownCapabilityRegistry.java:11-71` — capability + policy registry
- `review/src/main/java/io/casehub/devtown/review/PrLifecycleEvent.java:5-22` — sealed event interface
- `app/src/main/java/io/casehub/devtown/app/trust/ContributorAttestationPolicy.java:19-102` — attestation production
- `app/src/main/java/io/casehub/devtown/app/trust/PrLifecycleAttestationObserver.java:14-57` — CDI event observer
- `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java:200-219` — event construction site
- `engine/common-core/src/main/java/io/casehub/engine/common/spi/PlanItemStore.java:36` — findByCaseId for domain resolution
- `engine/common-core/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java:25,44` — bindingName and capabilityName fields
- GitHub #178 — Per-domain contributor trust splitting
