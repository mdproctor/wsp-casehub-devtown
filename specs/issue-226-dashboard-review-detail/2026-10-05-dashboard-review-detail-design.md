# Dashboard Review Detail Enhancements

**Issues:** #226 (routing decisions), #227 (AI review findings), #228 (rich event timeline), #229 (realistic simulation findings)
**Epic:** #222

## Summary

Enrich the PR review detail view to show routing decisions, AI review findings, and a rich event timeline. Make the existing dev-mode reviewer agents produce diff-aware findings so the dashboard has meaningful data to display during simulation.

## Existing Infrastructure (no changes needed)

The agent review pipeline is already fully wired:

- **`ReviewFinding`** (`devtown-domain`) — `Severity` enum (CRITICAL/HIGH/MEDIUM/LOW/INFO), `category`, `filePath`, `LineRange`, `message`, `confidence`
- **`ReviewerAgent`** interface → `ReviewerOutcome.Completed(List<ReviewFinding>)`
- **`PrReviewCaseHub.adaptReview()`** — serializes findings into `WorkerResult`, which the engine stores in the EventLog
- **`DevModePrDiffService`** — generates synthetic diffs from `SyntheticPatches`
- **`PrDiffCache`** — caches diffs; `ReviewContext` carries the diff to agents

The engine workers run automatically when `startReview()` is called. The scenario's `approve()` steps signal human review, not agent review — agents execute independently via the engine's worker infrastructure.

## Changes

### 1. Diff-Aware Dev-Mode Agents (#229)

The existing stub agents return hardcoded findings regardless of diff content. Make them inspect `ReviewContext.diff()`:

**`CodeAnalysisAgentStub`** — currently returns `securitySensitive=false` always. Change to:
- Scan file paths for security patterns (`auth/*`, `Security*`, `session/*`, `token/*`) → `securitySensitive=true`
- Detect cross-module changes (files in 3+ different top-level directories) → `architectureCrossing=true`
- Count total lines changed → `scope` classification (small/medium/large)
- Return `flaggedFiles` matching security/architecture patterns
- Return `crossingPoints` (pairs of modules with cross-references)

**`SecurityReviewAgent`** — currently returns one hardcoded finding. Change to:
- Read `ReviewContext.diff()` and filter for security-relevant files
- Pattern-match patch content for common security issues (hardcoded credentials, SQL concatenation, missing input validation, session handling)
- Produce findings with actual file paths and line numbers from patch hunk headers
- Return empty findings for PRs with no security-relevant files

**`ArchitectureReviewAgent`** — currently always declines. Change to:
- Analyze for module boundary crossings, new module creation, dependency changes
- Produce findings for large PRs (>500 lines) with separation of concerns analysis
- Decline only when no architecture-relevant changes detected

**`StyleReviewAgent`** — currently returns one hardcoded finding. Change to:
- Scan patch content for naming inconsistencies (camelCase vs snake_case, inconsistent prefixes)
- Check for missing Javadoc on public API additions
- Produce file-specific findings with actual paths

**`TestCoverageReviewAgent`** and **`PerformanceAnalysisAgent`** — same pattern: inspect the diff, produce findings relevant to the PR content.

Each agent's `handle()` method follows the same structure:
1. Get diff from `context.diff()`
2. Filter `diff.files()` for capability-relevant paths
3. Scan patch content for patterns
4. Build `ReviewFinding` list with actual file paths, line numbers from hunk headers, descriptive messages
5. Return `Completed(findings)` or `Declined` if no relevant files

### 2. Enriched ReviewDetail API (#226/#228)

**Expanded records in `GovernanceQueryService`:**

```java
public record ReviewDetail(
    UUID caseId,
    PrPayload pr,
    List<TimelineEvent> timeline,
    List<CapabilityStatus> capabilities,
    RoutingSummary routing,
    Map<String, List<FindingEntry>> findings
) {}

public record RoutingSummary(
    List<RoutingDecision> decisions,
    Map<String, Object> featureVector
) {}

public record RoutingDecision(
    String capability,
    String reason,
    double confidence,
    String bindingName
) {}

public record TimelineEvent(
    Instant timestamp,
    String category,
    String eventType,
    String actor,
    String summary,
    Map<String, Object> metadata
) {}

public record FindingEntry(
    String severity,
    String category,
    String filePath,
    String message,
    double confidence,
    Integer startLine,
    Integer endLine
) {}
```

**`reviewDetail()` query changes:**

