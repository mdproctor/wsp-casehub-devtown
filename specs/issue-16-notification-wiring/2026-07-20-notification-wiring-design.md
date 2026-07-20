# Epic 9: Notification Wiring — casehub-connectors integration

**Issue:** devtown#16
**Branch:** issue-16-notification-wiring
**Date:** 2026-07-20
**Status:** Approved

---

## Platform Prerequisites

The platform notification system (documented in `casehub-parent/docs/platform/notifications.md`) is partially implemented. devtown's notification wiring builds on the data model layer that exists today; the matching and dispatch pipeline activates when the platform ships it.

**Exists in casehub-platform-api (data model):**
- `SubscribableEvent` interface (`type()`, `tenancyId()`)
- `SubscriptionStore` / `ReactiveSubscriptionStore` (CRUD: `store()`, `findById()`, `find()`, `update()`, `delete()`, `findAllEnabled()`)
- `SubscriptionInput` (10 fields: `ownerId`, `tenancyId`, `name`, `eventType`, `filters`, `targets`, `includeActor`, `template`, `enabled`, `scope`)
- `SubscriptionScope` (`USER`, `SYSTEM`)
- `NotificationTarget` + `TargetType` (`USER`, `GROUP`, `EVENT_FIELD`, `ENTITY_WATCHERS`)
- `NotificationTemplate` (8 fields: `titlePattern`, `bodyPattern`, `severity`, `category`, `actionUrlPattern`, `entityType`, `entityIdField`, `actorIdField`)
- `NotificationSeverity` (`INFO`, `WARNING`, `URGENT`)
- `DeliveryChannelRegistry` interface + `NoOpDeliveryChannelRegistry` fallback
- `NotificationDeliverer` interface

**Exists in casehub-platform (runtime):**
- `NoOpSubscriptionStore` / `NoOpReactiveSubscriptionStore` (fallbacks)
- In-memory subscription store (`subscriptions-inmem` module)

**Does not yet exist (platform prerequisites for full pipeline):**
- `SubscriptionEngine` — pattern matching against active subscriptions, fires `SubscriptionMatched`
- `NotificationDispatcher` — target resolution → suppression → template → channel routing
- `TargetResolver`, `SuppressionEvaluator`, `TemplateResolver`, `ChannelRouter`
- `DigestBuffer` / `DigestFlushScheduler`
- `InAppNotificationDeliverer`

**External dependency:** connectors#86 (notification delivery bridge: `NotificationDeliverer` → `Connector.send()` auto-discovery) — in progress in casehub-connectors. devtown develops and tests against in-memory stores; external delivery activates when connectors#86 ships and the bridge module is on the classpath.

### Foundation gates

- **parent#5** (Connector SPI consolidation) ✅ CLOSED. Resolution: `NotificationChannel` implementations in `casehub-work-notifications` delegate to `Connector`. The platform notification system uses `NotificationDeliverer` SPI; connectors#86 bridges `NotificationDeliverer` → `Connector.send()`. This spec aligns with the consolidation — devtown owns event POJOs and subscriptions; delivery routes through the consolidated `Connector` path.
- **qhorus#200** (WatchdogAlertEvent + ConnectorAlertBridge) ✅ CLOSED. `WatchdogAlertEvent` is a CDI event fired via `Event.fireAsync()`. This spec's stalled commitment bridge observes that event directly.

### Build progression

devtown builds and tests everything in this spec using in-memory platform stores today. The full notification pipeline activates in two stages:
1. Platform ships `SubscriptionEngine` + `NotificationDispatcher` → matching and dispatch activate
2. connectors#86 ships `ConnectorNotificationDeliverer` → external delivery activates (Slack, Teams, email)

Bridge observers fire `SubscribableEvent` POJOs as CDI events. When `SubscriptionEngine` is deployed, it observes these events and drives the downstream pipeline. Until then, integration tests verify: foundation event → correct `SubscribableEvent` fired with correct fields.

