# Evolution Standalone App — Design Spec

**Issue:** casehubio/devtown#268, casehubio/devtown#230
**Branch:** issue-268-evolution-standalone-app
**Date:** 2026-10-27

## Summary

Restructure the evolution conductor UI from a planned dashboard tab (#230) into a standalone page at `/evolution` (#268). Two peer frontend experiences served from one Quarkus app, one esbuild bundle, one HTML entry point. The dependency is one-way: evolution reads review's event log; review doesn't know evolution exists — except for a small conductor-origin badge on PRs driven by the conductor.

## Architecture

### Routing

Path-based dispatch in `index.ts`. Before calling `loadSite()`, check `window.location.pathname`:

- `/evolution` (or `/evolution/...`) → mount evolution workbench
- Everything else → mount review dashboard (existing behaviour)

Both paths share auth (`/api/userinfo`), preferences fetch, and theme setup. Single esbuild entry point, single HTML file, single JS bundle.

**Quinoa SPA fallback:** Add `quarkus.quinoa.enable-spa-routing=true` in `application.properties` so that Quinoa serves `index.html` for unknown paths (both `/` and `/evolution` resolve to the same HTML, then JS dispatches).

### Frontend — Evolution Page

`views/evolution.ts` exports a function that:

1. Creates a minimal nav element — a back-link to `/` ("← DevTown Review")
2. Creates `<agency-evolution-workbench>` configured with:
   - `endpoint="/api/devtown/evolution"`
   - `tenancyId` from user info
   - Theme from system preference
3. Returns the container element for mounting

The workbench fills the viewport below the nav link. It provides its own tabs (summary, streams, inbox, audit, config, health) — no `page()`/`tabs()` wrapper needed.

### Frontend — Review Dashboard Refactor

Extract the existing review dashboard setup from `index.ts` into `views/review.ts`. The review view exports a function that builds and returns the `page()` with its 6 tabs, exactly as today.

`index.ts` becomes a dispatcher:

```
start() {
  // auth, preferences, theme (shared)
  if (pathname starts with /evolution) {
    mount evolution view
  } else {
    mount review view via loadSite()
  }
}
```

### Frontend — Package Wiring

Add to `package.json` dependencies:

```json
"@casehubio/mosaic-evolution-workbench": "file:.casehub-packages/packages/evolution-workbench",
"@casehubio/mosaic-evolution-config": "file:.casehub-packages/packages/evolution-config"
```

Run `npm install` — the `postinstall` script (`link-casehub-packages.mjs`) handles node_modules symlinks automatically.

### Frontend — Conductor-Origin Badge

In `pr-detail.ts`, after loading case details, fetch `GET /api/devtown/evolution/conductor-origin/{caseId}`. If the response indicates conductor origin:

- Render a small badge/tag: "Conductor" with an icon
- Badge links to `/evolution`
- Position: near the PR title/header area, consistent with existing badge styling

The fetch is fire-and-forget — if it fails or returns no origin, the badge simply doesn't render. No impact on existing PR detail rendering.

### Frontend — Cross-App Deep Links

The `DevtownImprovementStreamView` gains a `deepLink` field populated by the backend enricher. The evolution workbench's streams tab renders these as clickable links. Clicking navigates to `/?caseId={prCaseId}` — full page navigation to the review dashboard with the PR case selected.

The review dashboard's `start()` function reads `caseId` from URL query params and passes it to the PR Cases workbench for auto-selection. This is a new capability — today the dashboard doesn't read query params.

## Backend Changes

### DevtownEvolutionBootstrap — Startup Safety

Wrap `startCase()` in try-catch:

```java
try {
    caseHub.startCase();
} catch (Exception e) {
    log.error("Evolution case bootstrap failed — evolution features unavailable", e);
}
```

The app starts normally; evolution endpoints return appropriate errors if the case wasn't created.

### Conductor-Origin Endpoint

New endpoint: `GET /api/devtown/evolution/conductor-origin/{caseId}`

Returns a simple response indicating whether the given PR case was created by the evolution conductor, and if so, which improvement stream drove it.

Response shape:

```json
{
  "conductorDriven": true,
  "streamCategory": "ci-reliability"
}
```

Or `{ "conductorDriven": false }` for non-conductor PRs.

Implementation: query the evolution case's event log for a case-creation event referencing the given `caseId` as a target.

### DevtownImprovementStreamView — Deep Link Field

Add `@Nullable String deepLink` to the record. The `DevtownEvolutionEnricher` populates it with `/?caseId={prCaseId}` when the stream's target resolves to a known PR case.

## Testing

### Backend

- `DevtownEvolutionBootstrapTest` — verify app starts when `startCase()` throws
- `DevtownEvolutionResourceTest` — test conductor-origin endpoint (conductor-driven case, non-conductor case, unknown case)
- `DevtownEvolutionEnricherTest` — verify `deepLink` field populated for streams with known PR case targets

### Frontend

- Manual verification: navigate to `/evolution`, verify workbench loads with all tabs
- Manual verification: navigate to `/`, verify review dashboard unchanged
- Manual verification: click deep link in evolution streams, verify navigation to review dashboard
- Manual verification: verify conductor badge appears on conductor-driven PRs in pr-detail
- TypeScript type check: `npm run typecheck` passes

## File Changes

| File | Change |
|------|--------|
| `app/src/main/webui/package.json` | Add evolution-workbench + evolution-config deps |
| `app/src/main/webui/src/index.ts` | Path-based dispatch, extract review setup |
| `app/src/main/webui/src/views/evolution.ts` | New — evolution page with workbench |
| `app/src/main/webui/src/views/review.ts` | New — extracted review dashboard setup |
| `app/src/main/webui/src/components/pr-detail.ts` | Add conductor-origin badge |
| `app/src/main/resources/application.properties` | Add Quinoa SPA routing config |
| `app/src/main/java/.../evolution/DevtownEvolutionBootstrap.java` | Wrap startCase() in try-catch |
| `app/src/main/java/.../evolution/DevtownEvolutionResource.java` | Add conductor-origin endpoint |
| `app/src/main/java/.../evolution/DevtownEvolutionApi.java` | Add conductor-origin lookup method |
| `app/src/main/java/.../evolution/DevtownImprovementStreamView.java` | Add deepLink field |
| `app/src/main/java/.../evolution/DevtownEvolutionEnricher.java` | Populate deepLink |

## Scope Boundaries

**In scope:**
- Package wiring, evolution page, path routing, bootstrap fix, conductor-origin endpoint + badge, deep link enrichment

**Out of scope:**
- Evolution workbench internal changes (already built in casehub-mosaic)
- Review dashboard changes beyond the conductor badge
- Server-side routing or reverse proxy config
- Bundle splitting / code splitting optimisation

## References

- `app/src/main/webui/src/index.ts` — current single-page setup
- `app/src/main/webui/src/components/pr-detail.ts` — PR detail component
- `.casehub-packages/packages/evolution-workbench/` — pre-built workbench
- `.casehub-packages/packages/evolution-config/` — API client + editors
- `app/src/main/java/io/casehub/devtown/app/evolution/` — backend evolution classes
- casehubio/engine#1225 — design decision for standalone approach
- casehubio/engine#1180 — domain SPIs, API facade
