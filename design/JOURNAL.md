# Design Journal — issue-170-binding-activation-state

## §1 Activation Context Capture — 2026-08-03

**Decision:** Capture the changed layer content (not the full context) when a binding fires, and store it on `PlanItemRecord` as a first-class field rather than deriving it from event log joins.

**Rationale:** The event log join approach requires temporal correlation (`findClosestByTimestamp`) which breaks for repeatable bindings. Storing directly on the plan item eliminates ambiguity — each plan item carries its own activation snapshot from creation.

**CDI event consolidation:** Replaced three separate events (`PlanItemCompletedEvent`, `PlanItemFaultedEvent`, `PlanItemRejectedEvent`) with a single `PlanItemStateChangedEvent` carrying `previousStatus`/`newStatus` as strings. String-typed to avoid SPI→internal package dependency on `TaskStatus` enum.

**Open:** Whether changed layer alone is sufficient. Guard conditions may reference multiple layers. Deferred full condition-referenced key extraction as a future enhancement.
