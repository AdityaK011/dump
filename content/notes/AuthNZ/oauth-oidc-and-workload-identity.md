---
title: "OAuth, OIDC & Workload Identity Federation"
---

## Single Sign-On (SSO)

SSO is the user experience where someone signs in once and can access multiple applications without re-authenticating. It is implemented using federation protocols — the two dominant ones being **SAML** (enterprise/legacy) and **OIDC** (modern).

SSO requires:
- An **Identity Provider (IdP)** that authenticates the user
- Multiple **applications/services** that trust the IdP
- A federation protocol (SAML or OIDC) to communicate authentication state

## SAML (Security Assertion Markup Language)

SAML is an XML-based federation protocol designed for **browser-based enterprise SSO**. It predates OAuth/OIDC and remains dominant in enterprise SaaS integrations (Salesforce, Workday, ServiceNow).

### Key Roles

| Role | SAML Term | Example |
| --- | --- | --- |
| Identity store + authenticator | **Identity Provider (IdP)** | Okta, Azure AD, PingFederate |
| Application | **Service Provider (SP)** | Salesforce, AWS Console, Jira |

### SP-Initiated Flow (Most Common)

```
User           SP (App)              IdP (Okta)
 |                |                      |
 |  GET /app      |                      |
 |--------------->|                      |
 |                |                      |
 |  302 Redirect  |                      |
 |  Location: IdP/sso?SAMLRequest=...   |
 |<---------------|                      |
 |                                       |
 |  Browser follows redirect             |
 |-------------------------------------->|
 |                                       |
 |        User authenticates (pwd/MFA)   |
 |<------------------------------------->|
 |                                       |
 |  POST /acs (Assertion Consumer Svc)   |
 |  SAMLResponse=<signed XML>            |
 |<--------------------------------------|
 |  (browser auto-submits form to SP)    |
 |                |                      |
 |  SP validates  |                      |
 |  signature,    |                      |
 |  creates       |                      |
 |  session       |                      |
 |<---------------|                      |
```

The **SAMLResponse** contains a signed XML `<Assertion>` with:
- **Subject** — who authenticated (`NameID`)
- **Conditions** — validity window, audience restriction
- **AuthnStatement** — when and how authentication happened
- **AttributeStatement** — optional attributes (email, groups, roles)

### SAML Gotchas

- **Clock skew**: IdP and SP clocks must be within a few minutes. `NotBefore`/`NotOnOrAfter` conditions will fail otherwise.
- **Certificate rotation**: IdP signing certificates expire. SP must be updated with the new certificate before the old one expires.
- **Replay protection**: SPs should track `InResponseTo` and assertion IDs to prevent replay attacks.
- **XML signature wrapping attacks**: A known vulnerability class where an attacker modifies the XML structure while keeping the signature valid over the original subtree. Libraries must verify that the signature covers the correct element.

### When to Use SAML vs OIDC

| Criterion | SAML | OIDC |
| --- | --- | --- |
| Format | XML | JSON/JWT |
| Transport | Browser POST/Redirect | Browser Redirect + back-channel |
| Best for | Enterprise browser SSO | Modern apps, SPAs, mobile, APIs |
| API access | Not designed for it | Access tokens for APIs |
| Complexity | Higher (XML, certs, metadata) | Lower (JSON, JWKS) |
| Adoption | Legacy enterprise apps | Greenfield, cloud-native |

## JWT (JSON Web Token)

JWT (RFC 7519) is a compact, URL-safe token format used by OIDC for ID tokens and optionally for access tokens. Understanding JWT structure is fundamental to understanding OIDC.

### Structure

A JWT is three Base64url-encoded segments separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLC...
|_____________HEADER______________|.______________PAYLOAD_______________|.__SIGNATURE__|
```

**Header** — algorithm and token type:
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-1"
}
```

