---
layout: post
title: "Week 3 — Day 19: Building a Full Secure CI/CD Pipeline"
date: 2026-06-07 10:00:00 +0800
categories:
  - DevSecOps
  - Week3
tags:
  - CI/CD
  - GitHubActions
  - DevSecOps
  - Pipeline
  - AppSec
author: muhammed
description: Assembling all Week 3 security tools into a single GitHub Actions pipeline — Semgrep, Snyk, Checkov, tfsec, Trivy, and ZAP running as sequential security gates on every PR and push.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fmiro.medium.com%2Fv2%2Fresize%3Afit%3A1358%2F1*KDUPWnNlsLP9VM-0_oIL3Q.png&f=1&nofb=1&ipt=bdae080767c2c4fbaba60976e9d54a6f61241fcafb53c3958de0f1085a6555c4
---

## The Goal

This day ties together everything from Week 3: SAST, SCA, IaC scanning, image scanning, and DAST into one cohesive pipeline. Each tool is a gate — findings at a certain severity block the merge.

**Pipeline stages in order:**

```
PR opened
    │
    ▼
1. SAST (Semgrep)         ← code analysis, runs on every file change
    │
    ▼
2. SCA (Snyk)             ← dependency vulnerabilities
    │
    ▼
3. IaC Scan (Checkov + tfsec) ← Terraform/K8s misconfigs
    │
    ▼
4. Build Docker image
    │
    ▼
5. Image Scan (Trivy)     ← OS + library CVEs in the built image
    │
    ▼
6. Deploy to staging
    │
    ▼
7. DAST (ZAP Baseline)    ← runtime security checks on staging
    │
    ▼
Merge allowed (all gates passed)
```

---

## Security Gates — What Blocks a Merge

| Stage | Blocks merge on | Warns on |
|-------|----------------|---------|
| Semgrep | ERROR severity findings | WARNING findings |
| Snyk | High + Critical CVEs | Medium CVEs |
| Checkov | High + Critical checks | Medium checks |
| tfsec | HIGH + CRITICAL | MEDIUM |
| Trivy | CRITICAL CVEs | HIGH CVEs |
| ZAP | FAIL rules (High alerts) | WARN rules |

**Philosophy:** Block on things that are definitely exploitable or a clear misconfiguration. Warn on everything else — don't make the pipeline so noisy it gets bypassed.

---

## Full Pipeline — GitHub Actions

```yaml
# .github/workflows/secure-pipeline.yml
name: Secure CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: myapp
  IMAGE_TAG: ${{ github.sha }}

jobs:
  # ─────────────────────────────────────
  # Stage 1: SAST
  # ─────────────────────────────────────
  sast:
    name: SAST — Semgrep
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        run: |
          semgrep scan \
            --config p/owasp-top-ten \
            --config p/secrets \
            --sarif \
            --output semgrep.sarif \
            --severity ERROR \
            --error \
            .

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep.sarif
          category: semgrep

  # ─────────────────────────────────────
  # Stage 2: SCA
  # ─────────────────────────────────────
  sca:
    name: SCA — Snyk
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: >-
            --severity-threshold=high
            --sarif-file-output=snyk.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: snyk.sarif
          category: snyk

  # ─────────────────────────────────────
  # Stage 3: IaC Scanning
  # ─────────────────────────────────────
  iac-scan:
    name: IaC — Checkov + tfsec
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform/
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false

      - name: Upload Checkov SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif
          category: checkov

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: terraform/
          minimum_severity: HIGH
          format: sarif
          sarif_file: tfsec.sarif

      - name: Upload tfsec SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: tfsec.sarif
          category: tfsec

  # ─────────────────────────────────────
  # Stage 4+5: Build + Image Scan
  # ─────────────────────────────────────
  build-and-scan:
    name: Build + Trivy Image Scan
    runs-on: ubuntu-latest
    needs: [sast, sca]     # only build if code gates passed
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} .

      - name: Run Trivy image scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload Trivy SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy.sarif
          category: trivy

      - name: Push image to ECR
        if: github.ref == 'refs/heads/main'
        run: |
          aws ecr get-login-password --region ap-southeast-1 | \
            docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker tag ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} \
            ${{ secrets.ECR_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          docker push ${{ secrets.ECR_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  # ─────────────────────────────────────
  # Stage 6+7: Deploy to staging + DAST
  # ─────────────────────────────────────
  dast:
    name: DAST — ZAP Baseline
    runs-on: ubuntu-latest
    needs: [build-and-scan]
    steps:
      - uses: actions/checkout@v4

      - name: Start staging environment
        run: docker compose -f docker-compose.staging.yml up -d --wait

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: http://localhost:3000
          rules_file_name: .zap/rules.tsv
          cmd_options: -J zap.json -r zap.html

      - name: Upload ZAP report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-report
          path: |
            zap.html
            zap.json

      - name: Teardown staging
        if: always()
        run: docker compose -f docker-compose.staging.yml down
```

