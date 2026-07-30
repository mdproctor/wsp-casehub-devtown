# GitHub Webhook Receiver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Receive GitHub PR webhook events (opened/synchronize/closed/reopened) and drive PR review case lifecycle through the existing CasePlanModel.

**Architecture:** Hexagonal adapter in `github/` module → `PrReviewApplicationService` port in `review/` → `PrReviewCaseService` implementation in `app/`. Pure Java signature verifier, minimal DTO records, secondary case tracker index.

**Tech Stack:** Java 21, Quarkus 3.32.2 (JAX-RS, Jackson), JUnit 5, AssertJ

**Spec:** `specs/2026-06-19-github-webhook-receiver-design.md` (rev 4)

**Issue:** devtown#15

---

## File Structure

### New files

| File | Module | Purpose |
|------|--------|---------|
| `review/src/main/java/io/casehub/devtown/review/LifecycleResult.java` | review | Enum: UPDATED, NO_ACTIVE_CASE, ALREADY_COMPLETED, ALREADY_ABANDONED |
| `github/src/main/java/io/casehub/devtown/github/GitHubSignatureVerifier.java` | github | HMAC-SHA256 verification, constant-time comparison |
| `github/src/main/java/io/casehub/devtown/github/GitHubPullRequestEvent.java` | github | Minimal nested record DTO for GitHub webhook payload |
| `github/src/main/java/io/casehub/devtown/github/GitHubPayloadMapper.java` | github | DTO → PrPayload translation |
| `github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java` | github | JAX-RS endpoint at `/api/github/webhook` |
| `github/src/test/java/io/casehub/devtown/github/GitHubSignatureVerifierTest.java` | github | Signature verification tests |
| `github/src/test/java/io/casehub/devtown/github/GitHubPullRequestEventTest.java` | github | JSON deserialization tests |
| `github/src/test/java/io/casehub/devtown/github/GitHubPayloadMapperTest.java` | github | Mapping correctness tests |
| `github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java` | github | Endpoint routing + response tests |
| `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceLifecycleTest.java` | app | Signal ordering verification test |

### Modified files

| File | Module | Change |
|------|--------|--------|
| `github/pom.xml` | github | Add `jakarta.ws.rs-api` (provided), `junit-jupiter` + `assertj-core` + `mockito-core` (test) |
| `review/src/main/java/io/casehub/devtown/review/PrReviewApplicationService.java` | review | Rename `review()` → `startReview()`, add `revisePr()`, `closePr()` |
| `app/src/main/java/io/casehub/devtown/app/PrReviewService.java` | app | Rename + add default lifecycle stubs |
| `app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java` | app | Rename + add default lifecycle stubs |
| `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java` | app | Rename, mutable sub-maps, implement `revisePr()` + `closePr()` |
| `app/src/main/java/io/casehub/devtown/app/PrReviewResource.java` | app | Call `startReview()` instead of `review()` |
| `app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java` | app | Add secondary `prIndex`, `findActiveCaseByPr()`, cleanup in `onCaseLifecycle()` |
| `app/src/test/java/io/casehub/devtown/app/PrReviewServiceTest.java` | app | Rename method calls, add lifecycle result tests |
| `review/src/main/resources/devtown/pr-review.yaml` | review | Add `.pr.status == "merged" or` to success goal conditions |
| `app/src/main/resources/application.properties` | app | Add `devtown.github.webhook-secret` config |

---

### Task 1: Port interface — LifecycleResult enum and PrReviewApplicationService

**Files:**
- Create: `review/src/main/java/io/casehub/devtown/review/LifecycleResult.java`
- Modify: `review/src/main/java/io/casehub/devtown/review/PrReviewApplicationService.java`

- [ ] **Step 1: Create LifecycleResult enum**

```java
// review/src/main/java/io/casehub/devtown/review/LifecycleResult.java
package io.casehub.devtown.review;

public enum LifecycleResult {
    UPDATED,
    NO_ACTIVE_CASE,
    ALREADY_COMPLETED,
    ALREADY_ABANDONED
}
```

- [ ] **Step 2: Update PrReviewApplicationService interface**

Replace the entire file:

```java
// review/src/main/java/io/casehub/devtown/review/PrReviewApplicationService.java
package io.casehub.devtown.review;

public interface PrReviewApplicationService {
    PrReviewOutcome startReview(PrPayload pr);
    LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged);
    LifecycleResult closePr(String repo, int prNumber, boolean merged);
}
```

- [ ] **Step 3: Verify compilation fails (expected — implementations don't match)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl review -am`

Expected: SUCCESS for review module (it's an interface). Downstream modules will fail later — that's correct.

- [ ] **Step 4: Commit**

```
git add review/src/main/java/io/casehub/devtown/review/LifecycleResult.java review/src/main/java/io/casehub/devtown/review/PrReviewApplicationService.java
git commit -m "feat: add LifecycleResult enum and expand PrReviewApplicationService port

Rename review() → startReview(), add revisePr() and closePr() lifecycle
methods with LifecycleResult return type.

Refs casehubio/devtown#15"
```

---

### Task 2: Update all implementations and callers for renamed port

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewService.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewResource.java`
- Modify: `app/src/test/java/io/casehub/devtown/app/PrReviewServiceTest.java`

- [ ] **Step 1: Update PrReviewService @DefaultBean (Layer 1 baseline)**

```java
// app/src/main/java/io/casehub/devtown/app/PrReviewService.java
package io.casehub.devtown.app;

import io.casehub.devtown.review.LifecycleResult;
import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.PrReviewApplicationService;
import io.casehub.devtown.review.PrReviewOutcome;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
@DefaultBean
public class PrReviewService implements PrReviewApplicationService {

    @Override
    public PrReviewOutcome startReview(PrPayload pr) {
        var securityFindings = analyzeSecurityDirectly(pr);
        var architectureFindings = reviewArchitectureDirectly(pr);
        var allFindings = new ArrayList<String>(securityFindings);
        allFindings.addAll(architectureFindings);
        return new PrReviewOutcome("reviewed", allFindings);
    }

    @Override
    public LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged) {
        return LifecycleResult.NO_ACTIVE_CASE;
    }

    @Override
    public LifecycleResult closePr(String repo, int prNumber, boolean merged) {
        return LifecycleResult.NO_ACTIVE_CASE;
    }

    private List<String> analyzeSecurityDirectly(PrPayload pr) {
        return List.of("security-analysis-complete");
    }

    private List<String> reviewArchitectureDirectly(PrPayload pr) {
        return List.of("architecture-review-complete");
    }
}
```

- [ ] **Step 2: Update QhorusPrReviewService @Priority(1) (Layer 3)**

Add the two new methods at the end of the class, before the private helper methods:

```java
@Override
public LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged) {
    return LifecycleResult.NO_ACTIVE_CASE;
}

