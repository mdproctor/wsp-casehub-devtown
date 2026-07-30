# Configurable Trust Score + RBAC Roles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #114 — configurable default trust score for webhook-admitted PRs
**Issue group:** #114, #91

**Goal:** Replace the hardcoded 0.5 trust score in merge queue admission with
a configurable preference, and expand the RBAC topology from admin-only to
five roles with per-endpoint granularity.

**Architecture:** Both changes are leaf-level — no new modules, no new SPIs,
no migrations. #114 adds one `PreferenceKey` and one `resolvePreferences()`
call. #91 adds four string constants and updates six `@RolesAllowed` annotations.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api (PreferenceProvider),
Jakarta Security (`@RolesAllowed`), RESTAssured + `@TestSecurity` for tests.

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Use `mvn` not `./mvnw`
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing
- `project_path=/Users/mdproctor/claude/casehub/devtown` on every MCP call
- Commits reference issues: `Refs #114` or `Refs #91`

## File Map

| File | Action | Task |
|------|--------|------|
| `domain/src/main/java/.../queue/MergeQueuePreferenceKeys.java` | Modify — add `DEFAULT_ADMISSION_TRUST_SCORE` | 1 |
| `domain/src/test/java/.../queue/MergeQueuePreferenceKeysTest.java` | Modify — add tests for new key | 1 |
| `app/src/main/java/.../MergeQueueService.java` | Modify — replace hardcoded 0.5 in `admit()` | 1 |
| `app/src/test/java/.../MergeQueueAdmissionTrustScoreTest.java` | Create — integration test for configurable trust score | 1 |
| `domain/src/main/java/.../DevtownRoles.java` | Modify — add 4 role constants | 2 |
| `app/src/main/java/.../PrReviewResource.java` | Modify — `@RolesAllowed` | 2 |
| `app/src/main/java/.../CodeReviewComplianceResource.java` | Modify — `@RolesAllowed` | 2 |
| `app/src/main/java/.../GdprErasureResource.java` | Modify — `@RolesAllowed` | 2 |
| `app/src/main/java/.../governance/GovernanceResource.java` | Modify — `@RolesAllowed` | 2 |
| `app/src/test/java/.../DevtownRestSecurityTest.java` | Modify — add role-specific tests | 2 |

---

### Task 1: Configurable default admission trust score (#114)

**Files:**
- Modify: `domain/src/main/java/io/casehub/devtown/domain/queue/MergeQueuePreferenceKeys.java`
- Modify: `domain/src/test/java/io/casehub/devtown/domain/queue/MergeQueuePreferenceKeysTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/MergeQueueService.java` (lines 82–87, `admit()`)
- Create: `app/src/test/java/io/casehub/devtown/app/MergeQueueAdmissionTrustScoreTest.java`

**Interfaces:**
- Consumes: `PreferenceKey<DoublePreference>` from casehub-platform-api, `DoublePreference` from devtown-domain
- Produces: `MergeQueuePreferenceKeys.DEFAULT_ADMISSION_TRUST_SCORE` — used by `MergeQueueService.admit()`

- [ ] **Step 1: Write unit tests for the new preference key**

Add three tests to `MergeQueuePreferenceKeysTest`:

```java
@Test
void defaultAdmissionTrustScore_hasCorrectQualifiedNameAndDefault() {
    PreferenceKey<DoublePreference> key = MergeQueuePreferenceKeys.DEFAULT_ADMISSION_TRUST_SCORE;
    assertEquals("devtown.merge-queue.default-admission-trust-score", key.qualifiedName());
    assertEquals(0.5, key.defaultValue().value(), 0.001);
}

@Test
void defaultAdmissionTrustScore_parse() {
    PreferenceKey<DoublePreference> key = MergeQueuePreferenceKeys.DEFAULT_ADMISSION_TRUST_SCORE;
    DoublePreference parsed = key.parser().apply("0.3");
    assertEquals(0.3, parsed.value(), 0.001);
}

@Test
void defaultAdmissionTrustScore_parseInvalid_throws() {
    PreferenceKey<DoublePreference> key = MergeQueuePreferenceKeys.DEFAULT_ADMISSION_TRUST_SCORE;
    assertThrows(NumberFormatException.class, () -> key.parser().apply("not-a-number"));
}
```

