---
layout: post
title: "Forty-Three CDI Errors, Four Root Causes"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cdi, quarkus, build]
---

Forty-three CDI deployment failures at Quarkus augmentation. That's where the
session started.

Before that, a POM resolution failure — the local Maven cache had a stale
casehub-parent snapshot that predated `casehub-platform-api`. A local parent
install cleared it. Then the real list appeared.

Three distinct categories in those forty-three. `EventLogRepository`,
`CaseInstanceRepository`, and `SubCaseGroupRepository` all unsatisfied — the
persistence SPI gap, needs `casehub-engine-persistence-hibernate`, blocked until
claudony integration. `WorkloadProvider` ambiguous — the engine's
`CasehubWorkloadProvider` and casehub-work's `JpaWorkloadProvider` both on the
classpath, neither `@DefaultBean`. And `JQEvaluator` unsatisfied twice over:
`io.casehub.platform.expression.JQEvaluator` and
`io.casehub.engine.internal.jq.JQEvaluator`, both invisible to ARC.

I wanted to fix the JQEvaluator and WorkloadProvider issues now and file a
separate issue for persistence.

**The JQEvaluator gaps** came in two forms. The platform one was new since
engine#316 — the engine now injects `casehub-platform-expression`'s `JQEvaluator`,
and that module wasn't a production dep. We added it. `MockSecretManager` and
`MockConfigManager` come along as `@DefaultBean` implementations — no extra
wiring needed.

The engine-internal one was stranger. `javap` on the class confirmed
`@ApplicationScoped` is present in the bytecode, but ARC can't see it.
`casehub-engine-common` ships without a Jandex index. We added
`quarkus.index-dependency` entries — but scoped them to production only:

```properties
%prod.quarkus.index-dependency.casehub-engine.group-id=io.casehub
%prod.quarkus.index-dependency.casehub-engine.artifact-id=casehub-engine
%prod.quarkus.index-dependency.casehub-engine-common.group-id=io.casehub
%prod.quarkus.index-dependency.casehub-engine-common.artifact-id=casehub-engine-common
```

`@QuarkusTest` runs under the test profile. The `%prod.` prefix keeps the engine
from being indexed there — where the test properties already manage CDI discovery
carefully.

**The WorkloadProvider ambiguity** was straightforward once named:
`CasehubWorkloadProvider` is `@ApplicationScoped` without `@DefaultBean`, so it
competes with `JpaWorkloadProvider`. A `%prod.quarkus.arc.exclude-types` entry
suppressed the engine's version in production; the test config already excluded
`JpaWorkloadProvider` by name.

We also added `casehub-platform` for `MockPreferenceProvider`, which casehub-work's
`ExpiryLifecycleService` requires.

Down from forty-three to thirty-two. All persistence SPIs. Then the tests broke.

`DevtownBootTest` failed with `Unable to create Scheduler`. Not CDI —
Quartz initialisation. Claude traced it: the casehub-work jar in the local cache
was timestamped 17:20. A fix committed at 21:52 changed the default for
`casehub.work.routing.cursor.cleanup-cron` from `"0 2 * * ?"` to `"0 0 2 * * ?"`.

```
"0 2 * * ?"    ← 5 parts — unix cron, invalid for Quartz
"0 0 2 * * ?"  ← 6 parts — Quartz cron with seconds field, valid
```

The path that revealed it: adding `MockPreferenceProvider` satisfied
`ExpiryLifecycleService`, which activated `RoutingCursorCleanupJob`, which had a
`@Scheduled(cron = ...)` that Quartz had never previously parsed. A dormant bean,
invisible until we completed the dependency chain above it.

We rebuilt `casehub-work-runtime` from local source. Tests pass.

The remaining thirty-two failures are persistence-only — filed as devtown#40,
with three options: an in-memory dev profile, full reactive-PG wiring, or accept
the limitation until claudony lands. devtown#31 closed.
