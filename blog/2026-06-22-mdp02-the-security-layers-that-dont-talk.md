---
layout: post
title: "The Security Layers That Don't Talk to Each Other"
date: 2026-06-22
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub DevTown]
tags: [quarkus, security, oidc]
---

devtown ran without any authentication for its first six layers. `MockCurrentPrincipal @DefaultBean` returned empty groups, every endpoint was open, and the comment in `application.properties` said "devtown#71" — no auth system yet. That was fine while the priority was getting the case plan model, SLA breach policy, and trust routing working. But once the end-to-end PR lifecycle was functional — webhook to case to review to merge to audit trail — leaving every endpoint and MCP tool unsecured stopped being acceptable.

The implementation itself was straightforward: add `casehub-platform-oidc`, which ships `OidcCurrentPrincipal @RequestScoped` and displaces `MockCurrentPrincipal` via CDI without any exclusion config. Add `@RolesAllowed` to the admin endpoints, `@PermitAll` to the webhook. Add `@TestSecurity` to the test classes. The CDI displacement chain — `@DefaultBean` yields to `@RequestScoped` yields to `@Alternative @Priority(1)` in tests — worked exactly as designed.

## The Part That Wasn't Obvious

Quarkus has `deny-unannotated-endpoints=true` — supposed to be the safety net that catches any endpoint you forgot to annotate. I assumed it failed the build. It doesn't. Augmentation adds `@DenyAll` to unannotated endpoints, the build succeeds, and the endpoint returns 403 at runtime. In dev mode with `auth.enabled-in-dev-mode=false`, even that 403 is bypassed. A developer can add an unannotated endpoint, run locally, see it work, and only discover the problem in production.

That's a meaningful gap. Worth knowing about, but not the real surprise.

The real surprise was what happens with library-provided JAX-RS endpoints. `casehub-engine-actor-state` contributes `ActorStateResource` — a diagnostic endpoint at `/actors/{actorId}/state` with no security annotations. With `deny-unannotated-endpoints=true`, augmentation adds `@DenyAll` to it. The obvious fix is a path-based HTTP permission rule: `quarkus.http.auth.permission.actor-state.policy=permit`. It doesn't work.

Quarkus security has two independent layers evaluated sequentially. The HTTP permission layer (Vert.x routes) evaluates first — the `permit` rule lets the request through. Then the JAX-RS annotation layer (`EagerSecurityHandler`) evaluates second — the augmented `@DenyAll` blocks it. 403. The two layers don't communicate. A path-based permit cannot override a JAX-RS deny.

The correct interim is excluding the bean entirely via `quarkus.arc.exclude-types`. The correct fix is patching the foundation to add `@PermitAll` to the resource. A path-based workaround is a workaround that silently doesn't work — the test would catch it (you'd get 403 instead of 200), but the design should be right before you discover that at runtime.

## Tenant Isolation Was Hiding in Plain Sight

The other finding: `CodeReviewComplianceResource` and `GdprErasureResource` both took `tenancyId` as a `@QueryParam`. Before OIDC, that was the only way to pass a tenant. After OIDC, the principal carries the tenant from the JWT claim — and the query parameter becomes a tenant isolation bypass. Any authenticated caller can read another tenant's compliance evidence or trigger another tenant's GDPR erasure by passing a different `tenancyId`.

The same pattern was in the MCP tools. Seven of eleven tools accepted `tenancy_id` as a `@ToolArg` with a fallback to the default tenant. Same bypass, different transport.

The fix was mechanical: inject `CurrentPrincipal`, use `principal.tenancyId()`, remove the caller-controlled parameter. But the pattern is worth noting because it's the kind of thing that survives indefinitely if nobody looks at the parameter list with fresh eyes after the auth model changes.
