# Contributor Trust: Filtering AI PR Slop Without Breaking Good Contributors

**Date:** 2026-05-13  
**Status:** Proposal — stakeholder buy-in  
**Audience:** Engineers familiar with open source workflows; no prior knowledge of CaseHub required

---

## The problem

Open source maintainers are overwhelmed. GitHub makes submission free, and AI coding assistants have made it trivial to generate plausible-looking but low-quality PRs at scale. Projects like Quarkus are seeing a flood of submissions that require more reviewer time to reject than the code took to generate.

The standard fixes don't hold up:

- **Rate limiting** — punishes prolific genuine contributors
- **Minimum account age** — gamed in a day
- **AI detection** — an arms race with no sustainable winning side
- **Stricter guidelines** — read by humans, ignored by slop generators

The underlying problem isn't that we can't identify AI slop. It's that a first-time human contributor and a fresh slop account look identical — no history.

---

## The proposal

Route PRs based on accumulated contributor reputation. Not blocking who submits, but controlling what gets reviewed first.

A contributor starts with no score. Outcomes build it:
- PR merges cleanly on first submission → score rises
- PR returned for rework → score drops
- PR rejected outright → score drops faster

High-reputation contributors get fast-tracked. New or low-reputation contributors land in a triage queue. Nothing is blocked — the queue just has a different priority.

The key property: **slop generators can't game this**. A new GitHub account means zero score, which means triage. To build reputation, PRs have to actually get merged. The score is the barrier.

---

## How the score works: the Bayesian Beta model

A naive approach counts merged PRs divided by submitted PRs. The problem: 2 merged from 2 submitted gives the same percentage as 200 merged from 200 — but these are not equivalent. One might be luck.

The platform uses a **Bayesian Beta model** — the same probabilistic framework used in medical trial analysis and A/B testing — which tracks two things simultaneously: the *estimated* success rate and the *confidence* in that estimate. With 2 data points, the confidence is low. With 200, the estimate is reliable. The model knows the difference.

Before seeing any evidence, every contributor starts with a prior assumption — roughly "unknown." Each PR outcome updates that assumption. A merge shifts the distribution toward "reliable"; a rejection shifts it toward "unreliable." A small number of outcomes produces a wide, uncertain distribution. The model won't act confidently on thin evidence.

The practical consequence: an account with 1 merged PR does not jump the queue. The platform requires a minimum number of observations before the score carries weight — configurable per project. A small library might set this at 5; a large security-critical project at 20.

This is also why the score doesn't just track acceptance rate — it tracks *confidence in that acceptance rate*. A contributor at 80% from 5 PRs gets treated more cautiously than one at 75% from 50. The raw percentage would rank them incorrectly. The Bayesian model ranks them correctly.

---

## Bootstrapping from history

Most open source projects don't need to wait months for the system to accumulate evidence. The evidence already exists.

GitHub's API exposes the full PR history for any public repository. Before deploying, you mine that history: for every contributor account, count submissions, merges, rejections, and return-for-rework cycles going back years. Feed those outcomes into the Bayesian model. On day one, established contributors arrive with scores that reflect their actual track record — not a blank slate.

A project like Quarkus with ten years of PR history can immediately differentiate between a contributor with 47 merged PRs over three years and an account created last week. New accounts start at zero, which is exactly right.

The history mining can go further: which PRs touched which file paths? A contributor with 30 merged PRs all touching documentation scores differently for a security-module change than one with 30 PRs spanning authentication, crypto, and networking. The platform can segment trust by domain if the project chooses.

---

## What the platform provides

**Bayesian Beta scoring** — already implemented in the foundation for scoring AI agent reliability. The model, the update logic, the confidence tracking, and the configurable minimum-observations threshold are all in place. Bayesian Beta is well-established across decades of clinical trial design, A/B testing, and recommender systems. We're applying proven mathematics through new infrastructure, pointed at a new problem.

**EigenTrust propagation** — published at WWW 2003 and widely used in peer-to-peer reputation systems, this is the same family of ideas as Google's PageRank. When a maintainer with 8 years of commits vouches for a newcomer, that vouch carries more weight than one from someone who joined three months ago. Trust propagates through the vouching graph, attenuated at each step. Circular inflation rings — A vouches for B, B vouches for A — collapse automatically under the mathematics. Also already implemented in the platform.

**Work queues and priority views** — the trust score becomes a practical workflow tool through the platform's work queue layer. Each queue is a labelled, filtered, prioritised view of incoming PRs:

- **Fast-track queue** — high-trust contributors, sorted by submission time
- **Triage queue** — zero or low-trust contributors; AI slop and genuine newcomers alike, ordered by file-path risk
- **Security queue** — any PR touching security-sensitive paths, regardless of contributor trust
- **Borderline queue** — contributors within the confidence margin; lighter-weight review before promotion

Labels can combine trust signals with static analysis. A zero-trust PR touching only documentation is lower risk than a zero-trust PR touching authentication. Both land in triage, but the ordering reflects the difference. The queues also support **SLA tracking** — a PR sitting in triage for 7 days without action gets surfaced automatically — and **reviewer load balancing**, distributing assignments across available reviewers based on capacity.

**Cryptographic audit trail** — every score update is recorded in a tamper-evident ledger. The records are cryptographically chained: you can't retroactively alter a score history without breaking the chain. Contributors can inspect exactly which events drove their score. Maintainers can verify the system is making decisions honestly. The EU AI Act's Art. 12 requires traceability for automated decisions affecting human participants — a system routing contributors algorithmically without an auditable record has a compliance problem from the start. The platform has this built in.

