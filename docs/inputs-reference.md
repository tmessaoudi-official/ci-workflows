# Inputs & Outputs Reference

Complete reference for all inputs, secrets, and outputs across every workflow.

## symfony.yml

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `php-version` | string | `'8.4'` | PHP version installed via shivammathur/setup-php |
| `composer-flags` | string | `'--prefer-dist --no-progress'` | Extra flags for `composer install` |
| `phpstan-level` | number | `8` | PHPStan analysis level (0–9) |
| `run-mutation` | boolean | `false` | Run Infection mutation testing |
| `working-directory` | string | `'.'` | Path to the Symfony project root |
| `php-extensions` | string | `'intl, pdo_pgsql, redis, amqp'` | PHP extensions to install |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `COMPOSER_AUTH` | No | JSON string for private Packagist or GitHub Packages auth |

### Outputs

None — coverage and junit artifacts are uploaded directly.

---

## angular.yml

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `node-version` | string | `'22'` | Node.js version |
| `working-directory` | string | `'.'` | Path to the Angular project root |
| `run-e2e` | boolean | `true` | Whether to run Playwright E2E tests |
| `package-manager` | string | `'npm'` | `npm` or `pnpm` |
| `base-href` | string | `'/'` | `--base-href` for the production build |

### Secrets

None.

### Outputs

None — `angular-dist` artifact is uploaded directly.

---

## spring-boot.yml

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `java-version` | string | `'21'` | Java version (Temurin distribution) |
| `working-directory` | string | `'.'` | Path to the Gradle project root |
| `gradle-args` | string | `''` | Extra arguments for every Gradle invocation |
| `run-archunit` | boolean | `true` | Informational: documents that ArchUnit tests are expected |

### Secrets

None.

### Outputs

None — JAR and test report artifacts are uploaded directly.

---

## nestjs.yml

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `node-version` | string | `'22'` | Node.js version |
| `package-manager` | string | `'pnpm'` | `npm` or `pnpm` |
| `working-directory` | string | `'.'` | Path to the NestJS project root |

### Secrets

None.

### Outputs

None — `nestjs-dist` artifact is uploaded directly.

---

## docker-build.yml

### Inputs

| Name | Type | Default | Required | Description |
|------|------|---------|----------|-------------|
| `image-name` | string | — | **Yes** | Image name without registry prefix (e.g. `my-org/my-app`) |
| `dockerfile` | string | `'Dockerfile'` | No | Path to Dockerfile |
| `context` | string | `'.'` | No | Docker build context |
| `push` | boolean | `false` | No | Push image to registry after build |
| `registry` | string | `'ghcr.io'` | No | Container registry hostname |
| `platforms` | string | `'linux/amd64'` | No | Comma-separated target platforms |

### Secrets

| Name | Required | Description |
|------|----------|-------------|
| `REGISTRY_TOKEN` | No | Registry auth token; defaults to `GITHUB_TOKEN` for GHCR |

### Outputs

| Name | Description |
|------|-------------|
| `image-digest` | SHA256 digest of the built image |
| `image-metadata` | JSON metadata including all tags applied |

---

## security-scan.yml

### Inputs

| Name | Type | Default | Description |
|------|------|---------|-------------|
| `scan-path` | string | `'.'` | Path to scan |
| `trivy-severity` | string | `'CRITICAL,HIGH'` | Severity levels that cause build failure |
| `fail-on-secrets` | boolean | `true` | Whether to fail when Gitleaks finds secrets |

### Secrets

None declared — `GITHUB_TOKEN` is accessed implicitly via the built-in secret.

### Outputs

None — SARIF results are uploaded to the GitHub Security tab directly.
