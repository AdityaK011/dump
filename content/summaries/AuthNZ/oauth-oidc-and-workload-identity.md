---
title: "Summary: OAuth, OIDC & Workload Identity Federation"
---

> **Full notes:** [[notes/AuthNZ/oauth-oidc-and-workload-identity|OAuth, OIDC & Workload Identity Federation -->]]

## Key Concepts

### SSO & SAML
- SSO = sign in once, access multiple apps via federation protocol
- SAML: XML-based, browser POST/redirect, enterprise SSO (Salesforce, Workday)
- SP-initiated flow: SP redirects to IdP -> user authenticates -> IdP POSTs signed SAMLResponse to ACS URL
- Gotchas: clock skew, cert rotation, replay protection (track InResponseTo), XML signature wrapping attacks

### JWT Structure
- Three Base64url segments: `Header.Payload.Signature`
- Header: `alg`, `typ`, `kid` (selects public key from JWKS)
- Payload: claims (`iss`, `sub`, `aud`, `exp`, `iat`, custom claims)
- **Not encrypted** -- anyone can decode payload; never store secrets in JWTs
- Decode: `echo $TOKEN | cut -d'.' -f2 | base64 --decode | jq`

### OAuth 2.0 (Authorization)
- Answers: "Can this client access this resource on behalf of this user?"
- Authorization Code Flow + PKCE (recommended for all clients per OAuth 2.1):
  1. Client generates `code_verifier` + `code_challenge` (SHA-256)
  2. Redirect to `/authorize` with challenge
  3. User authenticates + consents
  4. Auth code returned to redirect_uri
  5. Exchange code + verifier at `/token`
  6. Receive access_token + refresh_token
- PKCE prevents authorization code interception (attacker has code but not verifier)
- Access token: for API calls (opaque or JWT, short-lived)
- Refresh token: for getting new access tokens (opaque, long-lived)

### OIDC (Authentication)
- Built ON TOP of OAuth 2.0; adds identity layer
- Triggered by `scope=openid` -- without it, plain OAuth (no ID token)
- ID Token: always JWT, consumed ONLY by client, never sent to APIs
- Access Token: sent to resource server (API), never used for login
- ID Token validation: signature (JWKS), `iss`, `aud` (must match client_id), `exp`, `iat`, `nonce`
- Discovery: `/.well-known/openid-configuration` -> endpoints, JWKS URI, supported scopes
- Use `sub` (not email) as stable identifier; cross-issuer: `(iss, sub)` tuple

### Workload Identity Federation (WIF)
- External workloads authenticate to cloud without long-lived service account keys
- GitHub Actions -> OIDC token -> Google STS token exchange -> GCP access token
- GitHub OIDC token `sub`: `repo:myorg/myrepo:ref:refs/heads/main`
- GCP flow: OIDC claims -> attribute mapping -> federated principal -> IAM policy evaluation
- `principal://`: single specific identity (exact sub match)
- `principalSet://`: group of identities by attribute (e.g., all workflows from a repo)
- SA impersonation: federated identity gets `workloadIdentityUser` on SA -> short-lived SA token

### GitHub Actions Token Types
- **GITHUB_TOKEN** (`ghs_`): opaque, Installation Access Token, for GitHub API calls, auto-scoped
- **OIDC Token**: JWT from `token.actions.githubusercontent.com`, for external identity exchange
- Sending OIDC JWT to `api.github.com` -> 401 (GitHub API expects opaque token, not JWT)
- Missing `permissions: id-token: write` -> silently empty token -> 401
- Debug: `echo "Token length: ${#MY_TOKEN}"`

## Quick Reference

```
OAuth vs OIDC:
  OAuth 2.0: authorization ("can this client access?") -> access token
  OIDC: authentication ("who is the user?") -> ID token (JWT)
  OIDC = OAuth + scope=openid + ID token + discovery + JWKS

Token Audiences:
  ID Token     -> aud: client_id (consumed by client for login)
  Access Token -> aud: resource server (sent to API)
  WIF OIDC     -> aud: https://sts.googleapis.com (identity exchange)

SAML vs OIDC:
  SAML: XML, browser POST, enterprise SSO, complex (certs, metadata)
  OIDC: JSON/JWT, redirect + back-channel, modern apps/SPAs/mobile, simpler

WIF GitHub Actions -> GCP:
  1. permissions: id-token: write
  2. Runner requests JWT from GitHub IdP
  3. POST /token to Google STS (grant_type=token-exchange)
  4. Google validates JWT (JWKS, iss, aud, exp)
  5. Maps claims to attributes -> federated principal
  6. Evaluates IAM bindings -> issues GCP access token

Access Control Granularity:
  principalSet://...attribute.repository/myorg/myrepo  (any branch)
  principal://...subject/repo:myorg/myrepo:ref:refs/heads/main  (main only)

Common 401 Causes in GitHub Actions:
  - Missing permissions: id-token: write
  - Sending OIDC JWT to api.github.com (use GITHUB_TOKEN instead)
  - Empty token (silently failed token request)
  - Wrong aud claim for destination service
```
