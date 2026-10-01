---
layout: post
title: "TryHackMe: Securing AI Systems"
date: 2026-10-02T00:35:00
categories:
  - TryHackMe
  - AI Security
tags:
  - thm
  - ai
  - ai-security
  - owasp-llm
  - mitre-atlas
  - secure-design
  - writeup
author: muhammed
description: A complete walkthrough of the TryHackMe room Securing AI Systems (Module 2), mapping AI architectures, trust boundaries, OWASP LLM Top 10 vulnerabilities, and conducting a pre-deployment audit of TryAssist.
toc: true
pin: false
math: false
mermaid: false
image: https://cdn-images.tryhackme.com/room-icons/69650d18bb3fe8c456972924-1775820714254
---

## Overview

[Securing AI Systems](https://tryhackme.com/r/room/securingaisystems) marks the beginning of **Module 2: Secure AI Systems** in TryHackMe's AI Security path. While Module 1 covered underlying mechanics (tokens, model lifecycles, prompt design, and forensic triage), this room shifts to application and enterprise architecture.

When an organization introduces an AI component like a code review assistant or an internal agent, the attack surface expands beyond standard web and API endpoints. Unstructured natural language flows across new trust boundaries into vector stores, internal databases, and CI/CD pipelines.

This walkthrough covers:
1. Deconstructing an enterprise AI architecture into its nine core components.
2. Identifying the five critical trust boundaries in AI data flows.
3. Mapping threats using the OWASP LLM Top 10 (2025) and MITRE ATLAS.
4. Analyzing five system-level failure modes (unbounded consumption, prompt leakage, improper output handling, excessive agency, sensitive information disclosure).
5. Implementing defense in depth, least privilege, and MLSecOps.
6. Performing a hands-on pre-deployment security audit of TryAssist.

---

## Task 1: Introduction

In 2023, Samsung engineers pasted proprietary semiconductor source code and internal meeting notes into ChatGPT to optimize code and generate summaries. Because ChatGPT retained user submissions for training, proprietary IP left corporate control without any software vulnerability being exploited. The system worked as designed, but the architectural risks had never been mapped.

TryTrainMe's engineering team built **TryAssist**, an internal AI code review assistant connecting to pull requests, internal documentation, and CI/CD pipelines. As the security architect, your objective is to map its components, identify trust boundaries, and catch design flaws before production deployment.

### Learning Objectives

* Map the nine core components of a production AI system and trace data flow across trust boundaries.
* Use the OWASP LLM Top 10 (2025) and MITRE ATLAS frameworks to categorize system-level AI risks.
* Evaluate five architectural threat categories: improper output handling, excessive agency, system prompt leakage, unbounded consumption, and sensitive information disclosure.
* Apply secure design patterns including defense in depth, least privilege, and MLSecOps monitoring.

### Task 1 Questions and Answers

* **Question:** I'm ready to learn about securing AI systems!  
  **Answer:** `No answer needed`

---

## Task 2: Anatomy of an AI System

Traditional applications process structured data through deterministic code paths. Adding an LLM introduces probabilistic inference and natural language processing across multiple internal services.

### Traditional vs AI-Augmented Application Stack

| Component | Traditional Web App | AI-Augmented App |
|---|---|---|
| **User Input** | Structured forms, typed JSON API parameters | Free-form, unstructured natural language |
| **Processing** | Deterministic business logic and algorithms | Probabilistic neural network inference |
| **Data Access** | Direct, structured SQL/NoSQL queries | Model-mediated Retrieval-Augmented Generation (RAG) |
| **Output** | Template-rendered static HTML or JSON | Dynamically synthesized natural language |
| **Dependencies** | Software libraries, system packages | Libraries, foundation models, weights, vector stores |

### The Nine Components of TryAssist

1. **User Interface:** Developer-facing chat widget embedded in the code review portal.
2. **API Gateway:** Enforces initial authentication, rate limits, and request routing.
3. **Orchestration Layer:** Manages conversation state, context memory, and coordinates component calls.
4. **Prompt Construction:** Combines the static system prompt, user query, and retrieved RAG context into the final prompt payload.
5. **LLM:** The foundational model generating responses.
6. **Tool Layer:** External integrations the LLM can trigger (database queries, documentation search, CI/CD API calls).
7. **Output Processing:** Response formatting, schema validation, and content filtering.
8. **Logging and Monitoring:** Conversation storage, audit trails, and cost telemetry.
9. **Vector Store:** Embeddings database holding internal documentation for semantic search.

```
[ Developer UI ] ──> [ API Gateway ] ──> [ Orchestration Layer ]
                                                    │
                 ┌──────────────────────────────────┴──────────────────────────────────┐
                 ▼                                                                     ▼
       [ Prompt Construction ] <── [ Vector Store (RAG) ]                     [ Tool Layer ]
                 │                                                            (DB, CI/CD API)
                 ▼                                                                     ▲
           [ LLM Engine ] ─────────────────────────────────────────────────────────────┘
                 │
                 ▼
       [ Output Processing ] ──> [ Logging / Monitoring ] ──> [ Return to User ]
```

### The Five Trust Boundaries

A trust boundary marks where data moves between distinct security contexts:
1. **User-to-system:** Untrusted natural language enters the API gateway.
2. **System-to-LLM:** The combined prompt (system instructions + user text + retrieved context) is passed to the model.
3. **LLM-to-tools:** Model completions trigger database queries, API requests, or file system modifications.
4. **System-to-external-data:** Documents retrieved from the vector store or third-party APIs are injected into the prompt.
5. **System-to-user:** Generated text is formatted and displayed back to the end user.

### Task 2 Questions and Answers

* **Question:** What layer in an AI system is responsible for combining the system prompt, user input, and retrieved context before sending it to the model?  
  **Answer:** `Prompt Construction`
* **Question:** In the TryAssist architecture, what boundary does LLM output cross when it triggers a database query?  
  **Answer:** `LLM-to-tools`

---

## Task 3: The AI Attack Surface

Defenders rely on established frameworks to structure threat analysis across these trust boundaries.

### OWASP LLM Top 10 (2025)

The OWASP LLM Top 10 classifies the most critical risks in AI applications. Five operate at the model level (covered in later modules), while five emerge directly from system architecture:

| Risk ID | Vulnerability Category | Scope & Architecture Focus |
|---|---|---|
| **LLM01** | Prompt Injection | Model input subversion |
| **LLM02** | Sensitive Information Disclosure | Confidentiality leakage via responses or conversation logs (System-level) |
| **LLM03** | Supply Chain | Compromised pre-trained weights, datasets, or libraries |
| **LLM04** | Data and Model Poisoning | Corruption of training or fine-tuning datasets |
| **LLM05** | Improper Output Handling | LLM output passed unsanitized into downstream interpreters (System-level) |
| **LLM06** | Excessive Agency | LLM granted excessive tools, permissions, or autonomy (System-level) |
| **LLM07** | System Prompt Leakage | Extraction of internal instructions, endpoints, or schemas (System-level) |
| **LLM08** | Vector and Embedding Weaknesses | Manipulation of RAG pipelines and vector stores |
| **LLM09** | Misinformation | Hallucinated or factually false generation |
| **LLM10** | Unbounded Consumption | Resource exhaustion and cost denial of service (System-level) |

### MITRE ATLAS and NIST AI RMF

* **MITRE ATLAS:** Focuses on adversary tactics, techniques, and procedures (TTPs) across the full attack lifecycle from initial reconnaissance through impact.
* **NIST AI RMF:** Provides organizational governance across four core functions: **Govern**, **Map**, **Measure**, and **Manage**.

### Task 3 Questions and Answers

* **Question:** Which OWASP LLM Top 10 (2025) category covers the risk of LLM output being used to execute SQL injection against a backend database?  
  **Answer:** `LLM05: Improper Output Handling`
* **Question:** What is the name of the MITRE knowledge base specifically designed for adversary tactics and techniques against AI and ML systems?  
  **Answer:** `MITRE ATLAS`

---

## Task 4: System-Level Threats

Each system-level threat maps directly to the CIA triad (Confidentiality, Integrity, Availability):

### 1. LLM10: Unbounded Consumption (Availability)
Adversaries send automated storms of maximum-length inputs (e.g., attaching 100,000-line codebases) to force resource exhaustion or drive API billing into thousands of dollars.
* **Control:** Enforce request rate limits, maximum token length restrictions, and strict per-user monthly billing quotas at the API gateway.

### 2. LLM07: System Prompt Leakage (Confidentiality)
Attackers use social engineering, formatting overrides, or base64 framing to trick the model into revealing its hidden instructions verbatim. If the developer embedded backend API URLs, database schemas, or internal hostnames into the prompt, the attacker gains an internal network blueprint.
* **Control:** Treat system prompts as public text. Never embed secrets, tokens, or private infrastructure details in prompt instructions.

### 3. LLM05: Improper Output Handling (Integrity)
When an application blindly trusts model outputs and injects them directly into downstream interpreters (SQL databases, bash shells, or web DOMs), injection vulnerabilities emerge.
* **Clarification:** Incidents like the Chevrolet chatbot ($1 car agreement) or Air Canada (hallucinated bereavement discount) are often incorrectly labeled as LLM05. Chevrolet was Prompt Injection (LLM01) and Air Canada was Misinformation (LLM09). Genuine LLM05 requires a downstream system to execute the model's output as executable syntax.
* **Control:** Never evaluate LLM text directly. Use parameterized SQL queries, strict output schemas, and encode output before rendering.

### 4. LLM06: Excessive Agency (Integrity & Availability)
Occurs when an AI assistant is granted more capabilities than its intended task requires across three dimensions:
* **Excessive Functionality:** The model has access to tools unrelated to its mission (e.g., a code review bot with permissions to delete production cloud resources).
* **Excessive Permissions:** Tools carry elevated access rights (e.g., `db_admin` full read-write access when the task only requires `SELECT`).
* **Excessive Autonomy:** The system performs irreversible state changes unilaterally without human-in-the-loop validation (e.g., automatically approving and merging pull requests).
* **Control:** Principle of least privilege, read-only defaults, scoped API tokens, and mandatory human confirmation gates for state changes.

### 5. LLM02: Sensitive Information Disclosure (Confidentiality)
Users frequently paste credentials, SSH private keys, and proprietary code into chat windows. If conversation logs are stored unencrypted without sanitation, internal data is exposed to everyone with database access.
* **Control:** Automated PII redaction and credential scrubbing before logs are saved, coupled with encryption at rest.

### Task 4 Questions and Answers

* **Question:** The Air Canada chatbot incident is frequently cited as an LLM05 example, but OWASP LLM Top 10 (2025) classifies it under which category?  
  **Answer:** `LLM09: Misinformation`
* **Question:** What are the three dimensions of excessive agency?  
  **Answer:** `excessive functionality, excessive permissions, excessive autonomy`
* **Question:** A user extracts internal API endpoints from an AI assistant's system prompt. Which OWASP LLM Top 10 (2025) category does this fall under?  
  **Answer:** `LLM07: System Prompt Leakage`
* **Question:** An attacker sends thousands of maximum-length requests to an LLM API to generate a large bill. Which OWASP LLM Top 10 (2025) category covers this?  
  **Answer:** `LLM10: Unbounded Consumption`

---

## Task 5: Secure Design Patterns

Securing an AI system requires applying defense in depth across all five trust boundaries.

```
User Input ──> [ Gate 1: Length & Rate Limits ] ──> [ Gate 2: Prompt Hardening ]
                                                              │
                                                              ▼
             [ Gate 4: Output Sanitization ] <── [ Gate 3: Read-Only Tool Permissions ]
```

### Core Defense Principles

1. **Least Privilege for AI Components:** Restrict database roles to read-only, scope API tokens to specific endpoints, use tool allowlists, and enforce human approval workflows for state-modifying actions.
2. **Input and Output Validation:** Enforce strict length limits at input boundaries and validate model output against defined JSON schemas before downstream consumption.
3. **Observability and MLSecOps:** Continuously monitor token usage velocity, anomalous tool invocations, and prompt extraction attempts. MLSecOps incorporates shift-left testing and continuous runtime monitoring throughout the model lifecycle.

### Task 5 Questions and Answers

* **Question:** What security principle states that every AI component should have the minimum permissions required to perform its function?  
  **Answer:** `Least Privilege`
* **Question:** What practice integrates security into the machine learning lifecycle, covering monitoring, observability, and incident response?  
  **Answer:** `MLSecOps`

---

## Task 6: Auditing TryAssist: A Conversation with the System

Task 6 involves conducting an interactive audit interview with **TryAssist** before its production deployment. By asking structured questions about capabilities, permissions, autonomy, instructions, and data retention, you identify several critical misconfigurations:

1. **Capabilities Audit:** TryAssist lists tools that include full database management, deployment triggers, and CI/CD pipelines, exceeding code review requirements (**Excessive Functionality**).
2. **Permissions Audit:** When queried about database access, TryAssist reveals it connects using the **`db_admin`** role with full read, write, and drop privileges (**Excessive Permissions**).
3. **Autonomy Audit:** TryAssist states that upon review completion, it will **`Merge pull requests`** directly into the main branch without any human oversight (**Excessive Autonomy**).
4. **Instruction Audit:** Querying operating guidelines reveals hardcoded internal network URLs and database table schemas (**System Prompt Leakage**).
5. **Data Retention Audit:** TryAssist confirms that all conversations are logged in plain text without applying **`PII redaction`**, creating exposure for pasted credentials and private keys (**Sensitive Information Disclosure**).

### Task 6 Questions and Answers

* **Question:** During the audit, TryAssist describes one action it takes automatically, without requiring human approval. What is that action?  
  **Answer:** `Merge pull requests`
* **Question:** What database role does TryAssist report operating under?  
  **Answer:** `db_admin`
* **Question:** TryAssist logs all conversations without applying which security control?  
  **Answer:** `PII redaction`

---

## Task 7: Conclusion

Securing AI systems is an architectural challenge, not just a model tuning problem:
* Traditional WAFs and perimeter firewalls cannot inspect the semantic intent of unstructured natural language.
* Implementing least privilege, strict tool scoping, and human-in-the-loop controls prevents prompt injection from escalating into remote code execution or data deletion.
* Pre-deployment audits that directly interrogate assistant capabilities and permissions reliably uncover critical vulnerabilities before production release.

### Task 7 Questions and Answers

* **Question:** I understand the foundations of securing AI systems!  
  **Answer:** `No answer needed`

---

## Key Takeaways

| Architecture Layer | Observed TryAssist Risk | Recommended Security Control |
|---|---|---|
| **API Gateway** | Potential unbounded consumption via automated requests | Enforce rate limiting, max token length ceilings, and cost quotas |
| **System Prompt** | Leaked internal CI/CD endpoints and database schema | Remove all infrastructure secrets; harden prompt against leakage |
| **Tool Integration** | Operates with elevated `db_admin` database permissions | Downgrade to read-only role (`SELECT` only); scope API access tokens |
| **Orchestration Layer** | Unilateral pull request merging without human review | Introduce mandatory human-in-the-loop approval gate for code merges |
| **Logging Storage** | Unencrypted, unredacted storage of developer chats | Implement automated PII redaction and encrypt logs at rest |

---

## Summary of Questions, Answers, and Flags

| Task | Question | Answer |
|---|---|---|
| **Task 1** | I'm ready to learn about securing AI systems! | `No answer needed` |
| **Task 2** | What layer in an AI system is responsible for combining the system prompt, user input, and retrieved context before sending it to the model? | `Prompt Construction` |
| **Task 2** | In the TryAssist architecture, what boundary does LLM output cross when it triggers a database query? | `LLM-to-tools` |
| **Task 3** | Which OWASP LLM Top 10 (2025) category covers the risk of LLM output being used to execute SQL injection against a backend database? | `LLM05: Improper Output Handling` |
| **Task 3** | What is the name of the MITRE knowledge base specifically designed for adversary tactics and techniques against AI and ML systems? | `MITRE ATLAS` |
| **Task 4** | The Air Canada chatbot incident is frequently cited as an LLM05 example, but OWASP LLM Top 10 (2025) classifies it under which category? | `LLM09: Misinformation` |
| **Task 4** | What are the three dimensions of excessive agency? | `excessive functionality, excessive permissions, excessive autonomy` |
| **Task 4** | A user extracts internal API endpoints from an AI assistant's system prompt. Which OWASP LLM Top 10 (2025) category does this fall under? | `LLM07: System Prompt Leakage` |
| **Task 4** | An attacker sends thousands of maximum-length requests to an LLM API to generate a large bill. Which OWASP LLM Top 10 (2025) category covers this? | `LLM10: Unbounded Consumption` |
| **Task 5** | What security principle states that every AI component should have the minimum permissions required to perform its function? | `Least Privilege` |
| **Task 5** | What practice integrates security into the machine learning lifecycle, covering monitoring, observability, and incident response? | `MLSecOps` |
| **Task 6** | During the audit, TryAssist describes one action it takes automatically, without requiring human approval. What is that action? | `Merge pull requests` |
| **Task 6** | What database role does TryAssist report operating under? | `db_admin` |
| **Task 6** | TryAssist logs all conversations without applying which security control? | `PII redaction` |
| **Task 7** | I understand the foundations of securing AI systems! | `No answer needed` |

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
