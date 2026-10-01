---
layout: post
title: "TryHackMe: AI Threat Modelling"
date: 2026-10-02T00:50:00
categories:
  - TryHackMe
  - AI Security
tags:
  - thm
  - ai
  - threat-modelling
  - stride
  - mitre-atlas
  - owasp-llm
  - writeup
author: muhammed
description: A complete walkthrough of the TryHackMe room AI Threat Modelling (Module 2), adapting STRIDE for AI systems, mapping MITRE ATLAS techniques, applying the OWASP LLM Top 10 to enterprise architecture, and completing MegaCorp's threat model assessment.
toc: true
pin: false
math: false
mermaid: false
image: https://cdn-images.tryhackme.com/room-icons/6228f0d4ca8e57005149c3e3-1775757065770
---

## Overview

[AI Threat Modelling](https://tryhackme.com/r/room/aithreatmodelling) is the third room in **Module 2: Secure AI Systems** on TryHackMe. As enterprises integrate language models, recommendation engines, and automated fraud systems into production, security teams face attack surfaces that traditional threat modeling frameworks were not originally designed to assess.

Traditional threat modeling frameworks such as STRIDE have protected software architectures for decades. However, AI deployments introduce non-deterministic behaviors, black-box failure modes, neural weight theft, and multi-stage data supply chains.

This room guides defenders through an end-to-end AI threat assessment for **MegaCorp**, an enterprise deploying:
1. A customer-facing RAG-enabled chatbot connected to internal knowledge bases.
2. An internal recommendation engine personalizing product offerings using sensitive customer data.
3. An automated real-time fraud detection pipeline.

We evaluate how to adapt STRIDE, enrich findings with **MITRE ATLAS**, apply the **OWASP LLM Top 10 (2025)** to specific architecture components, and complete an interactive threat modeling assessment.

---

## Task 1: Introduction

Threat modeling for AI requires evaluating assets, behaviors, and failure modes unique to machine learning pipelines:
- **Data Poisoning**: Corrupting model weights at the training data source.
- **Model Theft / Extraction**: Cloning proprietary model intelligence via API queries.
- **Prompt Injection**: Subverting system instructions through direct or indirect natural language context.
- **Non-Deterministic Outputs**: Variations in model inference that make incident reproduction and auditing non-trivial.

### The Scenario: MegaCorp Threat Assessment
As a threat analyst at MegaCorp, you are tasked with conducting an exhaustive threat assessment across three critical AI services before an upcoming board meeting.

### Task 1 Questions & Answers

1. **I understand the learning objectives and am ready to learn about AI threat modelling!**  
   *Answer:* `No answer needed`

---

## Task 2: AI-Specific Assets and Attack Surfaces

Traditional applications revolve around familiar assets: relational databases, source code, API keys, configuration files, and authentication sessions. AI systems introduce an entirely new asset inventory.

### AI Asset Inventory

| Asset | What It Is | Architectural Impact / Risk |
| :--- | :--- | :--- |
| **Training Data** | Datasets used to teach the model its behavior | Poisoning corrupts model reasoning at the source; damage is baked directly into parameters. |
| **Model Weights / Parameters** | Learned numerical matrices defining behavior | Exfiltration gives adversaries a full functional copy of proprietary AI intellectual property. |
| **Embedding Vectors** | High-dimensional numerical representations of text | Used in RAG retrieval and feature matching; tampering shifts what context is fetched at inference. |
| **System Prompts** | Operational rules, boundaries, and persona guidelines | Leaks expose guardrails, security bounds, and internal business logic to attackers. |
| **Feature Stores** | Preprocessed live feature repositories | Tampering with features silently skews predictions without touching model weights. |
| **Model Registry / Artifacts** | Versioned repository of deployment-ready models | Compromise allows adversaries to swap legitimate models with backdoored variants. |

### Core Behavioral Differences
1. **Non-deterministic behavior**: Repeated queries with identical inputs can yield divergent completions, complicating regression testing and forensic reconstruction.
2. **The black box problem**: Deep neural networks cannot be stepped through like procedural source code. Defenders must reason in terms of input validation, output boundaries, and failure modes.

### Task 2 Questions & Answers

1. **In a RAG-based system, which AI asset type is used to retrieve relevant context at query time?**  
   *Answer:* `Embedding Vectors`

2. **An attacker gains access to MegaCorp's model registry and swaps the production model for a modified version. Which AI-specific asset has been compromised?**  
   *Answer:* `Model Registry / Artifacts`

---

## Task 3: Data Supply Chain and STRIDE's Gaps

Unlike traditional software supply chains where third-party libraries and container images can be scanned with SCA tools, AI systems possess a distinct **data supply chain**:

```
[ Stage 1: Data Collection ] 
       │ (Web scraping, partner feeds, user submissions)
       ▼
[ Stage 2: Cleaning & Labelling ]
       │ (Data filtering, deduplication, human/automated labeling)
       ▼
[ Stage 3: Model Training ]
       │ (Compute-intensive optimization into neural weights)
       ▼
[ Stage 4: Validation & Packaging ]
       │ (Benchmark scoring, model signing, registry storage)
       ▼
[ Stage 5: Inference & Serving ]
         (RAG context retrieval, vector lookup, prompt execution)
```

### Why Unmodified STRIDE Falls Short
- **Delayed & Diffuse Impact**: Tampering in traditional software throws exceptions or corrupts tables immediately. In training pipelines, data poisoning at **Stage 1 (Data Collection)** silently shifts decision boundaries over months without raising validation alerts.
- **Blurred Threat Categories**: Adversarial inputs that cause misclassification or guardrail bypass cross Tampering, Spoofing, and Elevation of Privilege simultaneously.
- **Broader Definition of Privilege**: An LLM agent equipped with database query or code execution tools turns prompt injection into full privilege escalation.
- **Intangible IP Theft**: Model extraction via API probing is technically Information Disclosure, but exfiltrates the entire trained system rather than a static record.

### Task 3 Questions & Answers

1. **An attacker injects crafted data points into a training pipeline over several months, gradually shifting the model's decision boundaries. At which supply chain stage does the attacker inject the malicious data?**  
   *Answer:* `Data Collection`

2. **Which STRIDE category is insufficient for capturing the delayed, diffuse effects of training data poisoning?**  
   *Answer:* `Tampering`

---

## Task 4: Adapting STRIDE for AI Systems

Defenders do not need to replace STRIDE; rather, they must retool each category to address AI mechanics:

```
S: Spoofing          -> Data Source Impersonation (RAG knowledge base poisoning)
T: Tampering         -> Data & Model Poisoning, Prompt Injection
R: Repudiation       -> Non-explainable Model Decisions & Missing Context Logs
I: Info Disclosure   -> Model Extraction (Stealing), Training Data Extraction
D: Denial of Service -> Inference Cost Exploitation (Denial of Wallet)
E: Elevation of Priv -> Jailbreaking, Guardrail Bypass, Excessive Agency
```

### Retooled STRIDE-AI Breakdown
1. **Spoofing (Data Source Impersonation)**: In RAG setups, injecting malicious records into the retrieval store tricks the model into treating untrusted external content as authoritative truth.
2. **Tampering (Data Poisoning)**: Submitting poisoned transactions shifts the fraud detection model's decision boundary.
3. **Repudiation (Audit Trail Gaps)**: Regulators ask why a loan or fraud alert was triggered. Without detailed logs of system prompt, user prompt, retrieved chunks, model version, and temperature, the decision cannot be proven or defended.
4. **Information Disclosure (Model Extraction)**: Querying an API systematically with thousands of inputs to reverse-engineer decision boundaries and construct a shadow surrogate model.
5. **Denial of Service (Denial of Wallet)**: Flooding endpoints with expensive maximum-context prompts. The service remains responsive, but inference costs skyrocket from $15,000 to $180,000.
6. **Elevation of Privilege (Jailbreaking & Excessive Agency)**: Bypassing safety prompts to access underlying database query tools, enabling arbitrary customer data queries.

### Task 4 Questions & Answers

1. **What is the primary AI-specific manifestation of Information Disclosure in the STRIDE-AI mapping?**  
   *Answer:* `Model Extraction`

2. **An attacker crafts prompts that cause an LLM to bypass its safety guidelines and content restrictions. Which STRIDE category does this map to?**  
   *Answer:* `Elevation of Privilege`

3. **Which OWASP LLM Top 10 (2025) entry addresses the risks of AI systems being granted too many permissions or too much autonomy?**  
   *Answer:* `Excessive Agency` *(also accepted: `LLM06:2025 - Excessive Agency`)*

4. **An attacker drives your monthly inference bill from $15,000 to $180,000 without taking your service offline. What is this type of attack commonly called?**  
   *Answer:* `Denial of Wallet`

---

## Task 5: MITRE ATLAS: The AI Threat Technique Catalogue

**MITRE ATLAS** (Adversarial Threat Landscape for Artificial-Intelligence Systems) provides the tactical knowledge base for AI security, operating as the machine learning counterpart to MITRE ATT&CK.

### ATLAS Structural Hierarchy
- **Tactics**: The adversary's overarching objective (e.g., *ML Attack Staging* - `AML.TA0012`).
- **Techniques**: The specific method employed (e.g., *Data Poisoning* - `AML.T0020`, *Model Extraction* - `AML.T0024`).
- **Sub-techniques**: Detailed execution variants (e.g., *Craft Adversarial Data* - `AML.T0043.004`).
- **Mitigations**: Prescribed defensive countermeasures.

### Critical ATLAS Techniques
- **Data Poisoning (`AML.T0020`)**: Corrupting training pipelines to embed persistent malicious associations.
- **Model Extraction (`AML.T0024`)**: Reconstructing a functional model via API queries.
- **Evade ML Model (`AML.T0015`)**: Crafting adversarial perturbations that trigger targeted misclassification.
- **LLM Prompt Injection (`AML.T0051`)**: Direct input injection or indirect document-based injection via RAG.
- **Backdoor ML Model (`AML.T0018`)**: Inserting dormant trigger patterns into model weights that activate only on specific inputs.

### Notable ATLAS Case Studies
- **ShadowRay (`AML.CS0023`)**: Exploitation of Ray distributed AI compute clusters in production environments.
- **Morris II Worm (`AML.CS0024`)**: A self-replicating prompt injection worm propagating across agentic LLMs via RAG email workflows.

### Task 5 Questions & Answers

1. **What does the acronym ATLAS stand for?**  
   *Answer:* `Adversarial Threat Landscape for Artificial-Intelligence Systems`

2. **Which ATLAS case study described a self-replicating prompt injection worm that spread between AI agents via RAG email systems?**  
   *Answer:* `Morris II Worm`

3. **What is the ATLAS technique ID for Model Extraction?**  
   *Answer:* `AML.T0024`

---

## Task 6: OWASP LLM Top 10: Mapping Risks to Components

While STRIDE categorizes threat types and ATLAS catalogues adversary techniques, the **OWASP Top 10 for LLM Applications (2025)** maps risks directly to specific architectural components.

```
Component Risk Focus:
├── LLM Inference Endpoint (7 Risks):
│   ├── LLM01: Prompt Injection
│   ├── LLM02: Sensitive Information Disclosure
│   ├── LLM05: Improper Output Handling
│   ├── LLM06: Excessive Agency
│   ├── LLM07: System Prompt Leakage
│   ├── LLM09: Misinformation
│   └── LLM10: Unbounded Consumption
│
├── Vector Database / RAG Pipeline (3 Risks):
│   ├── LLM01: Indirect Prompt Injection
│   ├── LLM08: Vector and Embedding Weaknesses
│   └── LLM09: Misinformation (Stale Sources)
│
└── Training Pipeline & Registry (3 Risks):
    ├── LLM02: Sensitive Data in Training
    ├── LLM03: Supply Chain
    └── LLM04: Data and Model Poisoning
```

### Architectural Mapping Table

| OWASP Entry | Title | Primary Architectural Location | Core Threat |
| :--- | :--- | :--- | :--- |
| **LLM01** | Prompt Injection | Inference Endpoint, RAG Vector Store | Control flow hijack via direct prompts or retrieved documents |
| **LLM02** | Sensitive Info Disclosure | Inference Endpoint, Training Pipeline | Memorization regurgitation, PII leakage |
| **LLM03** | Supply Chain | Training Pipeline, Model Registry, Plugins | Poisoned base models, untrusted Hugging Face weights |
| **LLM04** | Data & Model Poisoning | Training Pipeline, Feature Store | Manipulated training samples or runtime features |
| **LLM05** | Improper Output Handling | Web Frontend, Downstream API Gateway | Unsanitized model output passed to browser (XSS) or backend |
| **LLM06** | Excessive Agency | Tool Integrations, API Gateway | Unconstrained tools or overly permissive write access |
| **LLM07** | System Prompt Leakage | Inference Endpoint, Prompt Config | Extraction of hidden instructions, credentials, or logic |
| **LLM08** | Vector & Embedding Weaknesses | Vector Database, Embedding Process | Embedding inversion, retrieval manipulation |
| **LLM09** | Misinformation | Inference Endpoint, Vector Store | Hallucinations accepted as truth without validation |
| **LLM10** | Unbounded Consumption | Inference Endpoint, API Gateway | Denial of Wallet, resource exhaustion |

### Task 6 Questions & Answers

1. **How many of the OWASP LLM Top 10 entries affect the LLM Inference Endpoint?**  
   *Answer:* `7`

2. **An organisation notices their chatbot is rendering LLM output directly in the browser without sanitisation. Which OWASP entry does this fall under?**  
   *Answer:* `Improper Output Handling` *(also accepted: `LLM05:2025 - Improper Output Handling`)*

3. **Which component in a typical LLM architecture is the primary one that needs hardening against data and model supply chain risks (LLM03)?**  
   *Answer:* `Training Pipeline`

---

## Task 7: Practical: Threat Modelling MegaCorp's AI Assistant

Click the **View Site** button to launch the interactive MegaCorp Threat Assessment application.

### Assessment Methodology
In this exercise, you evaluate MegaCorp's multi-tier architecture by mapping OWASP risks to their corresponding architectural components and selecting appropriate technical justifications:

1. **Customer-Facing Web Interface**:
   - Primary Risk: **LLM05 (Improper Output Handling)**
   - Justification: Chat responses rendered directly in user browsers without HTML encoding create Cross-Site Scripting (XSS) vectors.
2. **LLM Inference Engine & Agent Controller**:
   - Primary Risks: **LLM01 (Prompt Injection)**, **LLM06 (Excessive Agency)**, **LLM07 (System Prompt Leakage)**, **LLM10 (Unbounded Consumption)**
   - Justifications: Direct user input can override system constraints, invoke unconstrained database lookup tools, leak internal instructions, or run up compute costs.
3. **RAG Vector Database & Document Store**:
   - Primary Risks: **LLM01 (Indirect Prompt Injection)**, **LLM08 (Vector and Embedding Weaknesses)**
   - Justifications: Ingesting untrusted policy documents allows attackers to embed prompt injections into search results that execute when the assistant retrieves context.
4. **Data Ingestion & Training Pipeline**:
   - Primary Risks: **LLM03 (Supply Chain)**, **LLM04 (Data and Model Poisoning)**
   - Justifications: Retraining pipelines consuming unverified external data sources or third-party base weights risk embedding backdoors directly into the model.

Submitting the verified component-to-risk matrix yields the final completion flag:

```text
THM{AI_THREAT_MODEL_COMPLETE}
```

### Task 7 Questions & Answers

1. **What's the flag?**  
   *Answer:* `THM{AI_THREAT_MODEL_COMPLETE}`

---

## Task 8: Conclusion

A resilient AI threat model combines three complementary frameworks:
1. **STRIDE-AI**: Provides the wide-angle view, categorizing threat types (what could go wrong).
2. **MITRE ATLAS**: Provides the technical detail, mapping specific adversary techniques, IDs, and real-world attack studies (how it happens).
3. **OWASP LLM Top 10**: Provides the architectural targeting, highlighting exactly which component carries which risk and at what severity (where to defend).

### Task 8 Questions & Answers

1. **I have successfully completed the room!**  
   *Answer:* `No answer needed`

---

## Summary of Questions, Answers, and Flags

| Task | Question | Answer / Flag |
| :--- | :--- | :--- |
| **Task 1: Introduction** | I understand the learning objectives and am ready to learn about AI threat modelling! | `No answer needed` |
| **Task 2: AI Assets & Attack Surfaces** | In a RAG-based system, which AI asset type is used to retrieve relevant context at query time? | `Embedding Vectors` |
| **Task 2: AI Assets & Attack Surfaces** | An attacker gains access to MegaCorp's model registry and swaps the production model for a modified version. Which AI-specific asset has been compromised? | `Model Registry / Artifacts` |
| **Task 3: Data Supply Chain** | An attacker injects crafted data points into a training pipeline over several months, gradually shifting the model's decision boundaries. At which supply chain stage does the attacker inject the malicious data? | `Data Collection` |
| **Task 3: Data Supply Chain** | Which STRIDE category is insufficient for capturing the delayed, diffuse effects of training data poisoning? | `Tampering` |
| **Task 4: Adapting STRIDE** | What is the primary AI-specific manifestation of Information Disclosure in the STRIDE-AI mapping? | `Model Extraction` |
| **Task 4: Adapting STRIDE** | An attacker crafts prompts that cause an LLM to bypass its safety guidelines and content restrictions. Which STRIDE category does this map to? | `Elevation of Privilege` |
| **Task 4: Adapting STRIDE** | Which OWASP LLM Top 10 (2025) entry addresses the risks of AI systems being granted too many permissions or too much autonomy? | `Excessive Agency` |
| **Task 4: Adapting STRIDE** | An attacker drives your monthly inference bill from $15,000 to $180,000 without taking your service offline. What is this type of attack commonly called? | `Denial of Wallet` |
| **Task 5: MITRE ATLAS** | What does the acronym ATLAS stand for? | `Adversarial Threat Landscape for Artificial-Intelligence Systems` |
| **Task 5: MITRE ATLAS** | Which ATLAS case study described a self-replicating prompt injection worm that spread between AI agents via RAG email systems? | `Morris II Worm` |
| **Task 5: MITRE ATLAS** | What is the ATLAS technique ID for Model Extraction? | `AML.T0024` |
| **Task 6: OWASP LLM Top 10** | How many of the OWASP LLM Top 10 entries affect the LLM Inference Endpoint? | `7` |
| **Task 6: OWASP LLM Top 10** | An organisation notices their chatbot is rendering LLM output directly in the browser without sanitisation. Which OWASP entry does this fall under? | `Improper Output Handling` |
| **Task 6: OWASP LLM Top 10** | Which component in a typical LLM architecture is the primary one that needs hardening against data and model supply chain risks (LLM03)? | `Training Pipeline` |
| **Task 7: Practical** | What's the flag? | `THM{AI_THREAT_MODEL_COMPLETE}` |
| **Task 8: Conclusion** | I have successfully completed the room! | `No answer needed` |

---

_Happy Hacking!_  
**Muhammed Omer**

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
