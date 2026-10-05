# Dashboard Review Detail Enhancements

**Issues:** #226 (routing decisions), #227 (AI review findings), #228 (rich event timeline), #229 (realistic simulation findings)
**Epic:** #222

## Summary

Enrich the PR review detail view to show routing decisions, AI review findings, and a rich event timeline. Update the dev-mode simulation to produce realistic structured review content so the dashboard has meaningful data to display.

## Domain Model

### ReviewFinding

New record in `devtown-domain`:

```java
public record ReviewFinding(
    String capability,
    String file,
    int line,
    String severity,       // "critical", "warning", "info"
    String title,
    String description,
    double confidence
) {}
```

Findings are serialized into the case context under `reviewFindings.<capabilityContextKey>` as a JSON array. The existing `CAPABILITY_CONTEXT_KEYS` map in `GovernanceQueryService` provides the mapping (e.g., `security-review` → `securityReview`, so findings land at `reviewFindings.securityReview`).

CasePlanModel bindings can evaluate findings — e.g., a binding condition can check for critical security findings and trigger human oversight.

### RoutingDecision

New record in `devtown-domain`:

```java
public record RoutingDecision(
    String capability,
    String reason,
    double confidence,
    String bindingName
) {}
```

Routing decisions are written to the case context under `routingDecisions` when bindings fire, capturing WHY each capability was assigned.

## Dev-Mode Agent Analysis (#229)

### DevModeReviewAnalyzer

New `@ApplicationScoped` service in `app/` that takes a `PrDiff` and a capability name, pattern-matches against file paths and patch content, and returns `List<ReviewFinding>`.

Analysis rules per capability:

| Capability | Trigger patterns | Finding types |
|-----------|-----------------|---------------|
| `security-review` | `auth/*`, `Security*`, `RBAC*`, `session/*`, `token/*` | Credential exposure, session fixation, input validation |
| `code-analysis` | Cross-module imports, methods >50 lines, deep nesting | Coupling warnings, complexity findings, dead code |
| `style-review` | Inconsistent naming, missing Javadoc on public API | Naming convention violations, formatting issues |
| `architecture-review` | Module boundary crossings, new module creation, `pom.xml` changes | Separation of concerns, dependency direction violations |
| `test-coverage` | Source files without corresponding test files | Missing test coverage |
| `performance-analysis` | N+1 query patterns, unbounded collections, missing pagination | Performance hotspots |

The analyzer inspects `PrDiff.FileDiff` entries — file paths for capability matching, patch content for specific finding patterns. Each finding includes file path, line number (from patch hunk headers), severity, and a descriptive explanation.

### Scenario Integration

The scenario's review completion steps (cases 4-7 in `ScenarioResource.executeStep`) change from simple `signalReviewSubmitted("approved")` to:

1. Fetch the PR's diff via `DevModePrDiffService`
2. Run `DevModeReviewAnalyzer` for the relevant capability
3. Write findings to the case context via `caseHub.signal(caseId, "reviewFindings.<key>", findings)`
4. Signal review completion with findings summary in the outcome

This ensures every scenario run produces realistic, varied findings that the dashboard can display.

## Enriched ReviewDetail API (#226/#228)

### Expanded Records

```java
public record ReviewDetail(
    UUID caseId,
    PrPayload pr,
    List<TimelineEvent> timeline,
    List<CapabilityStatus> capabilities,
    RoutingSummary routing,
    Map<String, List<ReviewFinding>> findings
) {}

public record RoutingSummary(
    List<RoutingDecision> decisions,
    Map<String, Object> featureVector
) {}

public record TimelineEvent(
    Instant timestamp,
    String category,     // "lifecycle", "binding", "agent", "workitem", "trust", "ci"
    String eventType,
    String actor,
    String summary,
    Map<String, Object> metadata
) {}
```

### Query Changes

`GovernanceQueryService.reviewDetail()` expands to query:

1. **All EventLog event types** — not just lifecycle. Includes `BINDING_EVALUATED`, `WORK_SUBMITTED`, `WORKER_EXECUTION_COMPLETED`, `WORKER_EXECUTION_FAILED`, `WORKER_OUTCOME_DECLINED`, `CONTEXT_UPDATED`, `GOAL_SATISFIED`.
2. **WorkItem state transitions** — query `WorkItemStore` for work items associated with the case, map status transitions to timeline events.
3. **Case context** — read `routingDecisions` for the routing summary, `reviewFindings.*` for each capability's findings.
4. **Human-readable summaries** — each `TimelineEvent` gets a descriptive summary based on its type and metadata (e.g., "Security review binding fired — auth paths detected in 4 files" instead of raw `BINDING_EVALUATED`).
5. **Category classification** — each event is tagged with a category for timeline filtering.

