# GitHub App Authentication Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #218 — GitHub App setup and authentication
**Issue group:** #218

**Goal:** Wire GitHub App authentication into all 5 REST client interfaces with PAT fallback for dev mode.

**Architecture:** A `@Produces` factory checks config to create either `AppTokenProvider` (JWT → installation token with volatile cache) or `PatTokenProvider` (static PAT). A shared `GitHubAuthHeadersFactory` (`ClientHeadersFactory`) injects the token as a `Bearer` header on all 5 REST client interfaces via `@RegisterClientHeaders`.

**Tech Stack:** Java 21, Quarkus 3.39.3, nimbus-jose-jwt, MicroProfile REST Client

## Global Constraints

- Java 21 source level on Java 26 JVM
- All code in `github/` module: `io.casehub.devtown.github` package
- Tests use JUnit 5 + AssertJ (already in github/pom.xml)
- nimbus-jose-jwt version: `10.3` (not in Quarkus BOM — declare in devtown parent pom `<dependencyManagement>`)
- All commits reference `Refs #218`
- MicroProfile REST Client `configKey` for existing APIs: `github-api`; for installation token exchange: `github-app-api`

---

## Batch 1: Token Provider Interface + JWT Generation

Safe wrap point: pure Java classes with unit tests, no CDI wiring yet. Build compiles, all tests pass.

