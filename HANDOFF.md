# HANDOFF — devtown

## Last Session

Closed #220 (dashboard OIDC auth) — GitHub OAuth via hybrid OIDC, SecurityIdentityAugmentor with TTL-cached membership, default-deny /api/* policy, UserInfo endpoint, auth-aware frontend. Landed as `465d1b9` on main. Follow-up `2a3b50b` added 16 tests (augmentor, group provider, userinfo). CI green. Blog entry written: `blog/2026-10-22-mdp02-auth-that-doesnt-forget.md`.

Parent POM bumped from 0.3 to 0.4-SNAPSHOT in this session. `PrReviewQhorusLifecycleTest` has 5 pre-existing failures from #252 (Eidos migration) — agents excluded in test profile, no commands dispatched.

## Immediate Next Step

`work start #253` — End-to-end on real PRs. Both prerequisites (#252 Eidos descriptors, #254 write-back observers) are merged. This validates the full pipeline: webhook → case → agent review → GitHub write-back.

## References

- Auth source: `app/src/main/java/io/casehub/devtown/app/spi/GitHub*` (augmentor, group provider)
- REST client: `github/src/main/java/io/casehub/devtown/github/GitHubOrgApi.java`
- OIDC config: `app/src/main/resources/application.properties` lines 95-120
- Blog: `blog/2026-10-22-mdp02-auth-that-doesnt-forget.md`
