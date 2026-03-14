---
layout: post
title: AWS Load Balancers — ALB, NLB, Target Groups, and Listeners
date: 2026-05-16T10:00:00
categories:
  - AWS Learning Path
tags:
  - aws
  - alb
  - nlb
  - networking
  - cloud
author: muhammed
description: A full walkthrough of AWS Elastic Load Balancing — ALB vs NLB, listeners, target groups, health checks, sticky sessions, and SSL termination
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## What is Elastic Load Balancing?

**Elastic Load Balancing (ELB)** automatically distributes incoming traffic across multiple targets — EC2 instances, containers, Lambda functions, or IP addresses — in one or more Availability Zones.
It is the standard way to make applications highly available and scalable on AWS.

If one target becomes unhealthy or an entire AZ goes down, the load balancer stops sending traffic there automatically.
When capacity scales up, the load balancer starts sending traffic to new targets immediately.

AWS has three types of load balancers:

| Type | Layer | Protocol | Use Case |
|---|---|---|---|
| **Application Load Balancer (ALB)** | 7 (HTTP) | HTTP, HTTPS, WebSocket | Web apps, APIs, microservices |
| **Network Load Balancer (NLB)** | 4 (TCP) | TCP, UDP, TLS | High performance, static IP, gaming, VoIP |
| **Gateway Load Balancer (GWLB)** | 3 (IP) | All IP traffic | Third-party network appliances (firewalls, IDS) |

The old **Classic Load Balancer** is deprecated — do not use it for new workloads.

---

## Application Load Balancer (ALB)

An ALB operates at **Layer 7** — it understands HTTP/HTTPS.
It can make routing decisions based on the request content: URL path, hostname, headers, query strings, and HTTP method.
This is what makes it ideal for microservices — one ALB can route `/api/users` to one service and `/api/orders` to another.

### Key features

- **Content-based routing** — route by path, host, header, query string, method
- **SSL/TLS termination** — decrypts HTTPS at the ALB, forwards HTTP to targets (or HTTPS)
- **WebSocket support** — long-lived connections for real-time apps
- **HTTP/2 support** — between client and ALB (ALB → target is HTTP/1.1)
- **Sticky sessions** — route a user's requests to the same target (cookie-based)
- **User authentication** — integrate with Cognito or any OIDC provider before forwarding
- **Lambda targets** — invoke a Lambda function from HTTP traffic

### Creating an ALB

> 📸 **SCREENSHOT:** EC2 → Load Balancers → Create Load Balancer → Application Load Balancer selected.
> Show the scheme options (Internet-facing vs Internal), VPC selection, and the AZ/subnet mapping section.

```bash
# Create an internet-facing ALB
aws elbv2 create-load-balancer \
  --name prod-alb \
  --type application \
  --scheme internet-facing \
  --subnets subnet-public-1a subnet-public-1b subnet-public-1c \
  --security-groups sg-alb

# List load balancers
aws elbv2 describe-load-balancers --output table
```

**Scheme:**
- **Internet-facing** — has a public DNS name, receives traffic from the internet
- **Internal** — only reachable within the VPC, used between tiers (e.g. frontend ALB → backend ALB)

---

## Target Groups

A **target group** is a collection of targets that receive traffic from a load balancer.
Targets can be EC2 instances, IP addresses, Lambda functions, or another ALB.
Each target group has its own health check configuration.

### Target types

| Type | Use Case |
|---|---|
| `instance` | EC2 instances by instance ID |
| `ip` | Any IP address — EC2, ECS tasks (awsvpc), on-premises |
| `lambda` | A single Lambda function |
| `alb` | Another ALB (for NLB → ALB chaining) |

> 📸 **SCREENSHOT:** EC2 → Target Groups → Create Target Group.
> Show the target type selection (Instance / IP / Lambda / ALB), protocol and port fields, and the VPC dropdown.

```bash
# Create a target group
aws elbv2 create-target-group \
  --name prod-app-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0abc123 \
  --target-type instance \
  --health-check-path /health \
  --health-check-interval-seconds 30

# Register targets (EC2 instances)
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/prod-app-tg/abc123 \
  --targets Id=i-0abc123 Id=i-0def456

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/prod-app-tg/abc123
```

### Health Checks

The load balancer sends periodic requests to each target to check if it is healthy.
Unhealthy targets are removed from rotation until they recover.

> 📸 **SCREENSHOT:** EC2 → Target Groups → select a group → Health Checks tab.
> Show the health check path (/health), interval, threshold counts, and the success codes (200).

| Setting | Default | Meaning |
|---|---|---|
| Protocol | HTTP | Protocol for health check requests |
| Path | `/` | URL path to check |
| Healthy threshold | 5 | Consecutive successes to mark healthy |
| Unhealthy threshold | 2 | Consecutive failures to mark unhealthy |
| Timeout | 5s | How long to wait for a response |
| Interval | 30s | Time between checks |
| Success codes | 200 | HTTP status codes that count as healthy |

Best practice: create a dedicated `/health` endpoint in your app that checks database connectivity and returns 200 only when everything is working.

---

## Listeners and Rules

A **listener** is a process that checks for connection requests using a configured protocol and port.
An **ALB rule** determines what to do with requests that match conditions.

### Listeners

```
Listener 1: HTTP:80  → redirect to HTTPS (301)
Listener 2: HTTPS:443 → forward to target group (default rule)
```

> 📸 **SCREENSHOT:** EC2 → Load Balancers → select ALB → Listeners tab.
> Show two listeners (HTTP:80 and HTTPS:443) and the default action for each.

```bash
# Create an HTTPS listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/prod-alb/abc123 \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:...:certificate/abc-123 \
  --default-actions Type=forward,TargetGroupArn=arn:...:targetgroup/prod-app-tg/abc123

# Create HTTP → HTTPS redirect listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/prod-alb/abc123 \
  --protocol HTTP \
  --port 80 \
  --default-actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }]'
```

