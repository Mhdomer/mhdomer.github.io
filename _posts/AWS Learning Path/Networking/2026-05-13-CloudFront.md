---
layout: post
title: AWS CloudFront — CDN, Distributions, Cache Policies, and Security
date: 2026-05-13T10:00:00
categories:
  - AWS Learning Path
  - Networking
tags:
  - aws
  - cloudfront
  - cdn
  - networking
  - cloud-security
author: muhammed
description: A full walkthrough of AWS CloudFront — distributions, origins, cache behaviours, cache policies, signed URLs, WAF integration, and real-world patterns
toc: true
pin: false
math: false
mermaid: false
image: assets/notes-img/cloudfront.png
Link: "[[]]"
Link1:
img:
---

## What is CloudFront?

**CloudFront** is AWS's Content Delivery Network (CDN).
A CDN is a globally distributed network of servers that caches copies of your content close to end users.
Instead of every user fetching content from your origin server (an S3 bucket or EC2 instance in one region), they receive it from the nearest CloudFront **edge location** — one of 400+ points of presence worldwide.

The result: lower latency, higher throughput, reduced load on your origin, and built-in DDoS protection.

CloudFront is a **global service** — it has no region.
You configure it once and it operates globally across all edge locations automatically.




---

## Core Concepts

| Term | Meaning |
|---|---|
| **Distribution** | A CloudFront deployment — the main configuration object |
| **Origin** | Where CloudFront fetches content from (S3, ALB, EC2, API Gateway) |
| **Edge location** | A CloudFront server close to the user that serves cached content |
| **Cache behaviour** | Rules for how CloudFront handles requests matching a specific path pattern |
| **Cache hit** | Content served from CloudFront cache — no origin request made |
| **Cache miss** | Content not in cache — CloudFront fetches from origin and caches the response |
| **TTL** | How long content stays cached before CloudFront checks the origin again |
| **Invalidation** | Manually removing content from the cache before TTL expires |

---

## Creating a Distribution

> **SCREENSHOT:** CloudFront → Distributions → Create Distribution.
> Show the Origin domain field with an S3 bucket selected, and the default cache behaviour section below it.

### From the console

Go to **CloudFront → Distributions → Create Distribution**.

Key settings to configure:
- **Origin domain** — your S3 bucket, ALB DNS name, or custom origin
- **Origin access** — for S3 origins, use Origin Access Control (OAC) to keep the bucket private
- **Default cache behaviour** — path pattern `*`, viewer protocol, allowed HTTP methods
- **WAF** — optionally attach an AWS WAF Web ACL
- **Price class** — which edge locations to use (all / only US+EU / only US+EU+Asia)
- **Alternate domain names (CNAMEs)** — your custom domain (e.g. `cdn.myapp.com`)
- **SSL certificate** — must be in `us-east-1` for CloudFront (ACM global certificate)

### With the CLI

```bash
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "my-dist-001",
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "s3-origin",
        "DomainName": "my-bucket.s3.amazonaws.com",
        "S3OriginConfig": {"OriginAccessIdentity": ""}
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "s3-origin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "AllowedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"]
      }
    },
    "Enabled": true,
    "Comment": "My app CDN"
  }'

# List distributions
aws cloudfront list-distributions --output table

# Get a distribution
aws cloudfront get-distribution --id EDFDVBD6EXAMPLE
```

---

## Origins

An **origin** is the source CloudFront fetches content from on a cache miss.
You can have multiple origins in a single distribution.

### S3 Origin

Used for static websites, assets, media files.
Always use **Origin Access Control (OAC)** — it keeps the S3 bucket completely private.
Only CloudFront can access the bucket; direct S3 URL access is blocked.

> 📸 **SCREENSHOT:** CloudFront distribution → Origins tab → Edit origin.
> Show the Origin Access Control section with OAC selected instead of "Public", and the bucket policy that CloudFront auto-generates.

**Bucket policy required for OAC:**

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

### Custom Origin (ALB, EC2, API Gateway)

Used for dynamic content — your backend API, web application, or any HTTP server.

```
CloudFront → ALB → EC2 instances
CloudFront → API Gateway → Lambda
CloudFront → EC2 (custom origin)
```

