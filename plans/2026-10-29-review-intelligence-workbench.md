# Review Intelligence Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #216 — UI tab coherence — differentiate Operations vs Reviews, add triage actions
**Issue group:** #216

**Goal:** Replace the PR Cases tab with a Review Intelligence Workbench featuring a timeline spine, intelligence pane, and restructured tab bar.

**Architecture:** Timeline spine renders cases as flat div lists with collapse/expand and smart defaults. Intelligence pane switches on node type to show contextual data (contributor, routing, findings, human gate). Tab bar restructured to 6 daily-use tabs. Two targeted backend additions: CBR similar-cases REST endpoint and routing entryId enrichment.

**Tech Stack:** Lit 3.2 (web components), TypeScript, esbuild, Java 21 (Quarkus 3.39.3), `@casehubio/pages-*` framework, `@McpDomain` annotations

## Global Constraints

- Java 21 source on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Frontend typecheck: `npm run typecheck` from `app/src/main/webui/`
- Use `mvn` not `./mvnw`
- All commits must reference #216: `Refs #216`
- Use `ide_insert_member` / `ide_replace_member` for Java code edits, `ide_refactor_rename` for renames
- Use `@McpDomain` / `@PlatformQuery` / `@RestPath` annotations for new endpoints (auto-generates REST/GraphQL/MCP)

---

## Batch 1: Backend Foundation

### Task 1: CBR similar-cases REST endpoint

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/api/DevtownReviewApi.java:27-88`
- Test: `app/src/test/java/io/casehub/devtown/app/api/DevtownReviewApiCbrTest.java`

**Interfaces:**
- Consumes: `GovernanceQueryService.reviewDetail(UUID)` → `ReviewDetail` (`.pr()` for PrPayload), `CbrRetrievalService.findSimilar(PrFeatureVector, String, String)` → `List<Precedent>`
- Produces: `GET /api/devtown/reviews/{caseId}/similar-cases` → `List<Precedent>` (consumed by intelligence pane Task 5)

- [ ] **Step 1: Write the failing test**

Create `app/src/test/java/io/casehub/devtown/app/api/DevtownReviewApiCbrTest.java`:

```java
package io.casehub.devtown.app.api;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.hamcrest.Matchers.is;

@QuarkusTest
class DevtownReviewApiCbrTest {

