# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`track-me-platform` is the shared foundation for the **Track Me** microservices — it is a
library/tooling project, not a runnable application. It publishes three artifacts to GitHub
Packages that downstream services consume:

- **`platform`** (`net.trackme.platform:platform`) — a Gradle `java-platform` BOM that pins
  every dependency version (Spring Boot 3.5.0, Spring Cloud 2024.0.1, JUnit, Testcontainers,
  plus `constraints` for lombok, mapstruct, liquibase, springdoc, all Spring Security modules,
  telegrambots, poi). Services apply this so they never declare versions themselves.
- **`commons`** (`net.trackme.commons:commons`) — shared Java library (base JPA entities,
  Spring Security ACL helpers, dynamic JPA-Criteria filtering). See [Commons](#commons-library).
- **`conventions`** — a Gradle convention plugin `net.trackme.java-quality` (checkstyle +
  Error Prone + jacoco) applied by every service for consistent code-quality gates.

Note: code comments, log messages, Javadoc, and the README are written in **Russian**. Match the
surrounding language when editing.

## Build & test commands

```bash
./gradlew build                              # compile + test + checkstyle + jacoco for platform & commons
./gradlew :commons:test                      # tests for one module
./gradlew :commons:test --tests 'net.trackme.commons.filters.FilterTest'          # single test class
./gradlew :commons:test --tests 'net.trackme.commons.filters.FilterTest.methodName'  # single method
./gradlew publishToMavenLocal                # publish platform + commons to ~/.m2 (for local consumers)
./gradlew publishConventionsToMavenLocal     # publish the net.trackme.java-quality plugin to ~/.m2
```

Java 21 (toolchain), Gradle 8.8 (wrapper). Tests use JUnit 5 + Mockito; ACL/service tests are
plain mock-based unit tests (`@ExtendWith(MockitoExtension.class)`), not Spring context tests.

## Build architecture — the three-build layout

This is the non-obvious part of the repo. There are **two separate Gradle builds**:

1. The **root build** contains `platform` and `commons` as normal subprojects.
2. **`conventions/` is an included build** (`includeBuild('conventions')` in `settings.gradle`
   `pluginManagement`). This lets the root build apply the plugin it produces, but it means
   `conventions` is **not** part of `./gradlew publish` or `./gradlew build`.

Consequences to remember:
- The convention plugin is published by **separate tasks** defined in the root `build.gradle`:
  `publishConventions` / `publishConventionsToMavenLocal`, which delegate into the included build.
- The included build does **not** inherit the root `gradle.properties`, so
  `conventions/build.gradle` reads the version manually from the parent `gradle.properties`.
  A `-Pversion` on the command line overrides it (CI uses this on release).
- `commons` declares its dependencies **without versions** (versions come from the `platform`
  BOM). The published POM injects resolved versions via
  `versionMapping { ... fromResolutionResult() }` in `commons/build.gradle`.

## The `net.trackme.java-quality` convention plugin

Lives in `conventions/src/main/groovy/net.trackme.java-quality.gradle`. It wires up `checkstyle`,
Error Prone and `jacoco`:

- **Checkstyle** uses Google style. The config XMLs are **packaged inside the plugin jar**
  (`conventions/src/main/resources/net/trackme/checkstyle/`) and extracted at build time to
  `build/checkstyle-config`, because Checkstyle requires real files, not classpath resources.
  Currently `ignoreFailures = true` — violations do **not** fail the build.
- **Error Prone** runs on every `JavaCompile` task via the `net.ltgt.errorprone` plugin (added as
  an `implementation` dep of the `conventions` build, so consumers need `gradlePluginPortal()` in
  their `pluginManagement` to resolve it). The analyzer version (`error_prone_core`) defaults to
  `2.50.0`, overridable via `gradle.properties`: `trackme.errorprone.version=…`. By default it is
  **soft** (`allErrorsAsWarnings`) — findings print as warnings and do **not** fail the build, like
  checkstyle; set `trackme.errorprone.strict=true` to let Error Prone errors fail compilation.
  Generated code (Lombok/MapStruct) is skipped (`disableWarningsInGeneratedCode`, `excludedPaths`).
- **JaCoCo** coverage verification **does** fail the build. Minimum line coverage defaults to
  `0.60`, overridable per-consumer via `gradle.properties`: `trackme.coverage.minimum=0.45`.
  `check` depends on `jacocoTestCoverageVerification`.

## Versioning & release

- Current version is `version=1.0.0-SNAPSHOT` in root `gradle.properties`.
- **Release = git tag `v*`.** `.github/workflows/publish.yml` runs
  `./gradlew publish publishConventions -Pversion="${GITHUB_REF_NAME#v}"` (strips the `v`).
  SNAPSHOTs are not published to GitHub Packages.
- PRs labeled `publish-snapshot` get a per-PR snapshot published by `pr-checks.yml`
  (version `…-pr<N>-SNAPSHOT`).
- `pr-checks.yml` also runs `./gradlew build` + SonarCloud (`./gradlew sonar`) on PRs and pushes
  to `master`/`develop`. Sonar needs `SONAR_TOKEN`; it is skipped for fork PRs (no secret access).
- Consuming the artifacts (and the plugin) requires a GitHub token with `read:packages`, supplied
  as `gpr.user`/`gpr.key` Gradle properties or `GITHUB_ACTOR`/`GITHUB_TOKEN` env vars.

## Commons library

`commons/src/main/java/net/trackme/commons/`

- **`dao/`** — JPA base classes forming a hierarchy:
  `CoreEntity<ID>` (id accessor interface) → `BusinessEntity<ID>` (`@MappedSuperclass` adding
  audit columns `created_by`/`created_date`/`last_updated_by`/`last_updated_date`; `@PrePersist`
  and `@PreUpdate` resolve the current user from `SecurityContextHolder`, falling back to the
  `"system"` user) → `VersionedBusinessEntity<ID>` (adds a `@Version` optimistic-lock column).
- **`acl/`** — `AclService` wraps Spring Security's `MutableAclService` to create ACLs (granting
  full permissions to owner, creator, and the `ROLE_SUPER_ADMIN`/`ROLE_ADMIN` roles), change the
  owner, and delete ACLs. `PostgresJdbcMutableAclService` extends `JdbcMutableAclService` with
  Postgres-specific class/sid identity queries and `aclClassIdSupported = true`.
- **`filters/`** — dynamic JPA Criteria filtering for search endpoints. `Filter` (a record) maps
  to a `Predicate` via `toPredicate(Root, CriteriaBuilder)`, supporting nested dot-paths
  (`a.b.c`), a special `.year`/`year` shortcut that BETWEEN-ranges a `startDate`/`date` field,
  string→type conversion through `TYPE_CONVERTERS` (+ enum handling), and the operations in the
  `OperationType` enum (serialized to short JSON codes like `EQ`, `GT`, `IN`, `LIKE`).
  `FilterRequest` wraps a validated list; `FilterFieldNotAllowedException` guards disallowed fields.
