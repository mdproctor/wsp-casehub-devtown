## D1: Client-side routing approach

**Choice:** Path-based dispatch in index.ts — check `window.location.pathname` before `loadSite()`, mount review dashboard or evolution workbench conditionally. Single bundle, single HTML file, shared auth/theme/preferences setup. Add Quinoa wildcard fallback for SPA routing.

**Alternatives:**
- Separate HTML entry points + esbuild entries — true code splitting but duplicates auth/preferences/theme setup, two HTML files to maintain, more moving parts for marginal bundle size benefit in an internal tool
- Hash-based routing (`#/evolution`) — zero server config but ugly URLs, doesn't match the `/evolution` path requirement, non-standard

**Rationale:** Simplest change that delivers clean `/evolution` URL. Bundle splitting isn't worth the complexity for an internal tool with 40+ mosaic packages already in the bundle. Both apps need the same auth flow and theme setup.

**Trade-offs:** Both apps' code ships in one bundle — acceptable for an internal tool where the bundle is already large.

**Sources:** `app/src/main/webui/src/index.ts`, `app/src/main/webui/esbuild.config.mjs`

**Exploration:** quick

**Status:** captured

## D2: Evolution page shell

**Choice:** Workbench only — the evolution workbench (`<agency-evolution-workbench>`) fills the viewport directly. A minimal nav element (breadcrumb or link) at the top for switching back to the review dashboard.

**Alternatives:**
- Full page shell — `page()`/`tabs()` wrapper consistent with review dashboard but adds redundant chrome over the workbench's own tabs
- Shared top nav — persistent navigation bar above both apps, but adds a permanent visual element that neither app currently has

**Rationale:** The workbench already has its own tab structure (summary, streams, inbox, audit, config, health). Wrapping it in another `page()` with `tabs()` would double the tab chrome. A minimal nav link preserves the clean full-page experience while keeping navigation discoverable.

**Trade-offs:** No persistent visual relationship between the two apps beyond the nav link. Acceptable — the issue explicitly describes them as peer experiences.

**Sources:** `.casehub-packages/packages/evolution-workbench/` (built-in tabs), issue #268 design decision

**Exploration:** quick

**Status:** captured

## D3: Cross-app deep linking direction

**Choice:** Full page navigation — clicking a stream target in evolution navigates to `/?tab=pr-cases&caseId=X` (or similar URL scheme). Uses existing pr-detail rendering in the review dashboard.

**Alternatives:**
- Panel overlay within evolution — richer context but requires embedding review components in evolution, violates the one-way dependency (evolution reads review's event log, review doesn't know evolution exists)
- New browser tab — preserves evolution state but splits user attention, poor UX for frequent navigation

**Rationale:** Clean separation between experiences. The review dashboard already renders PR detail — no need to duplicate that rendering in evolution. One-way dependency means evolution can link to review but not embed it.

**Trade-offs:** Navigating away from evolution loses context (current tab, scroll position). Browser back button returns to evolution. Acceptable for the stated one-way dependency model.

**Sources:** Issue #268 design decision, `app/src/main/webui/src/components/pr-detail.ts`

**Exploration:** quick

**Status:** captured

## D4: Conductor-origin indicator

**Choice:** Origin badge with link — a small badge/tag showing "Conductor" on pr-detail with a link to `/evolution`. Signals provenance without cluttering the view.

**Alternatives:**
- Badge + stream detail — includes which improvement stream and category drove the PR, more context but adds visual weight
- Badge only — no link, no detail, minimal intrusion but loses the navigation path back to evolution

**Rationale:** The badge conveys provenance at a glance. The link enables navigation to the evolution context when the user wants to understand why this PR was created. Stream detail would clutter the already-dense pr-detail component.

**Trade-offs:** Requires a new backend endpoint (`GET /api/devtown/evolution/conductor-origin/{caseId}`) to determine if a PR case was conductor-driven. Minimal overhead — single query.

**Sources:** `app/src/main/webui/src/components/pr-detail.ts`, issue #268

**Exploration:** quick

**Status:** captured
