---
layout: post
title: "Two Experiences, One Backend"
date: 2026-10-27
entry_type: article
subtype: diary
projects: [casehubio/devtown]
tags: [frontend, architecture, evolution, routing, casehub]
series: issue-268-evolution-standalone-app
---

# Two Experiences, One Backend

The evolution conductor was supposed to be a dashboard tab. That was the plan in #230 — wire the workbench component, mount it as tab nine in the review dashboard, ship it. The backend was done. The packages were built. I just needed to add two lines to `package.json` and one `hostPanel()` call.

I didn't do that.

The review dashboard is operational. It answers the question a developer asks every morning: what's happening with my PRs right now? Six tabs — PR cases, reviewers, contributors, workers, system health, definitions. Each tab is a different lens on the same question. They belong together because the mental model is the same: observe the current state, act on it, move on.

The evolution conductor asks a different question entirely. It's strategic. Which improvement streams are active? What's the conductor proposing to change about how reviews get routed? What's in the inbox waiting for a human decision? Should we tighten the gate policy or pause a category? These aren't operational questions with operational answers. They're steering decisions that shape how the operational system behaves over weeks and months.

Putting both in one dashboard would have been quick. It also would have collapsed two mental models into a single navigation space, and that's the kind of shortcut that calcifies. A tab labelled "Evolution" sitting next to "PR Cases" implies they're the same kind of thing. They're not. One is a control plane, the other is a data plane.

## The one-way dependency

The relationship between the two experiences is intentionally asymmetric. Evolution reads from the review system's event log — it needs to know what reviews happened in order to identify improvement patterns. Review knows almost nothing about evolution. The only intrusion is a small conductor-origin badge on PR detail: when the evolution conductor created a PR (because it identified an improvement opportunity), the badge shows up with a link back to `/evolution`.

This matters for the platform story. CaseHub applications should compose, not merge. The engine foundation provides cases, bindings, trust scoring, and audit. Domain applications — devtown being the first — provide the domain logic on top. Within devtown, the operational view and the strategic view are two applications that happen to share a Quarkus backend and a JavaScript bundle. They don't share navigation, and they don't share state beyond what the REST API exposes.

## How it actually works

The implementation is simpler than it sounds. One `index.ts` entry point, one esbuild bundle, one `index.html`. Before calling `loadSite()`, the start function checks `window.location.pathname`. If it starts with `/evolution`, it dynamic-imports the evolution view and mounts the workbench. Otherwise, it dynamic-imports the review view and mounts the dashboard.

```typescript
if (window.location.pathname.startsWith("/evolution")) {
  const { mountEvolutionPage } = await import("./views/evolution");
  mountEvolutionPage(container, user.tenancyId, prefersDark);
} else {
  const { mountReviewDashboard } = await import("./views/review");
  await mountReviewDashboard(container, siteOptions, prefs, prefersDark, initialCaseId);
}
```

Both paths share auth, preferences, and theme setup. The evolution workbench already has its own tabs — summary, streams, inbox, audit, config, health — so it fills the page directly below a single back-link to the review dashboard. No `page()`/`tabs()` wrapper needed.

Cross-app linking works through query parameters. The evolution workbench renders deep links on improvement streams. Clicking one navigates to `/?caseId={prCaseId}`, which the review dashboard reads on mount and passes to the PR Cases workbench for auto-selection. Full page navigation, not an overlay — clean separation.

## The bootstrap lesson

One thing we fixed along the way: `DevtownEvolutionBootstrap` was calling `startCase()` with no try-catch. If the evolution case failed to bootstrap — engine unavailable, migration error, anything — the entire Quarkus application crashed at startup. A strategic feature that's not yet critical was taking down the operational system it's meant to improve. We wrapped it in a try-catch with a clear log message: evolution features become unavailable, but reviews keep working.

That's the pattern to hold onto. The strategic layer should never take down the operational layer. If evolution is having a bad day, PR reviews still happen. The dependency is one-way not just in code, but in failure modes.

## What this opens up

The standalone page is a foundation, not a destination. The evolution workbench component is built but the conductor's full decision loop — proposing improvements, gating them through human review, measuring outcomes — is still being wired end to end. Having `/evolution` as its own space means that loop can grow without competing for real estate in the review dashboard.

It also sets a precedent for how the next CaseHub domain application should think about its UI. Not "add a tab to the monolith," but "what are the distinct mental models, and does each one deserve its own navigation space?" Sometimes the answer will be a tab. Sometimes it'll be a page. The question is what forces the right answer.
