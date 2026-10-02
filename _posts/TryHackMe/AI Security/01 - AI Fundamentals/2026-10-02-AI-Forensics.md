---
layout: post
title: "TryHackMe: AI Forensics"
date: 2026-10-02T00:20:00
categories:
  - TryHackMe
  - AI Security
tags:
  - thm
  - ai
  - dfir
  - forensics
  - scikit-learn
  - incident-response
  - writeup
author: muhammed
description: A complete walkthrough of the TryHackMe room AI Forensics, exploring how machine learning powers digital forensics, legal and ethical implications, and an interactive breach investigation using scikit-learn.
toc: true
pin: false
math: false
mermaid: false
image: https://cdn-images.tryhackme.com/room-icons/6228f0d4ca8e57005149c3e3-1775756737657
---

## Overview

[AI Forensics](https://tryhackme.com/r/room/aiforensics) explores the intersection of Digital Forensics and Incident Response (DFIR) with Machine Learning. Modern enterprise environments generate massive forensic telemetry volumes across endpoints, identity providers, and cloud infrastructure. Sifting through this volume manually creates significant investigation latency.

This room breaks down:
1. The AI Forensics landscape: data processing, anomaly detection, and operational scalability.
2. Core forensic domains enhanced by AI: image and video verification, communication analysis, timeline reconstruction, and malware classification.
3. Legal and ethical challenges: black box explainability in courtrooms, algorithmic bias, chain of custody preservation, and privacy regulations.
4. "The Digital Trail" practical lab: investigating a corporate espionage breach at RobbCo using `scikit-learn` machine learning triage scripts.

---

## Task 1: Introduction

Digital forensics requires establishing accurate timelines, preserving evidence integrity, and proving findings under strict legal standards. While traditional tools rely on fixed signatures and regex queries, AI introduces probabilistic classification and automated pattern recognition.

### Learning Objectives

* Understand the data volume and correlation challenges faced in modern DFIR.
* Identify where machine learning enhances triage speed, event correlation, and anomaly detection.
* Evaluate the legal and ethical limits of AI-generated evidence, including admissibility under the Daubert standard.
* Execute an end-to-end incident investigation combining machine learning triage with human analytical verification.

### Task 1 Questions and Answers

* **Question:** I'm ready to learn!  
  **Answer:** `No answer needed`

---

## Task 2: The AI Forensics Landscape

Forensic investigations often hinge on finding subtle indicators of compromise obscured within massive benign event logs. AI models address this through three capabilities:
* **High-Speed Data Processing:** Deep learning transformers process large unstructured log exports in parallel within milliseconds.
* **Anomaly Detection:** Unsupervised models establish behavioral baselines for users and host systems, surfacing statistical anomalies without pre-configured rules.
* **Elastic Scalability:** Distributed models continuously ingest telemetry across cloud, hybrid, and remote endpoints.

### Industry Tools and ML Applications

| DFIR Domain | AI/ML Capability | Representative Platforms | Underlying Mechanism |
|---|---|---|---|
| **UEBA / Anomaly Detection** | Flags aberrant user or system behavior against established baselines | Splunk UEBA, Elastic ML, Exabeam | Unsupervised algorithms like Isolation Forests and Autoencoders detect deviations without manual thresholds. |
| **Phishing & Messaging** | Detects social engineering language in email and chat logs | Microsoft Defender for O365, Splunk NLP | Transformer models (BERT, RoBERTa) evaluate message syntax, urgency markers, and domain spoofing. |
| **Malware Classification** | Identifies malicious binaries using static and dynamic attributes | Microsoft STAMINA, Cylance, VirusTotal Code Insight | Converts binary byte structures or API call sequences into feature vectors or 2D image representations for neural network classification. |
| **Alert Triage** | Scores and prioritizes high-fidelity security events | Palo Alto Cortex XSIAM, IBM QRadar Advisor, CrowdStrike Charlotte AI | Supervised classifiers trained on historical analyst feedback score alerts to filter repetitive noise. |
| **Timeline Correlation** | Reconstructs multi-source attack progressions | Timesketch, Velociraptor, Jupyter AI | Clusters event logs across hosts and establishes causal connections based on time sequencing. |

```
[ Raw Event Streams ] ──> [ Feature Normalization ] ──> [ Unsupervised ML (Isolation Forest) ]
                                                                      │
                                                                      ▼
[ Forensic Analyst Review ] <── [ Ranked Anomalies ] <── [ High-Probability Outliers ]
```

### Critical Limitations: Evaluating AI in DFIR

1. **Probabilistic Nature (Non-determinism):** Traditional forensics tools are deterministic: hashing an artifact or running a grep filter produces repeatable results. AI models are probabilistic. Because models generate outputs based on probability distributions, identical inputs can yield divergent results across runs. In court, lack of repeatability can undermine evidence credibility.
2. **Evaluation Metric Trade-Offs:**
   * **Accuracy:** Can be misleading on imbalanced datasets. If 99 out of 100 files are benign, a model predicting "benign" every time achieves 99% accuracy while missing the single malicious payload.
   * **Precision:** Measures the ratio of true positives among all flagged items. High precision minimizes false positives.
   * **Recall:** Measures the proportion of actual malicious items detected. High recall minimizes false negatives.
3. **Garbage In, Garbage Out (GIGO):** Training models on biased or incomplete telemetry guarantees confident, incorrect predictions.

### Task 2 Questions and Answers

* **Question:** What ability of AI helps turn a DFIR investigator by recognising patterns they might not have been able to comprehend?  
  **Answer:** `Anomaly Detection`
* **Question:** Which metric tells you the proportion of positively flagged results that were actually correct?  
  **Answer:** `Precision`
* **Question:** What term describes the AI characteristic where the same input may yield different outputs across different runs?  
  **Answer:** `Non-determinism`

---

## Task 3: AI & DFIR

AI and machine learning techniques directly address operational challenges across four core digital forensics subfields:

### 1. Image and Video Forensics
* **Convolutional Neural Networks (CNNs):** Excel at learning spatial hierarchies and pixel-level textures in visual media.
* **Error Level Analysis (ELA) + CNN:** ELA visualizes compression inconsistencies across JPEG blocks, while a CNN classifies tampered regions, achieving high accuracy in forgery detection.
* **Deepfake Identification:** Specialized CNNs detect temporal inconsistencies, abnormal facial landmark movements, and unnatural blinking patterns.
* **Generative Adversarial Networks (GANs):** A generator creates synthetic fakes while a discriminator attempts to identify them. Forensics teams leverage discriminator models to train robust detection filters against synthetic media.

### 2. Communication Analysis
* **Transformer NLP Models:** BERT and RoBERTa analyze communication intent rather than relying solely on static blacklists of malicious URLs.
* **Sentiment and Tone Analysis:** Analyzes corporate chat logs and email threads during insider threat investigations to gauge emotional tone, urgency spikes, or disgruntled employee behavior.

### 3. Timeline Reconstruction and User Behaviour
* **Correlating Time-Sequenced Data:** Ingests disparate audit trails (filesystem MFT records, authentication events, firewall logs) and automatically unifies them into a coherent chronological sequence.
* **Behavioral Anomaly Detection:** Flags physically impossible logins (e.g., successful logins from two distinct continents within minutes) and abnormal data access volumes.

### 4. Malware Analysis
* **Static Byte Representation:** Converts binary file bytes into greyscale image arrays for CNN-based classification (the STAMINA framework developed by Microsoft and Intel).
* **Dynamic Behavioral Analysis:** Executes samples in a sandbox and maps API call sequences into structured matrices to detect runtime evasion techniques.

### Task 3 Questions and Answers

* **Question:** What type of neural network is commonly used in image and video forensics due to its ability to learn spatial patterns in visual data?  
  **Answer:** `CNN`
* **Question:** What kind of analysis can be performed on social media or chat logs to assess the emotional tone of messages?  
  **Answer:** `Sentiment analysis`
* **Question:** What type of data do AI systems correlate to reconstruct the timeline of an incident automatically?  
  **Answer:** `time-sequenced data`
* **Question:** What type of analysis observes how a program behaves to determine whether it is malicious, e.g., using its API call sequence?  
  **Answer:** `Dynamic analysis`

---

## Task 4: AI Legal & Ethical Implications

Deploying AI in digital forensics introduces complex legal hurdles when evidence is presented in judicial proceedings.

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVIDENTIARY STANDARDS                        │
├─────────────────────────────────────────────────────────────────┤
│ [1. EXPLAINABILITY]  Must satisfy the Daubert standard          │
│ [2. BIAS AUDITING]   Algorithmic bias invalidates findings       │
│ [3. CHAIN OF CUSTODY] Intermediate ML steps must be logged      │
│ [4. DATA PRIVACY]    Evidence must not leak to third-party APIs │
└─────────────────────────────────────────────────────────────────┘
```

1. **Explainability and the Daubert Test:** The U.S. federal **Daubert test** assesses whether expert scientific testimony is based on peer-reviewed, empirically testable methodology with a known error rate. Black box models that cannot explain why an email was classified as malicious can lead courts to exclude the evidence entirely.
2. **Algorithmic Bias:** Machine learning algorithms trained on skewed historical datasets can reproduce societal prejudices. A notable real-world example is law enforcement facial recognition systems, which have shown significantly higher error rates when identifying minority suspects, leading to multiple documented wrongful arrests.
3. **Chain of Custody and Audit Trails:** Digital evidence requires unbroken, verifiable provenance. Running evidence through cloud-hosted LLMs without logging intermediate prompts, temperature configurations, and response hashes can invalidate the chain of custody.
4. **Data Protection (GDPR Compliance):** Processing evidence containing PII on public cloud AI endpoints can violate privacy laws or court-ordered evidence containment. Sensitive forensic processing should use air-gapped on-premises systems or **federated learning** architectures.

### Task 4 Questions and Answers

* **Question:** What legal test used in the U.S. assesses whether expert or scientific testimony is admissible in court?  
  **Answer:** `Daubert test`
* **Question:** What term describes AI models whose internal decision-making processes are difficult to interpret?  
  **Answer:** `Black boxes`
* **Question:** What real-world technology used by law enforcement has been shown to produce racially biased results in identifying suspects?  
  **Answer:** `Facial recognition`
* **Question:** What technique allows machine learning to be performed without transferring sensitive data to a central server, helping preserve privacy?  
  **Answer:** `Federated learning`

---

## Task 5: Practical - The Digital Trail

In this hands-on challenge, you investigate a corporate espionage incident at **RobbCo**, a software firm whose proprietary firmware and operating systems (RETROS BIOS, MF Boot Agent, UOS) were targeted.

### Setting Up the Environment

Activate the isolated forensic virtual environment:

```bash
source /opt/dfir-env/bin/activate
```

### Phase 1: Machine Learning Log Classification

Run the ML log classifier to analyze authentication events:

```bash
python3 /opt/dfir-lab/classify_logs.py /var/log/auth.log
```

The script parses `/var/log/auth.log` and flags an anomaly:
* Initial failed login attempts for `admin`.
* A successful login for user `j.morgan` at **`03:01:02`**.
* Subsequent privilege escalation activity leading to the founder account `r.house`.

### Phase 2: File System Anomaly Triage

Run the anomaly detector across sensitive system directories:

```bash
python3 /opt/dfir-lab/file_anomalies.py
```

The script surfaces several suspicious artifacts based on entropy, file paths, and creation timestamps:
* `/tmp/invoice_dump.txt`: Reconnaissance output containing bash history, usernames, and an open command targeting `~/Documents/Invoices/invoice_Q1_2075.ods`.
* `/home/j.morgan/Documents/Invoices/invoice_Q1_2075.ods`: A weaponized spreadsheet containing an embedded macro that harvested `.bash_history`, SSH keys, and `/etc/passwd`.

Inspect the user's mail spool to locate the delivery vector:

```bash
cat /home/j.morgan/Mail/inbox/email_invoice.eml
```

The email confirms a targeted spear-phishing attack delivered by sender **`akeane@poseidonenergy.net`**.

### Phase 3: Tooling and Infrastructure

Reviewing the secondary files flagged by the anomaly script:
* `/tmp/.syncd`: First-stage downloader retrieving `http://10.0.0.66/payload.sh`.
* `/tmp/.x`: Reverse shell payload connecting back to `10.0.0.66:4444`.

### Phase 4: Privilege Escalation and Persistence

Inspect `j.morgan`'s shell history to understand how the attacker escalated to `r.house`:

```bash
cat /home/j.morgan/.bash_history
```

The attacker abused legitimate sudo privileges to plant an SSH key:

```bash
sudo nano /home/r.house/.ssh/authorized_keys
```

To maintain persistence under `r.house`, the attacker established a secondary reverse shell:
* `/usr/local/bin/sysmon`: Connects out to `10.0.0.66:5555`, disguised as a system monitoring tool.
* `/opt/robbco/sys/boot_monitor.log`: Fabricated logs designed to justify `sysmon`'s process presence.

### Phase 5: Exfiltration of Source Code

The attacker staged the stolen intellectual property in shared memory:
* Staging Path: **`/dev/shm/.core_dump_2025.tgz.enc`**

Unpack and inspect the exfiltration archive:

```bash
base64 -d /dev/shm/.core_dump_2025.tgz.enc > /tmp/stolen.tar.gz
tar -xzvf /tmp/stolen.tar.gz -C /tmp/stolen_source
```

The archive reveals RobbCo's proprietary `MFBootAgent` source code and `RETROS_BIOS` assembly files.

### Task 5 Questions and Answers

* **Question:** At what time does the attacker successfully log in as j.morgan?  
  **Answer:** `03:01:02`
* **Question:** What attack method was used to gain initial access?  
  **Answer:** `Phishing`
* **Question:** Can you find the attacker's email address?  
  **Answer:** `akeane@poseidonenergy.net`
* **Question:** What command did the attacker run as j.morgan to gain access to the r.house account?  
  **Answer:** `sudo nano /home/r.house/.ssh/authorized_keys`
* **Question:** What is the full path of the archive used to steal RobbCo's source code?  
  **Answer:** `/dev/shm/.core_dump_2025.tgz.enc`

---

## Task 6: Conclusion

Machine learning provides forensic teams with significant efficiency gains, but it does not replace human verification:
* Anomaly detection algorithms quickly narrow thousands of logs down to high-priority events.
* Forensic findings must maintain strict chain of custody, explainability, and defensibility to satisfy legal standards like the Daubert test.
* Human analysts remain indispensable for validating automated classifications, discerning intent, and verifying the true impact of an attack.

### Task 6 Questions and Answers

* **Question:** All done!  
  **Answer:** `No answer needed`

---

## Key Takeaways

| Forensic Phase | Traditional Technique | AI-Enhanced Capability | Human Verification Requirement |
|---|---|---|---|
| **Log Triage** | Manual regex grep queries across raw auth logs | Unsupervised anomaly detection (Isolation Forests) | Distinguish legitimate administrative spikes from credential abuse |
| **Media Verification** | Hex header analysis and metadata inspection | Error Level Analysis (ELA) combined with CNN classifiers | Corroborate synthetic media artifacts with origin source devices |
| **Phishing Analysis** | Header domain matching and static URL lookups | Contextual NLP transformers (BERT / RoBERTa) | Verify social engineering context against external communications |
| **Timeline Building** | Manual chronological spreadsheet mapping | Automated correlation of multi-source time-sequenced data | Validate cause-and-effect sequences across independent hosts |
| **Legal Defensibility** | Documented tool signatures and manual notes | Model cards, deterministic seeding, and logged prompts | Satisfy Daubert admissibility and explain reasoning in court |

---

## Summary of Questions, Answers, and Flags

| Task | Question | Answer |
|---|---|---|
| **Task 1** | I'm ready to learn! | `No answer needed` |
| **Task 2** | What ability of AI helps turn a DFIR investigator by recognising patterns they might not have been able to comprehend? | `Anomaly Detection` |
| **Task 2** | Which metric tells you the proportion of positively flagged results that were actually correct? | `Precision` |
| **Task 2** | What term describes the AI characteristic where the same input may yield different outputs across different runs? | `Non-determinism` |
| **Task 3** | What type of neural network is commonly used in image and video forensics due to its ability to learn spatial patterns in visual data? | `CNN` |
| **Task 3** | What kind of analysis can be performed on social media or chat logs to assess the emotional tone of messages? | `Sentiment analysis` |
| **Task 3** | What type of data do AI systems correlate to reconstruct the timeline of an incident automatically? | `time-sequenced data` |
| **Task 3** | What type of analysis observes how a program behaves to determine whether it is malicious, e.g., using its API call sequence? | `Dynamic analysis` |
| **Task 4** | What legal test used in the U.S. assesses whether expert or scientific testimony is admissible in court? | `Daubert test` |
| **Task 4** | What term describes AI models whose internal decision-making processes are difficult to interpret? | `Black boxes` |
| **Task 4** | What real-world technology used by law enforcement has been shown to produce racially biased results in identifying suspects? | `Facial recognition` |
| **Task 4** | What technique allows machine learning to be performed without transferring sensitive data to a central server, helping preserve privacy? | `Federated learning` |
| **Task 5** | At what time does the attacker successfully log in as j.morgan? | `03:01:02` |
| **Task 5** | What attack method was used to gain initial access? | `Phishing` |
| **Task 5** | Can you find the attacker's email address? | `akeane@poseidonenergy.net` |
| **Task 5** | What command did the attacker run as j.morgan to gain access to the r.house account? | `sudo nano /home/r.house/.ssh/authorized_keys` |
| **Task 5** | What is the full path of the archive used to steal RobbCo's source code? | `/dev/shm/.core_dump_2025.tgz.enc` |
| **Task 6** | All done! | `No answer needed` |

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
