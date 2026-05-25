# Layer 2: SLA-Bounded Human Review Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire a `SlaBreachPolicy` into the devtown PR review flow so every human-approval WorkItem has a 24-hour deadline, escalates to `pr-leads` on first breach, and signals the case context with `{status: "sla-breach"}` on second breach.

**Architecture:** `DefaultSlaBreachPolicy` (pure Java, `domain` module) implements the stateless two-tier escalation pattern — detecting tier by inspecting `candidateGroups`. `SlaBreachPolicyBean` (`app`) displaces `NoOpSlaBreachPolicy` via CDI. `SlaBreachHandler` (`app`) observes `SlaBreachEvent` (synchronous CDI, fired by `ExpiryLifecycleService`) and signals the case on `Fail`. The pr-review YAML adds `candidateGroups` and `expiresIn` to the `human-approval` binding.

**Tech Stack:** Java 21, Quarkus 3.32.2, `casehub-work-api` (SlaBreachPolicy/BreachDecision SPI), `casehub-platform-api` (Preferences/PreferenceKey), CDI Events, JUnit 5, AssertJ, Awaitility

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Modify | `domain/pom.xml` | Add `casehub-work-api` compile dependency |
| Create | `domain/src/main/java/io/casehub/devtown/domain/sla/StringPreference.java` | Record implementing `SingleValuePreference` for String values |
| Create | `domain/src/main/java/io/casehub/devtown/domain/sla/IntPreference.java` | Record implementing `SingleValuePreference` for int values |
| Create | `domain/src/main/java/io/casehub/devtown/domain/sla/SlaPreferenceKeys.java` | Typed `PreferenceKey` constants for SLA configuration |
| Create | `domain/src/main/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicy.java` | Pure Java `SlaBreachPolicy` — stateless two-tier escalation |
| Create | `domain/src/test/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicyTest.java` | Unit tests — plain JUnit, MapPreferences |
| Create | `app/src/main/java/io/casehub/devtown/app/SlaBreachPolicyBean.java` | `@ApplicationScoped` CDI bean displacing `NoOpSlaBreachPolicy` |
| Create | `app/src/main/java/io/casehub/devtown/app/SlaBreachHandler.java` | `@Observes SlaBreachEvent` — signals case on Fail |
| Modify | `review/src/main/resources/devtown/pr-review.yaml` | Add `candidateGroups` and `expiresIn` to `human-approval` binding |
| Create | `app/src/test/java/io/casehub/devtown/app/SlaBreachLifecycleTest.java` | `@QuarkusTest` two-tier breach lifecycle |

---

## Task 1: Domain dependencies and preference value types

**Files:**
- Modify: `domain/pom.xml`
- Create: `domain/src/main/java/io/casehub/devtown/domain/sla/StringPreference.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/sla/IntPreference.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/sla/SlaPreferenceKeys.java`

These are pure data types — no tests needed; covered by Task 2's policy tests.

- [ ] **Step 1: Add `casehub-work-api` to `domain/pom.xml`**

Open `domain/pom.xml`. Add inside `<dependencies>`:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-work-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
```

The `casehub-work-api` version is managed by the parent BOM. `casehub-platform-api` is already in the BOM. Both are pure Java — no Quarkus transitive deps.

- [ ] **Step 2: Create `StringPreference`**

```java
// domain/src/main/java/io/casehub/devtown/domain/sla/StringPreference.java
package io.casehub.devtown.domain.sla;

import io.casehub.platform.api.preferences.SingleValuePreference;
import java.util.Objects;

public record StringPreference(String value) implements SingleValuePreference {
    public StringPreference {
        Objects.requireNonNull(value, "value must not be null");
    }
    public static StringPreference of(String value) { return new StringPreference(value); }
    public static StringPreference parse(String raw) { return new StringPreference(raw); }
}
```

- [ ] **Step 3: Create `IntPreference`**

```java
// domain/src/main/java/io/casehub/devtown/domain/sla/IntPreference.java
package io.casehub.devtown.domain.sla;

