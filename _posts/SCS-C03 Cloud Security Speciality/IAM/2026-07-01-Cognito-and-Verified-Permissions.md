---
layout: post
title: Amazon Cognito and Verified Permissions — App Identity and Fine-Grained Authorization
date: 2026-07-01T10:00:00
categories:
  - SCS-C03 Cloud Security Speciality
  - IAM
tags:
  - aws
  - cognito
  - verified-permissions
  - jwt
  - oidc
  - oauth2
  - cloud-security
  - scs-c03
author: muhammed
description: SCS-C03 Domain 4 — Cognito user pools vs identity pools, JWT validation, federated identities, Cognito hosted UI, Amazon Verified Permissions for app-level authorization
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[]]"
Link1:
img:
---

## Amazon Cognito Overview

**Amazon Cognito** is the AWS service for application-level identity — managing user registration, authentication, and authorization for web and mobile apps.
Cognito has two distinct components that solve different problems:

| | User Pool | Identity Pool |
|---|---|---|
| **Purpose** | Authentication — manage users, issue tokens | Authorization — exchange tokens for AWS credentials |
| **Output** | JWT tokens (ID, access, refresh) | Temporary AWS credentials (STS) |
| **Use case** | Sign in to your application | Access AWS services directly from client |

Most applications use both together — the user pool authenticates the user, then the identity pool exchanges the JWT for AWS credentials.

---

## Cognito User Pools

A **user pool** is a fully managed user directory.
It handles sign-up, sign-in, MFA, password policies, email/phone verification, and account recovery.
It supports federation — users can log in with Google, Apple, Facebook, or any SAML/OIDC provider.

```bash
# Create a user pool
aws cognito-idp create-user-pool \
  --pool-name my-app-users \
  --policies PasswordPolicy='{
    MinimumLength=12,
    RequireUppercase=true,
    RequireLowercase=true,
    RequireNumbers=true,
    RequireSymbols=true
  }' \
  --mfa-configuration OPTIONAL \
  --auto-verified-attributes email \
  --username-attributes email

# Create an app client (no secret for SPA/mobile, with secret for server-side)
aws cognito-idp create-user-pool-client \
  --user-pool-id eu-west-1_abc123 \
  --client-name my-web-app \
  --no-generate-secret \
  --explicit-auth-flows ALLOW_USER_SRP_AUTH ALLOW_REFRESH_TOKEN_AUTH \
  --supported-identity-providers COGNITO Google \
  --callback-urls '["https://myapp.com/callback"]' \
  --logout-urls '["https://myapp.com/logout"]'
```

---

## Cognito JWT Tokens

After sign-in, Cognito issues three tokens:

| Token | Contents | Lifetime |
|---|---|---|
| **ID token** | User claims — email, name, custom attributes, groups | 1 hour (default) |
| **Access token** | OAuth 2.0 scopes — used to authorize API calls | 1 hour (default) |
| **Refresh token** | Used to get new ID + access tokens | 30 days (default) |

Tokens are **JWTs signed with Cognito's RSA key**.
Validate tokens on your backend by verifying the signature against Cognito's JWKS endpoint:

```
https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json
```

**Validation steps:**
1. Decode the JWT header → get the `kid` (key ID)
2. Fetch the JWKS endpoint → find the public key matching the `kid`
3. Verify the token signature with the public key
4. Check `exp` claim (not expired), `iss` claim (matches your user pool), `aud` claim (matches your app client ID)

```bash
# Get user pool JWKS for manual validation
curl https://cognito-idp.eu-west-1.amazonaws.com/eu-west-1_abc123/.well-known/jwks.json

# Verify a token's claims (CLI approach)
aws cognito-idp get-user \
  --access-token YOUR_ACCESS_TOKEN_HERE
```

---

## Cognito Hosted UI and Federation

The **Hosted UI** is Cognito's built-in sign-in page — you don't need to build a custom login UI.
Enable it with a domain name (Cognito domain or custom domain) and configure social/SAML providers.

