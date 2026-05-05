# devtown Workspace

**Project repo:** /Users/mdproctor/claude/casehub/devtown
**Workspace type:** public

## Session Start

Run `add-dir /Users/mdproctor/claude/casehub/devtown` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md
- `design/` — epic journal (created by `epic` at branch start)

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | workspace   | |
| blog       | workspace   | |
| design     | workspace   | |
| snapshots  | workspace   | |
| specs      | workspace   | |
| handover   | workspace   | |

---

# casehub-devtown — Claude Code Project Guide

## Platform Context

This repo is one component of the casehubio multi-repo platform. **Before implementing anything — any feature, SPI, data model, or abstraction — run the Platform Coherence Protocol.**

The protocol asks: Does this already exist elsewhere? Is this the right repo for it? Does this create a consolidation opportunity? Is this consistent with how the platform handles the same concern in other repos?

**Platform architecture (fetch before any implementation decision):**
```
https://raw.githubusercontent.com/casehubio/parent/main/docs/PLATFORM.md
```

**Other repo deep-dives** (fetch the relevant ones when your implementation touches their domain):
- casehub-ledger: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-ledger.md`
- casehub-work: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-work.md`
- casehub-qhorus: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-qhorus.md`
- casehub-engine: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-engine.md`
- claudony: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/claudony.md`
- casehub-connectors: `https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-connectors.md`

---

## Project Type

type: java

**Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, GraalVM 25 (native image target)

---

## What This Project Is

`casehub-devtown` is the AI-assisted software development **application layer** built on the CaseHub platform foundation. It is deliberately NOT part of the foundation — the foundation (casehub-engine, casehub-qhorus, casehub-ledger, casehub-work) has no domain knowledge. This repo provides the software engineering domain logic on top of those primitives.

**This repo owns:**
- Capability tag definitions for the software development domain (security-review, architecture-review, style-review, test-coverage)
- Trust dimension definitions relevant to code review (review-thoroughness, false-positive-rate)
- Routing thresholds per capability — which trust score qualifies an agent for which type of work
- `TrustWeightedSelectionStrategy` — WorkerSelectionStrategy that routes code review tasks by capability-scoped trust
- Post-merge trust feedback — when a production incident traces back to a missed review, write FLAGGED attestation
- Merge queue orchestration (`casehub-refinery` — Bors-style batch-then-bisect as a CasePlanModel)
- Human code review WorkItem lifecycle (SLA, delegation, form schemas for review findings)

**This repo does NOT own:**
- The trust model itself (casehub-ledger)
- The commitment lifecycle (casehub-qhorus)
- The case engine or blackboard (casehub-engine)
- The WorkItem inbox (casehub-work)
- Notification delivery (casehub-connectors)
- Any foundation primitive — if you find yourself implementing something that would be useful to any domain, it belongs in the foundation, not here

---

## Layering Rule

This is an application, not a framework. When in doubt: if the capability requires knowledge of software engineering concepts (PRs, commits, CI, code review, merge queues), it belongs here. If it's purely about actors, trust, commitments, or audit records, it belongs in the foundation.

---

## Ecosystem Conventions

All casehubio projects align on these conventions:

**Quarkus version:** All projects use `3.32.2`. When bumping, bump all projects together.

**GitHub Packages — dependency resolution:** Add to `pom.xml` `<repositories>`:
```xml
<repository>
  <id>github</id>
  <url>https://maven.pkg.github.com/casehubio/*</url>
  <snapshots><enabled>true</enabled></snapshots>
</repository>
```
CI must use `server-id: github` + `GITHUB_TOKEN` in `actions/setup-java`.

**Cross-project SNAPSHOT versions:** All casehubio artifacts are `0.2-SNAPSHOT` resolved from GitHub Packages. Declare in `pom.xml` properties and `<dependencyManagement>` — no hardcoded versions in submodule poms.

**Java on this machine:**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26)    # Java 26, use for dev and tests
JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-25.jdk/Contents/Home  # GraalVM 25, native only
```

**Use `mvn` not `./mvnw`** — maven wrapper not configured on this machine.

---

## Work Tracking

**Issue tracking:** enabled
**GitHub repo:** casehubio/devtown

**Automatic behaviours (Claude follows these at all times):**
- **Before implementation begins** — check if an active issue exists. If not, run issue-workflow Phase 1 before writing any code.
- **Before any commit** — confirm issue linkage.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
