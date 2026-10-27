# Evolution Standalone App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #268 — Restructure evolution conductor as standalone app at /evolution
**Issue group:** #268, #230

**Goal:** Wire the pre-built evolution workbench as a standalone page at `/evolution` with client-side path routing, a bootstrap safety fix, conductor-origin badge on PRs, and cross-app deep linking.

**Architecture:** Path-based dispatch in `index.ts` before `loadSite()` — mount either the review dashboard or evolution workbench. One bundle, one HTML file, shared auth/preferences/theme. One-way dependency: evolution links to review, review shows a conductor badge but otherwise doesn't know evolution exists.

**Tech Stack:** TypeScript/Lit (frontend), Java 21/Quarkus 3.39.3 (backend), esbuild (build), pages-runtime/pages-ui (UI framework)

## Global Constraints

- Java 21 source on Java 26 JVM
- Quarkus 3.39.3
- All `@casehubio` packages via `file:` references to `.casehub-packages/packages/`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- TypeScript check: `npm run typecheck` from `app/src/main/webui/`
- Existing tests use Mockito with field injection for Java, manual verification for frontend
- Evolution backend classes at `app/src/main/java/io/casehub/devtown/app/evolution/`
- Frontend at `app/src/main/webui/src/`

---

## Batch 1: Backend safety + enrichment

### Task 1: Fix DevtownEvolutionBootstrap startup crash

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrap.java:20-23`
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrapTest.java`

**Interfaces:**
- Consumes: `DevtownEvolutionCaseHub.startCase()` (existing)
- Produces: No interface change — startup no longer crashes on evolution failure

- [ ] **Step 1: Write failing test — bootstrap survives startCase() exception**

Add to `DevtownEvolutionBootstrapTest.java`:

```java
@Test
void startupSurvivesWhenStartCaseFails() {
    when(caseInstanceRepository.findByNamespaceAndName("devtown", "evolution", "default"))
            .thenReturn(Optional.empty());
    when(caseHub.startCase()).thenThrow(new RuntimeException("engine unavailable"));

    assertDoesNotThrow(() -> bootstrap.onStartup(null));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownEvolutionBootstrapTest#startupSurvivesWhenStartCaseFails -f pom.xml`
Expected: FAIL — `RuntimeException: engine unavailable` propagates

- [ ] **Step 3: Wrap startCase() in try-catch**

In `DevtownEvolutionBootstrap.java`, replace the bare `caseHub.startCase()` call (line 22) with:

```java
try {
    caseHub.startCase();
} catch (Exception e) {
    Log.errorf(e, "Evolution case bootstrap failed — evolution features unavailable");
}
```

Add import `io.quarkus.logging.Log` if not present.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownEvolutionBootstrapTest -f pom.xml`
Expected: All 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrap.java app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrapTest.java
git commit -m "fix(#268): wrap evolution bootstrap in try-catch — startup no longer crashes

Refs casehubio/devtown#268"
```

---

### Task 2: Add deepLink field to DevtownImprovementStreamView + enricher

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownImprovementStreamView.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricher.java`
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricherTest.java`

**Interfaces:**
- Consumes: `ImprovementStreamView.improvementCaseId()` (existing engine type)
- Produces: `DevtownImprovementStreamView.deepLink()` — `@Nullable String`, value `"/?caseId={improvementCaseId}"` when the stream has a target case, `null` otherwise

- [ ] **Step 1: Write failing test — deepLink populated for stream with target**

Add to `DevtownEvolutionEnricherTest.java`:

```java
@Test
void enrichSetsDeepLinkForStreamWithTarget() {
    var stream = createStream("ci-reliability", "pr-123-target");
    var state = createState(List.of(stream));

    var result = enricher.enrich(state);

    assertNotNull(result.enrichedStreams().get(0).deepLink());
    assertTrue(result.enrichedStreams().get(0).deepLink().contains("caseId="));
}

@Test
void enrichSetsNullDeepLinkForStreamWithoutTarget() {
    var stream = createStream("ci-reliability", null);
    var state = createState(List.of(stream));

    var result = enricher.enrich(state);

    assertNull(result.enrichedStreams().get(0).deepLink());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownEvolutionEnricherTest -f pom.xml`
Expected: FAIL — `deepLink()` method doesn't exist on record

- [ ] **Step 3: Add deepLink field to DevtownImprovementStreamView**

Replace the record definition:

```java
public record DevtownImprovementStreamView(
        ImprovementStreamView stream,
        @Nullable String enrichedTarget,
        @Nullable String deepLink) {}
```

- [ ] **Step 4: Update DevtownEvolutionEnricher to populate deepLink**

In the enricher's stream mapping logic, construct the deepLink from the stream's `improvementCaseId`:

```java
String deepLink = stream.target() != null
        ? "/?caseId=" + stream.improvementCaseId()
        : null;
```

Pass `deepLink` as the third argument to the `DevtownImprovementStreamView` constructor.

- [ ] **Step 5: Fix existing tests — update constructor calls**

Existing tests that construct `DevtownImprovementStreamView` directly need the third parameter. Update all call sites to pass `null` for `deepLink` where not testing it.

- [ ] **Step 6: Run all enricher tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownEvolutionEnricherTest -f pom.xml`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/DevtownImprovementStreamView.java app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricher.java app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricherTest.java
git commit -m "feat(#268): add deepLink field to DevtownImprovementStreamView

Enricher populates /?caseId={improvementCaseId} for streams with targets.
Refs casehubio/devtown#268"
```

---

### Task 3: Add conductor-origin endpoint

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionResource.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionApi.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/ConductorOriginResponse.java`
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionApiTest.java`

**Interfaces:**
- Consumes: `EngineEvolutionApi` (existing — needs a method to check if a case was conductor-driven)
- Produces: `GET /api/devtown/evolution/conductor-origin/{caseId}` → `ConductorOriginResponse`

- [ ] **Step 1: Create ConductorOriginResponse record**

```java
package io.casehub.devtown.app.evolution;

import jakarta.annotation.Nullable;

public record ConductorOriginResponse(
        boolean conductorDriven,
        @Nullable String streamCategory) {

    public static ConductorOriginResponse notConductorDriven() {
        return new ConductorOriginResponse(false, null);
    }

    public static ConductorOriginResponse conductorDriven(String streamCategory) {
        return new ConductorOriginResponse(true, streamCategory);
    }
}
```

- [ ] **Step 2: Write failing test — conductor-origin lookup**

Add to `DevtownEvolutionApiTest.java`:

```java
@Test
void getConductorOriginReturnsDrivenWhenStreamTargetsCase() {
    var stream = createStream("ci-reliability", "target-case-id");
    when(engineApi.getStreams(any())).thenReturn(List.of(stream));

    var result = api.getConductorOrigin("default", "target-case-id");

    assertTrue(result.conductorDriven());
    assertEquals("ci-reliability", result.streamCategory());
}

@Test
void getConductorOriginReturnsNotDrivenWhenNoMatch() {
    when(engineApi.getStreams(any())).thenReturn(List.of());

    var result = api.getConductorOrigin("default", "unknown-case-id");

    assertFalse(result.conductorDriven());
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownEvolutionApiTest -f pom.xml`
Expected: FAIL — `getConductorOrigin` method doesn't exist

- [ ] **Step 4: Implement getConductorOrigin in DevtownEvolutionApi**

Add method to `DevtownEvolutionApi.java`:

```java
public ConductorOriginResponse getConductorOrigin(String tenancyId, String caseId) {
    UUID evolutionCaseId = caseResolver.resolve(tenancyId);
    var streams = engineApi.getStreams(evolutionCaseId);
    return streams.stream()
            .filter(s -> caseId.equals(s.target()))
            .findFirst()
            .map(s -> ConductorOriginResponse.conductorDriven(s.category()))
            .orElse(ConductorOriginResponse.notConductorDriven());
}
```

- [ ] **Step 5: Add REST endpoint to DevtownEvolutionResource**

Add to `DevtownEvolutionResource.java`:

```java
@GET
@Path("/conductor-origin/{caseId}")
public ConductorOriginResponse getConductorOrigin(@PathParam("caseId") String caseId) {
    return api.getConductorOrigin(DEFAULT_TENANCY, caseId);
}
```

- [ ] **Step 6: Run all evolution API tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownEvolutionApiTest -f pom.xml`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/devtown/app/evolution/ConductorOriginResponse.java app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionApi.java app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionResource.java app/src/test/java/io/casehub/devtown/app/evolution/DevtownEvolutionApiTest.java
git commit -m "feat(#268): add conductor-origin endpoint

GET /api/devtown/evolution/conductor-origin/{caseId} — returns whether a PR
was created by the evolution conductor and which stream drove it.
Refs casehubio/devtown#268"
```

