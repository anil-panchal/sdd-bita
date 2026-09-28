# CI/CD Security Reference

## Overview

CI/CD systems execute trusted credentials against repository content that may be attacker-controlled. Review workflow files, runner configuration, secret handling, artifact publishing, and deployment gates as part of the attack surface.

---

## Source and Change Controls

### Safe Patterns (Do Not Flag)

```yaml
# SAFE: CI runs on pull requests and protected branch pushes.
on:
  pull_request:
  push:
    branches: [main]

# SAFE: Least-privilege default token.
permissions:
  contents: read
```

### Check Context

```yaml
# CHECK: Browserless automation may be fine, but write scopes should be justified.
permissions:
  contents: write
  pull-requests: write

# CHECK: Production deploy triggered manually.
on:
  workflow_dispatch:
```

### Review Notes

- Protected `main`, required PR reviews, and status checks reduce the blast radius of pipeline changes.
- Manual production deployment is not enough by itself; also check environment protection, reviewer gates, and artifact provenance.

---

## Trust Boundary Violations

### Always Flag

```yaml
# Critical: Secrets are available to untrusted pull request code.
on:
  pull_request_target:

jobs:
  build:
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: ./scripts/build.sh
        env:
          PROD_TOKEN: ${{ secrets.PROD_TOKEN }}
```

```yaml
# Critical: Publish or deploy before tests and security gates.
jobs:
  release:
    steps:
      - run: docker push my-image:latest
```

### Review Notes

- `pull_request_target` is especially dangerous when paired with checkout of PR-controlled code, secret access, or write permissions.
- Untrusted PRs should never receive deployment credentials or long-lived publish tokens.

---

## Workflow and Action Integrity

### Safe Patterns (Do Not Flag)

```yaml
# SAFE: Internal reusable workflow.
jobs:
  security:
    uses: org/reusable-workflows/.github/workflows/security.yml@8f3c4f3d8d2e1f9b8c7a6e5d4c3b2a1f0e9d8c7b
```

### Flag These

```yaml
# Supply-chain risk: Floating tag from third party.
- uses: some-org/action@main   # FLAG
- uses: vendor/action@master   # FLAG
- uses: vendor/action@v1       # CHECK: Better pinned to commit SHA

# Remote script execution.
- run: curl -fsSL https://example.com/install.sh | bash  # FLAG
- run: wget -qO- https://example.com/bootstrap.sh | sh   # FLAG
```

### Review Notes

- Prefer trusted internal reusable workflows or pinned commit SHAs for third-party actions.
- Avoid downloading and executing remote scripts during pipeline runs unless integrity is verified.

---

## Secrets and Credentials

### Safe Patterns (Do Not Flag)

```yaml
# SAFE: OIDC / short-lived federation preferred over static cloud keys.
permissions:
  id-token: write
  contents: read
```

```yaml
# SAFE: Secret provided by platform secret store.
env:
  API_TOKEN: ${{ secrets.API_TOKEN }}
```

### Flag These

```yaml
# Hardcoded credential.
env:
  AWS_SECRET_ACCESS_KEY: "plain-text-secret"  # FLAG

# Password on command line.
- run: docker login -u "$USER" -p "$PASSWORD" registry.example.com  # FLAG

# Secret leakage in logs.
- run: |
    set -x
    echo "$PROD_TOKEN"  # FLAG
```

### Review Notes

- Static credentials in repository files are high-confidence findings.
- Even when secrets come from a secure store, logging or echoing them is still a vulnerability.
- Prefer OIDC, workload identity, or other short-lived credentials over access keys.

---

## Dependency and Security Scanning

### Safe Patterns (Do Not Flag)

```yaml
# SAFE: Build, test, and security scans before release.
jobs:
  build:
  test:
  sast:
  sca:
  secret-scan:
  release:
    needs: [build, test, sast, sca, secret-scan]
```

### Check Context

```yaml
# CHECK: Conditional IaC or container scanning depends on repo contents.
jobs:
  iac-scan:
  container-scan:
```

### Review Notes

- The CI/CD pipeline should validate code, dependencies, secrets, and IaC before publishing artifacts.
- Missing scanners are often medium-confidence governance gaps rather than direct exploitable bugs, but they still matter in pipeline reviews.

---

## SonarQube Enablement Check

Treat SonarQube as a firm-standard SAST control. When reviewing repository security posture, explicitly check whether SonarQube is enabled in the repo and not just whether scan results happen to exist today.

### What to Check

- Look for a Sonar workflow under `.github/workflows/`.
- For standard repositories, prefer a workflow that calls `McK-Internal/github-workflows/.github/workflows/reusable-scan-sonarqube.yaml@v4`.
- For monorepos, look for one or more manually maintained `sonar_<submodule>.yml` or `.yaml` workflows.
- Check for Sonar secrets or env wiring such as `SONAR_TOKEN` and `SONAR_HOST_URL`.
- If `sonar-project.properties` exists, verify `sonar.projectKey` is not stale or hardcoded from legacy onboarding, and `sonar.host.url` points to `https://sonarqube.mckinsey.com/`.
- If a workflow exists, check that it runs on both `pull_request` and `push` to protected branches so PR decoration and branch analysis both work.

### Safe Patterns (Do Not Flag)

