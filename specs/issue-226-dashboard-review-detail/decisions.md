## D1: Findings data model and storage

**Choice:** Typed domain records in `devtown-domain`, serialized into case context
**Alternatives:**
- Devtown JPA entities — duplicates data the context already holds, requires migration, works against the blackboard architecture
- Both context + JPA — most complete but unnecessary overhead for pre-release
**Rationale:** Case context is the architectural primitive for adaptive case management. Bindings evaluate it, goals reference it, EventLog audits it, WebSocket pushes it. Storing findings elsewhere would bypass the architecture.
**Trade-offs:** No SQL querying of findings across cases — but EventLog captures every context mutation, so historical analysis is still possible through the engine's audit trail.
**Sources:** GovernanceQueryService.java (existing reviewDetail), CaseHubRuntime.eventLog API, GovernanceEventBridge context.update topic
**Exploration:** quick
**Status:** captured

## D2: Dev-mode agent review infrastructure

**Choice:** Analyze synthetic diffs via pattern-matching
**Alternatives:**
- Pre-canned findings per PR profile — simpler but doesn't exercise the analysis pipeline
- Template + randomization — middle ground but still doesn't validate the full flow
**Rationale:** Dev-mode agents should use the same infrastructure as production agents. Pattern-matching on file paths and patch content produces realistic, varied findings and validates the entire pipeline from diff analysis through to dashboard rendering.
**Trade-offs:** More implementation work than pre-canned. But the pattern-matching logic serves as the reference implementation for real agent integration.
**Sources:** DevModePrDiffService.java, SyntheticPatches, ScenarioResource.java steps
**Exploration:** quick
**Status:** captured

## D3: Event timeline architecture

**Choice:** Backend-assembled rich timeline from EventLog
**Alternatives:**
- Frontend WebSocket assembly — lower latency but loses history on reload and duplicates interpretation logic
- Hybrid backend + WebSocket — best UX but most complex
**Rationale:** GovernanceQueryService.reviewDetail() already queries the EventLog. Extending it to query ALL event types (not just lifecycle), enrich with human-readable summaries, and return a typed timeline gives a single source of truth that works on page load.
**Trade-offs:** No sub-second live updates — but the existing 10-second polling refresh is adequate for a dashboard. WebSocket can be added later for real-time push.
**Sources:** GovernanceQueryService.reviewDetail(), CaseHubRuntime.eventLog(), GovernanceEventBridge (already broadcasting all event types)
**Exploration:** quick
**Status:** captured

## D4: Review detail UI decomposition

**Choice:** Decompose into focused sub-components
**Alternatives:**
- Extend in-place — simpler file structure but component triples in size
- Replace with pages-ui declarative — less code but loses interactive elements
**Rationale:** The detail pane needs routing summary, findings panel, event timeline, and context viewer. Each has distinct data sources and interaction patterns. Focused sub-components are easier to reason about, test, and extend independently.
**Trade-offs:** More files. But each component stays under 150 lines and has a clear single responsibility.
**Sources:** review-workbench.ts (current 240-line monolith), operations-workbench.ts (similar pattern)
**Exploration:** quick
**Status:** captured

## D5: Workbench composition model

**Choice:** Refactor review-workbench to compose existing blocks-ui primitives
**Alternatives:**
- Keep hand-rolled layout — already works but duplicates split-pane logic and misses accessibility (LiveRegionMixin, keyboard nav, responsive collapse)
- Full pages-ui declarative — loses interactive elements
**Rationale:** `blocks-split-workbench` provides resizable master-detail with selection-topic event bus, responsive collapse, keyboard navigation, and persisted divider position. `blocks-timeline` provides a strategy-driven timeline with filter chips, expand/collapse, and pagination. The orchestration-workbench already demonstrates this composition pattern. Review-workbench should follow the same structure.
**Trade-offs:** Dependency on blocks-ui packages (already in the project). The existing hand-rolled CSS is deleted.
**Depends on:** D4 (decomposition into sub-components)
**Sources:** blocks-split-workbench (split-workbench.ts), blocks-timeline (blocks-timeline.ts), orchestration-workbench (composition example)
**Exploration:** quick
**Status:** captured
