# Angular Workflow Usage Guide

## What It Does

```
lint-and-typecheck ──► test-unit ──► test-e2e (if run-e2e: true)
                  └──► build
```

| Job | Tool | Purpose |
|-----|------|---------|
| `lint-and-typecheck` | ESLint + tsc + Stylelint | Catches style violations and type errors before tests |
| `test-unit` | Vitest | Fast component and service tests with coverage |
| `build` | ng build | Verifies the production build compiles cleanly |
| `test-e2e` | Playwright | Browser-level acceptance tests (conditional) |

Note: `build` runs in parallel with `test-unit` — both depend only on `lint-and-typecheck`.

## Minimal Caller

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/angular.yml@v1
    with:
      node-version: '22'
```

## All Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `node-version` | string | `'22'` | Node.js version (LTS recommended) |
| `working-directory` | string | `'.'` | Path to the Angular project root |
| `run-e2e` | boolean | `true` | Whether to run Playwright E2E tests |
| `package-manager` | string | `'npm'` | `npm` or `pnpm` |
| `base-href` | string | `'/'` | `--base-href` passed to `ng build` (for apps deployed at a sub-path) |

## Secrets

This workflow requires no secrets.

## Artifacts Produced

| Artifact | Job | Retention |
|----------|-----|-----------|
| `coverage-angular-unit` | `test-unit` | 7 days |
| `angular-dist` | `build` | 7 days |
| `playwright-report` | `test-e2e` (on failure) | 7 days |

The `playwright-report` artifact is only uploaded when the E2E job fails, giving you
the Playwright HTML report with screenshots and traces for debugging.

## Expected Project Structure

```
your-angular-project/
├── package.json
├── package-lock.json        ← or pnpm-lock.yaml for pnpm
├── tsconfig.app.json        ← required for tsc --noEmit
├── angular.json
└── src/
```

Your `package.json` should define these scripts:
- `lint` — `ng lint` or `eslint .`
- `test` — `vitest run --coverage` (or Vitest configured in `vitest.config.ts`)
- `stylelint` — optional; skip if not present (workflow detects config file automatically)

## Tips

### Skip E2E on Feature Branch PRs

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/angular.yml@v1
    with:
      run-e2e: ${{ github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop' }}
```

### Using pnpm

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/angular.yml@v1
    with:
      package-manager: 'pnpm'
```

The workflow installs pnpm via `pnpm/action-setup@v4` before Node setup when `package-manager: pnpm` is set.

### App Deployed at a Sub-path

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/angular.yml@v1
    with:
      base-href: '/my-app/'
```
