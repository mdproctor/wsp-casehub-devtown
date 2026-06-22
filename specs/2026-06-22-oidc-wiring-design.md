# Design: OIDC wiring — activate CurrentPrincipal.groups() and @RolesAllowed

**Issue:** devtown#90  
**Branch:** `issue-90-wire-platform-oidc`  
**Date:** 2026-06-22  
**Reference implementation:** casehub-life#40 (identical pattern, same session)

---

## Context

`CurrentPrincipal.groups()` always returns empty because `MockCurrentPrincipal @DefaultBean`
is the active production principal — devtown has no OIDC module on the classpath yet
(noted in `application.properties` as "devtown#71"). `MemoryAdminResource` and
`IncidentFeedbackResource` already have `@RolesAllowed("admin")` but use a hardcoded
string literal. `GdprErasureResource` is unguarded despite being a privileged operation.

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

---

## 3. Configuration

### Production application.properties

```properties
# OIDC — devtown#90
# Required deployment env vars:
#   QUARKUS_OIDC_AUTH_SERVER_URL — e.g. https://auth.example.com/realms/devtown
#   QUARKUS_OIDC_CLIENT_ID       — e.g. casehub-devtown
quarkus.oidc.application-type=service

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
# GE-20260521-f50602: discovery-disabled requires jwks-path (lazy-loaded, never fetched with @TestSecurity)
quarkus.oidc.auth-server-url=http://localhost:8180/realms/test
quarkus.oidc.discovery-enabled=false
quarkus.oidc.jwks-path=protocol/openid-connect/certs
quarkus.keycloak.devservices.enabled=false
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

## 5. @RolesAllowed mapping

| Resource | Path | Current | Change |
|---|---|---|---|
| `MemoryAdminResource` | `/api/admin/memory` | `@RolesAllowed("admin")` (class-level) | → `@RolesAllowed(DevtownRoles.ADMIN)` |
| `IncidentFeedbackResource` | `/api/incident-feedback` | `@RolesAllowed("admin")` (class-level) | → `@RolesAllowed(DevtownRoles.ADMIN)` |
| `GdprErasureResource.POST` | `/api/actors/{actorId}/erasure` | none | add `@RolesAllowed(DevtownRoles.ADMIN)` |
| `PrReviewResource.POST` | `/api/reviews` | none | add `@PermitAll` |
| `CodeReviewComplianceResource.GET` | `/api/compliance/code-review/{caseId}` | none | add `@PermitAll` |
| `GitHubWebhookResource.POST` | `/api/github/webhook` | HMAC only | add `@PermitAll` |

**`@PermitAll` rationale:**

- `GitHubWebhookResource` — called by GitHub infrastructure; HMAC-SHA256 signature is its
  authentication mechanism. OIDC is not applicable.
- `PrReviewResource` — webhook calls go through `GitHubWebhookResource` → CDI service
  (not HTTP), so `@RolesAllowed` here only gates direct API callers. `@PermitAll` for
  now; devtown#91 tracks `devtown-service` role for machine-to-machine differentiation.
- `CodeReviewComplianceResource` — read-only compliance evidence; no PII. Tenant isolation
  via `tenancyId` query param is sufficient for now.

---

## 6. Tests

### Existing @QuarkusTest classes — add @TestSecurity

Before adding `@RolesAllowed`, identify all `@QuarkusTest` classes making HTTP calls to
guarded endpoints and add class-level `@TestSecurity(user = "devtown-admin", roles = {"devtown-admin"})`.
Verify with a full test run; any remaining 401 failures indicate an unpatched test class.

### New DevtownRestSecurityTest

`@QuarkusTest` covering authorization boundaries via RestAssured + `@TestSecurity`:

- Unauthenticated → 401 on admin-guarded endpoints (`/api/admin/memory/erase/contributor`,
  `/api/incident-feedback`, `/api/actors/{actorId}/erasure`)
- `devtown-admin` → not 401, not 403 on admin endpoints
- Unauthenticated → not 401 on `@PermitAll` endpoints (`/api/reviews`, `/api/github/webhook`,
  `/api/compliance/code-review/{id}`)
