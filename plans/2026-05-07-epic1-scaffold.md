# Epic 1: Project Scaffold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a buildable five-module Quarkus Maven project that resolves all CaseHub foundation dependencies from GitHub Packages and passes `mvn clean install`.

**Architecture:** Five Maven modules (`devtown-domain`, `devtown-review`, `devtown-queue`, `devtown-github`, `devtown-app`) under a parent POM that inherits from `casehub-parent`. Each module contains a single package placeholder; `devtown-app` runs a `@QuarkusTest` boot test to verify Quarkus starts cleanly.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven, GitHub Packages, GitHub Actions

**Working directory:** `/Users/mdproctor/claude/casehub/devtown` (the project repo, NOT the workspace)

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode clean install`

---

## File Map

```
pom.xml                                                          ← parent POM
devtown-domain/
  pom.xml
  src/main/java/io/casehub/devtown/domain/package-info.java
devtown-review/
  pom.xml
  src/main/java/io/casehub/devtown/review/package-info.java
  src/main/java/io/casehub/devtown/trust/package-info.java
devtown-queue/
  pom.xml
  src/main/java/io/casehub/devtown/queue/package-info.java
devtown-github/
  pom.xml
  src/main/java/io/casehub/devtown/github/package-info.java
devtown-app/
  pom.xml
  src/main/java/io/casehub/devtown/app/package-info.java
  src/main/java/io/casehub/devtown/notify/package-info.java
  src/main/resources/application.properties
  src/test/java/io/casehub/devtown/app/DevtownBootTest.java
  src/test/resources/application.properties
.github/workflows/ci.yml
.github/workflows/publish.yml
```

---

## Task 1: Parent POM

**Files:**
- Create: `pom.xml`

- [ ] **Step 1: Create the parent POM**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-devtown</artifactId>
  <version>0.2-SNAPSHOT</version>
  <packaging>pom</packaging>

  <name>CaseHub DevTown</name>
  <description>AI-assisted software development application on the CaseHub platform</description>
  <url>https://github.com/casehubio/devtown</url>

  <modules>
    <module>devtown-domain</module>
    <module>devtown-review</module>
    <module>devtown-queue</module>
    <module>devtown-github</module>
    <module>devtown-app</module>
  </modules>

  <dependencyManagement>
    <dependencies>
      <!-- casehub-engine-api — not in casehub-parent BOM -->
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-api</artifactId>
        <version>${casehub.version}</version>
      </dependency>

      <!-- casehub-qhorus-api — not in casehub-parent BOM -->
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-qhorus-api</artifactId>
        <version>${casehub.version}</version>
      </dependency>

      <!-- casehub-connectors — not in casehub-parent BOM -->
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-connectors</artifactId>
        <version>${casehub.version}</version>
      </dependency>

      <!-- devtown inter-module -->
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>devtown-domain</artifactId>
        <version>${project.version}</version>
      </dependency>
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>devtown-review</artifactId>
        <version>${project.version}</version>
      </dependency>
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>devtown-queue</artifactId>
        <version>${project.version}</version>
      </dependency>
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>devtown-github</artifactId>
        <version>${project.version}</version>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <distributionManagement>
    <repository>
      <id>github</id>
      <name>casehubio GitHub Packages</name>
      <url>https://maven.pkg.github.com/casehubio/devtown</url>
    </repository>
    <snapshotRepository>
      <id>github</id>
      <name>casehubio GitHub Packages</name>
      <url>https://maven.pkg.github.com/casehubio/devtown</url>
    </snapshotRepository>
  </distributionManagement>

</project>
```

- [ ] **Step 2: Verify parent POM parses**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode validate
```

Expected: `BUILD SUCCESS` (no modules compiled yet — just validates the POM is well-formed and the casehub-parent resolves from GitHub Packages)

- [ ] **Step 3: Commit**

```bash
git add pom.xml
git commit -m "build: parent POM — 5-module casehub-devtown scaffold

Refs #8"
```

---

## Task 2: devtown-domain module

**Files:**
- Create: `devtown-domain/pom.xml`
- Create: `devtown-domain/src/main/java/io/casehub/devtown/domain/package-info.java`

- [ ] **Step 1: Create module POM**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-devtown</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>devtown-domain</artifactId>
  <name>DevTown :: Domain</name>
  <description>Capability tags, trust dimensions, routing thresholds — pure Java domain vocabulary</description>

</project>
```

