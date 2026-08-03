# Pages Upgrade + OKLCH Theme Migration

**Issue:** devtown#172
**Branch:** `issue-172-pages-upgrade-oklch-theme`
**Date:** 2026-07-22

## Problem

devtown's frontend is on pages-runtime/pages-ui 0.2.0, which uses the old hex-based `CasehubTheme` system (`--casehub-*` CSS variables). The current pages source uses the OKLCH-based token system (`--pages-*` CSS variables via `pages-ui-tokens`) — 12-step neutral scales, surface overlays, proper shadow depth, alternating row shading. The visual gap between devtown and blocks-ui is a fundamental theme system difference, not fixable with CSS tweaks.

Additionally, the `dataset()` helper function used by all 13 datasets was removed from pages-ui. The `loadSite()` and `setTheme()` APIs changed signatures.

## Prerequisites (complete)

- **File-based npm deployment** — parent#392 shipped. All `@casehubio/*` npm packages are consumed via `file:` references pointing to sibling repo source directories. Published versions are used only for actual releases.
- **npm BOM enforcer** — parent#392 shipped. Version alignment enforced at release time across all casehub repos.

## Design

### 1. package.json — add new file-based dependencies

Parent's `340536a` already converted `pages-runtime` and `pages-ui` to `file:` references. Two new dependencies needed:

| Package | Purpose |
|---------|---------|
| `@casehubio/pages-ui-tokens` | `ThemeConfig`, `DEFAULT_THEME`, `injectTheme()`, `applyThemeMode()` |
| `@casehubio/pages-data` | `dataSetId()`, `ExternalDataSetDef` type |

Added as `file:` references following the same `../../../../../pages/packages/<pkg>` pattern.

blocks-ui file references are also added to prepare for blocks-ui#41 Phase 1 (component migration). The specific blocks-ui packages will be determined by the blocks-ui audit (in progress).

### 2. datasets.ts — replace `dataset()` with `ExternalDataSetDef` objects

The removed `dataset()` helper was sugar for constructing `ExternalDataSetDef`. Direct replacement:

**Before:**
```ts
import { dataset } from "@casehubio/pages-ui";
export const datasets = [
  dataset("queue-status", "/api/governance/queue-status", { dataPath: "reviews" }),
];
```

**After:**
```ts
import type { ExternalDataSetDef } from "@casehubio/pages-data";
import { dataSetId } from "@casehubio/pages-data";

export const datasets: ExternalDataSetDef[] = [
  { uuid: dataSetId("queue-status"), url: "/api/governance/queue-status", dataPath: "reviews" },
];
```

All 13 datasets migrated 1:1. No behavioral change. The `#{row.caseId}` URL template syntax is a native pages feature (`resolveTemplate()` in `pages-component`, resolved by `data-pipeline.ts` in `pages-runtime`) — works unchanged in `ExternalDataSetDef` URLs.

### 3. theme.ts — OKLCH token system

**Before:**
```ts
import { DARK_THEME, LIGHT_THEME } from "@casehubio/pages-runtime";
export const themes = { light: LIGHT_THEME, dark: DARK_THEME };
export type ThemeMode = keyof typeof themes;
```

**After:**
```ts
import { DEFAULT_THEME, type ThemeConfig } from "@casehubio/pages-ui-tokens";
export const themeConfig: ThemeConfig = DEFAULT_THEME;
```

Dark/light is now a mode string passed to `setTheme()`, not a separate theme object. The `ThemeConfig` controls hue, chroma, and contrast — the OKLCH color scales are generated from these parameters for both light and dark modes.

### 4. index.ts — new loadSite + setTheme API

**Before:**
```ts
loadSite(container, app).then(site => {
  site.setTheme(themes[initialMode]);
  // listener passes theme object
});
```

**After:**
```ts
import { themeConfig } from "./theme";

loadSite(container, app, { themeConfig }).then(site => {
  site.setTheme(prefersDark ? "dark" : "light");

  window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", (e) => {
    site.setTheme(e.matches ? "dark" : "light");
  });
});
```

`loadSite()` now accepts `themeConfig` in its options — it internally calls `injectTheme()` + `applyThemeMode()`. `setTheme()` takes `"light" | "dark"` string, not a theme object.

### 5. index.html — CSS token migration

**Before:**
```css
#app {
  font-family: var(--casehub-font, system-ui, sans-serif);
  background: var(--casehub-bg, #fff);
  color: var(--casehub-text, #333);
}
```

**After:**
```css
#app {
  min-height: 100%;
  font-family: var(--pages-font-family, system-ui, sans-serif);
  background: var(--pages-neutral-1, #fafafa);
  color: var(--pages-neutral-12, #111);
}
```

All `--casehub-*` variables → `--pages-*` OKLCH tokens. Fallback values provide reasonable defaults before theme injection runs.

### 6. View files — no changes

The 7 view files (`views/*.ts`) use pages-ui DSL functions (`page`, `table`, `metric`, `rows`, `columns`, `title`, `lookup`, `groupBy`, `col`). These APIs are unchanged in the current pages source. No view file modifications needed for this issue.

blocks-ui component migration (replacing `table()` → `<pages-data-table>`, `metric()` → `<kpi-metric-row>`, etc.) is blocks-ui#41 Phase 1 — separate scope.

## Not in scope (deferred with tracked issues)

| Deferred item | Issue | Rationale |
|--------------|-------|-----------|
| npm BOM enforcer infrastructure | parent#392 | Done — shipped by parent |
| `applyTheme()` should set background/color on target | casehub-pages#226 | pages-repo fix — devtown works around it in index.html |
| Dataset enhancements (refresh, WebSocket, SSE) | devtown#173 | Additive after base upgrade |
| blocks-ui component migration (Phase 1) | blocks-ui#41 | Separate epic — views rewritten to consume blocks-ui web components |

## Testing

- `npm run typecheck` — all imports resolve, no type errors
- `npm run build` — esbuild bundles successfully
- `mvn clean install -pl app` — Quinoa build passes
- Visual: dark/light mode toggle works, theme applies OKLCH token scales
- Visual: all 6 governance views render data correctly
- Visual: `#{row.caseId}` parameterised datasets load case-specific data