Add import for `DoublePreference`:
```java
import io.casehub.devtown.domain.preferences.DoublePreference;
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=MergeQueuePreferenceKeysTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `DEFAULT_ADMISSION_TRUST_SCORE` does not exist.

- [ ] **Step 3: Add the preference key to MergeQueuePreferenceKeys**

Use `ide_insert_member` to add after `DEQUEUE_ON_UNLABEL`:

```java
public static final PreferenceKey<DoublePreference> DEFAULT_ADMISSION_TRUST_SCORE =
    new PreferenceKey<>("devtown.merge-queue", "default-admission-trust-score",
        DoublePreference.of(0.5), DoublePreference::parse);
```

Add import:
```java
import io.casehub.devtown.domain.preferences.DoublePreference;
```

- [ ] **Step 4: Run unit tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl domain -Dtest=MergeQueuePreferenceKeysTest`
Expected: all pass, including the 3 new tests.

- [ ] **Step 5: Write integration test for configurable trust score in admit()**

Create `app/src/test/java/io/casehub/devtown/app/MergeQueueAdmissionTrustScoreTest.java`:

```java
package io.casehub.devtown.app;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.devtown.domain.queue.MergeQueuePreferenceKeys;
import io.casehub.devtown.merge.AdmissionResult;
import io.casehub.devtown.merge.MergeQueueStore;
import io.casehub.devtown.merge.QueueEntry;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import io.quarkus.test.security.TestSecurity;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import io.quarkus.hibernate.orm.PersistenceUnit;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
@TestProfile(BatchFormationTestProfile.class)
@TestSecurity(user = "devtown-admin", roles = {"devtown-admin"})
class MergeQueueAdmissionTrustScoreTest {

    @Inject MergeQueueService mergeQueueService;
    @Inject MergeQueueStore store;

    @Inject
    @PersistenceUnit("qhorus")
    EntityManager em;

    @BeforeEach
    @Transactional
    void cleanAll() {
        em.createQuery("DELETE FROM QueuedPrEntity").executeUpdate();
        em.createQuery("DELETE FROM BatchEntity").executeUpdate();
    }

    @Test
    void admit_usesDefaultTrustScore_whenNoPreferenceConfigured() {
        AdmissionResult result = mergeQueueService.admit(100, "casehubio/devtown", "abc123", "alice");

        assertThat(result).isEqualTo(AdmissionResult.ENQUEUED);

        var entries = store.queued();
        assertThat(entries).hasSize(1);
        QueueEntry entry = entries.get(0);
        assertThat(entry.pr().trustScore())
            .isCloseTo(MergeQueuePreferenceKeys.DEFAULT_ADMISSION_TRUST_SCORE.defaultValue().value(),
                org.assertj.core.data.Offset.offset(0.001));
    }

    @Test
    void admit_idempotent_duplicateReturnsAlreadyQueued() {
        mergeQueueService.admit(200, "casehubio/devtown", "def456", "bob");
        AdmissionResult second = mergeQueueService.admit(200, "casehubio/devtown", "def456", "bob");

        assertThat(second).isEqualTo(AdmissionResult.ALREADY_QUEUED);
        assertThat(store.queued()).hasSize(1);
    }
}
```

- [ ] **Step 6: Run integration test to verify it passes with the default**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=MergeQueueAdmissionTrustScoreTest`
Expected: both tests pass (default 0.5 is still hardcoded, matching the key default).

- [ ] **Step 7: Replace hardcoded 0.5 in MergeQueueService.admit()**

Use `ide_replace_member` on `MergeQueueService.admit` to replace the body:

```java
    Preferences prefs = resolvePreferences();
    double trustScore = prefs.getOrDefault(MergeQueuePreferenceKeys.DEFAULT_ADMISSION_TRUST_SCORE).value();
    QueuedPr pr = new QueuedPr(prNumber, repository, headSha, author,
        trustScore, PriorityLane.NORMAL, Instant.now(), Set.of());
    boolean inserted = enqueue(pr);
    return inserted ? AdmissionResult.ENQUEUED : AdmissionResult.ALREADY_QUEUED;
```

Add import:
```java
import io.casehub.devtown.domain.queue.MergeQueuePreferenceKeys;
import io.casehub.platform.api.preferences.Preferences;
```

Remove the now-unnecessary Javadoc reference to "trust=0.5" — update the
`admit()` Javadoc to say "trust score resolved from preferences".

- [ ] **Step 8: Run all merge queue tests to verify nothing broke**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest="MergeQueue*"`
Expected: all pass.

- [ ] **Step 9: Run diagnostics**

Use `ide_diagnostics` on `MergeQueueService.java` and `MergeQueuePreferenceKeys.java`.
Expected: no errors.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  domain/src/main/java/io/casehub/devtown/domain/queue/MergeQueuePreferenceKeys.java \
  domain/src/test/java/io/casehub/devtown/domain/queue/MergeQueuePreferenceKeysTest.java \
  app/src/main/java/io/casehub/devtown/app/MergeQueueService.java \
  app/src/test/java/io/casehub/devtown/app/MergeQueueAdmissionTrustScoreTest.java
