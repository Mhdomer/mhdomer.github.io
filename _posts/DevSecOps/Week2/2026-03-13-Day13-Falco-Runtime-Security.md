---
layout: post
title: "Day 13: Runtime Security with Falco - Kernel-Level Threat Detection"
date: 2026-03-13 10:00:00 +0800
categories:
  - DevSecOps
  - Week2
tags:
  - Falco
  - RuntimeSecurity
  - ContainerSecurity
  - Kubernetes
  - DevSecOps
author: muhammed
description: A practical walkthrough of Falco - kernel syscall-based runtime threat detection for containers and Kubernetes, writing custom detection rules, and alerting via Falcosidekick.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fmedia.licdn.com%2Fdms%2Fimage%2Fsync%2Fv2%2FD5627AQHhpbuHrWjt8g%2Farticleshare-shrink_800%2Farticleshare-shrink_800%2F0%2F1739776311104%3Fe%3D2147483647%26v%3Dbeta%26t%3DqkjvQasmKkDuzA0qXgLNx3QzuOu7lGCSB5dkbwyqUwI&f=1&nofb=1&ipt=65b9cf1c22e2e98d755ccdec4dcb035de897372776a00add99ffecd660869d90
---

## The Blind Spot in Container Security

Over the past few days, we covered build-time and deploy-time security:
- We hardened our Dockerfiles on Day 10 (non-root users, dropped capabilities).
- We scanned images for known CVEs using Trivy on Day 11.
- We locked down cluster identities and network access using Kubernetes RBAC and Network Policies on Day 12.

All of those controls are critical, but they share one major limitation: **they only check things before or as they are deployed.**

What happens when an attacker finds a brand-new zero-day vulnerability in your application runtime?

- An attacker sends an exploit to an Nginx container.
- The exploit succeeds, and the container suddenly spawns `/bin/sh`.
- The attacker attempts to read `/etc/shadow`, drops a crypto-miner binary into `/tmp`, or opens a socket connection to a Command & Control (C2) server.

Neither Trivy nor RBAC can see this happening. The container image was clean when scanned, and the pod was deployed legitimately.

This is why we need **Runtime Security**. And in the cloud native world, **Falco** is the undisputed standard.

---

## The Motion Sensor Analogy: How Falco Works

Think of your security stack like a high-security building:

- **Trivy / Inspector:** The building inspectors who review the architectural blueprints and check the materials before construction begins.
- **Kubernetes RBAC:** The electronic card readers at the building entrance deciding who gets through the lobby.
- **Falco:** The **infrared motion sensors and acoustic microphones** inside every room, listening for abnormal behavior 24/7!

Even if someone walked through the front door with a valid visitor badge (RBAC) and clean clothes (zero CVEs), the moment they start prying open an air vent with a crowbar, the motion sensor trips and sounds the alarm!

```
+-------------------------------------------------------------+
|                     Container / Application                 |
+-------------------------------------------------------------+
                              |
                     System Call (syscall)
           (e.g., execve "/bin/sh", openat "/etc/shadow")
                              |
                              v
+-------------------------------------------------------------+
|                      Linux Kernel                           |
|                                                             |
|   +-----------------------------------------------------+   |
|   |         eBPF Probe (Kernel Instrumentation)         |   |
|   +-----------------------------------------------------+   |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
|                 Falco Rule Engine (User Space)              |
|        Evaluates syscall against security rules             |
+-------------------------------------------------------------+
                              |
                              v
                    [ Alert: WARNING / CRITICAL ]
```

### What is a System Call (Syscall)?

Whenever any program inside a container wants to perform an action in the real world:
- Start a process (`execve`)
- Read or write a file (`openat`, `read`, `write`)
- Open a network socket (`socket`, `connect`)

It **cannot** do this on its own. It must ask the underlying Linux kernel via a **system call**.

Falco taps into this kernel crossroad using **eBPF (Extended Berkeley Packet Filter)**. It intercepts the syscall stream in real time, parses the arguments, enriches the event with Kubernetes metadata (pod name, namespace, container image), and checks it against a rules engine.

---

## Installing Falco on Kubernetes

The recommended method to deploy Falco across a Kubernetes cluster is using its official Helm chart with the **eBPF driver**:

### Step 1: Add the Helm Repository

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

### Step 2: Install with eBPF and Falcosidekick

```bash
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true
```

Falco deploys as a `DaemonSet`, meaning one Falco pod runs on every single worker node in your cluster, monitoring every container running on that host.

### Step 3: Verify Falco is Active

