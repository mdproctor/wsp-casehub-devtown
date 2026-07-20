# Epic 9: Notification Wiring — casehub-connectors integration

**Issue:** devtown#16
**Branch:** issue-16-notification-wiring
**Date:** 2026-07-20
**Status:** Approved

---

## Architecture

devtown produces domain events and subscriptions. The platform notification system handles everything downstream — target resolution, suppression, user preferences, digest batching, channel routing, delivery tracking, and retry.

```
devtown CDI observers (bridge foundation events)
  ↓ insert SubscribableEvent POJO
Platform SubscriptionEngine (alpha network matching)
  ↓ SubscriptionMatched (async)
Platform NotificationDispatcher (target → suppression → template → channel routing)
  ↓ NotificationInput
Platform DeliveryChannelRegistry
  ├── InAppNotificationDeliverer (inbox + SSE)
  └── ConnectorNotificationDeliverer (connectors#86 — slack, teams, email, sms)
        ↓ ConnectorMessage
     casehub-connectors Connector.send()
```

devtown owns event POJOs, subscription definitions, and bridge observers. Everything right of the subscription engine is platform infrastructure.

**External dependency:** connectors#86 (notification delivery bridge — `NotificationDeliverer` → `Connector.send()` auto-discovery). In progress in casehub-connectors. devtown can develop and test against in-memory deliverers; external delivery activates when connectors#86 ships and the bridge module is on the classpath.

---

## Notification Scenarios

Six notification scenarios, each mapping to a `SubscribableEvent` POJO, a `Subscription` definition, and a CDI observer bridging the originating foundation event.

### Event type mapping

| # | Scenario | Event type | Source event | Severity | Targets |
|---|----------|-----------|-------------|----------|---------|
| 1 | Review assignment | `io.casehub.devtown.review.assigned` | Qhorus COMMAND dispatch | INFO | EVENT_FIELD (reviewerId) |
| 2 | Merge success | `io.casehub.devtown.merge.succeeded` | Merge decision (approved) | INFO | ENTITY_WATCHERS (PR authors) |
| 3 | Merge failure | `io.casehub.devtown.merge.failed` | Merge decision (rejected) / bisect result | URGENT | EVENT_FIELD (authorId) + GROUP (devtown-ops) |
| 4 | Stalled commitment | `io.casehub.devtown.commitment.stalled` | WatchdogEvaluationService | WARNING | GROUP (devtown-ops) |
| 5 | Case fault | `io.casehub.devtown.case.faulted` | Engine CaseLifecycleEvent(FAULTED) | URGENT | GROUP (devtown-ops) |
| 6 | Escalation | `io.casehub.devtown.workitem.escalated` | Work EscalationPolicy fires | URGENT | GROUP (devtown-ops) + EVENT_FIELD (assigneeId) |

### Templates

| Scenario | Title pattern | Body pattern | Category |
|----------|--------------|-------------|----------|
| Review assigned | `PR #{prNumber} assigned for {capability} review` | `{prTitle} by {authorName} — deadline {deadline}` | `devtown.review.assigned` |
| Merge succeeded | `Batch merged: {prCount} PRs` | `{prList}` | `devtown.merge.succeeded` |
| Merge failed | `Merge rejected: {prTitle}` | `CI failure: {failureReason} — author: {authorName}` | `devtown.merge.failed` |
| Stalled commitment | `Stalled reviewer on PR #{prNumber}` | `{reviewerName} — {elapsedTime} past deadline` | `devtown.commitment.stalled` |
| Case faulted | `Case faulted: {caseId}` | `Last known state: {lastKnownState}` | `devtown.case.faulted` |
| Escalation | `SLA breach: {workItemType} on PR #{prNumber}` | `Deadline {deadline} exceeded by {overdueTime}` | `devtown.workitem.escalated` |

All templates use platform `NotificationTemplate` with `{field}` placeholder substitution from event POJO properties. `entityType` is `"pr-review"` for all scenarios. `entityIdField` is `"prNumber"` (or `"caseId"` for case fault). `actorIdField` varies per scenario.

---

## Module Placement

All notification code lives in a single package: `review/src/main/java/io/casehub/devtown/review/notification/`.

| Component | Location | Count |
|-----------|----------|-------|
| `SubscribableEvent` records | `review/notification/` | 6 records |
| Bridge observers | `review/notification/` | 4-5 classes (some foundation events share an observer) |
| `DevtownSubscriptionRegistrar` | `review/notification/` | 1 class |
| Domain module changes | None | `devtown-domain` stays pure Java |
| App module changes | Dependencies only | Platform + connectors bridge on classpath |

### Event POJO pattern

Each scenario is a record implementing `SubscribableEvent`:

```java
public record MergeFailedEvent(
    String prNumber, String prTitle, String authorId, String authorName,
    String failureReason, String repoId, String prUrl,
    String tenancyId
) implements SubscribableEvent {
    @Override public String type() { return "io.casehub.devtown.merge.failed"; }
    @Override public String tenancyId() { return tenancyId; }
}
```

### Bridge observer pattern

Each observer translates a foundation CDI event into a `SubscribableEvent` and inserts it into the subscription engine:

```java
@ApplicationScoped
public class CaseFaultNotificationBridge {

    @Inject SubscriptionEngine engine;

    @Transactional(NOT_SUPPORTED)
    void onCaseFault(@Observes(during = AFTER_SUCCESS) CaseLifecycleEvent event) {
        if (event.status() != CaseStatus.FAULTED) return;
        engine.insert(new CaseFaultedEvent(
            event.caseId(), event.lastKnownState(), event.tenancyId()));
    }
}
```

`@Observes(during = AFTER_SUCCESS)` + `@Transactional(NOT_SUPPORTED)` pairing on all observers (GE-20260427-893862).

### Subscription registration

Programmatic at `@Startup` via `SubscriptionStore`. Subscriptions are code-versioned, not Flyway-seeded:

```java
@ApplicationScoped
public class DevtownSubscriptionRegistrar {

    @Inject SubscriptionStore subscriptionStore;

    void onStartup(@Observes StartupEvent event) {
        registerIfAbsent("devtown-merge-failed", "io.casehub.devtown.merge.failed",
            NotificationSeverity.URGENT,
            List.of(target(GROUP, "devtown-ops"), target(EVENT_FIELD, "authorId")),
            template("Merge rejected: {prTitle}", "CI failure: {failureReason}",
                "pr-review", "prNumber", "authorId"));
        // ... 5 more
    }
}
```

---

## Dependencies

### New dependencies

| Module | Dependency | Scope | Purpose |
|--------|-----------|-------|---------|
| `review/pom.xml` | `casehub-platform-api` | compile | `SubscribableEvent`, `SubscriptionStore`, `NotificationTemplate` |
| `app/pom.xml` | `casehub-platform` | runtime | `NotificationDispatcher`, `SubscriptionEngine`, in-app deliverer |
| `app/pom.xml` | `connectors-notification-bridge` | runtime | connectors#86 — external channel delivery |
| `app/pom.xml` | `notifications-inmem` | test | In-memory notification store for `@QuarkusTest` |
| `app/pom.xml` | `subscriptions-inmem` | test | In-memory subscription store for `@QuarkusTest` |
| `app/pom.xml` | `delivery-tracking-inmem` | test | In-memory delivery tracking for `@QuarkusTest` |

---

## Test Strategy

### Unit tests (review/, no Quarkus)

- Each `SubscribableEvent` record: `type()` returns correct event type string, `tenancyId()` propagates
- Each bridge observer: given a foundation event, verify correct `SubscribableEvent` produced with all fields mapped. Mock subscription engine insertion.
- `DevtownSubscriptionRegistrar`: verify all 6 subscriptions registered with correct event types, targets, templates, severities

### Integration tests (app/, @QuarkusTest)

- End-to-end: fire a foundation event → verify `Notification` appears in the in-memory `NotificationStore` with correct title, body, category, severity
- Use in-memory platform stores — no real Slack, no real database
- Use `TestSlackConnector` to capture `ConnectorMessage` payloads and verify content
- Suppress Twilio/WhatsApp connectors via `quarkus.arc.exclude-types` (GE-20260521-45e61c)

### What devtown does NOT test

- Platform dispatch pipeline internals (suppression, digest, channel routing) — platform's responsibility
- Connector delivery (Slack HTTP, Teams webhook) — connectors' responsibility
- The notification bridge — connectors#86's responsibility

devtown tests the seam: domain event in → correct notification out.

---

## Garden Gotchas

| Entry | Applies to | Rule |
|-------|-----------|------|
| GE-20260427-893862 | All bridge observers | `@Observes(during = AFTER_SUCCESS)` + `@Transactional(NOT_SUPPORTED)` pairing |
| GE-20260521-45e61c | Test configuration | `quarkus.arc.exclude-types` for Twilio/WhatsApp in test `application.properties` |
| GE-20260607-0bfc83 | connectors#86 (not devtown) | Never regress delivery state inside connector try-catch |

---

## Done criteria (from issue #16)

> A failed merge bisect delivers a Slack message identifying the faulty PR. A stalled agent triggers an ops channel alert automatically.

With this design:
- Merge failure → `MergeFailedEvent` → subscription match → platform dispatch → connectors#86 bridge → `SlackConnector.send()` → Slack message with PR title, author, failure reason
- Stalled commitment → `StalledCommitmentEvent` → subscription match → platform dispatch → connectors#86 bridge → ops channel alert with reviewer name, PR number, elapsed time

Both acceptance criteria satisfied when connectors#86 is on the classpath and a Slack webhook is configured as a delivery channel.