> `[SCREENSHOT]` — *GitHub Actions showing the full pipeline with all 5 jobs (sast, sca, iac-scan, build-and-scan, dast) — all green on a clean run, with the dependency arrows showing the execution order*

> `[SCREENSHOT]` — *GitHub PR showing all 5 required status checks listed — Semgrep, Snyk, Checkov+tfsec, Trivy, ZAP — all with green checkmarks before merge is allowed*

---

## Branch Protection Rules

Set up branch protection to enforce the pipeline gates:

1. GitHub repo → Settings → Branches → Add rule for `main`
2. Enable: **Require status checks to pass before merging**
3. Add required checks:
   - `SAST — Semgrep`
   - `SCA — Snyk`
   - `IaC — Checkov + tfsec`
   - `Build + Trivy Image Scan`
   - `DAST — ZAP Baseline`
4. Enable: **Require branches to be up to date before merging**

> `[SCREENSHOT]` — *GitHub → Branch protection rules settings showing the 5 required status checks listed and "Require branches to be up to date" enabled*

Now a PR literally cannot be merged unless all security gates pass.

---

## Secrets Management in the Pipeline

Never hardcode credentials in workflow files. Use GitHub Actions secrets:

```yaml
env:
  SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
```

Set these in: repo → Settings → Secrets and variables → Actions → New repository secret

> `[SCREENSHOT]` — *GitHub → Settings → Secrets and variables → Actions showing the list of repository secrets (names visible, values masked) — SNYK_TOKEN, AWS_ACCESS_KEY_ID, ECR_REGISTRY listed*

**Better for AWS:** Use OIDC instead of long-lived access keys:

```yaml
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: ap-southeast-1
```

This generates temporary credentials via the GitHub OIDC provider — no static AWS keys stored anywhere.

---

## Pipeline Performance Optimization

Running all scans sequentially is slow. Parallelize where possible:

```yaml
# Run SAST, SCA, and IaC in parallel — they're independent
jobs:
  sast:
    ...
  sca:
    ...
  iac-scan:
    ...

  # Build only after code gates pass
  build-and-scan:
    needs: [sast, sca]
    ...

  # DAST only after image is built
  dast:
    needs: [build-and-scan]
    ...
```

> `[SCREENSHOT]` — *GitHub Actions run showing the parallel execution — sast, sca, and iac-scan running simultaneously, then build-and-scan starting once both complete, then dast last*

**Typical timing:**
- SAST + SCA + IaC in parallel: ~3-5 minutes
- Build + image scan: ~3-5 minutes
- DAST: ~2-3 minutes
- **Total: ~10-13 minutes** — fast enough for a development workflow

---

## Handling Pipeline Failures

### Failed SAST (Semgrep)

```bash
# See the exact finding locally before pushing
semgrep scan --config p/owasp-top-ten --severity ERROR .
# Fix the code
# Re-push
```

### Failed SCA (Snyk)

```bash
# See what needs fixing
snyk test
# Apply fixes
snyk fix
# Re-push
```

### Failed IaC (Checkov)

```bash
# See what's failing
checkov -d terraform/ --compact
# Fix the Terraform resource
# Re-push
```

### Failed Image Scan (Trivy)

```bash
# See the CVEs
trivy image --severity CRITICAL myapp:test
# Update the base image or vulnerable package
# Rebuild + re-push
```

### Failed DAST (ZAP)

- Read the ZAP report artifact from the Actions run
- Add the missing security headers or fix the identified vulnerability
- Re-push

---

## Pipeline as Code Checklist

Before merging your pipeline definition:

- [ ] All tools run with `--exit-code 1` or equivalent on findings above threshold
- [ ] SARIF output uploaded to GitHub Security tab for all tools
- [ ] No static AWS credentials — use OIDC or short-lived tokens
- [ ] Snyk token stored as a secret, not hardcoded
- [ ] Branch protection rules require all checks to pass
- [ ] IaC scan only triggers on `terraform/**` file changes (performance)
- [ ] DAST only runs against staging, never production
- [ ] Staging environment torn down after DAST completes
- [ ] Pipeline runs on PRs AND on push to main (in case someone pushes directly)

---

## Key Takeaways

- A secure pipeline is a series of gates — each one catches a different class of problem
- Run SAST, SCA, and IaC scans in parallel — they're independent and this halves your pipeline time
- Use `needs:` to enforce ordering — don't build if the code scan fails, don't DAST if the image is not built
- Branch protection rules are what actually enforce the gates — without them, developers can merge despite failures
- Use OIDC for AWS auth in GitHub Actions — eliminates long-lived access keys from your secrets store
- The pipeline is itself code — scan it with Semgrep (`p/ci` ruleset) for misconfigurations

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.github.com/en/actions" target="_blank">GitHub Actions Documentation</a></li>
  <li><a href="https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services" target="_blank">GitHub OIDC with AWS</a></li>
  <li><a href="https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches" target="_blank">GitHub Branch Protection Rules</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
