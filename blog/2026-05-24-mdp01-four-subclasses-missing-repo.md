---
layout: post
title: "Four Subclasses and a Missing Repository"
date: 2026-05-24
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cdi, quarkus, maven, ci, layer-2]
---

Layer 2 landed: every human-approval WorkItem now has a 24-hour deadline,
escalates to `pr-leads` on first breach, and signals the case context with
`{status: "sla-breach"}` on the second. The interesting part wasn't the
implementation — it was discovering how much the framework already handled.

The spec said `SlaBreachHandler` should observe `WorkItemLifecycleEvent` to
signal the case when a WorkItem expires. Reading `ExpiryLifecycleService` showed
the framework already dispatches `slaBreachPolicy.onBreach()` and fires a
separate `SlaBreachEvent` — but the `SlaBreachEvent` Javadoc showed
`@ObservesAsync` in its example.

Claude caught the discrepancy. `SlaBreachEvent` is fired via `Event.fire()` — the
synchronous CDI path. `@ObservesAsync` observers are only notified via
`Event.fireAsync()`. The Javadoc was wrong. We filed work#224 against casehub-work
and used `@Observes`.

With that: first expiry escalates the WorkItem in-place
(`candidateGroups → pr-leads`, `status → PENDING`), second expiry triggers the
handler, `caseHub.signal()` fires `CONTEXT_CHANGED`, case context updates. The
two-tier lifecycle test passed on the first run after the annotation change.

---

The 32 remaining CDI failures from last session took longer. The natural fix is
`quarkus.arc.selected-alternatives` — list the `@Alternative` beans, let Quarkus
activate them. We promoted `casehub-engine-persistence-memory` to compile scope
and tried every variant: `%prod.` prefix, no prefix, multiline backslash, with
and without `quarkus.index-dependency`. All produced the same 32 errors.

`selected-alternatives` activates alternatives for `@QuarkusTest` augmentation.
During `quarkus:build` it does nothing, silently. No warning, no error about
unresolvable alternatives — just the downstream CDI failures as if the property
doesn't exist.

CDI `@Alternative` is not inherited by subclasses. An `@ApplicationScoped` class
that extends an `@Alternative @ApplicationScoped` class is `@Default` — no
activation needed, no `selected-alternatives` required. We created four one-line
subclasses in `app/spi/`:

```java
@ApplicationScoped
public class DevtownEventLogRepository extends InMemoryEventLogRepository {}
```

Quarkus always indexes the application module's own source. The build passed.

---

That left CI still red — and had been, silently, since the repo was created.
GitHub Actions was failing with "casehub-parent:pom:0.2-SNAPSHOT absent." The
parent POM defines the `<repositories>` section pointing to
`maven.pkg.github.com/casehubio/*`. The problem is circular: Maven needs the
parent to know where to find packages; to find the parent, it needs to know where
to look.

Local builds work because `~/.m2/repository` already contains the parent from
previous installs. CI starts cold. Adding a `<repositories>` entry to devtown's
own pom — before Maven touches the parent — broke the circle. Three lines, CI
green.
