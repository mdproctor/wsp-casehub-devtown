# Design: OIDC wiring — activate CurrentPrincipal.groups() and @RolesAllowed

**Issue:** devtown#90  
**Branch:** `issue-90-wire-platform-oidc`  
**Date:** 2026-06-22  
**Reference implementation:** casehub-life#40 (identical pattern, same session)  
**Rev:** 4 (rev 3 + ActorStateResource interim fix, residual language corrections)

---

## Context

`CurrentPrincipal.groups()` always returns empty because `MockCurrentPrincipal @DefaultBean`
is the active production principal — devtown has no OIDC module on the classpath yet
(noted in `application.properties` as "devtown#71"). `MemoryAdminResource` and
`IncidentFeedbackResource` already have `@RolesAllowed("admin")` but use a hardcoded
string literal. `GdprErasureResource` is unguarded despite being a privileged operation.
`CodeReviewComplianceResource` and `GdprErasureResource` take `tenancyId` from
`@QueryParam` — a caller-controlled parameter that bypasses tenant isolation once OIDC
provides a principal-carried tenancy claim.

---

## 1. Dependencies (app/pom.xml)

```xml
<!-- OidcCurrentPrincipal @RequestScoped — displaces MockCurrentPrincipal @DefaultBean -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-oidc</artifactId>
</dependency>

<!-- @TestSecurity — controls SecurityIdentity in @QuarkusTest without a real OIDC server -->
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-test-security</artifactId>
  <scope>test</scope>
</dependency>
```

`quarkus-oidc` is transitive through `casehub-platform-oidc`; no explicit declaration needed.

---

## 2. CDI wiring

**Production:** `OidcCurrentPrincipal @RequestScoped` displaces `MockCurrentPrincipal @DefaultBean`
automatically — `@DefaultBean` yields to any other implementation, so no new exclusions are
needed. `TenantScopedPrincipal` and `QhorusInboundCurrentPrincipal` are already in
`quarkus.arc.exclude-types`. The comment in `application.properties` referencing "devtown#71"
(no auth system yet) should be updated to reference devtown#90 instead.

**Tests:** `FixedCurrentPrincipal @Alternative @Priority(1)` from `casehub-platform-testing`
(already a test dep) globally displaces `OidcCurrentPrincipal @RequestScoped` — no new
exclusions needed. `@TestSecurity` controls `SecurityIdentity` for `@RolesAllowed` checks;
`FixedCurrentPrincipal` controls `CurrentPrincipal` for business logic.

**Injection constraint:** `OidcCurrentPrincipal` must never be injected by concrete type —
always inject `CurrentPrincipal` (the interface). `OidcCurrentPrincipal` injects both
`SecurityIdentity` and `JsonWebToken`; in test mode with `@TestSecurity`, a mock
`SecurityIdentity` is provided but `JsonWebToken` resolution depends on the OIDC test
infrastructure being wired. `FixedCurrentPrincipal @Alternative @Priority(1)` displaces
at the `CurrentPrincipal` level only — direct injection of `OidcCurrentPrincipal` bypasses
the test displacement chain.

---

## 3. Configuration

### Production application.properties

```properties
# OIDC — devtown#90
# Required deployment env vars:
#   QUARKUS_OIDC_AUTH_SERVER_URL — e.g. https://auth.example.com/realms/devtown
#   QUARKUS_OIDC_CLIENT_ID       — e.g. casehub-devtown
quarkus.oidc.application-type=service

# Default-deny: every JAX-RS endpoint must carry @RolesAllowed or @PermitAll.
# Augmentation adds @DenyAll to unannotated endpoints — the build succeeds,
# but the endpoint returns 403 Forbidden at runtime. In dev mode with
# auth.enabled-in-dev-mode=false, @DenyAll is bypassed entirely — the gap
# only surfaces in production or in tests with security enabled.
quarkus.security.jaxrs.deny-unannotated-endpoints=true

# Dev profile — disable OIDC and security enforcement.
# DevModeDisabledAuthorizationController (quarkus-security-runtime-spi 3.32.2) makes
# isAuthorizationEnabled()=false, bypassing EagerSecurityHandler and the CDI interceptor.
%dev.quarkus.security.auth.enabled-in-dev-mode=false
%dev.quarkus.oidc.enabled=false
%dev.quarkus.keycloak.devservices.enabled=false
```

### Test application.properties

