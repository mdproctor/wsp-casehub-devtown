---
layout: post
title: "The feedback loop closes. The routing doesn't budge."
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [casehub, foundation, progress, trust]
---

The progress documents were three weeks stale. PROGRESS.md and the Gastown
analysis were both dated 2026-05-02, and a significant amount of foundation
work had shipped since. We fixed that by dispatching five separate Claudes —
each one assigned two or three repos, scanning closed issues, reporting back.
The results landed within minutes.

P0 is complete. All of it. The `ClaudonyInstanceActorIdProvider` that maps
ephemeral Claude session IDs to persona format — `claude:analyst@v1` — shipped
in claudony#107 on May 3rd. Trust now accumulates per persona across sessions
rather than resetting every time a new tmux session opens. The normative feedback
loop is structurally correct end-to-end: COMMAND dispatch → commitment lifecycle
→ attestations → trust scoring.

And then nothing happens with those scores.

P1.3 — injecting `WorkerSelectionStrategy` into `WorkOrchestrator` so trust
scores influence who receives work — is unstarted. Two prerequisites have since
been filed: engine#336 (`SelectionContext` carries no trust field, so a strategy
can't read scores even if one existed) and engine#337 (`WorkOrchestrator`
hard-codes `LeastLoadedStrategy` rather than resolving via CDI). On top of
that, qhorus#199 found that `TrustGateService` is never called at COMMAND
creation — obligations open unconditionally regardless of trust score. The trust
pipeline produces correct data. None of it affects routing.

The ledger moved further than I expected. Ed25519 bilateral signing landed in
mid-May — both sides now sign, making the audit trail non-repudiable in a way
that Dolt's admin-trusted history can't match. Trust export/import (P2.1)
shipped May 14th, closing the Gastown Wasteland parity gap. The key difference:
what CaseHub exports is derived from cryptographically attested commitment
outcomes rather than human-curated stamps, which is a meaningful distinction
if you care about what "reputation" actually means.

The question came up directly: are we almost there? My honest answer is around
30%.

The foundation infrastructure is in better shape than I'd realised — P0
complete, most of P2 done. But the application is at Layer 2 of 7, trust
routing is decorative until P1.3 and qhorus#199 land, the merge queue hasn't
been started, and the operational reliability layer — throttle, watchdog
recovery — is untouched. The 30% is real work. The remaining 70% isn't just
typing.

With the docs current, we ran through the small open issues. Two were already
resolved and just needed closing: `evalObjectTemplate` was gone from
`MapCaseContext`, the duplicate endpoint conflict had been fixed by qhorus#172.
Two needed actual changes: removing the qhorus reactive pool suppression from
test properties (qhorus#141 shipped, making it unnecessary), and adding
`doesNotFire_whenAlreadyDone` tests for `test-coverage` and `performance-analysis`
— `style-check` had the test, the other two parallel checks were missing it.
Tidying the attic.