**Payload (Claims)** — the actual data:
```json
{
  "iss": "https://accounts.google.com",
  "sub": "110011",
  "aud": "my-app.example.com",
  "exp": 1700000000,
  "iat": 1699996400,
  "nonce": "abc123",
  "email": "user@example.com",
  "email_verified": true,
  "name": "Jane Doe"
}
```

**Signature** — cryptographic proof of integrity. For RS256, the IdP signs with its private key; anyone can verify with the public key from the JWKS endpoint.

### Critical Points

- The payload is **Base64url-encoded, not encrypted**. Anyone holding the token can decode and read the claims. Never store secrets in JWT payloads.
- To decode in a terminal: `echo $TOKEN | cut -d'.' -f2 | base64 --decode | jq`
- The `kid` (Key ID) in the header is used to select the correct public key from the issuer's JWKS (JSON Web Key Set) endpoint for signature verification.

## OAuth 2.0

OAuth 2.0 (RFC 6749) is an **authorization** framework for delegated access. It lets an application access a user's resources on another service without the user sharing their password.

### Core Question OAuth Answers

> "Can this client access this resource on behalf of this user, and with what permissions?"

### Roles

```
+----------------+                         +--------------------+
| Resource Owner |  (the user)             | Authorization      |
| (User)         |                         | Server             |
+-------+--------+                         | (issues tokens)    |
        |                                  +--------+-----------+
        | authorizes                                |
        v                                           | issues tokens
+-------+--------+                         +--------v-----------+
| Client         |  ---- access token ---->| Resource Server    |
| (Application)  |                         | (API)              |
+----------------+                         +--------------------+
```

### Authorization Code Flow (with PKCE)

This is the recommended flow for most applications (RFC 6749 Section 4.1 + RFC 7636 for PKCE).

```
Client                    Browser/User           Authorization Server
  |                            |                          |
  |  1. Generate code_verifier |                          |
  |     + code_challenge       |                          |
  |                            |                          |
  |  2. Redirect to /authorize |                          |
  |     ?response_type=code    |                          |
  |     &client_id=...         |                          |
  |     &redirect_uri=...      |                          |
  |     &scope=read+write      |                          |
  |     &state=xyz             |                          |
  |     &code_challenge=...    |                          |
  |     &code_challenge_method=S256                       |
  |--------------------------->|------------------------->|
  |                            |                          |
  |                            |  3. User authenticates   |
  |                            |     and consents         |
  |                            |<------------------------>|
  |                            |                          |
  |  4. Redirect back          |                          |
  |     ?code=AUTH_CODE        |                          |
  |     &state=xyz             |                          |
  |<---------------------------|                          |
  |                                                       |
  |  5. POST /token                                       |
  |     grant_type=authorization_code                     |
  |     &code=AUTH_CODE                                   |
  |     &redirect_uri=...                                 |
  |     &code_verifier=...                                |
  |------------------------------------------------------>|
  |                                                       |
  |  6. { access_token, refresh_token, expires_in }       |
  |<------------------------------------------------------|
```

**PKCE** (Proof Key for Code Exchange, RFC 7636) prevents authorization code interception attacks. The client generates a random `code_verifier`, derives a `code_challenge` via SHA-256, sends the challenge in the authorization request, and proves possession of the verifier during token exchange. Originally designed for public clients (mobile/SPA) but now recommended for all clients per OAuth 2.1.

### Token Types

| Token | Purpose | Audience | Format | Lifetime |
| --- | --- | --- | --- | --- |
| **Access Token** | Authorize API calls | Resource Server | Opaque or JWT | Short (minutes-hours) |
| **Refresh Token** | Obtain new access tokens | Authorization Server | Opaque | Long (days-months) |

OAuth does **not** standardize identity. An access token says "this client may access these resources" but does not reliably answer "who is the user?"

## OIDC (OpenID Connect)

OIDC (OpenID Connect Core 1.0) is an **authentication** layer built on top of OAuth 2.0. It standardizes how applications verify user identity and obtain profile information.

### Core Question OIDC Answers

> "Who is the user that just authenticated?"

### What OIDC Adds to OAuth

