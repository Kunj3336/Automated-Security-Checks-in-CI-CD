# Security Integration & DevSecOps Implementation Guide

## 1. Secrets Scanning: Gitleaks vs. TruffleHog

| Feature / Criteria | Gitleaks | TruffleHog |
| :--- | :--- | :--- |
| **Primary Mechanism** | Regular expression (regex) rules + Shannon entropy analysis | Regex, entropy, and live credential verification against APIs |
| **Active Verification** | No (identifies matching patterns offline) | Yes (tests flagged credentials against target endpoints to check if active) |
| **Performance & Speed** | Extremely fast (lightweight Go binary; minimal overhead on PR builds) | Slower (live network verification adds execution latency) |
| **CI/CD Integration** | Simple native GitHub Action with configurable exit codes | Powerful enterprise integrations; requires higher network/runner permissions |
| **Recommendation** | **Best for fast, automated pull request gating** | **Best for deep organizational audits and historical sweeps** |

---

## 2. GitHub Actions Workflow Configuration

The production workflow file (`.github/workflows/security.yml`) configured to automate all four security layers on every push and pull request targeting the `master` branch:

```yaml
name: CI/CD Security Checks

on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - master

permissions:
  contents: read
  security-events: write

jobs:
  security:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Gitleaks Secret Scan
        uses: gitleaks/gitleaks-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Semgrep SAST Scan
        uses: semgrep/semgrep-action@v1
        with:
          config: auto

      - name: OSV Dependency Scan
        uses: google/osv-scanner-action/osv-scanner-action@v2.5.0
        with:
          scan-args: |-
            --lockfile=requirements.txt

      - name: Build Docker Image
        run: docker build -t devops-two-tier-app:${{ github.sha }} .

      - name: Trivy Container Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: devops-two-tier-app:${{ github.sha }}
          format: table
          severity: CRITICAL,HIGH
          exit-code: '0'
          ignore-unfixed: true
```
--- 
## 3. Failure Handling & Triage Procedures (SOP)

When an automated security check fails in CI/CD, developers must follow these standard operating procedures based on the specific failing layer:

- **Gitleaks (Exit Code != 0)**
1. Immediate Revocation: Assume any secret flagged in code is compromised. Revoke and rotate the exposed credential immediately in the external service.

2. Remove Token from Code: Move the sensitive value into environment variables (.env) or GitHub Repository Secrets.

3. Clean Git History: If the commit has already been pushed, remove the secret from git history using git-filter-repo or BFG Repo-Cleaner before reopening the PR.

- **Semgrep SAST (Exit Code != 0)**
1. Review Findings: Inspect the line number and rule ID flagged in the CI log summary.

2. Refactor Insecure Code: Remediate the antipattern (for example, setting debug=False or passing host bindings via configuration variables rather than hardcoding 0.0.0.0).

3. Documented False Positives: If a rule is confirmed to be an invalid positive by the security lead, bypass it with inline annotation: # nosemgrep: <rule-id>.

- **OSV-Scanner (Exit Code != 0)**
1. Identify Affected Dependency: Locate the vulnerable library and its affected version in requirements.txt or the CycloneDX SBOM.

2. Version Bump: Check the advisory database for the patched version, upgrade the dependency in the manifest, and reinstall in your local virtual environment.

3. Verify Compatibility: Run test suites locally (pytest) to confirm no breaking changes before committing the updated lockfile.

- **Trivy Container Scanning (Exit Code != 0)**
1. Base Image Updates: Upgrade the base Docker image tag in the Dockerfile to the latest stable or slim release (e.g., updating from older Debian releases to current minimal runtime images).

2. Package Updates: Introduce apt-get update && apt-get upgrade -y or package-specific update commands in intermediate Docker build stages.

3. Upstream Unfixed Gating: If a vulnerability has no upstream fix released (ignore-unfixed: true), document the risk and track it in the security register.

- **Build Gating Proof**
Demonstration of Trivy terminating the CI pipeline when exit-code: '1' is enforced on HIGH and CRITICAL findings:

## 4. Best Practices & Key Recommendations
- Shift Left Early: Implement pre-commit hooks locally for secret and syntax scanning so vulnerabilities never reach the shared repository.

- Granular Policy Gating: Use non-zero exit codes on actionable, high-confidence rules to prevent deployments while allowing low-severity findings to generate audit reports.

- Continuous Monitoring: Periodically scan long-running images and dependencies against updated vulnerability feeds to detect zero-day disclosures.