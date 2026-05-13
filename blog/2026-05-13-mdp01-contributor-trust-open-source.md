---
layout: post
title: "Contributor trust"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust, open-source, contributor-routing]
---

A colleague raised this at the start of a conversation: Quarkus is getting buried in low-quality AI-generated PRs. Reviewers can't keep up. Could DevTown help?

I'd been thinking about DevTown primarily as a closed-team system — agents and humans working together in a controlled pool where trust accumulates from outcomes. The open source contributor problem is different. You don't control who submits. GitHub identity is open. The slop generators know this.

But trust accumulation is still the right frame.

The mechanics are simple: a contributor's score rises when PRs merge cleanly on first submission, falls when they come back for rework, falls faster when they're rejected outright. High-trust contributors get fast-tracked; low or zero-trust contributors land in a triage queue. Phase 0 behaviour — before any history exists — is identical to what Quarkus does today. Everything gets reviewed. The system starts where the status quo is and improves automatically as evidence accumulates.

Claude caught the adversarial property immediately: a slop generator that creates a new GitHub account per PR resets to zero on every submission. Zero is the triage queue. Their score never climbs because their PRs don't get merged. No explicit AI detection needed — reputation is just expensive to build.

The harder problem is genuine new contributors. Not every zero-history account is slop. I pushed back on treating all of them identically. Claude's response: that's what vouching is for.

A high-trust contributor vouches for a newcomer — temporarily elevating their score, with the voucher's own score at risk if the vouchee's PRs get returned. The cost of vouching is real. You only do it for someone you actually believe in. This is EigenTrust — directed edges in a trust graph — and casehub-ledger already has EigenTrust in it. No new primitives required.

What it doesn't solve cleanly is identity. A determined adversary creates new accounts. But every reputation system faces this, and the answer is always the same: you can't prevent it, you can make it expensive. Reputation that takes months to build and collapses faster than it grows is a meaningful barrier even against adversaries who understand the system.

None of this is built. But the foundation is there. It's the first time the DevTown trust model has been pointed at a problem outside the internal team, and the fit is cleaner than I expected.
