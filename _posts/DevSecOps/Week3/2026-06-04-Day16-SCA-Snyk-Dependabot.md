---
layout: post
title: "Week 3 — Day 16: SCA with Snyk & Dependabot"
date: 2026-06-04 10:00:00 +0800
categories:
  - DevSecOps
  - Week3
tags:
  - SCA
  - Snyk
  - Dependabot
  - DevSecOps
  - SupplyChain
author: muhammed
description: A full walkthrough of Software Composition Analysis using Snyk and GitHub Dependabot — tracking vulnerable dependencies, generating SBOMs, and automating dependency updates.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fgo.snyk.io%2Frs%2F677-THP-415%2Fimages%2Flogo-vertical-black.png&f=1&nofb=1&ipt=edf0ec8b56bce70e32ec1ec1f7985cc6b702342ce79f8302afd5dc886846e283
---

## What is SCA?

Software Composition Analysis (SCA) scans your **dependencies** — the third-party libraries and packages your application uses — for known vulnerabilities.

SAST looks at your code. SCA looks at everyone else's code you imported.

**Why it matters:** The majority of modern application code is open-source dependencies. Log4Shell, Spring4Shell, and most major supply chain attacks exploited vulnerable libraries — not custom code.

**What SCA scans:**
- `package.json` / `package-lock.json` (Node.js)
- `requirements.txt` / `Pipfile.lock` (Python)
- `go.sum` (Go)
- `pom.xml` / `build.gradle` (Java)
- `Gemfile.lock` (Ruby)
- `Cargo.lock` (Rust)
- Container image layers (OS packages + app deps)

---

## Snyk

### What Snyk Does

Snyk scans your dependencies, container images, IaC, and code for vulnerabilities. It provides:
- Vulnerability details with severity and CVSS scores
- Direct fix suggestions (upgrade to version X)
- Auto-fix PRs
- License compliance checking

### Installation

```bash
# Via npm
npm install -g snyk

# Authenticate (creates a free account)
snyk auth

# Verify
snyk --version
```

> `[SCREENSHOT]` — *Terminal showing snyk auth opening a browser for login, then returning "Your account has been authenticated" in the terminal*

---

### Scanning a Project

```bash
# Scan dependencies in current directory
snyk test

# Scan and show full details
snyk test --all-projects

# Scan a specific manifest file
snyk test --file=requirements.txt

# Scan a Docker image
snyk container test nginx:latest

# Scan IaC files
snyk iac test ./terraform/
```

> `[SCREENSHOT]` — *Terminal showing snyk test output on a Node.js project — a table listing vulnerable packages with CVE IDs, severity, current version, and the fix version available*

---

### Understanding Snyk Output

```
✗ High severity vulnerability found in lodash
  Description: Prototype Pollution
  Info: https://snyk.io/vuln/SNYK-JS-LODASH-567746
  Introduced through: express@4.17.1 > lodash@4.17.15
  From: express@4.17.1 > body-parser@1.19.0 > lodash@4.17.15
  Fixed in: lodash@4.17.21
  Remediation:
    Upgrade express to version 4.18.2 or later
```

Key fields:
- **Introduced through** — which of YOUR direct dependencies pulled in the vulnerable one
- **From** — the full dependency chain (transitive path)
- **Fixed in** — the version that patches this CVE
- **Remediation** — which direct dep to upgrade

> `[SCREENSHOT]` — *Snyk test output showing the dependency chain clearly — a transitive vulnerability 3 levels deep, with the recommended direct dependency upgrade*

---

### Snyk Fix

```bash
# Apply all available fixes automatically
snyk fix

# Fix in dry-run mode (see what would change)
snyk fix --dry-run
```

Snyk updates `package.json` / `requirements.txt` with the fixed versions and shows a diff.

> `[SCREENSHOT]` — *Terminal showing snyk fix output listing the packages it's updating with old version → new version, and the number of vulnerabilities fixed*

---

### Snyk Monitor

`snyk monitor` uploads a snapshot of your dependencies to the Snyk dashboard for continuous monitoring — even after the scan runs, Snyk alerts you when new CVEs are published for your dependencies.

```bash
snyk monitor --project-name=myapp-production
```

> `[SCREENSHOT]` — *Snyk web dashboard showing the monitored project with vulnerability counts by severity, and a timeline showing when new vulnerabilities were discovered*

---

### Snyk in GitHub Actions

```yaml
# .github/workflows/snyk.yml
name: Snyk SCA Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif

      - name: Upload SARIF to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: snyk.sarif
```

> `[SCREENSHOT]` — *GitHub Actions run showing the Snyk step completing — either passing with "No high severity vulnerabilities" or failing with the vulnerable packages listed*

---

## GitHub Dependabot

Dependabot is GitHub's built-in dependency update tool. It:
- Automatically opens PRs to update vulnerable dependencies
- Scans your lock files for known CVEs
- Can be scheduled to open routine update PRs (not just security)

### Enabling Dependabot Security Alerts

1. GitHub repo → Settings → Security → Dependabot alerts → Enable
2. Also enable: Dependabot security updates (auto-PRs for security fixes)

> `[SCREENSHOT]` — *GitHub repo → Security tab → Dependabot alerts showing a list of vulnerable dependencies with severity badges, CVE IDs, and "Review security update" buttons*

