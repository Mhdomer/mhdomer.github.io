---
layout: post
title: "Mother's Secret - Code Review, Route State Machine Exploitation, and Path Traversal in Node.js"
date: 2026-05-29T10:00:00+03:00
categories:
  - TryHackMe
  - DevSecOps
tags:
  - tryhackme
  - devsecops
  - mothers-secret
  - code-review
  - lfi
  - path-traversal
  - nodejs
  - express
author: muhammed
description: Comprehensive walkthrough of TryHackMe Mother's Secret covering static code analysis of Express.js routes, emergency command override bypasses, Science Officer role escalation, and path traversal to extract Order 937.
toc: true
pin: false
math: false
mermaid: true
image:
---

## Overview

[Mother's Secret](https://tryhackme.com/room/motherssecret) serves as the capstone challenge room of Section 3 (Security in the Pipeline) in TryHackMe's DevSecOps Learning Path. Operating aboard the commercial starship USCSS Nostromo, the ship's mainframe, **MU-TH-UR 6000 (Mother)**, manages critical systems under corporate directives from the Weyland-TryHackMe Corporation.

As a standard Crew Member, access is strictly locked down. By auditing the underlying Node.js and Express.js routing logic, security engineers can identify state machine discrepancies, parameter injection flaws, and path traversal vulnerabilities to bypass authentication gates, escalate privileges to Science Officer (Ash), and uncover classified company secrets.

---

## 1. Challenge Architecture & Attack Chain

The challenge exposes an Express.js web service running on port 80. By performing white-box code analysis on the provided source scripts (`yaml.js` and `nostromo.js`), we map the complete exploitation path:

```mermaid
flowchart TD
    subgraph Recon["1. Code Review & Recon"]
        A[Download Task Source Files] --> B[Inspect Express Routes: /yaml & /api/nostromo]
        B --> C[Identify Emergency Override: 100375]
    end

    subgraph StateConfusion["2. Privilege Escalation"]
        C --> D[POST /yaml with override code 100375]
        D --> E[Hit /api/nostromo in sequence]
        E --> F[Session Confused: Role elevated to Science Officer Ash]
        F --> G[Retrieve Route Flag: Flag{X3n0M0Rph}]
    end

    subgraph Exploitation["3. LFI & Order 937"]
        G --> H[Query Order 937: THM_FLAG{0RD3R_937}]
        H --> I[Exploit Path Traversal: /api/nostromo/mother/secret.txt]
        I --> J[Target File Location: /opt/m0th3r]
        J --> K[Final Flag: Flag{Ensure_return_of_organism_meow_meow!}]
    end
```

---

## 2. Static Code Analysis: Auditing Mother's Routing Logic

Reviewing the task archive exposes how Mother evaluates access levels and handles file loading.

### Emergency Command Override
The operating manual and routing definitions specify an emergency command override used for alien loaders and diagnostic procedures:

```text
100375
```

Submitting this override value to the `/yaml` loader enables processing of privileged diagnostic routines without standard crew credential verification.

### Route State Confusion: Becoming the Science Officer
The application evaluates role hierarchy based on the order of route requests. By interacting with the routes in the expected sequence:
1. Submitting the emergency override to the YAML parser.
2. Invoking the Nostromo routing handler at `/api/nostromo`.

The mainframe's internal state machine misinterprets the unauthenticated caller as the ship's synthetic Science Officer: **Ash**.

Inspecting the response from the `/api/nostromo` endpoint yields the hidden route flag:

```text
Flag{X3n0M0Rph}
```

---

## 3. Uncovering Special Order 937

Aboard the Nostromo, corporate priorities supersede crew survival. Accessing the classified archives with Science Officer privileges reveals the infamous **Special Order 937**:

```text
937
```

> *"Priority one: Ensure return of organism for analysis. All other considerations secondary. Crew expendable."*

Extracting the contents of the classified **Flag** box exposes the associated token:

```text
THM_FLAG{0RD3R_937}
```

---

## 4. Exploiting Path Traversal: Extracting Mother's Secret

The `/api/nostromo/mother` route accepts a `file_path` parameter to fetch operational logs. However, the route fails to sanitize directory traversal sequences (`../`), allowing Local File Inclusion (LFI).

### Locating Mother's Secret
Code review and system path analysis indicate that Mother's core operational directives are stored on the filesystem at:

```text
/opt/m0th3r
```

### Retrieving the Secret Directive
Crafting a path traversal payload against the file reader endpoint:

```bash
curl -X POST http://MACHINE_IP/api/nostromo/mother \
  -H "Content-Type: application/json" \
  -d '{"file_path": "../../../../opt/m0th3r/secret.txt"}'
```

The server processes the path traversal sequence, escapes the web root directory, and reads the protected file from disk, revealing the ultimate flag:

```text
Flag{Ensure_return_of_organism_meow_meow!}
```

---

## 5. Question, Answer, and Flag Summary

| Task # | Task Title | Question / Challenge | Answer / Flag |
|---|---|---|---|
| **Task 1** | Ready for Take Off | Let's go! | *No answer needed* |
| **Task 2** | Mother's Secrets! | What is the number of the emergency command override? | `100375` |
| **Task 2** | Mother's Secrets! | What is the special order number? | `937` |
| **Task 2** | Mother's Secrets! | What is the hidden flag in the Nostromo route? | `Flag{X3n0M0Rph}` |
| **Task 2** | Mother's Secrets! | What is the name of the Science Officer with permissions? | `Ash` |
| **Task 2** | Mother's Secrets! | What are the contents of the classified "Flag" box? | `THM_FLAG{0RD3R_937}` |
| **Task 2** | Mother's Secrets! | Where is Mother's secret? | `/opt/m0th3r` |
| **Task 2** | Mother's Secrets! | What is Mother's secret? | `Flag{Ensure_return_of_organism_meow_meow!}` |

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
