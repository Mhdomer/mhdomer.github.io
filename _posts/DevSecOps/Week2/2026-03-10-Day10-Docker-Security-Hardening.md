---
layout: post
title: "Week 2 — Day 10: Docker Security Hardening"
date: 2026-03-10 10:00:00 +0800
categories:
  - DevSecOps
  - Week2
tags:
  - Docker
  - ContainerSecurity
  - DevSecOps
  - Hardening
author: muhammed
description: A full walkthrough of Docker security hardening — non-root users, read-only filesystems, dropped capabilities, multi-stage builds, minimal base images, and Docker Bench for Security.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fmedia.licdn.com%2Fdms%2Fimage%2Fv2%2FD4D12AQFW5-A1-LtP5Q%2Farticle-cover_image-shrink_600_2000%2Farticle-cover_image-shrink_600_2000%2F0%2F1685551394716%3Fe%3D2147483647%26v%3Dbeta%26t%3DuBVY-BcwJZQ3Ak83Ptf8kv_FLaezOB8hDv6VwCjV6E4&f=1&nofb=1&ipt=78e8662001c0c78b7b34cc4d36fcf501177a5736d2318e188ba746af450d7952
---

## The Container Security Problem

Containers share the host kernel. A misconfigured container running as root can potentially escape to the host, access other containers' data, or be leveraged to pivot across your infrastructure. Docker has strong isolation by default, but defaults are not hardened — you have to opt in.

---

## Principle 1 — Never Run as Root

By default, processes inside a Docker container run as root (UID 0). If an attacker exploits your app and escapes the container, they land as root on the host.

**Fix: create a non-root user in the Dockerfile.**

```dockerfile
FROM node:20-alpine

# Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Switch to non-root before the final CMD
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

**Verify:**
```bash
docker run --rm myapp whoami
# Should print: appuser (not root)
```

> `[SCREENSHOT]` — *Terminal showing docker run whoami returning "appuser" not "root"*

**For existing images you can't modify:**
```bash
docker run --user 1001:1001 nginx
```

---

## Principle 2 — Read-Only Filesystem

A writable root filesystem lets malware write new binaries, modify configs, or install persistence mechanisms. Make it read-only:

```bash
docker run --read-only myapp
```

If your app genuinely needs to write (logs, temp files), use targeted tmpfs mounts:

```bash
docker run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  myapp
```

> `[SCREENSHOT]` — *Terminal showing docker run --read-only failing to write to /app/data (permission denied) but succeeding with --tmpfs /tmp mounted*

**In Docker Compose:**
```yaml
services:
  app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
```

---

## Principle 3 — Drop Capabilities

Linux capabilities break root's all-or-nothing power into individual privileges. Docker containers start with a default set of ~14 capabilities. Drop everything not needed.

**Drop all, add back only what's required:**
```bash
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  myapp
```

`NET_BIND_SERVICE` allows binding to ports below 1024. Most apps don't even need this if you use ports above 1024.

**Common capabilities and when to use them:**

| Capability | Needed for |
|-----------|-----------|
| `NET_BIND_SERVICE` | Binding to ports < 1024 |
| `CHOWN` | Changing file ownership |
| `DAC_OVERRIDE` | Bypassing file permissions |
| `SETUID` / `SETGID` | Changing user/group ID |
| `SYS_PTRACE` | Debugging with ptrace |

> `[SCREENSHOT]` — *Terminal showing docker run --cap-drop ALL --cap-add NET_BIND_SERVICE working correctly, vs a second run with no --cap-add failing when trying to bind port 80*

**In Docker Compose:**
```yaml
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

---

## Principle 4 — No Privileged Mode

Never run `--privileged` in production. Privileged mode gives the container almost all Linux capabilities and mounts the host's devices — effectively bypassing container isolation.

```bash
# NEVER in production:
docker run --privileged myapp

# Also avoid:
docker run --pid=host myapp      # shares host PID namespace
docker run --network=host myapp  # shares host network namespace
docker run -v /:/host myapp      # mounts host root filesystem
```

---

## Principle 5 — Minimal Base Images

Smaller images have fewer packages, fewer CVEs, and a smaller attack surface.

| Base Image | Size | Use When |
|-----------|------|---------|
| `scratch` | 0 MB | Statically compiled Go binaries |
| `alpine` | ~5 MB | General purpose |
| `distroless` | ~20 MB | Production — no shell, no package manager |
| `slim` variants | varies | Node, Python apps needing some system libs |

