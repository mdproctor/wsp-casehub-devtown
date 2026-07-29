*Updated: #247 does not exist — removed from backlog.*

# HANDOFF — 2026-07-29

## Last Session

CI fix session. Took CI from red to green across five root causes: Yarn auth/immutable defaults, fragile monorepo frontend build (replaced with stub), platform SNAPSHOT API migration (sync returns, Preference interface, ExpressionEvaluator package move), silent SettingsScope path prefix bug (11 callers fixed), and engine CatchAllExceptionMapper swallowing JAX-RS status codes.

## Immediate Next Step

CI is green. Two integration tests disabled pending a test-scoped `WorkerProvisioner` stub.

## Cross-Module

**Enabled:**
- `engine` — CatchAllExceptionMapper fix landed on main (`51e94e8`), SNAPSHOT published · XS · Low

**Blocked by:**
- `platform` — SubscriptionEngine + NotificationDispatcher · L · High

## What's Left

- Frontend CI build — currently stubbed; needs `@casehubio` npm packages published with versioned deps (not `file:`) before real frontend CI is viable · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #172 | Pages upgrade OKLCH theme | M | Med | Branch exists, stash has WIP changes |
| #98 | Trust visibility UI | S | Low | blocks-ui#89 closed — unblocked |
| #120 | Case dependency graph | M | Med | |

## References

- Blog: `blog/2026-07-29-mdp10-five-root-causes-one-red-ci.md`
- Garden: `GE-20260729-5c56d9` (Yarn CI immutable), `GE-20260729-66f060` (scope path), `GE-20260729-392052` (exception mapper)
- Demo instructions: *Unchanged — `git show HEAD~1:HANDOFF.md`*
