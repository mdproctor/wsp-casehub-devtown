*Updated: #119, #171 closed — removed from backlog. connectors#86 and blocks-ui#89 closed — cross-module blockers cleared.*

# HANDOFF — 2026-07-22

## Last Session

Fixed governance UI (#171): all six views had col() references that didn't match the actual GovernanceQueryService REST response field names. Fixed every mismatch across operations, reviews, queue, reviewers, triage, and system views. Added dark mode theme wiring (prefers-color-scheme with live switching). Landed as d147795 on main.

Discovered a deeper issue: devtown is on pages-runtime/pages-ui 0.2.0 (old hex-based `CasehubTheme` with `--casehub-*` variables), while the current pages source (0.2.3) uses the OKLCH token system (`--pages-*` variables via `pages-ui-tokens`) that blocks-ui uses. This is why the UI looks flat — no grey scales, no alternating rows, no surface depth. Filed #172.

## Immediate Next Step

**Do #172** — upgrade pages-runtime/pages-ui to 0.2.3 and implement npm BOM.

### What #172 requires

1. **Publish pages 0.2.3 to GitHub Packages** — packages exist in pages monorepo source (`/Users/mdproctor/claude/casehub/pages/packages/`) but aren't published. The npm auth token for GitHub Packages wasn't working this session (401 on `npm install`). Fix auth or use `npm pack` + local install.

2. **npm BOM** — no dep should be locally defined. Version should be defined once. Two options explored:
   - `@casehubio/pages-bom` package that coordinates all pages packages at one version
   - Shared version catalog matching the Maven `<dependencyManagement>` pattern

3. **Adapt to removed `dataset()` API** — the `dataset()` helper function was removed between 0.2.0 and 0.2.3. devtown's `datasets.ts` uses it for all 12 datasets. Current pages uses YAML-based dataset definitions or a different programmatic approach. Check pages examples for the new pattern.

### Critical finding: `file:` symlinks don't work with esbuild

The current `file:` references in package.json create npm symlinks into the pages monorepo. This causes a fundamental esbuild resolution problem:

- **Without `preserveSymlinks`** (current): esbuild follows symlinks to the pages monorepo and resolves transitive deps from the pages root `node_modules`. This mostly works, BUT `pages-ui-tokens` (the theme system) gets silently dropped — `injectTheme()` and `generateThemeCSS()` are missing from the bundle. The app loads and renders but dark mode doesn't switch because no CSS variables are generated.

- **With `preserveSymlinks: true`**: esbuild resolves from devtown's `node_modules`, which is correct for `@casehubio/*` packages. But ALL transitive deps (`pages-component`, `pages-viz`, `pages-table`, `zod`, `js-yaml`, `jsonata`, `marked`, `lit`, `echarts`) fail to resolve because they aren't installed in devtown's node_modules — they live in the pages monorepo's hoisted node_modules.

**The fix must be one of:**
- Publish properly to GitHub Packages (eliminates symlinks entirely)
- Use `npm pack` in each pages package, then `npm install ./pages-runtime-0.2.3.tgz` (packs flatten workspace deps)
- Add ALL transitive deps to devtown's package.json as explicit `file:` references (fragile, defeats the purpose of a BOM)

**How to verify theme injection works:** In the browser console, `document.querySelector('style[data-pages-theme]')` should return a `<style>` element with ~5000+ chars containing both `.pages-theme-light` and `.pages-theme-dark` rulesets. If it returns null, the theme system isn't bundled.

4. **Adapt theme code** — old API vs new API:
   - `DARK_THEME` / `LIGHT_THEME` / `CasehubTheme` → `ThemeConfig` + `DEFAULT_THEME` (from `pages-ui-tokens`)
   - `applyTheme(element, theme)` → `injectTheme(config, target)` + `applyThemeMode(element, mode)`
   - `site.setTheme(themeObj)` → `site.setTheme("light" | "dark")`

5. **Fix `applyTheme()` in pages-viz** — it sets CSS custom properties but doesn't set `background`/`color` on the element itself. Every consumer has to add boilerplate CSS. This should be fixed in the library so it just works.

6. **Update index.html** — follow blocks-ui pattern:
   ```css
   body { background: var(--pages-neutral-1, #fafafa); color: var(--pages-neutral-12, #111); }
   ```

### Key file locations

| File | What |
|------|------|
| `/Users/mdproctor/claude/casehub/pages/packages/pages-ui-tokens/src/themes.ts` | New OKLCH theme system — `ThemeConfig`, `injectTheme()`, `applyThemeMode()`, `generateThemeCSS()` |
| `/Users/mdproctor/claude/casehub/pages/packages/pages-runtime/src/site.ts` | `loadSite()` — line 147 calls `injectTheme()`, line 148 calls `applyThemeMode()` |
| `/Users/mdproctor/claude/casehub/pages/packages/pages-viz/src/base/theme.ts` | Old theme system (removed from source, only in published 0.2.0 dist) |
| `/Users/mdproctor/claude/casehub/blocks-ui/examples/index.html` | Reference for body CSS with `--pages-neutral-*` tokens |
| `/Users/mdproctor/claude/casehub/blocks-ui/examples/src/shell.ts` | Reference for light/dark toggle button |
| `/Users/mdproctor/claude/casehub/devtown/app/src/main/webui/src/datasets.ts` | 12 dataset definitions using removed `dataset()` API |
| `/Users/mdproctor/claude/casehub/devtown/app/src/main/webui/src/theme.ts` | Currently uses old `DARK_THEME`/`LIGHT_THEME` — needs rewrite |
| `/Users/mdproctor/claude/casehub/devtown/app/src/main/webui/src/index.ts` | `loadSite()` call — needs `themeConfig` option |
| `/Users/mdproctor/claude/casehub/devtown/app/src/main/webui/.npmrc` | Uses `NODE_AUTH_TOKEN` env var for GitHub Packages auth |

### Transitive dependency chain

pages-runtime 0.2.3 depends on (all `workspace:*`): pages-component, pages-data, pages-table, pages-ui, pages-ui-tokens, pages-viz. These in turn need: lit, echarts, zrender, zod, jsonata, js-yaml, marked, tslib. When installing from `file:` refs, npm doesn't resolve `workspace:*` transitive deps — you need to either publish properly or add all transitives to devtown's package.json.

## Running the Demo

### Start the app

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev -pl app -Dsurefire.skip=true
```

Wait ~45 seconds. Confirm ready: `curl -s http://localhost:8080/q/health/ready` should return `{"status":"UP"}`.

The UI is at http://localhost:8080 — seven tabs (Operations, Reviews, Merge Queue, Reviewers, Triage, System, Definitions).

### Submit a PR to populate data

The webhook secret in dev mode is `demo`. Compute the HMAC and POST:

```bash
BODY='{"action":"opened","number":42,"repository":{"full_name":"casehubio/devtown","name":"devtown","owner":{"login":"casehubio"}},"pull_request":{"number":42,"title":"feat: add caching layer","draft":false,"merged":false,"head":{"sha":"abc123def456","ref":"feature/cache"},"base":{"ref":"main"},"user":{"login":"demo-dev"},"additions":120,"deletions":30,"changed_files":5},"sender":{"login":"demo-dev"}}'

SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "demo" | awk '{print $2}')

curl -s -X POST http://localhost:8080/api/github/webhook \
  -H 'Content-Type: application/json' \
  -H 'X-GitHub-Event: pull_request' \
  -H "X-Hub-Signature-256: sha256=$SIG" \
  -d "$BODY"
```

Expected response: `{"action":"case-started","status":"accepted"}`

### Verify

- Reload http://localhost:8080 — Operations tab shows PR #42 in Active Reviews table, event in Event Stream
- System tab shows Active Cases: 1
- Other tabs show EMPTY_RESULT (correct — no merge queue entries, no reviewers with trust scores, no triage items)

### Governance API endpoints (all @PermitAll in dev mode)

| Endpoint | Returns |
|----------|---------|
| `GET /api/governance/queue-status` | Active reviews with status counts |
| `GET /api/governance/system-health` | Fleet size, trust averages, open commitments |
| `GET /api/governance/recent-events?limit=100` | Event stream |
| `GET /api/governance/problems` | Stalled cases, expired commitments, worker failures |
| `GET /api/governance/reviewers` | Reviewer fleet with trust scores |
| `GET /api/governance/merge-queue` | Queued PRs and active batches |
| `GET /api/governance/merge-queue/metrics` | Queue depth, wait times, throughput |
| `GET /api/governance/triage` | Pending human decisions |

### Known limitations (current state)

- Theme uses old pages-viz 0.2.0 hex system — flat white/dark, no grey scales or surface depth (#172 fixes this)
- `POST /api/reviews` returns 500 in dev mode (use webhook path instead)
- Some components in dark mode may show UNKNOWN_COLUMN for fields that reference nested Map values (trustByCapability) — these can't be flattened with the current pages-ui col() API

## Active Work Slots

| Slot | Branch | Issue | Repos | What to do |
|------|--------|-------|-------|------------|
| 8 | `issue-89-trust-workbench` | blocks-ui#89 | blocks-ui | Build trust-workbench composite |
| 11 | `issue-123-worker-session-mgmt` | devtown#123 | devtown, engine | Implement EntityStateContributor SPIs |

## Cross-Module

**Blocked by:**
- `platform` — SubscriptionEngine + NotificationDispatcher · L · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #172 | pages 0.2.3 upgrade + npm BOM | M | Med | OKLCH theme, dataset API migration, BOM |
| #98 | Trust visibility UI | S | Low | blocks-ui#89 closed — unblocked |
| #120 | Case dependency graph | M | Med | Needs blocks-ui issue filed |
| #123 | Worker session mgmt UI | M | Med | Slot 11 |

## References

- Issue #172: https://github.com/casehubio/devtown/issues/172
- CasePlanModel browser spec: `specs/2026-07-21-caseplanmodel-browser-design.md`
