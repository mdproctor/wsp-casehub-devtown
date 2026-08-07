---
layout: post
title: "Contributor Trust Gets a Face"
date: 2026-08-07
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust, ui, contributor, intake-lanes, governance]
---

The trust model for contributors has been accumulating data since the attestation pipeline landed — intake lane classifications, dimension scores, PR outcome history — but none of it was visible outside the API. This session gave it a UI.

The contributor fleet view works like the existing reviewer fleet: a data table listing every known contributor with their trust score, intake lane, observation count, and dimension breakdown (merge-rate and first-attempt-quality). Clicking a row opens a contributor workbench showing the full detail — intake classification with the policy thresholds that produced it, and a scrollable outcome history from ledger attestations.

The interesting backend problem was enumeration. Contributors don't have a single authoritative registry. We pull from two sources: the trust export service (actors with existing `pr-contribution` capability scores) and the case tracker (PR authors from active cases who may not have trust history yet). Union by actor ID, classify each through `ContributorIntakePolicy` using the current preference-driven thresholds, and the fleet assembles itself. New contributors with no history land in TRIAGE — the same classification the runtime routing uses.

The merge queue got the same treatment. Clicking a queued PR now shows the author's trust profile in a contributor-workbench panel — intake lane rationale, dimension scores, recent outcomes. No additional backend work; it reuses the same `/contributors/{actorId}` endpoint. The identity mapping between queue entries and trust actors was already consistent, so the wiring was straightforward.

Claude caught something worth mentioning during code review: a `/dev/seed-contributors` endpoint I'd left in `GovernanceResource` — a quick debug tool that used `CDI.current().select()` to bypass injection and insert test trust data directly. It was sitting behind `@PermitAll` with fully-qualified class names throughout, exactly the kind of thing that ships to production and then surprises everyone. Removed it.

A separate fix fell out of the rebase: `CbrReviewerMatchingIntegrationTest` had been failing because `ImplementationCandidate`'s record fields were reordered upstream from `(bindingName, capabilityName, workerName)` to `(bindingName, workerName, capabilityName)`. The test was passing capability as worker and vice versa, so trust score lookups silently missed and both candidates fell into BOOTSTRAP with identical scores — producing `RunAll` instead of `Selected`. The kind of bug that passes compilation, produces a plausible-looking result, and only surfaces when you actually read the failure message.

The contributor trust model now has the same visibility the reviewer model has had since the trust workbench landed. The next question is whether intake lane transitions — a contributor moving from TRIAGE to STANDARD to FAST_TRACK as their history accumulates — should surface as events in the governance stream.
