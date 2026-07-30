# Notification Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #16 — Epic 9: Notification wiring — casehub-connectors integration
**Issue group:** #16

**Goal:** Wire devtown's 6 notification scenarios into the platform notification system via `SubscribableEvent` POJOs, bridge observers, and subscription registrations.

**Architecture:** Bridge observers in `app/notification/` translate foundation CDI events (`CaseLifecycleEvent`, `WatchdogAlertEvent`, `WorkItemLifecycleEvent`, `SlaBreachEvent`) into `SubscribableEvent` POJOs (in `review/notification/`) and fire them as CDI events. A startup registrar creates `Subscription` definitions via `SubscriptionStore`. Per-repo connector targets resolve via `PreferenceProvider` in the bridge observers — the resolved channel is carried on the event POJO's `targetChannel` field.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api (SubscribableEvent, SubscriptionStore, NotificationTemplate), casehub-work-api (SlaBreachEvent, BreachDecision), casehub-qhorus-api (WatchdogAlertEvent), casehub-engine-common (CaseLifecycleEvent)

## Global Constraints

- `casehub-platform-api` is already on `review/` and `app/` classpath — no new compile deps needed
- `casehub-qhorus` and `casehub-work` are already on `app/` classpath — source events available
- `subscriptions-inmem` needed as test-scope dependency in `app/pom.xml`
- Observer annotations depend on source event firing mechanism: `@ObservesAsync` for async-fired events (`CaseLifecycleEvent`, `WatchdogAlertEvent`), `@Observes(during = AFTER_SUCCESS)` for sync-fired events (`WorkItemLifecycleEvent`, `SlaBreachEvent`)
- All observers use `@Transactional(NOT_SUPPORTED)` (GE-20260427-893862)
- Suppress Twilio/WhatsApp in test config via `quarkus.arc.exclude-types` (GE-20260521-45e61c)
- `StringPreference` is at `io.casehub.devtown.domain.sla.StringPreference` — reuse, do not duplicate
- `NotificationTemplate` has 8 required fields: `titlePattern`, `bodyPattern` (nullable), `severity`, `category`, `actionUrlPattern` (nullable), `entityType`, `entityIdField`, `actorIdField`
- `SubscriptionInput` has 10 fields: `ownerId`, `tenancyId`, `name`, `eventType`, `filters`, `targets`, `includeActor`, `template`, `enabled`, `scope`
- `SubscriptionQuery` requires: `ownerId` (null for SYSTEM scope), `tenancyId`, `scope`, `enabled` (nullable), `cursor` (nullable), `limit` (positive int)

---

### Task 1: Domain preference keys and SubscribableEvent POJOs

**Files:**
- Create: `domain/src/main/java/io/casehub/devtown/domain/notification/NotificationPreferenceKeys.java`
- Create: `domain/src/test/java/io/casehub/devtown/domain/notification/NotificationPreferenceKeysTest.java`
- Create: `review/src/main/java/io/casehub/devtown/review/notification/ReviewAssignedEvent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/notification/MergeSucceededEvent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/notification/MergeFailedEvent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/notification/StalledCommitmentEvent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/notification/CaseFaultedEvent.java`
- Create: `review/src/main/java/io/casehub/devtown/review/notification/SlaEscalatedEvent.java`
- Create: `review/src/test/java/io/casehub/devtown/review/notification/SubscribableEventTest.java`

**Interfaces:**
- Produces: `NotificationPreferenceKeys.SLACK_CHANNEL` — `PreferenceKey<StringPreference>` with namespace `"devtown.notification"`, name `"slack-channel"`, default `"#devtown-ops"`
- Produces: `NotificationPreferenceKeys.TEAMS_CHANNEL` — `PreferenceKey<StringPreference>` with namespace `"devtown.notification"`, name `"teams-channel"`, default `""`
- Produces: 6 records implementing `SubscribableEvent` — each with `type()` returning `"io.casehub.devtown.<domain>.<action>"` and `tenancyId()` propagating from the source event. All include a `String targetChannel` field.

- [ ] **Step 1: Write preference key tests**

```java
package io.casehub.devtown.domain.notification;

import io.casehub.devtown.domain.sla.StringPreference;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class NotificationPreferenceKeysTest {

    @Test
    void slackChannelHasCorrectQualifiedName() {
        assertEquals("devtown.notification.slack-channel",
            NotificationPreferenceKeys.SLACK_CHANNEL.qualifiedName());
    }

    @Test
    void slackChannelDefaultIsDevtownOps() {
        assertEquals("#devtown-ops",
            NotificationPreferenceKeys.SLACK_CHANNEL.defaultValue().value());
    }

    @Test
    void slackChannelParsesStringValue() {
        StringPreference parsed = NotificationPreferenceKeys.SLACK_CHANNEL.parse("#custom-channel");
        assertEquals("#custom-channel", parsed.value());
    }

    @Test
    void teamsChannelHasCorrectQualifiedName() {
        assertEquals("devtown.notification.teams-channel",
            NotificationPreferenceKeys.TEAMS_CHANNEL.qualifiedName());
    }

    @Test
    void teamsChannelDefaultIsEmpty() {
        assertEquals("", NotificationPreferenceKeys.TEAMS_CHANNEL.defaultValue().value());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=NotificationPreferenceKeysTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `NotificationPreferenceKeys` not found

- [ ] **Step 3: Implement NotificationPreferenceKeys**

Use `ide_create_file`:

```java
package io.casehub.devtown.domain.notification;

import io.casehub.devtown.domain.sla.StringPreference;
import io.casehub.platform.api.preferences.PreferenceKey;

public final class NotificationPreferenceKeys {

    public static final PreferenceKey<StringPreference> SLACK_CHANNEL =
        new PreferenceKey<>("devtown.notification", "slack-channel",
            StringPreference.of("#devtown-ops"), StringPreference::parse);