```bash
# Add a Cognito domain for the hosted UI
aws cognito-idp create-user-pool-domain \
  --domain my-app-auth \
  --user-pool-id eu-west-1_abc123
# Access at: https://my-app-auth.auth.eu-west-1.amazoncognito.com

# Add Google as a federated identity provider
aws cognito-idp create-identity-provider \
  --user-pool-id eu-west-1_abc123 \
  --provider-name Google \
  --provider-type Google \
  --provider-details '{
    "client_id": "google-oauth-client-id",
    "client_secret": "google-oauth-client-secret",
    "authorize_scopes": "email profile openid"
  }' \
  --attribute-mapping email=email name=name
```

Federation flow:
```
User clicks "Sign in with Google"
    │
    ▼
Cognito Hosted UI redirects to Google OAuth
    │
    ▼
User authenticates with Google
    │
    ▼
Google returns OAuth code to Cognito
    │
    ▼
Cognito creates/updates a federated user in the user pool
    │
    ▼
Cognito issues ID + access tokens to your app
```

---

## Cognito Identity Pools

An **identity pool** (formerly Federated Identities) exchanges tokens from any identity provider for temporary AWS credentials via STS.
Supports both authenticated identities (logged-in users) and unauthenticated identities (guest access).

```bash
# Create an identity pool
aws cognito-identity create-identity-pool \
  --identity-pool-name my-app-identities \
  --allow-unauthenticated-identities  # or false to require login \
  --cognito-identity-providers ProviderName=cognito-idp.eu-west-1.amazonaws.com/eu-west-1_abc123,ClientId=client-id-here,ServerSideTokenCheck=true

# Get credentials for an authenticated user
aws cognito-identity get-credentials-for-identity \
  --identity-id eu-west-1:identity-id-here \
  --logins '{"cognito-idp.eu-west-1.amazonaws.com/eu-west-1_abc123": "USER_ID_TOKEN_HERE"}'
```

### Authenticated vs Unauthenticated Roles

The identity pool has two IAM roles:
- **Authenticated role**: assigned to users who have logged in — typically has more permissions
- **Unauthenticated role**: assigned to guests — minimal permissions (e.g. read-only from specific S3 prefix)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "cognito-identity.amazonaws.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "cognito-identity.amazonaws.com:aud": "eu-west-1:identity-pool-id"
      },
      "ForAnyValue:StringLike": {
        "cognito-identity.amazonaws.com:amr": "authenticated"
      }
    }
  }]
}
```

---

## Cognito User Pool Groups and RBAC

**User pool groups** implement role-based access control at the Cognito level.
Assign users to groups — each group is associated with an IAM role.
The user's ID token includes a `cognito:groups` claim listing their group memberships.

```bash
# Create a group with an associated IAM role
aws cognito-idp create-group \
  --user-pool-id eu-west-1_abc123 \
  --group-name admins \
  --role-arn arn:aws:iam::123456789012:role/cognito-admin-role \
  --precedence 1

# Add a user to a group
aws cognito-idp admin-add-user-to-group \
  --user-pool-id eu-west-1_abc123 \
  --username user@example.com \
  --group-name admins

# The ID token will contain:
# "cognito:groups": ["admins"]
# Your application checks this claim for authorization
```

---

## Cognito Lambda Triggers

**Lambda triggers** customize the Cognito auth flow — validate sign-ups, modify tokens, migrate existing users, add custom auth challenges.

| Trigger | When it fires | Use for |
|---|---|---|
| **Pre sign-up** | Before user is confirmed | Validate email domain, auto-confirm users |
| **Pre token generation** | Before tokens are issued | Add custom claims to ID token |
| **Post authentication** | After successful sign-in | Audit logging, update last-login |
| **Custom message** | Before verification email/SMS | Customize verification templates |
| **Migrate user** | On sign-in for unknown users | Migrate users from legacy auth system |

```bash
# Add a pre-token-generation trigger to add custom claims
aws cognito-idp update-user-pool \
  --user-pool-id eu-west-1_abc123 \
  --lambda-config PreTokenGeneration=arn:aws:lambda:eu-west-1:123456789012:function:add-custom-claims
