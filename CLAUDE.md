# ci-workflows — Contributor Guide for Claude Code

This repository is a library of reusable GitHub Actions workflows. Every workflow
file in `.github/workflows/` (except `_test.yml`) is designed to be called by other
repositories via `uses:`.

## The Contract Every Workflow Must Follow

### 1. `on: workflow_call:` only

All workflows in this library are reusable. The trigger is always `workflow_call`
(and nothing else). Callers wire their own `on: [push, pull_request]` triggers.

Exception: `_test.yml` uses `workflow_dispatch` + `push` for self-testing.

### 2. Inputs pattern

```yaml
on:
  workflow_call:
    inputs:
      php-version:
        description: 'PHP version to use'
        type: string
        default: '8.4'
      run-optional-step:
        description: 'Whether to run the optional step'
        type: boolean
        default: false
    secrets:
      MY_SECRET:
        required: false
```

- Only `string`, `number`, `boolean`, and `choice` types are allowed.
- Secrets must be explicitly re-declared even if the caller passes them. Optional
  secrets use `required: false`.
- Never hard-code values that reasonable callers would want to change: versions,
  test filters, coverage thresholds.
- Every input and secret must have a `description`.

### 3. Caching strategy

Cache keys follow the pattern:

```
${{ runner.os }}-<tool>-${{ hashFiles('<primary-lockfile>') }}
```

Always include a `restore-keys:` fallback (prefix without the hash) so partial
cache hits still save time. Never cache build output — only dependency downloads.

| Ecosystem     | Cache key file                  | Cache path(s)              |
|---------------|---------------------------------|----------------------------|
| PHP/Composer  | `**/composer.lock`              | `~/.composer/cache`        |
| Node/npm      | `**/package-lock.json`          | `~/.npm`                   |
| Node/pnpm     | `**/pnpm-lock.yaml`             | `~/.local/share/pnpm/store`|
| Java/Gradle   | `**/*.gradle.kts`, `**/*.gradle`| `~/.gradle/caches`, `~/.gradle/wrapper` |
| Playwright    | Playwright version string       | `~/.cache/ms-playwright`   |

### 4. Fail-fast job order

Jobs must be ordered cheapest → most expensive:

1. **Lint + static analysis** (parallel, no services, fast feedback)
2. **Unit tests** (needs: lint jobs)
3. **Integration tests** (needs: unit, may require service containers)
4. **E2E tests** (needs: integration, full service stack)
5. **Optional/slow jobs** (mutation testing, full security scans — conditional)

Jobs that can run in parallel (e.g. lint + type-check) must declare no `needs:`.
Jobs that are order-dependent must declare a `needs:` array.

### 5. Service containers

All service containers require health checks to prevent race conditions:

```yaml
services:
  postgres:
    image: postgres:17-alpine
    env:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

Use `postgres:17-alpine`, `redis:7-alpine`, `rabbitmq:3-alpine` as base images.

### 6. Permissions

Set `permissions:` at the job level with the minimum required:

| Operation                | Permission required                        |
|--------------------------|--------------------------------------------|
| Checkout (public repo)   | `contents: read`                           |
| Upload SARIF             | `security-events: write`                   |
| Push to GHCR             | `packages: write` + `contents: read`       |
| Upload artifact          | No special permission needed               |

Never use `permissions: write-all`. Callers from forks run with restricted tokens.

### 7. `working-directory` support

Set `working-directory` as a job-level default so all `run:` steps inherit it:

```yaml
jobs:
  my-job:
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
```

This does not apply to `uses:` steps — pass paths explicitly for those.

### 8. Action versions (pinned to current majors)

Always use the current major version of each action:

| Action                              | Version |
|-------------------------------------|---------|
| `actions/checkout`                  | `v4`    |
| `actions/setup-node`                | `v4`    |
| `actions/setup-java`                | `v4`    |
| `actions/cache`                     | `v4`    |
| `actions/upload-artifact`           | `v4`    |
| `actions/download-artifact`         | `v4`    |
| `shivammathur/setup-php`            | `v2`    |
| `gradle/actions/setup-gradle`       | `v4`    |
| `gradle/actions/wrapper-validation` | `v4`    |
| `docker/setup-buildx-action`        | `v3`    |
| `docker/login-action`               | `v3`    |
| `docker/build-push-action`          | `v6`    |
| `docker/metadata-action`            | `v5`    |
| `aquasecurity/trivy-action`         | `0.28.0`|
| `gitleaks/gitleaks-action`          | `v2`    |
| `github/codeql-action/upload-sarif` | `v3`    |
| `pnpm/action-setup`                 | `v4`    |

When adding a new action: verify the current major on the marketplace before using it.

## How to Add a New Workflow

1. Create `.github/workflows/<name>.yml` with `on: workflow_call:` as the only trigger.
2. Follow all eight contract rules above.
3. Add `docs/<name>.md` with the inputs table and a minimal caller example.
4. Add an example caller in `examples/<name>-caller.yml`.
5. Add the new workflow to `docs/inputs-reference.md`.
6. Run `_test.yml` locally with `act` to validate the YAML:
   ```bash
   act workflow_dispatch -W .github/workflows/_test.yml
   ```
7. Update `README.md` with a one-line entry in the "Available Workflows" table.

## Testing Locally with `act`

[`act`](https://github.com/nektos/act) runs GitHub Actions locally in Docker.

```bash
# Install (macOS/Linux)
brew install act
# or
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Lint all workflow files (the _test.yml job)
act workflow_dispatch -W .github/workflows/_test.yml

# Dry-run a specific workflow
act workflow_call -W .github/workflows/security-scan.yml --dry-run
```

Reusable `workflow_call` workflows cannot be triggered directly by `act` — test them
via a local caller file that references the local path.

## Versioning Policy

- Tags use semver: `v1.0.0`, `v1.1.0`, `v2.0.0`
- Major version tags (`v1`, `v2`) float to the latest minor/patch: callers using
  `@v1` get non-breaking updates automatically.
- Breaking input changes (rename, remove, type change) require a new major version.
- New optional inputs with defaults are non-breaking (minor version bump).
- Bug fixes and action version bumps are patch-level.

Production callers should pin to a tag (`@v1`) not `@main`.

## Repository Conventions

- Language: English
- Workflow file names: `kebab-case.yml`
- Job IDs: `kebab-case`
- Step names: sentence case, descriptive ("Run PHPStan static analysis")
- All YAML files: 2-space indentation