    public static final PreferenceKey<StringPreference> TEAMS_CHANNEL =
        new PreferenceKey<>("devtown.notification", "teams-channel",
            StringPreference.of(""), StringPreference::parse);

    private NotificationPreferenceKeys() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=NotificationPreferenceKeysTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (5 tests)

- [ ] **Step 5: Write SubscribableEvent POJO tests**

```java
package io.casehub.devtown.review.notification;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SubscribableEventTest {

    @Test
    void reviewAssignedEventType() {
        var e = new ReviewAssignedEvent("wi-1", "user-a", "#ops", "t1");
        assertEquals("io.casehub.devtown.review.assigned", e.type());
        assertEquals("t1", e.tenancyId());
    }

    @Test
    void mergeSucceededEventType() {
        var e = new MergeSucceededEvent("case-1", "pr-review", "42", "2",
            "PR #42, PR #43", "system", "#ops", "t1");
        assertEquals("io.casehub.devtown.merge.succeeded", e.type());
    }

    @Test
    void mergeFailedEventType() {
        var e = new MergeFailedEvent("42", "Fix auth", "user-b", "Bob",
            "CI timeout", "repo-1", "/pr/42", "#ops", "t1");
        assertEquals("io.casehub.devtown.merge.failed", e.type());
    }

    @Test
    void stalledCommitmentEventType() {
        var e = new StalledCommitmentEvent("OBLIGATION_FAN_OUT", "channel-1",
            "2 obligations unresponded", "2026-07-20T10:00:00Z", "system", "#ops", "t1");
        assertEquals("io.casehub.devtown.commitment.stalled", e.type());
    }

    @Test
    void caseFaultedEventType() {
        var e = new CaseFaultedEvent("case-1", "pr-review-v1", "FAULTED",
            null, "#ops", "t1");
        assertEquals("io.casehub.devtown.case.faulted", e.type());
    }

    @Test
    void slaEscalatedEventType() {
        var e = new SlaEscalatedEvent("task-1", "Review PR #42", "devtown:pr-review",
            "COMPLETION", "pr-leads", "/repos/my-repo", "#ops", "t1");
        assertEquals("io.casehub.devtown.sla.escalated", e.type());
    }

    @Test
    void allEventsPropagateTenancyId() {
        String tenancy = "tenant-42";
        assertEquals(tenancy, new ReviewAssignedEvent("w", "u", "#c", tenancy).tenancyId());
        assertEquals(tenancy, new MergeSucceededEvent("c", "d", "p", "n", "l", "a", "#c", tenancy).tenancyId());
        assertEquals(tenancy, new MergeFailedEvent("p", "t", "a", "n", "r", "repo", "u", "#c", tenancy).tenancyId());
        assertEquals(tenancy, new StalledCommitmentEvent("ct", "tn", "s", "f", "a", "#c", tenancy).tenancyId());
        assertEquals(tenancy, new CaseFaultedEvent("c", "d", "s", null, "#c", tenancy).tenancyId());
        assertEquals(tenancy, new SlaEscalatedEvent("t", "ti", "c", "b", "g", "s", "#c", tenancy).tenancyId());
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest=SubscribableEventTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — event classes not found

- [ ] **Step 7: Implement all 6 SubscribableEvent records**

Use `ide_create_file` for each. All in `review/src/main/java/io/casehub/devtown/review/notification/`:

**ReviewAssignedEvent.java:**
```java
package io.casehub.devtown.review.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

public record ReviewAssignedEvent(
    String workItemId,
    String assigneeId,
    String targetChannel,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.review.assigned"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

**MergeSucceededEvent.java:**
```java
package io.casehub.devtown.review.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

public record MergeSucceededEvent(
    String caseId,
    String caseDefinitionName,
    String prNumber,
    String prCount,
    String prList,
    String actorId,
    String targetChannel,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.merge.succeeded"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

**MergeFailedEvent.java:**
```java
package io.casehub.devtown.review.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

public record MergeFailedEvent(
    String prNumber,
    String prTitle,
    String authorId,
    String authorName,
    String failureReason,
    String repoId,
    String prUrl,
    String targetChannel,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.merge.failed"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

**StalledCommitmentEvent.java:**
```java
package io.casehub.devtown.review.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

public record StalledCommitmentEvent(
    String conditionType,
    String targetName,
    String summary,
    String firedAt,
    String actorId,
    String targetChannel,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.commitment.stalled"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

**CaseFaultedEvent.java:**
```java
package io.casehub.devtown.review.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

public record CaseFaultedEvent(
    String caseId,
    String caseDefinitionName,
    String caseStatus,
    String contextSnapshot,
    String targetChannel,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.case.faulted"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

**SlaEscalatedEvent.java:**
```java
package io.casehub.devtown.review.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

public record SlaEscalatedEvent(
    String taskId,
    String taskTitle,
    String callerRef,
    String breachType,
    String escalationGroups,
    String scope,
    String targetChannel,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.sla.escalated"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -Dtest=SubscribableEventTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (7 tests)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add domain/src/main/java/io/casehub/devtown/domain/notification/ domain/src/test/java/io/casehub/devtown/domain/notification/ review/src/main/java/io/casehub/devtown/review/notification/ review/src/test/java/io/casehub/devtown/review/notification/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#16): notification preference keys and SubscribableEvent POJOs"
```

---

### Task 2: Bridge observers

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/notification/CaseLifecycleNotificationBridge.java`
- Create: `app/src/main/java/io/casehub/devtown/app/notification/WatchdogAlertNotificationBridge.java`
- Create: `app/src/main/java/io/casehub/devtown/app/notification/ReviewAssignmentNotificationBridge.java`
- Create: `app/src/main/java/io/casehub/devtown/app/notification/SlaBreachNotificationBridge.java`
- Create: `app/src/test/java/io/casehub/devtown/app/notification/CaseLifecycleNotificationBridgeTest.java`
- Create: `app/src/test/java/io/casehub/devtown/app/notification/WatchdogAlertNotificationBridgeTest.java`
- Create: `app/src/test/java/io/casehub/devtown/app/notification/ReviewAssignmentNotificationBridgeTest.java`
- Create: `app/src/test/java/io/casehub/devtown/app/notification/SlaBreachNotificationBridgeTest.java`

**Interfaces:**
- Consumes: `ReviewAssignedEvent`, `MergeSucceededEvent`, `MergeFailedEvent`, `StalledCommitmentEvent`, `CaseFaultedEvent`, `SlaEscalatedEvent` (from Task 1)
- Consumes: `NotificationPreferenceKeys.SLACK_CHANNEL` (from Task 1)
- Consumes: `PreferenceProvider.resolve(SettingsScope)` → `Preferences.getOrDefault(PreferenceKey<T>)` → `T.value()`
- Produces: CDI events fired via `Event<T>.fire()` for each `SubscribableEvent` type

- [ ] **Step 1: Write CaseLifecycleNotificationBridge test**

This observer handles 3 scenarios: merge success (COMPLETED), merge failure (CANCELLED), case fault (FAULTED). Uses `@ObservesAsync` because `CaseLifecycleEvent` is fired via `Event.fireAsync()`.

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.domain.notification.NotificationPreferenceKeys;
import io.casehub.devtown.domain.sla.StringPreference;
import io.casehub.devtown.review.notification.CaseFaultedEvent;
import io.casehub.devtown.review.notification.MergeFailedEvent;
import io.casehub.devtown.review.notification.MergeSucceededEvent;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class CaseLifecycleNotificationBridgeTest {

    private CaseLifecycleNotificationBridge bridge;
    private List<MergeSucceededEvent> mergeSucceededFired;
    private List<MergeFailedEvent> mergeFailedFired;
    private List<CaseFaultedEvent> caseFaultedFired;
    private PreferenceProvider preferenceProvider;

    @BeforeEach
    void setUp() {
        mergeSucceededFired = new ArrayList<>();
        mergeFailedFired = new ArrayList<>();
        caseFaultedFired = new ArrayList<>();
        preferenceProvider = mock(PreferenceProvider.class);
        Preferences prefs = mock(Preferences.class);
        when(preferenceProvider.resolve(any(SettingsScope.class))).thenReturn(prefs);
        when(prefs.getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL))
            .thenReturn(StringPreference.of("#test-channel"));
        bridge = new CaseLifecycleNotificationBridge(
            capturingEvent(mergeSucceededFired),
            capturingEvent(mergeFailedFired),
            capturingEvent(caseFaultedFired),
            preferenceProvider);
    }

    @Test
    void completedCaseFiresMergeSucceeded() {
        CaseLifecycleEvent event = caseEvent("COMPLETED", "pr-review-v1", "ns1");
        bridge.onCaseLifecycle(event);
        assertEquals(1, mergeSucceededFired.size());
        assertEquals("io.casehub.devtown.merge.succeeded", mergeSucceededFired.getFirst().type());
        assertEquals("#test-channel", mergeSucceededFired.getFirst().targetChannel());
    }

    @Test
    void cancelledCaseFiresMergeFailed() {
        CaseLifecycleEvent event = caseEvent("CANCELLED", "pr-review-v1", "ns1");
        bridge.onCaseLifecycle(event);
        assertEquals(1, mergeFailedFired.size());
        assertEquals("io.casehub.devtown.merge.failed", mergeFailedFired.getFirst().type());
    }

    @Test
    void faultedCaseFiresCaseFaulted() {
        CaseLifecycleEvent event = caseEvent("FAULTED", "pr-review-v1", "ns1");
        bridge.onCaseLifecycle(event);
        assertEquals(1, caseFaultedFired.size());
        assertEquals("io.casehub.devtown.case.faulted", caseFaultedFired.getFirst().type());
    }

    @Test
    void otherStatusesAreIgnored() {
        CaseLifecycleEvent event = caseEvent("SUSPENDED", "pr-review-v1", "ns1");
        bridge.onCaseLifecycle(event);
        assertTrue(mergeSucceededFired.isEmpty());
        assertTrue(mergeFailedFired.isEmpty());
        assertTrue(caseFaultedFired.isEmpty());
    }

    @Test
    void tenancyIdPropagated() {
        CaseLifecycleEvent event = caseEvent("FAULTED", "pr-review-v1", "ns1");
        bridge.onCaseLifecycle(event);
        assertEquals("tenant-1", caseFaultedFired.getFirst().tenancyId());
    }

    private CaseLifecycleEvent caseEvent(String status, String defName, String namespace) {
        return new CaseLifecycleEvent(
            UUID.randomUUID(), "tenant-1", "CompleteCase", "CaseCompleted",
            status, "system", "ORCHESTRATOR", null,
            defName, namespace, null, null, null);
    }

    @SuppressWarnings("unchecked")
    private <T> Event<T> capturingEvent(List<T> captured) {
        Event<T> event = mock(Event.class);
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return null; })
            .when(event).fire(any());
        return event;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CaseLifecycleNotificationBridgeTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `CaseLifecycleNotificationBridge` not found

- [ ] **Step 3: Implement CaseLifecycleNotificationBridge**

Use `ide_create_file`:

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.domain.notification.NotificationPreferenceKeys;
import io.casehub.devtown.review.notification.CaseFaultedEvent;
import io.casehub.devtown.review.notification.MergeFailedEvent;
import io.casehub.devtown.review.notification.MergeSucceededEvent;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class CaseLifecycleNotificationBridge {

    private final Event<MergeSucceededEvent> mergeSucceededEvents;
    private final Event<MergeFailedEvent> mergeFailedEvents;
    private final Event<CaseFaultedEvent> caseFaultedEvents;
    private final PreferenceProvider preferenceProvider;

    @Inject
    public CaseLifecycleNotificationBridge(
            Event<MergeSucceededEvent> mergeSucceededEvents,
            Event<MergeFailedEvent> mergeFailedEvents,
            Event<CaseFaultedEvent> caseFaultedEvents,
            PreferenceProvider preferenceProvider) {
        this.mergeSucceededEvents = mergeSucceededEvents;
        this.mergeFailedEvents = mergeFailedEvents;
        this.caseFaultedEvents = caseFaultedEvents;
        this.preferenceProvider = preferenceProvider;
    }

    @Transactional(Transactional.TxType.NOT_SUPPORTED)
    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (event.caseStatus() == null) return;
        String targetChannel = resolveChannel(event.namespace());
        switch (event.caseStatus()) {
            case "COMPLETED" -> mergeSucceededEvents.fire(new MergeSucceededEvent(
                event.caseId().toString(),
                event.caseDefinitionName(),
                null, null, null,
                event.actorId(),
                targetChannel,
                event.tenancyId()));
            case "CANCELLED" -> mergeFailedEvents.fire(new MergeFailedEvent(
                null, null, null, null,
                event.satisfiedGoalName(),
                event.namespace(),
                null,
                targetChannel,
                event.tenancyId()));
            case "FAULTED" -> caseFaultedEvents.fire(new CaseFaultedEvent(
                event.caseId().toString(),
                event.caseDefinitionName(),
                event.caseStatus(),
                event.contextSnapshot() != null ? event.contextSnapshot().toString() : null,
                targetChannel,
                event.tenancyId()));
            default -> { }
        }
    }

    private String resolveChannel(String namespace) {
        return preferenceProvider
            .resolve(namespace != null ? SettingsScope.of(namespace) : SettingsScope.root())
            .getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL).value();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=CaseLifecycleNotificationBridgeTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (5 tests)

- [ ] **Step 5: Write WatchdogAlertNotificationBridge test**

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.review.notification.StalledCommitmentEvent;
import io.casehub.qhorus.api.watchdog.AlertContext;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class WatchdogAlertNotificationBridgeTest {

    private WatchdogAlertNotificationBridge bridge;
    private List<StalledCommitmentEvent> fired;

    @BeforeEach
    void setUp() {
        fired = new ArrayList<>();
        bridge = new WatchdogAlertNotificationBridge(capturingEvent(fired));
    }

    @Test
    void obligationFanOutFiresStalledEvent() {
        bridge.onWatchdogAlert(alertEvent(WatchdogConditionType.OBLIGATION_FAN_OUT, "channel-1"));
        assertEquals(1, fired.size());
        assertEquals("OBLIGATION_FAN_OUT", fired.getFirst().conditionType());
        assertEquals("channel-1", fired.getFirst().targetName());
    }

    @Test
    void conversationStallFiresStalledEvent() {
        bridge.onWatchdogAlert(alertEvent(WatchdogConditionType.CONVERSATION_STALL, "channel-2"));
        assertEquals(1, fired.size());
        assertEquals("CONVERSATION_STALL", fired.getFirst().conditionType());
    }

    @ParameterizedTest
    @EnumSource(value = WatchdogConditionType.class,
        names = {"OBLIGATION_FAN_OUT", "CONVERSATION_STALL"}, mode = EnumSource.Mode.EXCLUDE)
    void otherConditionTypesAreIgnored(WatchdogConditionType type) {
        bridge.onWatchdogAlert(alertEvent(type, "channel-x"));
        assertTrue(fired.isEmpty(), "Should not fire for " + type);
    }

    private WatchdogAlertEvent alertEvent(WatchdogConditionType type, String target) {
        AlertContext ctx = mock(AlertContext.class);
        when(ctx.conditionType()).thenReturn(type);
        return new WatchdogAlertEvent(UUID.randomUUID(), target, "notif-channel",
            "2 obligations unresponded", Instant.now(), ctx);
    }

    @SuppressWarnings("unchecked")
    private <T> Event<T> capturingEvent(List<T> captured) {
        Event<T> event = mock(Event.class);
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return null; })
            .when(event).fire(any());
        return event;
    }
}
```

- [ ] **Step 6: Implement WatchdogAlertNotificationBridge**

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.review.notification.StalledCommitmentEvent;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import java.util.Set;

@ApplicationScoped
public class WatchdogAlertNotificationBridge {

    private static final Set<WatchdogConditionType> REVIEWER_CONDITIONS = Set.of(
        WatchdogConditionType.OBLIGATION_FAN_OUT,
        WatchdogConditionType.CONVERSATION_STALL);

    private final Event<StalledCommitmentEvent> stalledEvents;

    @Inject
    public WatchdogAlertNotificationBridge(Event<StalledCommitmentEvent> stalledEvents) {
        this.stalledEvents = stalledEvents;
    }

    @Transactional(Transactional.TxType.NOT_SUPPORTED)
    void onWatchdogAlert(@ObservesAsync WatchdogAlertEvent event) {
        if (!REVIEWER_CONDITIONS.contains(event.conditionType())) return;
        stalledEvents.fire(new StalledCommitmentEvent(
            event.conditionType().name(),
            event.targetName(),
            event.summary(),
            event.firedAt().toString(),
            "system",
            event.notificationChannel(),
            null));
    }
}
```

Note: `WatchdogAlertEvent` has no `tenancyId` field — `tenancyId` is null. The stalled commitment notification targets GROUP(devtown-ops) which is system-scoped. If the subscription engine requires non-null tenancyId, this needs a default tenant lookup — address during implementation if tests surface the issue.

- [ ] **Step 7: Run both tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="WatchdogAlertNotificationBridgeTest,CaseLifecycleNotificationBridgeTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 8: Write ReviewAssignmentNotificationBridge test**

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.domain.notification.NotificationPreferenceKeys;
import io.casehub.devtown.domain.sla.StringPreference;
import io.casehub.devtown.review.notification.ReviewAssignedEvent;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.api.WorkItem;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class ReviewAssignmentNotificationBridgeTest {

    private ReviewAssignmentNotificationBridge bridge;
    private List<ReviewAssignedEvent> fired;
    private PreferenceProvider preferenceProvider;

    @BeforeEach
    void setUp() {
        fired = new ArrayList<>();
        preferenceProvider = mock(PreferenceProvider.class);
        Preferences prefs = mock(Preferences.class);
        when(preferenceProvider.resolve(any(SettingsScope.class))).thenReturn(prefs);
        when(prefs.getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL))
            .thenReturn(StringPreference.of("#reviews"));
        bridge = new ReviewAssignmentNotificationBridge(capturingEvent(fired), preferenceProvider);
    }

    @Test
    void createdWorkItemWithPrApprovalTypeFiresEvent() {
        WorkItemLifecycleEvent event = workItemEvent("created",
            List.of("human-decision:pr-approval"), "reviewer-1");
        bridge.onWorkItemCreated(event);
        assertEquals(1, fired.size());
        assertEquals("reviewer-1", fired.getFirst().assigneeId());
        assertEquals("#reviews", fired.getFirst().targetChannel());
    }

    @Test
    void createdWorkItemWithoutPrApprovalTypeIsIgnored() {
        WorkItemLifecycleEvent event = workItemEvent("created",
            List.of("human-oversight:routing-review"), "reviewer-1");
        bridge.onWorkItemCreated(event);
        assertTrue(fired.isEmpty());
    }

    @Test
    void nonCreatedEventTypeIsIgnored() {
        WorkItemLifecycleEvent event = workItemEvent("completed",
            List.of("human-decision:pr-approval"), "reviewer-1");
        bridge.onWorkItemCreated(event);
        assertTrue(fired.isEmpty());
    }

    private WorkItemLifecycleEvent workItemEvent(String eventName, List<String> types, String assigneeId) {
        WorkItem wi = mock(WorkItem.class);
        wi.id = UUID.randomUUID();
        wi.tenancyId = "tenant-1";
        wi.assigneeId = assigneeId;
        wi.callerRef = "devtown:pr-review";
        wi.types = types.stream().map(t -> { var tt = new io.casehub.work.api.WorkItemType(); tt.path = t; return tt; }).toList();
        wi.scope = io.casehub.platform.api.preferences.Path.of("my-repo");
        return WorkItemLifecycleEvent.of(eventName, wi, "system", null);
    }

    @SuppressWarnings("unchecked")
    private <T> Event<T> capturingEvent(List<T> captured) {
        Event<T> event = mock(Event.class);
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return null; })
            .when(event).fire(any());
        return event;
    }
}
```

- [ ] **Step 9: Implement ReviewAssignmentNotificationBridge**

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.domain.notification.NotificationPreferenceKeys;
import io.casehub.devtown.review.notification.ReviewAssignedEvent;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.SettingsScope;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.enterprise.event.TransactionPhase;

@ApplicationScoped
public class ReviewAssignmentNotificationBridge {

    private final Event<ReviewAssignedEvent> reviewAssignedEvents;
    private final PreferenceProvider preferenceProvider;

    @Inject
    public ReviewAssignmentNotificationBridge(
            Event<ReviewAssignedEvent> reviewAssignedEvents,
            PreferenceProvider preferenceProvider) {
        this.reviewAssignedEvents = reviewAssignedEvents;
        this.preferenceProvider = preferenceProvider;
    }

    @Transactional(Transactional.TxType.NOT_SUPPORTED)
    void onWorkItemCreated(
            @Observes(during = TransactionPhase.AFTER_SUCCESS)
            WorkItemLifecycleEvent event) {
        if (!event.type().endsWith(".created")) return;
        if (event.types() == null || !event.types().contains("human-decision:pr-approval")) return;
        String targetChannel = preferenceProvider
            .resolve(event.workItem() != null && event.workItem().scope != null
                ? SettingsScope.of(event.workItem().scope)
                : SettingsScope.root())
            .getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL).value();
        reviewAssignedEvents.fire(new ReviewAssignedEvent(
            event.workItemId().toString(),
            event.assigneeId(),
            targetChannel,
            event.tenancyId()));
    }
}
```

- [ ] **Step 10: Write SlaBreachNotificationBridge test**

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.domain.notification.NotificationPreferenceKeys;
import io.casehub.devtown.domain.sla.StringPreference;
import io.casehub.devtown.review.notification.SlaEscalatedEvent;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.BreachedTask;
import io.casehub.work.api.BreachType;
import io.casehub.work.api.SlaBreachContext;
import io.casehub.work.runtime.event.SlaBreachEvent;
import io.casehub.platform.api.preferences.Path;
import jakarta.enterprise.event.Event;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class SlaBreachNotificationBridgeTest {

    private SlaBreachNotificationBridge bridge;
    private List<SlaEscalatedEvent> fired;

    @BeforeEach
    void setUp() {
        fired = new ArrayList<>();
        bridge = new SlaBreachNotificationBridge(capturingEvent(fired));
    }

    @Test
    void escalateToDecisionFiresEvent() {
        Preferences prefs = mock(Preferences.class);
        when(prefs.getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL))
            .thenReturn(StringPreference.of("#escalations"));
        BreachedTask task = new BreachedTask(UUID.randomUUID(), "devtown:pr-review", "Review PR #42", Set.of("pr-leads"));
        SlaBreachContext ctx = new SlaBreachContext(BreachType.COMPLETION, task, Path.of("my-repo"), prefs);
        BreachDecision.EscalateTo decision = BreachDecision.EscalateTo.to("pr-leads", "managers");
        SlaBreachEvent event = new SlaBreachEvent(ctx, decision, "tenant-1");

        bridge.onSlaBreach(event);

        assertEquals(1, fired.size());
        SlaEscalatedEvent e = fired.getFirst();
        assertEquals("io.casehub.devtown.sla.escalated", e.type());
        assertEquals(task.taskId().toString(), e.taskId());
        assertEquals("Review PR #42", e.taskTitle());
        assertEquals("COMPLETION", e.breachType());
        assertTrue(e.escalationGroups().contains("pr-leads"));
        assertEquals("#escalations", e.targetChannel());
        assertEquals("tenant-1", e.tenancyId());
    }

    @Test
    void nonEscalateDecisionIsIgnored() {
        Preferences prefs = mock(Preferences.class);
        BreachedTask task = new BreachedTask(UUID.randomUUID(), "devtown:pr-review", "Review PR #42", Set.of());
        SlaBreachContext ctx = new SlaBreachContext(BreachType.COMPLETION, task, Path.of("my-repo"), prefs);
        BreachDecision.Fail decision = new BreachDecision.Fail("timeout");
        SlaBreachEvent event = new SlaBreachEvent(ctx, decision, "tenant-1");

        bridge.onSlaBreach(event);
        assertTrue(fired.isEmpty());
    }

    @SuppressWarnings("unchecked")
    private <T> Event<T> capturingEvent(List<T> captured) {
        Event<T> event = mock(Event.class);
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return null; })
            .when(event).fire(any());
        return event;
    }
}
```

- [ ] **Step 11: Implement SlaBreachNotificationBridge**

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.domain.notification.NotificationPreferenceKeys;
import io.casehub.devtown.review.notification.SlaEscalatedEvent;
import io.casehub.work.api.BreachDecision;
import io.casehub.work.api.BreachedTask;
import io.casehub.work.runtime.event.SlaBreachEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.TransactionPhase;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

@ApplicationScoped
public class SlaBreachNotificationBridge {

    private final Event<SlaEscalatedEvent> slaEscalatedEvents;

    @Inject
    public SlaBreachNotificationBridge(Event<SlaEscalatedEvent> slaEscalatedEvents) {
        this.slaEscalatedEvents = slaEscalatedEvents;
    }

    @Transactional(Transactional.TxType.NOT_SUPPORTED)
    void onSlaBreach(
            @Observes(during = TransactionPhase.AFTER_SUCCESS)
            SlaBreachEvent event) {
        if (!(event.decision() instanceof BreachDecision.EscalateTo escalation)) return;
        BreachedTask task = event.context().task();
        String targetChannel = event.context().preferences()
            .getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL).value();
        slaEscalatedEvents.fire(new SlaEscalatedEvent(
            task.taskId().toString(),
            task.title(),
            task.callerRef(),
            event.context().breachType().name(),
            String.join(", ", escalation.groups()),
            event.context().scope().toString(),
            targetChannel,
            event.tenancyId()));
    }
}
```

- [ ] **Step 12: Run all bridge observer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="CaseLifecycleNotificationBridgeTest,WatchdogAlertNotificationBridgeTest,ReviewAssignmentNotificationBridgeTest,SlaBreachNotificationBridgeTest" -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/src/main/java/io/casehub/devtown/app/notification/ app/src/test/java/io/casehub/devtown/app/notification/
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#16): notification bridge observers for 6 scenarios"
```

---

### Task 3: Subscription registrar, dependencies, and integration tests

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/notification/DevtownSubscriptionRegistrar.java`
- Create: `app/src/test/java/io/casehub/devtown/app/notification/DevtownSubscriptionRegistrarTest.java`
- Create: `app/src/test/java/io/casehub/devtown/app/notification/NotificationBridgeIntegrationTest.java`
- Modify: `app/pom.xml` — add `subscriptions-inmem` test dependency
- Modify: `app/src/test/resources/application.properties` — add connector exclusions if not present

**Interfaces:**
- Consumes: `SubscriptionStore.store(SubscriptionInput)`, `SubscriptionStore.find(SubscriptionQuery)` (from casehub-platform-api)
- Consumes: All 6 `SubscribableEvent` types (from Task 1), all 4 bridge observers (from Task 2)

- [ ] **Step 1: Add subscriptions-inmem test dependency to app/pom.xml**

Add to `app/pom.xml` `<dependencies>` section:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-subscriptions-inmem</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Write DevtownSubscriptionRegistrar test**

```java
package io.casehub.devtown.app.notification;

import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.subscription.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.*;

class DevtownSubscriptionRegistrarTest {

    private CapturingSubscriptionStore store;
    private DevtownSubscriptionRegistrar registrar;

    @BeforeEach
    void setUp() {
        store = new CapturingSubscriptionStore();
        registrar = new DevtownSubscriptionRegistrar(store);
    }

    @Test
    void registersAllSixSubscriptions() {
        registrar.register("default-tenant");
        assertEquals(6, store.stored.size());
    }

    @Test
    void mergeFailedSubscriptionHasCorrectShape() {
        registrar.register("default-tenant");
        SubscriptionInput mergeFailed = store.stored.stream()
            .filter(s -> s.name().equals("devtown-merge-failed"))
            .findFirst().orElseThrow();
        assertEquals("io.casehub.devtown.merge.failed", mergeFailed.eventType());
        assertEquals(NotificationSeverity.URGENT, mergeFailed.template().severity());
        assertEquals("devtown.merge", mergeFailed.template().entityType());
        assertEquals(SubscriptionScope.SYSTEM, mergeFailed.scope());
        assertTrue(mergeFailed.targets().stream()
            .anyMatch(t -> t.type() == TargetType.GROUP && t.id().equals("devtown-ops")));
        assertTrue(mergeFailed.targets().stream()
            .anyMatch(t -> t.type() == TargetType.EVENT_FIELD && t.id().equals("authorId")));
    }

    @Test
    void idempotentRegistration() {
        registrar.register("default-tenant");
        registrar.register("default-tenant");
        assertEquals(6, store.stored.size(), "Second registration should be no-op");
    }

    @Test
    void allTemplatesHaveEightFields() {
        registrar.register("default-tenant");
        for (SubscriptionInput sub : store.stored) {
            NotificationTemplate t = sub.template();
            assertNotNull(t.titlePattern(), sub.name() + " titlePattern");
            assertNotNull(t.severity(), sub.name() + " severity");
            assertNotNull(t.category(), sub.name() + " category");
            assertNotNull(t.entityType(), sub.name() + " entityType");
            assertNotNull(t.entityIdField(), sub.name() + " entityIdField");
            assertNotNull(t.actorIdField(), sub.name() + " actorIdField");
        }
    }

    static class CapturingSubscriptionStore implements SubscriptionStore {
        final List<SubscriptionInput> stored = new CopyOnWriteArrayList<>();

        @Override
        public Subscription store(SubscriptionInput input) {
            stored.add(input);
            return new Subscription(
                "sub-" + stored.size(), input.ownerId(), input.tenancyId(), input.name(),
                input.eventType(), input.filters(), input.targets(), input.includeActor(),
                input.template(), input.enabled(), input.scope(),
                Instant.now(), Instant.now());
        }

        @Override
        public SubscriptionPage find(SubscriptionQuery query) {
            List<Subscription> matches = stored.stream()
                .map(i -> new Subscription(
                    "sub-x", i.ownerId(), i.tenancyId(), i.name(), i.eventType(),
                    i.filters(), i.targets(), i.includeActor(), i.template(),
                    i.enabled(), i.scope(), Instant.now(), Instant.now()))
                .toList();
            return new SubscriptionPage(matches, null, matches.size());
        }

        @Override public Optional<Subscription> findById(String id, String ownerId, String tenancyId) { return Optional.empty(); }
        @Override public Optional<Subscription> update(String id, String ownerId, String tenancyId, SubscriptionUpdate update) { return Optional.empty(); }
        @Override public boolean delete(String id, String ownerId, String tenancyId) { return false; }
        @Override public Stream<Subscription> findAllEnabled() { return stored.stream().map(i -> new Subscription(
            "sub-x", i.ownerId(), i.tenancyId(), i.name(), i.eventType(),
            i.filters(), i.targets(), i.includeActor(), i.template(),
            i.enabled(), i.scope(), Instant.now(), Instant.now())); }
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownSubscriptionRegistrarTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: FAIL — `DevtownSubscriptionRegistrar` not found

- [ ] **Step 4: Implement DevtownSubscriptionRegistrar**

```java
package io.casehub.devtown.app.notification;

import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.subscription.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.event.Observes;

import java.util.List;

@ApplicationScoped
public class DevtownSubscriptionRegistrar {

    private final SubscriptionStore subscriptionStore;

    @Inject
    public DevtownSubscriptionRegistrar(SubscriptionStore subscriptionStore) {
        this.subscriptionStore = subscriptionStore;
    }

    void onStartup(@Observes StartupEvent event) {
        register("default");
    }

    void register(String tenancyId) {
        SubscriptionPage existing = subscriptionStore.find(new SubscriptionQuery(
            null, tenancyId, SubscriptionScope.SYSTEM, null, null, 100));

        registerIfAbsent(existing, tenancyId, new SubscriptionInput(
            "system:devtown", tenancyId, "devtown-review-assigned",
            "io.casehub.devtown.review.assigned", List.of(),
            List.of(new NotificationTarget(TargetType.EVENT_FIELD, "assigneeId")),
            false,
            new NotificationTemplate("PR #{prNumber} assigned for {capability} review",
                "{prTitle} by {authorName} — deadline {deadline}",
                NotificationSeverity.INFO, "devtown.review.assigned",
                "/api/workitems/{workItemId}", "devtown.review", "prNumber", "assigneeId"),
            true, SubscriptionScope.SYSTEM));

        registerIfAbsent(existing, tenancyId, new SubscriptionInput(
            "system:devtown", tenancyId, "devtown-merge-succeeded",
            "io.casehub.devtown.merge.succeeded", List.of(),
            List.of(new NotificationTarget(TargetType.ENTITY_WATCHERS, "")),
            false,
            new NotificationTemplate("Batch merged: {prCount} PRs", "{prList}",
                NotificationSeverity.INFO, "devtown.merge.succeeded",
                "/api/reviews/{prNumber}", "devtown.merge", "prNumber", "actorId"),
            true, SubscriptionScope.SYSTEM));

        registerIfAbsent(existing, tenancyId, new SubscriptionInput(
            "system:devtown", tenancyId, "devtown-merge-failed",
            "io.casehub.devtown.merge.failed", List.of(),
            List.of(
                new NotificationTarget(TargetType.GROUP, "devtown-ops"),
                new NotificationTarget(TargetType.EVENT_FIELD, "authorId")),
            false,
            new NotificationTemplate("Merge rejected: {prTitle}",
                "CI failure: {failureReason} — author: {authorName}",
                NotificationSeverity.URGENT, "devtown.merge.failed",
                "/api/reviews/{prNumber}", "devtown.merge", "prNumber", "authorId"),
            true, SubscriptionScope.SYSTEM));

        registerIfAbsent(existing, tenancyId, new SubscriptionInput(
            "system:devtown", tenancyId, "devtown-commitment-stalled",
            "io.casehub.devtown.commitment.stalled", List.of(),
            List.of(new NotificationTarget(TargetType.GROUP, "devtown-ops")),
            false,
            new NotificationTemplate("Stalled: {conditionType} on {targetName}",
                "{summary} — fired at {firedAt}",
                NotificationSeverity.WARNING, "devtown.commitment.stalled",
                null, "devtown.watchdog", "targetName", "actorId"),
            true, SubscriptionScope.SYSTEM));

        registerIfAbsent(existing, tenancyId, new SubscriptionInput(
            "system:devtown", tenancyId, "devtown-case-faulted",
            "io.casehub.devtown.case.faulted", List.of(),
            List.of(new NotificationTarget(TargetType.GROUP, "devtown-ops")),
            false,
            new NotificationTemplate("Case faulted: {caseDefinitionName}",
                "Case {caseId} in state {caseStatus}",
                NotificationSeverity.URGENT, "devtown.case.faulted",
                "/api/compliance/code-review/{caseId}", "devtown.case", "caseId", "actorId"),
            true, SubscriptionScope.SYSTEM));

        registerIfAbsent(existing, tenancyId, new SubscriptionInput(
            "system:devtown", tenancyId, "devtown-sla-escalated",
            "io.casehub.devtown.sla.escalated", List.of(),
            List.of(new NotificationTarget(TargetType.GROUP, "devtown-ops")),
            false,
            new NotificationTemplate("SLA breach: {taskTitle}",
                "{breachType} — escalated to {escalationGroups}",
                NotificationSeverity.URGENT, "devtown.sla.escalated",
                "/api/workitems/{taskId}", "devtown.sla", "taskId", "callerRef"),
            true, SubscriptionScope.SYSTEM));
    }

    private void registerIfAbsent(SubscriptionPage existing, String tenancyId, SubscriptionInput input) {
        boolean alreadyExists = existing.subscriptions().stream()
            .anyMatch(s -> s.name().equals(input.name()));
        if (!alreadyExists) {
            subscriptionStore.store(input);
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownSubscriptionRegistrarTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (4 tests)

- [ ] **Step 6: Write NotificationBridgeIntegrationTest**

This `@QuarkusTest` verifies CDI wiring — foundation event → bridge observer → `SubscribableEvent` fired. Uses a test observer to capture fired events.

```java
package io.casehub.devtown.app.notification;

import io.casehub.devtown.review.notification.CaseFaultedEvent;
import io.casehub.devtown.review.notification.MergeSucceededEvent;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.TimeUnit;

import static org.awaitility.Awaitility.await;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class NotificationBridgeIntegrationTest {

    @Inject Event<CaseLifecycleEvent> caseEvents;
    @Inject TestEventCapture capture;

    @BeforeEach
    void setUp() {
        capture.clear();
    }

    @Test
    void completedCaseEventWiresThroughBridge() {
        CaseLifecycleEvent event = new CaseLifecycleEvent(
            UUID.randomUUID(), "default", "CompleteCase", "CaseCompleted",
            "COMPLETED", "system", "ORCHESTRATOR", null,
            "pr-review-v1", "my-repo", null, null, null);
        caseEvents.fireAsync(event);

        await().atMost(5, TimeUnit.SECONDS).until(() -> !capture.mergeSucceeded.isEmpty());
        assertEquals("io.casehub.devtown.merge.succeeded", capture.mergeSucceeded.getFirst().type());
    }

    @Test
    void faultedCaseEventWiresThroughBridge() {
        CaseLifecycleEvent event = new CaseLifecycleEvent(
            UUID.randomUUID(), "default", "FaultCase", "CaseFaulted",
            "FAULTED", null, null, null,
            "pr-review-v1", "my-repo", null, null, null);
        caseEvents.fireAsync(event);

        await().atMost(5, TimeUnit.SECONDS).until(() -> !capture.caseFaulted.isEmpty());
        assertEquals("io.casehub.devtown.case.faulted", capture.caseFaulted.getFirst().type());
    }

    @ApplicationScoped
    public static class TestEventCapture {
        final CopyOnWriteArrayList<MergeSucceededEvent> mergeSucceeded = new CopyOnWriteArrayList<>();
        final CopyOnWriteArrayList<CaseFaultedEvent> caseFaulted = new CopyOnWriteArrayList<>();

        void onMergeSucceeded(@ObservesAsync MergeSucceededEvent e) { mergeSucceeded.add(e); }
        void onCaseFaulted(@ObservesAsync CaseFaultedEvent e) { caseFaulted.add(e); }

        void clear() { mergeSucceeded.clear(); caseFaulted.clear(); }
    }
}
```

- [ ] **Step 7: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=NotificationBridgeIntegrationTest -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: PASS (2 tests). If CDI validation fails for Twilio/WhatsApp, add to `app/src/test/resources/application.properties`:
```properties
quarkus.arc.exclude-types=io.casehub.connectors.twilio.TwilioSmsConnector,io.casehub.connectors.whatsapp.WhatsAppConnector
```

- [ ] **Step 8: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml`
Expected: BUILD SUCCESS — all existing tests still pass, no regressions

- [ ] **Step 9: Verify with ide_diagnostics**

Run `ide_diagnostics` on all new files to catch any remaining issues.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add app/pom.xml app/src/main/java/io/casehub/devtown/app/notification/DevtownSubscriptionRegistrar.java app/src/test/java/io/casehub/devtown/app/notification/DevtownSubscriptionRegistrarTest.java app/src/test/java/io/casehub/devtown/app/notification/NotificationBridgeIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(#16): subscription registrar and integration tests"
```
