# Dashboard Review Detail Enhancements

**Issues:** #226 (routing decisions), #227 (AI review findings), #228 (rich event timeline), #229 (realistic simulation findings)
**Epic:** #222

## Summary

Enrich the PR review detail view to show routing decisions, AI review findings, and a rich event timeline. Make the existing dev-mode reviewer agents produce diff-aware findings so the dashboard has meaningful data to display during simulation.

## Existing Infrastructure (no changes needed)

The agent review pipeline is already fully wired:

- **`ReviewFinding`** (`devtown-domain`) — `Severity` enum (CRITICAL/HIGH/MEDIUM/LOW/INFO), `category`, `filePath`, `LineRange(startLine, endLine)`, `message`, `confidence`
- **`ReviewerAgent`** interface → `ReviewerOutcome.Completed(List<ReviewFinding>)` — for security-review, architecture-review, style-review, test-coverage, performance-analysis
- **`CodeAnalysisAgent`** interface → `CodeAnalysisResult` — separate interface for code-analysis (not ReviewerAgent)
- **`PrReviewCaseHub.adaptReview()`** — serializes findings into `WorkerResult`, stored in EventLog
- **`DevModePrDiffService`** — generates synthetic diffs from `SyntheticPatches`
- **`PrDiffCache`** — caches diffs; `ReviewContext` carries the diff to agents
- **LLM agents** — `LlmSecurityReviewAgent`, `LlmArchitectureReviewAgent`, etc. extend `LlmReviewerAgent` and are already diff-aware (they call external LLMs). No changes needed to LLM agents.

The stubs (`SecurityReviewAgent`, `ArchitectureReviewAgent`, etc.) are the dev-mode fallbacks when LLM agents aren't configured. The `ReviewerAgentRegistry` routes between them based on priority.

## Changes

### 1. Diff-Aware Dev-Mode Agents (#229)

Two distinct interfaces, two distinct change patterns:

#### CodeAnalysisAgent (returns CodeAnalysisResult)

**`CodeAnalysisAgentStub`** — currently returns hardcoded `securitySensitive=false`, empty lists. Change to inspect `context.diff()`:
- Scan file paths for security patterns (`auth/*`, `Security*`, `session/*`, `token/*`) → `securitySensitive=true`
- Detect cross-module changes (files in 3+ different top-level directories) → `architectureCrossing=true`
- Count total lines changed → `scope` classification (small/medium/large)
- Return `flaggedFiles` matching security/architecture patterns
- Return `crossingPoints` (pairs of modules with cross-references)

#### ReviewerAgent stubs (return ReviewerOutcome)

Each stub's `handle(ReviewContext)` method follows the same structure:
1. Get diff from `context.diff()`
2. Filter `diff.files()` for capability-relevant paths
3. Scan patch content for patterns
4. Build `ReviewFinding` list with actual file paths, line numbers from hunk headers
5. Return `ReviewerOutcome.Completed(findings)` or `ReviewerOutcome.Declined` if no relevant files

**`SecurityReviewAgent`** — currently returns one hardcoded finding. Change to:
- Filter for security-relevant files (auth/session/token/RBAC paths)
- Pattern-match patch content for hardcoded credentials, SQL concatenation, missing input validation
- Produce findings with actual file paths and line numbers
- Return `Declined` for PRs with no security-relevant files

**`ArchitectureReviewAgent`** — currently always declines. Change to:
- Analyze for module boundary crossings, new module creation (`pom.xml` changes)
- Produce findings for large PRs (>500 lines) with separation of concerns analysis
- `Declined` only when no architecture-relevant changes detected

**`StyleReviewAgent`** — currently returns one hardcoded finding. Change to:
- Scan for naming inconsistencies, missing Javadoc on public API additions
- Produce file-specific findings

**`TestCoverageReviewAgent`** — same pattern: scan for source files without corresponding test files.

**`PerformanceAnalysisAgent`** — currently returns `ReviewerOutcome.Failed("analysis timed out on large diff")`. Change from simulating failure to producing analysis: scan for N+1 patterns, unbounded collections, missing pagination.

### 2. Fix lineRange Serialization

`PrReviewCaseHub.adaptReview()` (line 112-118) currently drops `lineRange` from the serialized output:

```java
// Current — drops lineRange
.map(f -> Map.of("severity", f.severity().name(), "category", f.category(),
    "filePath", f.filePath(), "message", f.message(), "confidence", f.confidence()))
```

Fix: include `lineRange` in the serialized output:

```java
.map(f -> {
    var m = new java.util.LinkedHashMap<String, Object>();
    m.put("severity", f.severity().name());
    m.put("category", f.category());
    m.put("filePath", f.filePath());
    m.put("message", f.message());
    m.put("confidence", f.confidence());
    if (f.lineRange() != null) {
        m.put("startLine", f.lineRange().startLine());
        m.put("endLine", f.lineRange().endLine());
    }
    return m;
})
```

