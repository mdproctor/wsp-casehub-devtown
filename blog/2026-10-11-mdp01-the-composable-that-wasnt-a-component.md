---
layout: post
title: "The Composable That Wasn't a Component"
date: 2026-10-11
entry_type: note
subtype: diary
projects: [casehubio/devtown]
tags: [frontend, lit, composition, refactoring, workbench-unification]
---

# The Composable That Wasn't a Component

Three detail panels in the devtown dashboard — merge queue, contributors, reviewers — had grown independently from the same DNA. Trust bars, dimension grids, status cards, review history cards, badge pills: the same rendering logic copied three times with different variable names and slightly different margins.

The issue (#231) called for extracting shared components. The interesting question was what shape those shared pieces should take.

The obvious move with Lit is `@customElement` — give each piece its own Shadow DOM, its own styles, its own tag name. That's how blocks-ui works. But these fragments are never used standalone. They're always inside a detail panel's Shadow DOM, always composed inline, never placed independently in HTML. A custom element boundary here means Shadow DOM isolation you don't want (can't inherit parent CSS variables without explicit forwarding) and inter-component event plumbing you don't need.

The better fit turned out to be plain render functions — `renderTrustBar(data)` returns a `TemplateResult`, the caller drops it into their template with `${}`, and the CSS lives in a shared styles module imported into the host's `static styles` array. This is how `renderSparkline` already works in blocks-ui-core. Functions compose where elements isolate.

We extracted five render functions (`renderTrustBar`, `renderDimensionGrid`, `renderStatusCard`, `renderReviewCard`, `renderBadge`), a shared CSS module with eight exported fragments, and four helper functions. Each detail panel now imports only what it uses.

The merge-queue detail also gained something new: review activity cards showing which agents reviewed a PR and what they found. The data was already available from the review detail endpoint — we just needed to fetch it when a PR is selected and render each completed capability as a review card.

One thing the code review caught: when I unified the `_fillClass` helper across panels, the reviewer-detail's original version used three tiers (high/mid/low) while the shared `trustFillClass` uses four (adding `very-low` for scores below 0.40). The panel-specific capability bar CSS didn't have a `.cap-fill.very-low` variant. Scores below 0.40 are rare for agent capabilities, but the missing class would have rendered an invisible bar — the kind of visual bug you only notice when the data finally hits that range.

The net result: 327 lines removed from three files, 286 lines added across eight new shared files. Each rendering pattern exists exactly once now.

But the composable extraction was just the opening move. Once the detail panels shared the same visual language, a bigger question surfaced: why are Reviews, Merge Queue, and Triage separate tabs at all?

A PR in the Reviews tab is being actively reviewed. The same PR in the Merge Queue tab has passed review and is queued for merge. The Triage tab shows PRs that need a human decision — a subset of active reviews. Three tabs, one entity, different lifecycle stages. The detail panels are now identical. The only things that differ are the action buttons (approve vs dequeue) and some queue-specific context (wait time, batch membership, dependencies).

We extended the shared composables to the fourth detail panel — `review-detail` — and added the same trust context, routing decisions, capability checks, and findings that merge-queue-detail already showed. Then we went the other direction: enriched merge-queue-detail with routing decisions, the findings panel, and the event timeline that had been review-only. Both panels now show everything. Same PR, same depth, different lifecycle buttons.

The routing summary's "Show code analysis" section was dumping a raw Java `toString()` string — `{securitySensitive=true, scope=multi-module, flaggedFiles=[auth/TokenHandler.java]}`. A small parser turns it into a key-value table. The Workers tab was empty because the mock fixtures used `sessionId`/`agentId` fields while the `blocks-session-list` component expects `id`/`name`/`workingDir` — a field-name mismatch, not a missing feature.

The realisation that three tabs are the same view led to a design for a unified PR Cases workbench (#233) — one split-pane with filter chips replacing three separate tabs. The work queue model from blocks-ui already handles the triage filtering. Batches fold in as secondary context under the "Queued" filter rather than a competing list. The composable foundation we built makes this possible — the detail panel is already unified, so the workbench unification is about the list, vitals, and filter state.
