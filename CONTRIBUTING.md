# Contributing to ci-workflows

## What this repo is

A library of reusable GitHub Actions workflows. Every workflow in `.github/workflows/` (except `_test.yml`) is callable via `uses:`.

## Adding a new workflow

1. Create `.github/workflows/<name>.yml` with `on: workflow_call:` as the only trigger
2. Follow the 8 contract rules in `CLAUDE.md`:
   - Lint → unit → integration → E2E job order (cheapest first)
   - All service containers must have `--health-cmd` options
   - Cache keys: `${{ runner.os }}-<tool>-${{ hashFiles('<lockfile>') }}`
   - `permissions:` at job level with minimum required
3. Write `docs/<name>.md` with the inputs table and a minimal caller example
4. Add an example in `examples/<name>-caller.yml`
5. Update `docs/inputs-reference.md`
6. Validate locally with `act`:
   ```bash
   act workflow_dispatch -W .github/workflows/_test.yml
   ```
7. Add a one-line entry to the "Available Workflows" table in `README.md`

## Versioning

- Breaking changes (renamed/removed inputs) → new major version (`v2.0.0`)
- New optional inputs with defaults → minor bump (`v1.1.0`)
- Bug fixes → patch bump (`v1.0.1`)

## Pull requests

1. Fork and create a branch (`feat/add-<name>-workflow`)
2. Validate with `act` before opening the PR
3. Include input/output documentation in the PR description
