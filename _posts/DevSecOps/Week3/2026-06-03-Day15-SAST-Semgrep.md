---
layout: post
title: "Week 3 — Day 15: SAST with Semgrep"
date: 2026-06-03 10:00:00 +0800
categories:
  - DevSecOps
  - Week3
tags:
  - SAST
  - Semgrep
  - DevSecOps
  - CI/CD
  - AppSec
author: muhammed
description: A full walkthrough of Semgrep for static application security testing — running scans, writing custom rules, understanding findings, and integrating into GitHub Actions as a PR gate.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fsemgrep.dev%2Fassets%2Fcontent%2Fglobal%2Fblog-thumbnail-default.png&f=1&nofb=1&ipt=900ecf3502e536c0aa705294b7f3cd19bb74d747be36f6f6f6dbb52de1bc61a9
---

## What is SAST?

Static Application Security Testing (SAST) analyzes source code **without executing it** to find security vulnerabilities — SQL injection, hardcoded secrets, insecure deserialization, path traversal, and more.

It runs early in the pipeline — on every commit or PR — giving developers fast feedback before code reaches production.

**What SAST can catch:**
- SQL injection patterns
- XSS vulnerabilities
- Hardcoded credentials
- Insecure use of crypto functions
- Command injection
- Path traversal

**What SAST cannot catch:**
- Runtime behavior (use DAST for that)
- Business logic flaws
- Auth misconfigurations at the infrastructure level

---

## Why Semgrep?

Semgrep is a fast, open-source SAST tool that:
- Supports 30+ languages natively
- Has a massive community ruleset (2000+ rules)
- Lets you write custom rules in a readable YAML syntax
- Integrates natively with GitHub, GitLab, and CI/CD pipelines
- Has no server required — runs as a CLI

---

## Installation

```bash
# Windows (via pip)
pip install semgrep

# Or via winget
winget install Semgrep.Semgrep

# Verify
semgrep --version
```

---

## Running Your First Scan

```bash
# Scan current directory with the auto ruleset (Semgrep picks the right rules for your languages)
semgrep scan --config auto .

# Scan with OWASP Top 10 rules
semgrep scan --config "p/owasp-top-ten" .

# Scan with security-audit ruleset
semgrep scan --config "p/security-audit" .
```

> `[SCREENSHOT]` — *Terminal showing semgrep scan --config auto . running on a project, output showing findings with file path, line number, rule ID, severity, and the matched code snippet highlighted*

---

## Understanding the Output

Each finding shows:

```
/src/app/db.py
  vulnerability.sql-injection
  ❯❯❱ Line 42: query = "SELECT * FROM users WHERE id = " + user_input
        Found SQL injection: user input is directly concatenated into SQL query.
        This can allow an attacker to read or modify any data in the database.
        Severity: ERROR
        Fix: Use parameterized queries or prepared statements.
```

| Field | Meaning |
|-------|---------|
| File + line | Exactly where the issue is |
| Rule ID | Which rule caught it |
| Severity | ERROR / WARNING / INFO |
| Message | What the vulnerability is and why it matters |
| Fix | How to remediate |

> `[SCREENSHOT]` — *Semgrep output showing 3-4 findings from a vulnerable Python app with the matched code lines highlighted in red*

---

## Key Rulesets

Semgrep's community registry has pre-built rulesets for most use cases:

| Ruleset | Command | Use for |
|---------|---------|---------|
| Auto (language-appropriate) | `--config auto` | Default — good starting point |
| OWASP Top 10 | `--config p/owasp-top-ten` | Web app vulnerabilities |
| Security audit | `--config p/security-audit` | Broad security checks |
| Secrets | `--config p/secrets` | Hardcoded credentials |
| Python-specific | `--config p/python` | Django, Flask, SQLAlchemy issues |
| Node.js | `--config p/nodejs` | Express, npm security |
| Docker | `--config p/dockerfile` | Dockerfile misconfigs |
| CI/CD | `--config p/ci` | GitHub Actions, Jenkins issues |

```bash
# Run multiple rulesets at once
semgrep scan \
  --config p/owasp-top-ten \
  --config p/secrets \
  --config p/security-audit \
  .
```

> `[SCREENSHOT]` — *Terminal showing semgrep scan with multiple --config flags running and the summary at the end: "X findings across Y files"*

---

## Filtering Output

```bash
# Only show errors (not warnings or info)
semgrep scan --config auto --severity ERROR .

# Output as JSON for parsing
semgrep scan --config auto --json --output results.json .

# Output as SARIF for GitHub Code Scanning
semgrep scan --config auto --sarif --output results.sarif .

# Exclude directories
semgrep scan --config auto --exclude-dir node_modules --exclude-dir .venv .
```

---

## Writing Custom Rules

This is Semgrep's killer feature — custom rules in readable YAML that match code patterns in your specific codebase.

### Rule Structure

```yaml
rules:
  - id: my-rule-id
    patterns:
      - pattern: <code pattern to match>
    message: >
      Description of what the rule found and why it's a problem.
      How to fix it.
    languages: [python]
    severity: ERROR
    metadata:
      category: security
      cwe: "CWE-89: SQL Injection"
```

---

### Example 1 — Detect SQL Injection in Python

