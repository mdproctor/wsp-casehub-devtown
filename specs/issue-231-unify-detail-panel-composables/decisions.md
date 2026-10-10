# Design Decisions — #231 Unify Detail Panel Composables

## D1: Composable scope — devtown-local

**Choice:** Keep all shared composables in devtown's `src/components/shared/`
**Alternatives:**
- Blocks-ui first — cross-app reuse, but requires release cycle and components are domain-specific
- Hybrid (generic to blocks-ui, domain to devtown) — premature split, no second consumer exists
**Rationale:** These render devtown-specific domain concepts (contributor trust, review outcomes). No other app needs them yet. Avoids release cycle overhead.
**Trade-offs:** If a second casehub app needs trust bars, we'll extract then.
**Sources:** blocks-ui-trust-score-panel already covers the generic case; issue #231 body
**Exploration:** quick
**Status:** captured

## D2: Composition pattern — render functions + shared styles

**Choice:** Plain functions returning `TemplateResult` with a shared CSS module
**Alternatives:**
- Custom elements (`@customElement`) — fully encapsulated but Shadow DOM boundary overhead, over-engineered for fragments always used inside detail panels
- Mixins — no boundary overhead but fragile at scale, name collision risk
**Rationale:** These are rendering fragments, not standalone components. Always consumed inside detail panel Shadow DOMs. Functions compose naturally (like `renderSparkline` in blocks-ui-core). Simplest approach with no unnecessary abstraction.
**Trade-offs:** Can't be used standalone in HTML; no isolated testing via DOM. If we later need standalone use, we'd wrap in a custom element.
**Sources:** blocks-ui-core `renderSparkline` pattern; Lit documentation on composition
**Exploration:** quick
**Status:** captured

## D3: Cross-reference fetch in scope

**Choice:** Include the `reviewDetail(caseId)` fetch in merge-queue PR detail as part of this issue
**Alternatives:**
- Defer to follow-up issue — pure extraction first
**Rationale:** Issue #231 explicitly calls out "Backend data needed" — the cross-reference is the main new capability beyond mechanical extraction.
**Trade-offs:** Slightly larger scope, but the mock server already has the data fixtures.
**Sources:** Issue #231 body "Backend data needed" section; mock-fixtures.json reviewer endpoints
**Exploration:** quick
**Status:** captured
