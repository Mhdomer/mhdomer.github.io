---
layout: post
title: "TryHackMe: The Building Blocks of AI"
date: 2026-09-16T19:45:00
categories:
  - TryHackMe
  - AI Security
tags:
  - thm
  - ai
  - machine-learning
  - deep-learning
  - llm
  - writeup
author: muhammed
description: A complete walkthrough of the TryHackMe room The Building Blocks of AI, covering the ML lifecycle, learning paradigms, neural networks, and transformer-based LLMs.
toc: true
pin: false
math: false
image: https://cdn-images.tryhackme.com/room-icons/6228f0d4ca8e57005149c3e3-1784893485639
---

## Overview

[The Building Blocks of AI](https://tryhackme.com/r/room/thebuildingblocksofai) is the entry point for TryHackMe's AI Security path. Before getting into prompt injection, model poisoning, or adversarial attacks, this room establishes the baseline mechanics: what machine learning actually does, how neural networks process features, and why transformers made modern LLMs possible.

Instead of deploying a Linux target VM, this room uses TryHackMe's embedded AI Agent platform in a side panel for interactive challenges.

---

## Task 1: Introduction and the THM AI Agent Platform

The first task introduces the sandbox environment. Instead of interacting with a target machine via SSH or an AttackBox web terminal, you interact directly with an embedded agent panel on the right side of the screen.

Key details about the agent interface:
* The agent has a defined system prompt, constraints, and specific goals depending on the task.
* In some tasks, the agent collaborates with you. In later challenges, it is programmed to resist revealing information.
* Prompt phrasing directly affects output quality and triggers.

The sandbox in Task 1 has no flag or objective. It is only there to verify that the chat interface loads and responds properly before starting the graded tasks.

---

## Task 2: AI vs Machine Learning and the ML Lifecycle

Task 2 breaks down the distinction between Artificial Intelligence and Machine Learning:

* **Artificial Intelligence (AI):** The broad field of computer science focused on creating machines capable of performing tasks that typically require human reasoning, decision-making, or problem-solving. Research began in the 1950s.
* **Machine Learning (ML):** A subfield of AI where systems learn to recognize patterns and make predictions from data without explicit, line-by-line programming.

```
Problem Definition → Data Collection & Cleaning → Model Training → Evaluation & Tuning → Deployment → Monitoring & Retraining
```

### The ML Lifecycle

1. **Problem Definition:** Deciding the concrete objective (for example, binary spam classification).
2. **Data Preparation:** Collecting, cleaning, and formatting datasets.
3. **Training:** Feeding data to the algorithm to adjust internal mathematical weights.
4. **Evaluation and Tuning:** Testing performance against unseen validation data and tuning hyperparameters.
5. **Deployment:** Serving the model via an API or embedding it into an application.
6. **Monitoring and Retraining:** Watching for data drift in production and updating the model periodically.

### Overfitting

A key risk highlighted in this task is **overfitting**. This occurs when a model learns the training data too closely, effectively memorizing the exact examples and background noise instead of extracting general patterns. When evaluated on fresh, unseen data, performance collapses.

### Task 2 Questions and Answers

* **Question:** What is the term for when a model becomes too familiar with its training data and fails to generalise to new data?  
  **Answer:** `Overfitting`
* **Question:** What is the subfield of AI that enables systems to learn from data without being explicitly programmed?  
  **Answer:** `Machine Learning`

---

## Task 3: The Brains of the Operation (ML Algorithms)

An ML algorithm is the mathematical method used to extract patterns from data. The finished, saved output of that training process is the **model**.

Every training loop relies on three components:
1. **Decision Process:** Makes a prediction based on input variables.
2. **Error Function (Loss Function):** Measures the distance between the model prediction and the true target value.
3. **Optimization Process:** Adjusts weights and parameters to minimize that error on the next iteration.

### The Four Learning Paradigms

| Paradigm | Training Data | Core Mechanism | Real-World Use Case |
|---|---|---|---|
| **Supervised Learning** | Fully labeled inputs and outputs | Maps inputs to known answers | Spam filtering, house price prediction |
| **Unsupervised Learning** | Unlabeled data | Finds intrinsic clusters and patterns on its own | Customer segmentation, network anomaly detection |
| **Semi-supervised Learning** | Small labeled dataset + large unlabeled pool | Uses labeled samples to guide broader clustering | Medical imaging analysis where labeling is expensive |
| **Reinforcement Learning** | Environment feedback (rewards / penalties) | Agent takes actions to maximize cumulative reward | Autonomous driving, game-playing engines (AlphaGo) |

### Mission Briefing Challenge

Task 3 requires opening the agent panel to complete four covert scenario assessments. For each scenario, you specify which learning paradigm fits and provide a brief one-sentence justification.

Once all four scenarios are classified correctly, the agent reveals the task flag.

### Task 3 Questions, Answers, and Flag

* **Question:** Which category of ML algorithm learns by receiving rewards or penalties based on actions taken in an environment?  
  **Answer:** `Reinforcement learning`
* **Question:** Which category of ML algorithm uses a small amount of labelled data to guide learning across a larger unlabelled dataset?  
  **Answer:** `Semi-supervised learning`
* **Question:** What's the flag?  
  **Answer:** `THM{4lg0r1thm_4g3nt}`

---

## Task 4: Neural Networks and Deep Learning

Task 4 covers how artificial neural networks draw architectural inspiration from biological nervous systems:

* **Nodes (Neurons):** Individual compute units that receive inputs, multiply them by assigned weights, sum them up, apply an activation function, and pass the result forward.
* **Connections (Synapses):** The links routing data between nodes, each carrying a specific weight.

```
[Input Layer] ──> [Hidden Layer 1] ──> [Hidden Layer 2] ──> [Output Layer]
 (Raw Data)       (Edges / Curves)     (Complex Shapes)     (Classification)
```

### Layer Structure

1. **Input Layer:** Ingests raw features. For example, a 4x4 grayscale image passes into 16 input nodes (one per pixel).
2. **Hidden Layers:** Extract features with increasing abstraction. Early layers might detect lines and edges, while deeper layers assemble those into recognizable objects.
3. **Output Layer:** Outputs the final prediction scores or class probabilities.

### Deep Learning vs Traditional ML

When a neural network contains **more than three layers** (including input and output), it is classified as **Deep Learning (DL)**.

The primary operational difference:
* Traditional supervised ML requires manual feature engineering, where humans identify which properties matter before feeding them into an algorithm.
* Deep Learning discovers relevant features automatically from raw, unstructured data (such as raw image pixels or audio waveforms), making it significantly more scalable given enough compute and data.

### NEURON-1 Interactive Challenge

In this task, the agent acts as **NEURON-1**, an uninitialized neural network. You pick a real-world object or data source (for example, a suspicious firewall log entry or an animal) and walk through three stages manually:
1. **Input Layer:** Define the raw input features.
2. **Hidden Layer:** Combine those features into intermediate pattern representations.
3. **Output Layer:** Produce the final classification based on the extracted patterns.

Walking NEURON-1 through all three layers completes the training cycle and yields the flag.

### Task 4 Questions, Answers, and Flag

* **Question:** What is the first layer in a neural network that receives raw input data?  
  **Answer:** `Input layer`
* **Question:** What term describes the weighted connections between nodes in a neural network?  
  **Answer:** `Synapses`
* **Question:** What's the flag?  
  **Answer:** `THM{n3ur0n_1_0nl1n3}`

---

## Task 5: Large Language Models (LLMs)

Task 5 covers the underlying architecture that powers modern generative text tools:

### What an LLM Actually Does

At its core, an LLM is a deep learning model trained to predict the next word (or token) in a sequence. A coherent response is generated by running this prediction loop repeatedly, one token at a time.

### Training Mechanics

1. **Pre-training:** Models are exposed to massive text corpora (billions to trillions of tokens).
2. **Loss and Backpropagation:** During pre-training, text is fed into the network with words masked out. The model makes a guess. If the guess is wrong, the difference between its prediction and reality is calculated via a loss function, and the **backpropagation** algorithm adjusts the model's weights backward through the network.
3. **Parameters:** The learned internal weights that store the model's statistical representation of language.

### The Transformer Breakthrough (2017)

Prior language architectures (RNNs and LSTMs) analyzed text sequentially, making them slow to train on long documents and prone to forgetting context.

Google's 2017 paper *Attention Is All You Need* introduced the **Transformer** architecture. Two innovations changed text processing:
1. **Parallelization:** Transformers process entire sequences simultaneously rather than word by word, making it possible to train on massive GPU clusters.
2. **Self-Attention Mechanism:** Allows the model to calculate relationship scores between every word in a sequence regardless of distance. In the sentence *"The bank approved the loan because it was financially stable,"* attention allows the model to link *"it"* directly back to *"the bank"* rather than *"the loan"*.

### Post-Training Alignment (RLHF)

Raw pre-trained base models only predict what text naturally follows an input. To convert a raw next-token predictor into a reliable assistant, models undergo **Reinforcement Learning from Human Feedback (RLHF)**. Human reviewers grade model responses, penalizing unhelpful or unsafe outputs, which refines the final conversational behavior.

### Task 5 Questions and Answers

* **Question:** What type of neural network, introduced by Google in 2017, powers modern LLMs?  
  **Answer:** `Transformer neural networks`
* **Question:** What is the name of the process where humans review and flag model outputs to refine its behaviour after pre-training?  
  **Answer:** `RLHF`
* **Question:** What mechanism do transformer networks use to assign different levels of importance to different words in a sequence?  
  **Answer:** `Attention`
* **Question:** What algorithm is used to adjust a model's parameters based on the difference between its prediction and the correct answer?  
  **Answer:** `Backpropagation`

---

## Key Takeaways

1. **AI vs ML vs DL vs LLMs:**
   * AI is the broad goal (machines performing intelligent tasks).
   * ML is the statistical approach (learning from data).
   * DL is the scalable multi-layer architecture (automatic feature extraction).
   * LLMs are transformer-based deep learning models specialized in sequence prediction.
2. **Security Implications:** Every layer introduced here creates a distinct attack surface. Unsupervised clustering can be skewed by data poisoning. Neural network nodes can be fooled by adversarial examples. Transformer attention and next-token prediction can be subverted through prompt injection and context window manipulation.

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