### Task 1: GitHubTokenProvider interface + PatTokenProvider

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubTokenProvider.java`
- Create: `github/src/main/java/io/casehub/devtown/github/PatTokenProvider.java`
- Test: `github/src/test/java/io/casehub/devtown/github/PatTokenProviderTest.java`

**Interfaces:**
- Consumes: nothing (leaf dependency)
- Produces: `GitHubTokenProvider` interface with `String getToken()`, used by Tasks 3, 4, 5

- [ ] **Step 1: Write the failing test for PatTokenProvider**

```java
package io.casehub.devtown.github;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class PatTokenProviderTest {

    @Test
    void returnsConfiguredToken() {
        var provider = new PatTokenProvider("ghp_test123");
        assertThat(provider.getToken()).isEqualTo("ghp_test123");
    }

    @Test
    void returnsEmptyWhenNotConfigured() {
        var provider = new PatTokenProvider("");
        assertThat(provider.getToken()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=PatTokenProviderTest -q`
Expected: FAIL — `PatTokenProvider` does not exist

- [ ] **Step 3: Create GitHubTokenProvider interface**

```java
package io.casehub.devtown.github;

public interface GitHubTokenProvider {
    String getToken();
}
```

- [ ] **Step 4: Create PatTokenProvider**

```java
package io.casehub.devtown.github;

public class PatTokenProvider implements GitHubTokenProvider {

    private final String token;

    public PatTokenProvider(String token) {
        this.token = token;
    }

    @Override
    public String getToken() {
        return token;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=PatTokenProviderTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add github/src/main/java/io/casehub/devtown/github/GitHubTokenProvider.java github/src/main/java/io/casehub/devtown/github/PatTokenProvider.java github/src/test/java/io/casehub/devtown/github/PatTokenProviderTest.java
git commit -m "feat(github): add GitHubTokenProvider interface and PatTokenProvider

Refs #218"
```

### Task 2: GitHubAppJwtGenerator + nimbus-jose-jwt dependency

**Files:**
- Modify: `pom.xml` (parent — add nimbus-jose-jwt to `<dependencyManagement>`)
- Modify: `github/pom.xml` (add nimbus-jose-jwt dependency)
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubAppJwtGenerator.java`
- Test: `github/src/test/java/io/casehub/devtown/github/GitHubAppJwtGeneratorTest.java`

**Interfaces:**
- Consumes: nothing (pure utility)
- Produces: `GitHubAppJwtGenerator.generateJwt(String appId, PrivateKey privateKey)` → `String` (JWT), used by Task 3. Also `GitHubAppJwtGenerator.loadPrivateKey(String pemFilePath)` → `PrivateKey`, used by Task 3.

- [ ] **Step 1: Add nimbus-jose-jwt dependency**

In `pom.xml` (parent), inside `<dependencyManagement><dependencies>`, add:

```xml
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>nimbus-jose-jwt</artifactId>
    <version>10.3</version>
</dependency>
```

In `github/pom.xml`, inside `<dependencies>`, add:

```xml
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>nimbus-jose-jwt</artifactId>
</dependency>
```

- [ ] **Step 2: Write the failing test for GitHubAppJwtGenerator**

```java
package io.casehub.devtown.github;

import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPrivateKey;
import java.security.interfaces.RSAPublicKey;
import java.time.Instant;
import java.util.Date;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GitHubAppJwtGeneratorTest {

    private static KeyPair keyPair;

    @BeforeAll
    static void generateKeyPair() throws Exception {
        var gen = KeyPairGenerator.getInstance("RSA");
        gen.initialize(2048);
        keyPair = gen.generateKeyPair();
    }

    @Test
    void generatesValidJwt() throws Exception {
        String jwt = GitHubAppJwtGenerator.generateJwt(
            "12345", (RSAPrivateKey) keyPair.getPrivate());

        SignedJWT parsed = SignedJWT.parse(jwt);
        assertThat(parsed.getHeader().getAlgorithm().getName()).isEqualTo("RS256");

        JWTClaimsSet claims = parsed.getJWTClaimsSet();
        assertThat(claims.getIssuer()).isEqualTo("12345");

        Instant now = Instant.now();
        assertThat(claims.getIssueTime().toInstant())
            .isBefore(now)
            .isAfter(now.minusSeconds(120));
        assertThat(claims.getExpirationTime().toInstant())
            .isAfter(now)
            .isBefore(now.plusSeconds(660));
    }

    @Test
    void jwtIsVerifiableWithPublicKey() throws Exception {
        String jwt = GitHubAppJwtGenerator.generateJwt(
            "12345", (RSAPrivateKey) keyPair.getPrivate());

        SignedJWT parsed = SignedJWT.parse(jwt);
        var verifier = new com.nimbusds.jose.crypto.RSASSAVerifier(
            (RSAPublicKey) keyPair.getPublic());
        assertThat(parsed.verify(verifier)).isTrue();
    }

    @Test
    void loadsPrivateKeyFromPemFile(@TempDir Path tempDir) throws Exception {
        Path pemFile = tempDir.resolve("test-key.pem");
        byte[] encoded = keyPair.getPrivate().getEncoded();
        String base64 = java.util.Base64.getMimeEncoder(64, "\n".getBytes())
            .encodeToString(encoded);
        Files.writeString(pemFile,
            "-----BEGIN PRIVATE KEY-----\n" + base64 + "\n-----END PRIVATE KEY-----\n");

        RSAPrivateKey loaded = GitHubAppJwtGenerator.loadPrivateKey(pemFile.toString());
        assertThat(loaded.getEncoded()).isEqualTo(keyPair.getPrivate().getEncoded());
    }

    @Test
    void throwsOnMissingKeyFile() {
        assertThatThrownBy(() -> GitHubAppJwtGenerator.loadPrivateKey("/nonexistent/key.pem"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Private key file not found");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=GitHubAppJwtGeneratorTest -q`
Expected: FAIL — `GitHubAppJwtGenerator` does not exist

- [ ] **Step 4: Implement GitHubAppJwtGenerator**

```java
package io.casehub.devtown.github;

import com.nimbusds.jose.JWSAlgorithm;
import com.nimbusds.jose.JWSHeader;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyFactory;
import java.security.interfaces.RSAPrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.time.Instant;
import java.util.Base64;
import java.util.Date;

public final class GitHubAppJwtGenerator {

    private GitHubAppJwtGenerator() {}

    public static String generateJwt(String appId, RSAPrivateKey privateKey) {
        try {
            Instant now = Instant.now();
            JWTClaimsSet claims = new JWTClaimsSet.Builder()
                .issuer(appId)
                .issueTime(Date.from(now.minusSeconds(60)))
                .expirationTime(Date.from(now.plusSeconds(600)))
                .build();

            SignedJWT jwt = new SignedJWT(
                new JWSHeader(JWSAlgorithm.RS256), claims);
            jwt.sign(new RSASSASigner(privateKey));
            return jwt.serialize();
        } catch (Exception e) {
            throw new IllegalStateException("Failed to generate GitHub App JWT", e);
        }
    }

    public static RSAPrivateKey loadPrivateKey(String pemFilePath) {
        Path path = Path.of(pemFilePath);
        if (!Files.exists(path)) {
            throw new IllegalArgumentException(
                "Private key file not found: " + pemFilePath);
        }
        try {
            String pem = Files.readString(path);
            String base64 = pem
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replace("-----BEGIN RSA PRIVATE KEY-----", "")
                .replace("-----END RSA PRIVATE KEY-----", "")
                .replaceAll("\\s", "");

            byte[] decoded = Base64.getDecoder().decode(base64);
            PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(decoded);
            KeyFactory kf = KeyFactory.getInstance("RSA");
            return (RSAPrivateKey) kf.generatePrivate(spec);
        } catch (IOException e) {
            throw new IllegalArgumentException(
                "Failed to read private key file: " + pemFilePath, e);
        } catch (Exception e) {
            throw new IllegalArgumentException(
                "Failed to parse private key from: " + pemFilePath, e);
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=GitHubAppJwtGeneratorTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add pom.xml github/pom.xml github/src/main/java/io/casehub/devtown/github/GitHubAppJwtGenerator.java github/src/test/java/io/casehub/devtown/github/GitHubAppJwtGeneratorTest.java
git commit -m "feat(github): add GitHubAppJwtGenerator with nimbus-jose-jwt

RS256 JWT signing for GitHub App authentication. Loads RSA private key
from PEM file, generates 10-minute JWT with clock skew tolerance.

Refs #218"
```

---

## Batch 2: App Token Provider + CDI Wiring

Safe wrap point: full auth pipeline wired — AppTokenProvider with caching, producer factory, headers factory, all 5 REST clients annotated. Build compiles, all tests pass.

### Task 3: InstallationTokenClient + InstallationToken + AppTokenProvider

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/InstallationToken.java`
- Create: `github/src/main/java/io/casehub/devtown/github/InstallationTokenClient.java`
- Create: `github/src/main/java/io/casehub/devtown/github/CachedToken.java`
- Create: `github/src/main/java/io/casehub/devtown/github/AppTokenProvider.java`
- Test: `github/src/test/java/io/casehub/devtown/github/AppTokenProviderTest.java`

**Interfaces:**
- Consumes: `GitHubTokenProvider` (interface from Task 1), `GitHubAppJwtGenerator.generateJwt()` and `GitHubAppJwtGenerator.loadPrivateKey()` (from Task 2)
- Produces: `AppTokenProvider` implementing `GitHubTokenProvider`, used by Task 4 (producer creates it). `InstallationTokenClient` REST client interface, injected by Task 4's producer. `CachedToken` record (internal to AppTokenProvider).

- [ ] **Step 1: Create InstallationToken record**

```java
package io.casehub.devtown.github;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.time.Instant;

public record InstallationToken(
    @JsonProperty("token") String token,
    @JsonProperty("expires_at") Instant expiresAt) {}
```

- [ ] **Step 2: Create InstallationTokenClient REST interface**

```java
package io.casehub.devtown.github;

import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.HeaderParam;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;

@RegisterRestClient(configKey = "github-app-api")
@Path("/app")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface InstallationTokenClient {

    @POST
    @Path("/installations/{installation_id}/access_tokens")
    InstallationToken createInstallationToken(
        @PathParam("installation_id") String installationId,
        @HeaderParam("Authorization") String jwtAuth);
}
```

- [ ] **Step 3: Create CachedToken record**

```java
package io.casehub.devtown.github;

import java.time.Duration;
import java.time.Instant;

record CachedToken(String token, Instant expiresAt) {
    boolean isExpiringSoon(Instant now) {
        return now.isAfter(expiresAt.minus(Duration.ofMinutes(5)));
    }
}
```

- [ ] **Step 4: Write failing tests for AppTokenProvider**

```java
package io.casehub.devtown.github;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPrivateKey;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

class AppTokenProviderTest {

    private static final String APP_ID = "12345";
    private static final String INSTALLATION_ID = "67890";

    @TempDir
    Path tempDir;
    private String keyPath;

    @BeforeEach
    void setUp() throws Exception {
        var gen = KeyPairGenerator.getInstance("RSA");
        gen.initialize(2048);
        var keyPair = gen.generateKeyPair();

        Path pemFile = tempDir.resolve("test-key.pem");
        byte[] encoded = keyPair.getPrivate().getEncoded();
        String base64 = java.util.Base64.getMimeEncoder(64, "\n".getBytes())
            .encodeToString(encoded);
        Files.writeString(pemFile,
            "-----BEGIN PRIVATE KEY-----\n" + base64 + "\n-----END PRIVATE KEY-----\n");
        keyPath = pemFile.toString();
    }

    @Test
    void fetchesTokenOnFirstCall() {
        var callCount = new AtomicInteger(0);
        InstallationTokenClient mockClient = (id, jwt) -> {
            callCount.incrementAndGet();
            return new InstallationToken("ghs_fresh",
                Instant.now().plusSeconds(3600));
        };

        var provider = new AppTokenProvider(APP_ID, keyPath, INSTALLATION_ID, mockClient);
        String token = provider.getToken();

        assertThat(token).isEqualTo("ghs_fresh");
        assertThat(callCount.get()).isEqualTo(1);
    }

    @Test
    void returnsCachedTokenOnSubsequentCalls() {
        var callCount = new AtomicInteger(0);
        InstallationTokenClient mockClient = (id, jwt) -> {
            callCount.incrementAndGet();
            return new InstallationToken("ghs_cached",
                Instant.now().plusSeconds(3600));
        };

        var provider = new AppTokenProvider(APP_ID, keyPath, INSTALLATION_ID, mockClient);
        provider.getToken();
        provider.getToken();
        provider.getToken();

        assertThat(callCount.get()).isEqualTo(1);
    }

    @Test
    void refreshesExpiringSoonToken() {
        var callCount = new AtomicInteger(0);
        InstallationTokenClient mockClient = (id, jwt) -> {
            int call = callCount.incrementAndGet();
            if (call == 1) {
                return new InstallationToken("ghs_expiring",
                    Instant.now().plusSeconds(60));
            }
            return new InstallationToken("ghs_refreshed",
                Instant.now().plusSeconds(3600));
        };

        var provider = new AppTokenProvider(APP_ID, keyPath, INSTALLATION_ID, mockClient);
        provider.getToken();
        String refreshed = provider.getToken();

        assertThat(refreshed).isEqualTo("ghs_refreshed");
        assertThat(callCount.get()).isEqualTo(2);
    }

    @Test
    void passesCorrectInstallationId() {
        var capturedId = new String[1];
        InstallationTokenClient mockClient = (id, jwt) -> {
            capturedId[0] = id;
            return new InstallationToken("ghs_test",
                Instant.now().plusSeconds(3600));
        };

        var provider = new AppTokenProvider(APP_ID, keyPath, INSTALLATION_ID, mockClient);
        provider.getToken();

        assertThat(capturedId[0]).isEqualTo(INSTALLATION_ID);
    }

    @Test
    void passesJwtAuthHeader() {
        var capturedAuth = new String[1];
        InstallationTokenClient mockClient = (id, jwt) -> {
            capturedAuth[0] = jwt;
            return new InstallationToken("ghs_test",
                Instant.now().plusSeconds(3600));
        };

        var provider = new AppTokenProvider(APP_ID, keyPath, INSTALLATION_ID, mockClient);
        provider.getToken();

        assertThat(capturedAuth[0]).startsWith("Bearer ");
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=AppTokenProviderTest -q`
Expected: FAIL — `AppTokenProvider` does not exist

- [ ] **Step 6: Implement AppTokenProvider**

```java
package io.casehub.devtown.github;

import org.jboss.logging.Logger;

import java.security.interfaces.RSAPrivateKey;
import java.time.Instant;

public class AppTokenProvider implements GitHubTokenProvider {

    private static final Logger LOG = Logger.getLogger(AppTokenProvider.class);

    private final String appId;
    private final String installationId;
    private final RSAPrivateKey privateKey;
    private final InstallationTokenClient tokenClient;
    private volatile CachedToken cached;

    public AppTokenProvider(String appId, String privateKeyPath,
                            String installationId,
                            InstallationTokenClient tokenClient) {
        this.appId = appId;
        this.installationId = installationId;
        this.privateKey = GitHubAppJwtGenerator.loadPrivateKey(privateKeyPath);
        this.tokenClient = tokenClient;
    }

    @Override
    public String getToken() {
        CachedToken current = cached;
        if (current != null && !current.isExpiringSoon(Instant.now())) {
            return current.token();
        }
        return refreshToken();
    }

    private synchronized String refreshToken() {
        CachedToken current = cached;
        if (current != null && !current.isExpiringSoon(Instant.now())) {
            return current.token();
        }

        LOG.infof("Refreshing GitHub installation token for installation %s",
            installationId);
        String jwt = GitHubAppJwtGenerator.generateJwt(appId, privateKey);
        InstallationToken response = tokenClient.createInstallationToken(
            installationId, "Bearer " + jwt);
        cached = new CachedToken(response.token(), response.expiresAt());
        return response.token();
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=AppTokenProviderTest -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add github/src/main/java/io/casehub/devtown/github/InstallationToken.java github/src/main/java/io/casehub/devtown/github/InstallationTokenClient.java github/src/main/java/io/casehub/devtown/github/CachedToken.java github/src/main/java/io/casehub/devtown/github/AppTokenProvider.java github/src/test/java/io/casehub/devtown/github/AppTokenProviderTest.java
git commit -m "feat(github): add AppTokenProvider with installation token caching

Generates JWT from App credentials, exchanges for installation token via
GitHub API, caches with volatile field and synchronized refresh.

Refs #218"
```

### Task 4: Producer + HeadersFactory + REST client wiring + config

**Files:**
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubTokenProviderProducer.java`
- Create: `github/src/main/java/io/casehub/devtown/github/GitHubAuthHeadersFactory.java`
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java` (add `@RegisterClientHeaders`)
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubPullRequestApi.java` (add `@RegisterClientHeaders`)
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubMergeApi.java` (add `@RegisterClientHeaders`)
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubRepoApi.java` (add `@RegisterClientHeaders`)
- Modify: `github/src/main/java/io/casehub/devtown/github/GitHubGitApi.java` (add `@RegisterClientHeaders`)
- Modify: `app/src/main/resources/application.properties` (add App config + github-app-api URL)
- Test: `github/src/test/java/io/casehub/devtown/github/GitHubTokenProviderProducerTest.java`
- Test: `github/src/test/java/io/casehub/devtown/github/GitHubAuthHeadersFactoryTest.java`

**Interfaces:**
- Consumes: `GitHubTokenProvider` (Task 1), `PatTokenProvider` (Task 1), `AppTokenProvider` (Task 3), `InstallationTokenClient` (Task 3)
- Produces: Fully wired auth pipeline — all REST clients send auth headers

- [ ] **Step 1: Write failing tests for GitHubTokenProviderProducer**

```java
package io.casehub.devtown.github;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyPairGenerator;
import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GitHubTokenProviderProducerTest {

    @Test
    void createsPatProviderWhenNoAppConfig() {
        var producer = new GitHubTokenProviderProducer();
        InstallationTokenClient dummyClient = (id, jwt) -> null;

        GitHubTokenProvider provider = producer.tokenProvider(
            Optional.empty(), Optional.empty(), Optional.empty(),
            Optional.of("ghp_mytoken"), dummyClient);

        assertThat(provider).isInstanceOf(PatTokenProvider.class);
        assertThat(provider.getToken()).isEqualTo("ghp_mytoken");
    }

    @Test
    void createsPatProviderWithEmptyTokenWhenNothingConfigured() {
        var producer = new GitHubTokenProviderProducer();
        InstallationTokenClient dummyClient = (id, jwt) -> null;

        GitHubTokenProvider provider = producer.tokenProvider(
            Optional.empty(), Optional.empty(), Optional.empty(),
            Optional.empty(), dummyClient);

        assertThat(provider).isInstanceOf(PatTokenProvider.class);
        assertThat(provider.getToken()).isEmpty();
    }

    @Test
    void createsAppProviderWhenAllAppConfigPresent(@TempDir Path tempDir) throws Exception {
        var gen = KeyPairGenerator.getInstance("RSA");
        gen.initialize(2048);
        var keyPair = gen.generateKeyPair();
        Path pemFile = tempDir.resolve("key.pem");
        byte[] encoded = keyPair.getPrivate().getEncoded();
        String base64 = java.util.Base64.getMimeEncoder(64, "\n".getBytes())
            .encodeToString(encoded);
        Files.writeString(pemFile,
            "-----BEGIN PRIVATE KEY-----\n" + base64 + "\n-----END PRIVATE KEY-----\n");

        var producer = new GitHubTokenProviderProducer();
        InstallationTokenClient mockClient = (id, jwt) ->
            new InstallationToken("ghs_test", Instant.now().plusSeconds(3600));

        GitHubTokenProvider provider = producer.tokenProvider(
            Optional.of("12345"), Optional.of(pemFile.toString()),
            Optional.of("67890"), Optional.of(""), mockClient);

        assertThat(provider).isInstanceOf(AppTokenProvider.class);
    }

    @Test
    void throwsOnPartialAppConfig() {
        var producer = new GitHubTokenProviderProducer();
        InstallationTokenClient dummyClient = (id, jwt) -> null;

        assertThatThrownBy(() -> producer.tokenProvider(
            Optional.of("12345"), Optional.empty(), Optional.empty(),
            Optional.empty(), dummyClient))
            .hasMessageContaining("Partial GitHub App config");
    }

    @Test
    void throwsOnPartialAppConfigMissingKeyPath() {
        var producer = new GitHubTokenProviderProducer();
        InstallationTokenClient dummyClient = (id, jwt) -> null;

        assertThatThrownBy(() -> producer.tokenProvider(
            Optional.of("12345"), Optional.empty(), Optional.of("67890"),
            Optional.empty(), dummyClient))
            .hasMessageContaining("Partial GitHub App config");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=GitHubTokenProviderProducerTest -q`
Expected: FAIL — `GitHubTokenProviderProducer` does not exist

- [ ] **Step 3: Implement GitHubTokenProviderProducer**

```java
package io.casehub.devtown.github;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.rest.client.inject.RestClient;

import java.util.Optional;

@ApplicationScoped
public class GitHubTokenProviderProducer {

    @Produces
    @ApplicationScoped
    public GitHubTokenProvider tokenProvider(
            @ConfigProperty(name = "devtown.github.app.id") Optional<String> appId,
            @ConfigProperty(name = "devtown.github.app.private-key-path") Optional<String> keyPath,
            @ConfigProperty(name = "devtown.github.app.installation-id") Optional<String> installationId,
            @ConfigProperty(name = "devtown.github.token") Optional<String> pat,
            @RestClient InstallationTokenClient tokenClient) {

        boolean hasAppId = appId.filter(s -> !s.isEmpty()).isPresent();
        boolean hasKeyPath = keyPath.filter(s -> !s.isEmpty()).isPresent();
        boolean hasInstId = installationId.filter(s -> !s.isEmpty()).isPresent();

        if (hasAppId && hasKeyPath && hasInstId) {
            return new AppTokenProvider(appId.get(), keyPath.get(),
                installationId.get(), tokenClient);
        }
        if (hasAppId || hasKeyPath || hasInstId) {
            throw new IllegalStateException(
                "Partial GitHub App config: all three of " +
                "devtown.github.app.id, devtown.github.app.private-key-path, " +
                "devtown.github.app.installation-id must be set, or none");
        }
        return new PatTokenProvider(pat.orElse(""));
    }
}
```

Note: The `@Produces` method filters empty strings — `${GITHUB_APP_ID:}` resolves
to `""` when the env var is unset, so `Optional.of("")` must be treated as absent.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=GitHubTokenProviderProducerTest -q`
Expected: PASS

- [ ] **Step 5: Write failing test for GitHubAuthHeadersFactory**

```java
package io.casehub.devtown.github;

import jakarta.ws.rs.core.MultivaluedHashMap;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class GitHubAuthHeadersFactoryTest {

    @Test
    void setsAuthorizationHeaderWhenTokenPresent() {
        var factory = new GitHubAuthHeadersFactory(new PatTokenProvider("ghp_test"));
        var incoming = new MultivaluedHashMap<String, String>();
        var outgoing = new MultivaluedHashMap<String, String>();

        factory.update(incoming, outgoing);

        assertThat(outgoing.getFirst("Authorization")).isEqualTo("Bearer ghp_test");
    }

    @Test
    void omitsAuthorizationHeaderWhenTokenEmpty() {
        var factory = new GitHubAuthHeadersFactory(new PatTokenProvider(""));
        var incoming = new MultivaluedHashMap<String, String>();
        var outgoing = new MultivaluedHashMap<String, String>();

        factory.update(incoming, outgoing);

        assertThat(outgoing.containsKey("Authorization")).isFalse();
    }

    @Test
    void setsAcceptAndApiVersionHeaders() {
        var factory = new GitHubAuthHeadersFactory(new PatTokenProvider("ghp_test"));
        var incoming = new MultivaluedHashMap<String, String>();
        var outgoing = new MultivaluedHashMap<String, String>();

        factory.update(incoming, outgoing);

        assertThat(outgoing.getFirst("Accept")).isEqualTo("application/vnd.github+json");
        assertThat(outgoing.getFirst("X-GitHub-Api-Version")).isEqualTo("2022-11-28");
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=GitHubAuthHeadersFactoryTest -q`
Expected: FAIL — `GitHubAuthHeadersFactory` does not exist

- [ ] **Step 7: Implement GitHubAuthHeadersFactory**

```java
package io.casehub.devtown.github;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.core.MultivaluedMap;
import org.eclipse.microprofile.rest.client.ext.ClientHeadersFactory;

@ApplicationScoped
public class GitHubAuthHeadersFactory implements ClientHeadersFactory {

    private final GitHubTokenProvider tokenProvider;

    @Inject
    public GitHubAuthHeadersFactory(GitHubTokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    public MultivaluedMap<String, String> update(
            MultivaluedMap<String, String> incomingHeaders,
            MultivaluedMap<String, String> clientOutgoingHeaders) {
        String token = tokenProvider.getToken();
        if (token != null && !token.isEmpty()) {
            clientOutgoingHeaders.putSingle("Authorization", "Bearer " + token);
        }
        clientOutgoingHeaders.putSingle("Accept", "application/vnd.github+json");
        clientOutgoingHeaders.putSingle("X-GitHub-Api-Version", "2022-11-28");
        return clientOutgoingHeaders;
    }
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -Dtest=GitHubAuthHeadersFactoryTest -q`
Expected: PASS

- [ ] **Step 9: Add @RegisterClientHeaders to all 5 REST client interfaces**

Add `import org.eclipse.microprofile.rest.client.annotation.RegisterClientHeaders;` and
`@RegisterClientHeaders(GitHubAuthHeadersFactory.class)` to each interface. The annotation
goes alongside the existing `@RegisterRestClient` annotation.

Files to modify (add annotation after `@RegisterRestClient` line):

`GitHubChecksApi.java`:
```java
@RegisterRestClient(configKey = "github-api")
@RegisterClientHeaders(GitHubAuthHeadersFactory.class)
@Path("/repos")
```

`GitHubPullRequestApi.java`:
```java
@RegisterRestClient(configKey = "github-api")
@RegisterClientHeaders(GitHubAuthHeadersFactory.class)
@Path("/repos")
```

`GitHubMergeApi.java`:
```java
@RegisterRestClient(configKey = "github-api")
@RegisterClientHeaders(GitHubAuthHeadersFactory.class)
@Path("/repos")
```

`GitHubRepoApi.java`:
```java
@RegisterRestClient(configKey = "github-api")
@RegisterClientHeaders(GitHubAuthHeadersFactory.class)
@Path("/repos")
```

`GitHubGitApi.java`:
```java
@RegisterRestClient(configKey = "github-api")
@RegisterClientHeaders(GitHubAuthHeadersFactory.class)
@Path("/repos")
```

- [ ] **Step 10: Add config properties to application.properties**

After the existing `devtown.github.merge-method=squash` line, add:

```properties
devtown.github.app.id=${GITHUB_APP_ID:}
devtown.github.app.private-key-path=${GITHUB_APP_PRIVATE_KEY_PATH:}
devtown.github.app.installation-id=${GITHUB_INSTALLATION_ID:}

# Installation token REST client (same base URL as github-api)
quarkus.rest-client.github-app-api.url=https://api.github.com
```

- [ ] **Step 11: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl github -q`
Expected: All github module tests pass

- [ ] **Step 12: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -q`
Expected: BUILD SUCCESS (all modules compile and tests pass)

- [ ] **Step 13: Commit**

```bash
git add github/src/main/java/io/casehub/devtown/github/GitHubTokenProviderProducer.java github/src/main/java/io/casehub/devtown/github/GitHubAuthHeadersFactory.java github/src/test/java/io/casehub/devtown/github/GitHubTokenProviderProducerTest.java github/src/test/java/io/casehub/devtown/github/GitHubAuthHeadersFactoryTest.java github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java github/src/main/java/io/casehub/devtown/github/GitHubPullRequestApi.java github/src/main/java/io/casehub/devtown/github/GitHubMergeApi.java github/src/main/java/io/casehub/devtown/github/GitHubRepoApi.java github/src/main/java/io/casehub/devtown/github/GitHubGitApi.java app/src/main/resources/application.properties
git commit -m "feat(github): wire auth headers to all REST clients via @Produces factory

GitHubTokenProviderProducer checks config: App credentials present →
AppTokenProvider, absent → PatTokenProvider. GitHubAuthHeadersFactory
injects Bearer token + GitHub recommended headers on all 5 REST clients.

Refs #218"
```

---

## Batch 3: Documentation

Safe wrap point: feature complete with documentation. Manifest and setup guide provide onboarding path.

### Task 5: GitHub App Manifest + Setup Guide

**Files:**
- Create: `github/src/main/resources/github-app-manifest.json`
- Create: `docs/guides/github-app-setup.md`

**Interfaces:**
- Consumes: config property names from Task 4 (documented in the guide)
- Produces: documentation artifacts (no code dependencies)

- [ ] **Step 1: Create GitHub App Manifest**

Create `github/src/main/resources/github-app-manifest.json`:

```json
{
  "name": "devtown",
  "url": "https://github.com/casehubio/devtown",
  "hook_attributes": {
    "url": "https://REPLACE_WITH_YOUR_DOMAIN/api/github/webhook",
    "active": true
  },
  "redirect_url": "https://REPLACE_WITH_YOUR_DOMAIN/api/github/app/callback",
  "public": false,
  "default_permissions": {
    "pull_requests": "write",
    "checks": "write",
    "contents": "read",
    "metadata": "read"
  },
  "default_events": [
    "pull_request",
    "pull_request_review",
    "check_suite",
    "check_run"
  ]
}
```

- [ ] **Step 2: Create setup guide**

Create `docs/guides/github-app-setup.md`:

```markdown
# GitHub App Setup Guide

## Overview

devtown authenticates with GitHub via a GitHub App, which provides:
- Higher rate limits (5000 req/hr per installation vs 60 unauthenticated)
- Bot identity (`devtown[bot]`) on PR comments and check runs
- Fine-grained permissions scoped to specific repos
- No personal token expiry or rotation needed

## Option A: One-Click Creation via Manifest

1. Navigate to:
   `https://github.com/settings/apps/new?manifest=<URL-encoded-manifest>`

   The manifest is at `github/src/main/resources/github-app-manifest.json`.
   Before using it, replace `REPLACE_WITH_YOUR_DOMAIN` with your actual
   domain (e.g., `devtown.example.com`).

2. GitHub will show the App configuration. Click **Create GitHub App**.

3. GitHub generates the App ID and private key automatically. Download
   the private key file (`.pem`) — this is shown only once.

4. Note the **App ID** shown on the App settings page.

## Option B: Manual Creation

1. Go to **Settings → Developer settings → GitHub Apps → New GitHub App**

2. Fill in:
   - **App name:** `devtown` (or `devtown-<org>` if name is taken)
   - **Homepage URL:** `https://github.com/casehubio/devtown`
   - **Webhook URL:** `https://<your-domain>/api/github/webhook`
   - **Webhook secret:** generate a secure random string

3. Set permissions:
   | Permission | Access |
   |-----------|--------|
   | Pull requests | Read & write |
   | Checks | Read & write |
   | Contents | Read |
   | Metadata | Read |

4. Subscribe to events:
   - `pull_request`
   - `pull_request_review`
   - `check_suite`
   - `check_run`

5. Set **Where can this GitHub App be installed?** to
   **Only on this account** (unless multi-org).

6. Click **Create GitHub App**.

7. On the App settings page, click **Generate a private key**.
   Save the downloaded `.pem` file securely.

## Installing on a Repository

1. Go to the App settings page → **Install App** (left sidebar)
2. Select the organization or account
3. Choose **Only select repositories** and pick the repos devtown manages
4. Click **Install**
5. Note the **Installation ID** from the URL:
   `https://github.com/settings/installations/<INSTALLATION_ID>`

## Configuring devtown

Set three environment variables:

```bash
export GITHUB_APP_ID=<app-id-from-settings-page>
export GITHUB_APP_PRIVATE_KEY_PATH=/path/to/your-app.private-key.pem
export GITHUB_INSTALLATION_ID=<installation-id-from-url>
```

For Kubernetes, mount the private key as a secret volume:

```yaml
volumes:
  - name: github-app-key
    secret:
      secretName: devtown-github-app-key
containers:
  - env:
      - name: GITHUB_APP_ID
        value: "123456"
      - name: GITHUB_APP_PRIVATE_KEY_PATH
        value: /secrets/github/private-key.pem
      - name: GITHUB_INSTALLATION_ID
        value: "78901234"
    volumeMounts:
      - name: github-app-key
        mountPath: /secrets/github
        readOnly: true
```

## PAT Fallback (Dev Mode)

When App config is not set, devtown falls back to a Personal Access Token:

```bash
export GITHUB_TOKEN=ghp_your_personal_access_token
```

This is suitable for local development. The PAT user identity appears on
API calls instead of `devtown[bot]`.

If neither App config nor PAT is set, API calls go out unauthenticated
(rate-limited to 60 requests/hour). This is acceptable when using NoOp
service stubs in dev mode.

## Verifying the Integration

After starting devtown with App config:

1. Check startup logs for: `Refreshing GitHub installation token for installation <id>`
2. The first webhook delivery should succeed without auth errors
3. Check run and PR comment operations should show `devtown[bot]` as the author

If you see `401 Unauthorized` errors:
- Verify the App ID matches the settings page
- Verify the private key file is readable and in PEM format
- Verify the installation ID (check the installations URL)
- Ensure the App has the required permissions on the target repo
```

- [ ] **Step 3: Commit**

```bash
git add github/src/main/resources/github-app-manifest.json docs/guides/github-app-setup.md
git commit -m "docs: add GitHub App manifest and setup guide

One-click manifest creation, manual setup alternative, Kubernetes
deployment, PAT fallback, and verification steps.

Refs #218"
```

---

## References

- [2026-10-15-github-app-auth-design.md](../specs/issue-218-github-app-auth/2026-10-15-github-app-auth-design.md) — design spec this plan implements
- `github/src/main/java/io/casehub/devtown/github/GitHubChecksApi.java` — existing REST client pattern (configKey, annotations)
- `github/src/main/java/io/casehub/devtown/github/GitHubCiStatusClient.java` — existing @RestClient injection pattern
- `app/src/main/resources/application.properties:52-55` — existing GitHub config properties
- `github/pom.xml` — existing module dependencies
- GitHub #218 — focal issue