```yaml
rules:
  - id: sql-injection-string-format
    patterns:
      - pattern: |
          $QUERY = "..." % $INPUT
          $DB.execute($QUERY)
      - pattern: |
          $DB.execute("..." % $INPUT)
      - pattern: |
          $DB.execute(f"...{$INPUT}...")
    message: >
      SQL query built with string formatting — SQL injection risk.
      Use parameterized queries: cursor.execute("SELECT * FROM t WHERE id = %s", (user_id,))
    languages: [python]
    severity: ERROR
    metadata:
      cwe: "CWE-89"
      owasp: "A03:2021 - Injection"
```

### Example 2 — Detect Hardcoded AWS Keys

```yaml
rules:
  - id: hardcoded-aws-access-key
    patterns:
      - pattern-regex: 'AKIA[0-9A-Z]{16}'
    message: >
      Hardcoded AWS access key detected.
      Remove immediately and rotate the key in IAM.
      Use environment variables or Secrets Manager instead.
    languages: [generic]
    severity: ERROR
    metadata:
      cwe: "CWE-798"
```

### Example 3 — Detect Dangerous `eval()` in JavaScript

```yaml
rules:
  - id: dangerous-eval
    patterns:
      - pattern: eval($X)
      - pattern-not: eval("...")   # allow literal strings (low risk)
    message: >
      eval() called with a non-literal argument — potential code injection.
      Avoid eval() entirely; use JSON.parse() for data or refactor the logic.
    languages: [javascript, typescript]
    severity: WARNING
```

### Example 4 — Detect Insecure `subprocess` in Python

```yaml
rules:
  - id: subprocess-shell-true
    patterns:
      - pattern: subprocess.run($CMD, ..., shell=True, ...)
      - pattern: subprocess.call($CMD, ..., shell=True, ...)
      - pattern: subprocess.Popen($CMD, ..., shell=True, ...)
    message: >
      subprocess called with shell=True and a variable command — command injection risk.
      Use shell=False and pass a list of arguments instead.
    languages: [python]
    severity: ERROR
```

---

## Testing Custom Rules

Semgrep has a built-in test mechanism — write test cases alongside your rules:

```yaml
rules:
  - id: sql-injection-string-format
    # ... rule definition ...

# Test file: test_sql_injection.py
# ruleid: sql-injection-string-format
query = "SELECT * FROM users WHERE id = %s" % user_id
db.execute(query)

# ok: sql-injection-string-format
db.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

```bash
# Run tests
semgrep --test .
```

> `[SCREENSHOT]` — *Terminal showing semgrep --test output: "1 passed, 0 failed" confirming the rule correctly catches the bad pattern and ignores the good one*

---

## Ignoring False Positives

Suppress a specific finding inline:

```python
# nosemgrep: sql-injection-string-format
query = build_safe_query(user_id)   # this function sanitizes input
```

Suppress across a file — add to `.semgrepignore`:

```
# .semgrepignore
tests/
vendor/
*.min.js
migrations/
```

---

## Integrating into GitHub Actions

```yaml
# .github/workflows/semgrep.yml
name: Semgrep SAST

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  semgrep:
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
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: Upload SARIF to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep.sarif
```

> `[SCREENSHOT]` — *GitHub Actions run showing the Semgrep step — green if no errors found, or red with the violation listed in the step output*

> `[SCREENSHOT]` — *GitHub repository → Security tab → Code scanning alerts showing Semgrep findings with their severity, rule ID, and the file/line they were found in*

**`--error` flag:** makes Semgrep exit with code 1 when any finding at the specified severity is found — this blocks the PR from merging.

---

## Lab — Scan a Vulnerable App

**Objective:** Run Semgrep against an intentionally vulnerable application and review findings.

1. Clone a vulnerable app:
```bash
git clone https://github.com/juice-shop/juice-shop
cd juice-shop
```

2. Run the OWASP Top 10 scan:
```bash
semgrep scan --config p/owasp-top-ten --severity ERROR . 2>/dev/null
```

> `[SCREENSHOT]` — *Semgrep scan output on Juice Shop showing multiple findings — SQL injection, XSS, hardcoded secrets — with file paths and line numbers*

3. Run the secrets scan:
```bash
semgrep scan --config p/secrets . 2>/dev/null
```

> `[SCREENSHOT]` — *Semgrep secrets scan output showing any hardcoded credentials or API keys found in the codebase*

4. Pick one finding → look at the code → understand why it's flagged → write the fix

5. Write a custom rule for one pattern you notice in the code:
```bash
# Create my-rules.yaml with your rule
semgrep scan --config my-rules.yaml .
```

---

## Key Takeaways

- SAST runs on code — catch vulnerabilities at development time, not production
- `--config auto` is your fastest start — Semgrep selects the right rules for your languages
- Custom rules are Semgrep's superpower — model rules on patterns specific to your codebase
- Use `--sarif` + GitHub upload to get persistent findings in the Security tab
- `--error` flag in CI makes Semgrep a hard gate — PRs with critical findings can't merge
- `nosemgrep` inline comments for accepted false positives — don't suppress whole files

---

## References

<div class="references">
<ul>
  <li><a href="https://semgrep.dev/docs/" target="_blank">Semgrep Documentation</a></li>
  <li><a href="https://semgrep.dev/r" target="_blank">Semgrep Rules Registry</a></li>
  <li><a href="https://semgrep.dev/learn" target="_blank">Semgrep Rule Writing Tutorial</a></li>
  <li><a href="https://owasp.org/www-project-juice-shop/" target="_blank">OWASP Juice Shop (vulnerable app for practice)</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