```
+-----------------------------------------------+
|                  OIDC Layer                    |
|  - ID Token (JWT)                             |
|  - Standardized identity claims               |
|  - Discovery (/.well-known/openid-config)     |
|  - JWKS endpoint for signature verification   |
|  - UserInfo endpoint                          |
|  - nonce for replay protection                |
+-----------------------------------------------+
|              OAuth 2.0 Framework               |
|  - Authorization Code flow                    |
|  - Access Tokens, Refresh Tokens              |
|  - Scopes, Consent                            |
+-----------------------------------------------+
```

### Activating OIDC: The `openid` Scope

The presence of `scope=openid` in the authorization request is what triggers OIDC mode. Without it, the authorization server performs a plain OAuth flow and does not return an ID token.

```
scope=openid             --> ID token only
scope=openid profile     --> ID token with name, picture, etc.
scope=openid email       --> ID token with email, email_verified
scope=openid profile email --> all of the above
```

### OIDC Authorization Code Flow

Identical to OAuth Authorization Code flow, but the token response includes an **ID Token**:

```json
{
  "access_token": "ya29.a0ARrdaM...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "1//0eXy...",
  "id_token": "eyJhbGciOiJSUzI1NiIs..."
}
```

### Access Token vs ID Token

| Property | Access Token | ID Token |
| --- | --- | --- |
| Purpose | Authorization (API access) | Authentication (identity proof) |
| Audience | Resource Server (API) | Client Application |
| Format | Opaque string or JWT | **Always** JWT |
| Standardized claims | No (provider-specific) | Yes (`sub`, `iss`, `aud`, `exp`, `nonce`) |
| Sent to APIs | Yes | **Never** |
| Used for login | No | Yes |

The ID token is consumed **only by the client** to establish a session. It must never be sent to a resource server. The access token is sent to APIs.

```
Client receives both tokens:

  ID Token --> Client validates --> extracts identity --> creates session

  Access Token --> sent to API:
      GET /api/data
      Authorization: Bearer ACCESS_TOKEN

  Resource Server validates access token, checks scopes, returns data.
```

### ID Token Validation (Required by OIDC Core Section 3.1.3.7)

The client **must** validate the ID token before trusting it:

1. **Signature** — verify against the issuer's public keys (JWKS endpoint)
2. **`iss`** — must match the expected issuer URL exactly
3. **`aud`** — must contain the client's own `client_id`
4. **`exp`** — must not be expired
5. **`iat`** — must be reasonably recent
6. **`nonce`** — must match the nonce sent in the authorization request (prevents replay)

### Discovery

OIDC providers publish configuration at a well-known URL:

```
GET https://accounts.google.com/.well-known/openid-configuration

{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  ...
}
```

### OIDC Provider (OP) vs OAuth Authorization Server

| OAuth Term | OIDC Term | Role |
| --- | --- | --- |
| Authorization Server | OpenID Provider (OP) / IdP | Authenticates users, issues tokens |

Common examples: Okta, Azure AD/Entra ID, Google, Auth0, Keycloak, Cognito.

### Why Not Just Use OAuth `email` Scope for Identity?

In pure OAuth, scopes are permission strings whose meaning is provider-specific. `scope=email` might grant access to "some API that returns email" — but there is no standard for where the email appears, what format it takes, or whether it is verified.

OIDC standardizes this: `scope=openid email` guarantees you receive an `email` claim and an `email_verified` boolean in the ID token or UserInfo response.

Additionally, **email is not a stable identifier**:
- Emails change
- Emails may not be unique across providers/tenants
- Some providers do not return email for all users

Best practice: use `sub` (subject) as the stable unique identifier. For cross-issuer uniqueness, use the tuple `(iss, sub)`.

## Workload Identity Federation (WIF)

Workload Identity Federation allows external workloads (GitHub Actions, AWS, on-prem services) to authenticate to cloud providers **without long-lived service account keys**. It combines OIDC identity assertion with OAuth 2.0 Token Exchange (RFC 8693).