---

## Architecture

```
devtown CDI observers (bridge foundation events)
  ↓ fire SubscribableEvent via CDI Event.fire()
Platform SubscriptionEngine (pattern matching) [not yet implemented]
  ↓ SubscriptionMatched (async CDI event)
Platform NotificationDispatcher (pipeline orchestration) [not yet implemented]
  ↓ NotificationInput
Platform DeliveryChannelRegistry
  ├── InAppNotificationDeliverer (inbox + SSE) [not yet implemented]
  └── ConnectorNotificationDeliverer (connectors#86 — in progress)
        ↓ ConnectorMessage
     casehub-connectors Connector.send()
```

devtown owns: event POJOs implementing `SubscribableEvent`, subscription definitions via `SubscriptionStore`, bridge observers translating foundation events to `SubscribableEvent` CDI events, and per-repo connector target configuration via `PreferenceProvider`.

---

## Notification Scenarios

Six notification scenarios, each mapping to a `SubscribableEvent` POJO, a `Subscription` definition, and a CDI observer bridging the originating foundation event.

### Event type mapping

| # | Scenario | Event type | Source event | Observer annotation | Severity | Targets | Entity type |
|---|----------|-----------|-------------|---------------------|----------|---------|-------------|
| 1 | Review assignment | `io.casehub.devtown.review.assigned` | `WorkItemLifecycleEvent(CREATED)` | `@Observes(during = AFTER_SUCCESS)` | INFO | EVENT_FIELD (assigneeId) | `devtown.review` |
| 2 | Merge success | `io.casehub.devtown.merge.succeeded` | `CaseLifecycleEvent` (caseStatus=`COMPLETED`) | `@ObservesAsync` | INFO | ENTITY_WATCHERS | `devtown.merge` |
| 3 | Merge failure | `io.casehub.devtown.merge.failed` | `CaseLifecycleEvent` (caseStatus=`CANCELLED`) | `@ObservesAsync` | URGENT | EVENT_FIELD (authorId) + GROUP (devtown-ops) | `devtown.merge` |
| 4 | Stalled commitment | `io.casehub.devtown.commitment.stalled` | `WatchdogAlertEvent` | `@ObservesAsync` | WARNING | GROUP (devtown-ops) | `devtown.watchdog` |
| 5 | Case fault | `io.casehub.devtown.case.faulted` | `CaseLifecycleEvent` (caseStatus=`FAULTED`) | `@ObservesAsync` | URGENT | GROUP (devtown-ops) | `devtown.case` |
| 6 | SLA breach escalation | `io.casehub.devtown.sla.escalated` | `SlaBreachEvent` (decision=`EscalateTo`) | `@Observes(during = AFTER_SUCCESS)` | URGENT | GROUP (devtown-ops) | `devtown.sla` |

