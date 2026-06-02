---
layout: post
title: "Week 2 — Day 13: Runtime Security with Falco"
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
description: A full walkthrough of Falco — syscall-based runtime threat detection for containers and Kubernetes, writing custom rules, and forwarding alerts to Slack or a SIEM.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fmedia.licdn.com%2Fdms%2Fimage%2Fsync%2Fv2%2FD5627AQHhpbuHrWjt8g%2Farticleshare-shrink_800%2Farticleshare-shrink_800%2F0%2F1739776311104%3Fe%3D2147483647%26v%3Dbeta%26t%3DqkjvQasmKkDuzA0qXgLNx3QzuOu7lGCSB5dkbwyqUwI&f=1&nofb=1&ipt=65b9cf1c22e2e98d755ccdec4dcb035de897372776a00add99ffecd660869d90
---

## The Gap That Scanning Misses

Trivy scans images for known CVEs. RBAC restricts who can deploy what. But neither tells you what's happening **inside a running container right now**.

What if a container starts a shell? What if it reads `/etc/shadow`? What if it opens a network connection to an unusual IP? What if a cryptominer process spawns?

That's what **Falco** detects — suspicious behavior at the system call level, in real time.

---

## How Falco Works

Falco runs as a privileged daemonset on each Kubernetes node (or as a host process). It hooks into the Linux kernel via:
- **eBPF probe** — modern, recommended (no kernel module needed)
- **Kernel module** — traditional approach
- **`/proc` filesystem** — minimal, fallback option

Every system call made by every container on the node is evaluated against Falco's rule set. When a call matches a rule, Falco fires an alert.

> `[SCREENSHOT]` — *Falco architecture diagram or the Falco pods running: kubectl get pods -n falco showing a falco pod on each node (DaemonSet)*

---

## Installing Falco on Kubernetes

### Via Helm (recommended)

```bash
# Add the Falco Helm repo
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# Install Falco with eBPF driver
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true
```

> `[SCREENSHOT]` — *Terminal showing helm install falco completing successfully, followed by kubectl get pods -n falco showing falco and falcosidekick pods running*

### Verify Falco is Running

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco | head -30
```

> `[SCREENSHOT]` — *kubectl logs output showing Falco starting up, loading rules files, and printing "Falco initialized, ready to help"*

---

## Understanding Falco Rules

Rules are written in YAML and describe patterns to detect.

### Rule Structure

```yaml
- rule: Shell Spawned in Container
  desc: A shell was spawned in a container — could indicate an attacker exploring the environment
  condition: >
    spawned_process and
    container and
    shell_procs and
    not container.image.repository in (trusted_images)
  output: >
    Shell spawned in container
    (user=%user.name user_loginuid=%user.loginuid
     container=%container.name
     image=%container.image.repository:%container.image.tag
     shell=%proc.name parent=%proc.pname
     cmdline=%proc.cmdline)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```

**Key fields:**
| Field | Purpose |
|-------|---------|
| `condition` | Boolean expression using Falco fields and macros |
| `output` | Log message when the rule fires |
| `priority` | DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL, ALERT, EMERGENCY |
| `tags` | For categorization and filtering |

---

## Default Rules — What Falco Catches Out of the Box

Falco ships with a rich default ruleset. Key rules:

| Rule | What it detects |
|------|----------------|
| `Terminal shell in container` | Interactive shell started in a running container |
| `Write below binary dir` | Writing to `/bin`, `/sbin`, `/usr/bin` etc. in a container |
| `Read sensitive file untrusted` | Container reading `/etc/shadow`, `/etc/sudoers` etc. |
| `Contact K8S API Server From Container` | Container calling the Kubernetes API |
| `Launch Privileged Container` | A privileged container is started |
| `Outbound Connection to C2 Servers` | Connection to known C2 IP addresses |
| `Crypto Mining Activity` | Processes associated with crypto mining tools |
| `Modify binary dirs` | Files written to system binary directories |
| `Container Drift Detected` | A new executable is created in a running container (ELF file) |

---

## Triggering Rules — Live Demo

### Trigger "Terminal shell in container"

```bash
# In one terminal, exec into a running container
kubectl exec -it <pod-name> -- /bin/sh

# In another terminal, watch Falco logs
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

> `[SCREENSHOT]` — *kubectl logs -n falco showing a Falco alert firing: "Warning Terminal shell in container (user=root container=myapp image=myapp:latest shell=sh)" with timestamp*

### Trigger "Read sensitive file"

```bash
# Inside the container shell
cat /etc/shadow
```

> `[SCREENSHOT]` — *Falco log showing "Warning Sensitive file opened for reading (user=root file=/etc/shadow container=myapp)"*

### Trigger "Write below binary dir"

```bash
# Inside the container
touch /bin/malware
```

> `[SCREENSHOT]` — *Falco log showing "Error Write below binary dir (user=root file=/bin/malware container=myapp)"*

---

## Writing Custom Rules

### Detect Unusual Outbound Connections

