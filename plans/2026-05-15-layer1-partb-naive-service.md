# Layer 1 Part B — Naive PR Review Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the naive PR review teaching baseline — two DTOs, a port interface, a `@DefaultBean` naive implementation with gap comments, and a thin REST resource, all in `app/`.

**Architecture:** `PrReviewApplicationService` is the CDI displacement boundary; `NaivePrReviewService` implements it with `@ApplicationScoped @DefaultBean` and makes direct stub calls with pedagogical gap comments. Each subsequent layer provides a non-`@DefaultBean` replacement. All new classes live in `app/` — `domain/` is untouched.

**Tech Stack:** Java 21, Quarkus 3.32.2 (`quarkus-rest`, `quarkus-rest-jackson`), JUnit 5, AssertJ 3.27.7

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `app/src/main/java/io/casehub/devtown/app/PrPayload.java` | Input record for a PR review request |
| Create | `app/src/main/java/io/casehub/devtown/app/PrReviewOutcome.java` | Output record: verdict + findings |
| Create | `app/src/main/java/io/casehub/devtown/app/PrReviewApplicationService.java` | Port interface — CDI displacement boundary |
| Create | `app/src/main/java/io/casehub/devtown/app/NaivePrReviewService.java` | `@DefaultBean` naive impl with 5 gap comments |
| Create | `app/src/main/java/io/casehub/devtown/app/PrReviewResource.java` | REST dispatcher: `POST /api/reviews` |
| Create | `app/src/test/java/io/casehub/devtown/app/NaivePrReviewServiceTest.java` | Plain unit tests — no Quarkus |
| Existing | `app/src/test/java/io/casehub/devtown/app/DevtownBootTest.java` | Boot test — run at end to verify CDI |

---

## Task 1: DTOs — PrPayload and PrReviewOutcome

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/PrPayload.java`
- Create: `app/src/main/java/io/casehub/devtown/app/PrReviewOutcome.java`

- [ ] **Step 1: Create PrPayload**

```java
package io.casehub.devtown.app;

public record PrPayload(String repo, int prNumber, String headSha, int linesChanged) {}
```

- [ ] **Step 2: Create PrReviewOutcome**

```java
package io.casehub.devtown.app;

import java.util.List;

public record PrReviewOutcome(String verdict, List<String> findings) {}
```

- [ ] **Step 3: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -q
```
Expected: BUILD SUCCESS with no errors.

---

## Task 2: Port Interface — PrReviewApplicationService

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/PrReviewApplicationService.java`

- [ ] **Step 1: Create the interface**

```java
package io.casehub.devtown.app;

public interface PrReviewApplicationService {
    PrReviewOutcome review(PrPayload pr);
}
```

- [ ] **Step 2: Compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -q
```
Expected: BUILD SUCCESS.

---

## Task 3: Naive Service — failing tests first

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/NaivePrReviewServiceTest.java`
- Create: `app/src/main/java/io/casehub/devtown/app/NaivePrReviewService.java`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.devtown.app;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class NaivePrReviewServiceTest {

    private final NaivePrReviewService service = new NaivePrReviewService();

    @Test
    void review_returnsNonNullOutcome() {
        var pr = new PrPayload("casehubio/devtown", 42, "abc123", 150);
        var outcome = service.review(pr);
        assertThat(outcome).isNotNull();
    }

    @Test
    void review_verdictIsNonBlank() {
        var pr = new PrPayload("casehubio/devtown", 42, "abc123", 150);
        var outcome = service.review(pr);
        assertThat(outcome.verdict()).isNotBlank();
    }

    @Test
    void review_findingsIsNonNull() {
        var pr = new PrPayload("casehubio/devtown", 42, "abc123", 150);
        var outcome = service.review(pr);
        assertThat(outcome.findings()).isNotNull();
    }
}
```

- [ ] **Step 2: Run test — confirm it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=NaivePrReviewServiceTest -q
```
Expected: COMPILATION ERROR — `NaivePrReviewService` does not exist yet.

- [ ] **Step 3: Create NaivePrReviewService**

```java
package io.casehub.devtown.app;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
@DefaultBean
public class NaivePrReviewService implements PrReviewApplicationService {

    @Override
    public PrReviewOutcome review(PrPayload pr) {
        // LAYER 1 GAP: no attribution — which agent ran this analysis? No record.
        var securityFindings = analyzeSecurityDirectly(pr);
        // LAYER 1 GAP: no response SLA — analysis can stall indefinitely with no escalation.
        var architectureFindings = reviewArchitectureDirectly(pr);
        // LAYER 1 GAP: no formal DECLINE — if a specialist can't review, it silently fails or errors.
        // LAYER 1 GAP: no tamper-evident audit trail — cannot trace a production incident to this review.
        // LAYER 1 GAP: no trust weighting — a novice and an expert are treated identically.
        var allFindings = new ArrayList<String>(securityFindings);
        allFindings.addAll(architectureFindings);
        return new PrReviewOutcome("reviewed", allFindings);
    }

    private List<String> analyzeSecurityDirectly(PrPayload pr) {
        return List.of("security-analysis-complete");
    }

    private List<String> reviewArchitectureDirectly(PrPayload pr) {
        return List.of("architecture-review-complete");
    }
}
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=NaivePrReviewServiceTest
```
Expected: 3 tests passing, BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/PrPayload.java \
  app/src/main/java/io/casehub/devtown/app/PrReviewOutcome.java \
  app/src/main/java/io/casehub/devtown/app/PrReviewApplicationService.java \
  app/src/main/java/io/casehub/devtown/app/NaivePrReviewService.java \
  app/src/test/java/io/casehub/devtown/app/NaivePrReviewServiceTest.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): Layer 1 Part B — naive PR review service with gap comments

Refs #27"
```

---

## Task 4: REST Resource — PrReviewResource

**Files:**
- Create: `app/src/main/java/io/casehub/devtown/app/PrReviewResource.java`

- [ ] **Step 1: Create the resource**

```java
package io.casehub.devtown.app;

import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/api/reviews")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class PrReviewResource {

    @Inject
    PrReviewApplicationService service;

    @POST
    public PrReviewOutcome review(PrPayload pr) {
        return service.review(pr);
    }
}
```

- [ ] **Step 2: Run full app module tests including DevtownBootTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app
```
Expected: All tests pass (NaivePrReviewServiceTest × 3 + DevtownBootTest × 1), BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/devtown add \
  app/src/main/java/io/casehub/devtown/app/PrReviewResource.java
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(app): PrReviewResource — POST /api/reviews dispatcher

Refs #27"
```

---

## Task 5: Full build verification

- [ ] **Step 1: Run full devtown build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/devtown/pom.xml
```
Expected: All modules build, all tests pass, BUILD SUCCESS.

- [ ] **Step 2: Confirm test count**

Expected output includes: `Tests run: 4` (3 unit + 1 boot) across the `app` module. Other modules show their existing counts unchanged.