    @Test
    void similarCases_returnsEmptyListWhenNoPrecedents() {
        UUID caseId = UUID.randomUUID();
        RestAssured.given()
            .when().get("/api/devtown/reviews/" + caseId + "/similar-cases")
            .then()
            .statusCode(200)
            .body("size()", is(0));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownReviewApiCbrTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — endpoint does not exist (404)

- [ ] **Step 3: Add CbrRetrievalService injection and endpoint to DevtownReviewApi**

Use `ide_insert_member` to add after the `feedbackService` field (line 31):

```java
@Inject CbrRetrievalService cbrService;
```

Add import: `import io.casehub.devtown.review.CbrRetrievalService;`

Use `ide_insert_member` to add after the `contributorDetail` method (after line 73):

```java
@PlatformQuery("Find similar cases by CBR")
@RestPath("/{caseId}/similar-cases")
public java.util.List<io.casehub.devtown.domain.cbr.Precedent> similarCases(@PathParam UUID caseId) {
    var detail = queryService.reviewDetail(caseId);
    var pr = detail.pr();
    var vector = io.casehub.devtown.domain.cbr.PrFeatureVector.from(
        pr.repository(), pr.prNumber(), pr.contributor(),
        pr.linesChanged(), pr.changedPaths() != null ? pr.changedPaths() : java.util.List.of()
    );
    return cbrService.findSimilar(vector, pr.repository(), "default");
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownReviewApiCbrTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (empty list from in-memory CBR store)

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/api/DevtownReviewApi.java app/src/test/java/io/casehub/devtown/app/api/DevtownReviewApiCbrTest.java
git commit -m "feat(#216): add CBR similar-cases REST endpoint to DevtownReviewApi

Refs #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: RoutingDecision entryId enrichment

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java:122,378-389`
- Test: `app/src/test/java/io/casehub/devtown/app/governance/RoutingDecisionEntryIdTest.java`

**Interfaces:**
- Consumes: `CaseLedgerEntryRepository.findWorkerDecisionsByCaseId(UUID)` → `List<WorkerDecisionEntry>` (each has `capabilityTag`, `id`)
- Produces: `RoutingDecision(String capability, String reason, double confidence, String bindingName, UUID entryId)` — `entryId` is null for in-progress dispatches (consumed by intelligence pane dispatch view, Task 5)

- [ ] **Step 1: Write the failing test**

Create `app/src/test/java/io/casehub/devtown/app/governance/RoutingDecisionEntryIdTest.java`:

```java
package io.casehub.devtown.app.governance;

import org.junit.jupiter.api.Test;

import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class RoutingDecisionEntryIdTest {

    @Test
    void routingDecision_includesEntryId() {
        UUID id = UUID.randomUUID();
        var rd = new GovernanceQueryService.RoutingDecision(
            "security-review", "trust-weighted", 0.85, "SecurityBinding", id
        );
        assertEquals(id, rd.entryId());
        assertEquals("security-review", rd.capability());
    }

    @Test
    void routingDecision_entryIdNullForInProgress() {
        var rd = new GovernanceQueryService.RoutingDecision(
            "code-analysis", "direct", 1.0, "AnalysisBinding", null
        );
        assertNull(rd.entryId());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=RoutingDecisionEntryIdTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — RoutingDecision constructor doesn't accept 5 args

- [ ] **Step 3: Update RoutingDecision record**

Use `ide_replace_member` on `GovernanceQueryService.RoutingDecision` (line 122):

```java
public record RoutingDecision(String capability, String reason, double confidence, String bindingName, UUID entryId) {}
```

- [ ] **Step 4: Update reviewDetail() routing decision extraction**

Use `ide_replace_member` on the routing decision extraction block in `reviewDetail()` (lines 378-389). Replace the stream that builds `routingDecisions`:

```java
List<RoutingDecision> routingDecisions;
{
    var rawDecisions = allEvents.stream()
        .filter(e -> e.eventType() == CaseHubEventType.AGENT_ROUTED)
        .map(e -> {
            var meta = e.metadata();
            return new RoutingDecision(
                meta != null && meta.has("capabilityName") ? meta.get("capabilityName").asText() : "unknown",
                meta != null && meta.has("reason") ? meta.get("reason").asText() : "",
                meta != null && meta.has("confidence") ? meta.get("confidence").asDouble(0.0) : 0.0,
                meta != null && meta.has("bindingName") ? meta.get("bindingName").asText() : "",
                null
            );
        })
        .toList();

    Map<String, UUID> entryIdByCapability = new HashMap<>();
    try {
        var entries = caseLedgerEntryRepository.findWorkerDecisionsByCaseId(caseId);
        for (var entry : entries) {
            entryIdByCapability.put(entry.capabilityTag, entry.id);
        }
    } catch (Exception ignored) {}

    routingDecisions = rawDecisions.stream()
        .map(rd -> new RoutingDecision(
            rd.capability(), rd.reason(), rd.confidence(), rd.bindingName(),
            entryIdByCapability.get(rd.capability())
        ))
        .toList();
}
```

Add `@Inject CaseLedgerEntryRepository caseLedgerEntryRepository;` field if not already present.

Add import: `import io.casehub.ledger.repository.CaseLedgerEntryRepository;`

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=RoutingDecisionEntryIdTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Run full build to check compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS (all existing callers of RoutingDecision updated if needed — check compilation errors for callers that construct RoutingDecision without the 5th arg and add `null` as the entryId)

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java app/src/test/java/io/casehub/devtown/app/governance/RoutingDecisionEntryIdTest.java
git commit -m "feat(#216): enrich RoutingDecision with WorkerDecisionEntry entryId

Refs #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Spine + Strategy Components

### Task 3: Spine node strategy module

**Files:**
- Modify: `app/src/main/webui/src/components/review-timeline-strategy.ts:13-31` (export constants)
- Create: `app/src/main/webui/src/components/spine-node-strategy.ts`
- Test: `app/src/main/webui/src/components/__tests__/spine-node-strategy.test.ts`

**Interfaces:**
- Consumes: `CATEGORY_ICONS`, `CATEGORY_STATUS` from `review-timeline-strategy.ts` (now exported)
- Produces: `SpineNode` type and `synthesizeSpineNodes(events: TimelineEvent[]): SpineNode[]` function (consumed by review-spine, Task 4)

- [ ] **Step 1: Export shared constants from review-timeline-strategy.ts**

In `review-timeline-strategy.ts`, change lines 13 and 23 from `const` to `export const`:

```typescript
export const CATEGORY_ICONS: Record<string, string> = {
```

```typescript
export const CATEGORY_STATUS: Record<string, 'completed' | 'active' | 'pending'> = {
```

Also export the `TimelineEvent` interface (line 4):

```typescript
export interface TimelineEvent {
```

- [ ] **Step 2: Write the failing test**

Create `app/src/main/webui/src/components/__tests__/spine-node-strategy.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { synthesizeSpineNodes, type SpineNode } from '../spine-node-strategy.js';

describe('spine-node-strategy', () => {
  it('synthesizes label for AGENT_ROUTED event', () => {
    const events = [{
      timestamp: '2026-10-29T10:00:00Z',
      category: 'AGENT',
      eventType: 'AGENT_ROUTED',
      actor: 'agent-3',
      summary: 'Worker dispatched',
      metadata: { capabilityName: 'security-review', reason: 'trust-weighted', confidence: 0.85 },
    }];
    const nodes = synthesizeSpineNodes(events);
    expect(nodes).toHaveLength(1);
    expect(nodes[0].label).toBe('Security review → agent-3');
    expect(nodes[0].icon).toBe('▶');
    expect(nodes[0].status).toBe('active');
    expect(nodes[0].nodeType).toBe('dispatch');
  });

  it('synthesizes label for completed review with findings', () => {
    const events = [{
      timestamp: '2026-10-29T10:01:00Z',
      category: 'AGENT',
      eventType: 'WORKER_EXECUTION_COMPLETED',
      actor: 'agent-3',
      summary: 'Review complete',
      metadata: { capabilityName: 'security-review', outcome: 'FINDINGS', findingCount: 2 },
    }];
    const nodes = synthesizeSpineNodes(events);
    expect(nodes[0].label).toBe('Security review: 2 findings');
    expect(nodes[0].icon).toBe('▶');
    expect(nodes[0].status).toBe('completed');
    expect(nodes[0].nodeType).toBe('findings');
  });

  it('synthesizes label for APPROVED review', () => {
    const events = [{
      timestamp: '2026-10-29T10:01:00Z',
      category: 'AGENT',
      eventType: 'WORKER_EXECUTION_COMPLETED',
      actor: 'agent-1',
      summary: 'Review complete',
      metadata: { capabilityName: 'style-review', outcome: 'APPROVED' },
    }];
    const nodes = synthesizeSpineNodes(events);
    expect(nodes[0].label).toBe('Style review: APPROVED');
    expect(nodes[0].nodeType).toBe('lifecycle');
  });

  it('synthesizes label for human gate', () => {
    const events = [{
      timestamp: '2026-10-29T10:02:00Z',
      category: 'WORKITEM',
      eventType: 'WORKITEM_CREATED',
      actor: 'system',
      summary: 'Human decision required',
      metadata: { decisionType: 'pr-approval' },
    }];
    const nodes = synthesizeSpineNodes(events);
    expect(nodes[0].label).toBe('Human gate');
    expect(nodes[0].icon).toBe('■');
    expect(nodes[0].status).toBe('pending');
    expect(nodes[0].nodeType).toBe('human-gate');
  });

  it('synthesizes label for case opened', () => {
    const events = [{
      timestamp: '2026-10-29T10:00:00Z',
      category: 'LIFECYCLE',
      eventType: 'CASE_CREATED',
      actor: 'system',
      summary: 'Case created',
      metadata: null,
    }];
    const nodes = synthesizeSpineNodes(events);
    expect(nodes[0].label).toBe('Opened');
    expect(nodes[0].icon).toBe('●');
    expect(nodes[0].nodeType).toBe('lifecycle');
  });

  it('synthesizes label for code analysis', () => {
    const events = [{
      timestamp: '2026-10-29T10:00:30Z',
      category: 'AGENT',
      eventType: 'WORKER_EXECUTION_COMPLETED',
      actor: 'system',
      summary: 'Code analysis complete',
      metadata: { capabilityName: 'code-analysis', securitySensitive: true },
    }];
    const nodes = synthesizeSpineNodes(events);
    expect(nodes[0].label).toBe('Code analysis: security-sensitive');
    expect(nodes[0].nodeType).toBe('lifecycle');
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `npm test -- --run src/components/__tests__/spine-node-strategy.test.ts` from `app/src/main/webui/`
Expected: FAIL — module not found

- [ ] **Step 4: Create spine-node-strategy.ts**

Create `app/src/main/webui/src/components/spine-node-strategy.ts`:

```typescript
import { CATEGORY_ICONS, CATEGORY_STATUS, type TimelineEvent } from './review-timeline-strategy.js';

export interface SpineNode {
  key: string;
  label: string;
  icon: string;
  status: 'completed' | 'active' | 'pending';
  timestamp: string;
  nodeType: 'case-header' | 'dispatch' | 'findings' | 'human-gate' | 'lifecycle';
  nodeData: unknown;
}

function formatCapability(cap: string): string {
  return cap.replace(/-/g, ' ').replace(/\b\w/g, c => c.toUpperCase()).replace(/^(\w)/, c => c.toUpperCase());
}

function parseMeta(metadata: Record<string, unknown> | null): Record<string, unknown> {
  if (!metadata) return {};
  if (typeof metadata === 'string') {
    try { return JSON.parse(metadata); } catch { return {}; }
  }
  return metadata;
}

export function synthesizeSpineNodes(events: TimelineEvent[]): SpineNode[] {
  return events.map((e, i) => {
    const meta = parseMeta(e.metadata);
    const cap = (meta.capabilityName as string) ?? '';
    const capLabel = cap ? formatCapability(cap) : '';
    const icon = CATEGORY_ICONS[e.category] ?? '●';
    const baseStatus = CATEGORY_STATUS[e.category] ?? 'completed';

    let label: string;
    let nodeType: SpineNode['nodeType'] = 'lifecycle';
    let status = baseStatus;

    if (e.eventType === 'CASE_CREATED') {
      label = 'Opened';
    } else if (e.eventType === 'AGENT_ROUTED') {
      label = `${capLabel} → ${e.actor}`;
      nodeType = 'dispatch';
      status = 'active';
    } else if (e.eventType === 'WORKER_EXECUTION_COMPLETED') {
      const outcome = meta.outcome as string | undefined;
      const findingCount = meta.findingCount as number | undefined;
      if (outcome === 'FINDINGS' && findingCount) {
        label = `${capLabel}: ${findingCount} findings`;
        nodeType = 'findings';
      } else if (outcome === 'APPROVED') {
        label = `${capLabel}: APPROVED`;
      } else if (cap === 'code-analysis') {
        label = meta.securitySensitive ? 'Code analysis: security-sensitive' : 'Code analysis: standard';
      } else {
        label = `${capLabel}: ${outcome ?? 'DONE'}`;
      }
      status = 'completed';
    } else if (e.eventType === 'WORKITEM_CREATED') {
      label = 'Human gate';
      nodeType = 'human-gate';
      status = 'pending';
    } else if (e.eventType === 'WORKITEM_COMPLETED') {
      label = 'Human gate: decided';
      nodeType = 'human-gate';
      status = 'completed';
    } else if (e.category === 'CI') {
      label = `CI: ${e.summary.toLowerCase().includes('pass') ? 'passed' : e.summary.toLowerCase().includes('fail') ? 'failed' : 'pending'}`;
      status = e.summary.toLowerCase().includes('pass') ? 'completed' : e.summary.toLowerCase().includes('fail') ? 'completed' : 'pending';
    } else {
      label = e.summary || e.eventType;
    }

    return {
      key: `${e.timestamp}-${i}`,
      label,
      icon,
      status,
      timestamp: e.timestamp,
      nodeType,
      nodeData: { ...meta, actor: e.actor, eventType: e.eventType, caseId: meta.caseId },
    };
  });
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm test -- --run src/components/__tests__/spine-node-strategy.test.ts` from `app/src/main/webui/`
Expected: PASS

- [ ] **Step 6: Run typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: No errors

- [ ] **Step 7: Commit**

```bash
git add app/src/main/webui/src/components/spine-node-strategy.ts app/src/main/webui/src/components/review-timeline-strategy.ts app/src/main/webui/src/components/__tests__/spine-node-strategy.test.ts
git commit -m "feat(#216): add spine node strategy with label synthesis

Refs #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: Review spine component

**Files:**
- Create: `app/src/main/webui/src/components/review-spine.ts`
- Test: `app/src/main/webui/src/components/__tests__/review-spine.test.ts`

**Interfaces:**
- Consumes: `UnifiedPrRow` from `pr-cases-workbench.ts`, `synthesizeSpineNodes()` from `spine-node-strategy.ts` (Task 3), `ReviewDetailData` timeline from `GET /reviews/{caseId}`
- Produces: `<review-spine>` custom element. Properties: `cases: UnifiedPrRow[]`, `endpoint: string`. Emits `review:selected` pages event with `{ caseId: string, nodeType: string, nodeData: unknown }`. Consumed by pr-cases-workbench integration (Task 6).

- [ ] **Step 1: Write the failing test**

Create `app/src/main/webui/src/components/__tests__/review-spine.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';

describe('review-spine', () => {
  it('shouldExpand returns true for REVIEW stage with non-terminal status', () => {
    // Import the function once the module exists
    const { shouldExpand } = require('../review-spine.js');
    expect(shouldExpand({ stage: 'REVIEW', status: 'IN_REVIEW' })).toBe(true);
    expect(shouldExpand({ stage: 'REVIEW', status: 'ANALYSING' })).toBe(true);
    expect(shouldExpand({ stage: 'HUMAN_GATE', status: 'WAITING' })).toBe(true);
  });

  it('shouldExpand returns false for completed or queued cases', () => {
    const { shouldExpand } = require('../review-spine.js');
    expect(shouldExpand({ stage: 'REVIEW', status: 'APPROVED' })).toBe(false);
    expect(shouldExpand({ stage: 'REVIEW', status: 'COMPLETED' })).toBe(false);
    expect(shouldExpand({ stage: 'QUEUED', status: 'QUEUED' })).toBe(false);
    expect(shouldExpand({ stage: 'BATCHED', status: 'QUEUED' })).toBe(false);
  });

  it('borderColor maps stage+status correctly', () => {
    const { borderColor } = require('../review-spine.js');
    expect(borderColor('REVIEW', 'ANALYSING')).toBe('var(--pages-info-6, #2563eb)');
    expect(borderColor('REVIEW', 'IN_REVIEW')).toBe('var(--pages-warning-6, #d97706)');
    expect(borderColor('HUMAN_GATE', 'WAITING')).toBe('var(--pages-danger-6, #dc2626)');
    expect(borderColor('REVIEW', 'APPROVED')).toBe('var(--pages-success-6, #16a34a)');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test -- --run src/components/__tests__/review-spine.test.ts` from `app/src/main/webui/`
Expected: FAIL — module not found

- [ ] **Step 3: Create review-spine.ts**

Create `app/src/main/webui/src/components/review-spine.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { emitPagesEvent } from '@casehubio/pages-data';
import { synthesizeSpineNodes, type SpineNode } from './spine-node-strategy.js';
import type { UnifiedPrRow } from './pr-cases-workbench.js';

export function shouldExpand(row: { stage: string; status: string }): boolean {
  if (row.stage === 'HUMAN_GATE') return true;
  if (row.stage === 'REVIEW' && row.status !== 'APPROVED' && row.status !== 'COMPLETED') return true;
  return false;
}

export function borderColor(stage: string, status: string): string {
  if (status === 'APPROVED' || status === 'COMPLETED') return 'var(--pages-success-6, #16a34a)';
  if (stage === 'HUMAN_GATE') return 'var(--pages-danger-6, #dc2626)';
  if (stage === 'REVIEW' && status === 'ANALYSING') return 'var(--pages-info-6, #2563eb)';
  if (stage === 'REVIEW') return 'var(--pages-warning-6, #d97706)';
  return 'var(--pages-neutral-6, #737373)';
}

function statusText(stage: string, status: string): string {
  if (status === 'APPROVED' || status === 'COMPLETED') return 'approved';
  if (stage === 'HUMAN_GATE') return 'needs decision';
  if (stage === 'REVIEW' && status === 'ANALYSING') return 'analysing';
  if (stage === 'REVIEW') return 'in review';
  return status.toLowerCase();
}

interface CaseEvents {
  nodes: SpineNode[];
  loading: boolean;
}

@customElement('review-spine')
export class ReviewSpine extends LitElement {
  @property({ type: Array }) cases: UnifiedPrRow[] = [];
  @property({ type: String }) endpoint = '';
  @property({ type: String, attribute: 'selected-case-id' }) selectedCaseId = '';

  @state() private _expandedCases = new Set<string>();
  @state() private _caseEvents = new Map<string, CaseEvents>();
  @state() private _manualOverrides = new Set<string>();

  static override styles = css`
    :host { display: block; }
    .case-row {
      background: var(--pages-neutral-2, #f5f5f5);
      padding: 10px 12px;
      border-radius: 4px;
      margin-bottom: 4px;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: space-between;
      transition: background 0.15s;
    }
    .case-row:hover { background: var(--pages-neutral-3, #e5e5e5); }
    .case-row.selected { background: var(--pages-primary-2, #dbeafe); }
    .case-header-content { display: flex; align-items: center; gap: 10px; }
    .expand-icon { color: var(--pages-neutral-6, #737373); font-size: 10px; min-width: 12px; }
    .case-title { font-size: 12px; font-weight: 600; color: var(--pages-neutral-9, #171717); }
    .case-meta { font-size: 10px; color: var(--pages-neutral-7, #525252); }
    .status-indicator {
      width: 8px; height: 8px; border-radius: 50%; display: inline-block;
    }
    .events-panel {
      padding: 6px 0 6px 22px;
      border-left: 2px solid var(--pages-neutral-4, #d4d4d4);
      margin-left: 12px;
    }
    .event-node {
      font-size: 11px;
      margin: 4px 0;
      padding: 2px 6px;
      cursor: pointer;
      border-radius: 3px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      transition: background 0.1s;
    }
    .event-node:hover { background: var(--pages-neutral-3, #e5e5e5); }
    .event-node.selected { background: var(--pages-primary-2, #dbeafe); }
    .event-node.active { color: var(--pages-warning-8, #92400e); }
    .event-node.pending { color: var(--pages-neutral-6, #737373); }
    .event-node.completed { color: var(--pages-success-8, #166534); }
    .event-label { display: flex; align-items: center; gap: 6px; }
    .event-time { font-size: 10px; color: var(--pages-neutral-5, #a3a3a3); }
    .loading { font-size: 11px; color: var(--pages-neutral-6, #737373); padding: 6px 22px; margin-left: 12px; }
  `;

  override willUpdate(changed: Map<string, unknown>): void {
    if (changed.has('cases')) {
      for (const c of this.cases) {
        if (!this._manualOverrides.has(c.caseId)) {
          if (shouldExpand(c)) {
            this._expandedCases.add(c.caseId);
          } else {
            this._expandedCases.delete(c.caseId);
          }
        }
      }
      this.requestUpdate();
    }
  }

  private _toggleExpand(caseId: string): void {
    this._manualOverrides.add(caseId);
    if (this._expandedCases.has(caseId)) {
      this._expandedCases.delete(caseId);
    } else {
      this._expandedCases.add(caseId);
      this._fetchEvents(caseId);
    }
    this.requestUpdate();
  }

  private async _fetchEvents(caseId: string): Promise<void> {
    if (this._caseEvents.has(caseId) && !this._caseEvents.get(caseId)!.loading) return;
    this._caseEvents.set(caseId, { nodes: [], loading: true });
    this.requestUpdate();
    try {
      const res = await fetch(`${this.endpoint}/${caseId}`);
      if (res.ok) {
        const detail = await res.json();
        const nodes = synthesizeSpineNodes(detail.timeline ?? []);
        this._caseEvents.set(caseId, { nodes, loading: false });
      } else {
        this._caseEvents.set(caseId, { nodes: [], loading: false });
      }
    } catch {
      this._caseEvents.set(caseId, { nodes: [], loading: false });
    }
    this.requestUpdate();
  }

  private _selectCase(caseId: string): void {
    emitPagesEvent(document, 'review:selected', { caseId, nodeType: 'case-header', nodeData: { caseId } });
  }

  private _selectNode(caseId: string, node: SpineNode): void {
    emitPagesEvent(document, 'review:selected', { caseId, nodeType: node.nodeType, nodeData: node.nodeData });
  }

  private _formatTime(ts: string): string {
    if (!ts) return '';
    const diff = Date.now() - new Date(ts).getTime();
    const mins = Math.floor(diff / 60000);
    if (mins < 1) return 'just now';
    if (mins < 60) return `${mins}m ago`;
    const hours = Math.floor(mins / 60);
    if (hours < 24) return `${hours}h ago`;
    return `${Math.floor(hours / 24)}d ago`;
  }

  override render() {
    return html`
      ${this.cases.map(c => this._renderCase(c))}
    `;
  }

  private _renderCase(c: UnifiedPrRow) {
    const expanded = this._expandedCases.has(c.caseId);
    const events = this._caseEvents.get(c.caseId);
    const border = borderColor(c.stage, c.status);
    const isSelected = c.caseId === this.selectedCaseId;

    if (expanded && !events) {
      this._fetchEvents(c.caseId);
    }

    return html`
      <div>
        <div class="case-row ${isSelected ? 'selected' : ''}"
             style="border-left: 3px solid ${border}"
             @click=${() => { this._toggleExpand(c.caseId); this._selectCase(c.caseId); }}>
          <div class="case-header-content">
            <span class="expand-icon">${expanded ? '▼' : '▶'}</span>
            <div>
              <div class="case-title">PR #${c.prNumber} <span style="font-weight:400;color:var(--pages-neutral-7,#525252)">${c.author}, ${c.repo.split('/').pop()}, ${c.linesChanged} lines</span></div>
              ${!expanded ? html`<div class="case-meta">${statusText(c.stage, c.status)}</div>` : nothing}
            </div>
          </div>
          ${!expanded ? html`<span class="status-indicator" style="background:${border}"></span>` : nothing}
        </div>
        ${expanded ? html`
          ${events?.loading ? html`<div class="loading">Loading timeline...</div>` : nothing}
          ${events && !events.loading && events.nodes.length > 0 ? html`
            <div class="events-panel">
              ${events.nodes.map(n => html`
                <div class="event-node ${n.status}"
                     @click=${(e: Event) => { e.stopPropagation(); this._selectNode(c.caseId, n); }}>
                  <span class="event-label">
                    <span>${n.icon}</span>
                    <span>${n.label}</span>
                  </span>
                  <span class="event-time">${this._formatTime(n.timestamp)}</span>
                </div>
              `)}
            </div>
          ` : nothing}
        ` : nothing}
      </div>
    `;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test -- --run src/components/__tests__/review-spine.test.ts` from `app/src/main/webui/`
Expected: PASS

- [ ] **Step 5: Run typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add app/src/main/webui/src/components/review-spine.ts app/src/main/webui/src/components/__tests__/review-spine.test.ts
git commit -m "feat(#216): add review-spine component with collapse/expand and smart defaults

Refs #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Intelligence Pane + Integration

### Task 5: Intelligence pane component

**Files:**
- Create: `app/src/main/webui/src/components/intelligence-pane.ts`

**Interfaces:**
- Consumes: Shared primitives (`renderTrustBar`, `renderDimensionGrid`, `renderBadge`, `renderReviewCard`, `renderWorkItemAlert`), `<devtown-routing-summary>`, `<devtown-findings-panel>`. REST endpoints: `GET /reviews/{caseId}`, `GET /reviews/contributors/{actorId}`, `GET /reviews/{caseId}/similar-cases`, `GET /reviews/reviewers/{actorId}`, `GET /trust/{actorId}/routing-history/{entryId}`
- Produces: `<intelligence-pane>` custom element. Properties: `nodeType`, `nodeData`, `endpoint`, `caseId`. Consumed by pr-cases-workbench integration (Task 6).

- [ ] **Step 1: Create intelligence-pane.ts**

Create `app/src/main/webui/src/components/intelligence-pane.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { renderTrustBar } from './shared/trust-bar.js';
import { renderDimensionGrid } from './shared/dimension-grid.js';
import { renderBadge } from './shared/badge.js';
import { renderReviewCard } from './shared/review-card.js';
import { renderWorkItemAlert } from './shared/work-item-alert.js';
import { hostStyles, sectionTitleStyles, narrativeStyles, trustBarStyles, dimensionGridStyles, badgeStyles, workItemAlertStyles, reviewCardStyles } from './shared/shared-styles.js';
import './routing-summary.js';
import './findings-panel.js';

type NodeType = 'case-header' | 'dispatch' | 'findings' | 'human-gate' | 'lifecycle' | '';

interface CacheEntry {
  promise: Promise<unknown>;
  timestamp: number;
}

const CACHE_TTL_MS = 30_000;

@customElement('intelligence-pane')
export class IntelligencePaneElement extends LitElement {
  @property({ type: String, attribute: 'node-type' }) nodeType: NodeType = '';
  @property({ type: Object, attribute: false }) nodeData: Record<string, unknown> = {};
  @property({ type: String }) endpoint = '';
  @property({ type: String, attribute: 'case-id' }) caseId = '';

  @state() private _reviewDetail: Record<string, unknown> | null = null;
  @state() private _contributor: Record<string, unknown> | null = null;
  @state() private _precedents: unknown[] = [];
  @state() private _reviewer: Record<string, unknown> | null = null;
  @state() private _routingDetail: Record<string, unknown> | null = null;
  @state() private _loading = false;

  private _cache = new Map<string, CacheEntry>();

  static override styles = [
    hostStyles, sectionTitleStyles, narrativeStyles, trustBarStyles,
    dimensionGridStyles, badgeStyles, workItemAlertStyles, reviewCardStyles,
    css`
      :host { display: block; height: 100%; overflow: auto; padding: 16px; font-family: var(--pages-font-family, system-ui); }
      .section { margin-bottom: 16px; }
      .section-title { font-size: 11px; text-transform: uppercase; color: var(--pages-neutral-6, #737373); margin-bottom: 6px; font-weight: 600; }
      .finding-card { padding: 8px; border-radius: 4px; margin: 6px 0; border: 1px solid var(--pages-neutral-4, #d4d4d4); }
      .finding-high { border-color: var(--pages-danger-6, #dc2626); background: var(--pages-danger-1, #fef2f2); }
      .finding-medium { border-color: var(--pages-warning-6, #d97706); background: var(--pages-warning-1, #fffbeb); }
      .finding-severity { font-size: 11px; font-weight: 600; }
      .finding-file { font-size: 10px; color: var(--pages-neutral-6, #737373); }
      .precedent-card { padding: 8px; border-radius: 4px; margin: 6px 0; border-left: 3px solid var(--pages-primary-6, #2563eb); background: var(--pages-primary-1, #eff6ff); }
      .pane-title { font-size: 14px; font-weight: 600; margin-bottom: 12px; }
      .meta-row { font-size: 11px; color: var(--pages-neutral-7, #525252); margin-bottom: 4px; }
      .empty { padding: 40px; text-align: center; color: var(--pages-neutral-6, #737373); }
      .data-gap { font-size: 11px; color: var(--pages-neutral-5, #a3a3a3); font-style: italic; }
      .rationale-card { padding: 10px; border-radius: 4px; background: var(--pages-neutral-2, #f5f5f5); margin: 6px 0; }
      .candidate { padding: 4px 0; font-size: 11px; border-bottom: 1px solid var(--pages-neutral-3, #e5e5e5); }
      .candidate:last-child { border-bottom: none; }
    `,
  ];

  private async _cachedFetch(url: string): Promise<unknown> {
    const cached = this._cache.get(url);
    if (cached && Date.now() - cached.timestamp < CACHE_TTL_MS) {
      return cached.promise;
    }
    const promise = fetch(url).then(r => r.ok ? r.json() : null);
    this._cache.set(url, { promise, timestamp: Date.now() });
    return promise;
  }

  override willUpdate(changed: Map<string, unknown>): void {
    if (changed.has('nodeType') || changed.has('caseId') || changed.has('nodeData')) {
      this._loadData();
    }
  }

  private async _loadData(): Promise<void> {
    if (!this.caseId && this.nodeType !== 'lifecycle') return;
    this._loading = true;
    this._reviewDetail = null;
    this._contributor = null;
    this._precedents = [];
    this._reviewer = null;
    this._routingDetail = null;

    try {
      switch (this.nodeType) {
        case 'case-header': {
          const [detail, precedents] = await Promise.all([
            this._cachedFetch(`${this.endpoint}/${this.caseId}`),
            this._cachedFetch(`${this.endpoint}/${this.caseId}/similar-cases`),
          ]);
          this._reviewDetail = detail as Record<string, unknown>;
          this._precedents = (precedents as unknown[]) ?? [];
          const pr = (this._reviewDetail?.pr as Record<string, unknown>) ?? {};
          if (pr.contributor) {
            this._contributor = await this._cachedFetch(`${this.endpoint}/contributors/${pr.contributor}`) as Record<string, unknown>;
          }
          break;
        }
        case 'dispatch': {
          const actorId = this.nodeData.actor as string;
          const entryId = this.nodeData.entryId as string | undefined;
          const fetches: Promise<unknown>[] = [
            this._cachedFetch(`${this.endpoint}/reviewers/${actorId}`),
          ];
          if (entryId) {
            fetches.push(this._cachedFetch(`/api/devtown/trust/${actorId}/routing-history/${entryId}`));
          }
          const [reviewer, routing] = await Promise.all(fetches);
          this._reviewer = reviewer as Record<string, unknown>;
          this._routingDetail = (routing as Record<string, unknown>) ?? null;
          break;
        }
        case 'findings': {
          const [detail, precedents] = await Promise.all([
            this._cachedFetch(`${this.endpoint}/${this.caseId}`),
            this._cachedFetch(`${this.endpoint}/${this.caseId}/similar-cases`),
          ]);
          this._reviewDetail = detail as Record<string, unknown>;
          this._precedents = (precedents as unknown[]) ?? [];
          break;
        }
        case 'human-gate': {
          this._reviewDetail = await this._cachedFetch(`${this.endpoint}/${this.caseId}`) as Record<string, unknown>;
          break;
        }
        case 'lifecycle':
          break;
      }
    } catch (e) {
      console.warn('Intelligence pane fetch error', e);
    }
    this._loading = false;
  }

  override render() {
    if (!this.nodeType) return html`<div class="empty">Select a case or event node to view intelligence</div>`;
    if (this._loading) return html`<div class="empty">Loading...</div>`;

    switch (this.nodeType) {
      case 'case-header': return this._renderCaseHeader();
      case 'dispatch': return this._renderDispatch();
      case 'findings': return this._renderFindings();
      case 'human-gate': return this._renderHumanGate();
      case 'lifecycle': return this._renderLifecycle();
      default: return html`<div class="empty">Unknown node type</div>`;
    }
  }

  private _renderCaseHeader() {
    const pr = (this._reviewDetail?.pr as Record<string, unknown>) ?? {};
    const routing = (this._reviewDetail?.routing as Record<string, unknown>) ?? {};
    const contrib = this._contributor;

    return html`
      <div class="pane-title">PR #${pr.prNumber} — ${pr.repo}</div>
      <div class="meta-row">${pr.contributor} · ${pr.linesChanged} lines</div>

      ${contrib ? html`
        <div class="section">
          <div class="section-title">Contributor Profile</div>
          ${renderBadge((contrib as any).intakeClassification?.lane ?? 'standard')}
          ${renderTrustBar({ score: (contrib as any).globalScore ?? 0, thresholds: [], observationCount: 0 })}
          ${(contrib as any).dimensionScores ? renderDimensionGrid(Object.entries((contrib as any).dimensionScores).map(([key, value]) => ({ key, value: value as number }))) : nothing}
          ${(contrib as any).githubIntelligence ? html`
            <div class="meta-row">${(contrib as any).githubIntelligence.mergedCount} merges, ${Math.round((contrib as any).githubIntelligence.mergeRatio * 100)}% merge rate</div>
          ` : nothing}
        </div>
      ` : nothing}

      ${this._precedents.length > 0 ? html`
        <div class="section">
          <div class="section-title">Similar Past Cases (${this._precedents.length})</div>
          ${this._precedents.slice(0, 5).map((p: any) => html`
            <div class="precedent-card">
              <div style="font-size:12px;font-weight:600">PR #${p.prNumber} (${p.repo})</div>
              <div class="meta-row">Similarity: ${Math.round((p.similarity ?? 0) * 100)}% · ${p.completionMinutes ? `${p.completionMinutes}min` : ''}</div>
            </div>
          `)}
        </div>
      ` : nothing}

      ${routing.decisions ? html`
        <div class="section">
          <div class="section-title">Routing</div>
          <devtown-routing-summary .routing=${routing}></devtown-routing-summary>
        </div>
      ` : nothing}
    `;
  }

  private _renderDispatch() {
    const capName = this.nodeData.capabilityName as string ?? 'unknown';
    const reason = this.nodeData.reason as string ?? '';
    const confidence = this.nodeData.confidence as number ?? 0;
    const actor = this.nodeData.actor as string ?? '';
    const entryId = this.nodeData.entryId as string | undefined;

    return html`
      <div class="pane-title">${capName.replace(/-/g, ' ').replace(/\b\w/g, c => c.toUpperCase())} → ${actor}</div>

      <div class="section">
        <div class="section-title">Selection Rationale</div>
        <div class="rationale-card">
          <div class="meta-row">Reason: ${reason}</div>
          <div class="meta-row">Confidence: ${Math.round(confidence * 100)}%</div>
        </div>
      </div>

      ${this._reviewer ? html`
        <div class="section">
          <div class="section-title">Agent Profile</div>
          <div class="meta-row">${(this._reviewer as any).totalDecisions ?? 0} total decisions · ${(this._reviewer as any).openCommitments ?? 0} open commitments</div>
          ${(this._reviewer as any).trustByCapability ? renderDimensionGrid(Object.entries((this._reviewer as any).trustByCapability).map(([key, value]) => ({ key, value: value as number }))) : nothing}
        </div>
      ` : nothing}

      ${this._routingDetail ? html`
        <div class="section">
          <div class="section-title">Candidate Alternatives</div>
          ${((this._routingDetail as any).rationale?.candidates ?? []).map((c: any) => html`
            <div class="candidate">
              <strong>${c.workerId}</strong> — trust: ${c.trustScore?.toFixed(2) ?? '—'}, final: ${c.finalScore?.toFixed(2) ?? '—'}
              ${c.exclusionReason ? html` <em>(${c.exclusionReason})</em>` : nothing}
            </div>
          `)}
        </div>
      ` : html`
        ${!entryId ? html`<div class="data-gap">Full rationale available after review completes</div>` : nothing}
      `}
    `;
  }

  private _renderFindings() {
    const findings = (this._reviewDetail?.findings as Record<string, any[]>) ?? {};
    const allFindings = Object.entries(findings).flatMap(([cap, items]) =>
      items.map(f => ({ ...f, capability: cap }))
    );

    return html`
      <div class="pane-title">Findings (${allFindings.length})</div>

      ${allFindings.map(f => html`
        <div class="finding-card ${f.severity === 'HIGH' ? 'finding-high' : f.severity === 'MEDIUM' ? 'finding-medium' : ''}">
          <div class="finding-severity" style="color:${f.severity === 'HIGH' ? 'var(--pages-danger-7, #b91c1c)' : 'var(--pages-warning-7, #a16207)'}">${f.severity}: ${f.message}</div>
          <div class="finding-file">${f.filePath}${f.startLine ? `:${f.startLine}` : ''} · confidence ${Math.round(f.confidence * 100)}%</div>
        </div>
      `)}

      ${this._precedents.length > 0 ? html`
        <div class="section" style="margin-top:16px">
          <div class="section-title">Precedent Comparison (CBR)</div>
          ${this._precedents.slice(0, 3).map((p: any) => html`
            <div class="precedent-card">
              <div style="font-size:12px">Similar finding in PR #${p.prNumber} (${p.repo})</div>
              <div class="meta-row">Similarity: ${Math.round((p.similarity ?? 0) * 100)}%</div>
            </div>
          `)}
        </div>
      ` : nothing}
    `;
  }

  private _renderHumanGate() {
    const nodeData = this.nodeData;

    return html`
      <div class="pane-title">Human Decision Required</div>
      ${renderWorkItemAlert({
        workItemId: nodeData.workItemId as string ?? '',
        decisionType: nodeData.decisionType as string ?? '',
        candidateGroup: nodeData.candidateGroup as string ?? '',
        expiresAt: nodeData.expiresAt as string ?? '',
        escalationStage: nodeData.escalationStage as string ?? '',
        status: nodeData.status as string ?? 'PENDING',
      })}
    `;
  }

  private _renderLifecycle() {
    return html`
      <div class="pane-title">${this.nodeData.eventType ?? 'Event'}</div>
      <div class="meta-row">Actor: ${this.nodeData.actor ?? 'system'}</div>
      <div class="meta-row">Timestamp: ${this.nodeData.timestamp ? new Date(this.nodeData.timestamp as string).toLocaleString() : '—'}</div>
    `;
  }
}
```

- [ ] **Step 2: Run typecheck**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add app/src/main/webui/src/components/intelligence-pane.ts
git commit -m "feat(#216): add intelligence pane with node-type switching and dedup cache

Refs #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 6: Tab restructure + workbench integration

**Files:**
- Modify: `app/src/main/webui/src/views/review.ts:8-36`
- Modify: `app/src/main/webui/src/index.ts:1-15`
- Modify: `app/src/main/webui/src/components/pr-cases-workbench.ts` (multiple sections)
- Modify: `app/src/main/webui/src/datasets.ts:25`

**Interfaces:**
- Consumes: `<review-spine>` (Task 4), `<intelligence-pane>` (Task 5), `UnifiedPrRow`, pages-split-workbench, pages events
- Produces: 6-tab dashboard with Review Workbench as default, spine/table toggle, intelligence pane in detail slot

- [ ] **Step 1: Update index.ts — register new panels, remove old ones**

In `app/src/main/webui/src/index.ts`, replace lines 1-15:

```typescript
import { registerPanel } from "@casehubio/pages-runtime";
import type { PermissionContext } from "@casehubio/pages-component";
import "./components/reviewer-workbench";
import "./components/contributor-workbench";
import "./components/pr-cases-workbench";
import "./components/system-workbench";
import "./components/operations-workbench";
import "./components/review-spine";
import "./components/intelligence-pane";

const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

registerPanel("reviewer-workbench", "devtown-reviewer-workbench");
registerPanel("contributor-workbench", "devtown-contributor-workbench");
registerPanel("pr-cases-workbench", "devtown-pr-cases-workbench");
registerPanel("system-workbench", "devtown-system-workbench");
registerPanel("operations-workbench", "devtown-operations-workbench");
```

Remove `import "@casehubio/mosaic-session-workbench";` and `registerPanel("session-workbench", ...)`.

- [ ] **Step 2: Update review.ts — new 6-tab structure**

Replace `app/src/main/webui/src/views/review.ts`:

```typescript
import { page, tabs, hostPanel } from "@casehubio/pages-ui";
import type { SiteOptions } from "@casehubio/pages-runtime";
import { loadSite } from "@casehubio/pages-runtime";
import { reviewersView } from "./reviewers";
import { createDatasets } from "../datasets";

export async function mountReviewDashboard(
  container: HTMLElement,
  siteOptions: SiteOptions,
  prefs: Record<string, string>,
  prefersDark: boolean,
  initialCaseId: string | null,
): Promise<void> {
  const app = page("DevTown",
    tabs(
      ["Review Workbench", hostPanel("pr-cases-workbench", {
        endpoint: "/api/devtown/governance",
        ...(initialCaseId ? { initialCaseId } : {}),
      })],
      ["Operations", hostPanel("operations-workbench", { endpoint: "/api/devtown/governance" })],
      ["Merge Queue", hostPanel("pr-cases-workbench", {
        endpoint: "/api/devtown/governance",
        filterMode: "queued",
      })],
      ["Reviewers", reviewersView],
      ["Contributors", hostPanel("contributor-workbench", { endpoint: "/api/devtown/reviews" })],
      ["System", hostPanel("system-workbench", { endpoint: "/api/devtown/governance" })],
    ),
    { settings: { mode: prefersDark ? "dark" : "light" }, datasets: createDatasets(prefs) },
  );

  const site = await loadSite(container, app, siteOptions);
  site.setTheme(prefersDark ? "dark" : "light");

  window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", (e) => {
    site.setTheme(e.matches ? "dark" : "light");
  });
}
```

Note: Remove import of `definitionsView`. The Merge Queue tab temporarily reuses `pr-cases-workbench` with a `filterMode` prop — Task 7 (Batch 4) will create the dedicated `merge-queue-workbench`.

- [ ] **Step 3: Update pr-cases-workbench.ts — add spine/table toggle, intelligence pane**

Add spine/table toggle state to the class (after line 210):

```typescript
@state() private _viewMode: 'spine' | 'table' = 'spine';
```

Add `filterMode` property (after line 208):

```typescript
@property({ type: String, attribute: 'filter-mode' }) filterMode = '';
```

In the `configure()` method (line 308), add:
```typescript
if (props.filterMode) this.filterMode = String(props.filterMode);
```

Update `_filteredRows()` to filter out QUEUED/BATCHED unless filterMode is 'queued':

```typescript
private _filteredRows(): UnifiedPrRow[] {
  let rows = this._unified;
  if (this.filterMode === 'queued') {
    rows = rows.filter(r => r.stage === 'QUEUED' || r.stage === 'BATCHED');
  } else {
    rows = rows.filter(r => r.stage !== 'QUEUED' && r.stage !== 'BATCHED');
  }
  switch (this._filter) {
    case 'review': return rows.filter(r => r.stage === 'REVIEW' || r.stage === 'HUMAN_GATE');
    case 'decision': return rows.filter(r => r.stage === 'HUMAN_GATE');
    case 'queued': return rows.filter(r => r.stage === 'QUEUED' || r.stage === 'BATCHED');
    default: return rows;
  }
}
```

Update the render method list slot to switch between spine and table, and update the detail slot to use intelligence-pane:

Replace the `<pages-split-workbench>` content in `render()`:

```typescript
return html`
  <pages-split-workbench selection-topic="review" title="Review Workbench">
    <div slot="list" class="list-panel">
      <div class="filter-chips">
        ${this._renderChip('all', 'All', counts.all)}
        ${this._renderChip('review', 'In Review', counts.review)}
        ${this._renderChip('decision', 'Needs Decision', counts.decision)}
        ${this.filterMode !== 'queued' ? html`
          <button class="filter-chip" @click=${() => { this._viewMode = this._viewMode === 'spine' ? 'table' : 'spine'; }}>
            ${this._viewMode === 'spine' ? '☰ Table' : '◆ Spine'}
          </button>
        ` : nothing}
      </div>

      ${renderVitals(vitals)}

      ${this._problems.length > 0 ? html`
        <div class="problems-banner">
          ${this._problems.map(p => html`
            <div class="problem-item" @click=${() => this._handleProblemClick(p)}>
              <span class="problem-severity ${p.severity === 'error' ? 'error' : ''}">${p.severity}</span>
              <span>${p.description}</span>
            </div>
          `)}
        </div>
      ` : nothing}

      ${this._viewMode === 'spine' && this.filterMode !== 'queued' ? html`
        <review-spine
          .cases=${this._filteredRows()}
          endpoint="/api/devtown/reviews"
          selected-case-id=${this._selectedCaseId}
        ></review-spine>
      ` : html`
        <h3>${this._filterLabel()} (${this._filteredRows().length})</h3>
        ${this._listData ? html`
          <pages-table
            .dataSet=${this._listData}
            .columnConfig=${FILTER_COLUMNS[this._filter]}
            @row-activate=${this._handleRowActivate}
          ></pages-table>
        ` : nothing}

        ${this._filter === 'queued' && this._batches.length > 0 ? html`
          <h3>
            <button class="batch-toggle" @click=${() => { this._batchesExpanded = !this._batchesExpanded; }}>
              ${this._batchesExpanded ? '▾' : '▸'} Active Batches (${this._batches.length})
            </button>
          </h3>
          ${this._batchesExpanded && this._batchData ? html`
            <pages-table
              .dataSet=${this._batchData}
              .columnConfig=${BATCH_TABLE_CONFIG}
              @row-activate=${this._handleBatchRowActivate}
            ></pages-table>
          ` : nothing}
        ` : nothing}
      `}
    </div>
    <div slot="detail" class="detail-panel">
      ${this._selectionKind === 'batch' && this._selectedBatch
        ? html`<devtown-pr-detail
            mode="batch"
            endpoint="${this.endpoint}"
            .batch=${this._selectedBatch}
          ></devtown-pr-detail>`
        : this.filterMode === 'queued'
          ? html`<devtown-pr-detail
              case-id=${this._selectedCaseId}
              endpoint="/api/devtown/reviews"
              .stage=${selectedRow?.stage ?? ''}
              .queueContext=${{ priorityLane: selectedRow?.priorityLane ?? '', trustScore: selectedRow?.trustScore ?? 0, waitMinutes: selectedRow?.waitMinutes ?? 0, headSha: selectedRow?.headSha ?? '', dependsOn: selectedRow?.dependsOn ?? [] }}
            ></devtown-pr-detail>`
          : html`<intelligence-pane
              node-type=${this._selectedNodeType}
              .nodeData=${this._selectedNodeData}
              endpoint="/api/devtown/reviews"
              case-id=${this._selectedCaseId}
            ></intelligence-pane>`
      }
    </div>
  </pages-split-workbench>
`;
```

Add state for intelligence pane selection:

```typescript
@state() private _selectedNodeType = '';
@state() private _selectedNodeData: Record<string, unknown> = {};
```

Update the `review:selected` event handler to capture nodeType and nodeData:

```typescript
onPagesEvent(document, 'review:selected', (detail: Record<string, unknown>) => {
  const caseId = detail?.caseId as string;
  if (caseId) {
    this._selectionKind = 'pr';
    this._selectedCaseId = caseId;
    this._selectedBatch = null;
    this._selectedNodeType = (detail?.nodeType as string) ?? 'case-header';
    this._selectedNodeData = (detail?.nodeData as Record<string, unknown>) ?? {};
  }
}),
```

- [ ] **Step 4: Update datasets.ts — add WebSocket trigger to queue-status**

In `app/src/main/webui/src/datasets.ts`, replace line 25:

```typescript
rest("queue-status", "/api/devtown/governance/queue-status", {
  dataPath: "reviews",
  refreshTime: opRefresh,
  triggerUrl: wsUrl,
  triggerFilter: (e: WsTriggerEvent) =>
    e.topic === "case.state" || e.topic === "planitem.state" || e.topic === "workitem.lifecycle",
}),
```

- [ ] **Step 5: Run typecheck and full build**

Run: `npm run typecheck` from `app/src/main/webui/`
Expected: No errors

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/webui/src/views/review.ts app/src/main/webui/src/index.ts app/src/main/webui/src/components/pr-cases-workbench.ts app/src/main/webui/src/datasets.ts
git commit -m "feat(#216): restructure tabs to 6-tab layout with spine/table toggle and intelligence pane

Refs #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Merge Queue + Cleanup

### Task 7: Merge queue workbench

**Files:**
- Create: `app/src/main/webui/src/components/merge-queue-workbench.ts`

**Interfaces:**
- Consumes: `QueuedPrEntry`, `ActiveBatchEntry`, `MergeQueueMetrics` types (from pr-cases-workbench.ts or re-extracted), `<devtown-pr-detail>` in queue/batch mode, `pages-split-workbench`
- Produces: `<merge-queue-workbench>` custom element registered as panel `merge-queue-workbench`. Replaces the temporary Merge Queue tab in review.ts (Task 6).

- [ ] **Step 1: Create merge-queue-workbench.ts**

Create `app/src/main/webui/src/components/merge-queue-workbench.ts` — a dedicated split-pane for merge queue data. Extract the queued PRs table, active batches section, and queue vitals from `pr-cases-workbench.ts`. Wire `<devtown-pr-detail>` in the detail slot for queue/batch modes.

The component fetches from `/api/devtown/governance/merge-queue` and `/api/devtown/governance/merge-queue/metrics`, renders queued PRs table in the list slot, and `devtown-pr-detail` in the detail slot. This replaces the temporary `filterMode: "queued"` approach from Task 6.

- [ ] **Step 2: Register panel in index.ts**

Add to `app/src/main/webui/src/index.ts`:

```typescript
import "./components/merge-queue-workbench";
registerPanel("merge-queue-workbench", "devtown-merge-queue-workbench");
```

- [ ] **Step 3: Update review.ts Merge Queue tab**

Replace the Merge Queue tab entry from Task 6's temporary `pr-cases-workbench` reuse:

```typescript
["Merge Queue", hostPanel("merge-queue-workbench", { endpoint: "/api/devtown/governance" })],
```

- [ ] **Step 4: Run typecheck and build**

Run: `npm run typecheck` from `app/src/main/webui/`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/webui/src/components/merge-queue-workbench.ts app/src/main/webui/src/index.ts app/src/main/webui/src/views/review.ts
git commit -m "feat(#216): add dedicated merge-queue-workbench component

Refs #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 8: pr-detail narrowing + Operations/System cleanup

**Files:**
- Modify: `app/src/main/webui/src/components/pr-detail.ts` (remove review-mode rendering)
- Modify: `app/src/main/webui/src/components/operations-workbench.ts` (add sessions, fix nav)
- Modify: `app/src/main/webui/src/components/system-workbench.ts` (remove problems, focus calibration)
- Delete: `app/src/main/webui/src/views/operations.ts` (use `ide_refactor_safe_delete`)
- Delete: `app/src/main/webui/src/views/system.ts` (use `ide_refactor_safe_delete`)

**Interfaces:**
- Consumes: None new — narrows existing component interfaces
- Produces: Cleaned pr-detail (queue/batch only), mounted operations workbench with sessions, focused system workbench

- [ ] **Step 1: Narrow pr-detail.ts**

Remove review-mode rendering from `pr-detail.ts`. The component now only handles `stage === 'QUEUED'` and `mode === 'batch'`. Remove:
- Review detail fetch and rendering (the contributor trust section, routing summary section, timeline section, findings panel — all moved to intelligence-pane)
- Import of `reviewTimelineStrategy` and `@casehubio/mosaic-agency-timeline`
- Import of `devtown-routing-summary` and `devtown-findings-panel`
- The `ContributorProfile` and `ReviewDetailData` interfaces (no longer needed in this component)

Keep:
- Queue context rendering (priority lane, trust score, wait time, dependencies)
- Batch mode rendering (batch narrative, PR list, risk, CI status)
- Action buttons (Approve, Request Changes, Dequeue, Signal CI Pass)
- WorkItem alert rendering

- [ ] **Step 2: Fix operations-workbench.ts navigation**

In `operations-workbench.ts`, find `_navigateToReview()` method. Replace the brittle `textContent?.trim() === 'Reviews'` tab selector with `data-panel-id` attribute matching:

```typescript
private _navigateToReview(caseId: string): void {
  const tabs = document.querySelectorAll('[data-panel-id]');
  for (const tab of tabs) {
    if (tab.getAttribute('data-panel-id') === 'pr-cases-workbench') {
      (tab as HTMLElement).click();
      break;
    }
  }
  emitPagesEvent(document, 'review:selected', { caseId });
}
```

- [ ] **Step 3: Add sessions section to operations-workbench.ts**

Add a collapsible "Sessions" section at the bottom of the operations workbench that fetches from `/api/sessions`. Since this is currently a dev-mode stub returning empty data, render "No session data" when empty.

- [ ] **Step 4: Refocus system-workbench.ts**

Remove the problems table display from `system-workbench.ts` (problems now appear in Operations only). Keep SLA calibration table, merge queue configuration settings, and fleet settings.

- [ ] **Step 5: Delete unused view files**

Use `ide_refactor_safe_delete` for:
- `app/src/main/webui/src/views/operations.ts`
- `app/src/main/webui/src/views/system.ts`

These are already unused (superseded by the workbench components).

- [ ] **Step 6: Run typecheck and full build**

Run: `npm run typecheck` from `app/src/main/webui/`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS

- [ ] **Step 7: Visual verification**

Run: `mvn quarkus:dev -pl app` and verify:
- All 6 tabs render without errors
- Review Workbench shows spine with collapse/expand
- Clicking spine nodes populates intelligence pane
- Table toggle switches to sortable table view
- Merge Queue tab shows queued PRs and batches
- Operations tab shows problems, events, sessions
- System tab shows SLA calibration without problems

- [ ] **Step 8: Commit**

```bash
git add app/src/main/webui/src/components/pr-detail.ts app/src/main/webui/src/components/operations-workbench.ts app/src/main/webui/src/components/system-workbench.ts
git commit -m "feat(#216): narrow pr-detail, mount operations with sessions, refocus system tab

Closes #216

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-10-29-review-intelligence-workbench-design.md](../specs/issue-216-review-workbench-ui/2026-10-29-review-intelligence-workbench-design.md) — design spec this plan implements
- `app/src/main/java/io/casehub/devtown/app/api/DevtownReviewApi.java:27-88` — @McpDomain REST API, CBR endpoint added here
- `app/src/main/java/io/casehub/devtown/app/governance/GovernanceQueryService.java:110-127,378-389` — ReviewDetail, RoutingDecision records, routing extraction
- `io.casehub.ledger.model.WorkerDecisionEntry` (engine/ledger-core) — entryId correlation source
- `io.casehub.ledger.repository.CaseLedgerEntryRepository:54-57` — `findWorkerDecisionsByCaseId()`
- `app/src/main/webui/src/views/review.ts:8-36` — current tab structure
- `app/src/main/webui/src/index.ts:1-15` — panel registration
- `app/src/main/webui/src/datasets.ts:25` — queue-status dataset (WebSocket trigger added)
- `app/src/main/webui/src/components/review-timeline-strategy.ts:13-31` — CATEGORY_ICONS, CATEGORY_STATUS (now exported)
- `app/src/main/webui/src/components/pr-cases-workbench.ts:84-101,205-648` — UnifiedPrRow, workbench rendering
- `app/src/main/webui/src/components/pr-detail.ts:62-79` — mode/stage properties, review/queue/batch rendering
- `domain/src/main/java/io/casehub/devtown/domain/cbr/PrFeatureVector.java:12-22,63-72` — feature vector record and factory
- `review/src/main/java/io/casehub/devtown/review/CbrRetrievalService.java:9` — `findSimilar()` interface
- GitHub #216 — full requirements and acceptance criteria