- [ ] **Step 2: Create package placeholder**

`devtown-domain/src/main/java/io/casehub/devtown/domain/package-info.java`:
```java
package io.casehub.devtown.domain;
```

- [ ] **Step 3: Verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl devtown-domain
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add devtown-domain/
git commit -m "build: devtown-domain module scaffold

Refs #8"
```

---

## Task 3: devtown-review module

**Files:**
- Create: `devtown-review/pom.xml`
- Create: `devtown-review/src/main/java/io/casehub/devtown/review/package-info.java`

- [ ] **Step 1: Create module POM**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-devtown</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>devtown-review</artifactId>
  <name>DevTown :: Review</name>
  <description>PR review CasePlanModel, review bindings, trust-weighted routing</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-domain</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-api</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-work-api</artifactId>
    </dependency>
  </dependencies>

</project>
```

- [ ] **Step 2: Create package placeholders**

`devtown-review/src/main/java/io/casehub/devtown/review/package-info.java`:
```java
package io.casehub.devtown.review;
```

`devtown-review/src/main/java/io/casehub/devtown/trust/package-info.java`:
```java
package io.casehub.devtown.trust;
```

- [ ] **Step 3: Verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl devtown-domain,devtown-review
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add devtown-review/
git commit -m "build: devtown-review module scaffold

Refs #8"
```

---

## Task 4: devtown-queue module

**Files:**
- Create: `devtown-queue/pom.xml`
- Create: `devtown-queue/src/main/java/io/casehub/devtown/queue/package-info.java`

- [ ] **Step 1: Create module POM**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-devtown</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>devtown-queue</artifactId>
  <name>DevTown :: Queue</name>
  <description>Merge queue orchestration — batch management and batch-then-bisect</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-domain</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-review</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-api</artifactId>
    </dependency>
  </dependencies>

</project>
```

- [ ] **Step 2: Create package placeholder**

`devtown-queue/src/main/java/io/casehub/devtown/queue/package-info.java`:
```java
package io.casehub.devtown.queue;
```

- [ ] **Step 3: Verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl devtown-domain,devtown-review,devtown-queue
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add devtown-queue/
git commit -m "build: devtown-queue module scaffold

Refs #8"
```

---

## Task 5: devtown-github module

**Files:**
- Create: `devtown-github/pom.xml`
- Create: `devtown-github/src/main/java/io/casehub/devtown/github/package-info.java`

- [ ] **Step 1: Create module POM**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-devtown</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>devtown-github</artifactId>
  <name>DevTown :: GitHub</name>
  <description>GitHub boundary — webhook/polling receiver, CI status reader, merge executor</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-domain</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-review</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-queue</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-client</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-client-jackson</artifactId>
    </dependency>
  </dependencies>

</project>
```

- [ ] **Step 2: Create package placeholder**

`devtown-github/src/main/java/io/casehub/devtown/github/package-info.java`:
```java
package io.casehub.devtown.github;
```

- [ ] **Step 3: Verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl devtown-domain,devtown-review,devtown-queue,devtown-github
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add devtown-github/
git commit -m "build: devtown-github module scaffold

Refs #8"
```

---

## Task 6: devtown-app module

**Files:**
- Create: `devtown-app/pom.xml`
- Create: `devtown-app/src/main/java/io/casehub/devtown/app/package-info.java`
- Create: `devtown-app/src/main/resources/application.properties`
- Create: `devtown-app/src/test/java/io/casehub/devtown/app/DevtownBootTest.java` ← write this first
- Create: `devtown-app/src/test/resources/application.properties`

- [ ] **Step 1: Create the boot test first (it will fail — no pom.xml yet)**

`devtown-app/src/test/java/io/casehub/devtown/app/DevtownBootTest.java`:
```java
package io.casehub.devtown.app;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DevtownBootTest {

    @Test
    void applicationBoots() {
    }
}
```

- [ ] **Step 2: Create module POM**

`devtown-app/pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-devtown</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>devtown-app</artifactId>
  <name>DevTown :: App</name>
  <description>Quarkus assembly — REST entry points, notification wiring, runtime deps</description>

  <dependencies>
    <!-- devtown modules -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-domain</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-review</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-queue</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>devtown-github</artifactId>
    </dependency>

    <!-- foundation runtime -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-work</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors</artifactId>
    </dependency>

    <!-- Quarkus extensions -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health</artifactId>
    </dependency>

    <!-- test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-qhorus-testing</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.platform.version}</version>
        <extensions>true</extensions>
        <executions>
          <execution>
            <goals>
              <goal>build</goal>
              <goal>generate-code</goal>
              <goal>generate-code-tests</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 3: Create package placeholders**

