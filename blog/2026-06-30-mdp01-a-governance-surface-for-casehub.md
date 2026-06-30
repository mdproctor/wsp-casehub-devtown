---
layout: post
title: "A Governance Surface for CaseHub"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [governance, casehub-pages, quinoa, ui]
---

## A Governance Surface for CaseHub

Gastown's community has built two impressive UIs. gastown-gui gives you operational completeness — every `gt` CLI operation has a GUI equivalent, from rig management to crew workspaces. gastown-viewer-intent goes deeper into observability: a kanban board, dependency graphs, convoy progress tracking, even a memory inspection panel. Both are serious tools built by people who understand what operators need.

DevTown's governance workbench builds on different foundations. CaseHub's platform provides trust scoring, formal obligation tracking, SLA-bounded human oversight, and cryptographic audit — capabilities that exist at the data model level, not as application features. The workbench is the first visual surface for these: six views that let you watch trust scores evolve, SLA deadlines count down, routing decisions explain themselves, and compliance evidence assemble in real time.

The interesting architectural decision was how to get the data to the browser. The MCP tools already had all the aggregation logic — queue status assembly, fleet-wide trust averaging, problem detection — but it was inline in an 800-line `DevtownMcpTools` class. We extracted it into `GovernanceQueryService`, a standalone CDI service. Both the MCP tools and the new REST endpoints delegate to it. One source of truth, two access protocols.

The WebSocket bridge was the other piece that needed thought. Seven CDI event types — case lifecycle, work item state, commitment failures, trust score updates, SLA breaches, merge queue mutations — all observed asynchronously and forwarded as JSON envelopes to connected browser sessions. The message format matches casehub-pages' WebSocket source protocol, so panels subscribe through the data pipeline with no custom wiring.

The frontend itself is casehub-pages via Quinoa — TypeScript DSL, esbuild, single `mvn quarkus:dev` for both Java and TypeScript hot reload. Every panel starts as a DSL component (`table()`, `metric()`, `barChart()`) and only graduates to a custom Web Component if the DSL can't express it. So far nothing has needed promotion.

The design review caught a security hole before it shipped. The static asset permission rule I'd written — `quarkus.http.auth.permission.static.paths=/*` with `policy=permit` — would have opened every path to unauthenticated access at the HTTP level. Quarkus resolves permissions by path specificity, not by declaration order, and `/*` matches everything. The fix was obvious once pointed out: target only the Quinoa-served paths (`/`, `/index.html`, `/dist/*`).

The three demo paths from the Gastown analysis now have a visual surface. When an agent completes a review and its trust score ticks up, that's visible in real time on the reviewer profile page. When an SLA breaches and escalation fires, the triage view shows it happening. The demo stops being "watch curl output" and becomes "watch the system operate."
