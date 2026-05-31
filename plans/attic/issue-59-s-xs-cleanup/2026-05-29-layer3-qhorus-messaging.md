# Layer 3 — casehub-qhorus Typed Messaging Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Layer 3 to devtown — typed COMMAND/RESPONSE/DONE/DECLINE per specialist reviewer via casehub-qhorus channels, establishing formal obligation for every review assignment.

**Architecture:** `QhorusPrReviewService @Alternative @Priority(1)` in `app/` implements `PrReviewApplicationService`. It creates a normative 3-channel set per PR review (`pr-review-{n}/work`, `/observe`, `/oversight`), dispatches a `COMMAND` to each specialist on `/work` with `target=capability`, calls the agent synchronously, then sends `DONE` or `DECLINE` with `inReplyTo=commandResult.messageId()`. Agent stubs live in `app/agents/`. Two new port types (`ReviewerOutcome`, `ReviewerAgent`) live in `review/`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-qhorus 0.2-SNAPSHOT, JUnit 5, AssertJ.

**Spec:** `specs/2026-05-29-layer3-qhorus-messaging-design.md`

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `review/src/main/java/io/casehub/devtown/review/ReviewerOutcome.java` | CREATE | Sealed interface: `Completed(findings)`, `Declined(reason)` |
| `review/src/main/java/io/casehub/devtown/review/ReviewerAgent.java` | CREATE | Driven-port interface: `capability()`, `handle(PrPayload)` |
| `review/src/test/java/io/casehub/devtown/review/ReviewerOutcomeTest.java` | CREATE | Unit tests for sealed interface |
| `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java` | MODIFY | Add `@Alternative @Priority(2)` |
| `app/src/main/java/io/casehub/devtown/app/agents/SecurityReviewAgent.java` | CREATE | Stub: always `Completed` |
| `app/src/main/java/io/casehub/devtown/app/agents/ArchitectureReviewAgent.java` | CREATE | Stub: always `Declined` |
| `app/src/main/java/io/casehub/devtown/app/agents/TestCoverageReviewAgent.java` | CREATE | Stub: always `Completed` |
| `app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java` | CREATE | Layer 3 implementation |
| `app/src/test/java/io/casehub/devtown/app/PrReviewQhorusLifecycleTest.java` | CREATE | 5-method `@QuarkusTest` |

No `pom.xml` changes — `casehub-qhorus` and `casehub-qhorus-testing` are already in `app/pom.xml`.

---

## Task 1: `ReviewerOutcome` sealed interface + unit test

**Files:**
- Create: `review/src/main/java/io/casehub/devtown/review/ReviewerOutcome.java`
- Create: `review/src/test/java/io/casehub/devtown/review/ReviewerOutcomeTest.java`

- [ ] **Step 1.1: Write `ReviewerOutcome`**

```java
// review/src/main/java/io/casehub/devtown/review/ReviewerOutcome.java
package io.casehub.devtown.review;

import java.util.List;

public sealed interface ReviewerOutcome
        permits ReviewerOutcome.Completed, ReviewerOutcome.Declined {

    record Completed(List<String> findings) implements ReviewerOutcome {}
    record Declined(String reason)          implements ReviewerOutcome {}
}
```

- [ ] **Step 1.2: Write the failing unit tests**

```java
// review/src/test/java/io/casehub/devtown/review/ReviewerOutcomeTest.java
package io.casehub.devtown.review;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class ReviewerOutcomeTest {

    @Test
    void completed_holdsFindings() {
        var outcome = new ReviewerOutcome.Completed(List.of("finding-1", "finding-2"));
        assertThat(outcome.findings()).containsExactly("finding-1", "finding-2");
    }

    @Test
    void declined_holdsReason() {
        var outcome = new ReviewerOutcome.Declined("out of scope");
        assertThat(outcome.reason()).isEqualTo("out of scope");
    }

    @Test
    void patternMatch_coversAllPermits() {
        ReviewerOutcome completed = new ReviewerOutcome.Completed(List.of("f1"));
        ReviewerOutcome declined  = new ReviewerOutcome.Declined("reason");

        String c = switch (completed) {
            case ReviewerOutcome.Completed x -> "completed:" + x.findings().size();
            case ReviewerOutcome.Declined x  -> "declined";
        };
        String d = switch (declined) {
            case ReviewerOutcome.Completed x -> "completed";
            case ReviewerOutcome.Declined x  -> "declined:" + x.reason();
        };

        assertThat(c).isEqualTo("completed:1");
        assertThat(d).isEqualTo("declined:reason");
    }
}
```

