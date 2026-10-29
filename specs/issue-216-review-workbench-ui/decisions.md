## D1: Timeline spine as primary view with table toggle

**Choice:** Timeline spine is the default primary navigation; a dense table mode is available via toggle
**Alternatives:**
- Spine-only (no table) — drops breadth scanning, ops leads lose sortable multi-column view during incidents
- Table-only (status quo) — doesn't expose decision flow, which is the workbench's value proposition
- List as filter with spine detail — hybrid that doesn't commit to either model
**Rationale:** The spine and table serve different cognitive modes. The spine shows decision flow (events, dispatches, findings, gates) — the workbench's value proposition over GitHub. The table shows PR metadata for scanning and sorting (author, lines, status, wait time) — necessary for ops scenarios like scanning 30+ cases during an incident. Both views share the same data source and selection mechanism; clicking a row in either mode opens the intelligence pane. The `pages-split-workbench` already supports different list-slot content.
**Trade-offs:** Two view modes to maintain. Mitigated by shared data source and shared selection/intelligence-pane interaction.
**Sources:** Issue #216 acceptance criteria, pr-cases-workbench.ts (649-line production component with unified data model, filter chips, vitals, problems banner, batch display)
**Exploration:** quick
**Status:** revised (R1-02: "placeholder" claim was wrong — pr-cases-workbench is production code; dual-mode view addresses ops scanning needs the reviewer surfaced)

## D2: Drop Workers and Definitions tabs

**Choice:** Remove both from the top-level tab bar
**Alternatives:**
- Keep Workers as top-level — dev/debug concern, not daily-use
- Keep Definitions as top-level — static config, rarely accessed
**Rationale:** Workers and Definitions are dev/debug views. Removing them focuses the tab bar on daily-use views. Session monitoring folds into the Operations tab as a collapsible section (see D10), so ops access is preserved without a top-level tab. Definitions remains accessible via direct URL.
**Trade-offs:** Definitions becomes URL-only. Acceptable — it's a static configuration view accessed during setup, not operations.
**Sources:** Issue #216 tab restructure spec, review.ts current tab bar
**Exploration:** quick
**Status:** revised (R1-03: split bundled decision — tab structure now in D10; Workers session monitoring folds into Operations instead of URL-only)

## D3: Inline fetch with lightweight dedup cache

**Choice:** Each intelligence pane section fetches its own data on click, with a lightweight `Map<string, {promise: Promise, timestamp: number}>` dedup cache (30-second TTL)
**Alternatives:**
- Shared data service layer — adds caching/dedup but introduces architectural abstraction beyond what's needed
- Pure inline fetch (no cache) — re-fetches on every click, causing flicker and unnecessary network calls during natural click-around
**Rationale:** The intelligence pane is more data-intensive than pr-detail — clicking between dispatch, case header, and findings nodes causes overlapping data requests (e.g. contributor trust appears in both case header and dispatch views). A ~20-line dedup cache eliminates duplicate concurrent requests and caches responses for 30 seconds. This is a mechanical optimisation, not an architectural abstraction.
**Trade-offs:** 30-second stale window. Acceptable for trust scores and profile data that change on review-completion timescales, not click timescales.
**Sources:** Existing fetch patterns in pr-detail.ts, intelligence pane data requirements per node type
**Exploration:** quick
**Status:** revised (R1-04: reviewer demonstrated click-around fetch density; dedup cache is the right response without overengineering)

## D4: All active cases, scrollable spine (virtual scrolling deferred)

