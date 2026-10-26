---
layout: post
title: "Forty-Three ObjectMappers Walk Into a Bar"
date: 2026-10-26
entry_type: note
subtype: diary
projects: [casehubio/devtown]
tags: [jackson, serialization, casehub-engine, cross-repo]
---

# Forty-Three ObjectMappers Walk Into a Bar

Jackson's `ObjectMapper` has a well-known quirk: `new ObjectMapper()` can't serialize `java.time.Instant`. You need to register `JavaTimeModule` explicitly, or the whole thing blows up with `IllegalArgumentException`. Quarkus handles this for you — its CDI-managed ObjectMapper comes pre-configured. But if you're in a POJO module with no CDI, and you wrote `private static final ObjectMapper MAPPER = new ObjectMapper()`, you're on your own.

The casehub-engine has forty-three of them.

I found this because `WorkerScheduleEventHandler.buildEventLog()` serializes `RetrievedMemory` objects into EventLog metadata. `RetrievedMemory` has an `Instant createdAt` field — added when we wired agent memory retrieval. The engine retries a failed worker (REROUTE), builds the event log, hits the `Instant`, and throws. The simulated review pipeline e2e test couldn't exercise the case-plan-engine path at all.

The root cause was obvious once I saw it. The fix options were not. Three choices: annotate `RetrievedMemory.createdAt` with Jackson annotations (fixes the symptom, not the cause), inject the CDI ObjectMapper through every handler's constructor (architecturally correct but touches 43 constructors, two CDI producer classes, all tests), or create a shared factory.

I went with the factory. `EngineMappers.defaultMapper()` in `common-core` — one method, returns `new ObjectMapper().registerModule(new JavaTimeModule())`. Each call site stores the result in a `static final` field, so the factory runs once per class load. No shared mutable state, no constructor changes, no API changes.

The interesting part was scope. I started planning to fix just `WorkerScheduleEventHandler` and file a follow-up for the rest. But twenty-two bare `new ObjectMapper()` calls in `runtime-core` alone, another ten in `common-core`, nine more in `planning-core` — any of those will fail the moment they touch a Java 8 date/time type. The follow-up issue would just sit there until someone else hit the same crash in a different handler. Fixing them all took less time than writing the issue would have.

One module — `annotations/runtime` — doesn't depend on `common-core`. Adding a dependency for a single ObjectMapper usage would be over-coupling. That one registers `JavaTimeModule` inline. The rest use the factory.

The engine is a cross-repo dependency — four commits on `main` there, no devtown code changes. When devtown rebuilds against the updated engine SNAPSHOT, the `Instant` serialization error disappears and the e2e test path unblocks.