- [ ] **Step 1.3: Run unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl review test -Dtest=ReviewerOutcomeTest -q
```

Expected: `BUILD SUCCESS`, 3 tests pass.

- [ ] **Step 1.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  review/src/main/java/io/casehub/devtown/review/ReviewerOutcome.java \
  review/src/test/java/io/casehub/devtown/review/ReviewerOutcomeTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(layer3): add ReviewerOutcome sealed interface (devtown#52)"
```

---

## Task 2: `ReviewerAgent` interface

**Files:**
- Create: `review/src/main/java/io/casehub/devtown/review/ReviewerAgent.java`

- [ ] **Step 2.1: Write `ReviewerAgent`**

No standalone test — tested implicitly through the @QuarkusTest in Task 5.

```java
// review/src/main/java/io/casehub/devtown/review/ReviewerAgent.java
package io.casehub.devtown.review;

public interface ReviewerAgent {
    /**
     * Returns the ReviewDomain capability constant this agent handles.
     * Used as the COMMAND target and CDI dispatch key.
     */
    String capability();

    /**
     * Handle a review request. Called synchronously; Layer 3 stubs are in-process.
     * Intentional divergence from AML's AgentBehaviour.handle(Message command): we pass
     * the typed domain object to avoid a nullable Message parameter and a qhorus type
     * leak into the port interface.
     */
    ReviewerOutcome handle(PrPayload pr);
}
```

