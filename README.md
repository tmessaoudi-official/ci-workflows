# ci-workflows

A library of reusable GitHub Actions workflows for PHP/Symfony, Angular, Java/Spring Boot,
and NestJS projects. Battle-tested, opinionated, and designed around one principle:

> **Write your CI pipeline once. Reference it from every project.**

Instead of copy-pasting 200 lines of CI YAML into every repository, each project
references a single line:

```yaml
uses: takieddine-messaoudi/ci-workflows/.github/workflows/symfony.yml@v1
```

Update the library once and every project inherits the change on its next run.

---

## Quick Start

Five lines to get production-grade CI for a Symfony project:

```yaml
# .github/workflows/ci.yml in your Symfony project
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: takieddine-messaoudi/ci-workflows/.github/workflows/symfony.yml@v1
    with:
      php-version: '8.4'
      phpstan-level: 8
```

That gives you: PHP CS Fixer, Rector, PHPStan, Pest unit + integration + E2E tests,
PostgreSQL and Redis service containers, coverage reporting, and code style enforcement —
with no further configuration.

---

## Available Workflows

| Workflow | Description | Docs |
|----------|-------------|------|
| [`symfony.yml`](.github/workflows/symfony.yml) | PHP CS Fixer + Rector + PHPStan + Pest (unit/integration/e2e) + Behat + Infection | [docs/symfony.md](docs/symfony.md) |
| [`angular.yml`](.github/workflows/angular.yml) | ESLint + tsc + Stylelint + Vitest + Playwright + ng build | [docs/angular.md](docs/angular.md) |
| [`spring-boot.yml`](.github/workflows/spring-boot.yml) | Gradle wrapper validation + JUnit + ArchUnit + bootJar | [docs/spring-boot.md](docs/spring-boot.md) |
| [`nestjs.yml`](.github/workflows/nestjs.yml) | ESLint + tsc + Jest (unit + e2e) + nest build | [docs/nestjs.md](docs/nestjs.md) |
| [`docker-build.yml`](.github/workflows/docker-build.yml) | BuildX multi-platform build + GHCR push + GHA layer cache | — |
| [`security-scan.yml`](.github/workflows/security-scan.yml) | Gitleaks secrets scan + Trivy vulnerability scan → SARIF | [docs/security.md](docs/security.md) |

---

## Workflow Inputs — At a Glance

Full reference: [docs/inputs-reference.md](docs/inputs-reference.md)

### symfony.yml

```yaml
with:
  php-version: '8.4'              # PHP version
  composer-flags: '--prefer-dist' # Extra composer install flags
  phpstan-level: 8                # 0–9
  run-mutation: false             # Infection (slow — run on main only)
  working-directory: '.'          # For monorepos
  php-extensions: 'intl, pdo_pgsql, redis, amqp'
secrets:
  COMPOSER_AUTH: ${{ secrets.COMPOSER_AUTH }}  # Optional
```

### angular.yml

```yaml
with:
  node-version: '22'
  package-manager: 'npm'    # or 'pnpm'
  run-e2e: true             # Playwright (disable for feature branch PRs)
  working-directory: '.'
  base-href: '/'            # For apps deployed at a sub-path
```

### spring-boot.yml

```yaml
with:
  java-version: '21'        # Temurin distribution
  working-directory: '.'
  gradle-args: ''           # Extra flags for every ./gradlew invocation
```

### nestjs.yml

```yaml
with:
  node-version: '22'
  package-manager: 'pnpm'  # or 'npm'
  working-directory: '.'
```

### docker-build.yml

```yaml
with:
  image-name: 'my-org/my-app'   # Required
  dockerfile: 'Dockerfile'
  context: '.'
  push: false                    # Set true to push to registry
  registry: 'ghcr.io'
  platforms: 'linux/amd64'
secrets:
  REGISTRY_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### security-scan.yml

```yaml
with:
  scan-path: '.'
  trivy-severity: 'CRITICAL,HIGH'
  fail-on-secrets: true