**Observer annotations** depend on source event firing mechanism:
- `CaseLifecycleEvent` → `@ObservesAsync` — fired via `Event.fireAsync()` from Vert.x handlers (Javadoc-documented)
- `WatchdogAlertEvent` → `@ObservesAsync` — fired via `Event.fireAsync()` (qhorus#200 design)
- `WorkItemLifecycleEvent` → `@Observes(during = AFTER_SUCCESS)` — fired synchronously
- `SlaBreachEvent` → `@Observes(during = AFTER_SUCCESS)` — fired synchronously during breach handling

GE-20260427-893862 (`@Observes(during = AFTER_SUCCESS)` + `@Transactional(NOT_SUPPORTED)`) applies to **synchronous** source events only. Async source events use `@ObservesAsync` + `@Transactional(NOT_SUPPORTED)`.

**Scenario 1 — review assignment scope:** `WorkItemLifecycleEvent(CREATED)` covers human review assignments (WorkItem with `types` containing `human-decision:pr-approval`). Agent review dispatch notification requires a commitment lifecycle CDI event that does not yet exist — tracked as devtown#166.

**Scenario 4 — watchdog condition filter:** The bridge filters `WatchdogAlertEvent.conditionType()` for `OBLIGATION_FAN_OUT` (unresponded obligations past deadline) and `CONVERSATION_STALL` (stalled correlations on a channel). Other condition types (`BARRIER_STUCK`, `AGENT_STALE`, `QUEUE_DEPTH`, etc.) are excluded — they are infrastructure concerns, not reviewer-facing notifications. The existing Qhorus channel dispatch (`messageService.dispatch()`) notifies agents; this notification path notifies human operators. Dual delivery is intentional — different audiences.

**Scenario 6 — SLA breach field mapping:** `SlaBreachEvent(SlaBreachContext context, BreachDecision decision, String tenancyId)`. The bridge extracts fields from verified APIs:
- `taskId` ← `context.task().taskId().toString()` (`BreachedTask.taskId()`: UUID)
- `taskTitle` ← `context.task().title()` (`BreachedTask.title()`: String)
- `callerRef` ← `context.task().callerRef()` (`BreachedTask.callerRef()`: String — the process that created the work item; used as actorIdField since SLA breach is system-initiated, no human actor)
- `breachType` ← `context.breachType().name()` (`SlaBreachContext.breachType()`: BreachType enum)
- `escalationGroups` ← `String.join(", ", ((EscalateTo) decision).groups())` (`EscalateTo.groups()`: Set<String> — joined for template substitution)
- `scope` ← `context.scope()` (`SlaBreachContext.scope()`: Path — the repo scope, used for per-repo preference resolution)

`BreachedTask` has no `assigneeId` — the notification targets GROUP(devtown-ops) only. `EscalateTo.groups()` identifies the escalation destination and appears in the template body. This is distinct from `AgentRoutingEscalationEvent` (engine routing failure due to trust qualification gaps) — a separate concern not included in this epic.

### Templates

| Scenario | Title pattern | Body pattern | Category | Action URL pattern | Entity type | Entity ID field | Actor ID field |
|----------|--------------|-------------|----------|-------------------|-------------|----------------|---------------|
| Review assigned | `PR #{prNumber} assigned for {capability} review` | `{prTitle} by {authorName} — deadline {deadline}` | `devtown.review.assigned` | `/api/workitems/{workItemId}` | `devtown.review` | `prNumber` | `assigneeId` |
| Merge succeeded | `Batch merged: {prCount} PRs` | `{prList}` | `devtown.merge.succeeded` | `/api/reviews/{prNumber}` | `devtown.merge` | `prNumber` | `actorId` |
| Merge failed | `Merge rejected: {prTitle}` | `CI failure: {failureReason} — author: {authorName}` | `devtown.merge.failed` | `/api/reviews/{prNumber}` | `devtown.merge` | `prNumber` | `authorId` |
| Stalled commitment | `Stalled: {conditionType} on {targetName}` | `{summary} — fired at {firedAt}` | `devtown.commitment.stalled` | null | `devtown.watchdog` | `targetName` | `actorId` |
| Case faulted | `Case faulted: {caseDefinitionName}` | `Case {caseId} in state {caseStatus}` | `devtown.case.faulted` | `/api/compliance/code-review/{caseId}` | `devtown.case` | `caseId` | `actorId` |
| SLA escalated | `SLA breach: {taskTitle}` | `{breachType} — escalated to {escalationGroups}` | `devtown.sla.escalated` | `/api/workitems/{taskId}` | `devtown.sla` | `taskId` | `callerRef` |

All templates use platform `NotificationTemplate` with all 8 required fields (`titlePattern`, `bodyPattern`, `severity`, `category`, `actionUrlPattern`, `entityType`, `entityIdField`, `actorIdField`). `severity` is embedded in the template (not a separate registration argument). `category` is `@NonNull`. `actionUrlPattern` is nullable (null for stalled commitment — no single resource to link to).

---

## Module Placement

Event POJO records live in `review/` (pure Java records implementing a platform API interface — integration contract, Tier 2). Bridge observers, subscription registrar, and per-repo configuration live in `app/` (CDI wiring, Tier 3). This follows the three-tier module protocol and is consistent with existing observers (`MergeDecisionObserver` in `app/src/main/java/io/casehub/devtown/app/ledger/`).

| Component | Location | Count |
|-----------|----------|-------|
| `SubscribableEvent` records | `review/src/main/java/io/casehub/devtown/review/notification/` | 6 records |
| Bridge observers | `app/src/main/java/io/casehub/devtown/app/notification/` | 4–5 classes |
| `DevtownSubscriptionRegistrar` | `app/src/main/java/io/casehub/devtown/app/notification/` | 1 class |
| `NotificationPreferenceKeys` | `domain/src/main/java/io/casehub/devtown/domain/` | 1 class (per-repo config keys) |
| Domain module changes | None | `devtown-domain` stays pure Java (except preference key constants) |

### Event POJO pattern

Each scenario is a record implementing `SubscribableEvent`:

```java
public record MergeFailedEvent(
    String prNumber, String prTitle, String authorId, String authorName,
    String failureReason, String repoId, String prUrl,
    String targetChannel,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.merge.failed"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

All `SubscribableEvent` POJOs include a `targetChannel` field — the per-repo resolved delivery channel (see §Per-Repo Connector Configuration). Bridge observers resolve this before firing the event.

### Bridge observer pattern

Each observer translates a foundation CDI event into a `SubscribableEvent` and fires it as a CDI event. The platform `SubscriptionEngine` (when deployed) observes `SubscribableEvent` subtypes via CDI and drives the downstream pipeline.

**Async source (CaseLifecycleEvent — fired via `Event.fireAsync()`):**

```java
@ApplicationScoped
public class CaseFaultNotificationBridge {

    @Inject Event<CaseFaultedEvent> caseFaultedEvents;
    @Inject PreferenceProvider preferenceProvider;

    @Transactional(NOT_SUPPORTED)
    void onCaseFault(@ObservesAsync CaseLifecycleEvent event) {
        if (!"FAULTED".equals(event.caseStatus())) return;
        String targetChannel = preferenceProvider
            .resolve(SettingsScope.of(event.namespace()))
            .getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL).value();
        caseFaultedEvents.fire(new CaseFaultedEvent(
            event.caseId().toString(),
            event.caseDefinitionName(),
            event.caseStatus(),
            event.contextSnapshot() != null ? event.contextSnapshot().toString() : null,
            targetChannel,
            event.tenancyId()));
    }
}
```

**Sync source (WorkItemLifecycleEvent — fired synchronously):**

```java
@ApplicationScoped
public class ReviewAssignmentNotificationBridge {