---

## Batch 2: Frontend — evolution page + routing

### Task 4: Wire evolution packages + create evolution page + path-based routing

**Files:**
- Modify: `app/src/main/webui/package.json`
- Create: `app/src/main/webui/src/views/evolution.ts`
- Create: `app/src/main/webui/src/views/review.ts`
- Modify: `app/src/main/webui/src/index.ts`
- Modify: `app/src/main/resources/application.properties` (auth paths only)

**Interfaces:**
- Consumes: `@casehubio/mosaic-evolution-workbench` (`<agency-evolution-workbench>` element with `endpoint`, `tenancyId` props), `@casehubio/pages-runtime` (`loadSite`), `@casehubio/pages-ui` (`page`, `tabs`, `hostPanel`)
- Produces: `/evolution` route mounts evolution workbench, `/` route mounts review dashboard

- [ ] **Step 1: Add evolution packages to package.json**

Add two entries to the `dependencies` object in `app/src/main/webui/package.json`, alphabetically between the existing `@casehubio/mosaic-execution-monitor` and `@casehubio/mosaic-gdpr-erasure-action` entries:

```json
"@casehubio/mosaic-evolution-config": "file:.casehub-packages/packages/evolution-config",
"@casehubio/mosaic-evolution-workbench": "file:.casehub-packages/packages/evolution-workbench",
```

- [ ] **Step 2: Run npm install**

Run: `npm install --prefix app/src/main/webui`
Expected: Installs cleanly, `postinstall` creates symlinks in `node_modules/@casehubio/`

- [ ] **Step 3: Create views/review.ts — extract review dashboard**

Create `app/src/main/webui/src/views/review.ts`:

```typescript
import { page, tabs, hostPanel } from "@casehubio/pages-ui";
import type { SiteOptions } from "@casehubio/pages-runtime";
import { loadSite } from "@casehubio/pages-runtime";
import { reviewersView } from "./reviewers";
import { definitionsView } from "./definitions";
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
      ["PR Cases", hostPanel("pr-cases-workbench", {
        endpoint: "/api/devtown/governance",
        ...(initialCaseId ? { initialCaseId } : {}),
      })],
      ["Reviewers", reviewersView],
      ["Contributors", hostPanel("contributor-workbench", { endpoint: "/api/devtown/reviews" })],
      ["Workers", hostPanel("session-workbench", { endpoint: "/api/sessions" })],
      ["System", hostPanel("system-workbench", { endpoint: "/api/devtown/governance" })],
      ["Definitions", definitionsView],
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

- [ ] **Step 4: Create views/evolution.ts — evolution standalone page**

Create `app/src/main/webui/src/views/evolution.ts`:

```typescript
import "@casehubio/mosaic-evolution-workbench";

export function mountEvolutionPage(
  container: HTMLElement,
  tenancyId: string,
  prefersDark: boolean,
): void {
  const nav = document.createElement("nav");
  nav.style.cssText = "padding: 8px 16px; border-bottom: 1px solid var(--border-color, #e0e0e0);";
  const link = document.createElement("a");
  link.href = "/";
  link.textContent = "← DevTown Review";
  link.style.cssText = "color: var(--link-color, #0066cc); text-decoration: none; font-size: 14px;";
  nav.appendChild(link);

  const workbench = document.createElement("agency-evolution-workbench");
  workbench.setAttribute("endpoint", "/api/devtown/evolution");
  workbench.setAttribute("tenancy-id", tenancyId);
  workbench.style.cssText = "display: block; height: calc(100vh - 41px);";

  if (prefersDark) {
    workbench.setAttribute("theme", "dark");
  }

  container.appendChild(nav);
  container.appendChild(workbench);
}
```

- [ ] **Step 5: Refactor index.ts — path-based dispatch**

Replace the contents of `app/src/main/webui/src/index.ts`:

```typescript
import { registerPanel } from "@casehubio/pages-runtime";
import type { PermissionContext } from "@casehubio/pages-component";
import "@casehubio/mosaic-session-workbench";
import "./components/reviewer-workbench";
import "./components/contributor-workbench";
import "./components/pr-cases-workbench";
import "./components/system-workbench";

const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

registerPanel("session-workbench", "agency-session-workbench");
registerPanel("reviewer-workbench", "devtown-reviewer-workbench");
registerPanel("contributor-workbench", "devtown-contributor-workbench");
registerPanel("pr-cases-workbench", "devtown-pr-cases-workbench");
registerPanel("system-workbench", "devtown-system-workbench");