import io.casehub.platform.api.preferences.SingleValuePreference;

public record IntPreference(int value) implements SingleValuePreference {
    public static IntPreference of(int value) { return new IntPreference(value); }
    public static IntPreference parse(String raw) { return new IntPreference(Integer.parseInt(raw)); }
}
```

- [ ] **Step 4: Create `SlaPreferenceKeys`**

```java
// domain/src/main/java/io/casehub/devtown/domain/sla/SlaPreferenceKeys.java
package io.casehub.devtown.domain.sla;

import io.casehub.platform.api.preferences.PreferenceKey;

public final class SlaPreferenceKeys {

    public static final PreferenceKey<IntPreference> ESCALATION_HOURS =
        new PreferenceKey<>("devtown.sla", "escalation-hours",
            IntPreference.of(8), IntPreference::parse);

    public static final PreferenceKey<StringPreference> ESCALATION_GROUP =
        new PreferenceKey<>("devtown.sla", "escalation-group",
            StringPreference.of("pr-leads"), StringPreference::parse);

    public static final PreferenceKey<StringPreference> BREACH_TERMINAL_REASON =
        new PreferenceKey<>("devtown.sla", "breach-terminal-reason",
            StringPreference.of("sla-breach"), StringPreference::parse);

    public static final PreferenceKey<IntPreference> COMPLETION_HOURS =
        new PreferenceKey<>("devtown.sla", "completion-hours",
            IntPreference.of(24), IntPreference::parse);

    public static final PreferenceKey<StringPreference> CANDIDATE_GROUP =
        new PreferenceKey<>("devtown.sla", "candidate-group",
            StringPreference.of("pr-reviewers"), StringPreference::parse);

    private SlaPreferenceKeys() {}
}
```

- [ ] **Step 5: Verify domain compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl domain --batch-mode -q
```

Expected: `BUILD SUCCESS` with no errors.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/pom.xml \
  domain/src/main/java/io/casehub/devtown/domain/sla/
git -C /Users/mdproctor/claude/casehub/devtown commit -m \
  "feat(domain): add SLA preference types and keys — Refs #41"
