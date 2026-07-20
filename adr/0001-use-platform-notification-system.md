# ADR-0001: Use platform notification system for devtown notifications

**Status:** Accepted
**Date:** 2026-07-20
**Issue:** devtown#16

## Context

devtown needs to notify human operators and PR authors about 6 scenarios: review assignments, merge success/failure, stalled commitments, case faults, and SLA escalations. casehub-work and casehub-clinical each built their own notification infrastructure on top of casehub-connectors. A third custom implementation would compound platform fragmentation.

## Decision

Use `casehub-platform`'s notification system — `SubscribableEvent`, `SubscriptionStore`, `NotificationDispatcher`, `DeliveryChannelRegistry` — rather than building custom notification infrastructure in devtown.

devtown provides:
- `SubscribableEvent` POJOs (6 records in `review/notification/`)
- CDI bridge observers translating foundation events to subscribable events (4 classes in `app/notification/`)
- `Subscription` registrations at startup via `SubscriptionStore` (1 registrar in `app/notification/`)
- Per-repo channel resolution via `PreferenceProvider` (preference keys in `domain/notification/`)

The platform provides target resolution, suppression, user preferences, digest batching, channel routing, delivery tracking, and retry.

External delivery (Slack, Teams) routes through connectors#86 (`NotificationDeliverer` → `Connector.send()` bridge), in progress in casehub-connectors.

## Consequences

- devtown has zero notification infrastructure code — only domain events and subscription definitions
- New notification scenarios require only a new `SubscribableEvent` record, a bridge observer, and a subscription registration
- casehub-work's existing `NotificationDispatcher` should eventually migrate to the platform system (not devtown's concern)
- Full pipeline activation depends on platform shipping `SubscriptionEngine` + `NotificationDispatcher` and connectors#86 shipping the delivery bridge
