---
layout: post
title: "Scoring Contributors, Not Detecting AI"
date: 2026-08-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [trust, open-source, intake]
series: issue-24-contributor-trust-routing
---

Open source maintainers have an AI problem, and every solution I've seen attacks the wrong end of it.

Rate limiting punishes prolific contributors. Account age gates take a day to bypass. AI detection is an arms race nobody wins. Stricter submission guidelines get read by humans and ignored by bots. The fundamental issue isn't identifying AI-generated code — it's that a first-time human contributor and a fresh slop account look identical. Both have zero history.

The answer is obvious once you stop trying to detect the tool and start measuring the person: score contributors by what happens to their PRs.

### Bayesian Beta, not percentages

A contributor's trust score needs to know two things: their success rate *and* how confident the system should be in that rate. Two merged PRs out of two submissions looks the same as two hundred out of two hundred if you're only counting percentages. It's not.

The platform already has a Bayesian Beta model — `TrustScoreComputer` tracks alpha and beta parameters for every scored actor, computing `α/(α+β)` while the Beta distribution width reflects confidence. We built this for scoring AI reviewer agents. The maths is identical for scoring human contributors. Same model, same ledger, different capability key: `pr-contribution` instead of `security-review`.

### Three lanes, not one queue

The domain model defines three intake lanes:

```java
public enum IntakeLane {
    FAST_TRACK(3),   // high trust — lighter review, 1hr SLA
    STANDARD(2),     // medium trust — normal review, 4hr SLA
    TRIAGE(1);       // unknown — queued for human attention
}
```

A contributor with a trust score of 0.80 from 15 observations hits FAST_TRACK — their PR starts the review case immediately with lighter review requirements. A contributor at 0.60 with 5 observations gets STANDARD — normal review, normal SLA. A brand new account? TRIAGE. No case is created. The PR sits in a triage queue until a maintainer looks at it.

This is the critical property: *slop PRs never consume reviewer resources*. They sit in triage. A maintainer scans the queue, admits the genuine newcomers, and bulk-rejects the rest. The flood stays in the lobby.

### Two dimensions from one pipeline

A reworked PR tells you two different things about its author. The merge itself is positive — the work was eventually good enough. The rework request is negative — it wasn't right the first time.

The model captures both independently:
- `merge-rate` — did the PR eventually merge? Higher is better.
- `first-attempt-quality` — did it merge *without* rework? Higher is better.

A contributor who always merges but always needs two rounds of review scores high on merge-rate, low on first-attempt-quality. That gives them STANDARD, not FAST_TRACK. The composite score from these two dimensions captures something a single merge percentage would miss: the cost of reviewing their work.

### What's next

The domain model is the foundation. The heavy work is ahead — a lifecycle attestation pipeline that turns PR outcomes (merged, rejected, returned for rework) into trust attestations; intake classification that routes incoming PRs by contributor trust; vouching for genuine newcomers using EigenTrust; and history mining that bootstraps scores from existing GitHub PR data so established contributors aren't starting from zero on day one.

The part I find most interesting is the vouching model. A vouch is just an ENDORSED attestation — EigenTrust already knows how to propagate trust through a graph. A 10-year committer's vouch carries more weight than a recent joiner's, mathematically, without any special-case code. And the voucher's own score is at risk if the newcomer's work gets rejected — skin in the game, not just a rubber stamp.
