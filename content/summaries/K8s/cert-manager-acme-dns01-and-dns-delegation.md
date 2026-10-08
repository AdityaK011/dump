---
title: "Summary: cert-manager and ACME DNS-01: Issuers, Challenges, and DNS Delegation"
---

> **Full notes:** [[notes/K8s/cert-manager-acme-dns01-and-dns-delegation|cert-manager and ACME DNS-01: Issuers, Challenges, and DNS Delegation -->]]

## Key Concepts

### Why DNS-01
- Only ACME challenge that issues **wildcards**; CA never needs to reach your service.
- Works for internal names, multi-cluster/multi-LB, and before the LB exists.
- Cost: the client needs **DNS write credentials**, used on every renewal, forever.
- HTTP-01 = port 80 token; TLS-ALPN-01 = port 443 `acme-tls/1`; neither does wildcards.

### DNS from zero (the mental model)
- DNS is a chain of "not me, ask them": nobody holds the full list, each level knows its own piece plus who's next.
- Read names right to left: root `.` → `com` → `example.com` → `www`.
- **Record** = one fact. **Zone** = a list of records one party manages. **Domain** = a name and everything under it. Not the same thing.
- **Authoritative nameserver** holds a zone, gives the real answer. **Resolver** (`8.8.8.8`, office DNS) holds nothing, walks the chain, caches.
- **Delegation** = NS record in the parent pointing at the child's nameservers. Parent holds the signpost, child holds the records.
- Resolvers start from built-in **root hints**. The **registrar** sets the TLD's signpost to your nameservers.
- **Reads** = DNS on port 53, open to anyone. **Writes** = provider's HTTPS API + IAM. Separate zones = separate write permissions.
- TTL = how long answers (including "doesn't exist") are cached.
- `dig +short NS <name>`, `dig +trace <name>`, `dig @<authoritative-ns> <name>` to see each piece.
- Apex and wildcard both validate at `_acme-challenge.<domain>`, so delegating that one name covers both.

### DNS fundamentals
- **Zone** = administrative cut with its own SOA and authoritative servers; not the same as a domain.
- **Authoritative** servers own zone data (AA bit). **Recursive** resolvers own nothing, walk and cache.
- Cold resolver starts from **root hints**: 13 root identities (`a`–`m.root-servers.net`), anycast from 1,000+ sites; priming query per RFC 8109.
- Walk: root → TLD referral → zone NS referral → authoritative answer. **Glue** needed when NS names are inside the zone.
- **Delegation** = NS records in the parent at the child name. Nothing syncs. Delete child but keep NS = dangling delegation, takeover risk.
- **Negative caching** (RFC 2308) = min(SOA TTL, SOA MINIMUM). Querying before the TXT exists can hide it for that long. Keep challenge TTLs at 30–120s.
- CNAME can't coexist with other data at a name (RFC 1034 §3.6.2).

### Writes vs reads
- **Writes** go through the provider's HTTPS API and IAM (RFC 2136 dynamic update + TSIG exists, managed providers don't expose it).
- **Reads** go through DNS, follow CNAMEs and delegations freely.
- API success is not visibility: replication to the authoritative fleet takes seconds to a minute.
- Write scope = whatever the provider IAM supports, usually per zone.

