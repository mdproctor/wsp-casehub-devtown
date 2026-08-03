---
layout: post
title: "One Field, Three Failures, and the Error That Wasn't an Exception"
date: 2026-08-03
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [java-records, binary-compatibility, debugging, snapshot-dependencies]
---

A routine dependency update broke a test that had nothing to do with the change. The root cause was a single new field on a Java record three repos away, and the reason it was hard to find is that Java's type hierarchy has a gap most people don't think about until it bites them.

The session started with two XS follow-ups from our SLA calibration work — adding `durationSeconds` to a context map, writing a doc comment. Twenty minutes of real work. The interesting part was everything else.

## The Symptom

The full test suite ran 549 tests. 548 passed. `HumanApprovalLifecycleTest` timed out after five seconds waiting for a WorkItem that never appeared. The test creates a CasePlanModel, fires a human-approval binding, and waits for the engine to create a WorkItem via the Vert.x event bus. It had been passing for months.

Running it in isolation: green. Every time. The failure only appeared in the full suite.

## The Trail

The test log showed three unrelated errors scattered among the passing tests:

```
ERROR [DefaultAsyncObserverExceptionHandler] Failure occurred while notifying
  an async Observer [method=CaseMemoryObserver#onCaseLifecycleEvent]
```

`CaseMemoryObserver` is new in the engine SNAPSHOT — it captures terminal lifecycle events (case completed, cancelled, failed) into the neocortex memory store. It fires on `@ObservesAsync`, which means it runs on Vert.x worker threads alongside everything else.

The observer was crashing. And because `@ObservesAsync` errors are handled by Quarkus's default exception handler — which logs and swallows — the crash was silent from the test runner's perspective. No test failed because of it. But the Vert.x event bus was left in a corrupted state, and the *next* test that needed the event bus — `HumanApprovalLifecycleTest` — found its `casehub.humantask.schedule` event silently dropped.

## The Root Cause

The neocortex memory API had added a field to `MemoryInput`:

```java
// Before
public record MemoryInput(String entityId, MemoryDomain domain,
    String tenantId, String caseId, String text,
    Map<String, String> attributes) {}

// After
public record MemoryInput(String entityId, MemoryDomain domain,
    String tenantId, String caseId, String text,
    Map<String, String> attributes, Double importance) {}
```

The engine SNAPSHOT was compiled against the old six-parameter constructor. At runtime, devtown pulled the new neocortex JAR with only the seven-parameter constructor. When `CaseMemoryObserver` tried to construct a `MemoryInput`, it hit `NoSuchMethodError`.

Here's the thing: `NoSuchMethodError` extends `Error`, not `Exception`. The observer's catch block:

```java
try {
    store.store(input);
} catch (Exception e) {
    LOG.warnf("failed to store memory: %s", e.getMessage());
}
```

This catches nothing. The `new MemoryInput(...)` call is *before* the try block anyway — it fails during object construction, not during `store()`. But even if it were inside the try, `catch (Exception e)` would still miss it. `Error` is a sibling of `Exception` in Java's throwable hierarchy, not a subclass.

The error propagated up through `@ObservesAsync`, hit the default handler, got logged, and left the Vert.x event bus in an undefined state. Five seconds later, a completely unrelated test timed out waiting for an event that would never arrive.

## The Fix

We fixed the immediate breakage — added `null` for `importance` across all `MemoryInput` and `Memory` call sites (sixteen in total). The engine itself needs rebuilding against the current neocortex API, which is a separate repo concern.

## What This Actually Teaches

Java records have no backward-compatible extension mechanism. Adding a field to a record changes the canonical constructor signature, and every compiled call site becomes a `NoSuchMethodError` waiting to happen. With classes, you can add a no-arg constructor or a builder. With records, you get a hard binary break.

In a single-repo world this is a compile error you fix immediately. In a multi-repo SNAPSHOT world, repo A compiles against the old API, repo B pulls the new API, and the failure happens at runtime — in a thread pool, behind an async observer, caught by a handler that logs but doesn't fail the test.

The debugging heuristic: when a test fails only in the full suite but passes in isolation, and the failure is a timeout rather than an assertion, look for async observer exceptions in the logs. The failing test is almost never the one with the bug.
