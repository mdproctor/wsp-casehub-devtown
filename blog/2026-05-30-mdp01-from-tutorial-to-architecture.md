---
layout: post
title: "From tutorial to architecture"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [arc42stories, layer-log, documentation]
---

The Layer 3 code was done. What wasn't done was writing it down.

Yesterday's session shipped `QhorusPrReviewService`, three agent stubs, five
`@QuarkusTest` assertions, and a blog entry about what it felt like to get
there. The LAYER-LOG entry — the architectural record every subsequent session
will read before touching Layer 3 — was in the handover's "What's Left" column.
So was an ARC42STORIES.MD update. And two issues that had been sitting open
for weeks despite their work being long merged. I'd also checked the handover's
top item, issue #41, and found it closed and on main — done in an earlier
session I'd forgotten.

So this was a cleanup session. We wrote the LAYER-LOG Layer 3 entry, updated
ARC42STORIES.MD to reflect Chapter 3 complete, and closed the stale issues.

The LAYER-LOG entry structure is the same for every layer: what it adds,
accountability gaps closed, key wiring decisions, gotchas, pattern to replicate.
The first three sections were mostly recall — the session was still fresh. The
gotchas required verification: I wanted to confirm the garden entry references
were accurate before citing them. They were.

Where it got more interesting was the code review Claude ran before committing.
Two findings worth noting. First, the LAYER-LOG claimed DONE discharges a
Commitment with "state → FULFILLED" — accurate as an internal fact, but the
test only asserts `findOpenByObligor` returns empty. The state label lives inside
qhorus. We hedged it: "the test-observable form is `commitmentStore.findOpenByObligor(...) isEmpty()`."
Second, the pattern-to-replicate section didn't note that DONE and DECLINE
don't set `target` on the message — only COMMAND does. A domain implementer
following the pattern step by step might add target to the response, which is
wrong. One line added.

The ARC42STORIES update was more mechanical: status header, flowchart style,
Chapter 3 full entry replacing the placeholder, the Formal DECLINE row in §11.
Nothing surprising — the code was clean, so the documentation was mostly
transcription.

The second half of the session was different. An instruction arrived to strip
the tutorial/showcase framing from CLAUDE.md and LAYER-LOG.md — language that
had accumulated from the original "reference architecture and field showcase"
positioning. The Arc42Stories direction for CaseHub harness apps is
production-first: the system is what it is, the documentation records what was
built, and any teaching value emerges from that record rather than being
designed in.

The changes were editorial but not trivial:

- "Primary goal: Reference architecture and field showcase" → a single
  production-first goal statement
- "Secondary goal: LLM and human tutorial material" → removed
- "Tutorial Structure" → "Foundation Layers"
- "What it shows" → "What it adds" throughout (four occurrences)
- "teaching placeholders" → "in-process placeholders for future Claudony agents"
- "This split is mandatory for the tutorial to work" → "This split enforces
  the dependency rule"

Small changes individually. The cumulative effect matters. "What it shows"
implies a reader being shown something. "What it adds" states what exists.
The framing shifts from exhibition to record, which is what Arc42Stories
requires of a §9.4 layer entry.

The `casehub-life` LAYER-LOG was the reference for what the cleaned-up version
should look like — sparse, direct, production-capability language throughout.
Devtown's version is longer because it has more decision surface (the Gastown
comparison generates significant rationale), but the register is now consistent.
