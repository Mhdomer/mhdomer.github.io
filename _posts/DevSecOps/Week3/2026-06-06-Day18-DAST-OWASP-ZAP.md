---
layout: post
title: "Day 18: DAST with OWASP ZAP - Dynamic Web Application Security Testing"
date: 2026-06-06 10:00:00 +0800
categories:
  - DevSecOps
  - Week3
tags:
  - DAST
  - OWASP
  - ZAP
  - AppSec
  - DevSecOps
author: muhammed
description: A full walkthrough of OWASP ZAP for dynamic application security testing - spidering, active scanning, API testing, and running ZAP in CI against a staging environment.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fthfvnext.bing.com%2Fth%2Fid%2FOIP.IDGJ2fpRVZM81yOvHbSJKgHaDi%3Fr%3D0%26cb%3Dthfvnextfalcon%26pid%3DApi&f=1&ipt=b205e0f12cdaed48939c6e59b56cedce38a403cd917133188eee3ec48b3c460b
---

## What is DAST?

Dynamic Application Security Testing (DAST) tests a **running application** by sending real HTTP requests and analyzing the responses - just like a real attacker would.

Unlike SAST (which reads code) and SCA (which reads dependency files), DAST actually interacts with your application at runtime.

**What DAST finds that SAST misses:**
- Runtime configuration errors (debug mode on in prod, verbose error messages)
- Auth bypass vulnerabilities
- Session management issues (fixation, insecure cookies)
- Server-side request forgery (SSRF)
- Misconfigurations in web server headers
- Race conditions
- Business logic flaws exposed through the API

**What DAST cannot find:**
- Code-level issues not reachable via HTTP
- Problems in code paths not triggered by the scanner
- Secrets in source code (use SAST/secret scanning for that)

---

## OWASP ZAP

OWASP ZAP (Zed Attack Proxy) is the most widely used open-source DAST tool. It works as an intercepting proxy + active scanner + automation platform.

**Scan modes:**
| Mode | Description |
|------|-------------|
| Baseline scan | Passive only - no active attacks, safe for production |
| Full scan | Active scanning - sends attack payloads, staging only |
| API scan | Scans OpenAPI/Swagger/GraphQL endpoints |

---

## Installation

```bash
# Download ZAP from https://www.zaproxy.org/download/
# Or via Docker (recommended for CI)
docker pull ghcr.io/zaproxy/zaproxy:stable

# Verify
docker run --rm ghcr.io/zaproxy/zaproxy:stable zap.sh -version
```

---

## Baseline Scan (Passive: Safe for Any Environment)

The baseline scan crawls the site and runs passive checks only - it reads responses but doesn't send attack payloads. Safe to run against production.

```bash
# Baseline scan against a URL
docker run --rm ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t https://your-staging-app.com \
  -r zap-report.html \
  -J zap-report.json
```


**What the baseline scan checks:**
- Missing security headers (`X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`, `Strict-Transport-Security`)
- Cookies without `HttpOnly` or `Secure` flags
- Information disclosure (server version in headers, stack traces in errors)
- Outdated or vulnerable JS libraries
- CORS misconfiguration

---

## Full Scan (Active: Staging Only)

The full scan adds active attack payloads - it will send SQLi, XSS, and other payloads to your app. **Only run this against staging or dedicated test environments.**

```bash
docker run --rm ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py \
  -t https://staging.yourapp.com \
  -r zap-full-report.html \
  -J zap-full-report.json \
  -m 10       # max crawl time in minutes
```


---

## API Scan

If your application exposes an API documented with OpenAPI/Swagger, ZAP can scan it directly:

```bash
# Scan using an OpenAPI spec
docker run --rm \
  -v $(pwd):/zap/wrk:rw \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-api-scan.py \
  -t /zap/wrk/openapi.yaml \
  -f openapi \
  -r api-scan-report.html
```


---

## Understanding ZAP Alerts

ZAP categorizes alerts by risk level:

| Risk | Description |
|------|-------------|
| High | Critical vulnerabilities - SQLi, XSS, SSRF, command injection |
| Medium | Significant weaknesses - missing security headers, CSRF, path traversal |
| Low | Minor issues - info disclosure, verbose error messages |
| Informational | Not vulnerabilities - just observations |


Each alert includes:
- **URL** - which endpoint triggered it
- **Parameter** - which parameter was vulnerable
- **Attack** - the payload ZAP used
- **Evidence** - what in the response confirmed the vulnerability
- **Solution** - how to fix it
- **CWE** - the relevant weakness category

---

## ZAP with DVWA (Intentionally Vulnerable App)

Practice against DVWA (Damn Vulnerable Web Application):