### The Problem WIF Solves

```
BEFORE (insecure):
  GitHub Actions --> stores GCP service account key as a secret
                     --> key is long-lived, can leak, hard to rotate

AFTER (WIF):
  GitHub Actions --> mints OIDC token (short-lived JWT)
                 --> exchanges it for GCP access token via STS
                     --> no stored secrets, tokens expire in minutes
```

### GitHub Actions to GCP Flow

```
GitHub Actions Runner          Google STS              GCP APIs
        |                          |                      |
        | 1. Request OIDC token    |                      |
        |    from GitHub IdP       |                      |
        |    (token.actions.       |                      |
        |     githubusercontent.com)|                      |
        |                          |                      |
        | 2. POST /token           |                      |
        |    grant_type=           |                      |
        |    urn:ietf:params:oauth:|                      |
        |    grant-type:token-     |                      |
        |    exchange              |                      |
        |    &subject_token=<JWT>  |                      |
        |    &audience=//iam...    |                      |
        |------------------------->|                      |
        |                          |                      |
        |                  3. Google validates:            |
        |                     - Fetches JWKS from GitHub   |
        |                     - Verifies JWT signature     |
        |                     - Checks iss, aud, exp       |
        |                     - Maps claims to attributes  |
        |                     - Checks IAM policy          |
        |                          |                      |
        | 4. GCP access token      |                      |
        |<-------------------------|                      |
        |                                                 |
        | 5. Call GCP API with access token               |
        |------------------------------------------------>|
```

### Inside the GitHub OIDC Token

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:myorg/myrepo:ref:refs/heads/main",
  "aud": "https://sts.googleapis.com",
  "repository": "myorg/myrepo",
  "repository_owner": "myorg",
  "ref": "refs/heads/main",
  "actor": "john-dev",
  "workflow": "deploy",
  "exp": 1700000600,
  "iat": 1700000000
}
```

### Claim-to-Attribute Mapping

Google does not directly convert OIDC claims into permissions. Instead, claims are mapped to Google attributes, which form a federated principal, which is then evaluated against IAM policies.

```
OIDC Claims (from JWT)
        |
        v
Attribute Mapping (configured in WIF pool)
   google.subject       = assertion.sub
   attribute.repository = assertion.repository
   attribute.actor      = assertion.actor
        |
        v
Federated Principal (constructed by Google)
   principal://iam.googleapis.com/projects/PROJECT_NUMBER/
     locations/global/workloadIdentityPools/POOL_ID/
     subject/repo:myorg/myrepo:ref:refs/heads/main
        |
        v
IAM Policy Binding (evaluates principal against roles)
   role: roles/storage.admin
   members:
     - principalSet://iam.googleapis.com/.../attribute.repository/myorg/myrepo
        |
        v
Access Token Issued (or denied)
```

### Granular Access Control

Bind to any workflow from a specific repo:
```yaml
members:
  - principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/attribute.repository/myorg/myrepo
```

Restrict to only the `main` branch:
```yaml
members:
  - principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/subject/repo:myorg/myrepo:ref:refs/heads/main
```

Add IAM conditions for even finer control:
```yaml
condition:
  expression: attribute.ref == "refs/heads/main"