interface UserInfo {
  actorId: string;
  roles: string[];
  tenancyId: string;
  authenticated: boolean;
}

function authFetch(input: RequestInfo | URL, init?: RequestInit): Promise<Response> {
  return fetch(input, init).then(r => {
    if (r.status === 401) {
      window.location.href = "/api/userinfo";
    }
    return r;
  });
}

function permissionsFrom(user: UserInfo): PermissionContext {
  const roleSet = new Set(user.roles);
  return {
    hasRole: (role: string) => roleSet.has(role),
    hasPermission: (permission: string) => roleSet.has(permission),
  };
}

async function start() {
  const userRes = await fetch("/api/userinfo");
  if (userRes.status === 401 || !userRes.ok) {
    window.location.href = "/api/userinfo";
    return;
  }
  const user: UserInfo = await userRes.json();

  const container = document.getElementById("app");
  if (!container) return;

  if (window.location.pathname.startsWith("/evolution")) {
    const { mountEvolutionPage } = await import("./views/evolution");
    mountEvolutionPage(container, user.tenancyId, prefersDark);
  } else {
    const prefs = await authFetch("/api/devtown/governance/preferences")
      .then(r => r.ok ? r.json() : {})
      .then(json => {
        const flat: Record<string, string> = {};
        for (const [section, values] of Object.entries(json)) {
          for (const [key, val] of Object.entries(values as Record<string, string>)) {
            flat[`${section}.${key}`] = val;
          }
        }
        return flat;
      })
      .catch(() => ({}));

    const initialCaseId = new URLSearchParams(window.location.search).get("caseId");

    const { mountReviewDashboard } = await import("./views/review");
    await mountReviewDashboard(container, {
      permissions: permissionsFrom(user),
      fetch: authFetch,
    }, prefs, prefersDark, initialCaseId);
  }
}

start().catch(console.error);
```

- [ ] **Step 6: Update application.properties auth paths**

In `app/src/main/resources/application.properties`, find the line:

```
quarkus.http.auth.permission.quinoa-static.paths=/,/index.html,/app.js,/assets/*,/favicon.ico
```

Replace with:

```
quarkus.http.auth.permission.quinoa-static.paths=/,/index.html,/app.js,/assets/*,/favicon.ico,/evolution,/evolution/*
```

- [ ] **Step 7: Run TypeScript type check**

Run: `npm run typecheck --prefix app/src/main/webui`
Expected: No type errors

- [ ] **Step 8: Run full Maven build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml`
Expected: BUILD SUCCESS — Quinoa builds the frontend, all tests pass

- [ ] **Step 9: Commit**

```bash
git add app/src/main/webui/package.json app/src/main/webui/package-lock.json app/src/main/webui/src/index.ts app/src/main/webui/src/views/evolution.ts app/src/main/webui/src/views/review.ts app/src/main/resources/application.properties
git commit -m "feat(#268,#230): evolution standalone page at /evolution with path routing

Wire evolution-workbench + evolution-config packages. Extract review dashboard
to views/review.ts. Path-based dispatch in index.ts — /evolution mounts
workbench, / mounts review dashboard. Shared auth and theme.
Closes casehubio/devtown#230
Refs casehubio/devtown#268"
```

---

## Batch 3: Cross-app indicators

### Task 5: Conductor-origin badge in pr-detail.ts

**Files:**
- Modify: `app/src/main/webui/src/components/pr-detail.ts`

**Interfaces:**
- Consumes: `GET /api/devtown/evolution/conductor-origin/{caseId}` → `{ conductorDriven: boolean, streamCategory?: string }` (from Task 3)
- Produces: Visual badge on conductor-driven PRs with link to `/evolution`

- [ ] **Step 1: Add conductor-origin state property**

In `pr-detail.ts`, add to the `@state()` properties (after line 77):

```typescript
@state() private _conductorOrigin: { conductorDriven: boolean; streamCategory?: string } | null = null;
```

- [ ] **Step 2: Fetch conductor-origin in _fetchDetail()**

At the end of `_fetchDetail()` (after the main fetch succeeds, around line 140), add a non-blocking fetch:

```typescript
fetch(`/api/devtown/evolution/conductor-origin/${this.caseId}`)
  .then(r => r.ok ? r.json() : null)
  .then(data => { this._conductorOrigin = data; })
  .catch(() => { /* badge simply won't render */ });