```bash
# Start DVWA
docker run -d -p 80:80 vulnerables/web-dvwa

# Run ZAP baseline against it
docker run --rm \
  --network host \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t http://localhost/dvwa \
  -r dvwa-report.html
```



---

## Security Headers: Common Findings

One of the most common ZAP findings is missing security headers. Here's what each header does and how to set it:

### Content-Security-Policy (CSP)
Prevents XSS by restricting script sources:
```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'
```

### Strict-Transport-Security (HSTS)
Forces HTTPS:
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### X-Content-Type-Options
Prevents MIME sniffing:
```
X-Content-Type-Options: nosniff
```

### X-Frame-Options
Prevents clickjacking:
```
X-Frame-Options: DENY
```

### Referrer-Policy
Controls referrer information leakage:
```
Referrer-Policy: strict-origin-when-cross-origin
```

**In Nginx:**
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Content-Security-Policy "default-src 'self'" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```


---

## ZAP Automation Framework

For complex CI scenarios, ZAP has an automation framework driven by a YAML config:

```yaml
# zap-automation.yaml
env:
  contexts:
    - name: app-context
      urls:
        - https://staging.yourapp.com
      authentication:
        method: form
        parameters:
          loginUrl: https://staging.yourapp.com/login
          usernameField: username
          passwordField: password
          username: testuser
          password: testpass

jobs:
  - type: spider
    parameters:
      maxDuration: 5

  - type: activeScan
    parameters:
      policy: Default Policy

  - type: report
    parameters:
      template: traditional-html
      reportDir: /zap/wrk/reports/
      reportFile: zap-report.html
```

```bash
docker run --rm \
  -v $(pwd):/zap/wrk:rw \
  ghcr.io/zaproxy/zaproxy:stable \
  zap.sh -cmd -autorun /zap/wrk/zap-automation.yaml
```

---

## Integrating ZAP into GitHub Actions

```yaml
# .github/workflows/dast.yml
name: DAST - ZAP Scan

on:
  push:
    branches: [main]

jobs:
  zap-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start staging app
        run: docker compose up -d --wait

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: http://localhost:3000
          rules_file_name: .zap/rules.tsv
          cmd_options: -J zap-report.json

      - name: Upload ZAP report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-report
          path: zap-report.html
```


**ZAP rules file** - control which alerts fail the build:

```tsv
# .zap/rules.tsv
# Rule ID  Action  Description
10020      FAIL    X-Frame-Options header missing
10021      FAIL    X-Content-Type-Options missing
10038      WARN    Content Security Policy missing
10098      IGNORE  Cross-Domain Misconfiguration (low risk env)
```

- `FAIL` - the step fails (non-zero exit code)
- `WARN` - logged but doesn't fail the build
- `IGNORE` - not reported

---

## Lab: Baseline Scan Against DVWA or Juice Shop

**Objective:** Run a ZAP baseline scan against a local vulnerable app and review the report.

1. Start Juice Shop locally:
```bash
docker run -d -p 3000:3000 bkimminich/juice-shop
```

2. Run ZAP baseline:
```bash
docker run --rm \
  --network host \
  -v $(pwd)/reports:/zap/wrk:rw \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  -t http://localhost:3000 \
  -r /zap/wrk/juice-shop-baseline.html \
  -J /zap/wrk/juice-shop-baseline.json
```


3. Open `reports/juice-shop-baseline.html` in a browser


4. Pick 2 Medium findings → look at what header or config is missing → implement the fix in Nginx or the app config

5. Re-run the scan → confirm the fixed alert is gone from the report


---

## Key Takeaways

- DAST tests the running app - it finds what SAST and SCA miss because it interacts with real responses
- Baseline scan is safe for any environment - no attack payloads, just passive observation
- Full/active scan only against staging - never production
- Security headers are the most common DAST finding and the easiest to fix - add them in Nginx/ALB config
- ZAP in CI as a baseline scan gives you continuous coverage on every deployment
- Use a rules file (`.tsv`) to decide what fails the build vs what's just a warning
- Combine SAST + SCA + IaC scan + DAST in sequence for a full pipeline security gate

---

## References

<div class="references">
<ul>
  <li><a href="https://www.zaproxy.org/docs/" target="_blank">OWASP ZAP Documentation</a></li>
  <li><a href="https://github.com/zaproxy/action-baseline" target="_blank">ZAP Baseline GitHub Action</a></li>
  <li><a href="https://securityheaders.com/" target="_blank">Security Headers Checker</a></li>
  <li><a href="https://owasp.org/www-project-juice-shop/" target="_blank">OWASP Juice Shop</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
