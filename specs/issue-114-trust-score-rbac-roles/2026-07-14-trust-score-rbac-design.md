# Configurable Default Trust Score + RBAC Role Expansion

**Issues:** #114, #91
**Date:** 2026-07-14
**Status:** Approved

## Problem

Two independent gaps in devtown's configuration and security:

1. **#114:** `MergeQueueService.admit()` hardcodes `0.5` as the trust score for
   webhook-admitted PRs. This should be configurable via `PreferenceProvider`
   so operators can tune the default without code changes.

2. **#91:** `DevtownRoles` has only `ADMIN`. Every authenticated endpoint is
   admin-or-nothing. The system needs finer-grained roles for engineers,
   auditors, data controllers, and service accounts.

## Design

### #114 — Configurable default trust score

**Change 1:** Add `DEFAULT_ADMISSION_TRUST_SCORE` to `MergeQueuePreferenceKeys`.

```java
public static final PreferenceKey<DoublePreference> DEFAULT_ADMISSION_TRUST_SCORE =
    new PreferenceKey<>("devtown.merge-queue", "default-admission-trust-score",
        DoublePreference.of(0.5), DoublePreference::parse);
```

Namespace `devtown.merge-queue` matches the existing merge-queue keys. Default
value `0.5` preserves current behaviour when no YAML config is present.

**Change 2:** Replace hardcoded value in `MergeQueueService.admit()`.

```java
@Override
public AdmissionResult admit(int prNumber, String repository, String headSha, String author) {
    Preferences prefs = resolvePreferences();
    double trustScore = prefs.getOrDefault(MergeQueuePreferenceKeys.DEFAULT_ADMISSION_TRUST_SCORE).value();
    QueuedPr pr = new QueuedPr(prNumber, repository, headSha, author,
        trustScore, PriorityLane.NORMAL, Instant.now(), Set.of());
    boolean inserted = enqueue(pr);
    return inserted ? AdmissionResult.ENQUEUED : AdmissionResult.ALREADY_QUEUED;
}
```

`getOrDefault()` is mandatory — `ConfigFilePreferenceProvider` returns empty
`Preferences` when YAML is missing (GE-20260603-5a5cc0).

Scope is global (`SettingsScope.of("devtown", "merge-queue")`), not per-repo.
Per-repo scoping is a future concern.

### #91 — RBAC role expansion

**Change 1:** Add four role constants to `DevtownRoles`.

```java
public final class DevtownRoles {
    public static final String ADMIN           = "devtown-admin";
    public static final String ENGINEER        = "devtown-engineer";
    public static final String AUDITOR         = "devtown-auditor";
    public static final String DATA_CONTROLLER = "devtown-data-controller";
    public static final String SERVICE         = "devtown-service";

    private DevtownRoles() {}
}
```

**Change 2:** Update `@RolesAllowed` annotations.

| Resource | Path | Current | New |
|----------|------|---------|-----|
| `PrReviewResource` | POST /api/reviews | ADMIN | ADMIN, ENGINEER, SERVICE |
| `CodeReviewComplianceResource` | GET /api/compliance/code-review/* | ADMIN | ADMIN, ENGINEER, AUDITOR |
| `GdprErasureResource` | POST /api/actors/*/erasure | ADMIN | ADMIN, DATA_CONTROLLER |
| `GovernanceResource` | GET /api/governance/* | ADMIN | ADMIN, ENGINEER, AUDITOR |
| `IncidentFeedbackResource` | POST /api/incidents/feedback | ADMIN | ADMIN (unchanged) |
| `MemoryAdminResource` | POST /api/memory/* | ADMIN | ADMIN (unchanged) |
| `GitHubWebhookResource` | POST /api/github/webhook | PermitAll | PermitAll (unchanged) |

**Rationale for unchanged endpoints:**
- IncidentFeedbackResource: directly affects trust scores — operational decision
- MemoryAdminResource: destructive operation on case memory — admin only
- GitHubWebhookResource: HMAC signature is the auth mechanism, not OIDC roles

**Rationale for GovernanceResource opening:**
- All endpoints are GET-only (no mutation risk)
- devtown's design philosophy is transparency and auditability
- Engineers need dashboard data to understand PR review state
- Auditors need it to verify operational health
- No per-method splitting needed — all reads serve the same observability purpose

## Scope

- No Flyway migrations
- No new modules
- No new REST endpoints
- No changes to `MergeQueuePort` interface (trust score is a configuration concern, not a caller concern)
- OIDC provider role assignment is IDP configuration, not a code change

## Testing

### #114
- Unit test: `MergeQueuePreferenceKeysTest` — verify key namespace, name, default value, parsing
- Integration test: `MergeQueueService.admit()` uses resolved preference value instead of hardcoded 0.5
- Edge case: custom trust score (e.g. 0.3) flows through to `QueuedPr`

### #91
- Unit test: `DevtownRoles` constants match expected string values
- Integration test: verify each resource's `@RolesAllowed` accepts the correct roles
- Test with `@TestSecurity` annotations for each role permutation