```

### Service Account Impersonation

A common pattern is to not grant permissions directly to the federated identity, but instead allow it to impersonate a service account:

1. Federated identity gets `roles/iam.workloadIdentityUser` on a specific SA
2. Workflow exchanges OIDC token for a short-lived SA token
3. SA token is used for GCP API calls

This provides an additional indirection layer — the SA's permissions can be managed independently of the federation configuration.

### Why WIF is Secure

- **No long-lived secrets** — OIDC tokens expire in minutes
- **Audience restriction** — `aud` claim prevents cross-service replay
- **Issuer validation** — signature verified via JWKS; only configured issuers are trusted
- **Fine-grained IAM** — permissions bound to specific repos, branches, workflows
- **Federation must be explicitly configured** — without a Workload Identity Pool, Provider, and IAM binding, tokens are rejected

### Token Audience Summary

| Token Type | Audience | Purpose |
| --- | --- | --- |
| ID Token | Client | Authentication |
| Access Token | Resource Server | Authorization |
| Federated OIDC Token | STS | Identity assertion for token exchange |

## Interview Prep

### Q: Are OAuth 2.0 and OIDC competing protocols?

**A:** No. OIDC is built **on top of** OAuth 2.0. OAuth 2.0 is an authorization framework that answers "can this client access this resource?" by issuing access tokens. OIDC is an authentication layer that answers "who is the user?" by adding an ID token (always a JWT), standardized identity claims (`sub`, `email`, `name`), discovery endpoints, and validation rules. Every OIDC flow is an OAuth flow with the `openid` scope added.

### Q: What is the difference between an Access Token and an ID Token?

**A:** The access token is for **authorization** — it is sent to a resource server (API) to prove the client has permission. It can be opaque or a JWT, and its format is not standardized by OAuth. The ID token is for **authentication** — it is consumed only by the client to verify who the user is. It is always a JWT with standardized claims (`iss`, `sub`, `aud`, `exp`, `nonce`). The ID token should **never** be sent to a resource server.

### Q: If an OAuth access token is a JWT with `sub` and `email` claims, can I use it for login?

**A:** You should not. Even when an access token happens to be a JWT with identity-like claims, OAuth does not standardize the claim structure, and critically, the access token's `aud` is the resource server, not the client. Using it for login violates the trust model. OIDC solves this by introducing the ID token, which has `aud` set to the client's `client_id` and comes with mandatory validation rules.

### Q: What triggers OIDC mode vs plain OAuth?

**A:** The `openid` scope. When the authorization request includes `scope=openid`, the authorization server operates in OIDC mode and returns an ID token alongside the access token. Without `openid`, it is a plain OAuth 2.0 flow with no ID token.

### Q: Walk through the SAML SP-initiated SSO flow end-to-end.

**A:** (1) The user visits the application (Service Provider). (2) The SP determines the user is unauthenticated and generates a SAML `AuthnRequest`. (3) The SP redirects the browser to the IdP's SSO URL with the AuthnRequest (Base64-encoded via HTTP-Redirect or form POST). (4) The IdP authenticates the user (password, MFA, or re-uses existing session for SSO). (5) The IdP generates a SAML `Response` containing a signed `Assertion` with the user's NameID, AuthnStatement, and optional AttributeStatement. (6) The IdP returns the SAMLResponse via an auto-submitting HTML form that POSTs to the SP's Assertion Consumer Service (ACS) URL. (7) The SP validates the XML signature against the IdP's certificate, checks conditions (audience, time validity, InResponseTo), extracts the user identity, and creates a session.

### Q: Why is `sub` preferred over `email` as a user identifier in OIDC?

**A:** The `sub` claim is a stable, unique identifier for the user within the scope of the issuer. It does not change even if the user updates their email. Emails can change, may not be unique across tenants/providers, may be unverified, and some providers may not return them for all users. Best practice: use `sub` (or `(iss, sub)` for cross-issuer uniqueness) as the primary key for user identity.

### Q: How does PKCE protect against authorization code interception?

**A:** Without PKCE, if an attacker intercepts the authorization code (e.g., via a malicious app registered for the same custom URI scheme on mobile), they can exchange it for tokens. PKCE prevents this: the legitimate client generates a random `code_verifier`, derives a `code_challenge` = SHA256(code_verifier), and sends the challenge in the authorization request. When exchanging the code, the client sends the original `code_verifier`. The authorization server verifies that SHA256(code_verifier) matches the stored challenge. The attacker has the code but not the verifier, so the exchange fails.

### Q: Walk through how GitHub Actions authenticates to GCP using Workload Identity Federation.

**A:** (1) The workflow has `permissions: id-token: write`. The runner requests an OIDC token from GitHub's IdP (`token.actions.githubusercontent.com`). (2) The runner sends a token exchange request to Google STS with the JWT as `subject_token`. (3) Google STS fetches GitHub's JWKS, verifies the signature, checks `iss`/`aud`/`exp`. (4) Google maps JWT claims to attributes using the configured mapping. (5) Google constructs a federated principal and evaluates it against IAM bindings. (6) If the principal matches, Google issues a short-lived GCP access token (or SA impersonation token). (7) The runner uses this token to call GCP APIs.

### Q: Why is the `aud` claim critical in OIDC and WIF?

**A:** The `aud` (audience) claim specifies who the token is intended for. In OIDC, the ID token's `aud` must match the client's `client_id` — preventing one app from accepting an ID token meant for another (token confusion). In WIF, the GitHub OIDC token's `aud` is set to `https://sts.googleapis.com` — only Google STS should accept it. If an attacker replays this token against AWS STS, it is rejected because the `aud` does not match.