```

Message: `feat(#114): configurable default trust score for webhook-admitted PRs`

---

### Task 2: RBAC role expansion (#91)

**Files:**
- Modify: `domain/src/main/java/io/casehub/devtown/domain/DevtownRoles.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewResource.java` (line 18)
- Modify: `app/src/main/java/io/casehub/devtown/app/CodeReviewComplianceResource.java` (line 27)
- Modify: `app/src/main/java/io/casehub/devtown/app/GdprErasureResource.java` (line 21)
- Modify: `app/src/main/java/io/casehub/devtown/app/governance/GovernanceResource.java` (line 27)
- Modify: `app/src/test/java/io/casehub/devtown/app/DevtownRestSecurityTest.java`

**Interfaces:**
- Consumes: nothing from Task 1
- Produces: `DevtownRoles.ENGINEER`, `.AUDITOR`, `.DATA_CONTROLLER`, `.SERVICE` constants

- [ ] **Step 1: Write tests for new role constants and endpoint access**

Expand `DevtownRestSecurityTest` with role-specific test methods. Add these
tests after the existing methods:

```java
@Test
@TestSecurity(user = "engineer-user", roles = {DevtownRoles.ENGINEER})
void engineer_canPostReviews() {
    given().contentType("application/json")
            .body("{\"repo\":\"test/repo\",\"prNumber\":1,\"headSha\":\"abc\",\"linesChanged\":10,\"baseRef\":\"main\",\"contributor\":\"dev\",\"changedPaths\":[]}")
            .when().post("/api/reviews")
            .then().statusCode(200);
}

@Test
@TestSecurity(user = "engineer-user", roles = {DevtownRoles.ENGINEER})
void engineer_canReadCompliance() {
    given().when().get("/api/compliance/code-review/00000000-0000-0000-0000-000000000001")
            .then().statusCode(404);
}

@Test
@TestSecurity(user = "engineer-user", roles = {DevtownRoles.ENGINEER})
void engineer_canReadGovernance() {
    given().when().get("/api/governance/queue-status")
            .then().statusCode(200);
}

@Test
@TestSecurity(user = "engineer-user", roles = {DevtownRoles.ENGINEER})
void engineer_cannotErase() {
    given().contentType("application/json")
            .body("{\"reason\":\"test\"}")
            .when().post("/api/actors/test-actor/erasure")
            .then().statusCode(403);
}

@Test
@TestSecurity(user = "engineer-user", roles = {DevtownRoles.ENGINEER})
void engineer_cannotRecordIncidentFeedback() {
    given().contentType("application/json")
            .body("{}")
            .when().post("/api/incident-feedback")
            .then().statusCode(403);
}

@Test
@TestSecurity(user = "auditor-user", roles = {DevtownRoles.AUDITOR})
void auditor_canReadCompliance() {
    given().when().get("/api/compliance/code-review/00000000-0000-0000-0000-000000000001")
            .then().statusCode(404);
}

@Test
@TestSecurity(user = "auditor-user", roles = {DevtownRoles.AUDITOR})
void auditor_canReadGovernance() {
    given().when().get("/api/governance/queue-status")
            .then().statusCode(200);
}

@Test
@TestSecurity(user = "auditor-user", roles = {DevtownRoles.AUDITOR})
void auditor_cannotPostReviews() {
    given().contentType("application/json")
            .body("{\"repo\":\"test/repo\",\"prNumber\":1,\"headSha\":\"abc\",\"linesChanged\":10,\"baseRef\":\"main\",\"contributor\":\"dev\",\"changedPaths\":[]}")
            .when().post("/api/reviews")
            .then().statusCode(403);
}

@Test
@TestSecurity(user = "auditor-user", roles = {DevtownRoles.AUDITOR})
void auditor_cannotErase() {
    given().contentType("application/json")
            .body("{\"reason\":\"test\"}")
            .when().post("/api/actors/test-actor/erasure")
            .then().statusCode(403);
}

@Test
@TestSecurity(user = "dc-user", roles = {DevtownRoles.DATA_CONTROLLER})
void dataController_canErase() {
    given().contentType("application/json")
            .body("{\"reason\":\"gdpr-request\"}")
            .when().post("/api/actors/test-actor/erasure")
            .then().statusCode(200);
}

@Test
@TestSecurity(user = "dc-user", roles = {DevtownRoles.DATA_CONTROLLER})
void dataController_cannotPostReviews() {
    given().contentType("application/json")
            .body("{\"repo\":\"test/repo\",\"prNumber\":1,\"headSha\":\"abc\",\"linesChanged\":10,\"baseRef\":\"main\",\"contributor\":\"dev\",\"changedPaths\":[]}")
            .when().post("/api/reviews")
            .then().statusCode(403);
}

@Test
@TestSecurity(user = "service-user", roles = {DevtownRoles.SERVICE})
void service_canPostReviews() {
    given().contentType("application/json")
            .body("{\"repo\":\"test/repo\",\"prNumber\":1,\"headSha\":\"abc\",\"linesChanged\":10,\"baseRef\":\"main\",\"contributor\":\"dev\",\"changedPaths\":[]}")
            .when().post("/api/reviews")
            .then().statusCode(200);
}

@Test
@TestSecurity(user = "service-user", roles = {DevtownRoles.SERVICE})
void service_cannotReadGovernance() {
    given().when().get("/api/governance/queue-status")
            .then().statusCode(403);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownRestSecurityTest`
Expected: compilation errors — `ENGINEER`, `AUDITOR`, `DATA_CONTROLLER`, `SERVICE` do not exist.

- [ ] **Step 3: Add role constants to DevtownRoles**

Use `ide_edit_member` on `DevtownRoles` (member=`DevtownRoles`) to replace the class:

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

- [ ] **Step 4: Run tests to verify they compile but fail on 403/401**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownRestSecurityTest`
Expected: new tests fail — engineers get 403 on `/api/reviews`, etc.

- [ ] **Step 5: Update @RolesAllowed on PrReviewResource**

Use `ide_edit_member` (file=`PrReviewResource.java`, member=`PrReviewResource`)
to replace the class declaration annotation:

```java
@Path("/api/reviews")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@RolesAllowed({DevtownRoles.ADMIN, DevtownRoles.ENGINEER, DevtownRoles.SERVICE})
public class PrReviewResource {
```

- [ ] **Step 6: Update @RolesAllowed on CodeReviewComplianceResource**

Use `ide_edit_member` (file=`CodeReviewComplianceResource.java`,
member=`CodeReviewComplianceResource`) to replace the class declaration:

```java
@Path("/api/compliance/code-review")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@RolesAllowed({DevtownRoles.ADMIN, DevtownRoles.ENGINEER, DevtownRoles.AUDITOR})
public class CodeReviewComplianceResource {
```

- [ ] **Step 7: Update @RolesAllowed on GdprErasureResource**

Use `ide_edit_member` (file=`GdprErasureResource.java`,
member=`GdprErasureResource`) to replace the class declaration:

```java
@Path("/api/actors/{actorId}/erasure")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@RolesAllowed({DevtownRoles.ADMIN, DevtownRoles.DATA_CONTROLLER})
public class GdprErasureResource {
```

- [ ] **Step 8: Update @RolesAllowed on GovernanceResource**

Use `ide_edit_member` (file=`GovernanceResource.java`,
member=`GovernanceResource`) to replace the class declaration:

```java
@Path("/api/governance")
@Produces(MediaType.APPLICATION_JSON)
@RolesAllowed({DevtownRoles.ADMIN, DevtownRoles.ENGINEER, DevtownRoles.AUDITOR})
public class GovernanceResource {
```

- [ ] **Step 9: Run all security tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=DevtownRestSecurityTest`
Expected: all pass — engineers can POST reviews and read governance/compliance,
auditors can read governance/compliance but not POST, data controllers can erase
but not review, services can POST reviews but not read governance.

- [ ] **Step 10: Run diagnostics on all modified files**

Use `ide_diagnostics` on each modified resource file.
Expected: no errors.

- [ ] **Step 11: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app`
Expected: all pass. Existing tests that use `@TestSecurity(roles = {"devtown-admin"})`
continue to work because ADMIN is still in every `@RolesAllowed` set.

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  domain/src/main/java/io/casehub/devtown/domain/DevtownRoles.java \
  app/src/main/java/io/casehub/devtown/app/PrReviewResource.java \
  app/src/main/java/io/casehub/devtown/app/CodeReviewComplianceResource.java \
  app/src/main/java/io/casehub/devtown/app/GdprErasureResource.java \
  app/src/main/java/io/casehub/devtown/app/governance/GovernanceResource.java \
  app/src/test/java/io/casehub/devtown/app/DevtownRestSecurityTest.java
```

Message: `feat(#91): expand RBAC role topology — engineer, auditor, data-controller, service`