### ACME (RFC 8555)
- Account key signs every JWS. **Order** → one **authorization** per name → several **challenges**.
- Valid authorizations are reused (Let's Encrypt up to 30 days), so renewals can skip challenges.
- TXT at `_acme-challenge.<name>` = `base64url(SHA-256(token + "." + JWK thumbprint))` (RFC 7638 thumbprint).
- Wildcard `*.example.com` validates at `_acme-challenge.example.com`, same as apex. Apex + wildcard = **two TXT values in one RRset** at once.
- **CAA** (RFC 8659) + `accounturi` / `validationmethods` (RFC 8657) restrict which account and method can issue.

### cert-manager objects
- `Issuer` (namespaced, secrets from own ns) vs `ClusterIssuer` (cluster-wide, secrets from `--cluster-resource-namespace`).
- Chain: `Certificate` → `CertificateRequest` → `Order` → `Challenge` (one per name) → `Secret`.
- `Certificate` = desired state; generates key into a temp Secret (`cert-manager.io/next-private-key`), only the CSR leaves the cluster.
- `CertificateRequest` must be **approved** first; built-in approver approves everything unless replaced.
- `rotationPolicy: Always` default since **v1.18** (was `Never`).
- Renewal default at **1/3 lifetime remaining** (90d cert → day 60). Let's Encrypt moving to 45-day certs.
- Orders/Challenges are GC'd after success: debug while it's broken.

### DNS-01 flow
1. Compute TXT value. 2. Find zone by **SOA walk** up the labels. 3. Present via provider API. 4. **Self-check** against authoritative NS directly. 5. Tell CA to validate. 6. CA checks from multiple vantage points. 7. Clean up TXT. 8. Finalize CSR, write Secret.
- `--dns01-recursive-nameservers` for split-horizon clusters; `--dns01-recursive-nameservers-only` when port 53 egress is blocked.
- Set `hostedZoneID` / `hostedZoneName` to avoid zone-list permissions.

### Solver selectors
- Most specific wins: exact `dnsNames` > longest `dnsZones` suffix > most `matchLabels`; ties go to list order.
- **No selector = matches every name** (fallback).

### Delegated `_acme-challenge` pattern
- Parent `example.com` at Provider A; challenge zone `acme.example.com` at Provider B, delegated via NS.
- Variant 1: `_acme-challenge.app.example.com CNAME app.acme.example.com` + solver `cnameStrategy: Follow`.
- Variant 2: NS-delegate each `_acme-challenge.<name>` as its own zone; no CNAME following, one zone per name.
- cert-manager credential only writes the challenge zone (zone-scoped tokens, zone-level IAM, Route 53 record-name/type condition keys).
- Removes traffic hijack (A, MX, NS). **Does not** prevent issuance for CNAMEd names.
- Gotchas: other tools writing TXT at the CNAME'd name break; remove CNAMEs when decommissioning.

### Ambient credentials
- = controller's own identity (metadata server, GKE WI, IRSA/Pod Identity, Azure WI, env vars).
- `--cluster-issuer-ambient-credentials=true`, `--issuer-ambient-credentials=false` by default.
- Off for Issuers because one controller serves every namespace: a tenant's Issuer would borrow the platform's DNS identity. **Confused deputy**, not granted by the tenant's RBAC.
- Explicit secrets stay inside the namespace boundary. Keyless option: `serviceAccountRef` (tenant-owned SA) where supported.

### Guardrails
- RBAC: who can create `Issuer` / `ClusterIssuer` (Helm chart aggregates cert-manager verbs into `admin`/`edit`).
- Admission (VAP GA **v1.30**): require non-empty `dnsZones` or `dnsNames` on every solver, then allowlist both.
- **Selector-less bypass**: a rule that iterates `dnsZones` passes vacuously for a solver with no selector, which matches everything. `matchLabels`-only is the same, labels are tenant-controlled.
- **Shared ClusterIssuer gap**: any namespace can reference it and get certs for another team's names. Fix with admission on `Certificate.dnsNames` or **approver-policy** (`CertificateRequestPolicy`, disable built-in approver).
- Outside the cluster: CAA `accounturi`, CT log monitoring.

## Quick Reference

```
WRITE (API + IAM)                          READ (DNS)
cert-manager ──▶ Provider B API            CA ─▶ resolver ─▶ root ─▶ com ─▶ Provider A
                 acme.example.com                       _acme-challenge.app CNAME app.acme
                 (TXT only)                             acme NS provider-b ─▶ Provider B: TXT

Certificate ─▶ CertificateRequest ─▶ Order ─▶ Challenge ─▶ TXT ─▶ validate ─▶ finalize ─▶ Secret
```

| Flag | Default | Why |
|---|---|---|
| `--cluster-issuer-ambient-credentials` | `true` | Only admins create ClusterIssuers |
| `--issuer-ambient-credentials` | `false` | Tenants mustn't borrow the controller's identity |
| `--dns01-recursive-nameservers` | unset | Bypass split-horizon in-cluster DNS |
| `--dns01-recursive-nameservers-only` | `false` | No direct authoritative queries (port 53 egress blocked) |

| Stuck symptom | Usual cause |
|---|---|
| Presented false, 403 | Credential scope or missing `cnameStrategy: Follow` |
| Propagation wait forever | Private zone in VPC, cached NXDOMAIN |
| Incorrect TXT (apex + wildcard) | Provider replaced RRset instead of appending |
| Issuer "no credentials" | Ambient off for Issuers, by design |

```bash
kubectl get certificate,certificaterequest,order,challenge -n app
kubectl describe challenge -n app <name>
cmctl status certificate app-tls -n app
dig +trace _acme-challenge.app.example.com TXT
dig @ns-b1.provider-b.net app.acme.example.com TXT +norecurse
```