```

---

## Amazon Verified Permissions

**Amazon Verified Permissions** provides fine-grained, application-level authorization using the **Cedar** policy language.
It is separate from IAM — it controls what your application's users can do (not what AWS resources they can access).

Use Verified Permissions when:
- You need resource-level authorization (e.g. user A can only edit documents they own)
- Authorization logic is complex and policy-driven (not hardcoded in application code)
- You want a centralized authorization service your microservices query at runtime

```
Application request: "Can user Alice delete document doc-123?"
    │
    ▼
Verified Permissions (Cedar policy store)
    │ evaluates Cedar policies
    ▼
Decision: Allow / Deny
    │
    ▼
Application enforces the decision
```

```bash
# Create a policy store
aws verifiedpermissions create-policy-store \
  --validation-settings Mode=STRICT

# Create a Cedar policy (allow document owners to delete their documents)
aws verifiedpermissions create-policy \
  --policy-store-id policy-store-id-here \
  --definition StaticPolicy='{
    "statement": "permit(principal, action == Action::\"DeleteDocument\", resource) when { resource.owner == principal.id };"
  }'

# Evaluate an authorization request
aws verifiedpermissions is-authorized \
  --policy-store-id policy-store-id-here \
  --principal EntityType=User,EntityId=alice \
  --action ActionType=Action,ActionId=DeleteDocument \
  --resource EntityType=Document,EntityId=doc-123 \
  --entities '[{
    "identifier": {"entityType": "Document", "entityId": "doc-123"},
    "attributes": {"owner": {"string": "alice"}}
  }]'
```

---

## Cognito vs IAM Identity Center

| | Cognito | IAM Identity Center |
|---|---|---|
| **Audience** | Your application's end users | AWS console and CLI users |
| **Scale** | Millions of end users | Hundreds of corporate employees |
| **Protocol** | OAuth 2.0, OIDC | SAML 2.0, SCIM |
| **Output** | JWTs for your app | Temporary AWS credentials |
| **Use case** | Mobile/web app sign-in | Multi-account AWS access |

---

## Exam Key Points

- **User pool**: authentication — sign-up, sign-in, MFA, federation, issues JWTs
- **Identity pool**: authorization — exchanges JWTs for temporary AWS credentials via STS
- **JWT validation**: verify signature against Cognito's JWKS endpoint + check `exp`, `iss`, `aud` claims
- **Groups in user pools**: RBAC — group name appears in `cognito:groups` claim in the ID token; your app reads this claim
- **Pre-token generation trigger**: add custom claims to the token without changing the app client
- **Verified Permissions**: application-level authorization using Cedar — for what users can do in your app, not in AWS
- **Cognito vs IAM Identity Center**: Cognito for your app users, Identity Center for AWS account access

---

## Quick Reference

```bash
# User Pool
aws cognito-idp list-user-pools --max-results 20
aws cognito-idp admin-get-user --user-pool-id eu-west-1_abc123 --username user@example.com
aws cognito-idp admin-list-groups-for-user --user-pool-id eu-west-1_abc123 --username user@example.com
aws cognito-idp list-users --user-pool-id eu-west-1_abc123 --filter "status = 'COMPROMISED'"

# Identity Pool
aws cognito-identity list-identity-pools --max-results 20
aws cognito-identity describe-identity-pool --identity-pool-id eu-west-1:pool-id

# Disable a compromised user
aws cognito-idp admin-disable-user --user-pool-id eu-west-1_abc123 --username user@example.com
# Sign out from all devices
aws cognito-idp admin-user-global-sign-out --user-pool-id eu-west-1_abc123 --username user@example.com
```