**Choice:** Spine shows all active cases ordered by most-recent-event, with filter chips. Virtual scrolling deferred.
**Alternatives:**
- Focus mode (curated subset) — limits visibility, loses the "at a glance" value
- Grouped by stage — imposes structure the user may not want
- Virtual scrolling day-1 — unnecessary cost given D7's revised flat-list architecture
**Rationale:** The spine is compact (just event nodes). Showing all cases lets users spot activity across the pipeline. Filter chips (In Review, Needs Decision, Queued) carry over from the current PR Cases view. Virtual scrolling deferral is safe because D7 (revised) uses flat `<div>` elements — virtualisation is a standard windowing problem over uniform-ish height items, not heterogeneous component subtree recreation.
**Trade-offs:** At high volume (50+ active cases), scrolling may be slow. Virtual scrolling can be added as a standard windowing solution.
**Sources:** Issue #216 timeline spine description, pr-cases-workbench.ts filter chips
**Exploration:** quick
**Status:** revised (R1-05: deferral contingent on D7's revised flat-list architecture; dependency noted)

## D5: CBR precedents data shape adequacy

**Choice:** Use existing `MemoryQueryResolver.similarCases()` endpoint; compute aggregate statistics (confirmation rate per finding type) client-side from the returned precedent list
**Alternatives:**
- Extend `SimilarCaseResult` with server-side aggregate statistics — premature for small precedent pools (5-10 results)
- New dedicated intelligence endpoint — endpoint already exists, duplication is unnecessary
**Rationale:** `MemoryQueryResolver.similarCases()` already exists (line 145), carries `@McpDomain(value = "devtown/model")`, and wraps `CbrRetrievalService.findSimilar()`. The current `SimilarCaseResult` returns per-case capability outcomes (outcome, detail, duration) but not aggregate statistics like "92% confirmation rate." With small precedent pools (5-10 results), computing confirmation rate client-side from `capabilityOutcomes` is trivial. If the precedent pool grows, server-side aggregation becomes a separate decision.
**Trade-offs:** Client-side computation with small data sets. Negligible performance impact.
**Sources:** MemoryQueryResolver.java (existing similarCases query), Precedent.java, CapabilityOutcome.java (outcome + detail + duration — sufficient for client-side rate computation)
**Exploration:** quick
**Status:** revised (R1-06: reframed from "create endpoint" to "data shape adequacy" — endpoint already exists)

## D6: Existing WebSocket via triggerUrl + triggerFilter pattern

**Choice:** Spine data set uses the `triggerUrl + triggerFilter` pattern from `datasets.ts` — WebSocket events trigger selective REST re-fetches
**Alternatives:**
- Manual client-side WebSocket subscription with case-ID filtering — inconsistent with codebase, more code, manual reconnection handling
- New dedicated filtered channel — lower bandwidth but requires backend work for marginal benefit
- Poll with short interval — simpler but not truly real-time
**Rationale:** `datasets.ts` (lines 43-55) already uses `triggerUrl + triggerFilter` for plan-items, goal-status, and case-context datasets. WS events trigger full REST re-fetch rather than direct DOM updates from WebSocket messages. This is simpler (no manual case-ID filtering), consistent with the codebase, naturally handles reconnection (pages-data framework manages it), and ensures the spine always gets complete, consistent state from REST.
**Trade-offs:** Full re-fetch on each trigger event. Acceptable at current scale — the spine data endpoint returns a compact list.
**Sources:** datasets.ts (triggerUrl + triggerFilter pattern), DevtownGovernanceApi event stream
**Exploration:** quick
**Status:** revised (R1-07: aligned with existing codebase pattern instead of describing a manual WebSocket approach)

## D7: Custom review-spine component with flat list rendering

**Choice:** New `<review-spine>` component renders a flat `<div>` list — case headers followed by event divs — reusing `reviewTimelineStrategy`'s data model (event categories, icon mapping, rendering functions) as plain imports
**Alternatives:**
- One `agency-timeline` per case — 20 cases = 20 independent component subtrees with shadow roots, filter state, pagination logic, detail expansion, all unused by the spine
- Extend agency-timeline with compact mode — cross-repo mosaic change for a concern specific to devtown's spine
- Build custom spine from scratch — duplicates event categorisation and icon mapping already in reviewTimelineStrategy
**Rationale:** The spine's visual model is a flat list grouped by case: case header + indented event nodes. This is NOT a full timeline widget per case. `agency-timeline` (extending `PagesEventTimeline`) provides per-case filter categories, pagination, detail expansion, and endpoint binding — all features the spine doesn't need. The spine reuses `reviewTimelineStrategy`'s *data model* (event categories, icon mapping), not its *component tree*. Extract the strategy's rendering logic into shared functions; compose them differently in the spine (flat list) vs the intelligence pane detail view (full timeline). `agency-timeline` with `reviewTimelineStrategy` remains the right choice for the intelligence pane's detail view — exactly as in `pr-detail.ts` line 358.
**Trade-offs:** New component instead of reusing an existing one. The component is simpler and lighter — the trade-off is in the right direction.
**Sources:** review-timeline-strategy.ts (category icons, status mapping, toNodes/renderNode functions), pr-detail.ts line 358 (agency-timeline usage in detail view)
**Exploration:** quick
**Status:** revised (R1-08: reviewer demonstrated that N timeline instances add unused features and prevent virtualisation; flat-list rendering is the correct architecture)

## D8: Single intelligence pane with conditional render

**Choice:** One intelligence-pane component switches on nodeType, rendering appropriate sections
**Alternatives:**
- Separate pane per node type — more files, more coordination
- Slot-based composition — maximum flexibility but composition logic in parent
- Registry/SPI pattern — appropriate for unbounded extension points, not for 5-7 known internal types
**Rationale:** Node types are a known, small set (case header, dispatch, findings, human gate, contributor, CI, lifecycle). One component with conditional rendering is clearer than 5-7 separate components. Shared sub-components (trust-bar, dimension-grid, routing-summary, findings-panel) keep rendering DRY. The exact node type list will be enumerated during spec writing.
**Trade-offs:** Component grows with node types. Acceptable for 5-7 types.
**Sources:** pr-detail.ts current rendering, shared/ sub-components
**Exploration:** quick
**Status:** captured

## D9: Compose intelligence pane from existing shared primitives

**Choice:** Intelligence pane composes existing shared sub-components (trust-bar, dimension-grid, badge, review-card, status-card, routing-summary, findings-panel) for its own node-type-specific layouts. No higher-level section extraction from pr-detail.
**Alternatives:**
- Extract full "contributor trust section" from pr-detail — wrong abstraction (pr-detail shows PR author trust, intelligence pane dispatch shows reviewer trust for specific capability — different data shapes despite shared primitives)
- Build all rendering fresh — wasteful, the shared primitives are already the right reuse level
**Rationale:** Verified: most rendering in pr-detail is ALREADY extracted to shared components (renderWorkItemAlert, renderBadge, renderTrustBar, renderDimensionGrid, renderReviewCard, devtown-routing-summary, devtown-findings-panel). What remains inline is: 12 lines of checks-in-progress rendering (trivial, would be MORE complex as a separate component) and ~30 lines of contributor trust narrative (mixes badge, trust bar, dimension grid, recent activity — but the surrounding narrative and data context differs between pr-detail and intelligence pane). The shared primitives ARE the right reuse level.
**Trade-offs:** Some rendering logic is duplicated at the composition level (assembling shared primitives into layouts). Acceptable — layout composition is trivial and differs per consumer.
**Sources:** pr-detail.ts render method (verified: 7 of 9 rendering sections are already shared), shared/ sub-components
**Exploration:** quick
**Status:** revised (R1-10: narrowed scope — existing shared primitives are sufficient, no higher-level extraction needed)

## D10: Operations vs System tab distinction

**Choice:** Operations = live situational awareness (problems, active reviews, event stream, session health as collapsible section). System = configuration and calibration (SLA calibration, merge queue metrics, fleet configuration).
**Alternatives:**
- Merge Operations into System — loses the ops-focused "what's happening now" view in a configuration-heavy tab
- Keep Operations and System separate with current overlapping content — duplicate vitals and problems displays cause confusion
- Three-tab split (Operations, Metrics, Configuration) — over-segments at current UI complexity
**Rationale:** `operations-workbench.ts` shows vitals, problems, active reviews, and event stream — all "what's happening now" content. `system-workbench.ts` shows system health cards, merge queue metrics, problems, and SLA calibration — mixing live data with configuration. The distinction should be clean: Operations is the incident-response view (problems, activity, sessions). System is the tuning view (SLA calibration, merge queue configuration, fleet settings). Problems appear in Operations (actionable) but not System (which shows calibration data, not incidents). Session monitoring from Workers folds into Operations as a collapsible section.
**Trade-offs:** Requires refactoring system-workbench.ts to remove problems display and active data. Worth it for conceptual clarity.
**Sources:** operations-workbench.ts, system-workbench.ts, review.ts tab bar
**Exploration:** surfaced by review (R1-03)
**Status:** captured

## D11: pr-detail coexists with intelligence pane, narrowed to queue/triage/batch modes

**Choice:** pr-detail retains queue, triage, and batch rendering modes. The intelligence pane handles review-mode detail with node-type-specific rendering. The split workbench's detail slot switches between intelligence pane (review case selected — from either spine or table mode per D1) and pr-detail (queue/triage/batch context active).
**Alternatives:**
- Retire pr-detail entirely — forces replication of queue/triage/batch rendering in the intelligence pane, which has different concerns
- Keep pr-detail for all modes — defeats the purpose of the intelligence pane for review detail
- Merge everything into intelligence pane with mode switching — creates a god-component handling both review intelligence and operational context
**Rationale:** `pr-detail.ts` (lines 63-70) serves three distinct modes: review detail, queue context (priority lane, trust score, wait time, dependencies), and triage context (decision type, candidate group, escalation stage). The intelligence pane replaces review-mode detail with richer, node-type-specific views. Queue and triage modes are operational views with different concerns — they show queue position and decision urgency, not review intelligence. Keeping pr-detail for these modes avoids scope creep in the intelligence pane.
**Trade-offs:** Two detail components in the detail slot, requiring mode-switching logic. Straightforward — the workbench already knows whether the selection is a review case (from spine or table) vs queue/triage entry.
**Sources:** pr-detail.ts mode/stage/queueContext/triageContext properties, pr-cases-workbench.ts detail slot rendering
**Exploration:** surfaced by review (R1-11)
**Status:** captured

## D12: Parallel client-side fetches for intelligence pane (no backend aggregation)

**Choice:** Intelligence pane fires parallel REST fetches per node type — no backend aggregation endpoint
**Alternatives:**
- Aggregated endpoint `GET /api/.../intelligence/{caseId}?nodeType={type}&nodeId={id}` — creates a god-endpoint coupling frontend display concerns to backend orchestration; union response type changes shape per node type
- Sequential fetches — unnecessary, browsers support 6+ parallel requests per origin
**Rationale:** Node types have diverse data requirements: case header needs ContributorDetail + routing, dispatch needs trust + agent profile, findings needs CBR precedents. An aggregated endpoint requires switch-on-nodeType logic in the backend, returning a union type — complexity in the wrong place. The frontend already knows what each node type needs. With D3's dedup cache, parallel fetches are fast (wall-clock ~300ms for first click, instant on re-click within 30s). No waterfall risk — all fetches fire simultaneously.
**Trade-offs:** Multiple HTTP calls per node click (3-5 parallel). Acceptable — each is small, and the dedup cache eliminates most re-fetches.
**Sources:** Intelligence pane data requirements per node type, D3 dedup cache
**Exploration:** surfaced by review (R1-12)
**Status:** captured

## D13: Dedicated spine node strategy for synthesised summaries

**Choice:** Spine uses a dedicated `spineNodeStrategy` that synthesises human-readable summaries from event + case context + routing data, distinct from `reviewTimelineStrategy`
**Alternatives:**
- Reuse `reviewTimelineStrategy` directly — its `toNodes()` does simple field mapping (summary → label), which doesn't produce the synthesised summaries the spine needs
- Extend reviewTimelineStrategy with a mode flag — conflates two different rendering intents in one strategy
**Rationale:** Spine nodes like "Code analysis: security-sensitive" and "Security review → agent-3" are synthesised summaries combining event type + capability name + routing decision + assigned actor. The current `reviewTimelineStrategy.toNodes()` (lines 85-95) does 1:1 field mapping — it maps `e.summary` to `node.label`. The spine needs a richer mapping: event + context values + routing decisions → compact, informative label with actor and outcome. This is a distinct strategy from `reviewTimelineStrategy`, which preserves full event detail for the intelligence pane's timeline view.
**Trade-offs:** Two strategies sharing the same underlying event data. The category icons and status mapping are shared; the summary synthesis is strategy-specific.
**Sources:** review-timeline-strategy.ts (toNodes field mapping), issue #216 spine mockup node labels
**Exploration:** surfaced by review (R1-13)
**Status:** captured

## D14: Smart default expand — expand active, collapse done

**Choice:** Cases with in-progress or attention-needed nodes expand automatically; completed/approved cases stay collapsed
**Alternatives:**
- All collapsed by default — scannable but hides the decision flow that is the workbench's value proposition; requires click to see anything
- All expanded by default — works for 3-5 cases but becomes unwieldy at 10+; wastes space on completed cases
**Rationale:** Focuses attention where it matters. Active cases show their full event timeline; completed cases compress to a single summary row with status dots. The spine naturally adapts to volume — with few active cases, most are expanded; as volume grows, completed cases collapse and the view stays manageable.
**Trade-offs:** Expand logic needs a clear definition of "active" (has in-progress, reviewing, or pending-decision nodes). Edge case: a case with all nodes complete but not yet merged — collapsed even though it might need attention. Mitigated by the "Needs Decision" filter chip.
**Depends on:** D7 (flat-list spine architecture supports per-case expand state)
**Sources:** Visual companion mockup (spine-collapse-expand.html), user feedback
**Exploration:** quick
**Status:** captured