```yaml
- rule: Unexpected Outbound Connection
  desc: A container is making an outbound connection to an unexpected destination
  condition: >
    outbound and
    container and
    not fd.sport in (80, 443, 5432, 6379) and
    not container.image.repository in (trusted_network_images)
  output: >
    Unexpected outbound connection from container
    (container=%container.name
     image=%container.image.repository
     connection=%fd.name
     proc=%proc.name)
  priority: WARNING
```

### Detect Crypto Mining by CPU Behavior

```yaml
- rule: Crypto Mining Process
  desc: Likely crypto mining activity detected
  condition: >
    spawned_process and
    container and
    proc.name in (xmrig, minergate, ethminer, cgminer, bfgminer)
  output: >
    Crypto mining process detected
    (container=%container.name
     image=%container.image.repository
     proc=%proc.name
     cmdline=%proc.cmdline)
  priority: CRITICAL
```

### Detect Container Drift (New Executables)

```yaml
- rule: New Executable Dropped in Container
  desc: A new ELF binary was created in a running container — potential malware drop
  condition: >
    container and
    evt.type=write and
    fd.typechar=f and
    (fd.name startswith /bin/ or fd.name startswith /usr/bin/ or fd.name startswith /tmp/) and
    spawned_process
  output: >
    New binary dropped in container
    (container=%container.name
     file=%fd.name
     user=%user.name)
  priority: CRITICAL
```

---

## Falco Fields Reference

Useful fields for writing conditions and output:

| Field | Value |
|-------|-------|
| `container.name` | Container name |
| `container.image.repository` | Image name |
| `container.image.tag` | Image tag |
| `proc.name` | Process name |
| `proc.cmdline` | Full command line |
| `proc.pname` | Parent process name |
| `user.name` | Username |
| `fd.name` | File/socket name |
| `fd.sport` | Source port |
| `fd.dport` | Destination port |
| `k8s.pod.name` | Kubernetes pod name |
| `k8s.ns.name` | Kubernetes namespace |

---

## Falco Sidekick — Forwarding Alerts

Falco outputs to stdout by default. **Falco Sidekick** routes alerts to external systems.

Supported outputs include: Slack, PagerDuty, Splunk, Elasticsearch, Teams, Datadog, AWS SNS, and 50+ others.

### Configure Slack Output

```bash
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/YOUR/WEBHOOK/URL" \
  --set falcosidekick.config.slack.minimumpriority="warning"
```

> `[SCREENSHOT]` — *Slack channel showing a Falco alert message with container name, image, rule name, and severity — formatted as a Slack message card*

### Falco Sidekick UI

```bash
# Port-forward the Sidekick UI
kubectl port-forward svc/falco-falcosidekick-ui 2802:2802 -n falco
# Open http://localhost:2802
```

> `[SCREENSHOT]` — *Falco Sidekick UI in browser showing a dashboard with alert counts by priority, a timeline of recent alerts, and a table of the latest events with rule names, containers, and priorities*

---

## Falco in AWS EKS

When running on EKS, some additional setup is needed since managed node groups don't allow kernel module installation. Use the eBPF driver:

```bash
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set ebpf.enabled=true
```

For Fargate (no node access), Falco cannot run — it requires access to the host kernel. Use Sysdig or AWS-native options instead.

---

## Lab — Install Falco and Trigger a Rule

1. Install Falco via Helm on your cluster:
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts && helm repo update
helm install falco falcosecurity/falco -n falco --create-namespace --set driver.kind=ebpf
```

2. Wait for the DaemonSet to be ready:
```bash
kubectl rollout status daemonset/falco -n falco
```

3. Start watching logs:
```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

4. In another terminal, exec into any running pod and open a shell:
```bash
kubectl exec -it <any-pod> -- /bin/sh
```

> `[SCREENSHOT]` — *Two terminals side by side: left showing kubectl exec into a pod, right showing the Falco log immediately printing the "Terminal shell in container" warning alert*

5. Inside the shell, try to read a sensitive file:
```bash
cat /etc/shadow 2>/dev/null || echo "file not found"
```

> `[SCREENSHOT]` — *Falco log showing the "Read sensitive file untrusted" alert firing with the container name and filename*

6. Exit the shell — confirm no more alerts fire after exiting

---

## Key Takeaways

- Falco detects what's happening inside containers right now — scanning tools can't do this
- The default ruleset covers the most common attack patterns — enable it and you're already ahead
- Write custom rules for your environment — unusual ports, specific process names, sensitive file paths
- Falco Sidekick routes alerts to your team's communication and SIEM tools
- Exec'ing into a production container should always fire a Falco alert — if it doesn't, something's wrong
- Combine Falco (runtime) + Trivy (build time) + RBAC (access control) for defense in depth

---

## References

<div class="references">
<ul>
  <li><a href="https://falco.org/docs/" target="_blank">Falco Documentation</a></li>
  <li><a href="https://github.com/falcosecurity/falco/tree/master/rules" target="_blank">Falco Default Rules</a></li>
  <li><a href="https://github.com/falcosecurity/falcosidekick" target="_blank">Falco Sidekick</a></li>
  <li><a href="https://falco.org/docs/reference/rules/supported-fields/" target="_blank">Falco Supported Fields</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