### Listener Rules

Rules let you route to different target groups based on request content.
Rules are evaluated in priority order (lowest number first).
The default rule (priority `*`) matches everything else.

> 📸 **SCREENSHOT:** EC2 → Load Balancers → select ALB → Listeners tab → View/edit rules.
> Show the rules editor with multiple rules — one matching `/api/*` forwarding to api-tg, one matching `/admin/*` requiring authentication, and the default rule forwarding to web-tg.

**Example routing setup:**

```
IF path is /api/*       → forward to api-target-group
IF path is /admin/*     → authenticate via Cognito, then forward
IF host is api.myapp.com → forward to api-target-group
IF header X-Version = v2 → forward to v2-target-group
DEFAULT                 → forward to web-target-group
```

```bash
# Add a path-based routing rule
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/prod-alb/abc/xyz \
  --priority 10 \
  --conditions '[{"Field":"path-pattern","Values":["/api/*"]}]' \
  --actions '[{"Type":"forward","TargetGroupArn":"arn:...:targetgroup/api-tg/abc"}]'
```

---

## Network Load Balancer (NLB)

An NLB operates at **Layer 4** — it works with raw TCP/UDP packets.
It does not read HTTP headers or understand URLs.
What it offers instead is extreme performance and static IP addresses.

### Key features

- **Ultra-low latency** — millions of requests per second, sub-millisecond latency
- **Static IP per AZ** — each AZ gets a fixed Elastic IP, useful for firewall whitelisting
- **Preserves client IP** — the target sees the original client IP address
- **TLS passthrough or termination** — can terminate TLS or pass it to targets
- **Cross-zone load balancing** — disabled by default (costs money when enabled)
- **PrivateLink** — expose your service to other VPCs without VPC peering

### When to use NLB over ALB

| Use NLB when | Use ALB when |
|---|---|
| You need static IP addresses | You need path/host-based routing |
| Protocol is TCP/UDP (not HTTP) | Protocol is HTTP/HTTPS |
| Extreme performance is needed | You want native HTTP features |
| You need to preserve client IP at Layer 4 | You need user authentication |
| Exposing service via AWS PrivateLink | Routing microservices |

```bash
# Create an NLB
aws elbv2 create-load-balancer \
  --name prod-nlb \
  --type network \
  --scheme internet-facing \
  --subnets subnet-public-1a subnet-public-1b
```

---

## SSL/TLS Termination

The ALB handles the SSL handshake and decrypts HTTPS traffic.
Your backend targets receive plain HTTP — they do not need to handle SSL.
This reduces CPU overhead on your application servers.

### ACM Integration

Use **AWS Certificate Manager (ACM)** to provision free SSL certificates.
ACM automatically renews them before they expire.

> 📸 **SCREENSHOT:** ACM → Request Certificate → enter domain names.
> Show the domain validation method (DNS validation recommended) and the CNAME record that ACM creates in Route 53 for validation.

```bash
# Request a certificate
aws acm request-certificate \
  --domain-name myapp.com \
  --subject-alternative-names *.myapp.com \
  --validation-method DNS \
  --region eu-west-1

# List certificates
aws acm list-certificates --output table
```

Once issued, attach the certificate ARN to your ALB HTTPS listener.

### End-to-end HTTPS

For sensitive workloads, you may want HTTPS all the way to the target:

```
Client → HTTPS → ALB (SSL termination) → HTTPS → Target
```

Configure the target group protocol as HTTPS and install a certificate on your application servers.
The certificate on the targets can be self-signed or from a private CA — the ALB does not validate it by default (but you can enable validation).

---

## Sticky Sessions

**Sticky sessions** (session affinity) ensure a user's requests always go to the same target.
Used for applications that store session state on the server (not recommended — use Redis instead, but sometimes you need it for legacy apps).

| Stickiness type | Cookie name | Duration |
|---|---|---|
| Load balancer generated | `AWSALB` | 1 second to 7 days |
| Application-based | Custom cookie name | Set by your app |

> 📸 **SCREENSHOT:** EC2 → Target Groups → select group → Attributes tab.
> Show the Stickiness toggle enabled, stickiness type (Load balancer generated), and duration field.

---

## Access Logs

ALB and NLB can log every request to S3.
Access logs include: timestamp, client IP, request, response, target, latency, user agent.
Useful for debugging, security analysis, and compliance.

```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-alb-logs \
    Key=access_logs.s3.prefix,Value=prod-alb
```

---

## Common Architecture

```
Internet
    │
    ▼
[Security Group: allow 80/443 from 0.0.0.0/0]
Application Load Balancer (public subnets, multi-AZ)
    │
    ├── /api/*  → API Target Group
    │              [Security Group: allow 8080 from ALB SG only]
    │              EC2 instances in private subnets
    │
    └── /*      → Web Target Group
                   [Security Group: allow 3000 from ALB SG only]
                   EC2 instances in private subnets
```

The ALB sits in public subnets and is the only resource that accepts public traffic.
The EC2 instances sit in private subnets and only accept traffic from the ALB's security group.

---

## Quick Reference

```bash
# Load balancers
aws elbv2 describe-load-balancers --output table

# Target groups
aws elbv2 describe-target-groups --output table

# Target health
aws elbv2 describe-target-health --target-group-arn ARN

# Listeners
aws elbv2 describe-listeners --load-balancer-arn ARN

# Rules
aws elbv2 describe-rules --listener-arn ARN

# Register/deregister targets
aws elbv2 register-targets --target-group-arn ARN --targets Id=i-xxx
aws elbv2 deregister-targets --target-group-arn ARN --targets Id=i-xxx
```