### 3. Enriched ReviewDetail API (#226/#228)

**Endpoint:** `GET /api/devtown/reviews/{caseId}` (served by `DevtownReviewApi.reviewDetail()`)

**New/expanded records in `GovernanceQueryService`:**

```java
public enum TimelineCategory { LIFECYCLE, ORCHESTRATION, AGENT, WORKITEM, TRUST, CI, SIGNAL }

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
    CodeAnalysisResult featureVector
) {}

public record RoutingDecision(
    String capability,
    String reason,
    double confidence,
    String bindingName
) {}

public record TimelineEvent(
    Instant timestamp,
    TimelineCategory category,
    String eventType,
    String actor,
    String summary,
    JsonNode metadata
) {}
```

Key type decisions:
- `TimelineCategory` is an enum (closed set of 7 values)
- `findings` uses `ReviewFinding` directly (preserves `Severity` enum and `LineRange`)
- `featureVector` uses `CodeAnalysisResult` (typed record, not `Map<String, Object>`)
- `metadata` uses Jackson `ObjectNode` (matches engine EventLog format, not `Map<String, Object>`)

**`reviewDetail()` query changes — actual CaseHubEventType values:**

| EventLog event type | Timeline category | Summary template |
|---|---|---|
| `CASE_STARTED`, `CASE_COMPLETED`, `CASE_FAULTED` | LIFECYCLE | "Case started/completed/faulted" |
| `ORCHESTRATION_STARTED`, `ORCHESTRATION_COMPLETED` | ORCHESTRATION | "Routing started for {capability}" |
| `AGENT_ROUTED` | ORCHESTRATION | "Agent {actorId} selected for {capability}" |
| `AGENT_DISPATCHED`, `AGENT_COMPLETED`, `AGENT_FAILED` | AGENT | "Agent dispatched/completed/failed" |
| `WORKER_EXECUTION_COMPLETED`, `WORKER_EXECUTION_FAILED` | AGENT | "{capability} review completed — {n} findings" |
| `WORKER_OUTCOME_DECLINED` | AGENT | "{capability} review declined — {reason}" |
| `WORK_SUBMITTED`, `WORK_COMPLETED` | WORKITEM | "Work item created/completed" |
| `SIGNAL_RECEIVED`, `CONTEXT_SIGNAL_APPLIED` | SIGNAL | "Signal: {key} = {value}" |
| `GOAL_REACHED` | LIFECYCLE | "Goal reached: {goalName}" |
| `ACTION_GATE_PENDING`, `ACTION_GATE_APPROVED` | WORKITEM | "Human gate pending/approved" |

**Routing decisions:** Extracted from `AGENT_ROUTED` events — the metadata carries capability name, selected agent, routing confidence, and the binding that triggered it.

**Findings:** Extracted from `WORKER_EXECUTION_COMPLETED` events — the `WorkerResult` output map contains the `findings` list. Deserialized back to `ReviewFinding` using the domain type.

**Feature vector:** Extracted from the code-analysis worker's `WORKER_EXECUTION_COMPLETED` event — output contains `securitySensitive`, `architectureCrossing`, `scope`, `flaggedFiles`, `crossingPoints`.

**Breaking change:** The existing `ReviewDetail` grows from 4 to 6 fields, and `List<EventEntry>` → `List<TimelineEvent>`. All call sites must be updated (DevtownReviewApi, GovernanceQueryResolver MCP).

### 4. Frontend Decomposition (#226/#227/#228)

**Component architecture:**

```
devtown-review-workbench (orchestrator)
├── blocks-split-workbench (layout — selection-topic="review")
│   ├── slot="list" → devtown-review-list
│   └── slot="detail" → devtown-review-detail (scrolling pane with sections)
│       ├── PR header + metadata (inline)
│       ├── devtown-routing-summary
│       ├── devtown-findings-panel
│       ├── blocks-timeline (with reviewTimelineStrategy)
│       └── Action buttons (inline)
```

The detail pane is a single scrolling column with sections — not nested splits or tabs. Four sections (routing, findings, timeline, actions) stack vertically in the detail slot.

**Sub-component communication:** Via the pages event bus (`emitPagesEvent`/`onPagesEvent` from `@casehubio/blocks-ui-core`). `blocks-split-workbench` uses `selection-topic="review"` events. Sub-components within the detail pane receive data as properties from `devtown-review-detail` (parent-managed state), not via the event bus — they are presentation components, not independent data fetchers.

**devtown-review-list** — Extracted from current review-workbench. Emits `review:selected` / `review:deselected`.

**devtown-review-detail** — Receives `caseId`, fetches `GET /api/devtown/reviews/{caseId}`, distributes data as properties to children.