@Override
public LifecycleResult closePr(String repo, int prNumber, boolean merged) {
    return LifecycleResult.NO_ACTIVE_CASE;
}
```

And rename the existing `review(PrPayload)` method to `startReview(PrPayload)`. Add the `LifecycleResult` import.

- [ ] **Step 3: Update PrReviewCaseService @Priority(2) (Layer 5)**

Rename `review(PrPayload)` to `startReview(PrPayload)`. Add stub lifecycle methods (real implementation in Task 7):

```java
@Override
public LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged) {
    return LifecycleResult.NO_ACTIVE_CASE;
}

@Override
public LifecycleResult closePr(String repo, int prNumber, boolean merged) {
    return LifecycleResult.NO_ACTIVE_CASE;
}
```

Add the `LifecycleResult` import.

- [ ] **Step 4: Update PrReviewResource**

Change `service.review(pr)` to `service.startReview(pr)`:

```java
@POST
public PrReviewOutcome review(PrPayload pr) {
    return service.startReview(pr);
}
```

- [ ] **Step 5: Update PrReviewServiceTest**

Change all `service.review(pr)` calls to `service.startReview(pr)`. Add lifecycle method tests:

```java
@Test
void revisePr_returnsNoActiveCase() {
    var result = service.revisePr("casehubio/devtown", 42, "newsha", 200);
    assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
}

@Test
void closePr_returnsNoActiveCase() {
    var result = service.closePr("casehubio/devtown", 42, false);
    assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
}
```

Add `LifecycleResult` import.

- [ ] **Step 6: Build and run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile test -pl app -am -Dtest="PrReviewServiceTest"`

Expected: BUILD SUCCESS. All tests pass.

- [ ] **Step 7: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/PrReviewService.java app/src/main/java/io/casehub/devtown/app/QhorusPrReviewService.java app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/main/java/io/casehub/devtown/app/PrReviewResource.java app/src/test/java/io/casehub/devtown/app/PrReviewServiceTest.java
git commit -m "refactor: rename review() → startReview(), add lifecycle stubs to all implementations

All three PrReviewApplicationService implementations updated:
- PrReviewService @DefaultBean: stubs return NO_ACTIVE_CASE
- QhorusPrReviewService @Priority(1): stubs return NO_ACTIVE_CASE
- PrReviewCaseService @Priority(2): stubs (real impl in next commit)

Refs casehubio/devtown#15"
```

---

### Task 3: Mutable sub-map fix in PrReviewCaseService

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

- [ ] **Step 1: Change Map.of() to LinkedHashMap in initial context construction**

In `PrReviewCaseService.startReview()` (formerly `review()`), find:

```java
var prContext = Map.<String, Object>of(
    "id", String.valueOf(pr.prNumber()),
    "repo", pr.repo(),
    "linesChanged", pr.linesChanged(),
    "baseRef", pr.baseRef(),
    "headSha", pr.headSha(),
    "contributor", pr.contributor(),
    "changedPaths", pr.changedPaths()
);
```

Replace with:

```java
var prContext = new LinkedHashMap<String, Object>(Map.of(
    "id", String.valueOf(pr.prNumber()),
    "repo", pr.repo(),
    "linesChanged", pr.linesChanged(),
    "baseRef", pr.baseRef(),
    "headSha", pr.headSha(),
    "contributor", pr.contributor(),
    "changedPaths", pr.changedPaths()
));
```

Add `import java.util.LinkedHashMap;` (it may already be imported for `initialContext`).

- [ ] **Step 2: Build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -am`

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java
git commit -m "fix: use mutable LinkedHashMap for PR context sub-map