```

- [ ] **Step 3: Add badge rendering in render()**

After the header line (around line 259, after the `PR #${d.pr.prNumber}` heading), add:

```typescript
${this._conductorOrigin?.conductorDriven ? html`
  <a href="/evolution" class="conductor-badge"
     title="Created by evolution conductor (${this._conductorOrigin.streamCategory})">
    <span class="badge-icon">⚡</span> Conductor
  </a>
` : ''}
```

- [ ] **Step 4: Add badge styles**

Add to the component's `static styles` (in the `css` template literal):

```css
.conductor-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 12px;
  background: var(--surface-accent, #e8f0fe);
  color: var(--text-accent, #1a73e8);
  font-size: 12px;
  font-weight: 500;
  text-decoration: none;
  margin-left: 8px;
  vertical-align: middle;
}
.conductor-badge:hover {
  background: var(--surface-accent-hover, #d2e3fc);
}
.badge-icon {
  font-size: 14px;
}
```

- [ ] **Step 5: Run TypeScript type check**

Run: `npm run typecheck --prefix app/src/main/webui`
Expected: No type errors

- [ ] **Step 6: Commit**

```bash
git add app/src/main/webui/src/components/pr-detail.ts
git commit -m "feat(#268): add conductor-origin badge to PR detail

Shows a 'Conductor' badge with link to /evolution when a PR was created
by the evolution conductor. Non-blocking fetch — badge doesn't render
if the endpoint fails.
Refs casehubio/devtown#268"
```

---

### Task 6: Query param deep link support in review dashboard

**Files:**
- Modify: `app/src/main/webui/src/views/review.ts` (already created in Task 4)
- Modify: `app/src/main/webui/src/components/pr-cases-workbench.ts` (if it exists) or `app/src/main/webui/src/index.ts`

**Interfaces:**
- Consumes: `window.location.search` query params (`?caseId=X`)
- Produces: Review dashboard auto-selects the specified case on load

- [ ] **Step 1: Verify query param is already wired**

Task 4 already wires `initialCaseId` from URL search params into `mountReviewDashboard()` and passes it to `hostPanel("pr-cases-workbench", { initialCaseId })`. Verify this is correctly passed through by reading the `pr-cases-workbench` component.

- [ ] **Step 2: Check pr-cases-workbench component accepts initialCaseId**

Read `app/src/main/webui/src/components/pr-cases-workbench.ts` (or wherever the workbench is defined). If it doesn't accept `initialCaseId` as a property, add it:

```typescript
@property() initialCaseId?: string;
```

And in `connectedCallback()` or `firstUpdated()`, if `initialCaseId` is set, trigger case selection.

- [ ] **Step 3: Verify deep link flow end-to-end**

The evolution workbench's streams tab renders `deepLink` values (from Task 2) as clickable links. Clicking navigates to `/?caseId={improvementCaseId}`. The review dashboard reads the param and passes it to the PR Cases workbench.

- [ ] **Step 4: Run TypeScript type check**

Run: `npm run typecheck --prefix app/src/main/webui`
Expected: No type errors

- [ ] **Step 5: Run full Maven build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/webui/src/
git commit -m "feat(#268): support caseId query param for deep link navigation

Review dashboard reads ?caseId= from URL and passes to PR Cases workbench
for auto-selection. Enables evolution → review deep linking.
Refs casehubio/devtown#268"
```

---

## References

- [2026-10-27-evolution-standalone-app-design.md] — design spec this plan implements
- `app/src/main/webui/src/index.ts` — current single-page entry point
- `app/src/main/webui/src/components/pr-detail.ts` — PR detail component (~409 lines)
- `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionBootstrap.java` — bootstrap (no try-catch)
- `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionResource.java` — REST resource (13 endpoints)
- `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionApi.java` — application service
- `app/src/main/java/io/casehub/devtown/app/evolution/DevtownImprovementStreamView.java` — record (2 fields)
- `app/src/main/java/io/casehub/devtown/app/evolution/DevtownEvolutionEnricher.java` — stream enrichment
- `.casehub-packages/packages/evolution-workbench/` — pre-built workbench element
- `.casehub-packages/packages/evolution-config/` — API client + editors
- `app/src/main/resources/application.properties:152` — SPA routing already enabled
- `app/src/main/resources/application.properties:155` — auth permission paths
- casehubio/devtown#268 — restructure evolution as standalone app
- casehubio/devtown#230 — surface evolution conductor UI (superseded)
