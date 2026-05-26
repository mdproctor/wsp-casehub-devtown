---
layout: post
title: "After the squash"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
---

After a squash, there's always a debris field.

Yesterday's history compaction took devtown's main branch from 40
commits down to 17. Three commits were skipped due to conflicts. One
of them contained the tests that were supposed to close issue #32.
GitHub showed the issue closed. The tests were not in main.

The handoff also noted that PR#47 was "already merged to upstream."
It wasn't. The squash rewrite made the pre-squash PR unmergeable —
GitHub reported it CONFLICTING — and it was sitting open. The
upstream history shifted, the PR couldn't compute a merge base, and
nothing flagged it.

We added the two missing tests and closed PR#47 with a note. The
tests are the symmetric counterparts to
`styleCheck_doesNotFire_whenAlreadyDone`, which was already there.
All three parallel checks share the same `codeAnalysis.complete`
gate:

```java
@Test void testCoverage_doesNotFire_whenAlreadyDone() {
    assertThat(condition("test-coverage").test(ctx(Map.of(
        "pr", pr(100),
        "codeAnalysis", analysis(false, false),
        "testCoverage", Map.of("outcome", "APPROVED"))))).isFalse();
}
```

Symmetry restored. Layer 3 next.