For custom origins, CloudFront communicates with your origin over HTTPS.
You can restrict your ALB to only accept traffic from CloudFront IP ranges — preventing users from bypassing CloudFront and hitting your ALB directly.

### Origin Groups (Failover)

An origin group has a primary and secondary origin.
If the primary returns a 5xx error, CloudFront automatically retries the request against the secondary.

> 📸 **SCREENSHOT:** CloudFront → Origins tab → Create Origin Group.
> Show the primary and secondary origin selected, and the failover criteria (which HTTP status codes trigger the failover).

---

## Cache Behaviours

A **cache behaviour** maps a URL path pattern to an origin and a set of caching rules.
Behaviours are evaluated in order — the most specific pattern wins.

```
/api/*         → ALB origin     (no caching, forward all headers)
/images/*      → S3 origin      (cache for 24 hours)
/static/*      → S3 origin      (cache for 7 days)
*              → ALB origin     (default — cache for 1 hour)
```

> 📸 **SCREENSHOT:** CloudFront distribution → Behaviours tab.
> Show multiple path patterns listed in order, each with its origin and cache policy shown.

### Viewer Protocol Policy

| Setting | Behaviour |
|---|---|
| `HTTP and HTTPS` | Allow both |
| `Redirect HTTP to HTTPS` | HTTP requests get 301 to HTTPS |
| `HTTPS Only` | HTTP requests get rejected with 403 |

Always use **Redirect HTTP to HTTPS** or **HTTPS Only** in production.

### Allowed HTTP Methods

- `GET, HEAD` — read-only (static assets, images)
- `GET, HEAD, OPTIONS` — CORS pre-flight
- `GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE` — full API (dynamic content)

---

## Cache Policies and TTL

A **cache policy** controls what CloudFront includes in the cache key and how long content is cached.

AWS provides managed policies:

| Policy | TTL | Use Case |
|---|---|---|
| `CachingOptimized` | 24 hours | S3 static content |
| `CachingDisabled` | 0 | Dynamic content, APIs |
| `CachingOptimizedForUncompressedObjects` | 24 hours | No Gzip/Brotli |

### Custom cache policy

```bash
aws cloudfront create-cache-policy \
  --cache-policy-config '{
    "Name": "my-cache-policy",
    "DefaultTTL": 3600,
    "MaxTTL": 86400,
    "MinTTL": 0,
    "ParametersInCacheKeyAndForwardedToOrigin": {
      "EnableAcceptEncodingGzip": true,
      "EnableAcceptEncodingBrotli": true,
      "HeadersConfig": {"HeaderBehavior": "none"},
      "CookiesConfig": {"CookieBehavior": "none"},
      "QueryStringsConfig": {"QueryStringBehavior": "none"}
    }
  }'
```

### Origin Request Policy

Separate from the cache policy.
Controls what headers, cookies, and query strings are forwarded to the origin (but not used in the cache key).

---

## Cache Invalidation

When you update content at the origin, the old version stays cached until the TTL expires.
An **invalidation** tells CloudFront to remove specific paths from all edge caches immediately.

> 📸 **SCREENSHOT:** CloudFront distribution → Invalidations tab → Create Invalidation.
> Show the path field with `/*` entered to invalidate all cached objects.

```bash
# Invalidate everything (use after a deployment)
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths '{"Quantity": 1, "Items": ["/*"]}'

# Invalidate specific paths
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths '{"Quantity": 2, "Items": ["/index.html", "/app.js"]}'

# Check invalidation status
aws cloudfront list-invalidations --distribution-id EDFDVBD6EXAMPLE
```

The first 1,000 invalidation paths per month are free.
After that, you are charged per path.
Wildcard `/*` counts as one path regardless of how many files it matches.

---

## HTTPS and SSL Certificates

CloudFront serves HTTPS by default using a `*.cloudfront.net` certificate.
For a custom domain (e.g. `cdn.myapp.com`), you need to:

1. Request a certificate in **ACM (us-east-1)** — CloudFront requires certificates in us-east-1, even if your stack is in another region.
2. Attach the certificate to your distribution.
3. Add a CNAME or Alias record in Route 53 pointing your custom domain to the CloudFront distribution domain.

> 📸 **SCREENSHOT:** CloudFront distribution → General tab → Settings → Edit.
> Show the "Alternate domain name" field with your custom domain entered, and the Custom SSL certificate dropdown showing the ACM cert selected.

```
myapp.com → CNAME → d1234567890abc.cloudfront.net
         or
myapp.com → Alias → d1234567890abc.cloudfront.net (for root domains)
```

---

## Security

### AWS WAF Integration

Attach a **WAF Web ACL** to your distribution to filter malicious requests at the edge — before they reach your origin.
WAF rules can block by IP, rate-limit, detect SQL injection, XSS, and more.

> 📸 **SCREENSHOT:** CloudFront distribution → General tab → Settings section.
> Show the WAF Web ACL field with a Web ACL selected from the dropdown.

### Geo Restriction

Block or allow requests based on the viewer's country.

```bash
aws cloudfront update-distribution \
  --id EDFDVBD6EXAMPLE \
  --distribution-config '{
    ...
    "Restrictions": {
      "GeoRestriction": {
        "RestrictionType": "blacklist",
        "Quantity": 2,
        "Items": ["CN", "RU"]
      }
    }
  }'
```

### Signed URLs and Signed Cookies

Used to restrict access to private content — only users with a valid signed URL or cookie can access the content.

**Signed URL** — protects a single file.
**Signed Cookie** — protects multiple files with a single cookie.

```
Use signed URLs for:   individual downloads, one-off content
Use signed cookies for: premium video streams, authenticated file areas
```

Both require a **CloudFront key pair** managed in your AWS account's security settings.

---

## CloudFront Functions and Lambda@Edge

You can run code at CloudFront edge locations to modify requests and responses.

| | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Runtime | JavaScript (ES5) | Node.js, Python |
| Max execution time | 1ms | 5 seconds (viewer), 30 seconds (origin) |
| Triggers | Viewer request/response | Viewer + Origin request/response |
| Cost | Very cheap | More expensive |
| Use case | URL rewrites, header manipulation, redirects | Complex auth, A/B testing, dynamic personalisation |

**Example CloudFront Function — redirect trailing slash:**

```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    if (uri.endsWith('/')) {
        request.uri = uri + 'index.html';
    }
    return request;
}
```

---

## Real-World Patterns

### Pattern 1 — Static website with S3 + CloudFront

```
User → CloudFront (edge cache) → S3 bucket (private, OAC)
                                      ↕
                              Route 53 Alias record
```

S3 bucket is completely private — no public access.
CloudFront is the only way to reach the content.
HTTPS enforced. WAF attached. Custom domain via Route 53.

### Pattern 2 — API + Static frontend

```
User → CloudFront
         ├── /api/*  → API Gateway / ALB (no cache)
         └── /*      → S3 (cache 24h)
```

One CloudFront distribution handles both the frontend (cached, S3) and the API (not cached, ALB) — no CORS issues because they share the same domain.

### Pattern 3 — Multi-region failover

```
User → CloudFront → Origin Group
                      ├── Primary: eu-west-1 ALB
                      └── Failover: us-east-1 ALB
```

If the primary region returns 5xx, CloudFront automatically retries on the failover origin.

---

## Quick Reference

```bash
# List distributions
aws cloudfront list-distributions \
  --query 'DistributionList.Items[*].[Id,DomainName,Status]' --output table

# Get distribution details
aws cloudfront get-distribution --id EDFDVBD6EXAMPLE

# Invalidate cache
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths '{"Quantity":1,"Items":["/*"]}'

# List invalidations
aws cloudfront list-invalidations --distribution-id EDFDVBD6EXAMPLE

# Disable a distribution (required before deletion)
aws cloudfront update-distribution --id EDFDVBD6EXAMPLE \
  --if-match ETAG --distribution-config '{"Enabled": false, ...}'

# Delete distribution
aws cloudfront delete-distribution --id EDFDVBD6EXAMPLE --if-match ETAG
```































##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)


























##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