- [ ] **Step 2.2: Verify compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl review compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 2.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  review/src/main/java/io/casehub/devtown/review/ReviewerAgent.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(layer3): add ReviewerAgent driven-port interface (devtown#52)"
```

---

## Task 3: Annotate `PrReviewCaseService` with `@Alternative @Priority(2)`

Without this change, adding `QhorusPrReviewService @Alternative @Priority(1)` alongside the existing `PrReviewCaseService @ApplicationScoped` would cause `AmbiguousResolutionException` at startup — two non-`@DefaultBean` beans implementing the same interface.

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

- [ ] **Step 3.1: Add `@Alternative @Priority(2)` to `PrReviewCaseService`**

```java
// app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java
package io.casehub.devtown.app;

import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.PrReviewApplicationService;
import io.casehub.devtown.review.PrReviewOutcome;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
@Alternative
@Priority(2)
public class PrReviewCaseService implements PrReviewApplicationService {

    private static final String VERDICT_CASE_OPENED = "case-opened";

    @Inject
    PrReviewCaseHub caseHub;

    @ConfigProperty(name = "devtown.policy.human-approval-threshold", defaultValue = "500")
    int humanApprovalThreshold;

    @ConfigProperty(name = "devtown.policy.security-review-required", defaultValue = "true")
    boolean securityReviewRequired;

    @ConfigProperty(name = "devtown.policy.require-senior-approval", defaultValue = "false")
    boolean requireSeniorApproval;

    @Override
    public PrReviewOutcome review(PrPayload pr) {
        // TODO(parent#26): replace @ConfigProperty injection with PreferenceProvider.resolve(scope).asMap()
        var policy = Map.<String, Object>of(
            "humanApprovalThreshold", humanApprovalThreshold,
            "securityReviewRequired", securityReviewRequired,
            "requireSeniorApproval", requireSeniorApproval
        );
        var prContext = Map.<String, Object>of(
            "id", String.valueOf(pr.prNumber()),
            "repo", pr.repo(),
            "linesChanged", pr.linesChanged(),
            "baseRef", pr.baseRef(),
            "headSha", pr.headSha()
        );
        var initialContext = Map.<String, Object>of(
            "pr", prContext,
            "policy", policy
        );
        // CompletionStage<UUID> case ID — not surfaced in PrReviewOutcome until Layer 6 adds case tracking (devtown#10)
        caseHub.startCase(initialContext);
        return new PrReviewOutcome(VERDICT_CASE_OPENED, List.of());
    }
}
```

- [ ] **Step 3.2: Compile `app/` to verify no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "refactor(layer3): add @Alternative @Priority(2) to PrReviewCaseService (devtown#52)"
```

---

## Task 4: Agent stubs

Three `@ApplicationScoped` beans. `ArchitectureReviewAgent` always DECLINEs — deterministic for tests and matches the tutorial narrative.

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/agents/SecurityReviewAgent.java`
- Create: `app/src/main/java/io/casehub/devtown/app/agents/ArchitectureReviewAgent.java`
- Create: `app/src/main/java/io/casehub/devtown/app/agents/TestCoverageReviewAgent.java`

- [ ] **Step 4.1: Write `SecurityReviewAgent`**

```java
// app/src/main/java/io/casehub/devtown/app/agents/SecurityReviewAgent.java
package io.casehub.devtown.app.agents;

import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.ReviewerAgent;
import io.casehub.devtown.review.ReviewerOutcome;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@ApplicationScoped
public class SecurityReviewAgent implements ReviewerAgent {

    @Override
    public String capability() {
        return ReviewDomain.SECURITY_REVIEW;
    }

    @Override
    public ReviewerOutcome handle(PrPayload pr) {
        return new ReviewerOutcome.Completed(List.of("rate-limiting absent on /payment"));
    }
}
```

- [ ] **Step 4.2: Write `ArchitectureReviewAgent`**

```java
// app/src/main/java/io/casehub/devtown/app/agents/ArchitectureReviewAgent.java
package io.casehub.devtown.app.agents;

import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.ReviewerAgent;
import io.casehub.devtown.review.ReviewerOutcome;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class ArchitectureReviewAgent implements ReviewerAgent {

    @Override
    public String capability() {
        return ReviewDomain.ARCHITECTURE_REVIEW;
    }

    @Override
    public ReviewerOutcome handle(PrPayload pr) {
        return new ReviewerOutcome.Declined("distributed transaction outside scope");
    }
}
```

- [ ] **Step 4.3: Write `TestCoverageReviewAgent`**

```java
// app/src/main/java/io/casehub/devtown/app/agents/TestCoverageReviewAgent.java
package io.casehub.devtown.app.agents;

import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.ReviewerAgent;
import io.casehub.devtown.review.ReviewerOutcome;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@ApplicationScoped
public class TestCoverageReviewAgent implements ReviewerAgent {

    @Override
    public String capability() {
        return ReviewDomain.TEST_COVERAGE;
    }

    @Override
    public ReviewerOutcome handle(PrPayload pr) {
        return new ReviewerOutcome.Completed(List.of("coverage 67%; payment path untested"));
    }
}
```

- [ ] **Step 4.4: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/agents/SecurityReviewAgent.java \
  app/src/main/java/io/casehub/devtown/app/agents/ArchitectureReviewAgent.java \
  app/src/main/java/io/casehub/devtown/app/agents/TestCoverageReviewAgent.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(layer3): add specialist reviewer agent stubs (devtown#52)"
```

---

## Task 5: `PrReviewQhorusLifecycleTest` + `QhorusPrReviewService`

These are written together — the test drives the implementation. Write both, then run. Each test uses a distinct PR number to avoid cross-test contamination in the `InMemoryMessageStore` (which accumulates across test methods in the same `@QuarkusTest` session).

**API facts (decompiled from jars, use exactly these):**
- `ChannelService.create(name, owner, ChannelSemantic, allowedWriters)` — 4-arg
- `ChannelService.findByName(String)` → `Optional<Channel>`; `Channel.id` is `UUID`
- `MessageService.dispatch(MessageDispatch)` → `DispatchResult`; `DispatchResult.messageId()` is `Long`
- `MessageService.pollAfter(UUID channelId, Long afterSequence, int limit)` → `List<Message>`
- `Message.messageType` is a field (`MessageType`), `Message.target` is `String`, `Message.content` is `String`
- `CommitmentStore.findOpenByObligor(String obligor, UUID channelId)` → `List<Commitment>`
- `MessageDispatch.builder()` — no-arg factory, then chain `.channelId(UUID).sender(String).type(MessageType).content(String).correlationId(String).inReplyTo(Long).target(String).actorType(ActorType).build()`
- `CommitmentState` is `io.casehub.qhorus.api.message.CommitmentState`

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/PrReviewQhorusLifecycleTest.java`
- Create: `app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java`

- [ ] **Step 5.1: Write `PrReviewQhorusLifecycleTest`**

```java
// app/src/test/java/io/casehub/devtown/app/PrReviewQhorusLifecycleTest.java
package io.casehub.devtown.app;

import io.casehub.devtown.domain.ReviewDomain;
import io.casehub.devtown.review.PrPayload;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.store.CommitmentStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.List;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class PrReviewQhorusLifecycleTest {

    // Injected by concrete type — CDI selects exactly QhorusPrReviewService regardless
    // of @Priority ordering. PrReviewCaseService (@Priority(2)) is selected for the
    // PrReviewApplicationService injection point but not for this direct-type injection.
    @Inject QhorusPrReviewService service;

    @Inject ChannelService  channelService;
    @Inject MessageService  messageService;
    @Inject CommitmentStore commitmentStore;

    // Each test uses a distinct PR number to prevent cross-test contamination:
    // InMemoryMessageStore accumulates across test methods in the same @QuarkusTest session.

    @Test
    void channelsCreated() {
        var pr = new PrPayload("casehubio/devtown", 201, "sha201", "main", 100);

        service.review(pr);

        assertThat(channelService.findByName("pr-review-201/work")).isPresent();
        assertThat(channelService.findByName("pr-review-201/observe")).isPresent();
        assertThat(channelService.findByName("pr-review-201/oversight")).isPresent();
    }

    @Test
    void commandsDispatched() {
        var pr = new PrPayload("casehubio/devtown", 202, "sha202", "main", 100);

        service.review(pr);

        var work = channelService.findByName("pr-review-202/work").orElseThrow();
        List<Message> messages = messageService.pollAfter(work.id, 0L, 100);
        List<Message> commands = messages.stream()
                .filter(m -> m.messageType == MessageType.COMMAND)
                .toList();

        assertThat(commands).hasSize(3);
        assertThat(commands).extracting(m -> m.target)
                .containsExactlyInAnyOrder(
                        ReviewDomain.SECURITY_REVIEW,
                        ReviewDomain.ARCHITECTURE_REVIEW,
                        ReviewDomain.TEST_COVERAGE);
    }

    @Test
    void doneDischargesCommitment() {
        var pr = new PrPayload("casehubio/devtown", 203, "sha203", "main", 100);

        service.review(pr);

        var work = channelService.findByName("pr-review-203/work").orElseThrow();

        // DONE from security and test-coverage agents discharges their commitments (FULFILLED)
        assertThat(commitmentStore.findOpenByObligor(ReviewDomain.SECURITY_REVIEW, work.id))
                .as("security-review commitment should be discharged after DONE")
                .isEmpty();
        assertThat(commitmentStore.findOpenByObligor(ReviewDomain.TEST_COVERAGE, work.id))
                .as("test-coverage commitment should be discharged after DONE")
                .isEmpty();
    }

    @Test
    void declineRecorded() {
        var pr = new PrPayload("casehubio/devtown", 204, "sha204", "main", 100);

        service.review(pr);

        var work = channelService.findByName("pr-review-204/work").orElseThrow();
        List<Message> declines = messageService.pollAfter(work.id, 0L, 100).stream()
                .filter(m -> m.messageType == MessageType.DECLINE)
                .toList();

        assertThat(declines).as("exactly one DECLINE from ArchitectureReviewAgent").hasSize(1);

        // DECLINE also closes the commitment (state = DECLINED, not OPEN)
        assertThat(commitmentStore.findOpenByObligor(ReviewDomain.ARCHITECTURE_REVIEW, work.id))
                .as("architecture-review commitment should be closed after DECLINE")
                .isEmpty();
    }

    @Test
    void declineContentRecorded() {
        var pr = new PrPayload("casehubio/devtown", 205, "sha205", "main", 100);

        service.review(pr);

        var work = channelService.findByName("pr-review-205/work").orElseThrow();
        Message decline = messageService.pollAfter(work.id, 0L, 100).stream()
                .filter(m -> m.messageType == MessageType.DECLINE)
                .findFirst()
                .orElseThrow(() -> new AssertionError("No DECLINE message found on /work"));

        assertThat(decline.content)
                .isEqualTo("distributed transaction outside scope");
    }
}
```

- [ ] **Step 5.2: Write `QhorusPrReviewService`**

```java
// app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java
package io.casehub.devtown.app;

import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.PrReviewApplicationService;
import io.casehub.devtown.review.PrReviewOutcome;
import io.casehub.devtown.review.ReviewerAgent;
import io.casehub.devtown.review.ReviewerOutcome;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

/**
 * Layer 3: replaces direct specialist invocation (Layer 1) with typed speech-act messaging.
 * Each specialist review request becomes a COMMAND that creates a Commitment; DONE fulfils it;
 * DECLINE is a formal scope boundary with a recorded reason.
 *
 * CDI priority: @Priority(1) — present in full build for tutorial reading; inactive at runtime
 * (PrReviewCaseService @Priority(2) wins). PrReviewService @DefaultBean is the Layer 1 fallback.
 */
@ApplicationScoped
@Alternative
@Priority(1)
public class QhorusPrReviewService implements PrReviewApplicationService {

    private static final String ORCHESTRATOR = "pr-orchestrator";

    @Inject
    ChannelService channelService;

    @Inject
    MessageService messageService;

    @Inject
    Instance<ReviewerAgent> agents;

    @Override
    public PrReviewOutcome review(PrPayload pr) {
        String prefix = "pr-review-" + pr.prNumber();
        Channel work = findOrCreate(prefix + "/work");
        findOrCreate(prefix + "/observe");
        findOrCreate(prefix + "/oversight");

        List<String> allFindings = new ArrayList<>();

        for (ReviewerAgent agent : agents) {
            String correlationId = UUID.randomUUID().toString();

            var commandResult = messageService.dispatch(MessageDispatch.builder()
                    .channelId(work.id)
                    .sender(ORCHESTRATOR)
                    .type(MessageType.COMMAND)
                    .content(pr.repo() + "#" + pr.prNumber())
                    .correlationId(correlationId)
                    .target(agent.capability())
                    .actorType(ActorType.SYSTEM)
                    .build());

            ReviewerOutcome outcome = agent.handle(pr);

            switch (outcome) {
                case ReviewerOutcome.Completed completed -> {
                    messageService.dispatch(MessageDispatch.builder()
                            .channelId(work.id)
                            .sender(agent.capability())
                            .type(MessageType.DONE)
                            .content(String.join("; ", completed.findings()))
                            .correlationId(correlationId)
                            .inReplyTo(commandResult.messageId())
                            .actorType(ActorType.AGENT)
                            .build());
                    allFindings.addAll(completed.findings());
                }
                case ReviewerOutcome.Declined declined ->
                    messageService.dispatch(MessageDispatch.builder()
                            .channelId(work.id)
                            .sender(agent.capability())
                            .type(MessageType.DECLINE)
                            .content(declined.reason())
                            .correlationId(correlationId)
                            .inReplyTo(commandResult.messageId())
                            .actorType(ActorType.AGENT)
                            .build());
            }
        }

        return new PrReviewOutcome("qhorus-reviewed", allFindings);
    }

    private Channel findOrCreate(String name) {
        return channelService.findByName(name)
                .orElseGet(() -> channelService.create(name, ORCHESTRATOR, ChannelSemantic.APPEND, ORCHESTRATOR));
    }
}
```

- [ ] **Step 5.3: Compile `app/` (should succeed — all referenced types now exist)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app compile -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 5.4: Run only `PrReviewQhorusLifecycleTest`**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl app test -Dtest=PrReviewQhorusLifecycleTest -q
```

Expected: `BUILD SUCCESS`, all 5 tests pass.

If any test fails, diagnose before continuing. Common issues:
- `NullPointerException` on `commitmentStore` — check that `casehub-qhorus-testing` is indexed in `src/test/resources/application.properties`; the `InMemoryCommitmentStore` must be discoverable
- `AmbiguousResolutionException` at startup — check that `PrReviewCaseService` has `@Alternative @Priority(2)` and `QhorusPrReviewService` has `@Alternative @Priority(1)`
- `commitmentStore.findOpenByObligor` returns non-empty after DONE — confirm `target=agent.capability()` is set on COMMAND (this sets `obligor=capability` in the Commitment; without `target`, `obligor=null` and `findOpenByObligor` would find nothing)

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java \
  app/src/test/java/io/casehub/devtown/app/PrReviewQhorusLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(layer3): add QhorusPrReviewService and lifecycle tests (devtown#52)"
```

---

## Task 6: Full test suite + final commit

- [ ] **Step 6.1: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q
```

Expected: `BUILD SUCCESS`. All existing tests (`HumanApprovalLifecycleTest`, `PrReviewCaseHubTest`, `PrReviewServiceTest`, `SlaBreachLifecycleTest`, etc.) continue to pass.

If any existing test fails with `AmbiguousResolutionException`, the `@Alternative @Priority` annotation on `PrReviewCaseService` is missing or incorrect — verify Task 3.

- [ ] **Step 6.2: Commit workspace HANDOFF update**

```bash
# Update workspace HANDOFF.md with current state, then commit to workspace main.
# (Done at session end via handover skill — not part of code commit.)
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] `ReviewerOutcome` sealed interface — Task 1
- [x] `ReviewerAgent` interface in `review/` — Task 2
- [x] `PrReviewCaseService @Alternative @Priority(2)` — Task 3
- [x] Three agent stubs in `app/agents/` — Task 4
- [x] `QhorusPrReviewService @Alternative @Priority(1)` — Task 5
- [x] Normative 3-channel layout per PR — Task 5 (findOrCreate work/observe/oversight)
- [x] COMMAND with `target=capability`, `sender=ORCHESTRATOR`, `content=repo#prNumber` — Task 5
- [x] DONE with `sender=capability`, `content=findings.join("; ")`, `inReplyTo=commandResult.messageId()` — Task 5
- [x] DECLINE with `sender=capability`, `content=reason`, `inReplyTo=commandResult.messageId()` — Task 5
- [x] `channelsCreated` test — Task 5
- [x] `commandsDispatched` test (3 COMMANDs, each with target) — Task 5
- [x] `doneDischargesCommitment` test — Task 5
- [x] `declineRecorded` test — Task 5
- [x] `declineContentRecorded` test — Task 5
- [x] No `pom.xml` changes — confirmed in file map
- [x] `LAYER-LOG.md` Layer 3 entry — **NOT in this plan; written at layer close (work-end)**