**devtown-routing-summary** — Receives `RoutingSummary`. Renders:
- Capability badges with confidence indicators
- Binding name that triggered assignment
- Reason text from code analysis
- Collapsible feature vector (securitySensitive, architectureCrossing, scope, flaggedFiles)

**devtown-findings-panel** — Receives `Map<string, ReviewFinding[]>`. Renders:
- Grouped by capability, each collapsible with count badge
- Each finding: severity badge (CRITICAL=red, HIGH=orange, MEDIUM=amber, LOW=blue, INFO=gray), file:line, message, confidence bar
- Sorted by severity within each group

**reviewTimelineStrategy** — `TimelineStrategy` for `blocks-timeline`:
- `filterCategories`: lifecycle, orchestration, agent, workitem, trust, ci, signal
- `toNodes()`: maps `TimelineEvent` → `TimelineNode` with category-specific icons
- `renderNode()`: category-specific rendering with actor and summary
- `renderDetail()`: click-to-expand showing full event metadata (addresses #228 AC "clicking an event shows its full context")

**Filtering:** `blocks-timeline` filterCategories handle type filtering. For PR-number and actor filtering (#228 AC), add filter controls to `devtown-review-detail` that pass `activeFilters` down.

### Scope Note — Inline Diff Annotation

The findings panel renders a flat list grouped by capability. Inline diff annotation (showing findings anchored to diff hunks, like GitHub PR comments) is a valuable future enhancement but requires a diff viewer component — out of scope for this batch. The `ReviewFinding.LineRange` data is preserved so inline annotation can be added later without API changes.

## Data Flow

```
startReview(prPayload)
  → Engine starts case
  → code-analysis worker → CodeAnalysisAgentStub analyses diff → CONTEXT_SIGNAL_APPLIED
  → routing bindings evaluate → ORCHESTRATION_STARTED / AGENT_ROUTED / AGENT_DISPATCHED
  → review workers fire → SecurityReviewAgent etc. analyse diff
  → WorkerResult(findings) → WORKER_EXECUTION_COMPLETED with output
  → GovernanceEventBridge broadcasts events via WebSocket

reviewDetail(caseId) query:
  EventLog (all event types) → TimelineEvent list
  WORKER_EXECUTION_COMPLETED metadata → ReviewFinding maps (per capability)
  AGENT_ROUTED metadata → RoutingDecision list
  code-analysis WORKER_EXECUTION_COMPLETED output → CodeAnalysisResult

Frontend:
  review:selected → devtown-review-detail fetches /api/devtown/reviews/{caseId}
  → routing-summary renders decisions + feature vector
  → findings-panel renders grouped findings
  → blocks-timeline renders rich event timeline with category filters
```

## Testing

- **Agent tests** — unit test each dev-mode agent: security PR diff → security findings with correct file paths and line numbers; simple rename diff → no security findings. CodeAnalysisAgentStub separately (CodeAnalysisResult assertions).
- **Serialization test** — verify `adaptReview()` preserves lineRange in WorkerResult output.
- **GovernanceQueryServiceTest** — NEW test for `reviewDetail()` (none exists currently): verify enriched response extracts routing decisions from AGENT_ROUTED events, findings from WORKER_EXECUTION_COMPLETED events, and timeline from all event types.
- **Integration** — run full scenario via `ScenarioResource`, verify findings appear in `/api/devtown/reviews/{caseId}` response.
- **Frontend** — manual via `quarkus:dev`: run scenario, verify routing summary, findings panel, and rich timeline render correctly.

## References

- `ReviewFinding.java` (devtown-domain) — existing domain type with Severity enum and LineRange
- `ReviewerAgent.java` / `ReviewerOutcome.java` (review) — existing agent pipeline
- `CodeAnalysisAgent.java` / `CodeAnalysisResult.java` (review) — separate interface for code analysis
- `PrReviewCaseHub.java:94-119` — worker→finding serialization (needs lineRange fix)
- `CaseHubEventType.java` (engine-api) — actual enum values: ORCHESTRATION_STARTED, AGENT_ROUTED, AGENT_DISPATCHED, WORKER_EXECUTION_COMPLETED, CONTEXT_SIGNAL_APPLIED, GOAL_REACHED
- `SecurityReviewAgent.java`, `ArchitectureReviewAgent.java`, `StyleReviewAgent.java`, `TestCoverageReviewAgent.java`, `PerformanceAnalysisAgent.java` — stubs to make diff-aware
- `CodeAnalysisAgentStub.java` — separate interface stub to make diff-aware
- `LlmSecurityReviewAgent.java` etc. — LLM agents, no changes needed
- `GovernanceQueryService.java:318-368` — existing reviewDetail() to extend
- `DevtownReviewApi.java:41` — endpoint at `/api/devtown/reviews/{caseId}`
- `blocks-split-workbench` — layout composition primitive
- `blocks-timeline` — strategy-driven timeline component with renderDetail support
- `orchestration-workbench` — composition pattern example
