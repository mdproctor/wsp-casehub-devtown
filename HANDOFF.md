# Handoff — devtown housekeeping + workspace fixes

2026-05-13

## What shipped this session

**devtown#21** — `CapabilityRegistry.isKnown()` promoted to SPI default method. `DevtownCapabilityRegistry` override removed. `CapabilityRegistrySpiTest` added — anonymous impl proves the contract is on the interface. 72 tests passing.

**devtown#22** — assertj-core bumped to 3.27.7. Test-scope only.

**Workspace/project split fixes** — discovered systemic problems; partially fixed:
- Git Discipline section added to devtown CLAUDE.md: always `git rev-parse --show-toplevel` before any git op; source code commits → project repo, methodology → workspace
- Routing tables fixed for devtown, aml, clinical: adr/blog/specs → project, design/snapshots/handover → workspace. Foundation repos deferred (active sessions).
- Design Document Convention added to devtown CLAUDE.md: design doc is `design/JOURNAL.md` (created by `epic`); between epics, skip DESIGN.md sync entirely.
- Development Workflow section updated: `work-start` → brainstorm → TDD → `java-dev` → review → `implementation-doc-sync`.
- Specs promoted: Epic 1 + 2 specs copied to `docs/specs/` in project repo.

**Skills Claude is fixing** (deferred, not in this session):
- `workspace-init` — bake in git discipline + correct routing defaults
- `java-git-commit` — read design doc path from CLAUDE.md, skip gracefully between epics
- `publish-blog` — source path mismatch (`docs/_posts/` vs `blog/`), missing `blog_router.py`

## Active follow-ups

| Issue | What | Priority |
|-------|------|----------|
| devtown#19 | Dependency tracker for ledger#76 (CAPABILITY_DIMENSION) | Passive |
| devtown#20 | CaseOperation naming/placement review | Epics 4/5 |
| parent#14 | Trust maturity model → PLATFORM.md | Parent session |

## New idea logged

Open source contributor trust: DevTown's trust model (Bayesian Beta, EigenTrust, `RoutingPolicy`) extended to model contributor trust from PR outcomes. Vouching by high-trust contributors for newcomers. Spec for stakeholder buy-in to be written in a future session.
See: `IDEAS.md`

## Next: Epic 3

**PR Review CasePlanModel** (devtown#10) — content-driven routing and parallel checks.

Pre-conditions all met:
- Domain vocabulary from Epic 2 ✅
- Foundation P0.1, P0.2 ✅
- Foundation gaps: P1.3 (TrustWeightedSelectionStrategy) and HITL wiring — known, not blocking design

Start with `work-start` → brainstorming → plan → implement.

## Key references

- Protocol added: `parent/docs/protocols/spi-default-method-contract-test.md`
- Blog: `blog/2026-05-13-mdp01-contributor-trust-open-source.md`
- Foundation gates + improvements: `docs/PROGRESS.md` in project repo