```

---

## Why Reusable Workflows Over Composite Actions?

Both allow you to share CI logic, but they work differently:

| | Reusable Workflows | Composite Actions |
|--|-------------------|-------------------|
| **Unit of sharing** | Entire jobs | Individual steps |
| **Service containers** | Supported | Not supported |
| **Secrets** | Explicit re-declaration | Inherited |
| **Caller syntax** | `uses:` at the job level | `uses:` at the step level |
| **Parallelism** | Caller controls `needs:` | Sequential within a job |
| **Matrix strategy** | Supported at caller level | Not applicable |

For CI pipelines that need service containers (PostgreSQL, Redis, RabbitMQ), parallel
jobs across stages, and the ability for callers to add their own jobs (like `docker-build`)
around the shared pipeline — reusable workflows are the right tool.

Composite actions are better for sharing a few steps (like "setup PHP + install Composer")
that you want to embed inside a larger job you fully control.

---

## Security by Default

Every project that uses this library gets security scanning as a first-class citizen,
not an afterthought:

**Gitleaks** scans the full git history for accidentally committed secrets — API keys,
passwords, tokens, private keys. It catches secrets that were deleted in a later commit
but are still accessible in git history.

**Trivy** scans all lockfiles (composer.lock, package-lock.json, pom.xml, *.gradle.kts)
for known CVEs. Results appear in your repository's GitHub Security tab via SARIF upload,
giving you a dashboard of vulnerabilities across all branches.

Details on both tools, configuration, and remediation: [docs/security.md](docs/security.md)

---

## Full-stack Example (Backend + Frontend in Parallel)

```yaml
name: Full-stack CI
on: [push, pull_request]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  backend:
    uses: takieddine-messaoudi/ci-workflows/.github/workflows/symfony.yml@v1
    with:
      working-directory: 'backend'
      run-mutation: ${{ github.ref == 'refs/heads/main' }}
    secrets:
      COMPOSER_AUTH: ${{ secrets.COMPOSER_AUTH }}

  frontend:
    uses: takieddine-messaoudi/ci-workflows/.github/workflows/angular.yml@v1
    with:
      working-directory: 'frontend'
      run-e2e: ${{ github.ref == 'refs/heads/main' }}

  security:
    uses: takieddine-messaoudi/ci-workflows/.github/workflows/security-scan.yml@v1
    needs: [backend, frontend]

  docker:
    uses: takieddine-messaoudi/ci-workflows/.github/workflows/docker-build.yml@v1
    needs: [backend, frontend]
    with:
      image-name: ${{ github.repository }}
      push: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
    secrets:
      REGISTRY_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Backend and frontend CI run completely in parallel. Docker build only starts when
both pass. Security scan runs after both and in parallel with Docker.

---

## How to Contribute a New Workflow

See [CONTRIBUTING.md](CONTRIBUTING.md) for the complete contributor guide. The short version:

1. Every workflow uses `on: workflow_call:` only (no `push:` or `pull_request:`)
2. Follow the fail-fast job order: lint → unit tests → integration → E2E
3. All service containers must have health checks
4. Cache keys: `${{ runner.os }}-<tool>-${{ hashFiles('<lockfile>') }}`
5. Use `permissions:` at the job level with the minimum required
6. Add a doc in `docs/<name>.md` and an example in `examples/<name>-caller.yml`
7. Validate with `act workflow_dispatch -W .github/workflows/_test.yml`

---

## Versioning

Production callers should pin to a version tag, not `@main`:

```yaml
# Good — gets non-breaking updates within v1 automatically
uses: takieddine-messaoudi/ci-workflows/.github/workflows/symfony.yml@v1

# Also good — pinned to an exact release
uses: takieddine-messaoudi/ci-workflows/.github/workflows/symfony.yml@v1.2.0

# Avoid for production — may break on any commit
uses: takieddine-messaoudi/ci-workflows/.github/workflows/symfony.yml@main
```

Version semantics:
- **Patch** (`v1.0.1`): bug fixes, action version bumps
- **Minor** (`v1.1.0`): new optional inputs with defaults (non-breaking)
- **Major** (`v2.0.0`): renamed/removed inputs, changed job structure

Major version tags (`v1`, `v2`) float to the latest minor/patch within that major.

---

## License

MIT — see [LICENSE](LICENSE).