```bash
kubectl get pods -n falco
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

---

## Understanding Falco Rules

Falco rules are defined in YAML and follow a straightforward syntax:

```yaml
- rule: Shell Spawned in Production Container
  desc: Detects an interactive shell spawned inside a running workload
  condition: >
    spawned_process and
    container and
    shell_procs and
    k8s.ns.name = "production"
  output: >
    Interactive shell spawned in container
    (user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name
     container=%container.name image=%container.image.repository
     shell=%proc.name cmdline=%proc.cmdline)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```

### Deconstructing the Rule

- **`condition`**: A logical expression built using Falco fields and macros. Here, it checks if a new process was spawned, if it occurred inside a container, if the process is a known shell (`bash`, `sh`, `zsh`), and if it occurred in the `production` namespace.
- **`output`**: The formatted log string emitted when the condition matches. Notice the rich context: username, pod name, namespace, image tag, and the full command line executed!
- **`priority`**: Severity level (`EMERGENCY`, `ALERT`, `CRITICAL`, `ERROR`, `WARNING`, `NOTICE`, `INFO`, `DEBUG`).

---

## Default Detections Out of the Box

Falco ships with an extensive library of default rules. Some of the most critical built-in detections include:

| Default Rule Name | What It Catches |
| :--- | :--- |
| `Terminal shell in container` | Someone running `kubectl exec` or an attacker spawning a shell |
| `Write below binary dir` | An attacker attempting to modify `/bin`, `/sbin`, or `/usr/bin` |
| `Read sensitive file untrusted` | Unauthorized processes reading `/etc/shadow`, `/etc/sudoers` |
| `Contact K8S API Server From Container`| Workloads querying the internal Kubernetes API unexpectedly |
| `Crypto Mining Activity` | Known mining process names (`xmrig`, `minergate`) or mining pool ports |
| `Launch Privileged Container` | A new container launched with full root host capabilities |

---

## Triggering Real Falco Alerts (Hands-on)

Let's test Falco live to see how it catches suspicious activity:

### Test 1: Spawn a Shell Inside a Pod

In one terminal window, start streaming Falco logs:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

In a second terminal window, exec into any running workload:

```bash
kubectl exec -it <test-pod-name> -- /bin/sh
```

Within a fraction of a second, Falco outputs a warning alert:

```
14:22:01.129381023: Warning Interactive shell spawned in container (user=root pod=test-pod-684b9c-xyz ns=default container=web image=nginx:alpine shell=sh cmdline=sh)
```

### Test 2: Attempting to Read `/etc/shadow`

Inside that pod shell, run:

```bash
cat /etc/shadow
```

Falco catches the sensitive file access immediately:

```
14:22:15.908234120: Warning Sensitive file opened for reading by untrusted program (user=root file=/etc/shadow program=cat)
```

### Test 3: Attempting to Drop a Binary in `/bin`

Inside the container, try creating a file in the system binaries directory:

```bash
touch /bin/malware_payload
```

Falco detects the directory write anomaly:

```
14:22:30.450123991: Error File below /bin or /etc opened for writing (user=root file=/bin/malware_payload program=touch)
```

---

## Forwarding Alerts with Falcosidekick

By default, Falco prints alerts to stdout. In production, you need alerts sent to your team's chat channels and SIEM.

**Falcosidekick** acts as an event forwarder for Falco, supporting over 50 destinations including Slack, Discord, Microsoft Teams, AWS SNS, Datadog, Elasticsearch, and PagerDuty.

### Configuring Slack Integration via Helm

```bash
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK" \
  --set falcosidekick.config.slack.minimumpriority="warning"
```

Now, any Warning, Error, or Critical security event in your cluster pops up as an interactive card in your Slack security channel!

---

## Junior Pitfalls to Avoid

1. **Forgetting Node-Level Resource Limits:** Because Falco monitors every syscall on the host, a node with thousands of rapid file operations can cause high CPU usage if rules are poorly tuned. Always monitor Falco's CPU and memory usage in production.
2. **Alert Fatigue from CI/CD Runners:** CI/CD runner pods (like Jenkins or GitLab runners) constantly spawn shells and run build commands. Falco will alert on every single build step if you do not add an exception for your build namespaces.
3. **Running Falco on AWS Fargate:** AWS Fargate does not give access to the underlying EC2 host or Linux kernel. Falco cannot run as an eBPF daemon on Fargate; you must use EC2 managed node groups.
4. **Treating Shell Access as Normal in Production:** In healthy production environments, engineers should never need to `kubectl exec` into containers. Use ephemeral debug containers (`kubectl debug`) or rely on logs and observability metrics instead.

---

## Key Takeaways

- Trivy catches vulnerabilities at build time; Falco catches active exploits and intrusions at runtime.
- Falco uses eBPF to monitor Linux kernel system calls (`execve`, `openat`, `connect`) with near-zero overhead.
- Default Falco rules catch interactive shells, sensitive credential reads, binary tampering, and crypto-mining out of the box.
- Use Falcosidekick to route alerts to Slack, PagerDuty, or AWS SNS for rapid incident response.
- Combine build-time scanning, RBAC guardrails, and runtime detection for true defense-in-depth.

---

## References

<div class="references">
<ul>
  <li><a href="https://falco.org/docs/" target="_blank">Falco Official Documentation</a></li>
  <li><a href="https://github.com/falcosecurity/falco/tree/master/rules" target="_blank">Falco Core Detection Rules Repository</a></li>
  <li><a href="https://github.com/falcosecurity/falcosidekick" target="_blank">Falcosidekick Output Forwarder</a></li>
  <li><a href="https://ebpf.io/" target="_blank">Introduction to eBPF Technology</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
