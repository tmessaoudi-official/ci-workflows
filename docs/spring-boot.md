# Spring Boot Workflow Usage Guide

## What It Does

```
build-and-test ──► build-jar
```

| Job | Tool | Purpose |
|-----|------|---------|
| `build-and-test` | Gradle + JUnit | Validates wrapper, runs all tests (unit + integration + ArchUnit), uploads reports |
| `build-jar` | Gradle bootJar | Produces the executable JAR artifact |

The `gradle/actions/setup-gradle` action (v5, MIT; see the README for why not v6) handles all Gradle caching automatically
(Gradle home, wrapper, build cache) — no manual `actions/cache` needed.

## Minimal Caller

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/spring-boot.yml@v1
    with:
      java-version: '21'
```

## All Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `java-version` | string | `'21'` | Java version (Temurin distribution) |
| `working-directory` | string | `'.'` | Path to the Gradle project root |
| `gradle-args` | string | `''` | Extra arguments appended to every `./gradlew` invocation |
| `run-archunit` | boolean | `true` | Documents that the project includes ArchUnit tests (informational — tests always run) |

## Secrets

This workflow requires no secrets.

## Service Containers Used

| Job | Containers |
|-----|-----------|
| `build-and-test` | PostgreSQL 17 |

Connection injected as Spring Boot env vars:
- `SPRING_DATASOURCE_URL=jdbc:postgresql://127.0.0.1:5432/test_db`
- `SPRING_DATASOURCE_USERNAME=test`
- `SPRING_DATASOURCE_PASSWORD=test`
- `SPRING_JPA_HIBERNATE_DDL_AUTO=create-drop`

Your `application-test.properties` or `application-test.yml` should be configured
to read from these env vars (Spring Boot auto-maps them via relaxed binding).

## Artifacts Produced

| Artifact | Job | Retention |
|----------|-----|-----------|
| `junit-spring-boot` | `build-and-test` | 7 days |
| `html-test-report-spring-boot` | `build-and-test` | 7 days |
| `spring-boot-jar` | `build-jar` | 7 days |

Both test report artifacts are uploaded even on failure (`if: always()`), so you can
always download the HTML report to diagnose failing tests.

## Expected Project Structure

```
your-spring-boot-project/
├── gradlew                  ← must be executable (chmod +x)
├── gradlew.bat
├── gradle/
│   └── wrapper/
│       └── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── build.gradle.kts         ← or build.gradle
└── src/
    └── test/
        └── java/
```

## Tips

### Wrapper Executable Permission

The workflow runs `chmod +x ./gradlew` automatically. If your repo was checked out on
Windows and the wrapper lost its execute bit, this step handles it.

### Gradle Build Scan on Main

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/spring-boot.yml@v1
    with:
      gradle-args: ${{ github.ref == 'refs/heads/main' && '--scan' || '' }}
```

### Multi-module Projects

The JAR artifact path uses `**/build/libs/*.jar` and the test report path uses
`**/build/test-results/test/*.xml`, which works correctly for both single-module
and multi-module Gradle projects.