### Dependabot Configuration File

```yaml
# .github/dependabot.yml
version: 2
updates:
  # npm dependencies
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
      day: monday
      time: "09:00"
    open-pull-requests-limit: 10
    labels:
      - dependencies
      - security

  # Python dependencies
  - package-ecosystem: pip
    directory: /
    schedule:
      interval: weekly

  # Docker base image
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly

  # GitHub Actions
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

> `[SCREENSHOT]` — *GitHub repo → Pull requests tab showing several Dependabot PRs open — each updating a specific package with the CVE it fixes listed in the PR description*

### Dependabot PR Structure

Each Dependabot PR includes:
- Package name and version bump
- Release notes from the package
- Compatibility score (based on test results from similar repos)
- CVEs fixed

> `[SCREENSHOT]` — *A Dependabot PR open in GitHub showing the package version bump (e.g., lodash 4.17.15 → 4.17.21), the CVE description, and the compatibility score*

**Workflow:** Review the PR → check the diff → merge if tests pass. Dependabot handles the boring part; you just approve.

---

## SBOM — Software Bill of Materials

An SBOM is a complete inventory of all components in your application. Think of it as a manifest of every library, version, and license your software includes.

**Why SBOMs matter:**
- Quickly identify if you're affected when a new CVE drops (Log4Shell scenario)
- Compliance requirements (US Executive Order on software security mandates SBOMs)
- License auditing — ensure no GPL libraries in a proprietary product

### Generating an SBOM with Snyk

```bash
# Generate SBOM in CycloneDX format
snyk sbom --format cyclonedx1.4+json --file package.json > sbom.json

# Generate SBOM in SPDX format
snyk sbom --format spdx2.3+json --file package.json > sbom.spdx.json
```

> `[SCREENSHOT]` — *Terminal showing snyk sbom command completing and the output JSON file containing the full component inventory with package names, versions, and license identifiers*

### Generating an SBOM with Syft

Syft is a dedicated SBOM generator by Anchore:

```bash
# Install
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh

# Generate SBOM for a Docker image
syft nginx:latest -o cyclonedx-json > nginx-sbom.json

# Generate SBOM for a directory
syft dir:. -o spdx-json > app-sbom.json
```

Then scan the SBOM for vulnerabilities with Grype:
```bash
grype sbom:nginx-sbom.json
```

> `[SCREENSHOT]` — *Terminal showing syft scanning a Docker image and outputting the SBOM, followed by grype scanning the SBOM and listing vulnerabilities found in the components*

---

## SCA vs SAST — Side by Side

| | SAST (Semgrep) | SCA (Snyk/Dependabot) |
|--|---------------|----------------------|
| What it scans | Your code | Third-party dependencies |
| Finds | Logic/security flaws in your code | Known CVEs in libraries |
| When to run | Every commit/PR | Every commit/PR + continuous monitoring |
| Fix requires | Changing your code | Upgrading a dependency |
| False positive rate | Medium | Low (CVE database is authoritative) |
| Speed | Fast | Fast |

Run both — they find completely different problems.

---

## Lab — Run Snyk on a Node.js Project

**Objective:** Scan a vulnerable project, understand the dependency chain, and fix findings.

1. Clone a project with known vulnerable dependencies:
```bash
git clone https://github.com/vulhub/vulhub
cd vulhub/node/express-fileupload
```

2. Run Snyk:
```bash
npm install
snyk test
```

> `[SCREENSHOT]` — *snyk test output showing vulnerabilities found in the project's dependencies with their CVE IDs and severity*

3. Pick a High severity finding — read the description and dependency chain
4. Run fix:
```bash
snyk fix --dry-run
```

> `[SCREENSHOT]` — *snyk fix --dry-run output showing which packages would be upgraded and which vulnerabilities each upgrade fixes*

5. Apply the fix and re-run `snyk test` to verify the count dropped

6. Enable Dependabot on a GitHub repo:
   - Push the project to a GitHub repo
   - Add `.github/dependabot.yml` with the npm config
   - Go to Security tab — check if Dependabot alerts are generated

> `[SCREENSHOT]` — *GitHub Security tab → Dependabot alerts showing the same vulnerabilities found by snyk, confirming both tools catch the same issues*

---

## Key Takeaways

- Your dependencies are your attack surface too — most real-world breaches exploit known CVEs in libraries, not zero-days in custom code
- Use Snyk for on-demand CLI scanning and CI/CD integration; Dependabot for continuous automated PRs
- Transitive vulnerabilities (3+ levels deep) are just as dangerous — Snyk shows the full chain and which direct dep to upgrade
- `snyk monitor` gives you continuous alerting even after the pipeline runs
- SBOMs are becoming a compliance requirement — generate them and store them with your releases
- SCA + SAST together cover both your code and your dependencies — run both in every pipeline

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.snyk.io/" target="_blank">Snyk Documentation</a></li>
  <li><a href="https://docs.github.com/en/code-security/dependabot" target="_blank">GitHub Dependabot Documentation</a></li>
  <li><a href="https://github.com/anchore/syft" target="_blank">Syft SBOM Generator</a></li>
  <li><a href="https://cyclonedx.org/" target="_blank">CycloneDX SBOM Standard</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
