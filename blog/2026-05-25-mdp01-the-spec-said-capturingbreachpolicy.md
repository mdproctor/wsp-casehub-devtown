---
layout: post
title: "The Spec Said CapturingBreachPolicy"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cdi, quarkus, testing, layer-2]
---

The spec called for a `CapturingBreachPolicy @Alternative @ApplicationScoped` static inner
class — observe `SlaBreachEvent` calls from `ExpiryLifecycleService`, record the
`SlaBreachContext` received, verify the policy isn't NoOp.

Reasonable on the surface. But `selected-alternatives` is a build-time property.
Activating `CapturingBreachPolicy` as an `@Alternative` displaces `SlaBreachPolicyBean` for
every test in the file. The displacement check — the primary CDI concern — would be
untestable in the same class.

The second problem: testing that `ExpiryLifecycleService` dispatches to `SlaBreachPolicy`
is a casehub-work concern. `SlaBreachLifecycleTest` already covers it at Checkpoints 3 and
4. A wiring test that duplicates integration coverage while breaking displacement is worse
than what it replaces.

I brought Claude in to settle the alternative. We went with two methods and no alternatives
at all:

```java
@Test
void slaBreachPolicyBean_displaces_noOp() {
    assertThat(policy).isInstanceOf(SlaBreachPolicyBean.class);
}

@Test
void slaBreachHandler_onFail_signalsCaseContext() throws Exception {
    UUID caseId = caseHub.startCase(MINIMAL_CTX).toCompletableFuture().get(5, SECONDS);
    assertThat(caseId).isNotNull();
    breachEvents.fire(new SlaBreachEvent(ctx, new BreachDecision.Fail("sla-breach")));
    await().atMost(5, SECONDS).untilAsserted(() ->
        assertThat(instance.getCaseContext().getPath("humanApproval.status"))
            .isEqualTo("sla-breach"));
}
```

`Event<SlaBreachEvent>` injected as the test driver — fires directly, bypasses
`ExpiryLifecycleService`. Since `@Observes` is synchronous CDI, the handler completes
before `fire()` returns. If `@Observes` is accidentally changed to `@ObservesAsync` — the
Layer 2 gotcha — the second test times out: the event dispatches off-thread and the await
windows don't overlap. Two concerns, two tests, no build-time changes.

The test context needed care. `linesChanged=100` below the threshold keeps the
`human-approval` binding from firing after `startCase()` resolves. Without it, firing the
breach event into an already-active binding is a data race. We went through all eight
bindings in the pr-review YAML to confirm none would fire with the pre-seeded context.

One postscript. `SlaBreachContext`'s Javadoc says to construct tests with
`MapPreferences.empty()`. I wrote it in the spec. Claude flagged it during implementation:
`cannot find symbol: method empty()`. `MapPreferences` has one constructor, no static
factories. The Javadoc is wrong; use `new MapPreferences(Map.of())`.
