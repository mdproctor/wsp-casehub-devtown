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