```

---

## Task 2: DefaultSlaBreachPolicy — TDD

**Files:**
- Create: `domain/src/test/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicyTest.java`
- Create: `domain/src/main/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicy.java`

- [ ] **Step 1: Write the failing tests**

```java
// domain/src/test/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicyTest.java
package io.casehub.devtown.domain.sla;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.platform.api.path.Path;
import io.casehub.platform.api.preferences.MapPreferences;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.BreachType;
import io.casehub.work.api.BreachedTask;
import io.casehub.work.api.SlaBreachContext;
import java.time.Duration;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class DefaultSlaBreachPolicyTest {

    private final DefaultSlaBreachPolicy policy = new DefaultSlaBreachPolicy();

    // ── First breach (candidateGroups = pr-reviewers) → EscalateTo pr-leads ──

    @Test
    void firstBreach_escalatesToDefaultEscalationGroup() {
        var ctx = ctx(Set.of("pr-reviewers"), defaultPrefs());
        var decision = policy.onBreach(ctx);
        assertThat(decision).isInstanceOf(BreachDecision.EscalateTo.class);
        var escalate = (BreachDecision.EscalateTo) decision;
        assertThat(escalate.groups()).containsExactly("pr-leads");
    }

    @Test
    void firstBreach_deadlineIsEscalationHours() {
        var ctx = ctx(Set.of("pr-reviewers"), defaultPrefs());
        var decision = (BreachDecision.EscalateTo) policy.onBreach(ctx);
        assertThat(decision.deadline()).isEqualTo(Duration.ofHours(8));
    }

    @Test
    void firstBreach_claimExpiredAlsoEscalates() {
        var ctx = ctxType(BreachType.CLAIM_EXPIRED, Set.of("pr-reviewers"), defaultPrefs());
        assertThat(policy.onBreach(ctx)).isInstanceOf(BreachDecision.EscalateTo.class);
    }

    // ── Second breach (candidateGroups = pr-leads) → Fail ─────────────────────

    @Test
    void secondBreach_failsWithTerminalReason() {
        var ctx = ctx(Set.of("pr-leads"), defaultPrefs());
        var decision = policy.onBreach(ctx);
        assertThat(decision).isInstanceOf(BreachDecision.Fail.class);
        assertThat(((BreachDecision.Fail) decision).reason()).isEqualTo("sla-breach");
    }

    @Test
    void secondBreach_withCustomTerminalReason() {
        var prefs = new MapPreferences(
            Map.of("devtown.sla.breach-terminal-reason", "custom-breach-reason"));
        var ctx = ctx(Set.of("pr-leads"), prefs);
        var decision = (BreachDecision.Fail) policy.onBreach(ctx);
        assertThat(decision.reason()).isEqualTo("custom-breach-reason");
    }

    // ── Custom preferences ────────────────────────────────────────────────────

    @Test
    void customEscalationGroup_usedForBothTierDetectionAndEscalation() {
        var prefs = new MapPreferences(Map.of("devtown.sla.escalation-group", "senior-leads"));
        var ctx1 = ctx(Set.of("pr-reviewers"), prefs);
        assertThat(policy.onBreach(ctx1)).isInstanceOf(BreachDecision.EscalateTo.class);
        assertThat(((BreachDecision.EscalateTo) policy.onBreach(ctx1)).groups())
            .containsExactly("senior-leads");

        var ctx2 = ctx(Set.of("senior-leads"), prefs);
        assertThat(policy.onBreach(ctx2)).isInstanceOf(BreachDecision.Fail.class);
    }

    @Test
    void customEscalationHours_appliedToDeadline() {
        var prefs = new MapPreferences(Map.of("devtown.sla.escalation-hours", "4"));
        var ctx = ctx(Set.of("pr-reviewers"), prefs);
        var decision = (BreachDecision.EscalateTo) policy.onBreach(ctx);
        assertThat(decision.deadline()).isEqualTo(Duration.ofHours(4));
    }

    // ── Blank escalation group ────────────────────────────────────────────────

    @Test
    void blankEscalationGroup_returnsSafeFail() {
        var prefs = new MapPreferences(Map.of("devtown.sla.escalation-group", ""));
        var ctx = ctx(Set.of("pr-reviewers"), prefs);
        var decision = (BreachDecision.Fail) policy.onBreach(ctx);
        assertThat(decision.reason()).isEqualTo("escalation-group-not-configured");
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private static Preferences defaultPrefs() {
        return new MapPreferences(Map.of());
    }

    private static SlaBreachContext ctx(Set<String> groups, Preferences prefs) {
        return ctxType(BreachType.COMPLETION_EXPIRED, groups, prefs);
    }

    private static SlaBreachContext ctxType(BreachType type, Set<String> groups, Preferences prefs) {
        var task = new BreachedTask(UUID.randomUUID(), "case:x/pi:y", "PR review", groups);
        return new SlaBreachContext(type, task, Path.root(), prefs);
    }
}
```

- [ ] **Step 2: Run tests to confirm they fail (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain \
  -Dtest=DefaultSlaBreachPolicyTest --batch-mode 2>&1 | tail -20
```

Expected: compilation error `cannot find symbol: class DefaultSlaBreachPolicy`.

- [ ] **Step 3: Implement `DefaultSlaBreachPolicy`**

```java
// domain/src/main/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicy.java
package io.casehub.devtown.domain.sla;

import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.work.api.SlaBreachPolicy;
import java.time.Duration;

public class DefaultSlaBreachPolicy implements SlaBreachPolicy {

    @Override
    public BreachDecision onBreach(SlaBreachContext ctx) {
        var p = ctx.preferences();
        var escalationGroup = p.getOrDefault(SlaPreferenceKeys.ESCALATION_GROUP).value();
        var terminalReason  = p.getOrDefault(SlaPreferenceKeys.BREACH_TERMINAL_REASON).value();
        int escalationHours = p.getOrDefault(SlaPreferenceKeys.ESCALATION_HOURS).value();

        if (escalationGroup.isBlank()) {
            return new BreachDecision.Fail("escalation-group-not-configured");
        }
        if (ctx.task().candidateGroups().contains(escalationGroup)) {
            return new BreachDecision.Fail(terminalReason);
        }
        return BreachDecision.EscalateTo.to(escalationGroup)
                .withDeadline(Duration.ofHours(escalationHours));
    }
}
```

- [ ] **Step 4: Run tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain \
  -Dtest=DefaultSlaBreachPolicyTest --batch-mode 2>&1 | tail -10
```

Expected: `Tests run: 8, Failures: 0, Errors: 0, Skipped: 0` and `BUILD SUCCESS`.

- [ ] **Step 5: Run full domain test suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain --batch-mode 2>&1 | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  domain/src/main/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicy.java \
  domain/src/test/java/io/casehub/devtown/domain/sla/DefaultSlaBreachPolicyTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m \
  "feat(domain): DefaultSlaBreachPolicy — stateless two-tier SLA escalation — Refs #41"
```

---

## Task 3: Update pr-review.yaml

**Files:**
- Modify: `review/src/main/resources/devtown/pr-review.yaml`

- [ ] **Step 1: Add `candidateGroups` and `expiresIn` to the `human-approval` binding**

In `review/src/main/resources/devtown/pr-review.yaml`, find the `human-approval` binding (in Group 3) and replace it:

```yaml
    ## Group 3: Human gate — fires alongside CI when PR exceeds threshold
    - name: human-approval
      on: { contextChange: {} }
      when: ".pr.linesChanged > .policy.humanApprovalThreshold and .humanApproval == null"
      humanTask:
        title: "PR approval required"
        candidateGroups: [pr-reviewers]
        expiresIn: PT24H
        outputMapping: "{ humanApproval: . }"
```

`expiresIn: PT24H` is ISO-8601 duration, parsed by `Duration.parse()` in `CaseDefinitionYamlMapper`.

- [ ] **Step 2: Run existing HITL test — no regression**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=HumanApprovalLifecycleTest --batch-mode 2>&1 | tail -15
```

Expected: `Tests run: 1, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

The test verifies that:
- WorkItem is created by `human-approval` binding
- WorkItem can be completed with `{"status": "approved"}`
- Case context gets `humanApproval.status = "approved"` via outputMapping

Adding `candidateGroups` and `expiresIn` does not break any of these assertions.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  review/src/main/resources/devtown/pr-review.yaml
git -C /Users/mdproctor/claude/casehub/devtown commit -m \
  "feat(review): add candidateGroups and expiresIn to human-approval binding — Refs #41"
```

---

## Task 4: CDI beans and SLA lifecycle integration test

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/SlaBreachPolicyBean.java`
- Create: `app/src/main/java/io/casehub/devtown/app/SlaBreachHandler.java`
- Create: `app/src/test/java/io/casehub/devtown/app/SlaBreachLifecycleTest.java`

- [ ] **Step 1: Write the failing integration test**

The test exercises the full two-tier breach lifecycle:
- Tier 1: WorkItem expires → policy returns EscalateTo(pr-leads) → WorkItem reassigned in-place
- Tier 2: reassigned WorkItem expires → policy returns Fail("sla-breach") → case signaled

```java
// app/src/test/java/io/casehub/devtown/app/SlaBreachLifecycleTest.java
package io.casehub.devtown.app;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.engine.spi.CaseInstanceRepository;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.runtime.service.ExpiryLifecycleService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class SlaBreachLifecycleTest {

    @Inject PrReviewCaseHub         caseHub;
    @Inject WorkItemQueries         workItemQueries;
    @Inject WorkItemStore           workItemStore;
    @Inject ExpiryLifecycleService  expiryService;
    @Inject CaseInstanceRepository  caseInstanceRepository;

    @Test
    void twoTierSlaBreach_signalsCaseOnTerminalFail() throws Exception {

        // Pre-seed all parallel checks so only human-approval is outstanding.
        // PR has linesChanged=600 > humanApprovalThreshold=500, so human-approval fires.
        var pr = Map.<String, Object>of(
                "id", "99", "repo", "casehubio/devtown",
                "linesChanged", 600, "baseRef", "main", "headSha", "def456");
        var policy = Map.<String, Object>of(
                "humanApprovalThreshold", 500,
                "securityReviewRequired", false,
                "requireSeniorApproval", false);
        var initialCtx = Map.<String, Object>of(
                "pr", pr,
                "policy", policy,
                "codeAnalysis",        Map.of("complete", true,
                                               "securitySensitive", false,
                                               "architectureCrossing", false),
                "styleCheck",          Map.of("outcome", "PENDING"),
                "testCoverage",        Map.of("outcome", "PENDING"),
                "performanceAnalysis", Map.of("outcome", "PENDING"),
                "ci",                  Map.of("status", "passing"));

        // ── Checkpoint 1: start case ──────────────────────────────────────────
        UUID caseId = caseHub.startCase(initialCtx).toCompletableFuture().get(5, SECONDS);
        assertThat(caseId).isNotNull();

        // ── Checkpoint 2: WorkItem created with candidateGroups=pr-reviewers ──
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
            var items = workItemQueries.scanAll().stream()
                    .filter(i -> isHumanApprovalFor(i, caseId)).toList();
            assertThat(items).as("human-approval WorkItem").hasSize(1);
            assertThat(items.get(0).candidateGroups)
                    .as("candidateGroups from YAML").isEqualTo("pr-reviewers");
            assertThat(items.get(0).expiresAt)
                    .as("expiresAt set from expiresIn: PT24H").isNotNull();
        });

        // ── Checkpoint 3: trigger Tier 1 breach ───────────────────────────────
        // Set expiresAt to past so checkExpired() picks it up.
        expireWorkItem(caseId);
        expiryService.checkExpired();

        // ExpiryLifecycleService.executeEscalateTo() mutates WorkItem in-place:
        // candidateGroups → pr-leads, status → PENDING, new expiresAt.
        await().atMost(3, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
            var items = workItemQueries.scanAll().stream()
                    .filter(i -> isHumanApprovalFor(i, caseId)).toList();
            assertThat(items).hasSize(1);
            assertThat(items.get(0).candidateGroups)
                    .as("escalated to pr-leads").isEqualTo("pr-leads");
            assertThat(items.get(0).status)
                    .as("WorkItem is PENDING again after in-place escalation")
                    .isEqualTo(WorkItemStatus.PENDING);
        });

        // Case context is unchanged after escalation (no case signal on EscalateTo).
        var instanceAfterTier1 = caseInstanceRepository.findByUuid(caseId)
                .await().atMost(java.time.Duration.ofSeconds(2));
        assertThat(instanceAfterTier1.getCaseContext().getPath("humanApproval"))
                .as("humanApproval context unchanged after escalation").isNull();

        // ── Checkpoint 4: trigger Tier 2 breach ───────────────────────────────
        expireWorkItem(caseId);
        expiryService.checkExpired();

        // SlaBreachHandler observes SlaBreachEvent (synchronous CDI fire from checkExpired).
        // It calls caseHub.signal(caseId, "humanApproval", Map.of("status", "sla-breach")).
        // signal() fires CONTEXT_CHANGED on Vert.x event bus — async. Use Awaitility.
        await().atMost(5, SECONDS).pollInterval(100, MILLISECONDS).untilAsserted(() -> {
            var instance = caseInstanceRepository.findByUuid(caseId)
                    .await().atMost(java.time.Duration.ofSeconds(2));
            assertThat(instance).isNotNull();
            Object status = instance.getCaseContext().getPath("humanApproval.status");
            assertThat(status)
                    .as("humanApproval.status should be sla-breach after terminal breach")
                    .isEqualTo("sla-breach");
        });

        // The WorkItem itself should be EXPIRED (terminal).
        var finalItems = workItemQueries.scanAll().stream()
                .filter(i -> isHumanApprovalFor(i, caseId)).toList();
        assertThat(finalItems).hasSize(1);
        assertThat(finalItems.get(0).status).isEqualTo(WorkItemStatus.EXPIRED);
        assertThat(finalItems.get(0).resolution).isEqualTo("sla-breach");
    }

    @Transactional
    void expireWorkItem(UUID caseId) {
        workItemQueries.scanAll().stream()
                .filter(i -> isHumanApprovalFor(i, caseId)
                          && i.status == WorkItemStatus.PENDING)
                .forEach(i -> {
                    i.expiresAt = Instant.now().minusSeconds(60);
                    workItemStore.put(i);
                });
    }

    private static boolean isHumanApprovalFor(WorkItem item, UUID caseId) {
        return item.callerRef != null && item.callerRef.contains(caseId.toString());
    }
}
```

- [ ] **Step 2: Run test to confirm it fails (beans not yet created)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=SlaBreachLifecycleTest --batch-mode 2>&1 | tail -20
```

Expected: CDI deployment failure — `SlaBreachPolicy` has no implementation other than `NoOpSlaBreachPolicy @DefaultBean`. The test should fail at augmentation or startup. (If it fails at checkpoint 3 because SlaBreachHandler doesn't exist yet, that's also fine.)

- [ ] **Step 3: Create `SlaBreachPolicyBean`**

```java
// app/src/main/java/io/casehub/devtown/app/SlaBreachPolicyBean.java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.sla.DefaultSlaBreachPolicy;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class SlaBreachPolicyBean extends DefaultSlaBreachPolicy {}
```

No `@DefaultBean` — this takes CDI priority over `NoOpSlaBreachPolicy @DefaultBean` in `casehub-work-runtime`. This is the `@DefaultBean` displacement pattern used throughout devtown.

- [ ] **Step 4: Create `SlaBreachHandler`**

```java
// app/src/main/java/io/casehub/devtown/app/SlaBreachHandler.java
package io.casehub.devtown.app;

import io.casehub.work.api.BreachDecision;
import io.casehub.work.runtime.event.SlaBreachEvent;
import io.casehub.workadapter.CallerRef;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import java.util.Map;
import org.jboss.logging.Logger;

@ApplicationScoped
public class SlaBreachHandler {

    private static final Logger LOG = Logger.getLogger(SlaBreachHandler.class);

    @Inject PrReviewCaseHub caseHub;

    void onBreach(@Observes SlaBreachEvent event) {
        try {
            CallerRef ref = CallerRef.parse(event.context().task().callerRef());
            if (ref == null) return;

            switch (event.decision()) {
                case BreachDecision.Fail fail ->
                    caseHub.signal(ref.caseId(), "humanApproval",
                        Map.of("status", fail.reason()));
                default -> {}
            }
        } catch (Exception e) {
            LOG.errorf(e, "SlaBreachHandler failed for callerRef=%s — case may not be signaled",
                event.context().task().callerRef());
        }
    }
}
```

**Why `@Observes` (not `@ObservesAsync`):** `ExpiryLifecycleService` fires `SlaBreachEvent` via `Event.fire()` (synchronous CDI delivery). `@ObservesAsync` observers are only notified via `Event.fireAsync()` — they would never receive this event. The try-catch protects the surrounding `@Transactional` boundary in `checkExpired()`.

- [ ] **Step 5: Run the test — should now reach checkpoint 4**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app \
  -Dtest=SlaBreachLifecycleTest --batch-mode 2>&1 | tail -25
```

Expected: `Tests run: 1, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

If the test fails at checkpoint 4 (Awaitility timeout on `humanApproval.status`), verify:
1. `SlaBreachHandler.onBreach` is being called — add a temporary `LOG.info` before the switch
2. `caseHub.signal()` is reaching the case — check that `caseId` parsed from callerRef is correct
3. The `CaseInstance` context is being read correctly

- [ ] **Step 6: Run full app test suite — no regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app --batch-mode 2>&1 | tail -15
```

Expected: All existing tests pass plus the new `SlaBreachLifecycleTest`.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/SlaBreachPolicyBean.java \
  app/src/main/java/io/casehub/devtown/app/SlaBreachHandler.java \
  app/src/test/java/io/casehub/devtown/app/SlaBreachLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m \
  "feat(app): SlaBreachPolicyBean + SlaBreachHandler — two-tier SLA breach signaling — Refs #41"
```

---

## Task 5: Full build verification

- [ ] **Step 1: Run full build from devtown root**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install --batch-mode 2>&1 | tail -20
```

Expected: `BUILD SUCCESS`. All modules compile; all tests pass.

- [ ] **Step 2: Verify Layer 1 gap comment is still present in NaivePrReviewService**

The Layer 1 gap comment must remain unchanged — it is the pedagogical marker for Layer 2:

```bash
grep "LAYER 1 GAP: no response SLA" \
  /Users/mdproctor/claude/casehub/devtown/app/src/main/java/io/casehub/devtown/app/NaivePrReviewService.java
```

Expected: the exact gap comment is still present. `NaivePrReviewService` is never modified — Layer 2 displaces it via CDI, not by editing it.

---

## Self-review

**Spec coverage check:**
- ✅ `casehub-work-api` added to `domain/pom.xml` (Task 1)
- ✅ `StringPreference` / `IntPreference` records (Task 1)
- ✅ `SlaPreferenceKeys` with all five keys and defaults (Task 1)
- ✅ `DefaultSlaBreachPolicy` — stateless two-tier via `candidateGroups` (Task 2)
- ✅ Unit tests: first breach, second breach, blank group, custom values (Task 2)
- ✅ `pr-review.yaml` `candidateGroups` + `expiresIn: PT24H` (Task 3)
- ✅ `SlaBreachPolicyBean @ApplicationScoped` displacing NoOp (Task 4)
- ✅ `SlaBreachHandler @Observes SlaBreachEvent` (Task 4)
- ✅ Integration test: full two-tier breach lifecycle (Task 4)
- ✅ `@Observes` (not `@ObservesAsync`) rationale documented — `Event.fire()` from `ExpiryLifecycleService` (Task 4)
- ✅ Failure goal deferred (engine#326) — not implemented; out of scope for Layer 2
- ✅ `BreachDecision.Extend` deferred (work#211) — `default -> {}` covers it safely

**Placeholder scan:** None found.

**Type consistency:**
- `SlaPreferenceKeys.ESCALATION_GROUP` returns `StringPreference` → `.value()` returns `String` — used correctly in `DefaultSlaBreachPolicy`
- `SlaPreferenceKeys.ESCALATION_HOURS` returns `IntPreference` → `.value()` returns `int` — used correctly as `Duration.ofHours(escalationHours)`
- `BreachDecision.EscalateTo.to(String...)` factory — matches API in `casehub-work-api`
- `.withDeadline(Duration)` returns `EscalateTo` — correct
- `CallerRef.parse(String)` returns `CallerRef` or `null` — null-checked in handler
- `caseHub.signal(UUID, String, Object)` — matches `CaseHub.signal()` signature