### Q: What is the difference between `principal://` and `principalSet://` in GCP WIF IAM bindings?

**A:** `principal://` matches a **single specific identity** — typically by `sub` claim (e.g., a specific repo + branch). `principalSet://` matches a **group of identities** based on an attribute (e.g., all workflows where `repository == myorg/myrepo`, regardless of branch). Use `principal://` for the most restrictive bindings (only main branch deploys), `principalSet://` for broader bindings (any branch in a trusted repo).

### Q: When should you use SAML vs OIDC vs OAuth?

**A:** Use **SAML** for enterprise browser-based SSO into legacy/established SaaS (Salesforce, Workday, ServiceNow). Use **OIDC** for modern app login, SPAs, mobile apps, and anywhere you need both identity and API access — simpler (JSON vs XML), supports discovery, works with mobile/native clients. Use **OAuth 2.0** (without OIDC) for pure delegated API access where you don't need to know who the user is. If you need both login and API access, use OIDC (gives ID + access tokens in one flow).

---

## GitHub Actions Token Anatomy: Opaque vs Structured Tokens

### Two Categories of Tokens

Tokens fall into two structural categories: **opaque** and **structured**.

**Opaque tokens** are random high-entropy strings. There is no data inside — the issuer's database is the only thing that knows what the token means. Validation requires a "phone home" to the issuer. GitHub tokens (`ghp_` for PATs, `ghs_` for installation tokens) are opaque. Think of them as physical keys — you can't tell what door they open by looking at the metal.

**Structured tokens** are data containers. A JWT (JSON Web Token) is the most common structured format. It's three Base64-encoded segments separated by dots: `Header.Payload.Signature`. Anyone with the token string can decode the payload and read its claims. Validation can happen offline by checking the cryptographic signature against the issuer's public keys.

| Property | Opaque Token | Structured Token (JWT) |
| --- | --- | --- |
| Format | Random string | Base64-encoded JSON, 3 dot-separated parts |
| Readability | None | Decode the payload to see claims |
| Validation | Must call the issuer's API | Can verify offline via signature |
| Size | Small | Large (embedded data) |
| Used for | API authorization | Identity federation / OIDC |

### JWT Anatomy (GitHub Actions OIDC)

A JWT has three parts:

**Header** — metadata about the token: algorithm (`RS256`) and type (`JWT`).

**Payload (Claims)** — the actual data. In a GitHub Actions OIDC token:

```json
{
  "sub": "repo:octocat/hello-world:ref:refs/heads/main",
  "aud": "https://github.com/octocat",
  "repository": "octocat/hello-world",
  "repository_owner": "octocat",
  "run_id": "123456789",
  "workflow": "My CI Workflow",
  "iss": "https://token.actions.githubusercontent.com",
  "iat": 1709500000,
  "exp": 1709500600
}
```

Standard claims: `iss` (issuer — who created it), `sub` (subject — who it's about), `aud` (audience — who it's for), `exp` (expiration), `iat` (issued at).