### REST Endpoint

The existing `/api/devtown/governance/review-detail/{caseId}` endpoint returns the expanded `ReviewDetail`. No new endpoints needed — the existing structure just carries more data.

## Frontend Decomposition (#226/#227/#228)

### Component Architecture

```
review-workbench (orchestrator)
├── blocks-split-workbench (layout primitive)
│   ├── slot="list" → devtown-review-list
│   └── slot="detail" → devtown-review-detail
│       ├── devtown-routing-summary
│       ├── devtown-findings-panel
│       └── blocks-timeline (with review strategy)
```

### devtown-review-list

Extracted from the current review-workbench list panel. A focused component that:
- Fetches `/api/devtown/governance/queue-status`
- Renders the PR table with pages-table
- Emits `review:selected` with `{ caseId }` when a row is activated
- Emits `review:deselected` when selection is cleared

### devtown-review-detail

Receives `caseId` prop (set by review-workbench on selection). Fetches the enriched `reviewDetail()` and distributes data to sub-components:
- PR header and metadata (inline)
- `<devtown-routing-summary .routing=${detail.routing}>`
- `<devtown-findings-panel .findings=${detail.findings}>`
- `<blocks-timeline .strategy=${reviewTimelineStrategy} .data=${detail.timeline}>`
- Action buttons (approve, request changes, enqueue) — inline

### devtown-routing-summary

Renders the routing decisions:
- Each assigned capability as a badge with confidence indicator
- Binding name that triggered the assignment
- Reason text (e.g., "auth paths detected: src/auth/*, src/config/Security*")
- Feature vector summary — collapsible section showing the code analysis results

### devtown-findings-panel

Renders AI review findings:
- Grouped by capability, each group collapsible
- Count badge per group (e.g., "Security Review (3)")
- Each finding: severity badge (critical=red, warning=amber, info=blue), title, file:line reference, description, confidence bar
- Findings sorted by severity within each group

### Review Timeline Strategy

A `TimelineStrategy` implementation for `blocks-timeline`:

```typescript
const reviewTimelineStrategy: TimelineStrategy = {
  defaultLayout: 'vertical',
  filterCategories: ['lifecycle', 'binding', 'agent', 'workitem', 'trust', 'ci'],
  toNodes: (events: TimelineEvent[]) => events.map(e => ({
    key: `${e.timestamp}-${e.eventType}`,
    timestamp: e.timestamp,
    category: e.category,
    label: e.summary,
    icon: categoryIcon(e.category),
    metadata: e.metadata,
  })),
  renderNode: (node) => html`...`,  // category-specific rendering
};
```

Filter categories map to visual treatments:
- **lifecycle** — case started, completed, failed (neutral)
- **binding** — binding evaluated, fired (blue/accent)
- **agent** — agent dispatched, completed, declined (green/amber/red)
- **workitem** — work item created, claimed, completed (purple)
- **trust** — trust score updated (teal)
- **ci** — CI status change (gray)

## Data Flow

```
Scenario step
  → DevModeReviewAnalyzer.analyze(diff, capability)
  → caseHub.signal(caseId, "reviewFindings.<key>", findings)
  → Engine writes to case context → EventLog records context update
  → WebSocket broadcasts context.update

Page load / refresh:
  review-workbench → review:selected event
  → devtown-review-detail fetches /api/devtown/governance/review-detail/{caseId}
  → GovernanceQueryService.reviewDetail() queries:
     - EventLog (all event types) → timeline
     - Case context (routingDecisions) → routing summary
     - Case context (reviewFindings.*) → findings
  → Data distributed to sub-components
```

## Testing

- `DevModeReviewAnalyzerTest` — unit tests: each capability produces expected findings for known synthetic diffs. Security PR → security findings, simple rename → no security findings.
- `GovernanceQueryServiceTest` — verify enriched `reviewDetail()` returns routing decisions, findings, and rich timeline events.
- `ScenarioResource` integration — run full scenario, verify findings appear in case context and are queryable via the review detail API.
- Frontend: manual verification via `quarkus:dev` — run scenario, check review detail pane shows routing summary, findings panel, and rich timeline.

## References

- `GovernanceQueryService.java` — existing reviewDetail() to extend
- `DevModePrDiffService.java` — synthetic diff generator (input to review analysis)
- `ScenarioResource.java` — scenario steps to enhance with findings
- `GovernanceEventBridge.java` — WebSocket bridge already broadcasting rich events
- `blocks-split-workbench` — layout composition primitive
- `blocks-timeline` — strategy-driven timeline component
- `orchestration-workbench` — composition pattern example
- `review-workbench.ts` — current component to decompose
- `operations-workbench.ts` — parallel pattern for operations view (also benefits from enriched events)
