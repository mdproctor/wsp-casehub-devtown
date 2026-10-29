## D1: Timeline spine replaces PR list entirely

**Choice:** Timeline spine is the primary navigation — no list view, no toggle
**Alternatives:**
- Keep both with toggle — adds complexity, spine IS the vision
- List as filter with spine detail — hybrid that doesn't commit to either model
**Rationale:** The PR list was a placeholder. The timeline spine shows case lifecycle as decision events, which is the workbench's value proposition over GitHub.
**Trade-offs:** Users cannot scan a flat sortable list of PRs. Filter chips on the spine compensate.
**Sources:** Issue #216 acceptance criteria, pr-cases-workbench.ts current implementation
**Exploration:** quick
**Status:** captured

## D2: Drop Workers and Definitions tabs

**Choice:** Remove both from the tab bar
**Alternatives:**
- Fold Workers into Operations — adds clutter to an already dense view
- Keep Workers as top-level — dev/debug concern, not daily-use
**Rationale:** Workers and Definitions are dev/debug views. The six-tab structure (Review Workbench, Operations, Merge Queue, Reviewers, Contributors, System) is focused on daily use.
**Trade-offs:** Workers and Definitions become inaccessible from the main nav. Can be reached via direct URL if needed.
**Sources:** Issue #216 tab restructure spec
**Exploration:** quick
**Status:** captured

## D3: Keep inline fetch pattern

**Choice:** Each intelligence pane section fetches its own data on click — no shared data service layer
**Alternatives:**
- Introduce data service — adds caching/dedup but overengineers for the current need
**Rationale:** Consistent with the existing codebase. Intelligence pane sections are click-driven, not always-loaded. No redundant fetches in the common case (user clicks one node at a time).
**Trade-offs:** No cross-component cache. Switching back to a previously viewed node re-fetches.
**Sources:** Existing fetch patterns in pr-detail.ts, reviewer-detail.ts, contributor-detail.ts
**Exploration:** quick
**Status:** captured

## D4: All active cases, scrollable spine

**Choice:** Spine shows all active cases ordered by most-recent-event, with filter chips
**Alternatives:**
- Focus mode (curated subset) — limits visibility, loses the "at a glance" value
- Grouped by stage — imposes structure the user may not want
**Rationale:** The spine is compact (just event nodes). Showing all cases lets users spot activity across the pipeline. Filter chips (In Review, Needs Decision, Queued) carry over from the current PR Cases view.
**Trade-offs:** At high volume (50+ active cases), scrolling may be slow. Pagination or virtual scrolling can be added later.
**Sources:** Issue #216 timeline spine description, pr-cases-workbench.ts filter chips
**Exploration:** quick
**Status:** captured

## D5: CBR precedents via @McpDomain resolver

**Choice:** Add a CBR precedents query in a @McpDomain resolver, generating REST/GraphQL/MCP endpoints
**Alternatives:**
- Embed in review detail response — bloats the payload
- Skip CBR for now — defers a key intelligence feature
**Rationale:** @McpDomain is the project pattern for all query endpoints. CbrRetrievalService.findSimilar() already exists — the resolver just wraps it. Intelligence pane fetches precedents on demand when a findings node is clicked.
**Trade-offs:** New endpoint to maintain. Minimal — it delegates to an existing service.
**Sources:** CbrRetrievalService interface, MemoryQueryResolver.similarCases(), GovernanceQueryResolver pattern
**Exploration:** quick
**Status:** captured

## D6: Existing WebSocket for real-time spine updates

**Choice:** Subscribe to existing /api/governance/events WebSocket
**Alternatives:**
- New dedicated filtered channel — lower bandwidth but requires backend work
- Poll with short interval — simpler but not truly real-time
**Rationale:** The WebSocket already pushes case lifecycle events. The spine filters for visible case IDs client-side.
**Trade-offs:** All events pushed regardless of visibility — some wasted bandwidth. Acceptable at current scale.
**Sources:** datasets.ts WebSocket configuration, DevtownGovernanceApi event stream
**Exploration:** quick
**Status:** captured

## D7: Compose spine with agency-timeline

**Choice:** New spine component stacks one agency-timeline per case, reusing event categorization
**Alternatives:**
- Build custom spine — full control but duplicates event logic
- Extend agency-timeline with compact mode — cross-repo mosaic change
**Rationale:** agency-timeline and reviewTimelineStrategy already handle event ordering, categorization (LIFECYCLE, ORCHESTRATION, AGENT, WORKITEM, TRUST, CI, SIGNAL), and styling. The spine wraps each with a case header card and compact layout.
**Trade-offs:** May need compact styling adjustments to agency-timeline rendering. If the compact style diverges too much, the spine may need to override rendering.
**Sources:** review-timeline-strategy.ts, agency-timeline from @casehubio/mosaic-agency-timeline
**Exploration:** quick
**Status:** captured

## D8: Single intelligence pane with conditional render

**Choice:** One intelligence-pane component switches on nodeType, rendering appropriate sections
**Alternatives:**
- Separate pane per node type — more files, more coordination
- Slot-based composition — maximum flexibility but composition logic in parent
**Rationale:** Node types are a known, small set (case header, dispatch, findings, human gate, contributor). One component with conditional rendering is clearer than 5 separate components. Shared sub-components (trust-bar, dimension-grid, routing-summary, findings-panel) keep rendering DRY.
**Trade-offs:** Component grows with node types. Acceptable for 5 types.
**Sources:** pr-detail.ts current rendering, shared/ sub-components
**Exploration:** quick
**Status:** captured

## D9: Extract shared renderers from pr-detail

**Choice:** Pull remaining inline rendering from pr-detail into shared sub-components for reuse
**Alternatives:**
- Build fresh, retire pr-detail — wasteful, rendering logic is solid
- Wrap pr-detail as-is — doesn't achieve node-type-specific layout
**Rationale:** Most rendering pieces (trust-bar, dimension-grid, review-card, badge, status-card) are already in shared/. Remaining inline rendering in pr-detail (routing decisions section, checks-in-progress section, contributor trust section) should be extracted to reusable components. Both the intelligence pane and a simplified pr-detail (if needed elsewhere) use the same pieces.
**Trade-offs:** Refactoring work on pr-detail before building new features. Pays off immediately in the intelligence pane.
**Sources:** pr-detail.ts, shared/trust-bar.ts, shared/dimension-grid.ts, routing-summary.ts, findings-panel.ts
**Exploration:** quick
**Status:** captured
