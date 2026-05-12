# Symfony Workflow Usage Guide

## What It Does

The Symfony workflow runs a complete CI pipeline in this order:

```
code-style ──┐
             ├──► tests-unit ──► tests-integration ──► tests-e2e
static-analysis ──┘                    │
                                       └──► mutation (if run-mutation: true)
```

| Job | Tool | Purpose |
|-----|------|---------|
| `code-style` | PHP CS Fixer + Rector | Enforces coding standards, proposes modernizations |
| `static-analysis` | PHPStan | Finds type errors and logic issues without running code |
| `tests-unit` | Pest | Fast isolated tests; produces code coverage report |
| `tests-integration` | Pest + PostgreSQL + Redis | Tests that involve the database or cache |
| `tests-e2e` | Behat + full service stack | Acceptance scenarios from a user's perspective |
| `mutation` | Infection | Verifies test quality by mutating source code |

## Minimal Caller

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/symfony.yml@v1
    with:
      php-version: '8.4'
```

## All Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `php-version` | string | `'8.4'` | PHP version installed via shivammathur/setup-php |
| `composer-flags` | string | `'--prefer-dist --no-progress'` | Extra flags for `composer install` |
| `phpstan-level` | number | `8` | PHPStan analysis level (0–9) |
| `run-mutation` | boolean | `false` | Run Infection mutation testing (recommended for main branch only) |
| `working-directory` | string | `'.'` | Path to the Symfony project root |
| `php-extensions` | string | `'intl, pdo_pgsql, redis, amqp'` | Extensions installed by setup-php |

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `COMPOSER_AUTH` | No | JSON string for authenticating to private Packagist or GitHub Packages |

## Service Containers Used

| Job | Containers |
|-----|-----------|
| `tests-integration` | PostgreSQL 17, Redis 7 |
| `tests-e2e` | PostgreSQL 17, Redis 7, RabbitMQ 3 |

Connection details (injected as env vars in the workflow):

- `DATABASE_URL=postgresql://test:test@127.0.0.1:5432/test_db`
- `REDIS_URL=redis://127.0.0.1:6379`
- `MESSENGER_TRANSPORT_DSN=amqp://guest:guest@127.0.0.1:5672/%2f/messages`

Your `.env.test` should have matching values, or override via Symfony's `APP_ENV=test`.

## Artifacts Produced

| Artifact | Job | Retention |
|----------|-----|-----------|
| `coverage-unit` | `tests-unit` | 7 days |
| `junit-unit` | `tests-unit` | 7 days |
| `junit-integration` | `tests-integration` | 7 days |

## Expected Project Structure

```
your-symfony-project/
├── composer.json
├── composer.lock           ← required for cache key
├── bin/
│   └── console
├── vendor/
│   └── bin/
│       ├── php-cs-fixer    ← require friendsofphp/php-cs-fixer
│       ├── rector          ← require rector/rector
│       ├── phpstan         ← require phpstan/phpstan
│       ├── pest            ← require pestphp/pest
│       └── behat           ← require behat/behat (for e2e job)
└── .php-cs-fixer.dist.php  ← PHP CS Fixer configuration
```

## Tips

### Skip E2E on Pull Requests

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/symfony.yml@v1
    with:
      php-version: '8.4'
      # run-e2e is not an input — to skip Behat, use workflow_call conditions at the
      # caller level or configure Behat suites in your behat.yml to skip gracefully
```

### Monorepo (Symfony in a Subdirectory)

```yaml
jobs:
  backend:
    uses: your-username/ci-workflows/.github/workflows/symfony.yml@v1
    with:
      php-version: '8.4'
      working-directory: 'backend'
```

### Run Mutation Testing on Main Branch Only

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/symfony.yml@v1
    with:
      run-mutation: ${{ github.ref == 'refs/heads/main' }}
```
