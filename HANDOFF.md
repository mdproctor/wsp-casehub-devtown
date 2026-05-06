# Handoff — devtown Bootstrap
2026-05-07

## What this project is

`casehub-devtown` is the AI-assisted software development **application layer** on the CaseHub platform. It is what Gastown does — PR review orchestration, merge queue, agent coordination for software development — but built on CaseHub's domain-agnostic foundation rather than baked into infrastructure. The gap between them is not feature parity; it is architectural: devtown will inherit formal obligation tracking, cryptographic audit, trust-weighted routing, and adaptive case management that Gastown structurally cannot provide.

The full comparison is in `docs/gastown-casehub-analysis-v2.md` (845 lines). Read it. The orchestration advantages with YAML examples are in `docs/orchestration-advantages.md`. These two documents are the design brief.

## Current state

The repo exists but has **no code yet** — only documentation and the CLAUDE.md. This is a greenfield build. The foundation it sits on is mostly built and partially wired. The critical remaining foundation gap is qhorus#124 (claudony persona→session mapping) which blocks end-to-end trust accumulation per persona. Everything else can be built now.

## Foundation status (what devtown can rely on today)

| Foundation capability | Status |
|----------------------|--------|
| Case engine, bindings, blackboard (casehub-engine) | ✅ Production |
| CasePlanModel, stage gating, sub-cases | ✅ Production |
| qhorus typed channels (COMMAND/RESPONSE/DONE/FAILURE/DECLINE) | ✅ Production |
| Commitment lifecycle (7 states: OPEN→FULFILLED/DECLINED/FAILED/DELEGATED/EXPIRED) | ✅ Production |
| Commitment outcomes → trust scoring (LedgerAttestation) | ✅ DONE (qhorus#123) |
| WorkerContextProvider + channels() in-worker access | ✅ DONE (engine#220/PR#224) |
| engine COMMAND dispatch after worker scheduling | ✅ DONE (engine#186) |
| CaseLedgerEntry (Merkle audit per case event) | ✅ DONE (2026-04-26) |
| ActorTypeResolver — canonical actorId across repos | ✅ DONE (all consumers) |
| casehub-work WorkItem lifecycle (SLA, delegation, escalation) | ✅ Production |
| casehub-work GitHub Issues sync | ✅ DONE (work#157) |
| casehub-connectors Slack/Teams/SMS/email SPI | ✅ Production |
| EigenTrust transitive trust scoring | ✅ Production |
| InstanceActorIdProvider SPI (qhorus) | ✅ DONE |
| Claudony persona→session mapping (qhorus#124) | ⚠️ Pending — trust accumulates per session not persona |
| TrustWeightedSelectionStrategy wired in engine (P1.3) | ⚠️ Pending |
| RecoveryPolicy SPI in casehub-engine (P1.2) | ⚠️ Pending |
| Agent concurrency throttling in claudony (P1.1) | ⚠️ Pending |
| SLA propagation case→WorkItem (parent#6) | ⚠️ Pending |
| casehub-work-adapter HITL wiring (WorkItem COMPLETED → case signal) | ⚠️ Pending — human-in-the-loop not yet end-to-end |

## What to build first (ordered by foundation readiness)

### Now — foundation ready today

**Epic 1 — Project scaffold:** pom.xml, Quarkus app, dependencies on all foundation modules. Get it to compile and boot. Basic package structure (`io.casehub.devtown.*`).

**Epic 2 — Domain model:** Capability tag constants, trust dimension constants, routing thresholds. Pure Java — no foundation dependency needed. This is the domain vocabulary everything else uses.

**Epic 3 — PR review CasePlanModel:** Write the YAML case definition for a single PR review. Goals (pr-approved, security-verified, ci-passing). Bindings (code-analysis first, then specialist bindings fire from analysis findings). The `docs/orchestration-advantages.md` YAML examples are the starting point.

**Epic 7 — Failure handling bindings:** DECLINED → immediate re-route to backup. FAILED → try backup, scope-reduce, escalate. Additional bindings in the PR review case — write alongside Epic 3.

### After foundation P0 fully wired (qhorus#124)

**Epic 4 — Merge queue (casehub-refinery):** Batch management, batch-then-bisect CasePlanModel, bisect sub-case spawning, PR rejection and notify.

**Epic 5 — Cross-repo coordinated merge:** Parent case + per-repo sub-cases. Rollback binding on sub-case fault.

### After P1.3 (TrustWeightedSelectionStrategy wired in engine)

**Epic 6 — Trust-weighted reviewer routing:** Wire TrustWeightedSelectionStrategy. Capability-scoped thresholds. Post-merge trust feedback (FLAGGED attestation on production incident). Closes the prescriptive→normative→evaluative feedback loop.

### Parallel tracks (no foundation P1 needed)

**Epic 8 — GitHub integration:** PR webhook receiver, CI status reader, merge executor worker.

**Epic 9 — Notification wiring:** casehub-connectors Slack/Teams for review assignments, merge outcomes, stalled commitment alerts.

**Epic 10 — Observability and tooling:** MCP tools for devtown fleet status, obligation health, review queue depth. Extend claudony dashboard (`gt feed` / `gt problems` equivalent).

## Key design decisions to make before writing code

1. **Capability tags:** Enum vs typed constants? Recommend typed constants with registry SPI — extensible without recompile.
2. **CasePlanModel format:** YAML from classpath (configurable per-repo, strategy changes without deployment) vs Java DSL. Start YAML.
3. **GitHub integration:** Webhook (clean, needs public endpoint) vs polling (dev-friendly). Start polling, switch to webhook for production.
4. **Merge queue scope:** Single-repo batch first (matches Gastown Refinery). Cross-repo coordination in Epic 5.

## What devtown adds that Gastown cannot

- Every review assignment creates a formal Commitment — DECLINED vs FAILED structurally captured, not inferred from timeouts
- Every merge decision is a Merkle-signed audit entry — independently verifiable
- Trust accumulates automatically from review outcomes — routing improves without human curation
- Routing adapts to what the code actually contains — binding fires from analysis findings, not author labels
- Human review runs in parallel with CI — total time is max(human, CI) not sum

## References

- `docs/gastown-casehub-analysis-v2.md` — architectural comparison, roadmap, 32-finding coherence audit
- `docs/orchestration-advantages.md` — ACM vs workflow with YAML examples for PR review scenarios
- Platform architecture: `https://raw.githubusercontent.com/casehubio/parent/main/docs/PLATFORM.md`
- GitHub epics: casehubio/devtown issues labelled `epic`
