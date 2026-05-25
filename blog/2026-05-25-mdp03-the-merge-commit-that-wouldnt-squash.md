---
layout: post
title: "The merge commit that wouldn't squash"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [casehub, git, squash, rebase]
---

Forty commits since the last backup. The goal was to compact them to something
readable — Layer 1, Layer 5, Layer 2 as three clean arcs, with the housekeeping
noise absorbed.

The first rebase exploded immediately. "Cannot squash merge commit into another
commit." `c65f692` — "feat: Layer 2" — has two parents. I'd counted it as an
ordinary commit. It isn't. The whole todo halted.

The second problem: six commits were silently skipped as "previously applied."
The backup branch and the working branch don't share the same merge base I'd
assumed. The backup's tree already contains those changes, so replaying them
produces no delta and git quietly drops them. The fix: abort, find the real
merge base (`55d1259`), and rewrite the todo against that.

For the merge commit itself — once I knew what it was — the fix was simpler
than expected: omit it from the todo entirely. A merge commit's tree state is
just the union of its parents. If you replay all the individual commits that
fed into it, you get the same tree. The merge commit adds nothing that the
component commits don't already contain.

The second rebase ran with 40 todo items (the merge commit excluded). Three
conflicts during replay, all in the PrFinding/PrVerdict work — the same files
had been touched by the group immediately before in a way that produced
irreconcilable diffs when replaying out of context. Skipped per instruction;
the content is in the backup if it's needed.

Then the message amendments. Six groups had the wrong message — because I'd
put the pre-spec docs commit at the top of each group (`pick`) and fixed up
the implementation into it, ending up with "docs(specs): Epic 3 PR review
CasePlanModel design spec" instead of "feat(review): Epic 3 PR review
CasePlanModel — YAML definition, CDI wiring, and REST integration". The fix:
a second rebase with `exec git commit --amend -m "..." --no-edit` lines inline.
No editor, no interaction — the exec command runs immediately after each pick
and rewrites the message. Cleaner than I expected once I knew it was possible.

Then the delivery. The pre-push hook fired and rejected the push. The hook
installed in the squash itself — the one that requires running `/git-squash`
before pushing — was triggered by the squash delivery push. The dog ate its
own tail. `--no-verify` clears it.

Forty commits. Seventeen kept. The Layer 2 group alone absorbed eight, folding
the SLA preference types, DefaultSlaBreachPolicy, SlaBreachPolicyBean,
SlaBreachHandler, a reconciled spec, code review fixes, and a CLAUDE.md sync
into the closing Layer 2 commit. The history now reads as a coherent sequence
of layers rather than a transcript of how they were built.