**Signature** — cryptographic hash proving the payload hasn't been tampered with.

The payload is **not encrypted**. It's only Base64 encoded. Anyone with the token string can read the repository name, branch, workflow, and other metadata.

To decode a JWT in the terminal:

```bash
echo $ID_TOKEN | cut -d'.' -f2 | base64 --decode | jq
```

### `GITHUB_TOKEN` vs OIDC Token

These are the two tokens available in a GitHub Actions workflow. They serve completely different purposes.

**`GITHUB_TOKEN`** — An **Installation Access Token** — opaque, OAuth-style. Automatically provided by GitHub in every workflow run. Used to **do things** on GitHub: create PRs, read repos, push code. Sent as:

```http
Authorization: Bearer ghs_aBcDeFgHiJkLmNoP...
```

**OIDC Token** — A **JWT** issued by `https://token.actions.githubusercontent.com`. Used to **prove identity** to external services (AWS, GCP, Azure, Vault). Does not grant any permissions to the GitHub API. Requires explicit workflow permission:

```yaml
permissions:
  id-token: write   # required to request the JWT
  contents: read
```

The OIDC token says "I am the runner for the main branch of octocat/hello-world." The `GITHUB_TOKEN` says "I have permission to write to this repository." These are fundamentally different assertions.

### The 401 Trap: Context Mismatch

| Token Type | Destination | Result |
| --- | --- | --- |
| `GITHUB_TOKEN` (opaque) | `api.github.com` | Success |
| OIDC JWT (structured) | `api.github.com` | **401 Unauthorized** |
| OIDC JWT (structured) | AWS / GCP / Vault | Success (identity exchange) |

GitHub's API expects an opaque permission token. When it receives a JWT, the string doesn't match any known token prefix (`ghs_`, `ghp_`). The API rejects it immediately — 401.

### Token vs User-Agent Independence

The `Authorization` header and `User-Agent` header are completely independent. Switching from OIDC to PAT to `GITHUB_TOKEN` changes the value of `Authorization` but has zero effect on `User-Agent`. If your tool is written in Go, the server sees `Go-http-client/2.0` regardless of which token you use. The token is your ID card; the User-Agent is the vehicle you're driving.

### The Empty Token Trap

The most common silent failure in GitHub Actions:

1. Workflow is missing `permissions: id-token: write`.
2. The token request silently returns an empty value.
3. The downstream tool sends an empty `Authorization` header.
4. GitHub returns **401 Unauthorized**.

Debug with:

```bash
echo "Token length: ${#MY_TOKEN}"
```

### Fixing the 401

**If calling the GitHub API:** use `GITHUB_TOKEN`, not OIDC.

```yaml
steps:
  - run: ./my-go-tool
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**If calling AWS/GCP/Vault:** use the OIDC token exchange flow.

```yaml
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/my-role
      aws-region: us-east-1
```

Quick troubleshooting:

| Check | Fix |
| --- | --- |
| Permissions | Ensure `id-token: write` is in the workflow |
| Token populated | `echo "Token length: ${#MY_TOKEN}"` |
| Issuer URL | Tool must expect `https://token.actions.githubusercontent.com` |
| Audience | `aud` claim must match what the receiver expects |
| Header format | Must be `Authorization: Bearer <TOKEN>` — not missing "Bearer" |

### Token Anatomy Interview Prep

**Q: What is the structural difference between an opaque token and a JWT?**

