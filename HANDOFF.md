# HANDOFF — devtown

## Last Session

Closed #220 (dashboard OIDC auth) — GitHub OAuth via hybrid OIDC, SecurityIdentityAugmentor with TTL-cached membership, default-deny /api/* policy, UserInfo endpoint, auth-aware frontend. Landed as `465d1b9` on main. Follow-up `2a3b50b` added 16 tests (augmentor, group provider, userinfo). CI green. Blog entry written: `blog/2026-10-22-mdp02-auth-that-doesnt-forget.md`.

Parent POM bumped from 0.3 to 0.4-SNAPSHOT in this session. `PrReviewQhorusLifecycleTest` has 5 pre-existing failures from #252 (Eidos migration) — agents excluded in test profile, no commands dispatched.

## Immediate Next Step

#253 (epic) closed this session — all 5 child issues done. Next candidates:
- #223 (Agents post findings on GitHub PRs) — L/Med, extends the write-back observers
- #225 (Multi-user testing + production auth) — M/Med, builds on #220 auth work
- #247 (Wire real LLM agents) — XL/High, production agent wiring

## References

- Auth source: `app/src/main/java/io/casehub/devtown/app/spi/GitHub*` (augmentor, group provider)
- REST client: `github/src/main/java/io/casehub/devtown/github/GitHubOrgApi.java`
- OIDC config: `app/src/main/resources/application.properties` lines 95-120
- Blog: `blog/2026-10-22-mdp02-auth-that-doesnt-forget.md`
