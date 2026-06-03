# Design Journal — issue-62-merge-executor-human-oversight

## §Design — Bootstrap Fallback: Foundation vs Devtown Strategy

**Decision:** Option A — extend `TrustRoutingPolicy` in the foundation rather than create a devtown `@Priority(3)` strategy.

**Why:** `RoutingPolicy.fallbackType` already exists in `DevtownCapabilityRegistry` for `merge-executor`, `security-review`, and `architecture-review`. PLATFORM.md §67 states every capability must declare a `fallbackType`. The correct enforcement point is the foundation: `TrustRoutingPolicy` gets `Optional<String> bootstrapFallbackType`; `TrustWeightedAgentStrategy.select()` checks it before scoring. Devtown then maps `RoutingPolicy.fallbackType()` through `DevtownTrustRoutingPolicyProvider` — one method call, no new classes.

**Why not Option B (devtown @Priority(3) strategy):** Would duplicate the trust-blend scoring logic from `TrustWeightedAgentStrategy` (private `score()` method, not delegatable). Also, PLATFORM.md reserves `@Priority(2)` for `SemanticAgentRoutingStrategy` — devtown would need `@Priority(3)`, conflicting with the foundation priority ladder long-term.

**Filed:** casehubio/engine#415 — covers both `TrustRoutingPolicy` record change (engine-api module) and bootstrap guard in `TrustWeightedAgentStrategy.select()` (engine-ledger module).

**Devtown work remaining:** Once engine#415 merges, update `DevtownTrustRoutingPolicyProvider.forCapability()` to pass `routingPolicy.fallbackType()` as `bootstrapFallbackType`. Single addition, no structural changes.