```properties
# OIDC test config — devtown#90
# These properties are test-only (no equivalent in main application.properties),
# so no %test. prefix is needed — the file itself is test-scoped.
# GE-20260521-f50602: discovery-disabled requires jwks-path (lazy-loaded, never fetched with @TestSecurity)
quarkus.oidc.auth-server-url=http://localhost:8180/realms/test
quarkus.oidc.discovery-enabled=false
quarkus.oidc.jwks-path=protocol/openid-connect/certs
quarkus.keycloak.devservices.enabled=false

# Default-deny must also apply in tests — otherwise tests pass on unannotated
# endpoints that would fail in production.
quarkus.security.jaxrs.deny-unannotated-endpoints=true
```

---

## 4. DevtownRoles constants

New class in `devtown-domain` (pure Java, no Quarkus deps — accessible from `app/`
which depends on `domain/`):

```java
package io.casehub.devtown.domain;

public final class DevtownRoles {
    /** Platform administrators: incident feedback, memory erasure, GDPR erasure. */
    public static final String ADMIN = "devtown-admin";

    private DevtownRoles() {}
}
```

Role is renamed from the existing `"admin"` string to `"devtown-admin"` for specificity —
prevents collision with IDP's generic admin role and forces explicit IDP configuration.

**Richer role set** (devtown#91) deferred: `devtown-engineer`, `devtown-auditor`,
`devtown-data-controller`, `devtown-service` tracked for when the need arises.

---

## 5. @RolesAllowed mapping — JAX-RS endpoints (devtown-owned)

| Resource | Path | Current | Change |
|---|---|---|---|
| `MemoryAdminResource` | `/api/admin/memory` | `@RolesAllowed("admin")` (class-level) | → `@RolesAllowed(DevtownRoles.ADMIN)` |
| `IncidentFeedbackResource` | `/api/incident-feedback` | `@RolesAllowed("admin")` (class-level) | → `@RolesAllowed(DevtownRoles.ADMIN)` |
| `GdprErasureResource.POST` | `/api/actors/{actorId}/erasure` | none | add `@RolesAllowed(DevtownRoles.ADMIN)` |
| `CodeReviewComplianceResource.GET` | `/api/compliance/code-review/{caseId}` | none | add `@RolesAllowed(DevtownRoles.ADMIN)` |
| `PrReviewResource.POST` | `/api/reviews` | none | add `@RolesAllowed(DevtownRoles.ADMIN)` |
| `GitHubWebhookResource.POST` | `/api/github/webhook` | HMAC only | add `@PermitAll` |

**Annotation rationale:**

- `GitHubWebhookResource` — called by GitHub infrastructure; HMAC-SHA256 signature is its
  authentication mechanism. OIDC is not applicable. `@PermitAll` is required (not decorative)
  because `deny-unannotated-endpoints=true` would otherwise add `@DenyAll` via
  augmentation, returning 403 at runtime.
- `PrReviewResource` — webhook calls go through `GitHubWebhookResource` → CDI service
  (not HTTP), so `@RolesAllowed` here only gates direct API callers. Starting restrictive:
  `@RolesAllowed(DevtownRoles.ADMIN)` now. devtown#91 tracks `devtown-service` role for
  machine-to-machine differentiation — relax to
  `@RolesAllowed({DevtownRoles.ADMIN, DevtownRoles.SERVICE})` when it lands.
- `CodeReviewComplianceResource` — compliance evidence includes trust routing decisions,
  audit chain digests, and GDPR erasure receipt IDs. This is privileged data.
  `@RolesAllowed(DevtownRoles.ADMIN)` now; relax to `devtown-auditor` when devtown#91 ships.

### Tenant isolation fix — remove `@QueryParam("tenancyId")`

`CodeReviewComplianceResource` and `GdprErasureResource` currently take `tenancyId`
as a `@QueryParam`. With OIDC providing `CurrentPrincipal.tenancyId()` from the JWT
`tenancyId` claim, the caller-supplied parameter is a tenant isolation bypass.

**Fix for both resources:**
1. Inject `CurrentPrincipal`
2. Use `principal.tenancyId()` as the authoritative tenant identifier
3. Remove `@QueryParam("tenancyId")` parameter

`MemoryAdminResource` already injects `CurrentPrincipal` and uses `principal.tenancyId()`
— no change needed.

**CodeReviewComplianceResource after:**

```java
@Path("/api/compliance/code-review")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@RolesAllowed(DevtownRoles.ADMIN)
public class CodeReviewComplianceResource {

    @Inject CodeReviewComplianceService service;
    @Inject CurrentPrincipal principal;

    @GET
    @Path("/{caseId}")
    public Response getEvidence(@PathParam("caseId") UUID caseId) {
        return service.findEvidence(caseId, principal.tenancyId())
                .map(evidence -> Response.ok(evidence).build())
                .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }
}
```

**GdprErasureResource after:**

```java
@Path("/api/actors/{actorId}/erasure")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@RolesAllowed(DevtownRoles.ADMIN)
public class GdprErasureResource {

    @Inject GdprErasureService service;
    @Inject CurrentPrincipal principal;

    @POST
    public ErasureReceipt erase(
            @PathParam("actorId") final String actorId,
            final ErasureRequest request) {
        return service.erase(actorId, principal.tenancyId(),
                request != null ? request.reason() : null);
    }

    public record ErasureRequest(String reason) {}
}
```

---

## 6. Library-provided JAX-RS endpoints

`casehub-engine-actor-state` (already in `app/pom.xml`) contributes two JAX-RS resources
via Jandex discovery. Only one is active at a time:

| Resource | Path | Active when | Security |
|---|---|---|---|
| `ActorStateResource` | `GET /actors/{actorId}/state` | `casehub.qhorus.reactive.enabled` absent or `false` (default) | **none** |
| `ReactiveActorStateResource` | `GET /actors/{actorId}/state` | `casehub.qhorus.reactive.enabled=true` | **none** |

Neither carries security annotations. With `deny-unannotated-endpoints=true`, augmentation
adds `@DenyAll` to the active resource — requests return 403 at runtime.

**Note on path-based rules:** a `quarkus.http.auth.permission` `permit` rule cannot fix
this. Quarkus security has two independent layers evaluated sequentially: (1) HTTP
permission layer (Vert.x routes) evaluates `quarkus.http.auth.permission.*` first, then
(2) JAX-RS annotation layer (`EagerSecurityHandler`) evaluates `@RolesAllowed`/`@DenyAll`/
`@PermitAll` second. A path-based `permit` lets the request through the HTTP layer, but the
augmented `@DenyAll` blocks it at the JAX-RS layer → 403. The two layers are independent.

**Right fix:** patch the foundation — add `@PermitAll` to both resources in
`casehub-engine-actor-state`. One-line change per class. No end users, breaking changes
cost nothing. Ship the engine patch before devtown enables `deny-unannotated-endpoints=true`.
File: `casehubio/engine` issue.

**Interim fix (if engine can't ship first):** exclude the bean entirely via
`quarkus.arc.exclude-types` in devtown's `application.properties`:

```properties
# Interim: ActorStateResource (casehub-engine-actor-state) has no security annotations.
# deny-unannotated-endpoints=true adds @DenyAll via augmentation — 403 at runtime.
# Path-based permit cannot override JAX-RS @DenyAll (independent security layers).
# Excluding the bean removes the endpoint entirely until the foundation adds @PermitAll.
# ActorStateResource is diagnostic — not required for the PR review workflow.
quarkus.arc.exclude-types=...,io.casehub.actorstate.ActorStateResource,io.casehub.actorstate.ReactiveActorStateResource
```

Append to the existing `quarkus.arc.exclude-types` line. Remove when the engine ships
with `@PermitAll`.

---

## 7. Security boundary — annotation vs path-based

Devtown has three categories of HTTP endpoint, each requiring a different security mechanism.
`deny-unannotated-endpoints=true` covers only the first category.

### Category 1: Devtown-owned JAX-RS endpoints → annotations

All resources in `app/` and `github/` — covered by §5. Every endpoint carries `@RolesAllowed`
or `@PermitAll`. `deny-unannotated-endpoints=true` enforces completeness at runtime (403
on unannotated endpoints).

### Category 2: Library-provided JAX-RS endpoints → patch foundation (or exclude)

`ActorStateResource` / `ReactiveActorStateResource` from `casehub-engine-actor-state` —
covered by §6. Foundation should carry its own annotations. Path-based rules cannot
override augmented `@DenyAll` (independent security layers — see §6). If the foundation
can't ship first, exclude the bean via `quarkus.arc.exclude-types`.

### Category 3: Non-JAX-RS Vert.x routes → path-based rules

These routes are not reachable by `@RolesAllowed`, `@PermitAll`, or
`deny-unannotated-endpoints` (JAX-RS scoped only).

**MCP transport** — `DevtownMcpTools` exposes 11 tools (8 read, 3 write) via
`quarkus-mcp-server-http` on the `/mcp` path. Write operations (`retry_reviewer`,
`reroute_review`, `force_complete_check`) have significant side effects: cancelling cases,
force-completing checks with operator overrides, starting new cases. The MCP server extension
uses Vert.x routes, not JAX-RS.

Security stance: MCP is an operator interface intended for localhost-only access (Claude CLI
sessions, local debugging). In current deployments it is not network-exposed. However, the
spec must not rely on deployment topology for security — if the application is network-exposed,
MCP must be gated.

```properties
# MCP transport — restrict to authenticated operators.
# quarkus-mcp-server-http registers Vert.x routes on /mcp/*.
# Path-based rule because MCP is not JAX-RS.
quarkus.http.auth.permission.mcp.paths=/mcp/*
quarkus.http.auth.permission.mcp.policy=authenticated
```

This requires an authenticated principal for all MCP calls. In dev mode,
`auth.enabled-in-dev-mode=false` bypasses this — local Claude CLI sessions are unaffected.

**MCP tenant isolation** — 7 of 11 tools accept `tenancy_id` as a `@ToolArg` with
fallback to `TenancyConstants.DEFAULT_TENANT_ID`. This is the same tenant isolation
bypass that §5 fixes for REST. With the `authenticated` path-based rule, a
`SecurityIdentity` is present and `OidcCurrentPrincipal @RequestScoped` activates
per-request for Vert.x routes — `CurrentPrincipal` is injectable from `DevtownMcpTools
@ApplicationScoped` via the request-scoped CDI proxy.

**Fix:** inject `CurrentPrincipal`, use `principal.tenancyId()`, remove `tenancy_id`
`@ToolArg` from all 7 tools:

| Tool | Line | Change |
|---|---|---|
| `inspectReview` | 305 | remove `tenancy_id` param; inject `principal.tenancyId()` |
| `getReviewerHealth` | 367 | same |
| `getPriorDecisions` | 416 | same |
| `exportProv` | 457 | same |
| `retryReviewer` | 473 | same |
| `rerouteReview` | 501 | same |
| `forceCompleteCheck` | 549 | same |

`getQueueStatus`, `getRecentEvents`, `getSystemHealth`, `listProblems` do not take
`tenancy_id` — no change needed.

**SmallRye Health** — `/q/health`, `/q/health/ready`, `/q/health/live` are Vert.x routes
contributed by the SmallRye Health extension. They must remain accessible for container
orchestration probes (Kubernetes liveness/readiness). They are unaffected by
`deny-unannotated-endpoints` (JAX-RS scoped). No path-based rule needed — Quarkus
health endpoints are public by default and should remain so.

**Policy summary:**

| Category | Mechanism | Enforced at |
|---|---|---|
| Devtown JAX-RS | `@RolesAllowed` / `@PermitAll` | Runtime (403 via augmented `@DenyAll`) |
| Library JAX-RS | Patch foundation; exclude bean as interim | Build time (bean removal) |
| MCP (Vert.x) | `quarkus.http.auth.permission` | Runtime |
| Health (Vert.x) | Public by default | — |

---

## 8. Tests

### Existing @QuarkusTest classes — add @TestSecurity

Before adding `@RolesAllowed`, identify all `@QuarkusTest` classes making HTTP calls to
guarded endpoints and add class-level `@TestSecurity(user = "devtown-admin", roles = {"devtown-admin"})`.
Verify with a full test run; any remaining 401 failures indicate an unpatched test class.

### New DevtownRestSecurityTest

`@QuarkusTest` covering authorization boundaries via RestAssured + `@TestSecurity`:

- Unauthenticated → 401 on admin-guarded endpoints (`/api/admin/memory/erase/contributor`,
  `/api/incident-feedback`, `/api/actors/{actorId}/erasure`, `/api/reviews`,
  `/api/compliance/code-review/{id}`)
- `devtown-admin` → not 401, not 403 on admin endpoints
- Unauthenticated → not 401 on `@PermitAll` endpoints (`/api/github/webhook`)
- If `ActorStateResource` is excluded: verify `/actors/{actorId}/state` returns 404 (not 403)
- If foundation shipped `@PermitAll`: unauthenticated → 200 on `/actors/{actorId}/state`

### Test cleanup — remove dead `@QueryParam("tenancyId")` assertions

After removing `@QueryParam("tenancyId")` from `GdprErasureResource` and
`CodeReviewComplianceResource`, JAX-RS silently ignores caller-supplied query params
that no longer map to a method parameter. Tests pass but `.queryParam("tenancyId", ...)`
calls become dead code:

- `GdprErasureResourceTest` — 5 occurrences (lines 44, 72, 103, 112, 129)
- `CodeReviewComplianceServiceTest` — 1 occurrence (line 160, RestAssured call)

Remove the `.queryParam("tenancyId", ...)` calls from both test classes. The tenant
is now carried by `CurrentPrincipal` (controlled in tests via `FixedCurrentPrincipal`).

### Foundation issue to file

`casehubio/engine` — add `@PermitAll` to `ActorStateResource` and
`ReactiveActorStateResource` in `casehub-engine-actor-state`. Once shipped, remove the
`ActorStateResource` and `ReactiveActorStateResource` entries from
`quarkus.arc.exclude-types` in devtown.
