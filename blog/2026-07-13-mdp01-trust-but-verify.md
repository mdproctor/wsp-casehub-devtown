---
layout: post
title: "Trust, but Verify"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust-routing, evidential-checking, attestation]
series: issue-141-evidential-checker-v1-v4
---

*Part of a series on [#141 — EvidentialChecker V1-V4 integration](https://github.com/casehubio/devtown/issues/141). Previous: [The View from Forty Thousand Feet](2026-07-12-mdp01-the-view-from-forty-thousand-feet.md).*

## The Gap in Trust-Gated Attestation

`TrustGatedAttestationPolicy` has been scaling confidence based on trust scores since Layer 6. A high-trust agent's DONE gets boosted confidence; a below-threshold agent gets scaled-down confidence. But both get `SOUND`. The policy modulates *how much* the system trusts the claim — it never questions *whether the claim is real*.

That's the gap. A below-threshold agent saying "I reviewed your security-sensitive PR" gets a SOUND attestation at reduced confidence, but nobody checks whether it actually looked at anything. The qhorus runtime already has the machinery to check — `EvidentialChecker` with V1-V4 benchmark variants — but nothing in devtown was wired to use it.

## What Fires and What Waits

The design wraps `TrustGatedAttestationPolicy` with `EvidentialAttestationPolicy` at CDI `@Priority(2)`. For DONE claims from agents in configured trust phases, it runs V1-V4 before accepting the attestation:

| Variant | Check | Fires today? |
|---|---|---|
| V1 | DONE references a real artefact | No — `artefactUuid` not yet populated |
| V2 | Channel has actual messages | **Yes** — `channelId` available |
| V3 | Not confirming a FAILED obligation | **Yes** — `correlationId` available |
| V4 | Contains verification token | No — `expectedToken` not yet populated |

Two of four fire immediately. V1 and V4 become active when `CommitmentContext` is enriched upstream — no devtown changes needed at that point.

The phase classification uses `TrustRoutingPolicy`'s public API directly. About twelve lines to classify an agent as BOOTSTRAP, BORDERLINE, BELOW_THRESHOLD, QUALITY_FAILED, or QUALIFIED. The policy's `evidentialCheckPhases` set determines which phases trigger checks — configurable per capability.

## Per-Capability Tuning

Not every capability needs the same scrutiny. `style-review` has low stakes — trust modulation alone is sufficient. `merge-executor` is irreversible — check everyone who isn't fully qualified:

| Capability | Phases checked |
|---|---|
| `security-review` | BELOW_THRESHOLD, QUALITY_FAILED, BOOTSTRAP |
| `architecture-review` | BELOW_THRESHOLD, QUALITY_FAILED |
| `style-review` | *(none)* |
| `merge-executor` | BELOW_THRESHOLD, QUALITY_FAILED, BOOTSTRAP, BORDERLINE |

When a check finds violations — zero messages on the channel, confirming a FAILED obligation — the attestation flips to FLAGGED at 0.8 confidence. Not a probabilistic assessment but structural proof that the claim is invalid.

## The SNAPSHOT Tax

Half the session was SNAPSHOT drift cleanup. The engine renamed `inputSchema` to `inputProjection`, `ChannelCreateRequest` gained a field, `CommitmentContext` grew from five to eight constructor args, `PlanItemStore.findDelegated` added a tenancyId parameter, and `NamedStrategy` gained an `id()` method. Seven files touched for API changes that had nothing to do with the feature.

This is the cost of SNAPSHOT-coupled multi-repo development. Each upstream change is individually trivial; cumulatively they consumed as much time as the actual implementation.
