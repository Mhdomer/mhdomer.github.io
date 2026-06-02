---
layout: post
title: "Week 2 — Day 9: HashiCorp Vault Basics"
date: 2026-03-09 10:00:00 +0800
categories:
  - DevSecOps
  - Week2
tags:
  - Vault
  - HashiCorp
  - SecretsManagement
  - CloudSecurity
  - DevSecOps
author: muhammed
description: A full walkthrough of HashiCorp Vault — architecture, secrets engines, auth methods, dynamic secrets, and hands-on usage for multi-cloud and on-prem secrets management.
toc: true
pin: false
math: false
mermaid: false
image: https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.datocms-assets.com%2F2885%2F1620082933-blog-library-product-vault-logo.jpg&f=1&nofb=1&ipt=8f0fe9f801e63c782294b8fbed181b060245f57751c11d003136465db5731828
---

## Why Vault When AWS Has Secrets Manager?

AWS Secrets Manager is great — but it's AWS-only. If you have:
- Multiple cloud providers (AWS + GCP + Azure)
- On-prem infrastructure
- Kubernetes clusters across different environments
- A requirement to manage secrets independently of any cloud vendor

...then **HashiCorp Vault** is the answer. It's a unified secrets management platform that works anywhere.

---

## Vault Architecture

```
┌─────────────────────────────────────┐
│              Vault Server            │
│                                     │
│  ┌─────────────┐  ┌──────────────┐  │
│  │   Storage   │  │   Secrets    │  │
│  │  Backend    │  │   Engines    │  │
│  │ (S3, Consul,│  │ (KV, AWS,    │  │
│  │  DynamoDB)  │  │  DB, PKI...) │  │
│  └─────────────┘  └──────────────┘  │
│                                     │
│  ┌─────────────────────────────┐    │
│  │       Auth Methods          │    │
│  │ (Token, AWS, K8s, LDAP...)  │    │
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │         Policies            │    │
│  │  (HCL — who can do what)    │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

**Key concepts:**
- **Secrets Engines** — plugins that store or generate secrets (KV store, AWS credentials, database passwords, certificates)
- **Auth Methods** — how clients authenticate to Vault (tokens, AWS IAM, Kubernetes service accounts, LDAP)
- **Policies** — HCL documents defining what paths an authenticated client can access
- **Leases** — all dynamic secrets have a TTL; when the lease expires, Vault revokes the secret

---

## Running Vault Locally (Dev Mode)

Dev mode runs an in-memory Vault with a root token — perfect for learning. **Never use dev mode in production.**

```bash
# Install Vault (Windows)
choco install vault

# Or download from https://developer.hashicorp.com/vault/downloads

# Start in dev mode
vault server -dev
```

> `[SCREENSHOT]` — *Terminal showing vault server -dev output with the Root Token and Unseal Key printed, and the UI address (http://127.0.0.1:8200)*

The terminal will print a **Root Token** — copy it. Then in a new terminal:

```bash
# Set environment variables
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='<root-token-from-output>'

# Verify it's working
vault status
```

> `[SCREENSHOT]` — *Terminal showing vault status output with Sealed: false, Storage Type: inmem, HA Enabled: false*

---

## Secrets Engines

### KV (Key-Value) Secrets Engine

The simplest engine — stores static key-value pairs. Two versions:
- **KV v1** — no versioning
- **KV v2** — keeps a history of previous versions (default in dev mode)

```bash
# Enable KV v2 at path "secret/"
vault secrets enable -path=secret kv-v2

# Write a secret
vault kv put secret/myapp/db username="dbadmin" password="S3cr3t!"

# Read a secret
vault kv get secret/myapp/db

# Read just one field
vault kv get -field=password secret/myapp/db
```

> `[SCREENSHOT]` — *Terminal showing vault kv get output returning the key-value pairs for secret/myapp/db with metadata (version, created_time)*

**KV v2 versioning:**
```bash
# Put a new version
vault kv put secret/myapp/db password="NewP@ss!"

# Read a specific version
vault kv get -version=1 secret/myapp/db

# Delete current version
vault kv delete secret/myapp/db

# Permanently destroy a version
vault kv destroy -versions=1 secret/myapp/db
```

---

### AWS Secrets Engine (Dynamic Credentials)

This is where Vault gets powerful. Instead of storing static AWS keys, Vault **generates temporary IAM credentials on demand** and revokes them when the lease expires.

```bash
# Enable AWS secrets engine
vault secrets enable aws

# Configure with an IAM user that has permission to create credentials
vault write aws/config/root \
  access_key=AKIAIOSFODNN7EXAMPLE \
  secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  region=ap-southeast-1

# Create a role that generates credentials with S3 read access
vault write aws/roles/s3-reader \
  credential_type=iam_user \
  policy_document='{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": "*"
    }]
  }'

# Generate temporary credentials
vault read aws/creds/s3-reader
```

> `[SCREENSHOT]` — *Terminal showing vault read aws/creds/s3-reader output with access_key, secret_key, and lease_duration (e.g. 768h)*

The generated IAM user exists only for the lease duration. When the lease expires (or you revoke it), the IAM user is deleted automatically. No long-lived keys anywhere.

---

### Database Secrets Engine (Dynamic DB Credentials)

Same concept for databases — Vault generates a unique username/password per request, valid for a limited time.

```bash
# Enable database secrets engine
vault secrets enable database

