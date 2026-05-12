# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-05-12 — Open source contributor trust: prioritised routing + vouching

**Priority:** high  
**Status:** active

DevTown's trust model (Bayesian Beta scoring, `RoutingPolicy` thresholds) can be extended to model *contributor* trust, not just reviewer trust. A contributor's score accumulates from PR outcomes: clean first-merge strengthens it; returned-for-rework weakens it; outright rejection weakens it faster. High-trust contributors get fast-tracked; zero/low-trust contributors (including AI slop generators and new accounts) land in a triage queue. The adversarial property is useful: slop generators can't easily game this — new GitHub account = Phase 0 = always triage, and negative history accumulates quickly.

To avoid the bootstrapping problem for genuine new contributors, high-trust contributors can *vouch* for newcomers — transferring a temporary score boost with their own score at risk if the vouchee underperforms. This maps directly to EigenTrust (already in the ledger), making vouching a directed edge in the trust graph. Key design constraint: vouching must be asymmetric (can only vouch for lower-trust actors) and costly (voucher score degrades on vouchee failure), to prevent vouching rings.

The remaining structural gap is identity: trust only accumulates if the same actor can be tracked across submissions. Stable GitHub account identity covers genuine contributors. The adversarial case (new account per PR) doesn't break the system — it just means those actors never escape the triage queue.

**Context:** Conversation prompted by a colleague noting that reviewing community PRs for Quarkus on GitHub has become unmanageable due to the volume of low-quality AI submissions. Explored whether DevTown's trust primitives (already designed for reviewer routing) could be extended to contributor-side routing. The platform already has the mechanics (Bayesian Beta, EigenTrust, `RoutingPolicy`); this is new application-layer design work.

**Next step:** Full high-level stakeholder buy-in spec — not an implementation spec. To be written in a future session for sharing with a colleague.

**Promoted to:**