**A:** An opaque token is a random string with no embedded data — validation requires calling back to the issuer's server. A JWT is a self-contained data structure with three Base64-encoded parts (Header, Payload, Signature) separated by dots. The payload contains claims like `iss`, `sub`, `aud`, `exp`. Validation can happen offline: you fetch the issuer's public keys (via the JWKS endpoint), verify the signature, and check the claims. The tradeoff is that JWTs are larger and their contents are readable by anyone (they're encoded, not encrypted), while opaque tokens reveal nothing to an interceptor without the issuer's database.

**Q: Walk through what happens when a GitHub Actions workflow sends an OIDC JWT to `api.github.com` instead of `GITHUB_TOKEN`.**

**A:** The workflow requests an OIDC token from `https://token.actions.githubusercontent.com`. This returns a JWT like `eyJhbGciOiJSUzI1NiIs...`. The Go tool then sends an HTTP request to `api.github.com` with `Authorization: Bearer eyJhbGciOiJSUzI1NiIs...`. GitHub's API gateway receives the request and inspects the Authorization header. It first checks the token prefix — valid GitHub tokens start with `ghs_` (installation), `ghp_` (PAT), `gho_` (OAuth), or `github_pat_` (fine-grained PAT). The JWT string doesn't match any prefix. The gateway attempts a database lookup — no match. It doesn't attempt JWT verification because the GitHub REST API is not an OIDC relying party — it doesn't trust tokens from its own OIDC provider for API access. The gateway returns `401 Unauthorized`. The fix: use `GITHUB_TOKEN` for API calls, and OIDC JWT only for authenticating to external cloud providers via token exchange.

**Q: Why does the `User-Agent` header not change when you switch authentication tokens?**

**A:** `Authorization` and `User-Agent` are independent HTTP headers serving different purposes. `Authorization` carries the credential — it changes when you switch tokens. `User-Agent` identifies the software making the request — it's set by the HTTP client library, not by the authentication mechanism. A Go program using `net/http` sends `Go-http-client/2.0` regardless of whether the Authorization header contains a PAT, a `GITHUB_TOKEN`, or an OIDC JWT.

**Q: What is the most common cause of a silent 401 in GitHub Actions when OIDC is configured?**

**A:** Missing `permissions: id-token: write` in the workflow YAML. Without this permission, the call to the GitHub OIDC provider to mint a JWT silently returns an empty string (no error thrown). The downstream step runs anyway and sends an HTTP request with `Authorization: Bearer ` (empty value). The API returns 401. The fix is twofold: add the permission, and defensively check `echo "Token length: ${#MY_TOKEN}"` in the workflow.

**Q: Is `GITHUB_TOKEN` an OAuth token or an OIDC token?**

**A:** Neither, precisely. It's an **Installation Access Token** — a short-lived, automatically-scoped token that GitHub generates for the GitHub App installation associated with the repository. It behaves like an OAuth 2.0 Bearer token (opaque string, sent in the Authorization header, validated by GitHub's servers), but it's not issued via an OAuth flow. It's automatically injected into every workflow run with permissions scoped to the triggering repository. The OIDC token is entirely separate — it's a JWT issued by `https://token.actions.githubusercontent.com` and must be explicitly requested with `permissions: id-token: write`.

**Q: Can you read the contents of a `GITHUB_TOKEN`?**

**A:** No. It's an opaque token — a random string prefixed with `ghs_`. There are no embedded claims, no payload to decode. To determine what permissions it has, you must call `https://api.github.com` with it and inspect the response headers (`X-OAuth-Scopes`) or make an introspection-style call. In contrast, an OIDC JWT can be decoded by anyone: `echo $TOKEN | cut -d'.' -f2 | base64 --decode | jq` reveals all claims including the repository, branch, actor, and workflow.

## See also

- [[notes/AuthNZ/mcp-oauth|MCP OAuth 2.1]]
- [[notes/Git/user-agent|GitHub API User-Agent]]
- [[notes/Git/git-proactiveauth|Git proactiveAuth]]
- [RFC 6749 — OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 7636 — PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 8693 — OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [SAML 2.0 Technical Overview (OASIS)](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)
- [Google Workload Identity Federation Docs](https://cloud.google.com/iam/docs/workload-identity-federation)
- [GitHub OIDC Documentation](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)

---

## Related Notes

- [[notes/K8s/kubelens-mcp-server|KubeLens MCP Server]] -- practical implementation of OAuth 2.0 proxy with Dynamic Client Registration (RFC 7591), PKCE (RFC 7636), and server-side session management in Go
