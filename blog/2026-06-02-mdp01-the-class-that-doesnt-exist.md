---
layout: post
title: "The class that doesn't exist"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [arc42stories, documentation, quality]
---

The build was clean — 174 tests. What the session was actually about is making architecture documentation you can trust.

devtown's ARC42STORIES.MD had been generated from LAYER-LOG entries but never verified against the codebase. Three layers were missing. A fourth had been blocked on engine#326 for months. The document made architectural claims about module placement and CDI annotations that had quietly become wrong.

Engine#326 closed before this session, unblocking Layer 2.

We worked through Layers 3, 6, and 2. Three and six were transcription — LAYER-LOG entries existed, the work was writing them into §9.4 format. Layer 2 required writing the LAYER-LOG entry first: the breach escalation path, the sealed `BreachDecision` interface, stateless tier detection via `candidateGroups`. Once written, transcribed.

The more interesting part was the quality sweep.

Three checks, run after the entries were written, caught what the text alone couldn't.

Claude verified every class name in the Key files sections with `find`. `HumanApprovalLifecycleHandler` appeared in three places — the §5 C4Container diagram, a Chapter 2 Layer Impact table, a C4Component diagram. The class doesn't exist. The bridge from `WorkItemLifecycleEvent` to `PlanItem` is in `casehub-work-adapter`, a foundation extension — devtown contributes nothing here. The name was plausible enough to survive the migration without triggering doubt.

Then `gh issue view` for every tracked issue in §12. Six issues listed as active risks were closed — engine#330, work#211, parent#26, platform#24, work#212, devtown#31. LAYER-LOG entries were accurate when written; the issues closed in the months between. The document looked complete.

Then CDI annotations. Layer 5's `PrReviewCaseService` was documented as `@ApplicationScoped (no @DefaultBean)`. That annotation became stale when Layer 3 introduced `QhorusPrReviewService @Alternative @Priority(1)` — at that point both competing implementations needed `@Alternative @Priority(N)`. The Layer 5 LAYER-LOG predated Layer 3 and never got updated.

None of these are visible from reading the document. Each required querying an external source: filesystem, GitHub, source files.

One correction came from a different direction. I had written into the cross-repo migration issues that LAYER-LOG.md should remain as a "source-of-truth draft" after migration — both documents kept in sync permanently. That's wrong. Two documents to sync is obviously worse than one. The CasHub profile already says these files are retired when migration is complete. The rule I'd invented had no source.

We created migration issues for clinical, aml, life, and openclaw. The three systematic checks are in each one, alongside specific layer state and the correct retirement policy.
