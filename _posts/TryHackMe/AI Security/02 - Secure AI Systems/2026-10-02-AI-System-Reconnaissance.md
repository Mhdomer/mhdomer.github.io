---
layout: post
title: "TryHackMe: AI System Reconnaissance"
date: 2026-10-02T00:55:00
categories:
  - AI Security
  - 02 - Secure AI Systems
tags:
  - tryhackme
  - thm
  - ai
  - reconnaissance
  - mlflow
  - triton
  - mitre-atlas
  - writeup
author: muhammed
description: A complete walkthrough of the TryHackMe room AI System Reconnaissance (Module 2), scanning ML services, fingerprinting Triton, MLflow, and Qdrant, extracting model metadata, and mapping the Cyphira AI attack surface.
toc: true
pin: false
math: false
mermaid: false
image: https://cdn-images.tryhackme.com/room-icons/69650d18bb3fe8c456972924-1775820771840
---

## Overview

[AI System Reconnaissance](https://tryhackme.com/r/room/aisystemreconnaissance) is the fourth room in **Module 2: Secure AI Systems** on TryHackMe. While threat modeling evaluates hypothetical risks and architectural boundaries, reconnaissance confirms what AI components are actually deployed and exposed across an enterprise network.

When organisations introduce machine learning pipelines, their networks change in ways standard port scans and vulnerability scanners overlook. Experiment tracking servers, model registries, tensor serving engines, and vector databases operate on unfamiliar ports, expose binary gRPC protocols, and return rich metadata structures by default.

In this room, we conduct a security audit for **Cyphira**, a fintech company that deployed AI capabilities without security review. We systematically discover, fingerprint, enumerate, and map AI infrastructure across the `10.10.45.0/24` subnet using Nmap, curl, and grpcurl.

---

## Task 1: Introduction

In January 2026, security researchers identified over 42,000 exposed AI agent instances on Shodan, with 93% vulnerable or leaking API keys via unauthenticated interfaces. Similarly, threat intelligence reports recorded tens of thousands of automated reconnaissance sessions probing conversational endpoints and model serving APIs.

AI reconnaissance focuses on discovering:
- Model serving engines (Triton, TorchServe, vLLM, Ollama).
- Orchestration and experiment tracking servers (MLflow, Kubeflow, Ray).
- Vector databases (Qdrant, Weaviate, Milvus, Chroma).
- Supporting infrastructure (unauthenticated Jupyter notebooks, MinIO artifact storage).

### Task 1 Questions & Answers

1. **I understand the learning objectives and am ready to learn about AI system reconnaissance!**  
   *Answer:* `No answer needed`

---

## Task 2: The AI Infrastructure Stack

Enterprise AI architectures rely on a mesh of specialized services spanning multiple ports and protocols:

### AI Infrastructure Port Reference Table

| Component | Default Port(s) | Protocol(s) | Common Recon Endpoints |
| :--- | :--- | :--- | :--- |
| **NVIDIA Triton** | 8000, 8001, 8002 | HTTP, gRPC, Prometheus | `/v2/health/ready`, `/v2/models` |
| **TensorFlow Serving** | 8500, 8501 | gRPC, HTTP | `/v1/models/<name>` |
| **TorchServe** | 8080, 8081, 8082 | HTTP | `/ping`, `/models` |
| **Ollama** | 11434 | HTTP | `/api/tags`, `/api/show` |
| **vLLM** | 8000 | HTTP | `/v1/models` |
| **MLflow** | 5000 | HTTP | `/api/2.0/mlflow/experiments/search` |
| **Kubeflow** | 80, 443 | HTTP | `/pipeline/apis/v1beta1/pipelines` |
| **Ray** | 8265, 8000 | HTTP | `/api/jobs/`, Ray Dashboard |
| **Qdrant** | 6333, 6334 | HTTP, gRPC | `/collections` |
| **Weaviate** | 8080 | HTTP, GraphQL | `/v1/schema`, `/v1/meta` |
| **Milvus** | 19530 | gRPC | Port 19530 connection |
| **Jupyter Notebook** | 8888 | HTTP | `/api/kernels`, `/api/contents` |
| **MinIO** | 9000, 9001 | HTTP (S3 API) | Bucket listing |
| **Prometheus Metrics**| 8002, 8082 | HTTP | `/metrics` |

### Scanning Cyphira's Internal Subnet
Executing targeted Nmap scans against `10.10.45.0/24`:

```bash
nmap -p 5000,6333,6334,8000,8001,8002,8888,9000,9001 -sV 10.10.45.0/24
```

The sweep reveals active AI nodes:
- `10.10.45.12`: Port 5000 (MLflow)
- `10.10.45.15`: Ports 8000, 8001, 8002 (NVIDIA Triton Inference Server)
- `10.10.45.18`: Port 6333 (Qdrant Vector Database)
- `10.10.45.20`: Port 8888 (Jupyter Notebook)
- `10.10.45.22`: Ports 9000, 9001 (MinIO Object Storage)

### Task 2 Questions & Answers

1. **What is the IP address of the host running an HTTP service on port 8888 in your scan results?**  
   *Answer:* `10.10.45.20`

2. **Which port does MLflow Tracking Server run on by default?**  
   *Answer:* `5000`

---

## Task 3: Fingerprinting AI Services

Generic Nmap scans frequently mislabel AI services as `http-alt` or unknown binary streams. Precise framework identification requires inspecting HTTP headers, response JSON structures, error conditions, and gRPC reflection.

### 1. HTTP Headers & Signatures
- **TorchServe**: Returns `Server: TorchServe/0.x.x`.
- **NVIDIA Triton**: Returns the custom header `NV-Status`. Sending the header `endpoint-load-metrics-format: text` causes Triton to return CPU and GPU hardware utilization metrics directly in response headers.
- **FastAPI ML Backends**: Return `server: uvicorn`.
- **OpenAI-Compatible Endpoints (vLLM, Ollama)**: Return `{"object": "model"}` arrays.

### 2. Error Message Fingerprinting
AI inference engines enforce rigid tensor schemas. Supplying invalid inputs triggers diagnostic stack traces revealing internal frameworks:
- Submitting malformed payloads to TensorFlow Serving returns `tensorinfo_map` errors.
- Probing MLflow returns `mlflow.server` or `mlflow.tracking` stack traces.

### 3. gRPC Reflection Probing
Because Triton exposes gRPC on port 8001, we inspect exposed RPC methods via `grpcurl`:

```bash
grpcurl -plaintext 10.10.45.15:8001 list
```

The output confirms active reflection:
```text
grpc.reflection.v1alpha.ServerReflection
inference.GRPCInferenceService
```

### Task 3 Questions & Answers

1. **Which unique HTTP response header does the service on 10.10.45.15:8000 return to identify as an NVIDIA product?**  
   *Answer:* `NV-Status`

2. **When you run grpcurl against 10.10.45.15:8001, what is the name of the inference service listed in the reflection output?**  
   *Answer:* `inference.GRPCInferenceService`

---

## Task 4: Enumerating AI Systems

Once services are fingerprinted, automated enumeration extracts intellectual property, model architectures, internal credentials, and storage URIs.

### 1. MLflow Tracking & Registry Enumeration
An exposed MLflow server can be completely enumerated via REST:
- **List Experiments**: `POST /api/2.0/mlflow/experiments/search`
- **List Registered Models**: `GET /api/2.0/mlflow/registered-models/list`
- **Search Model Versions & Artifact URIs**: `GET /api/2.0/mlflow/model-versions/search`
- **Search Training Runs & Parameters**: `POST /api/2.0/mlflow/runs/search`

The endpoint `/api/2.0/mlflow/model-versions/search` reveals internal S3 storage locations (`s3://cyphira-ml-models/...`), deployment stages (Production vs. Staging), and developer user IDs.

### 2. Triton Model Metadata
Querying Triton at `http://10.10.45.15:8000/v2/models/fraud_detector/config` exposes the exact input tensor dimensions, data types (FP32, INT8), batch sizes, and model backend formats.

### 3. Jupyter Notebook Credential Harvesting
Querying Jupyter on `http://10.10.45.20:8888/api/contents` lists stored notebooks. Inspecting the active notebook cells exposes hardcoded cleartext credentials:

```python
MLFLOW_TRACKING_URI = "http://10.10.45.12:5000"
MLFLOW_TRACKING_USERNAME = "mlflow_svc"
MLFLOW_TRACKING_PASSWORD = "Cyphira-MLflow-2024!"
```

### Task 4 Questions & Answers

1. **What MLflow REST API endpoint would you use to retrieve the artifact storage location for a specific model version?**  
   *Answer:* `/api/2.0/mlflow/model-versions/search`

2. **What is the cleartext password for the MLflow service account stored in the Jupyter notebook on 10.10.45.20?**  
   *Answer:* `Cyphira-MLflow-2024!`

---

## Task 5: Mapping the AI Attack Surface

Reconnaissance findings must be linked to demonstrate systemic compromise. At Cyphira, the attack chain flows across trusted boundaries:

```
[ Compromised Jupyter Notebook (10.10.45.20) ]
       │ (Cleartext MLflow credentials & Hugging Face token)
       ▼
[ MLflow Tracking & Registry (10.10.45.12) ]
       │ (Model version metadata & S3 artifact storage paths)
       ▼
[ MinIO / S3 Object Storage (10.10.45.22) ]
       │ (Unprotected model weights & serialization files)
       ▼
[ Production Triton Serving (10.10.45.15) ]
         (Serving fraud detection & RAG embeddings)
```

### MITRE ATLAS Technique Mapping
- **Active Scanning (`AML.T0006`)**: Network scanning and port discovery targeting AI services.
- **Discover ML Artifacts (`AML.T0007`)**: Enumerating model registries and artifact URIs.
- **ML Supply Chain Compromise (`AML.T0010`)**: Exposed Hugging Face tokens and third-party base model dependencies.
- **Discover ML Model Family (`AML.T0014`)**: Fingerprinting model formats and tensor configurations.
- **Reconnaissance Tactic (`AML.TA0002`)**: The overarching adversary tactic encompassing all discovery activity.

### Task 5 Questions & Answers

1. **The Cyphira Jupyter notebook at 10.10.45.20 contains a Hugging Face token (hf_kR7mXpQvL9nJwT2yBcDfAeGh8iKlMnOp). The internal-kb-embedder model on MLflow references sentence-transformers/all-MiniLM-L6-v2 as its base model. What ATLAS technique ID covers the risk of these exposed supply chain dependencies?**  
   *Answer:* `AML.T0010`

2. **You scanned the Cyphira subnet with nmap, probed endpoints with curl, and extracted metadata from MLflow APIs. All of these activities fall under one overarching ATLAS tactic. What is its ID?**  
   *Answer:* `AML.TA0002`

---

## Task 6: Structured Reconnaissance Methodology and Detection

### Five-Phase Reconnaissance Methodology
1. **Passive Reconnaissance**: Shodan/Censys queries, GitHub dorks for `HF_TOKEN` and `MLFLOW_TRACKING_URI`, arXiv publication reviews.
2. **Active Scanning**: Multi-port Nmap scans against AI ports (5000, 6333, 8000-8002, 8080, 8265, 8888, 9000, 11434).
3. **API Fingerprinting**: Header checks (`NV-Status`), error-inducing payloads, and gRPC reflection dumps.
4. **Metadata Extraction**: Scripted API queries to pull experiment runs, model schemas, and notebook credentials.
5. **Supply Chain Review**: Identifying external model dependencies, public artifact buckets, and internal package names susceptible to dependency confusion.

### SOC & SIEM Detection Signatures
From the defender's perspective in the SIEM:
- **Automated MLflow Harvesting**: Rapid sequential calls to `/api/2.0/mlflow/registered-models/list` and `/api/2.0/mlflow/model-versions/search` without UI session cookies directly match the signature of **MLOKit** (IBM X-Force Red's ML reconnaissance tool).
- **Out-of-CIDR Prometheus Scraping**: External queries to `/metrics` on ports 8002 or 8082 indicate unauthorized topology enumeration.
- **Anomalous Jupyter Ingestion**: Direct queries to `/api/kernels` or `/api/contents` without session authentication.

### Immediate Hardening Quick Wins
1. **Enable MLflow authentication**: Configure basic authentication or place MLflow behind an authenticating reverse proxy.
2. **Bind notebooks securely**: Never run Jupyter with `--ip=0.0.0.0` or `--allow-root` without token enforcement and ingress access controls.
3. **Perimeter isolation**: Block ports 5000, 6333, 8000-8002, 8080, 8265, 8888, and 9000 at the external firewall.
4. **Isolate Prometheus metrics**: Restrict ports 8002 and 8082 strictly to the monitoring CIDR.

### Task 6 Questions & Answers

1. **A SIEM log shows requests to /api/2.0/mlflow/registered-models/list from an IP with no corresponding MLflow UI session. What tool's access pattern does this match?**  
   *Answer:* `MLOKit`

2. **What is the single most effective quick win for preventing unauthenticated access to the MLflow tracking server?**  
   *Answer:* `Enable MLflow authentication`

---

## Task 7: Conclusion

AI components cannot be protected if security teams do not know they exist. Standard network vulnerability assessments routinely overlook AI platforms because they operate on non-traditional ports and speak specialized protocols.

By applying structured reconnaissance: scanning the AI port matrix, analyzing unique headers like `NV-Status`, leveraging gRPC reflection, and interrogating tracking servers, defenders can discover shadow AI deployments and eliminate critical misconfigurations before adversaries exploit them.

### Task 7 Questions & Answers

1. **All done!**  
   *Answer:* `No answer needed`

---

## Summary of Questions, Answers, and Flags

| Task | Question | Answer / Flag |
| :--- | :--- | :--- |
| **Task 1: Introduction** | I understand the learning objectives and am ready to learn about AI system reconnaissance! | `No answer needed` |
| **Task 2: AI Infrastructure Stack** | What is the IP address of the host running an HTTP service on port 8888 in your scan results? | `10.10.45.20` |
| **Task 2: AI Infrastructure Stack** | Which port does MLflow Tracking Server run on by default? | `5000` |
| **Task 3: Fingerprinting AI Services** | Which unique HTTP response header does the service on 10.10.45.15:8000 return to identify as an NVIDIA product? | `NV-Status` |
| **Task 3: Fingerprinting AI Services** | When you run grpcurl against 10.10.45.15:8001, what is the name of the inference service listed in the reflection output? | `inference.GRPCInferenceService` |
| **Task 4: Enumerating AI Systems** | What MLflow REST API endpoint would you use to retrieve the artifact storage location for a specific model version? | `/api/2.0/mlflow/model-versions/search` |
| **Task 4: Enumerating AI Systems** | What is the cleartext password for the MLflow service account stored in the Jupyter notebook on 10.10.45.20? | `Cyphira-MLflow-2024!` |
| **Task 5: Mapping Attack Surface** | The Cyphira Jupyter notebook at 10.10.45.20 contains a Hugging Face token... What ATLAS technique ID covers the risk of these exposed supply chain dependencies? | `AML.T0010` |
| **Task 5: Mapping Attack Surface** | You scanned the Cyphira subnet with nmap, probed endpoints with curl... All of these activities fall under one overarching ATLAS tactic. What is its ID? | `AML.TA0002` |
| **Task 6: Methodology & Detection** | A SIEM log shows requests to /api/2.0/mlflow/registered-models/list from an IP with no corresponding MLflow UI session. What tool's access pattern does this match? | `MLOKit` |
| **Task 6: Methodology & Detection** | What is the single most effective quick win for preventing unauthenticated access to the MLflow tracking server? | `Enable MLflow authentication` |
| **Task 7: Conclusion** | All done! | `No answer needed` |

---

_Happy Hacking!_  
**Muhammed Omer**

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
