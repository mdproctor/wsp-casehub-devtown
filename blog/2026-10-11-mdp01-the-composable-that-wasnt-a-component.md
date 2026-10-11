---
layout: post
title: "The Composable That Wasn't a Component"
date: 2026-10-11
entry_type: note
subtype: diary
projects: [casehubio/devtown]
tags: [frontend, lit, composition, refactoring]
---

# The Composable That Wasn't a Component

Three detail panels in the devtown dashboard — merge queue, contributors, reviewers — had grown independently from the same DNA. Trust bars, dimension grids, status cards, review history cards, badge pills: the same rendering logic copied three times with different variable names and slightly different margins.

The issue (#231) called for extracting shared components. The interesting question was what shape those shared pieces should take.

The obvious move with Lit is `@customElement` — give each piece its own Shadow DOM, its own styles, its own tag name. That's how blocks-ui works. But these fragments are never used standalone. They're always inside a detail panel's Shadow DOM, always composed inline, never placed independently in HTML. A custom element boundary here means Shadow DOM isolation you don't want (can't inherit parent CSS variables without explicit forwarding) and inter-component event plumbing you don't need.

The better fit turned out to be plain render functions — `renderTrustBar(data)` returns a `TemplateResult`, the caller drops it into their template with `${}`, and the CSS lives in a shared styles module imported into the host's `static styles` array. This is how `renderSparkline` already works in blocks-ui-core. Functions compose where elements isolate.

We extracted five render functions (`renderTrustBar`, `renderDimensionGrid`, `renderStatusCard`, `renderReviewCard`, `renderBadge`), a shared CSS module with eight exported fragments, and four helper functions. Each detail panel now imports only what it uses.

The merge-queue detail also gained something new: review activity cards showing which agents reviewed a PR and what they found. The data was already available from the review detail endpoint — we just needed to fetch it when a PR is selected and render each completed capability as a review card.

One thing the code review caught: when I unified the `_fillClass` helper across panels, the reviewer-detail's original version used three tiers (high/mid/low) while the shared `trustFillClass` uses four (adding `very-low` for scores below 0.40). The panel-specific capability bar CSS didn't have a `.cap-fill.very-low` variant. Scores below 0.40 are rare for agent capabilities, but the missing class would have rendered an invisible bar — the kind of visual bug you only notice when the data finally hits that range.

The net result: 327 lines removed from three files, 286 lines added across eight new shared files. Each rendering pattern exists exactly once now. The three panels are shorter and the duplication is gone, but the real payoff is that the next detail panel — whatever it turns out to be — can import `renderTrustBar` and `renderStatusCard` instead of copying them a fourth time.
