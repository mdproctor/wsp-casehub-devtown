# Design Decisions — Issue #266: Evolution Apply-Stage Workers

## D1: Apply execution model — direct preference writes

**Choice:** Workers write to PreferenceProvider directly in the apply stage
**Alternatives:**
- PR-based config changes — produces YAML diffs as PRs with git history audit, but slower and fights the self-improvement template's operational flow assumptions
- Devtown-specific case template — custom template skipping PR stages, but adds template maintenance overhead without adding value over the existing conductor gating
**Rationale:** The conductor's gated propose and review stages already provide human oversight before apply fires. The case lifecycle (EventLog, conductor inbox, worker output with old/new values, CaseLedgerEntry Merkle audit) provides richer auditability than git history alone — it links each change to the health signal that triggered it, the human approval that authorized it, and the regression evaluator that monitors impact.
**Trade-offs:** No git-diffable history of preference changes; audit trail lives in the case system rather than version control. Mitigated by worker output recording before/after values in the case context.
**Sources:** DevtownCategoryProvider stages (analyze→propose[gated]→review[gated]→apply→observe), self-improvement.yaml operational branch, PreferenceProvider JPA persistence (platform#24), CaseLedgerEntry (devtown#73)
**Exploration:** quick
**Status:** captured