1. **All EventLog event types** — expand from lifecycle-only to include `BINDING_EVALUATED`, `WORK_SUBMITTED`, `WORKER_EXECUTION_COMPLETED`, `WORKER_EXECUTION_FAILED`, `WORKER_OUTCOME_DECLINED`, `CONTEXT_UPDATED`, `GOAL_SATISFIED`
2. **Worker outputs → findings** — extract `WorkerResult` from `WORKER_EXECUTION_COMPLETED` events, read the `findings` list from the output map
3. **Binding evaluations → routing decisions** — extract binding name, capability, condition match reason from `BINDING_EVALUATED` events
4. **Code analysis output → feature vector** — extract `securitySensitive`, `architectureCrossing`, `scope`, `flaggedFiles` from the code-analysis worker's output
5. **Human-readable summaries** — map each event type to descriptive text (e.g., "Security review completed — 3 findings (1 high, 2 medium)" instead of raw `WORKER_EXECUTION_COMPLETED`)
6. **Category classification** — tag events as `lifecycle`, `binding`, `agent`, `workitem`, `trust`, or `ci`

### 3. Frontend Decomposition (#226/#227/#228)

**Component architecture:**

```
review-workbench (orchestrator)
├── blocks-split-workbench (layout — selection-topic="review")
│   ├── slot="list" → devtown-review-list
│   └── slot="detail" → devtown-review-detail
│       ├── PR header + metadata (inline)
│       ├── devtown-routing-summary
│       ├── devtown-findings-panel
│       ├── blocks-timeline (with reviewTimelineStrategy)
│       └── Action buttons (inline)
```

**devtown-review-list** — Extracted from current review-workbench. Renders PR table, emits `review:selected`/`review:deselected` via pages event bus.

**devtown-review-detail** — Receives `caseId`, fetches enriched `/api/devtown/governance/review-detail/{caseId}`, distributes data:
- PR header + metadata: inline rendering
- `.routing` → `<devtown-routing-summary>`
- `.findings` → `<devtown-findings-panel>`
- `.timeline` → `<blocks-timeline .strategy=${reviewTimelineStrategy}>`

**devtown-routing-summary** — Receives `RoutingSummary`. Renders:
- Capability badges with confidence indicators (color-coded by threshold)
- Binding name that triggered assignment
- Reason text from code analysis
- Collapsible feature vector (security-sensitive, architecture-crossing, scope, flagged files)

**devtown-findings-panel** — Receives `Map<string, FindingEntry[]>`. Renders:
- Grouped by capability, each group collapsible with count badge
- Each finding: severity badge (CRITICAL=red, HIGH=orange, MEDIUM=amber, LOW=blue, INFO=gray), file:line, message, confidence bar
- Sorted by severity within each group

**reviewTimelineStrategy** — `TimelineStrategy` for `blocks-timeline`:
- `filterCategories`: lifecycle, binding, agent, workitem, trust, ci
- `toNodes()`: maps `TimelineEvent` → `TimelineNode` with category-specific icons
- `renderNode()`: category-specific rendering with actor and summary

## Data Flow

```
startReview(prPayload)
  → Engine starts case
  → code-analysis worker runs → CodeAnalysisAgentStub analyses diff → routing bindings evaluate
  → review workers fire per binding → SecurityReviewAgent / StyleReviewAgent / etc. analyse diff
  → WorkerResult(findings) written to EventLog
  → GovernanceEventBridge broadcasts events via WebSocket

reviewDetail(caseId) query:
  EventLog → all event types → TimelineEvent list
  WORKER_EXECUTION_COMPLETED metadata → FindingEntry maps
  BINDING_EVALUATED metadata → RoutingDecision list
  code-analysis output → feature vector

Frontend:
  review:selected → devtown-review-detail fetches reviewDetail(caseId)
  → routing-summary renders routing decisions
  → findings-panel renders grouped findings
  → blocks-timeline renders rich event timeline with filters
```

## Testing

- **Agent tests** — unit test each dev-mode agent: security PR diff → security findings, simple rename diff → no security findings, architecture PR → architecture findings
- **GovernanceQueryServiceTest** — verify enriched `reviewDetail()` extracts routing decisions, findings, and rich timeline from EventLog
- **Integration** — run full scenario via `ScenarioResource`, verify findings appear in review detail API response
- **Frontend** — manual via `quarkus:dev`: run scenario, verify routing summary, findings panel, and rich timeline render correctly

## References

- `ReviewFinding.java` (devtown-domain:1) — existing domain type
- `ReviewerAgent.java` / `ReviewerOutcome.java` (review) — existing agent pipeline
- `PrReviewCaseHub.java:94-119` — existing worker→finding serialization
- `SecurityReviewAgent.java`, `ArchitectureReviewAgent.java`, `StyleReviewAgent.java` — stubs to make diff-aware
- `CodeAnalysisAgentStub.java` — stub to make diff-aware
- `GovernanceQueryService.java:318-368` — existing reviewDetail() to extend
- `DevModePrDiffService.java` — synthetic diff generator
- `blocks-split-workbench` — layout composition primitive
- `blocks-timeline` — strategy-driven timeline component
- `orchestration-workbench` — composition pattern example
