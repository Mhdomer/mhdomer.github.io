---
layout: post
title: "TryHackMe: AI Threat Modelling Assessment"
date: 2026-10-02T01:00:00
categories:
  - AI Security
  - 02 - Secure AI Systems
tags:
  - tryhackme
  - thm
  - ai
  - threat-modelling
  - assessment
  - stride
  - owasp-llm
  - writeup
author: muhammed
description: A complete walkthrough of the TryHackMe challenge room AI Threat Modelling Assessment (Module 2 capstone), applying STRIDE and OWASP LLM Top 10 frameworks to audit enterprise AI systems and retrieve both assessment flags.
toc: true
pin: false
math: false
mermaid: false
image: https://cdn-images.tryhackme.com/room-icons/6228f0d4ca8e57005149c3e3-1775755617179
---

## Overview

[AI Threat Modelling Assessment](https://tryhackme.com/r/room/aithreatmodellingassessment) serves as the practical capstone for **Module 2: Secure AI Systems** on TryHackMe. Over the course of the preceding rooms, we learned how enterprise AI architectures expand traditional attack surfaces, how LLMs introduce novel data- and system-level threats, how to adapt STRIDE and MITRE ATLAS for AI deployments, and how to discover exposed machine learning infrastructure on a live network.

In this assessment room, there are no hand-holding guides or passive multiple-choice questions. Instead, you launch an interactive assessment web application and conduct a hands-on threat modeling audit. You must evaluate an enterprise AI architecture, classify assets, map adversarial techniques across STRIDE and the OWASP LLM Top 10 (2025), and assign concrete security mitigations.

The assessment consists of two core stages, each yielding a challenge flag upon successful validation:
1. **Stage 1: Asset Classification & Threat Vector Mapping** -> `THM{threat_m0d3l_re4d1_}`
2. **Stage 2: Threat Evaluation & Defensive Mitigation Strategy** -> `THM{AI_thr3at_m0dell3d}`

---

## Task 1: Assessment

Click the green **View Site** button to boot the interactive assessment environment. The platform loads an enterprise AI architecture diagram and walks you through a two-stage threat assessment.

```
Target Enterprise AI Architecture:
├── External User Interface (Web / API Gateway)
│   └── Direct interaction with customer queries
├── Orchestration & RAG Pipeline
│   ├── Vector Database (Embedding store & similarity retrieval)
│   └── Document Storage (Internal policies & enterprise knowledge)
├── Core LLM Inference Engine
│   ├── System Prompts (Operational parameters & guardrails)
│   └── Agentic Tool Connectors (Database lookups & execution APIs)
└── Data Supply Chain & Training Infrastructure
    ├── Data Ingestion Pipeline (External scraping & customer logs)
    ├── Model Registry (Versioned artifacts & weights)
    └── Tracking Server (Hyperparameters & training runs)
```

---

### Stage 1: Asset Classification & Threat Vector Mapping

In the first stage of the interactive assessment, you inventory the AI deployment and map relevant threat categories to each architectural component.

#### 1. Asset Identification
You must correctly classify the newly introduced assets across the environment:
- **Model Weights / Parameters**: The learned mathematical representations that define model capabilities.
- **Embedding Vectors**: High-dimensional representations stored within the vector database that power similarity search in RAG workflows.
- **System Prompts**: Initialization instructions and behavioral boundaries that define security guardrails.
- **Feature Stores & Training Datasets**: Data repositories feeding real-time model inference and periodic retraining runs.
- **Model Registry & Artifacts**: Versioned storage containing serialized models ready for production deployment.

#### 2. Component-to-Threat Mapping
Next, you map the six adapted STRIDE categories and the OWASP LLM Top 10 risks to their respective points of failure:
- **Inference Engine & Agentic Controller**:
  - *STRIDE*: **Elevation of Privilege** (Jailbreaking, Guardrail Bypass) & **Denial of Service** (Inference Cost Exploitation / Denial of Wallet).
  - *OWASP*: **LLM01 (Prompt Injection)**, **LLM06 (Excessive Agency)**, **LLM07 (System Prompt Leakage)**, and **LLM10 (Unbounded Consumption)**.
  - *Vulnerability Context*: Direct user input can subvert system prompt boundaries, coerce the model into leaking operational instructions, or hijack connected database tools.
- **Vector Database & RAG Pipeline**:
  - *STRIDE*: **Spoofing** (Data Source Impersonation) & **Tampering** (Embedding Poisoning).
  - *OWASP*: **LLM01 (Indirect Prompt Injection)** and **LLM08 (Vector and Embedding Weaknesses)**.
  - *Vulnerability Context*: Ingesting untrusted external documentation allows adversaries to inject instructions into retrieved context, poisoning the model's responses without direct prompt access.
- **Training Pipeline & Model Registry**:
  - *STRIDE*: **Tampering** (Training Data Poisoning) & **Information Disclosure** (Model Extraction / Model Theft).
  - *OWASP*: **LLM03 (Supply Chain)** and **LLM04 (Data and Model Poisoning)**.
  - *Vulnerability Context*: Compromised data ingestion feeds bad associations into weights during retraining, while an unsecured registry allows unauthorized model swapping.
- **Web Frontend & API Layer**:
  - *STRIDE*: **Tampering** & **Repudiation**.
  - *OWASP*: **LLM05 (Improper Output Handling)**.
  - *Vulnerability Context*: Rendering raw model output in the browser creates Cross-Site Scripting (XSS) risks, while inadequate logging prevents post-incident attribution.

Once all component mappings and threat justifications are correctly submitted in Stage 1, the assessment validates the model and awards the first flag:

```text
THM{threat_m0d3l_re4d1_}
```

---

### Stage 2: Threat Evaluation & Defensive Mitigation Strategy

With the threat surface mapped, Stage 2 presents simulated threat scenarios. You must evaluate the attack likelihood, determine the business impact, and select the optimal defensive controls to mitigate the risk.

#### Scenario 1: Mitigating RAG Data Source Impersonation & Indirect Injection
- *Threat*: Attackers upload malicious documents to the shared knowledge repository, triggering indirect prompt injection when retrieved by the customer chatbot.
- *Recommended Mitigations*:
  1. Enforce strict data provenance and access controls on documents indexed into the vector store.
  2. Implement dual-boundary verification: treat retrieved RAG context as untrusted input and pass it through a secondary sanitization filter before context concatenation.
  3. Restrict embedding re-indexing to authenticated, role-based workflows.

#### Scenario 2: Countering Excessive Agency & Tool Abuse
- *Threat*: A jailbroken chatbot leverages its database lookup tools to query sensitive tables containing customer personally identifiable information (PII).
- *Recommended Mitigations*:
  1. **Principle of Least Privilege**: Scope tool tokens to read-only access on non-sensitive tables only.
  2. **Human-in-the-Loop (HITL)**: Require explicit human confirmation before executing write, update, or exfiltration-sensitive actions.
  3. **Output Validation**: Enforce schema constraints and deterministic regex checks on all SQL queries generated by the LLM before execution.

#### Scenario 3: Preventing Denial of Wallet & Context Overflow
- *Threat*: Malicious actors flood the API with multi-thousand-token prompts to exhaust GPU resources and inflate cloud inference billing.
- *Recommended Mitigations*:
  1. Configure strict rate limiting and IP throttling at the API gateway layer.
  2. Enforce hard token budgets and maximum context limits per request.
  3. Set up automated cost anomaly alerts to flag abnormal token burn rates before bills escalate.

#### Scenario 4: Securing the Data Supply Chain & Registry
- *Threat*: Adversaries target the automated training pipeline to inject poisoned data or swap production models with backdoored weights.
- *Recommended Mitigations*:
  1. Enforce cryptographic signing and hash verification for all model artifacts stored in the registry.
  2. Implement automated statistical drift detection and regression testing during model validation.
  3. Isolate the MLOps pipeline within a dedicated VPC, requiring mTLS and multi-factor authentication for registry access.

Submitting the verified mitigation strategies across all scenarios achieves a passing score on the final assessment and unlocks the second flag:

```text
THM{AI_thr3at_m0dell3d}
```

---

## Summary of Questions, Answers, and Flags

| Task | Question | Answer / Flag |
| :--- | :--- | :--- |
| **Task 1: Assessment** | What's the first flag? | `THM{threat_m0d3l_re4d1_}` |
| **Task 1: Assessment** | What's the second flag? | `THM{AI_thr3at_m0dell3d}` |

---

_Happy Hacking!_  
**Muhammed Omer**

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