# Configure for PostgreSQL
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly" \
  connection_url="postgresql://{{username}}:{{password}}@localhost:5432/mydb" \
  username="vault-admin" \
  password="vault-admin-pass"

# Create a role
vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Get credentials
vault read database/creds/readonly
```

> `[SCREENSHOT]` — *Terminal showing vault read database/creds/readonly output with a generated username like "v-readonly-abc123" and a random password with lease_duration 1h*

Every application gets a unique DB user. If one app is compromised, its credentials expire within the hour and never worked for anything else.

---

## Auth Methods

### Token Auth (Default)

Every client authenticates to Vault with a token. Tokens have policies attached and optional TTLs.

```bash
# Create a token with a specific policy and TTL
vault token create -policy=read-secrets -ttl=1h

# Revoke a token
vault token revoke <token>
```

### AWS IAM Auth

Applications running on EC2, ECS, or Lambda authenticate using their IAM role — no token pre-seeding needed.

```bash
# Enable AWS auth
vault auth enable aws

# Configure
vault write auth/aws/config/client \
  access_key=AKIAIOSFODNN7EXAMPLE \
  secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Create a role mapping an IAM role to a Vault policy
vault write auth/aws/role/my-ec2-role \
  auth_type=iam \
  bound_iam_principal_arn=arn:aws:iam::123456789:role/my-ec2-role \
  policies=read-secrets \
  ttl=1h
```

The EC2 instance calls Vault using its IAM role identity — Vault verifies with AWS that the role is who it claims to be. No static Vault token ever touches the instance.

---

## Policies

Policies are HCL documents that define what a client can do.

```hcl
# read-secrets.hcl — allow reading from secret/myapp/
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# deny writing
path "secret/data/myapp/admin" {
  capabilities = ["deny"]
}
```

```bash
# Write the policy
vault policy write read-secrets read-secrets.hcl

# List policies
vault policy list

# Read a policy
vault policy read read-secrets
```

> `[SCREENSHOT]` — *Terminal showing vault policy list output and vault policy read showing the HCL content of the read-secrets policy*

**Capabilities:**
| Capability | HTTP method | Description |
|-----------|-------------|-------------|
| `create` | POST | Create new secret |
| `read` | GET | Read a secret |
| `update` | POST | Update existing |
| `delete` | DELETE | Delete |
| `list` | LIST | List keys |
| `deny` | — | Explicit deny, overrides everything |

---

## Vault Agent

Vault Agent runs as a sidecar or daemon alongside your application. It:
- Authenticates to Vault automatically
- Retrieves secrets and writes them to a file or environment
- Renews leases automatically before they expire

```hcl
# vault-agent.hcl
auto_auth {
  method "aws" {
    config = {
      type = "iam"
      role = "my-ec2-role"
    }
  }
}

template {
  source      = "/etc/vault/db-config.ctmpl"
  destination = "/etc/myapp/db-config.env"
}
```

The template file:
{% raw %}
```
DB_USER={{ with secret "secret/data/myapp/db" }}{{ .Data.data.username }}{{ end }}
DB_PASS={{ with secret "secret/data/myapp/db" }}{{ .Data.data.password }}{{ end }}
```
{% endraw %}

> `[SCREENSHOT]` — *Terminal showing vault agent starting up, authenticating via AWS IAM auth, and writing rendered templates to the destination file*

---

## Lab — Run Vault in Dev Mode

**Objective:** Start Vault locally, store a secret, and retrieve it via CLI.

1. Start Vault in dev mode and copy the root token
2. Set environment variables `VAULT_ADDR` and `VAULT_TOKEN`
3. Write a secret:

```bash
vault kv put secret/lab/apikey \
  service="github" \
  token="ghp_exampletoken123"
```

4. Read it back:

```bash
vault kv get secret/lab/apikey
```

> `[SCREENSHOT]` — *Terminal showing the vault kv get output with the service and token fields displayed*

5. Open the Vault UI at `http://127.0.0.1:8200` — log in with the root token

> `[SCREENSHOT]` — *Vault UI showing the secrets browser with the secret/lab/apikey secret visible and its key-value pairs*

6. Write a second version:

```bash
vault kv put secret/lab/apikey token="ghp_updatedtoken456"
```

7. Read the previous version:

```bash
vault kv get -version=1 secret/lab/apikey
```

> `[SCREENSHOT]` — *Terminal showing vault kv get -version=1 returning the original token value from version 1*

---

## Key Takeaways

- Vault is cloud-agnostic — use it when you need multi-cloud or vendor-independent secrets management
- Dynamic secrets are the key differentiator — credentials that auto-expire reduce breach impact dramatically
- Auth methods tie Vault to your existing identity systems — AWS IAM, Kubernetes, LDAP
- Vault Agent removes the secret-zero problem — applications never need a static token
- Dev mode is for learning only — production Vault needs HA, storage backend, and TLS

---

## References

<div class="references">
<ul>
  <li><a href="https://developer.hashicorp.com/vault/docs" target="_blank">HashiCorp Vault Documentation</a></li>
  <li><a href="https://developer.hashicorp.com/vault/tutorials/getting-started" target="_blank">Vault Getting Started Tutorial</a></li>
  <li><a href="https://developer.hashicorp.com/vault/docs/secrets/aws" target="_blank">Vault AWS Secrets Engine</a></li>
</ul>
</div>

---

## You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