---

## Vouching: the bootstrapping problem

The scoring system handles established contributors and clear bad actors well. The gap is genuine newcomers — good contributors with no history.

Vouching solves this. A high-reputation contributor vouches for a newcomer, temporarily elevating their score. The voucher's own score is at risk: if the vouchee's PRs get returned, the voucher loses points too.

EigenTrust governs the weight of each vouch. A vouch from a 10-year core committer carries more than one from a contributor who merged their first PR last month. Constraints enforced by the system:

- You can only vouch for contributors with *lower* reputation than you — no mutual inflation
- Vouching capacity is limited — you can't sponsor hundreds of people simultaneously
- The trust graph is visible — the chain of vouches is inspectable

This is how professional references work, except the cost is automatic and quantified. A senior maintainer can get a trusted colleague into the fast lane immediately. That colleague now has skin in the game.

---

## A day in the life

Maria is a Quarkus core maintainer. Monday morning, she opens her review dashboard.

**Fast-track queue: 4 PRs.** Contributors the system is confident in — combined, they have 300+ merged PRs and clean track records. She skims them efficiently. Three merge same-day. One needs a small change; she leaves a comment and it's back within the hour.

**Borderline queue: 2 PRs.** Contributors with enough history to be promising but not enough to be certain. She gives these a proper read. One is solid — she approves it and the contributor's score climbs. The other has an issue she flags; the contributor fixes it.

**Triage queue: 31 PRs.** The flood. But already ordered: PRs touching documentation at the top, PRs touching security or networking at the bottom. She works through the top 10, rejects 8 as slop, approves 2 from what look like genuine first-timers.

One of those first-timers messages her — they've been trying to contribute for weeks and don't understand the delay. She looks at their history: 3 merged PRs, clean, but the minimum observations threshold is 5. She vouches for them. They move to the borderline queue immediately.

By noon, Maria has processed 40 PRs without touching the bottom half of triage, where the worst slop sits. Before this system, she'd have spent the morning deciding which pile to ignore.

---

## How this fits into DevTown: one pipeline, two trust dimensions

DevTown already manages PR review as a case with goals — `pr-approved`, `security-verified`, `ci-passing` — coordinating AI agents and human reviewers through trust-weighted routing. Contributor trust doesn't add a second system alongside this. It extends the same case model upstream, to the moment the PR arrives.

Every PR is a case with two actor dimensions:

1. **The submitter** — who opened this PR? What is their track record?
2. **The reviewers** — who should evaluate it? Are they qualified for this kind of work?

Both dimensions live in the same trust ledger. The pipeline is continuous:

```
PR submitted
  → contributor trust checked → intake queue assigned
  → reviewer assigned by trust-weighted routing
  → review outcome recorded
  → contributor score updated
  → reviewer score updated
  → both scores inform the next PR
```

Two self-improving feedback loops run in parallel. A contributor whose PRs consistently merge cleanly builds a score that earns faster review. A reviewer who accurately catches issues gets routed to more complex work. Neither loop requires manual curation — outcomes drive both.

**Three views, one system**

- **Intake dashboard** — for maintainers. Prioritised queues, contributor scores, vouching controls, SLA alerts.
- **Review dashboard** — for reviewers. Assigned cases ordered by urgency, complexity, and capability match.
- **Contributor view** — for PR authors. Their own score, which events drove it, active vouches, queue position.

Three audiences, one data model.

**Open source vs internal team: graceful degradation**

For an internal team where everyone is trusted, contributor trust is effectively inactive — the minimum observations threshold is set to zero, or all team members are pre-vouched. The system operates as pure reviewer routing with no intake friction.

For an open source project, both dimensions are live. The intake queue is the new high-value capability; the reviewer routing is the foundation it sits on.

The open source use case is also the *more complete* demonstration of DevTown. Internal teams only see reviewer routing. Open source maintainers see the full pipeline — intake prioritisation, reviewer routing, trust accumulation on both sides, vouching, audit trail, SLA management — in a single coherent workflow.

---

## Identity

The weak point: reputation lives on GitHub account identity. A determined adversary creates new accounts.

But new accounts always start in triage. Building reputation requires PRs that actually get merged — real effort, even for adversaries. Vouching provides a social escape valve for genuine newcomers. The adversary who games this has to put in the work of a genuine contributor, at which point the incentive disappears.

---

## What we'd still need to build

The scoring model, trust propagation, audit ledger, and work queue infrastructure are already in the platform. What's new:

1. **GitHub event integration** — webhooks for PR lifecycle events feeding into contributor score updates
2. **History mining** — a one-time import of existing PR history to bootstrap scores at deployment
3. **Routing logic** — queue assignment at PR submission, driven by score and confidence threshold
4. **Vouching interface** — a maintainer-facing tool for issuing vouches
5. **Contributor score visibility** — so contributors understand their standing and can raise disputes

---

## What this isn't

- **Not AI detection.** We track outcomes, not code content.
- **Not gatekeeping.** Every PR enters the queue. Reputation controls priority.
- **Not a new trust model.** Bayesian Beta and EigenTrust are established mathematics. The infrastructure is new; the models are not.

---

## Open questions

1. **Score scope** — per-project, per-organisation, or portable across open source?
2. **Vouching governance** — maintainers only, or any sufficiently trusted contributor?
3. **Score decay** — should a dormant account retain its score indefinitely, or decay without recent activity?
4. **Contributor visibility** — can contributors see their own score? Can they dispute individual events?