    @Inject Event<ReviewAssignedEvent> reviewAssignedEvents;
    @Inject PreferenceProvider preferenceProvider;

    @Transactional(NOT_SUPPORTED)
    void onWorkItemCreated(@Observes(during = AFTER_SUCCESS) WorkItemLifecycleEvent event) {
        if (event.eventType() != WorkEventType.CREATED) return;
        if (!event.types().contains("human-decision:pr-approval")) return;
        String targetChannel = preferenceProvider
            .resolve(SettingsScope.of(event.workItem().scope))
            .getOrDefault(NotificationPreferenceKeys.SLACK_CHANNEL).value();
        reviewAssignedEvents.fire(new ReviewAssignedEvent(
            event.workItemId().toString(),
            event.assigneeId(),
            targetChannel,
            event.tenancyId()));
    }
}
```

**Sync source (SlaBreachEvent — fired synchronously during breach handling):**

```java
@ApplicationScoped
public class SlaBreachNotificationBridge {

    @Inject Event<SlaEscalatedEvent> slaEscalatedEvents;

    @Transactional(NOT_SUPPORTED)
    void onSlaBreach(@Observes(during = AFTER_SUCCESS) SlaBreachEvent event) {
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

### Subscription registration

Programmatic at `@Startup` via `SubscriptionStore`. Subscriptions are code-versioned, not Flyway-seeded. Uses idempotent find-then-store against the actual `SubscriptionStore` API:

```java
@ApplicationScoped
public class DevtownSubscriptionRegistrar {

    @Inject SubscriptionStore subscriptionStore;
    @Inject TenancyProvider tenancyProvider;

    void onStartup(@Observes StartupEvent event) {
        String tenancyId = tenancyProvider.defaultTenancyId();

        registerIfAbsent(tenancyId, new SubscriptionInput(
            "system:devtown",
            tenancyId,
            "devtown-merge-failed",
            "io.casehub.devtown.merge.failed",
            List.of(),
            List.of(
                new NotificationTarget(TargetType.GROUP, "devtown-ops"),
                new NotificationTarget(TargetType.EVENT_FIELD, "authorId")),
            false,
            new NotificationTemplate(
                "Merge rejected: {prTitle}",
                "CI failure: {failureReason} — author: {authorName}",
                NotificationSeverity.URGENT,
                "devtown.merge.failed",
                "/api/reviews/{prNumber}",
                "devtown.merge",
                "prNumber",
                "authorId"),
            true,
            SubscriptionScope.SYSTEM));
        // ... 5 more subscriptions
    }

    private void registerIfAbsent(String tenancyId, SubscriptionInput input) {
        SubscriptionPage page = subscriptionStore.find(new SubscriptionQuery(
            input.ownerId(), tenancyId, SubscriptionScope.SYSTEM,
            null, null, 100));
        boolean exists = page.subscriptions().stream()
            .anyMatch(s -> s.name().equals(input.name()));
        if (!exists) {
            subscriptionStore.store(input);
        }
    }
}
```

---

## Per-Repo Connector Configuration

Issue #16 requires: "connector targets configurable per-repo (which Slack channel gets which repo's notifications)."

devtown uses `PreferenceProvider` (parent#26 ✅ CLOSED) scoped by repository. Per-repo channel resolution happens in devtown's bridge observers — not in the generic connector bridge (connectors#86). This keeps devtown-specific preference knowledge inside devtown.

**Preference keys** (in `devtown-domain`, pure Java — follows the same pattern as `SlaPreferenceKeys`):

```java
public final class NotificationPreferenceKeys {
    public static final PreferenceKey<StringPreference> SLACK_CHANNEL =
        new PreferenceKey<>("devtown.notification", "slack-channel",
            StringPreference.of("#devtown-ops"), StringPreference::parse);
    public static final PreferenceKey<StringPreference> TEAMS_CHANNEL =
        new PreferenceKey<>("devtown.notification", "teams-channel",
            StringPreference.of(""), StringPreference::parse);
}
```

`PreferenceKey` is a 4-arg record: `PreferenceKey<T extends Preference>(String namespace, String name, T defaultValue, Function<String, T> parser)`. Values use `StringPreference` (implements `SingleValuePreference` → `Preference`), consistent with existing devtown preference keys.

**Resolution boundary:** devtown bridge observers resolve per-repo channel configuration and embed the resolved `targetChannel` in the `SubscribableEvent` POJO. The observer injects `PreferenceProvider`, calls `resolve(SettingsScope.of(repoPath)).getOrDefault(SLACK_CHANNEL)`, and writes the result into the event POJO. The platform/connector pipeline sees a plain string field — no devtown preference key coupling crosses the boundary.

For `SlaBreachEvent`, the bridge observer can read preferences directly from `SlaBreachContext.preferences()` (which carries the scoped `Preferences` for the breached task's scope) instead of injecting `PreferenceProvider`.

**Default behaviour:** `#devtown-ops` (the `defaultValue` in the preference key) applies when no per-repo override is configured. Repository A can override to `#team-a-alerts`; repository B to `#team-b-alerts`.

---

## Dependencies

### New dependencies

| Module | Dependency | Scope | Purpose |
|--------|-----------|-------|---------|
| `review/pom.xml` | `casehub-platform-api` | compile | `SubscribableEvent`, `NotificationTemplate`, `NotificationTarget` |
| `app/pom.xml` | `casehub-platform` | runtime | Subscription engine, notification dispatcher (when shipped) |
| `app/pom.xml` | `connectors-notification-bridge` | runtime | connectors#86 — external channel delivery (when shipped) |
| `app/pom.xml` | `subscriptions-inmem` | test | In-memory subscription store for `@QuarkusTest` |

---

## Test Strategy

### Unit tests (review/, no Quarkus)

- Each `SubscribableEvent` record: `type()` returns correct event type string, `tenancyId()` propagates
- Scenario-specific field mapping assertions

### Unit tests (app/, no Quarkus)

- Each bridge observer: given a foundation event, verify correct `SubscribableEvent` produced with all fields mapped. Use capturing `Event<T>` stub.
- `DevtownSubscriptionRegistrar`: verify all 6 subscriptions registered with correct event types, targets, templates (all 8 fields), severities, scopes

### Integration tests (app/, @QuarkusTest)

- Bridge observer CDI wiring: fire a foundation event → verify `SubscribableEvent` CDI event captured by a test observer
- Subscription idempotency: register twice → verify single subscription in store
- Use in-memory platform stores (subscriptions-inmem)
- Suppress Twilio/WhatsApp connectors via `quarkus.arc.exclude-types` (GE-20260521-45e61c)

### End-to-end pipeline test (activates when platform ships SubscriptionEngine)

- Fire a foundation event → verify `Notification` appears in the in-memory `NotificationStore` with correct title, body, category, severity
- Use `TestSlackConnector` to capture `ConnectorMessage` payloads and verify content

### What devtown does NOT test

- Platform dispatch pipeline internals (suppression, digest, channel routing) — platform's responsibility
- Connector delivery (Slack HTTP, Teams webhook) — connectors' responsibility
- The notification bridge module — connectors#86's responsibility

devtown tests the seam: foundation event in → correct `SubscribableEvent` fired with correct fields + correct subscriptions registered with correct definitions.

---

## Garden Gotchas

| Entry | Applies to | Rule |
|-------|-----------|------|
| GE-20260427-893862 | **Synchronous** bridge observers only | `@Observes(during = AFTER_SUCCESS)` + `@Transactional(NOT_SUPPORTED)`. Async source events (`CaseLifecycleEvent`, `WatchdogAlertEvent`) use `@ObservesAsync` + `@Transactional(NOT_SUPPORTED)` instead. |
| GE-20260521-45e61c | Test configuration | `quarkus.arc.exclude-types` for Twilio/WhatsApp in test `application.properties` |
| GE-20260607-0bfc83 | connectors#86 (not devtown) | Never regress delivery state inside connector try-catch |

---

## Done criteria (from issue #16)

> A failed merge bisect delivers a Slack message identifying the faulty PR. A stalled agent triggers an ops channel alert automatically.

With this design:
- Merge failure → `MergeFailedEvent` fired as CDI event → subscription engine match → platform dispatch → connectors#86 bridge → `SlackConnector.send()` → Slack message with PR title, author, failure reason
- Stalled commitment → `StalledCommitmentEvent` fired from `WatchdogAlertEvent(OBLIGATION_FAN_OUT)` → subscription engine match → platform dispatch → connectors#86 bridge → ops channel alert with condition type, target, summary

Both acceptance criteria satisfied when: (1) platform ships `SubscriptionEngine` + `NotificationDispatcher`, (2) connectors#86 is on the classpath, and (3) a Slack webhook is configured as a delivery channel with per-repo `PreferenceProvider` overrides.
