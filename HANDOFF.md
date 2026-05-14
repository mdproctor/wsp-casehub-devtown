# Handoff — devtown housekeeping + contributor trust spec

2026-05-14

## What shipped this session

**devtown#21** — `CapabilityRegistry.isKnown()` promoted to SPI default method. `DevtownCapabilityRegistry` override removed. `CapabilityRegistrySpiTest` added — anonymous impl proves the contract is on the interface. 72 tests passing.

**devtown#22** — assertj-core bumped to 3.27.7. Test-scope only.

**Workspace/project split fixes** — discovered systemic problems; partially fixed:
- Git Discipline section added to devtown CLAUDE.md
- Routing tables fixed for devtown, aml, clinical: adr/blog/specs → project, design/snapshots/handover → workspace. Foundation repos deferred (active sessions).
- Design Document Convention added to devtown CLAUDE.md: skip DESIGN.md sync between epics
- Development Workflow updated: `work-start` → brainstorm → TDD → `java-dev` → review → `implementation-doc-sync`
- Specs promoted: Epic 1 + 2 specs copied to `docs/specs/`

**Contributor trust spec** — devtown#24 created and fully written:
- Full spec: `specs/2026-05-13-contributor-trust-open-source.md`
- TL;DR: `docs/contributor-trust-proposal.md` — tables/bullets, audience is engineers unfamiliar with CaseHub
- GitHub issue: devtown#24 — TL;DR first, full spec in collapsible section
- IDEAS.md updated: status → promoted, Promoted to → devtown#24
- Key arguments: Bayesian Beta for confidence-weighted scoring, EigenTrust for vouching propagation, history mining for day-one bootstrapping, work queues as the practical routing layer, fits DevTown as intake side of existing reviewer-routing pipeline
- Ghostty comparison: Ghostty approach (contribution guidelines) is step one, not an alternative — works until scale/adversarial pressure breaks it

**Skills Claude is fixing** (deferred):
- `workspace-init` — bake in git discipline + correct routing defaults
- `java-git-commit` — read design doc path from CLAUDE.md, skip gracefully between epics
- `publish-blog` — source path mismatch (`docs/_posts/` vs `blog/`), missing `blog_router.py`

## Active follow-ups

| Issue | What | Priority |
|-------|------|----------|
| devtown#24 | Contributor trust spec — awaiting colleague feedback | Active |
| devtown#19 | Dependency tracker for ledger#76 (CAPABILITY_DIMENSION) | Passive |
| devtown#20 | CaseOperation naming/placement review | Epics 4/5 |
| parent#14 | Trust maturity model → PLATFORM.md | Parent session |

## Next: Epic 3

**PR Review CasePlanModel** (devtown#10) — content-driven routing and parallel checks.

Pre-conditions all met:
- Domain vocabulary from Epic 2 ✅
- Foundation P0.1, P0.2 ✅
- Foundation gaps: P1.3 (TrustWeightedSelectionStrategy) and HITL wiring — known, not blocking design

Start with `work-start` → brainstorming → plan → implement.

## Key references

- Contributor trust TL;DR: `docs/contributor-trust-proposal.md`
- Contributor trust full spec: `specs/2026-05-13-contributor-trust-open-source.md`
- Protocol added: `parent/docs/protocols/spi-default-method-contract-test.md`
- Blog: `blog/2026-05-13-mdp01-contributor-trust-open-source.md`
- Foundation gates + improvements: `docs/PROGRESS.md` in project repo
