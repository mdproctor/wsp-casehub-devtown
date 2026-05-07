# Epic 1: Project Scaffold — Design Spec
2026-05-07

## Context

`casehub-devtown` is the AI-assisted software development application on the CaseHub platform. This spec covers the Maven project structure, module boundaries, dependency graph, and CI/publish workflows that form the buildable skeleton — no application logic, just the structure everything else will be built into.

Done when: `mvn clean install` succeeds with all foundation modules resolved from GitHub Packages.

## Module Structure

Five-module Maven multi-module layout. More modules early forces discipline — cleaner interfaces, stops domain logic bleeding across boundaries, makes it easier to combine than to split later.

```
casehub-devtown (parent pom, packaging=pom)
├── devtown-domain      pure Java — capability tags, trust dimensions, routing thresholds
├── devtown-review      PR review CasePlanModel, review bindings, trust-weighted routing logic
├── devtown-queue       merge queue orchestration, batch-then-bisect
├── devtown-github      GitHub boundary — webhook/polling receiver, CI status reader, merge executor
└── devtown-app         Quarkus assembly — REST entry points, notification wiring, runtime deps
```

`trust` is a package within `devtown-review` (trust-weighted routing is inseparable from review routing decisions). `notify` is a package within `devtown-app` (notification wiring is application-level connector setup).

## Dependency Graph

| Module | Depends on |
|--------|-----------|
| `devtown-domain` | nothing — pure Java constants, no foundation deps |
| `devtown-review` | `devtown-domain`, `casehub-engine-api`, `casehub-qhorus-api`, `casehub-ledger-api`, `casehub-work-api` |
| `devtown-queue` | `devtown-domain`, `devtown-review`, `casehub-engine-api` |
| `devtown-github` | `devtown-domain`, `devtown-review`, `devtown-queue`, `quarkus-rest-client`, `quarkus-rest-client-jackson` |
| `devtown-app` | all above modules + runtime: `casehub-qhorus`, `casehub-ledger`, `casehub-work`, `casehub-connectors`, `quarkus-rest`, `quarkus-rest-jackson`, `quarkus-jdbc-h2`, `quarkus-smallrye-health` |

All casehub artifacts are `0.2-SNAPSHOT` resolved from `https://maven.pkg.github.com/casehubio/*`. Parent POM coordinates: `io.casehub:casehub-parent:0.2-SNAPSHOT`.

## POM Hierarchy

Parent pom (`casehub-devtown`) inherits from `io.casehub:casehub-parent:0.2-SNAPSHOT` and imports the Quarkus platform BOM. It pins all casehub artifact versions via `<dependencyManagement>` using `${casehub.version}` = `0.2-SNAPSHOT`. Stack: Java 21 source/target (on Java 26 JVM at build time), Quarkus 3.32.2.

Each child module inherits from the parent pom. No version declarations in child poms except for non-casehub, non-Quarkus deps not covered by either BOM.

## Package Structure

Each module owns one top-level package:

| Module | Root package |
|--------|-------------|
| `devtown-domain` | `io.casehub.devtown.domain` |
| `devtown-review` | `io.casehub.devtown.review` |
| `devtown-queue` | `io.casehub.devtown.queue` |
| `devtown-github` | `io.casehub.devtown.github` |
| `devtown-app` | `io.casehub.devtown.app` |

The scaffold creates the package directories with placeholder classes (one per module) sufficient for Maven to compile and Quarkus to boot. For `devtown-app` this means at minimum a `@QuarkusTest`-annotated empty test so that `mvn verify` exercises the Quarkus boot path.

## Quarkus Application

`devtown-app` is the only runnable module. It gets:
- `src/main/java/io/casehub/devtown/app/DevtownApplication.java` — empty `@QuarkusMain` (or just rely on Quarkus auto-discovery; no explicit main needed unless customising startup)
- `src/main/resources/application.properties` — skeleton with commented stubs for GitHub connection config, database config, and Quarkus HTTP port

## GitHub Actions Workflows

Two workflows, matching the claudony pattern:

**`ci.yml`** — Build and test on every change:
- Triggers: `push` to `main`, `pull_request` targeting `main`, `repository_dispatch: upstream-published`, `workflow_dispatch`
- Java 21 / Temurin, `server-id: github` for GitHub Packages read
- Runs: `mvn --batch-mode verify`
- `GITHUB_TOKEN` passed for packages read

**`publish.yml`** — Publish to GitHub Packages:
- Triggers: `push` to `main`, `workflow_dispatch`
- Same Java setup, `server-id: github`
- Runs: `mvn --batch-mode deploy -DskipTests`
- `GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}` for packages write (scoped to this repo)

## Out of Scope

This epic contains no application logic. No CasePlanModel YAML, no domain constants, no worker implementations — those belong to Epics 2 and 3. The placeholder classes exist only to satisfy the compiler.
