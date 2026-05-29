---
layout: post
title: "Arc42Stories Meets Reality"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-parent, casehub-devtown]
tags: [arc42stories, documentation, reference-architecture]
---

The goal going in was simple: generate ARC42STORIES.MD for devtown from LAYER-LOG.md, DESIGN.md, and the Arc42Stories spec. Claude had the source material. It would be a documentation session.

It turned into a design session for the spec itself.

## The Document Arrived with Tutorial Residue

Claude generated ARC42STORIES.MD — a solid first pass. Reading it back, though, the framing was wrong in a consistent way. The secondary goal was described as "enabling replication" but the word "tutorial" kept appearing. Layers were described as "independently teachable." The YAML was "the teaching artifact." The port interface placement in `review/` was "mandatory for the tutorial to work."

None of it was wrong exactly. But it kept positioning the architecture as existing in service of documentation rather than the other way around. We'd agreed early in this project that domain apps are reference architectures, not tutorials. The document didn't reflect that.

We worked through ARC42STORIES.MD, AGENTIC-HARNESS-GUIDE.md, and tutorial-strategy.md and removed or reframed the tutorial language. The position that stuck: tutorial content is a side effect of documenting a reference architecture well. Spot tutorials and architectural highlights are extracted from it — they don't drive it.

## The Chapter Entries Were Too Heavy

The spec's Chapter entry template had fifteen fields. The generated chapters (C1, C2) ran to 50-60 lines each — full paragraph descriptions of purpose, four-column Layer Impact tables, C4 component diagrams, a Dependencies and risks section.

Most of it duplicated the Layer entries directly below.

The chapter/layer duality exists for a reason: Chapters are the delivery story (what shipped, when, why it mattered) and Layers are the implementation story (how it was built, what not to do). Chapters should be 10-20 lines — delivery metadata, 2-3 sentences on what works end-to-end after it ships, accountability gaps as bullets, and a two-column Layer Impact table. The detail lives in the Layer entry.

We rebuilt the spec's Chapter template to match and trimmed C1 and C2 accordingly. We also added a Layer × Chapter matrix to §9.2 — a simple table showing which layers each chapter touches, readable in both directions. Immediately useful for seeing which layers are foundational (L5 participates in every chapter) versus additive (L3 appears in one).

## The Pruning Dead End

When I raised the document growth problem, the first proposal was PLAYBOOKS.md — extract the "Pattern to replicate" sections from stable layers into a separate replication guide.

I pushed back: would the extracted steps be usable without the Gotchas and Key wiring that contextualise them? Probably not. If you add those, you're duplicating the layer entry. Then the sharper question: is the layer itself the pattern?

Yes. The "Pattern to replicate" section works because it sits next to the specific wiring and gotchas that ground it. Extract it and you get steps without context. The document grows as layers are added — that's correct. The answer to size is navigation (matrix, headers, `git log --grep` links), not extraction into satellite documents that inevitably drift.

No pruning policy. No PLAYBOOKS.md. One clean conclusion: a well-navigated 1500-line document beats two poorly-maintained 750-line ones.

## What the Spec Was Missing

Three gaps we added:

**Definition of done.** When is ARC42STORIES.MD genuinely useful? The spec didn't say. Now it has three tiers: functional (🔲 markers allowed, expected during development), complete (no markers, at Journey close), and not yet useful (§1-8 without §9 adds almost nothing over plain arc42). The minimum viable ARC42STORIES.MD: §1 with artifact schema, §3 context diagram, §9.2 Chapter Index with matrix, and at least one complete §9.4 Layer entry.

**Why this exists — three new questions.** The original spec answered how the system is built incrementally, how an LLM picks up a session, and how a different domain replicates the architecture. We added: how does a new team member onboard without reading git log, how does an autonomous agent know what's done and what to avoid, and how does a platform team track coherence across multiple reference architectures when all of them have ARC42STORIES.MDs.

**Migration guide.** All five existing CaseHub projects — devtown, AML, clinical, QuarkMind, life — have LAYER-LOG.mds and can't start fresh. The guide maps what goes where and names the natural transition point: finish a layer, write the ARC42STORIES.MD entry instead of the LAYER-LOG.md entry.

tutorial-strategy.md, which had been the planning document for how each reference architecture would be built layer by layer, is now archived. The planning is obsolete for completed layers. What mattered has been absorbed.