```yaml
jobs:
  sonar_scan:
    name: SonarQube Scan
    uses: McK-Internal/github-workflows/.github/workflows/reusable-scan-sonarqube.yaml@v4
    with:
      project_key: 20355-McK-Internal_repo-name
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

### Check Context

```yaml
# Monorepo onboarding may require manual per-submodule workflows.
name: Sonar Scan

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  sonar-scan:
    env:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    steps:
      - run: |
          docker run ... sonar-scanner-cli \
            -Dsonar.projectKey=productID-org_repo-submodule
```

```properties
# Legacy config file: review carefully if present.
sonar.host.url=https://sonarqube.mckinsey.com/
# sonar.projectKey should not be left behind from older onboarding flows
```

### Report When Missing

- If there is no Sonar workflow and no other clear onboarding evidence, tell the user SonarQube does not appear to be enabled for the repo.
- Treat this as a required firm security-standard gap, not as an exploitable code vulnerability.
- Direct the user to the onboarding guide. The standard onboarding flow creates a PR that adds the Sonar workflow, and the repository team must review and merge that PR.
- For monorepos, tell the user that each onboarded submodule needs its own Sonar workflow file.
- If multiple Sonar configuration PRs exist, advise merging the most recent one and closing older ones to avoid conflicts.

### Review Notes

- Existing repositories migrating from on-prem SonarQube to SonarQube Cloud may still have stale `sonar-project.properties` values or older scan steps.
- Missing SonarQube often explains the absence of SAST checks or PR decorations; call it out explicitly instead of assuming scans are merely pending.

---

## Artifact Publishing and Provenance

### Flag These

```yaml
# Release from untrusted branch.
if: startsWith(github.ref, 'refs/heads/feature/')  # CHECK

# Mutable tag without traceability.
- run: docker push registry.example.com/app:latest  # CHECK

# Publishing unsigned artifacts with privileged credentials.
- run: npm publish  # CHECK if no preceding integrity or approval controls
```

### Safer Pattern

```yaml
jobs:
  release:
    needs: [test, sast, sca]
    if: github.ref == 'refs/heads/main'
    steps:
      - run: docker build -t registry.example.com/app:${{ github.sha }} .
      - run: docker push registry.example.com/app:${{ github.sha }}
```

### Review Notes

- Prefer immutable tags, traceability to commit SHA, and attestation / SBOM generation for releasable artifacts.
- Promotion should reuse a previously verified artifact rather than rebuilding for each environment.

---

## Runner and Execution Isolation

### Flag These

```yaml
# High risk: Public or untrusted PRs on privileged self-hosted runners.
runs-on: self-hosted  # CHECK / FLAG depending on trust boundary

# Dangerous docker socket exposure in build jobs.
- run: docker run -v /var/run/docker.sock:/var/run/docker.sock tool  # FLAG
```

### Check Context

```yaml
# CHECK: Broad workspace persistence can leak secrets or artifacts between jobs.
- uses: actions/cache@v4
  with:
    path: ~/.cache
```

### Review Notes

- Self-hosted runners need strong isolation, network segmentation, and cleanup.
- Untrusted code should not run on runners that also hold production credentials or long-lived workspace state.

---

## GitHub Actions-Specific Review Points

```yaml
# CHECK: Default checkout token persistence may be unnecessary for read-only jobs.
- uses: actions/checkout@v4
  with:
    persist-credentials: false

# CHECK: Excessive job permissions.
permissions: write-all

# CHECK: Environment approval should protect production.
environment:
  name: production
```

### Review Notes

- Minimize `GITHUB_TOKEN` scopes per job.
- Be careful with reusable workflows that inherit secrets broadly.
- `workflow_run`, `repository_dispatch`, and cross-workflow artifact passing can create hidden trust edges.

---

## rg Patterns

```bash
# Dangerous workflow triggers
rg -n 'pull_request_target|workflow_run|repository_dispatch' .github/workflows .gitlab-ci.yml Jenkinsfile azure-pipelines.yml

# Excessive permissions and self-hosted runners
rg -n 'permissions:\s*write-all|contents:\s*write|runs-on:\s*self-hosted' .github/workflows .gitlab-ci.yml Jenkinsfile azure-pipelines.yml

# Unpinned actions and remote script execution
rg -n 'uses: .+@(main|master|v[0-9]+)$|curl .*\| (bash|sh)|wget .*\| (bash|sh)' .github/workflows .gitlab-ci.yml Jenkinsfile azure-pipelines.yml

# Secrets in files or risky login patterns
rg -n 'SECRET|TOKEN|PASSWORD|docker login .* -p |set -x|echo .*\$\{\{ secrets\.' .github/workflows .gitlab-ci.yml Jenkinsfile azure-pipelines.yml

# SonarQube enablement
rg -n 'reusable-scan-sonarqube|SONAR_TOKEN|SONAR_HOST_URL|sonar\.projectKey|sonar[_-]' .github/workflows
rg -n 'sonar\.projectKey|sonar\.host\.url' sonar-project.properties

# Deploy or publish jobs
rg -n 'deploy|release|publish|docker push|npm publish|twine upload' .github/workflows .gitlab-ci.yml Jenkinsfile azure-pipelines.yml
```

---

## References

- Internal CI/CD patterns were adapted from `packages/module-skills/src/skills/cicd-pipeline-skill`
- [SonarQube - Get Onboarded](https://mckinsey.atlassian.net/wiki/spaces/EPT/pages/667091486/SonarQube+-+Get+Onboarded)
- [SonarQube Cloud](https://sonarqube.mckinsey.com/)
- [SLSA Security Controls](https://slsa.dev/spec/v1.0/controls)
- [GitHub Actions Security Hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