**Alpine example:**
```dockerfile
FROM python:3.12-alpine

RUN apk add --no-cache gcc musl-dev
RUN pip install --no-cache-dir -r requirements.txt
```

**Distroless (no shell = harder to exploit):**
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt -t /app/deps

FROM gcr.io/distroless/python3
COPY --from=builder /app/deps /app/deps
COPY . /app
ENV PYTHONPATH=/app/deps
CMD ["/app/server.py"]
```

> `[SCREENSHOT]` — *Terminal showing docker images listing showing the size difference between python:3.12 (~1GB), python:3.12-slim (~130MB), and python:3.12-alpine (~50MB)*

---

## Principle 6 — Multi-Stage Builds

Multi-stage builds keep build tools out of the final image — compilers, test frameworks, and dev dependencies don't belong in production.

```dockerfile
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                  # installs ALL deps including devDependencies
COPY . .
RUN npm run build           # compile TypeScript, bundle, etc.

# Stage 2: Production
FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production   # only production deps
COPY --from=builder /app/dist ./dist   # only the built output

USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

The final image contains only the Alpine Node runtime, production deps, and compiled output. The build tools never make it in.

> `[SCREENSHOT]` — *Terminal showing docker build output with two stages completing, followed by docker images showing the final image size is much smaller than a single-stage build*

---

## Principle 7 — No Secrets in Images

Never put secrets in Dockerfiles or image layers — they're permanent in the image history.

```dockerfile
# WRONG — this is stored in every layer and visible in docker history
ENV API_KEY=supersecret123
RUN curl -H "Authorization: $API_KEY" https://api.example.com

# WRONG — ARG values appear in docker history too
ARG API_KEY
RUN curl -H "Authorization: $API_KEY" https://api.example.com
```

**Right approach:** Use runtime secrets injection (environment variables from Secrets Manager, Vault, or Docker secrets).

```bash
# Runtime injection — never baked into the image
docker run -e API_KEY=$(aws secretsmanager get-secret-value ...) myapp
```

Check your image history for leaked secrets:
```bash
docker history --no-trunc myapp | grep -i "secret\|password\|key\|token"
```

> `[SCREENSHOT]` — *Terminal showing docker history --no-trunc output on a clean image with no secrets visible in the layer commands*

---

## Docker Bench for Security

Docker Bench is an automated script that checks your Docker host and containers against CIS Docker Benchmark.

```bash
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /lib/systemd/system:/lib/systemd/system:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security
```

> `[SCREENSHOT]` — *Terminal showing Docker Bench output with PASS (green), WARN (yellow), and INFO lines — showing checks like "Ensure a user for the container has been created" and "Ensure the container's root filesystem is mounted as read only"*

Each check is marked:
- `[PASS]` — compliant
- `[WARN]` — needs attention
- `[INFO]` — informational
- `[NOTE]` — not applicable

Work through the `[WARN]` items and fix them one by one.

---

## Hardened Dockerfile Template

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only production artifacts
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist

# Set ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root
USER appuser

# Don't expose unnecessary env vars
ENV NODE_ENV=production

EXPOSE 3000

# Use exec form (no shell wrapper)
CMD ["node", "dist/server.js"]
```

Run it with:
```bash
docker run \
  --read-only \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --tmpfs /tmp \
  --security-opt no-new-privileges \
  myapp
```

`--security-opt no-new-privileges` prevents the process from gaining new privileges via `setuid` binaries inside the container.

---

## Key Takeaways

- Non-root user is the single most impactful change — do this in every Dockerfile
- Read-only filesystem + tmpfs for writable paths limits what an attacker can do post-exploitation
- Drop ALL capabilities and add back only what's needed
- Multi-stage builds keep build tools out of production images
- Never put secrets in Dockerfiles — they end up in image layers permanently
- Run Docker Bench to get a full checklist of what needs fixing on your host

---

## References

<div class="references">
<ul>
  <li><a href="https://docs.docker.com/engine/security/" target="_blank">Docker Security Documentation</a></li>
  <li><a href="https://github.com/docker/docker-bench-security" target="_blank">Docker Bench for Security</a></li>
  <li><a href="https://github.com/GoogleContainerTools/distroless" target="_blank">Google Distroless Images</a></li>
  <li><a href="https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html" target="_blank">OWASP Docker Security Cheat Sheet</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
