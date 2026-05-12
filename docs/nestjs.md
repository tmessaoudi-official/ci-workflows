# NestJS Workflow Usage Guide

## What It Does

```
lint-and-typecheck ──► test-unit ──► test-e2e
                  └──► build
```

| Job | Tool | Purpose |
|-----|------|---------|
| `lint-and-typecheck` | ESLint + tsc | Catches code style violations and type errors |
| `test-unit` | Jest | Isolated unit tests for services, controllers, guards |
| `test-e2e` | Jest + PostgreSQL + Redis | API-level integration tests |
| `build` | nest build | Verifies the TypeScript compilation and module wiring |

`build` runs in parallel with `test-unit` — both depend only on `lint-and-typecheck`.

## Minimal Caller

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/nestjs.yml@v1
    with:
      package-manager: 'pnpm'
```

## All Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `node-version` | string | `'22'` | Node.js version |
| `package-manager` | string | `'pnpm'` | `npm` or `pnpm` |
| `working-directory` | string | `'.'` | Path to the NestJS project root |

## Secrets

This workflow requires no secrets.

## Service Containers Used

| Job | Containers |
|-----|-----------|
| `test-e2e` | PostgreSQL 17, Redis 7 |

Connection injected as env vars:
- `DATABASE_URL=postgresql://test:test@127.0.0.1:5432/test_db`
- `REDIS_URL=redis://127.0.0.1:6379`
- `NODE_ENV=test`

## Artifacts Produced

| Artifact | Job | Retention |
|----------|-----|-----------|
| `coverage-nestjs-unit` | `test-unit` | 7 days (skipped if no coverage output) |
| `nestjs-dist` | `build` | 7 days |

## Expected `package.json` Scripts

```json
{
  "scripts": {
    "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
    "build": "nest build",
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json"
  }
}
```

## Tips

### Using npm Instead of pnpm

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/nestjs.yml@v1
    with:
      package-manager: 'npm'
```

### TypeORM / Prisma Database Setup

For E2E tests, the workflow sets `DATABASE_URL` and `NODE_ENV=test`. Configure your
`DataSource` or `PrismaClient` to read from `process.env.DATABASE_URL`. The test
database is ephemeral — schema should be created via migrations or `synchronize: true`
in test mode.