Map.of() returns an unmodifiable map. WritablePanelImpl shallow-copies
top-level data but not sub-maps. Sub-path signals like
signal(caseId, \"pr.headSha\", newSha) throw UnsupportedOperationException
when navigating into the immutable pr sub-map.

Refs casehubio/devtown#15"
```

---

### Task 4: Case tracker secondary index

**Files:**
- Modify: `app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java`

- [ ] **Step 1: Add secondary prIndex and findActiveCaseByPr()**

Add field:

```java
private final ConcurrentHashMap<String, UUID> prIndex = new ConcurrentHashMap<>();
```

Add method:

```java
public Optional<CaseInfo> findActiveCaseByPr(String repo, int prNumber) {
    UUID caseId = prIndex.get(repo + "#" + prNumber);
    if (caseId == null) return Optional.empty();
    CaseInfo info = cases.get(caseId);
    if (info == null || info.status().isTerminal()) return Optional.empty();
    return Optional.of(info);
}
```

Add `import java.util.Optional;`.

- [ ] **Step 2: Update register() to maintain prIndex**

In the `register` method, after `cases.put(caseId, ...)`, add:

```java
prIndex.put(payload.repo() + "#" + payload.prNumber(), caseId);
```

- [ ] **Step 3: Update onCaseLifecycle() to clean up prIndex on terminal**

In `onCaseLifecycle()`, after `updateStatus(event.caseId(), event.caseStatus(), Instant.now())`, add:

```java
if (CaseTrackingStatus.fromCaseStatus(event.caseStatus()).isTerminal()) {
    CaseInfo terminated = cases.get(event.caseId());
    if (terminated != null) {
        prIndex.remove(terminated.payload().repo() + "#" + terminated.payload().prNumber());
    }
}
```

- [ ] **Step 4: Build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl app -am`

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/mcp/PrReviewCaseTracker.java
git commit -m "feat: add secondary prIndex to PrReviewCaseTracker

ConcurrentHashMap<String, UUID> keyed by repo#prNumber for O(1) case
lookup by PR coordinates. Maintained on register(), cleaned up in
onCaseLifecycle() when case reaches terminal status.

Refs casehubio/devtown#15"
```

---

### Task 5: GitHubSignatureVerifier (TDD)

**Files:**
- Modify: `github/pom.xml`
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubSignatureVerifierTest.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubSignatureVerifier.java`

- [ ] **Step 1: Add test dependencies to github/pom.xml**

Add inside `<dependencies>`:

```xml
<dependency>
  <groupId>jakarta.ws.rs</groupId>
  <artifactId>jakarta.ws.rs-api</artifactId>
  <scope>provided</scope>
</dependency>
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-core</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-core</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Write failing tests**

```java
// github/src/test/java/io/casehub/devtown/github/GitHubSignatureVerifierTest.java
package io.casehub.devtown.github;

import org.junit.jupiter.api.Test;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;

import static org.assertj.core.api.Assertions.assertThat;

class GitHubSignatureVerifierTest {

    private static final String SECRET = "test-webhook-secret";
    private static final String BODY = "{\"action\":\"opened\",\"number\":42}";

    @Test
    void validSignature_returnsTrue() {
        String signature = "sha256=" + computeHmac(BODY, SECRET);
        assertThat(GitHubSignatureVerifier.verify(BODY, signature, SECRET)).isTrue();
    }

    @Test
    void invalidSignature_returnsFalse() {
        assertThat(GitHubSignatureVerifier.verify(BODY, "sha256=deadbeef", SECRET)).isFalse();
    }

    @Test
    void nullSignatureHeader_returnsFalse() {
        assertThat(GitHubSignatureVerifier.verify(BODY, null, SECRET)).isFalse();
    }

    @Test
    void emptySignatureHeader_returnsFalse() {
        assertThat(GitHubSignatureVerifier.verify(BODY, "", SECRET)).isFalse();
    }

    @Test
    void missingPrefix_returnsFalse() {
        String hmac = computeHmac(BODY, SECRET);
        assertThat(GitHubSignatureVerifier.verify(BODY, hmac, SECRET)).isFalse();
    }

    @Test
    void wrongPrefix_returnsFalse() {
        String hmac = computeHmac(BODY, SECRET);
        assertThat(GitHubSignatureVerifier.verify(BODY, "sha1=" + hmac, SECRET)).isFalse();
    }

    @Test
    void emptyBody_validSignature_returnsTrue() {
        String empty = "";
        String signature = "sha256=" + computeHmac(empty, SECRET);
        assertThat(GitHubSignatureVerifier.verify(empty, signature, SECRET)).isTrue();
    }

    private static String computeHmac(String body, String secret) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
            byte[] hash = mac.doFinal(body.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException | InvalidKeyException e) {
            throw new RuntimeException(e);
        }
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am -Dtest="GitHubSignatureVerifierTest"`

Expected: FAIL — `GitHubSignatureVerifier` class not found.

- [ ] **Step 4: Implement GitHubSignatureVerifier**

```java
// github/src/main/java/io/casehub/devtown/github/GitHubSignatureVerifier.java
package io.casehub.devtown.github;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;

public final class GitHubSignatureVerifier {

    private static final String PREFIX = "sha256=";

    private GitHubSignatureVerifier() {}

    public static boolean verify(String body, String signatureHeader, String secret) {
        if (signatureHeader == null || !signatureHeader.startsWith(PREFIX)) {
            return false;
        }
        String receivedHex = signatureHeader.substring(PREFIX.length());
        byte[] received;
        try {
            received = HexFormat.of().parseHex(receivedHex);
        } catch (IllegalArgumentException e) {
            return false;
        }

        byte[] expected = computeHmac(body, secret);
        return MessageDigest.isEqual(expected, received);
    }

    private static byte[] computeHmac(String body, String secret) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
            return mac.doFinal(body.getBytes(StandardCharsets.UTF_8));
        } catch (NoSuchAlgorithmException | InvalidKeyException e) {
            throw new IllegalStateException("HMAC-SHA256 unavailable", e);
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am -Dtest="GitHubSignatureVerifierTest"`

Expected: BUILD SUCCESS. All 7 tests pass.

- [ ] **Step 6: Commit**

```
git add github/pom.xml github/src/main/java/io/casehub/devtown/github/GitHubSignatureVerifier.java github/src/test/java/io/casehub/devtown/github/GitHubSignatureVerifierTest.java
git commit -m "feat: add GitHubSignatureVerifier with constant-time HMAC-SHA256

Pure Java, no CDI. Uses MessageDigest.isEqual() per PP-20260529-b7765c.
Seven test cases covering valid, invalid, null, empty, wrong prefix.

Refs casehubio/devtown#15"
```

---

### Task 6: GitHubPullRequestEvent DTO and GitHubPayloadMapper (TDD)

**Files:**
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubPullRequestEventTest.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubPullRequestEvent.java`
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubPayloadMapperTest.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubPayloadMapper.java`

- [ ] **Step 1: Write DTO parsing tests**

```java
// github/src/test/java/io/casehub/devtown/github/GitHubPullRequestEventTest.java
package io.casehub.devtown.github;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class GitHubPullRequestEventTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private static final String OPENED_EVENT = """
            {
              "action": "opened",
              "number": 42,
              "pull_request": {
                "head": { "sha": "abc123" },
                "base": { "ref": "main" },
                "user": { "login": "octocat" },
                "draft": false,
                "merged": false,
                "additions": 100,
                "deletions": 50,
                "changed_files": 5
              },
              "repository": {
                "full_name": "casehubio/devtown"
              },
              "sender": { "login": "octocat" },
              "installation": { "id": 12345 }
            }
            """;

    @Test
    void parsesAction() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.action()).isEqualTo("opened");
    }

    @Test
    void parsesNumber() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.number()).isEqualTo(42);
    }

    @Test
    void parsesPullRequestHead() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.pull_request().head().sha()).isEqualTo("abc123");
    }

    @Test
    void parsesPullRequestBase() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.pull_request().base().ref()).isEqualTo("main");
    }

    @Test
    void parsesPullRequestUser() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.pull_request().user().login()).isEqualTo("octocat");
    }

    @Test
    void parsesDraftFlag() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.pull_request().draft()).isFalse();
    }

    @Test
    void parsesLinesChanged() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.pull_request().additions()).isEqualTo(100);
        assertThat(event.pull_request().deletions()).isEqualTo(50);
    }

    @Test
    void parsesRepository() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event.repository().full_name()).isEqualTo("casehubio/devtown");
    }

    @Test
    void ignoresUnknownFields() throws Exception {
        var event = MAPPER.readValue(OPENED_EVENT, GitHubPullRequestEvent.class);
        assertThat(event).isNotNull();
    }

    @Test
    void parsesMergedFlag() throws Exception {
        var merged = OPENED_EVENT.replace("\"merged\": false", "\"merged\": true");
        var event = MAPPER.readValue(merged, GitHubPullRequestEvent.class);
        assertThat(event.pull_request().merged()).isTrue();
    }

    @Test
    void parsesDraftTrue() throws Exception {
        var draft = OPENED_EVENT.replace("\"draft\": false", "\"draft\": true");
        var event = MAPPER.readValue(draft, GitHubPullRequestEvent.class);
        assertThat(event.pull_request().draft()).isTrue();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am -Dtest="GitHubPullRequestEventTest"`

Expected: FAIL — `GitHubPullRequestEvent` not found.

- [ ] **Step 3: Implement GitHubPullRequestEvent**

```java
// github/src/main/java/io/casehub/devtown/github/GitHubPullRequestEvent.java
package io.casehub.devtown.github;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record GitHubPullRequestEvent(
    String action,
    int number,
    PullRequest pull_request,
    Repository repository
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record PullRequest(
        Head head, Base base, User user,
        boolean draft, boolean merged,
        int additions, int deletions, int changed_files
    ) {
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Head(String sha) {}
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record Base(String ref) {}
        @JsonIgnoreProperties(ignoreUnknown = true)
        public record User(String login) {}
    }
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Repository(String full_name) {}
}
```

- [ ] **Step 4: Run DTO tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am -Dtest="GitHubPullRequestEventTest"`

Expected: BUILD SUCCESS. All 11 tests pass.

- [ ] **Step 5: Write mapper tests**

```java
// github/src/test/java/io/casehub/devtown/github/GitHubPayloadMapperTest.java
package io.casehub.devtown.github;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class GitHubPayloadMapperTest {

    private final GitHubPullRequestEvent event = new GitHubPullRequestEvent(
        "opened", 42,
        new GitHubPullRequestEvent.PullRequest(
            new GitHubPullRequestEvent.PullRequest.Head("abc123"),
            new GitHubPullRequestEvent.PullRequest.Base("main"),
            new GitHubPullRequestEvent.PullRequest.User("octocat"),
            false, false, 100, 50, 5
        ),
        new GitHubPullRequestEvent.Repository("casehubio/devtown")
    );

    @Test
    void mapsRepo() {
        var payload = GitHubPayloadMapper.toPrPayload(event);
        assertThat(payload.repo()).isEqualTo("casehubio/devtown");
    }

    @Test
    void mapsPrNumber() {
        var payload = GitHubPayloadMapper.toPrPayload(event);
        assertThat(payload.prNumber()).isEqualTo(42);
    }

    @Test
    void mapsHeadSha() {
        var payload = GitHubPayloadMapper.toPrPayload(event);
        assertThat(payload.headSha()).isEqualTo("abc123");
    }

    @Test
    void mapsBaseRef() {
        var payload = GitHubPayloadMapper.toPrPayload(event);
        assertThat(payload.baseRef()).isEqualTo("main");
    }

    @Test
    void mapsLinesChangedAsSum() {
        var payload = GitHubPayloadMapper.toPrPayload(event);
        assertThat(payload.linesChanged()).isEqualTo(150);
    }

    @Test
    void mapsContributor() {
        var payload = GitHubPayloadMapper.toPrPayload(event);
        assertThat(payload.contributor()).isEqualTo("octocat");
    }

    @Test
    void changedPathsIsEmptyList() {
        var payload = GitHubPayloadMapper.toPrPayload(event);
        assertThat(payload.changedPaths()).isEmpty();
    }
}
```

- [ ] **Step 6: Run mapper tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am -Dtest="GitHubPayloadMapperTest"`

Expected: FAIL — `GitHubPayloadMapper` not found.

- [ ] **Step 7: Implement GitHubPayloadMapper**

```java
// github/src/main/java/io/casehub/devtown/github/GitHubPayloadMapper.java
package io.casehub.devtown.github;

import io.casehub.devtown.review.PrPayload;

import java.util.List;

public final class GitHubPayloadMapper {

    private GitHubPayloadMapper() {}

    public static PrPayload toPrPayload(GitHubPullRequestEvent event) {
        var pr = event.pull_request();
        return new PrPayload(
            event.repository().full_name(),
            event.number(),
            pr.head().sha(),
            pr.base().ref(),
            pr.additions() + pr.deletions(),
            pr.user().login(),
            List.of()
        );
    }
}
```

- [ ] **Step 8: Run all github module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am`

Expected: BUILD SUCCESS. All 25 tests pass (7 verifier + 11 DTO + 7 mapper).

- [ ] **Step 9: Commit**

```
git add github/src/main/java/io/casehub/devtown/github/GitHubPullRequestEvent.java github/src/main/java/io/casehub/devtown/github/GitHubPayloadMapper.java github/src/test/java/io/casehub/devtown/github/GitHubPullRequestEventTest.java github/src/test/java/io/casehub/devtown/github/GitHubPayloadMapperTest.java
git commit -m "feat: add GitHubPullRequestEvent DTO and GitHubPayloadMapper

Minimal nested records with @JsonIgnoreProperties(ignoreUnknown = true).
Snake_case field names matching GitHub JSON. Mapper translates to PrPayload
with linesChanged = additions + deletions, changedPaths = empty list.

Refs casehubio/devtown#15"
```

---

### Task 7: PrReviewCaseService lifecycle methods (TDD with signal ordering)

**Files:**
- Create: `app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceLifecycleTest.java`
- Modify: `app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java`

**Key fact:** `PrReviewCaseHub extends YamlCaseHub extends CaseHub`. `CaseHub` has `@Inject CaseHubRuntime runtime` and exposes `signal(UUID, String, Object)` which delegates to `runtime.signal()`. The existing `PrReviewCaseService` already injects `PrReviewCaseHub caseHub` — use `caseHub.signal()` for all signal calls. `PrReviewCaseTracker caseTracker` is already injected.

- [ ] **Step 1: Write tests using Mockito for signal ordering verification**

```java
// app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceLifecycleTest.java
package io.casehub.devtown.app;

import io.casehub.devtown.app.mcp.PrReviewCaseTracker;
import io.casehub.devtown.review.LifecycleResult;
import io.casehub.devtown.review.PrPayload;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.InOrder;

import java.util.List;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class PrReviewCaseServiceLifecycleTest {

    private PrReviewCaseTracker tracker;
    private PrReviewCaseHub caseHub;
    private PrReviewCaseService service;
    private UUID caseId;

    @BeforeEach
    void setUp() {
        tracker = new PrReviewCaseTracker();
        caseId = UUID.randomUUID();

        var payload = new PrPayload("casehubio/devtown", 42, "oldsha", "main", 100, "octocat", List.of());
        tracker.register(caseId, "default", payload);

        caseHub = mock(PrReviewCaseHub.class);
        when(caseHub.signal(any(), any(), any())).thenReturn(CompletableFuture.completedFuture(null));

        service = new PrReviewCaseService();
        service.caseHub = caseHub;
        service.caseTracker = tracker;
    }

    @Test
    void revisePr_signalsMetadataBeforeInvalidation() {
        service.revisePr("casehubio/devtown", 42, "newsha", 200);

        InOrder inOrder = inOrder(caseHub);
        inOrder.verify(caseHub).signal(eq(caseId), eq("pr.headSha"), eq("newsha"));
        inOrder.verify(caseHub).signal(eq(caseId), eq("pr.linesChanged"), eq(200));
        inOrder.verify(caseHub).signal(eq(caseId), eq("codeAnalysis"), isNull());
    }

    @Test
    void revisePr_nullsAllAnalysisFields() {
        service.revisePr("casehubio/devtown", 42, "newsha", 200);

        verify(caseHub).signal(caseId, "codeAnalysis", null);
        verify(caseHub).signal(caseId, "securityReview", null);
        verify(caseHub).signal(caseId, "architectureReview", null);
        verify(caseHub).signal(caseId, "styleCheck", null);
        verify(caseHub).signal(caseId, "testCoverage", null);
        verify(caseHub).signal(caseId, "performanceAnalysis", null);
    }

    @Test
    void revisePr_doesNotNullHumanApproval() {
        service.revisePr("casehubio/devtown", 42, "newsha", 200);

        verify(caseHub, never()).signal(eq(caseId), eq("humanApproval"), any());
    }

    @Test
    void revisePr_returnsUpdated_whenCaseExists() {
        var result = service.revisePr("casehubio/devtown", 42, "newsha", 200);
        assertThat(result).isEqualTo(LifecycleResult.UPDATED);
    }

    @Test
    void revisePr_returnsNoActiveCase_whenNoCaseExists() {
        var result = service.revisePr("casehubio/other", 99, "sha", 10);
        assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
    }

    @Test
    void closePr_notMerged_signalsClosedStatus() {
        service.closePr("casehubio/devtown", 42, false);

        verify(caseHub).signal(caseId, "pr.status", "closed");
    }

    @Test
    void closePr_merged_signalsMergedStatus() {
        service.closePr("casehubio/devtown", 42, true);

        verify(caseHub).signal(caseId, "pr.status", "merged");
    }

    @Test
    void closePr_returnsNoActiveCase_whenNoCaseExists() {
        var result = service.closePr("casehubio/other", 99, false);
        assertThat(result).isEqualTo(LifecycleResult.NO_ACTIVE_CASE);
    }

    @Test
    void startReview_existingCase_delegatesToRevisePr() {
        service.startReview(new PrPayload("casehubio/devtown", 42, "newsha", "main", 200, "octocat", List.of()));

        verify(caseHub).signal(eq(caseId), eq("pr.headSha"), eq("newsha"));
        verify(caseHub, never()).startCase(any());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="PrReviewCaseServiceLifecycleTest"`

Expected: FAIL — lifecycle methods still return `NO_ACTIVE_CASE` (stubs from Task 2).

- [ ] **Step 3: Implement revisePr(), closePr(), and idempotent startReview()**

In `PrReviewCaseService`, replace the `revisePr` stub:

```java
@Override
public LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged) {
    var active = caseTracker.findActiveCaseByPr(repo, prNumber);
    if (active.isEmpty()) return LifecycleResult.NO_ACTIVE_CASE;

    UUID caseId = active.get().caseId();

    // Signal ordering contract: metadata BEFORE invalidation.
    // initial-analysis binding fires on .codeAnalysis == null and reads pr.headSha as input.
    caseHub.signal(caseId, "pr.headSha", newHeadSha);
    caseHub.signal(caseId, "pr.linesChanged", linesChanged);

    // Invalidate stale analysis — triggers binding re-evaluation.
    // Human approvals are preserved (explicit policy — see spec §14).
    caseHub.signal(caseId, "codeAnalysis", null);
    caseHub.signal(caseId, "securityReview", null);
    caseHub.signal(caseId, "architectureReview", null);
    caseHub.signal(caseId, "styleCheck", null);
    caseHub.signal(caseId, "testCoverage", null);
    caseHub.signal(caseId, "performanceAnalysis", null);

    return LifecycleResult.UPDATED;
}
```

Replace the `closePr` stub:

```java
@Override
public LifecycleResult closePr(String repo, int prNumber, boolean merged) {
    var active = caseTracker.findActiveCaseByPr(repo, prNumber);
    if (active.isEmpty()) return LifecycleResult.NO_ACTIVE_CASE;

    UUID caseId = active.get().caseId();
    caseHub.signal(caseId, "pr.status", merged ? "merged" : "closed");

    return LifecycleResult.UPDATED;
}
```

Add idempotency check to `startReview()` — at the start of the method, before case creation:

```java
var existing = caseTracker.findActiveCaseByPr(pr.repo(), pr.prNumber());
if (existing.isPresent()) {
    revisePr(pr.repo(), pr.prNumber(), pr.headSha(), pr.linesChanged());
    return new PrReviewOutcome(VERDICT_CASE_OPENED, List.of());
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest="PrReviewCaseServiceLifecycleTest"`

Expected: BUILD SUCCESS. All 9 tests pass.

- [ ] **Step 5: Commit**

```
git add app/src/main/java/io/casehub/devtown/app/PrReviewCaseService.java app/src/test/java/io/casehub/devtown/app/PrReviewCaseServiceLifecycleTest.java
git commit -m "feat: implement revisePr(), closePr(), and idempotent startReview()

revisePr() updates PR metadata (headSha, linesChanged) BEFORE nulling
stale analysis fields — signal ordering verified by InOrder mock.
Human approvals preserved across pushes (explicit policy).
closePr() signals pr.status as 'merged' or 'closed'.
startReview() checks for existing active case and delegates to
revisePr() if found — idempotent for GitHub webhook retries.

Refs casehubio/devtown#15"
```

---

### Task 8: GitHubWebhookResource (TDD)

**Files:**
- Create: `github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java`

- [ ] **Step 1: Write resource tests**

```java
// github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java
package io.casehub.devtown.github;

import io.casehub.devtown.review.LifecycleResult;
import io.casehub.devtown.review.PrPayload;
import io.casehub.devtown.review.PrReviewApplicationService;
import io.casehub.devtown.review.PrReviewOutcome;
import jakarta.ws.rs.core.Response;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class GitHubWebhookResourceTest {

    private static final String SECRET = "test-secret";
    private RecordingService service;
    private GitHubWebhookResource resource;

    static class RecordingService implements PrReviewApplicationService {
        PrPayload lastStartReview;
        String lastReviseRepo;
        int lastRevisePr;
        String lastCloseRepo;
        boolean lastCloseMerged;
        LifecycleResult revisePrResult = LifecycleResult.UPDATED;
        LifecycleResult closePrResult = LifecycleResult.UPDATED;

        @Override
        public PrReviewOutcome startReview(PrPayload pr) {
            lastStartReview = pr;
            return new PrReviewOutcome("case-opened", List.of());
        }

        @Override
        public LifecycleResult revisePr(String repo, int prNumber, String newHeadSha, int linesChanged) {
            lastReviseRepo = repo;
            lastRevisePr = prNumber;
            return revisePrResult;
        }

        @Override
        public LifecycleResult closePr(String repo, int prNumber, boolean merged) {
            lastCloseRepo = repo;
            lastCloseMerged = merged;
            return closePrResult;
        }
    }

    @BeforeEach
    void setUp() {
        service = new RecordingService();
        resource = new GitHubWebhookResource();
        resource.service = service;
        resource.webhookSecret = SECRET;
    }

    private String sign(String body) {
        return "sha256=" + computeHmac(body, SECRET);
    }

    @Test
    void invalidSignature_returns401() {
        var response = resource.receive("{}", "pull_request", "sha256=bad", "delivery-1");
        assertThat(response.getStatus()).isEqualTo(401);
    }

    @Test
    void unknownEventType_returns200Ignored() {
        String body = "{}";
        var response = resource.receive(body, "push", sign(body), "delivery-1");
        assertThat(response.getStatus()).isEqualTo(200);
        assertThat(responseBody(response)).containsEntry("status", "ignored");
    }

    @Test
    void openedNonDraft_callsStartReview() {
        String body = openedEvent(false);
        resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(service.lastStartReview).isNotNull();
        assertThat(service.lastStartReview.prNumber()).isEqualTo(42);
    }

    @Test
    void openedDraft_doesNotCallStartReview() {
        String body = openedEvent(true);
        var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(service.lastStartReview).isNull();
        assertThat(responseBody(response)).containsEntry("reason", "draft");
    }

    @Test
    void synchronize_callsRevisePr() {
        String body = synchronizeEvent();
        resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(service.lastReviseRepo).isEqualTo("casehubio/devtown");
        assertThat(service.lastRevisePr).isEqualTo(42);
    }

    @Test
    void closedMerged_callsClosePrWithTrue() {
        String body = closedEvent(true);
        resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(service.lastCloseRepo).isEqualTo("casehubio/devtown");
        assertThat(service.lastCloseMerged).isTrue();
    }

    @Test
    void closedNotMerged_callsClosePrWithFalse() {
        String body = closedEvent(false);
        resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(service.lastCloseMerged).isFalse();
    }

    @Test
    void readyForReview_callsStartReview() {
        String body = readyForReviewEvent();
        resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(service.lastStartReview).isNotNull();
    }

    @Test
    void reopened_callsStartReview() {
        String body = reopenedEvent();
        resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(service.lastStartReview).isNotNull();
    }

    @Test
    void unknownAction_returns200Ignored() {
        String body = eventWithAction("labeled");
        var response = resource.receive(body, "pull_request", sign(body), "delivery-1");
        assertThat(response.getStatus()).isEqualTo(200);
        assertThat(responseBody(response)).containsEntry("status", "ignored");
    }

    @SuppressWarnings("unchecked")
    private Map<String, Object> responseBody(Response response) {
        return (Map<String, Object>) response.getEntity();
    }

    private String eventWithAction(String action) {
        return """
            {"action":"%s","number":42,"pull_request":{"head":{"sha":"abc"},"base":{"ref":"main"},"user":{"login":"octocat"},"draft":false,"merged":false,"additions":10,"deletions":5,"changed_files":1},"repository":{"full_name":"casehubio/devtown"}}
            """.formatted(action);
    }

    private String openedEvent(boolean draft) {
        return """
            {"action":"opened","number":42,"pull_request":{"head":{"sha":"abc"},"base":{"ref":"main"},"user":{"login":"octocat"},"draft":%s,"merged":false,"additions":10,"deletions":5,"changed_files":1},"repository":{"full_name":"casehubio/devtown"}}
            """.formatted(draft);
    }

    private String synchronizeEvent() {
        return eventWithAction("synchronize");
    }

    private String closedEvent(boolean merged) {
        return """
            {"action":"closed","number":42,"pull_request":{"head":{"sha":"abc"},"base":{"ref":"main"},"user":{"login":"octocat"},"draft":false,"merged":%s,"additions":10,"deletions":5,"changed_files":1},"repository":{"full_name":"casehubio/devtown"}}
            """.formatted(merged);
    }

    private String readyForReviewEvent() {
        return eventWithAction("ready_for_review");
    }

    private String reopenedEvent() {
        return eventWithAction("reopened");
    }

    private static String computeHmac(String body, String secret) {
        try {
            var mac = javax.crypto.Mac.getInstance("HmacSHA256");
            mac.init(new javax.crypto.spec.SecretKeySpec(secret.getBytes(java.nio.charset.StandardCharsets.UTF_8), "HmacSHA256"));
            return java.util.HexFormat.of().formatHex(mac.doFinal(body.getBytes(java.nio.charset.StandardCharsets.UTF_8)));
        } catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am -Dtest="GitHubWebhookResourceTest"`

Expected: FAIL — `GitHubWebhookResource` not found.

- [ ] **Step 3: Implement GitHubWebhookResource**

```java
// github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java
package io.casehub.devtown.github;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.devtown.review.LifecycleResult;
import io.casehub.devtown.review.PrReviewApplicationService;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.HeaderParam;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.Map;

@Path("/api/github/webhook")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class GitHubWebhookResource {

    private static final Logger LOG = Logger.getLogger(GitHubWebhookResource.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Inject
    PrReviewApplicationService service;

    @ConfigProperty(name = "devtown.github.webhook-secret", defaultValue = "")
    String webhookSecret;

    @POST
    public Response receive(String body,
                            @HeaderParam("X-GitHub-Event") String eventType,
                            @HeaderParam("X-Hub-Signature-256") String signature,
                            @HeaderParam("X-GitHub-Delivery") String deliveryId) {

        if (!GitHubSignatureVerifier.verify(body, signature, webhookSecret)) {
            return Response.status(401).entity(Map.of("error", "invalid signature")).build();
        }

        try {
            return switch (eventType) {
                case "pull_request" -> handlePullRequest(body);
                default -> ok(Map.of("status", "ignored", "event", eventType));
            };
        } catch (Exception e) {
            LOG.errorf(e, "Webhook processing failed. deliveryId=%s body=%s", deliveryId, body);
            return Response.serverError().entity(Map.of("error", "internal")).build();
        }
    }

    private Response handlePullRequest(String body) throws Exception {
        var event = MAPPER.readValue(body, GitHubPullRequestEvent.class);

        return switch (event.action()) {
            case "opened" -> handleOpened(event);
            case "ready_for_review" -> handleReadyForReview(event);
            case "synchronize" -> handleSynchronize(event);
            case "closed" -> handleClosed(event);
            case "reopened" -> handleReopened(event);
            default -> ok(Map.of("status", "ignored", "action", event.action()));
        };
    }

    private Response handleOpened(GitHubPullRequestEvent event) {
        if (event.pull_request().draft()) {
            return ok(Map.of("status", "ignored", "reason", "draft"));
        }
        service.startReview(GitHubPayloadMapper.toPrPayload(event));
        return ok(Map.of("status", "accepted", "action", "case-started"));
    }

    private Response handleReadyForReview(GitHubPullRequestEvent event) {
        service.startReview(GitHubPayloadMapper.toPrPayload(event));
        return ok(Map.of("status", "accepted", "action", "case-started"));
    }

    private Response handleSynchronize(GitHubPullRequestEvent event) {
        var pr = event.pull_request();
        var result = service.revisePr(
            event.repository().full_name(), event.number(),
            pr.head().sha(), pr.additions() + pr.deletions()
        );
        return ok(Map.of("status", "accepted", "action", lifecycleAction("case-updated", result)));
    }

    private Response handleClosed(GitHubPullRequestEvent event) {
        boolean merged = event.pull_request().merged();
        var result = service.closePr(event.repository().full_name(), event.number(), merged);
        String action = merged ? "case-merged" : "case-abandoned";
        return ok(Map.of("status", "accepted", "action", lifecycleAction(action, result)));
    }

    private Response handleReopened(GitHubPullRequestEvent event) {
        service.startReview(GitHubPayloadMapper.toPrPayload(event));
        return ok(Map.of("status", "accepted", "action", "case-started"));
    }

    private static String lifecycleAction(String normalAction, LifecycleResult result) {
        return switch (result) {
            case UPDATED -> normalAction;
            case NO_ACTIVE_CASE -> "no-active-case";
            case ALREADY_COMPLETED -> "already-completed";
            case ALREADY_ABANDONED -> "already-abandoned";
        };
    }

    private static Response ok(Map<String, Object> body) {
        return Response.ok(body).build();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -am -Dtest="GitHubWebhookResourceTest"`

Expected: BUILD SUCCESS. All 10 tests pass.

- [ ] **Step 5: Commit**

```
git add github/src/main/java/io/casehub/devtown/github/GitHubWebhookResource.java github/src/test/java/io/casehub/devtown/github/GitHubWebhookResourceTest.java
git commit -m "feat: add GitHubWebhookResource — webhook endpoint with two-level dispatch

POST /api/github/webhook with X-GitHub-Event + X-Hub-Signature-256 headers.
Routes by event type then action. Handles opened (draft-aware),
ready_for_review, synchronize, closed (merged/abandoned), reopened.
401 for bad signature, 200 for no-ops, 500 for exceptions.

Refs casehubio/devtown#15"
```

---

### Task 9: CasePlanModel goal updates (pr-review.yaml)

**Files:**
- Modify: `review/src/main/resources/devtown/pr-review.yaml`

- [ ] **Step 1: Add `.pr.status == "merged" or` to pr-approved goal**

Change the `pr-approved` goal condition from:

```yaml
condition: >-
    (.codeAnalysis.securitySensitive == false or .securityReview.outcome == "APPROVED") and
    (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
    .styleCheck.outcome == "APPROVED" and
    .testCoverage.outcome == "APPROVED" and
    .performanceAnalysis.outcome == "APPROVED"
```

To:

```yaml
condition: >-
    .pr.status == "merged" or
    ((.codeAnalysis.securitySensitive == false or .securityReview.outcome == "APPROVED") and
     (.codeAnalysis.architectureCrossing == false or .architectureReview.outcome == "APPROVED") and
     .styleCheck.outcome == "APPROVED" and
     .testCoverage.outcome == "APPROVED" and
     .performanceAnalysis.outcome == "APPROVED")
```

- [ ] **Step 2: Add `.pr.status == "merged" or` to security-verified goal**

Change from:

```yaml
condition: >-
    .codeAnalysis.securitySensitive == false or
    .securityReview.outcome == "APPROVED"
```

To:

```yaml
condition: >-
    .pr.status == "merged" or
    .codeAnalysis.securitySensitive == false or
    .securityReview.outcome == "APPROVED"
```

- [ ] **Step 3: Add `.pr.status == "merged" or` to ci-passing goal**

Change from:

```yaml
condition: '.ci.status == "passing"'
```

To:

```yaml
condition: '.pr.status == "merged" or .ci.status == "passing"'
```

- [ ] **Step 4: Run existing goal condition tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl review -am -Dtest="PrReviewGoalConditionTest"`

Expected: BUILD SUCCESS. Existing tests still pass (`.pr.status` is absent in existing test contexts → null → `null == "merged"` is false → original conditions evaluate unchanged).

- [ ] **Step 5: Commit**

```
git add review/src/main/resources/devtown/pr-review.yaml
git commit -m "feat: add external merge short-circuit to success goal conditions

Prepends .pr.status == \"merged\" or to pr-approved, security-verified,
and ci-passing goals. When a PR is merged externally (webhook closed
with merged=true), all three success goals short-circuit to satisfied.
Completion spec unchanged — allOf still works.

Refs casehubio/devtown#15"
```

---

### Task 10: Configuration and build verification

**Files:**
- Modify: `app/src/main/resources/application.properties`

- [ ] **Step 1: Add webhook secret config**

Add to `application.properties`:

```properties
# ── GitHub webhook integration ──────────────────────────────────────────────────
devtown.github.webhook-secret=${GITHUB_WEBHOOK_SECRET:}
```

- [ ] **Step 2: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`

Expected: BUILD SUCCESS. All tests pass across all modules.

- [ ] **Step 3: Commit**

```
git add app/src/main/resources/application.properties
git commit -m "chore: add GitHub webhook secret configuration

Refs casehubio/devtown#15"
```

---

## Post-Implementation

### Engine issues to file

1. **`WritablePanelImpl` shallow copy** — `WritablePanelImpl(String, Map<String, Object>)` does `this.data.putAll(initial)` without deep-copying sub-maps. Any harness passing `Map.of()` as a sub-map value gets `UnsupportedOperationException` on sub-path signals.
2. **Composed `GoalExpression`** — `GoalExpression` schema supports only flat `allOf`/`anyOf` lists. Nested `anyOf(allOf(...), goal)` would enable separate `externally-merged` goal without polluting individual goal conditions.

### Documentation sync

Run `implementation-doc-sync` per the development workflow to check for drift in:
- `ARC42STORIES.MD` — may need system context diagram update (GitHub → devtown relationship)
- `docs/repos/casehub-devtown.md` — layer status table if this counts as layer progress
