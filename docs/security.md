# Security Checks

Every workflow in this library includes security scanning. This document explains
what is checked, how it works, and what to do when a scan fails.

## What Runs

### 1. Gitleaks — Secret Detection

**What it does**: Scans the full git history of the repository for accidentally
committed secrets: API keys, passwords, tokens, private keys, connection strings.

**Why git history?** A secret committed in a past commit is still leaked, even if
you deleted it in a later commit. Gitleaks checks every commit, not just the
current state of the files.

**What it catches**:
- AWS access keys and secret keys
- GitHub personal access tokens and fine-grained tokens
- Generic API keys and bearer tokens
- Private SSH keys
- Database connection strings with embedded credentials
- `.env` file contents accidentally committed
- Slack webhooks, Stripe keys, Twilio auth tokens, and 200+ other patterns

**Configuration**: Gitleaks uses its built-in ruleset by default. To customize,
add a `.gitleaks.toml` to your repository root. Example to allowlist a false positive:

```toml
[allowlist]
  description = "Allowlisted files"
  paths = [
    '''^\.gitleaks\.toml$''',
    '''tests/fixtures/mock-credentials\.json$'''
  ]
```

**When it fails**: The build fails if any unallowlisted secret pattern matches.
Resolution: revoke the leaked credential immediately (do not just delete the file —
treat it as compromised), then use `git filter-repo` to scrub the history, then
add an allowlist entry if it was a false positive.

### 2. Trivy — Vulnerability Scanning

**What it does**: Scans the repository filesystem for known vulnerabilities in
open-source dependencies (not containers — that's a separate use case).

**Sources scanned**:
- `composer.lock` — PHP packages (CVE database)
- `package-lock.json` / `pnpm-lock.yaml` — npm packages
- `Gemfile.lock` — Ruby gems
- `go.sum` — Go modules
- `Pipfile.lock` / `poetry.lock` — Python packages
- `*.gradle.kts`, `pom.xml` — Java dependencies

**Severity levels**: `UNKNOWN`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.

The workflow defaults to failing on `CRITICAL,HIGH`. Change via the `trivy-severity`
input:

```yaml
security:
  uses: your-username/ci-workflows/.github/workflows/security-scan.yml@v1
  with:
    trivy-severity: 'CRITICAL'  # Only fail on critical
```

**SARIF upload**: Results are always uploaded to the GitHub Security tab
(`github/codeql-action/upload-sarif`) even when the scan finds nothing. This
keeps the Security tab populated with up-to-date results.

**`ignore-unfixed: true`**: The workflow ignores vulnerabilities that have no
available fix. This reduces noise from dependencies that haven't released a patch yet.
Remove this flag if you want full visibility.

**When it fails**: A vulnerability is found at or above the severity threshold.
Resolution options:
1. Update the dependency to a patched version
2. If no fix exists: add a `.trivyignore` file to suppress specific CVEs with justification
3. Lower the threshold (last resort — reduces protection)

Example `.trivyignore`:
```
# CVE-2023-12345 — affects only the CLI tool, not the library we use
CVE-2023-12345
```

## Security by Default, Not by Opt-in

Security scanning is always included when you call `security-scan.yml`. It is
intentionally not optional — the whole point of this library is that every project
inherits production-grade CI automatically. If you skip the security job, you are
opting out of protection; that decision should be explicit and visible in your
caller file.

## Permissions

The `security-scan.yml` workflow requests the minimum permissions:
- `contents: read` — to checkout the code
- `security-events: write` — to upload SARIF results to the Security tab

The Security tab is only visible to repository admins and collaborators with
"Security" permission. SARIF results are not public even on public repositories.

## Integration with Callers

Standard pattern — run security after the main CI job passes:

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/symfony.yml@v1

  security:
    uses: your-username/ci-workflows/.github/workflows/security-scan.yml@v1
    needs: [ci]  # Only scan after tests pass
```

You can also run security in parallel with CI if you prefer faster failure detection:

```yaml
jobs:
  ci:
    uses: your-username/ci-workflows/.github/workflows/symfony.yml@v1

  security:
    uses: your-username/ci-workflows/.github/workflows/security-scan.yml@v1
    # No 'needs:' — runs in parallel
```