`devtown-app/src/main/java/io/casehub/devtown/app/package-info.java`:
```java
package io.casehub.devtown.app;
```

`devtown-app/src/main/java/io/casehub/devtown/notify/package-info.java`:
```java
package io.casehub.devtown.notify;
```

- [ ] **Step 4: Create main application.properties**

`devtown-app/src/main/resources/application.properties`:
```properties
quarkus.http.port=8080

# GitHub integration — set via environment
# devtown.github.token=${GITHUB_TOKEN:}
# devtown.github.base-url=https://api.github.com
```

- [ ] **Step 5: Create test application.properties**

Mirrors the claudony pattern — satisfies qhorus datasource and CDI indexing requirements.

`devtown-app/src/test/resources/application.properties`:
```properties
quarkus.http.test-port=0

# casehub-qhorus 0.2-SNAPSHOT embeds properties that strict config validation rejects
# as unmapped (SRCFG00050) when the extension descriptor isn't registered at augmentation
smallrye.config.mapping.validate-unknown=false

# Index qhorus-testing jar so InMemory*Store CDI beans are discoverable in tests
quarkus.index-dependency.qhorus-testing.group-id=io.casehub
quarkus.index-dependency.qhorus-testing.artifact-id=casehub-qhorus-testing

# Qhorus named datasource — in-memory H2 for tests
%test.quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:devtown-qhorus-test;DB_CLOSE_ON_EXIT=FALSE
%test.quarkus.datasource.qhorus.reactive=false
%test.quarkus.flyway.qhorus.migrate-at-start=false
%test.quarkus.hibernate-orm.qhorus.database.generation=drop-and-create
```

- [ ] **Step 6: Run the boot test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl devtown-domain,devtown-review,devtown-queue,devtown-github,devtown-app
```

Expected: `BUILD SUCCESS` with `DevtownBootTest > applicationBoots() PASSED`

If the test fails with datasource or config errors, add the missing properties to `devtown-app/src/test/resources/application.properties` following the same pattern (named datasource URL, `reactive=false`, `migrate-at-start=false`, `database.generation=drop-and-create`).

- [ ] **Step 7: Commit**

```bash
git add devtown-app/
git commit -m "build: devtown-app module — Quarkus boot test passes

Refs #8"
```

---

## Task 7: GitHub Actions workflows

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.github/workflows/publish.yml`

- [ ] **Step 1: Create CI workflow**

`.github/workflows/ci.yml`:
```yaml
name: Build and Test

on:
  repository_dispatch:
    types: [upstream-published]
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read

    steps:
      - uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          server-id: github
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN

      - name: Build and test
        run: mvn --batch-mode verify
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: Create publish workflow**

`.github/workflows/publish.yml`:
```yaml
name: Publish

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          server-id: github
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN

      - name: Publish to GitHub Packages
        run: mvn --batch-mode deploy -DskipTests
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/
git commit -m "ci: add build/test and publish workflows

Refs #8"
```

---

## Task 8: Full build verification

- [ ] **Step 1: Run the full build from clean**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode clean install
```

Expected: `BUILD SUCCESS`. All five modules compile, `DevtownBootTest` passes.

- [ ] **Step 2: Close the epic issue**

```bash
git commit --allow-empty -m "chore: Epic 1 complete — scaffold builds and boots

Closes #8"
```

If a non-empty commit is preferred, make a trivial doc touch instead:
```bash
echo "" >> README.md && git add README.md
git commit -m "chore: Epic 1 complete — scaffold builds and boots

Closes #8"
```
