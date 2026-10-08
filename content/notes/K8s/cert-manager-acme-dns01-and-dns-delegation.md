---
title: "cert-manager and ACME DNS-01: Issuers, Challenges, and DNS Delegation"
---

## Why DNS-01 at all

There are three ways to prove to an ACME certificate authority that you control a name. HTTP-01 makes you serve a token on port 80, TLS-ALPN-01 makes you answer a special TLS handshake on port 443, and DNS-01 makes you publish a TXT record. The first two need the CA to reach your service from the internet. DNS-01 only needs the CA to resolve a name.

That one difference is why DNS-01 shows up everywhere in platform work:

- It's the only ACME challenge that can issue **wildcard** certificates (RFC 8555 §7.1.3 and the Let's Encrypt policy built on it).
- It works for **internal services** that have public names but no public ingress.
- It works **before the load balancer exists**, so you can have a certificate ready when the first pod comes up.
- It doesn't care how many clusters or replicas serve the name, because nothing is served.

The price is that the thing requesting certificates needs **write access to DNS**, and DNS write access is one of the most powerful credentials in an organisation. Whoever can write `example.com` can redirect its traffic, intercept its mail, and get certificates for it. Most of this note is about getting the first property without handing out the second.

The note is in four parts. It opens with DNS from zero, a mental model you can rebuild from memory, with commands to watch each piece happen. Then the more precise DNS details you need to reason about the challenge. Then the ACME and cert-manager resource model and the end-to-end flow. Then the security design: delegated challenge zones, ambient credentials, and the admission guardrails that keep namespace tenants inside their lane.

---

## DNS from zero

If you only remember one section of this note, make it this one. Everything later, from delegation to the challenge zone to the credentials question, is this model applied.

### The problem DNS solves

Computers talk to IP addresses like `93.184.216.34`. Humans want to type `www.example.com`. DNS is the system that turns the second into the first, and it does it for billions of names owned by millions of different organisations, with no single company holding the full list.

That last part is the whole design. Nobody could maintain one giant file of every name on the internet, so DNS splits the job up. Each owner keeps the list for their own names, and there's a chain of pointers that tells you whose list to read.

### The mental model: a chain of "not me, ask them"

Picture asking for directions in a building where nobody knows everything:

1. You walk into the lobby and ask "where's Alice in accounts at the Example company?" The receptionist doesn't know Alice, but knows which floor each company is on. "Example is on floor 12, ask their front desk."
2. Floor 12's front desk doesn't know every desk either, but knows the departments. "Accounts is room 1204, ask there."
3. Room 1204 knows Alice. "Desk 7."

Nobody holds the whole map. Each person knows their own piece plus who to send you to next. DNS works the same way, reading the name **right to left**:

```
www.example.com.
               ^ the root  (the lobby: knows who runs every top-level domain)
           ^^^   com        (floor 12: knows who runs every *.com name)
   ^^^^^^^       example    (room 1204: knows the actual records for example.com)
^^^              www        (desk 7: the answer, an IP address)
```

The trailing dot is the root. You almost never type it, but it's always there, and DNS tools print it.

### The four words you need

**Record.** One fact about one name, like "`www.example.com` has IP `93.184.216.34`". It has a name, a type, a value and a TTL (how long anyone may cache it).

**Zone.** A list of records that one party manages and publishes together. The `example.com` zone is the list the owner of `example.com` keeps. Think of a zone as one file, or one table in a database.

**Nameserver.** A server that publishes a zone and answers questions about it. An *authoritative* nameserver is the one that actually holds the zone, the room 1204 in the analogy. Its answers are the real ones.

**Resolver.** A server that holds nothing of its own. It does the walking for you, asks each nameserver in turn and remembers the answers for a while. `8.8.8.8`, `1.1.1.1`, your office network's DNS server and your cluster's upstream DNS are all resolvers. It's the person who walks around the building on your behalf and remembers the directions for next time.

Keep authoritative and resolver apart in your head. Authoritative servers own the truth and answer questions about their own zones only. Resolvers own nothing, ask the authoritative ones and cache.

### Domain is not the same as zone

This trips everyone up, so here it is plainly. A *domain* is just a name and everything under it: `example.com` and `www.example.com` and `eu.example.com` and so on. A *zone* is who manages which part.

Usually the owner of `example.com` manages everything under it in one zone. But they can cut a piece out and hand it to someone else. If `eu.example.com` is handed to another team or provider, then the domain `example.com` still includes `eu.example.com`, but the `example.com` *zone* no longer holds those records. Another zone does.

That handing-out is called **delegation**, and it's the same move the root makes for `com` and `com` makes for `example.com`. Every level of DNS is built from it.

### Delegation: an NS record is a signpost

How does the lobby know Example is on floor 12? It has a note for it. In DNS that note is an **NS record**: "for anything under this name, ask these nameservers".

The `com` zone holds an NS record for `example.com` pointing at the provider that hosts it:

```
; in the com zone (run by the .com registry)
example.com.   NS   ns1.provider-a.net.
example.com.   NS   ns2.provider-a.net.
```

And if `example.com` wants to hand `eu.example.com` to someone else, it adds the same kind of record in its own zone:

```
; in the example.com zone (hosted at Provider A)
eu.example.com.   NS   ns1.provider-b.net.
eu.example.com.   NS   ns2.provider-b.net.
```

Now the full tree looks like this:

```
. (root zone)               "com?  ask the com servers"
└── com                     "example.com?  ask Provider A"
    └── example.com         holds www, app, mail ... (Provider A)
        └── eu.example.com  "ask Provider B"  → holds api.eu, web.eu (Provider B)
```

The one rule to remember: **the parent holds a signpost (NS records), the child holds the records.** Nothing syncs between them. If the signpost points at the wrong servers, or at servers that no longer host the zone, lookups break.

### How the top of the chain gets set

Two questions that come up once you see the chain:

*How does a resolver find the root?* It doesn't look it up. Every resolver ships with a short built-in list of root server addresses, called root hints. There are 13 named root servers (`a.root-servers.net` to `m.root-servers.net`), each copied to many locations worldwide. They change so rarely that baking them in is fine.

*How does `com` know about `example.com`?* Through the registrar. When you buy `example.com` from a registrar, you tell them which nameservers host it (your DNS provider gives you those names). The registrar passes that to the `.com` registry, and the registry adds the NS records to the `com` zone. That's the only part of the chain you don't edit directly. Everything below your own domain, you control yourself.

### A lookup, start to finish

Your laptop wants `www.example.com`:

1. Your laptop asks its resolver. Your laptop never walks the chain itself. It just asks one resolver and waits.
2. The resolver asks a root server: "`www.example.com`?" The root replies "I don't know, but `com` is run by these servers."
3. The resolver asks a `com` server. It replies "I don't know, but `example.com` is run by `ns1.provider-a.net`."
4. The resolver asks `ns1.provider-a.net`. That server holds the `example.com` zone, so it answers: "`www.example.com` is `93.184.216.34`."
5. The resolver gives the answer to your laptop and caches every step. The next person asking about anything under `com` skips step 2, and anyone asking about `www.example.com` again within the TTL gets the cached answer straight away.

Steps 2 and 3 are called referrals: "not me, ask them". Step 4 is the authoritative answer.

### Writing is a different path from reading

This is the part that's easy to miss and matters most for certificates.

Everything above is **reading**. Reading uses the DNS protocol, port 53, and is open to everyone. Any resolver on the internet can ask any authoritative server anything.

**Writing** (adding or changing a record) doesn't go through DNS at all. You change records through your DNS provider: their web console, their HTTPS API, or Terraform calling that API. The provider checks your login or API token, saves the change in its database and then publishes it to its nameservers, usually within seconds to a minute.

```
 you / cert-manager                                anyone on the internet
        │                                                  │
        │ WRITE: HTTPS API + login/IAM                     │ READ: DNS query, port 53
        ▼                                                  ▼
  provider's database  ── publishes to ──▶  provider's authoritative nameservers
```

So "who can change `example.com`?" is answered by the provider's permissions, not by DNS. And because a zone is the unit most providers hand out permissions for, splitting names into separate zones is how you give someone write access to a small piece without the rest. That's exactly what the delegated challenge zone later in this note does.

### Caching and TTL

Every record carries a TTL in seconds. A resolver may reuse an answer for that long without asking again. A long TTL means fewer lookups but slow changes. A short TTL means changes show up fast.

Resolvers also cache "that name doesn't exist". If you look up a record before it's been created, a resolver can keep telling you it doesn't exist for a while after you create it. For certificate challenges that's a real trap, and it's covered in more detail below.

### The record types you'll actually meet

| Type | Says | Example |
|------|------|---------|
| `A` | This name's IPv4 address | `www.example.com. A 93.184.216.34` |
| `AAAA` | This name's IPv6 address | `www.example.com. AAAA 2606:2800:...` |
| `CNAME` | This name is an alias, go look up that other name instead | `shop.example.com. CNAME shops.vendor.net.` |
| `NS` | The nameservers for this zone, or a signpost to a child zone | `eu.example.com. NS ns1.provider-b.net.` |
| `TXT` | Free text, used for proofs of ownership | `_acme-challenge.example.com. TXT "q8F3..."` |
| `MX` | Where to deliver this domain's email | `example.com. MX 10 mail.example.com.` |
| `SOA` | Admin info for a zone, marks where the zone starts | One per zone, at the top |
| `CAA` | Which certificate authorities may issue for this name | `example.com. CAA 0 issue "letsencrypt.org"` |

A `CNAME` and an `NS` record both send you elsewhere, but differently. A CNAME says "this one name is really that other name". An NS record says "this whole branch of the tree belongs to those servers".

### See it yourself

Ten minutes with `dig` will fix all of this in memory far better than rereading. Pick any real domain you have access to.

```bash
# Who are the authoritative servers for a domain?
dig +short NS example.com

# Ask your resolver for a record (normal lookup)
dig +short A www.example.com

# Ask an authoritative server directly, bypassing your resolver's cache
dig +short A www.example.com @ns1.provider-a.net

# Watch the whole chain: root -> com -> example.com -> answer
dig +trace www.example.com

# See a delegated child: NS records at a name below the apex
dig +short NS eu.example.com

# TXT records, as used by ownership challenges
dig +short TXT _acme-challenge.example.com

# Who may issue certificates for this name?
dig +short CAA example.com
```

In `dig +trace` output, each block is one step of the walk. The NS records at the bottom of each block are the signposts, and the server named in the next block's footer is who answered. When the NS records for a name point at a different provider than its parent's, you're looking at a delegation.

### The whole thing on one card

- DNS turns names into data (mostly IPs) without anyone holding the full list.
- Read names right to left: root, top-level domain, your domain, the host.
- A **zone** is a list of records one party manages. A **domain** is just a name and what's under it.
- **Authoritative** nameservers hold zones and give the real answers. **Resolvers** hold nothing, walk the chain for you and cache.
- **Delegation** is an NS record in the parent pointing at the child's nameservers. The parent holds the signpost, the child holds the records.
- Resolvers start from built-in **root hints**. Your registrar sets the signpost from the top-level domain to you.
- **Reads** go over DNS to anyone. **Writes** go through the provider's API and its permissions.
- Splitting names into separate zones is how you hand out write access to just one piece.
- **TTL** is how long answers are cached, including "doesn't exist" answers.

### Common misconceptions

*"My DNS server holds my records."* Your laptop's DNS server is a resolver. It holds a cache of other people's records, not your zone. Your zone lives at your DNS provider.

*"If I can see the record with dig, the CA can too."* You're seeing your resolver's view at this moment. The CA uses its own resolvers, from several places, with their own caches. Query the authoritative server directly (`dig @ns1...`) to see the truth.

*"Changing NS records at my provider moves my domain."* Not on their own. The signpost that matters for your domain lives in the parent zone (`com`), and only your registrar can change it. Editing the NS records inside your own zone doesn't change where the world sends queries.

*"A subdomain is automatically its own zone."* No. `eu.example.com` is just a name in the `example.com` zone until someone delegates it with NS records.

*"CNAME and NS do the same thing."* A CNAME redirects one name. NS hands over a whole subtree, and the child zone can then hold any records it likes.

---

## DNS fundamentals for certificate people

### Domains, zones, and records

A **domain** is a name in the tree: `app.example.com`. A **zone** is an administrative cut of that tree that one set of servers is authoritative for. They're often the same thing, which is where the confusion starts. `example.com` the zone might contain records for `example.com`, `www.example.com`, and `app.example.com`, while `eu.example.com` has been cut out into its own zone run by someone else.

```
                          . (root zone)
                          |
                         com (com zone, run by the registry)
                          |
                     example.com  ─────────── zone: example.com (Provider A)
                    /     |      \
                 www     app      eu  ─────── zone: eu.example.com (Provider B)
                                 /  \
                              api    web
```

Inside a zone, data lives in **resource record sets** (RRsets): every record with the same name, class, and type. The types that matter here:

| Type | Purpose | Relevance to ACME |
|------|---------|-------------------|
| `SOA` | Start of authority. One per zone, marks the apex. Holds the negative-cache TTL | cert-manager walks up the tree looking for SOA to find which zone a name lives in |
| `NS` | Names the authoritative servers for a zone. At a zone cut in the parent, it's a delegation | How the challenge subzone pattern works |
| `A` / `AAAA` | IPv4 / IPv6 address | Glue records for in-zone nameservers |
| `CNAME` | Alias: "this name is really that name". No other data may exist at a name with a CNAME (RFC 1034 §3.6.2, RFC 2181 §10.1) | Points `_acme-challenge` somewhere else |
| `TXT` | Arbitrary strings. An RRset can hold several values | The DNS-01 challenge token |
| `CAA` | Which CAs may issue for this name (RFC 8659), optionally which account and method (RFC 8657) | Last-line guardrail against mis-issuance |

### Authoritative servers vs recursive resolvers

There are two very different kinds of DNS server, and mixing them up explains most "it works on my laptop" DNS-01 failures.

An **authoritative nameserver** holds the zone data and answers only for the zones it serves. Ask it about a name it doesn't own and it either refuses or points you elsewhere. It sets the `AA` (authoritative answer) bit on its responses.

A **recursive resolver** owns nothing. It takes a question from a client, chases referrals from the root down to the authoritative servers, caches what it learns for the TTL, and returns the answer. Your laptop's `8.8.8.8`, the cluster's upstream resolver, and the CA's internal resolver fleet are all recursive resolvers.

### Root hints and the resolution walk

A recursive resolver with a cold cache has to start somewhere. It ships with a **root hints** file (traditionally `named.root`) that lists the 13 root server identities, `a.root-servers.net` through `m.root-servers.net`, with their addresses. Each identity is anycast from many sites, well over a thousand instances in total. On startup the resolver sends a priming query (RFC 8109) for `. NS` to one of them to get a fresh list.

Resolving `_acme-challenge.app.example.com TXT` from cold looks like this:

```
 Recursive resolver                                   Servers
 ──────────────────                                   ───────
   │  Q: _acme-challenge.app.example.com TXT ?
   ├─────────────────────────────────────────────────▶ root (a.root-servers.net)
   │◀───────────────────── referral: com. NS a.gtld-servers.net ... (+ glue A/AAAA)
   │
   ├─────────────────────────────────────────────────▶ com TLD server
   │◀───────────────────── referral: example.com. NS ns1.provider-a.net ...
   │
   ├─────────────────────────────────────────────────▶ ns1.provider-a.net (authoritative)
   │◀───────────────────── answer (AA): "q8F3...base64url..."   TTL 60
   │
   ▼  caches every step for its TTL, returns answer to client
```

Each step is a **referral**: "I don't have it, but these servers are authoritative for the next zone down". The referral comes from the NS records the parent holds at the zone cut. If a nameserver's own name is inside the zone it serves (`ns1.example.com` for `example.com`), the parent must also hand out its address as **glue**, otherwise the resolver would need to resolve `ns1.example.com` to find out how to resolve `example.com`.

You can watch this happen with `dig +trace`, which does the iteration itself instead of asking a resolver:

```bash
dig +trace _acme-challenge.app.example.com TXT
```

### Delegation is just NS records in the parent

To give `eu.example.com` to another provider, you create the zone at Provider B, get its nameserver names, and add NS records for `eu.example.com` in the `example.com` zone at Provider A:

```
; in zone example.com (Provider A)
eu.example.com.    3600  IN  NS  ns-b1.provider-b.net.
eu.example.com.    3600  IN  NS  ns-b2.provider-b.net.
```

From then on Provider A's servers answer queries under `eu.example.com` with a referral, and Provider B is the source of truth. Nothing synchronises between the two. If you later delete the zone at Provider B but leave the NS records in place, you have a **dangling delegation**, and if Provider B lets anyone create a zone with that name on those nameservers, someone else can take it over. Removing the NS records in the parent is part of decommissioning the child.

If the parent is DNSSEC-signed, it also needs a `DS` record for a signed child. Without one the child is an insecure delegation: it still resolves, it just isn't validated. That's fine for a challenge zone, as long as nobody adds a DS record pointing at keys the child doesn't have, because then validating resolvers return `SERVFAIL`.

### TTLs and negative caching

Resolvers cache positive answers for the record's TTL. They also cache "this name doesn't exist" (`NXDOMAIN`) and "this name exists but has no record of that type" (`NODATA`). RFC 2308 says the negative TTL is the smaller of the SOA record's own TTL and its `MINIMUM` field.

This bites DNS-01 in a specific way. If anything queries `_acme-challenge.app.example.com` before the TXT record exists, including your own debugging `dig`, a resolver can remember the absence for the negative TTL. A zone with a 3600-second SOA minimum can make a perfectly correct record invisible to that resolver for an hour. Keep the SOA minimum and the challenge record TTL low (30 to 120 seconds) on any zone used for challenges.

### Writes go through an API, reads go through DNS

This is the part people skip, and it's the key to the whole security model.

The DNS protocol is almost entirely read-only in practice. There is a standard write path, Dynamic Update (RFC 2136) authenticated with TSIG (RFC 8945), but managed DNS providers don't expose it. Instead every provider has its own HTTPS control-plane API with its own identity system:

```
          WRITE PATH (control plane)                    READ PATH (data plane)
          ──────────────────────────                    ──────────────────────

  cert-manager                                          ACME CA validators
      │                                                (several vantage points)
      │ HTTPS + provider IAM                                    │
      │ e.g. "create TXT in zone X"                             │ DNS/UDP+TCP 53
      ▼                                                         │ via recursive resolvers
 ┌──────────────────────┐   internal replication   ┌────────────▼─────────────┐
 │ Provider API         │ ───────────────────────▶ │ Authoritative servers    │
 │ (Route 53, Cloud DNS,│   seconds to a minute    │ (anycast, many PoPs)     │
 │  Cloudflare, ...)    │                          │                          │
 └──────────────────────┘                          └──────────────────────────┘
```

Three consequences fall out of this picture:

1. **Authorisation is the provider's IAM, not DNS.** "Who can change `example.com`" is answered by an API token, cloud IAM role, or TSIG key, and the granularity is whatever the provider supports: usually per zone, sometimes per record name.
2. **A successful API call doesn't mean the record is visible.** The API writes to a database, and the authoritative fleet picks it up some time later. Some providers expose this (Route 53 reports a change as `PENDING` then `INSYNC`), most don't.
3. **Different readers see different things at different times.** The authoritative servers might have the record while a recursive resolver is still serving a cached `NXDOMAIN`.

cert-manager has to cope with all three, which is why it has a propagation self-check (covered below).

---

## ACME in one page

ACME (RFC 8555) is the protocol between a client like cert-manager and a CA like Let's Encrypt. Every request is a JWS signed with the client's **account key**, so the CA always knows which account is asking.

```
 Client                                              ACME server
 ──────                                              ───────────
   │ newAccount (signed with account key)      ─────▶│  account URL
   │ newOrder {identifiers: [app.example.com]} ─────▶│  order: pending
   │◀───────────── authorizations: [authz URL] ──────│
   │ GET authz                                 ─────▶│
   │◀──── challenges: http-01, dns-01, tls-alpn-01 ──│  each with a token
   │
   │ (provision TXT record out of band)
   │
   │ POST challenge URL {}  "I'm ready"        ─────▶│
   │                                                 │── validators query DNS
   │ poll authz                                ─────▶│  authz: valid
   │ finalize {csr}                            ─────▶│  order: processing → valid
   │ GET certificate                           ─────▶│
   │◀─────────────────────── PEM chain ──────────────│
```

The vocabulary:

- An **order** asks for one certificate covering a set of identifiers.
- Each identifier gets an **authorization**, which is the CA's record of "has this account proven control of this name". Authorizations can be reused for a while (Let's Encrypt caches valid ones for up to 30 days), which is why a renewal sometimes completes with no challenge at all.
- Each authorization offers several **challenges**. The client picks one and completes it.

### What goes in the TXT record

For DNS-01 (RFC 8555 §8.4), the client builds a **key authorization** from the challenge token and the account key's JWK thumbprint (RFC 7638):

```
keyAuthorization = token + "." + base64url(SHA-256(JWK(accountKey)))
TXT value        = base64url(SHA-256(keyAuthorization))
record name      = _acme-challenge.<identifier>
```

So `app.example.com` is validated at `_acme-challenge.app.example.com`. For a wildcard, the `*.` is stripped: `*.example.com` is validated at `_acme-challenge.example.com`, the same name as the apex. An order for both `example.com` and `*.example.com` needs **two different TXT values at the same name at the same time**. That's legal because a TXT RRset can hold several values, but it trips up provider integrations that replace the RRset instead of appending to it.

Binding the value to the account key thumbprint means someone who sees the TXT record can't use it with a different account.

### The three challenge types

| | HTTP-01 | DNS-01 | TLS-ALPN-01 |
|---|---|---|---|
| Proof | File at `http://<name>/.well-known/acme-challenge/<token>` | TXT at `_acme-challenge.<name>` | Self-signed cert with `acmeIdentifier` extension, ALPN `acme-tls/1`, port 443 |
| Needs inbound traffic | Yes, port 80 | No | Yes, port 443 |
| Wildcards | No | Yes | No |
| Needs DNS write credentials | No | Yes | No |
| Multi-cluster / multi-LB | Painful, every endpoint must serve the token | Doesn't matter | Painful |
| Typical failure | LB, redirect, or WAF eats the request | Propagation, wrong zone, credentials | Terminating proxy in front |

### CAA: the CA-side guardrail

Before issuing, a CA must check CAA records for the name, walking up the tree to the first one it finds (RFC 8659). A CAA record can restrict issuance to one CA, and RFC 8657 adds parameters to restrict it further:

```
example.com.  3600  IN  CAA  0 issue "letsencrypt.org; validationmethods=dns-01; accounturi=https://acme-v02.api.letsencrypt.org/acme/acct/123456789"
example.com.  3600  IN  CAA  0 issuewild "letsencrypt.org; validationmethods=dns-01; accounturi=https://acme-v02.api.letsencrypt.org/acme/acct/123456789"
```

With `accounturi`, a stolen DNS write credential isn't enough to get a certificate through a different ACME account. Somebody would also need the account private key. It's cheap and underused.

---

## The cert-manager resource model

cert-manager turns the ACME exchange into a chain of Kubernetes objects, each reconciled by its own controller. Four live in `cert-manager.io/v1`, two in `acme.cert-manager.io/v1`.

```
  you write                      cert-manager creates
  ─────────                      ────────────────────

  Issuer / ClusterIssuer   (how to get certs: ACME server, account key, solvers)
          ▲
          │ issuerRef
  Certificate ──────────▶ CertificateRequest ──────▶ Order ──────▶ Challenge (one per name)
  (desired state:          (one CSR, one attempt)   (ACME order)  (one ACME challenge)
   names, secret,                 │                                     │
   duration)                      │                                     ▼
          │                       │                              TXT record at provider
          ▼                       ▼
     Secret (tls.crt, tls.key, ca.crt)  ◀──── issued chain copied back
```

### Issuer vs ClusterIssuer

Both describe *how* to obtain certificates: which CA, which ACME account, which solvers. The difference is scope and who's expected to write them.

| | `Issuer` | `ClusterIssuer` |
|---|---|---|
| Scope | Namespaced | Cluster-wide |
| Usable by | `Certificate`s in the same namespace | `Certificate`s in any namespace |
| Where it reads referenced Secrets (account key, DNS credentials) | Its own namespace | The **cluster resource namespace**, `--cluster-resource-namespace` (defaults to cert-manager's own namespace) |
| Typical author | Application team / namespace tenant | Platform team |
| Ambient credentials | Off by default (`--issuer-ambient-credentials=false`) | On by default (`--cluster-issuer-ambient-credentials=true`) |

A minimal DNS-01 ClusterIssuer:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@example.com
    privateKeySecretRef:
      name: letsencrypt-dns-account-key   # cert-manager generates the ACME account key here
    solvers:
    - selector:
        dnsZones: ["example.com"]
      dns01:
        cloudDNS:
          project: dns-project
          hostedZoneName: example-com     # skip zone discovery, so no list permission needed
```

Use the staging directory (`https://acme-staging-v02.api.letsencrypt.org/directory`) while you're building this. Production Let's Encrypt has rate limits that are easy to hit in a reconcile loop, most notably 5 authorization failures per identifier per account per hour, and 5 duplicate certificates per exact set of names per week.

### Certificate

The `Certificate` is the only object most people write. It's desired state: these names, in this Secret, renewed on this schedule.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
  namespace: app
spec:
  secretName: app-tls
  dnsNames:
  - app.example.com
  - "*.app.example.com"
  duration: 2160h      # 90d, a request; the CA decides the actual lifetime
  renewBefore: 720h    # default is 1/3 of the actual lifetime
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-dns
```

The `Certificate` controllers decide *when* to issue: the Secret is missing, its contents don't match the spec, or it's inside the renewal window. When they decide to issue, they generate a private key into a temporary Secret labelled `cert-manager.io/next-private-key: "true"`, build a CSR from it, and create a `CertificateRequest`. The private key never leaves the cluster. Only the CSR goes to the CA.

`rotationPolicy: Always` generates a fresh key on every issuance. It became the default in cert-manager v1.18. Before that the default was `Never`, which reused the key forever, and a lot of older manifests still set it explicitly.

### CertificateRequest

One `CertificateRequest` is one attempt to turn one CSR into one signed certificate. It's issuer-agnostic: the same object shape works for ACME, Vault, a private CA, or an external issuer. Before any issuer acts on it, it has to be **approved**. By default cert-manager's built-in approver approves everything, and that hook is where policy plugins like approver-policy sit (more below).

### Order and Challenge

The ACME issuer turns an approved `CertificateRequest` into an `Order`, which maps one-to-one to an ACME order. The `Order` controller calls `newOrder`, reads the authorizations, and for every authorization that isn't already valid creates a `Challenge` with the solver it picked.

The `Challenge` controller does the real work: it **presents** the challenge (creates the TXT record), runs a **self-check** until the record is visible, tells the ACME server to validate, waits for the result, and finally **cleans up** the record. When every challenge is valid, the `Order` controller finalizes with the CSR, downloads the chain, and writes it into `CertificateRequest.status.certificate`. The `Certificate` controller then copies that and the private key into the target Secret.

Everything downstream of the `Certificate` is owned by it, so `Order`s and `Challenge`s are garbage-collected after success. If you're debugging, look while it's still broken.

---

## The DNS-01 flow, step by step

```
 Certificate ctrl   CR/ACME issuer     Order ctrl         Challenge ctrl          DNS provider       ACME CA
 ────────────────   ──────────────     ──────────         ──────────────          ────────────       ───────
  1 needs issuance
    gen key, CSR
    create CR ──────▶ 2 approved?
                       create Order ───▶ 3 newOrder ──────────────────────────────────────────────────▶
                                         ◀──────────────────────────────────── authz + dns-01 token ──
                                         create Challenge ─▶ 4 compute TXT value
                                                              find zone (SOA walk)
                                                              Present ───────────▶ create TXT
                                                            5 self-check loop:
                                                              query authoritative NS ◀─ TXT visible?
                                                            6 accept challenge ─────────────────────▶
                                                                                    ◀── validators query
                                                                                         DNS ───────
                                                              poll until valid ◀────────────────────
                                                            7 CleanUp ────────────▶ delete TXT
                                         8 finalize(CSR) ───────────────────────────────────────────────▶
                                         ◀──────────────────────────────────────────────── cert chain ──
                       status.certificate
  9 write Secret ◀──── (tls.crt, tls.key, ca.crt)
```

A few of these steps are worth slowing down for.

### Step 4: finding the zone

cert-manager knows the name it needs to write (`_acme-challenge.app.example.com`) but the provider API wants a zone. It works this out by querying for `SOA` records, starting at the full name and walking up a label at a time until it finds one. The name that owns the SOA is the zone apex. This is why delegated challenge zones "just work": if `_acme-challenge.app.example.com` is itself a delegated zone, the SOA walk lands on it and cert-manager writes there.

Some providers also need a zone ID, which cert-manager either takes from config (`hostedZoneID` for Route 53, `hostedZoneName` for Cloud DNS) or discovers by listing zones. Setting it explicitly means the credential doesn't need list permission on every zone in the account.

### Step 5: the self-check

cert-manager doesn't trust that a successful API write means the record is live. Before it tells the CA to validate, it checks for itself, because a failed validation costs a rate-limit slot and marks the authorization invalid.

By default it does what a validator would: uses recursive resolvers to find the zone's authoritative nameservers, then queries **those authoritative servers directly** for the TXT value, so resolver caches don't get in the way. Two controller flags change this:

```bash
# Use these resolvers instead of /etc/resolv.conf for the SOA/NS lookups
--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53

# Don't query authoritative servers at all, only the recursive ones above
--dns01-recursive-nameservers-only
```

You want the first when the cluster's resolver has a split-horizon view. If `example.com` also exists as a private zone inside your VPC, the in-cluster resolver answers from the private zone, finds no TXT record, and the self-check spins forever while the public record is sitting there correct. You want the second when the cluster's network can't reach arbitrary authoritative servers on port 53.

The self-check passing is necessary, not sufficient. The CA validates from several network vantage points (multi-perspective issuance corroboration is now required by CA/Browser Forum baseline requirements), through its own resolvers, so anycast nodes that haven't caught up or stale negative caches can still fail it.

### Step 6 to 7: validation and cleanup

Once the CA says the challenge is valid, the TXT record is useless and cert-manager deletes it. If cert-manager is killed between presenting and cleanup, the record stays. Stale `_acme-challenge` records are harmless for security, since the value is bound to an old token, but a provider that replaces whole RRsets can break concurrent challenges for the same name.

### Renewal

By default renewal starts when a third of the lifetime remains, so a 90-day Let's Encrypt certificate renews at day 60. Each renewal creates a fresh `CertificateRequest`, `Order`, and (unless a cached authorization is still valid) `Challenge`s. That's the main reason to be careful with DNS-01 credentials: they're not a one-time bootstrap secret, they're exercised continuously, forever. Certificate lifetimes are also getting shorter (Let's Encrypt has announced a move to 45-day certificates, and offers 6-day short-lived ones), which makes reliable unattended DNS-01 more important, not less.

---

## Solvers and selectors

An ACME issuer has a list of solvers. For each name in an order, cert-manager picks exactly one solver using its `selector`:

```yaml
solvers:
- selector:
    dnsNames: ["legacy.example.com"]        # exact names
  http01:
    ingress:
      ingressClassName: nginx
- selector:
    dnsZones: ["internal.example.com"]      # this name and everything under it
  dns01:
    route53:
      region: us-east-1
      hostedZoneID: Z0INTERNAL
- selector:
    dnsZones: ["example.com"]
    matchLabels:
      tls.example.com/solver: dns           # labels on the Certificate
  dns01:
    cloudDNS:
      project: dns-project
- dns01:                                    # NO selector: matches every name
    cloudflare:
      apiTokenSecretRef:
        name: cloudflare-token
        key: token
```

The rule is "most specific match wins". An exact `dnsNames` match beats a `dnsZones` match, a longer zone suffix beats a shorter one, and more matching labels beat fewer. When two solvers are equally specific, the earlier one in the list wins. A solver with **no selector at all matches every name** and is only used when nothing more specific matches.

That last rule is convenient for single-tenant setups and is the root of one of the guardrail bypasses below.

---

## The delegated `_acme-challenge` pattern

### The problem

You have `example.com` at Provider A. It might be the registrar's DNS, a cloud account the networking team owns, or a provider whose API tokens are scoped per zone. You want cert-manager in your clusters to complete DNS-01 for names in it. The naive option is to give cert-manager an API token for `example.com`, which means every cluster that renews certificates holds a credential that can rewrite the company's apex, MX, and every A record.

You want cert-manager to be able to write **only** `_acme-challenge.*` records, and the parent provider can't express that.

### The solution: move the challenge names into a zone you're happy to hand out

DNS lets you send any name anywhere, either with a CNAME or with an NS delegation. So you create a separate zone, at Provider B or just in a different account, that holds nothing but challenge records, and you point the challenge names at it. Then cert-manager gets a credential for that zone only.

There are two ways to wire it up.

#### Variant 1: CNAME into a shared validation zone

One-time setup at Provider A, per name you want certificates for:

```
; zone example.com @ Provider A  (humans or Terraform, rarely changes)
acme.example.com.                   3600 IN NS     ns-b1.provider-b.net.
acme.example.com.                   3600 IN NS     ns-b2.provider-b.net.
_acme-challenge.app.example.com.    3600 IN CNAME  app.acme.example.com.
_acme-challenge.api.example.com.    3600 IN CNAME  api.acme.example.com.
_acme-challenge.example.com.        3600 IN CNAME  apex.acme.example.com.

; zone acme.example.com @ Provider B  (cert-manager writes here, constantly)
app.acme.example.com.               60   IN TXT    "q8F3...current token..."
```

The CA resolves `_acme-challenge.app.example.com`, follows the CNAME (Let's Encrypt and the major CAs do), lands in `acme.example.com`, which is delegated to Provider B, and reads the TXT record there.

cert-manager has to be told to follow the CNAME too, otherwise it tries to write the TXT at the original name in `example.com` and fails on permissions:

```yaml
solvers:
- selector:
    dnsZones: ["example.com"]
  dns01:
    cnameStrategy: Follow           # resolve the CNAME chain, write at the target
    cloudDNS:
      project: acme-validation-project
      hostedZoneName: acme-example-com
```

The validation zone could be a subdomain of `example.com` as above, or a completely unrelated domain like `example-validation.net`. A separate domain means the validation zone's security doesn't depend on `example.com`'s at all, which some teams prefer.

#### Variant 2: NS-delegate each `_acme-challenge` name as its own zone

```
; zone example.com @ Provider A
_acme-challenge.app.example.com.   3600 IN NS  ns-b1.provider-b.net.
_acme-challenge.app.example.com.   3600 IN NS  ns-b2.provider-b.net.

; zone _acme-challenge.app.example.com @ Provider B
_acme-challenge.app.example.com.   60   IN TXT "q8F3..."
```

No CNAME, so no `cnameStrategy` needed. cert-manager's SOA walk finds the delegated zone on its own. The cost is one zone per name, which some providers charge for and limit.

A useful detail: an apex certificate and a wildcard certificate both validate at the same name. `example.com` and `*.example.com` both put their TXT values at `_acme-challenge.example.com`. So delegating that single name as its own zone is enough to automate both, which makes Variant 2 cheap when what you want is one wildcard for the whole domain.

#### Worked example: following Variant 2 end to end

Take a parent zone `example.com` at Provider A, and a tiny zone `_acme-challenge.example.com` at Provider B that holds only challenge records. Someone added the signpost to Provider A once, by hand or Terraform:

```
; zone example.com @ Provider A
_acme-challenge.example.com.   NS  ns-b1.provider-b.net.
_acme-challenge.example.com.   NS  ns-b2.provider-b.net.
```

cert-manager has write access to the Provider B zone only. Here's what happens when it requests `*.example.com`, step by step, with the "DNS from zero" model in mind:

1. **Write.** cert-manager calls Provider B's HTTPS API: "add TXT `q8F3...` at `_acme-challenge.example.com` in zone `_acme-challenge-example-com`." Provider B checks cert-manager's credential, saves the record and publishes it to `ns-b1`/`ns-b2`. DNS isn't involved in this step at all, and Provider A is never contacted.
2. **Read, by the CA.** The CA's resolver asks the root, which refers it to `com`. `com` refers it to Provider A, because that's what the registrar set for `example.com`.
3. Provider A doesn't hold the record. It holds the signpost, so it answers with a referral: "`_acme-challenge.example.com` is served by `ns-b1.provider-b.net`."
4. The resolver asks `ns-b1.provider-b.net`, which holds the zone and returns the TXT value authoritatively.
5. The value matches, so the CA issues the certificate. cert-manager deletes the TXT record through Provider B's API.

You can check the signpost yourself with `dig +short NS _acme-challenge.example.com`: Provider B's nameservers mean the delegation is in place. And with `dig +short NS example.com`: Provider A's nameservers mean the parent is elsewhere.

What this buys you: if cert-manager's credential leaks, the attacker can write TXT records in a zone that only holds challenge records. They can't touch `www`, `mail` or anything else in `example.com`, because those live at Provider A and need a different credential. What it doesn't buy you is covered just below: the credential can still get certificates issued.

### The resulting picture

```
                         READ PATH (anyone, via DNS)
  ACME validator ──▶ resolver ──▶ root ──▶ com ──▶ Provider A: example.com
                                                     │  _acme-challenge.app CNAME app.acme.example.com
                                                     │  acme NS provider-b
                                                     ▼
                                                   Provider B: acme.example.com
                                                        app TXT "q8F3..."

                         WRITE PATH (credentials, via provider APIs)
  ┌──────────────────────┐                    ┌──────────────────────────────┐
  │ DNS team / Terraform │ ── full admin ───▶ │ Provider A: example.com      │
  └──────────────────────┘                    │ apex, MX, A, CNAMEs, NS      │
                                              └──────────────────────────────┘
  ┌──────────────────────┐                    ┌──────────────────────────────┐
  │ cert-manager         │ ── TXT write ────▶ │ Provider B: acme.example.com │
  │ (every cluster)      │    this zone only  │ nothing but challenge TXTs   │
  └──────────────────────┘                    └──────────────────────────────┘
```

### What least privilege looks like at the provider

The whole point is that the challenge zone credential is narrow. How narrow depends on the provider:

- Zone-scoped API tokens (Cloudflare-style "DNS edit on zone X") give you exactly the challenge zone and nothing else.
- Cloud IAM with zone-level bindings (Cloud DNS supports IAM on individual managed zones) lets you grant a DNS admin role on `acme-example-com` only, with no project-wide role.
- Route 53 policies can be scoped to one hosted zone ARN, and can go further with the `route53:ChangeResourceRecordSetsNormalizedRecordNames` and `route53:ChangeResourceRecordSetsRecordTypes` condition keys to allow only `TXT` under `_acme-challenge` prefixes. That makes per-record scoping possible inside a shared zone, though the separate-zone pattern is still easier to audit.

```json
{
  "Effect": "Allow",
  "Action": "route53:ChangeResourceRecordSets",
  "Resource": "arn:aws:route53:::hostedzone/Z0ACMEVALIDATION",
  "Condition": {
    "ForAllValues:StringEquals": { "route53:ChangeResourceRecordSetsRecordTypes": ["TXT"] },
    "ForAllValues:StringLike":   { "route53:ChangeResourceRecordSetsNormalizedRecordNames": ["*.acme.example.com"] }
  }
}
```

### What the pattern does and doesn't buy you

It does take away the ability to hijack traffic. A leaked challenge-zone credential can't change `app.example.com`'s A record, can't touch MX, and can't add new names.

It doesn't stop certificate issuance for the names that are CNAMEd in. Whoever holds the credential can still get a valid certificate for `app.example.com` from any CA that follows the CNAME, because publishing TXT values is exactly what proves control. That's the inherent cost of automated DNS-01, and the defences for it are CAA with `accounturi`, certificate transparency monitoring, and controlling who can make cert-manager use the credential (the next two sections).

It also only covers names someone remembered to CNAME. A new hostname needs a one-time change in the parent zone before its first certificate, which is a feature: the DNS owners decide which names can be automated.

Two gotchas. Because a CNAME can't coexist with other data at the same name, any other tool that writes TXT records directly at `_acme-challenge.app.example.com` (an old certbot cron job, a domain verification flow) breaks once the CNAME is there. And if you remove a name, remove its CNAME too. A CNAME into a zone you've deleted is a dangling record.

---

## Ambient credentials and the namespace privilege boundary

### What "ambient" means

cert-manager's DNS providers can get credentials two ways. **Explicit** credentials are referenced in the issuer spec: a Secret holding an API token or a service account key. **Ambient** credentials come from the environment the cert-manager controller pod runs in: the cloud metadata server, workload identity federation (GKE Workload Identity, EKS IRSA or Pod Identity, Azure Workload Identity), or environment variables on the controller pod.

```yaml
# Explicit: the issuer points at a Secret it is allowed to read
dns01:
  cloudDNS:
    project: dns-project
    serviceAccountSecretRef:
      name: clouddns-key
      key: key.json

# Ambient: no credentials in the spec at all.
# cert-manager falls back to whatever identity its own pod has.
dns01:
  cloudDNS:
    project: dns-project
```

Ambient credentials are attractive because there's no long-lived key to rotate or leak. You bind the cert-manager controller's Kubernetes service account to a cloud identity and you're done.

### The flags and their defaults

```bash
--cluster-issuer-ambient-credentials=true    # default: ClusterIssuers may use ambient creds
--issuer-ambient-credentials=false           # default: namespaced Issuers may not
```

When ambient credentials are disabled for an issuer type, a solver with no explicit credentials simply fails to authenticate instead of falling back to the controller's identity.

### Why namespaced Issuers are off by default

The cert-manager controller is a single deployment that reconciles every Issuer in every namespace. Its identity is shared. If namespaced Issuers could use ambient credentials, the security of that identity would be decided by **whoever can create an Issuer in any namespace**.

```
  ┌───────────────────────────── cluster ─────────────────────────────┐
  │                                                                   │
  │  namespace: team-a (tenant can create Issuers)                    │
  │  ┌─────────────────────────────────────────┐                      │
  │  │ Issuer                                  │                      │
  │  │   dns01.cloudDNS.project: dns-project   │  no credentials      │
  │  │   (no serviceAccountSecretRef)          │  in the spec         │
  │  └────────────────────┬────────────────────┘                      │
  │                       │ reconciled by                             │
  │  namespace: cert-manager                                          │
  │  ┌────────────────────▼────────────────────┐                      │
  │  │ cert-manager controller                 │                      │
  │  │   identity: dns-writer@cloud            │── DNS write on ──▶ example.com
  │  │   (set up for the platform's            │   every zone it was granted
  │  │    ClusterIssuer)                       │                      │
  │  └─────────────────────────────────────────┘                      │
  └───────────────────────────────────────────────────────────────────┘

  With --issuer-ambient-credentials=true, team-a just borrowed the platform's
  DNS identity. Nothing in team-a's RBAC said it could.
```

This is a textbook **confused deputy**. Kubernetes RBAC says team-a can create Issuers in team-a. It says nothing about the cloud IAM role bound to cert-manager. The controller, acting on team-a's behalf, uses its own authority. With explicit credentials the boundary holds naturally, because a namespaced Issuer can only reference Secrets in its own namespace, and team-a can only put credentials there that team-a already has.

ClusterIssuers are different because only cluster administrators can create them, so "the creator of this object can use the controller's identity" is an acceptable statement. That's the whole reasoning behind the asymmetric defaults.

The practical rule: leave `--issuer-ambient-credentials=false`. If tenants need their own DNS-01 issuers, give them explicit, narrowly scoped credentials in their own namespace. Some providers also support `serviceAccountRef`, where the Issuer names a Kubernetes ServiceAccount in its own namespace, cert-manager requests a token for it, and exchanges that for cloud credentials through workload identity federation. That gives keyless credentials without the shared-identity problem, because the identity belongs to the tenant's namespace, not the controller.

Turning ambient on for ClusterIssuers isn't free either. The controller's identity now needs write access to every zone any ClusterIssuer solves for. Keep that identity's grants as narrow as the challenge-zone pattern lets you, so "cluster-wide" doesn't also mean "every zone in the company".

---

## Guardrails

The ACME CA checks that you can write the TXT record. It can't check whether the Kubernetes tenant that asked should have been allowed to. That's the cluster's job, and there are several layers.

### Layer 1: RBAC on the issuer objects

Tenants usually shouldn't be able to create `ClusterIssuer`s at all, and whether they can create `Issuer`s is a deliberate decision. The default `admin` and `edit` ClusterRoles don't include cert-manager resources, but cert-manager's Helm chart ships aggregated roles that add `issuers` and `certificates` to them. Check what your tenants actually have:

```bash
kubectl auth can-i create issuers.cert-manager.io -n team-a --as=system:serviceaccount:team-a:deployer
kubectl auth can-i create clusterissuers.cert-manager.io --as=system:serviceaccount:team-a:deployer
```

### Layer 2: admission policy on Issuer and ClusterIssuer specs

RBAC is all-or-nothing on the object. Admission policy can look at the spec. Typical rules: only allow approved ACME servers, require every solver to be scoped to zones the tenant owns, and disallow certain providers in namespaced Issuers.

Here's a `ValidatingAdmissionPolicy` (GA in Kubernetes v1.30, CEL-based, no webhook to run):

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: acme-solvers-must-be-scoped
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["cert-manager.io"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["issuers", "clusterissuers"]
  variables:
  - name: solvers
    expression: "has(object.spec.acme) && has(object.spec.acme.solvers) ? object.spec.acme.solvers : []"
  validations:
  - expression: >-
      variables.solvers.all(s,
        has(s.selector) &&
        ((has(s.selector.dnsZones) && size(s.selector.dnsZones) > 0) ||
         (has(s.selector.dnsNames) && size(s.selector.dnsNames) > 0)))
    message: "every ACME solver needs a non-empty dnsZones or dnsNames selector"
  - expression: >-
      variables.solvers.all(s,
        !has(s.selector) || !has(s.selector.dnsZones) ||
        s.selector.dnsZones.all(z, z == 'team-a.example.com' || z.endsWith('.team-a.example.com')))
    message: "dnsZones must be inside team-a.example.com"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: acme-solvers-must-be-scoped-team-a
spec:
  policyName: acme-solvers-must-be-scoped
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: team-a
```

In practice you'd parameterise the allowed zone per namespace with `paramKind` instead of hardcoding it, but the shape is the same. The first validation is the important one.

### The selector-less solver bypass

A common first draft of this policy only has the second rule: "for every solver, every `dnsZones` entry must be in the tenant's allowed list". It reads correctly and is wrong.

```yaml
solvers:
- dns01:            # no selector
    cloudDNS:
      project: dns-project
```

This solver has no `dnsZones`, so a rule that iterates over `dnsZones` has nothing to check and passes vacuously. But a solver with no selector **matches every name**, so it's the broadest possible solver, not the narrowest. The policy waved through exactly the case it existed to block. The same thing happens with a selector that has only `matchLabels`: labels live on the `Certificate`, which the tenant also writes, so a labels-only selector constrains nothing.

The general lesson: when you write policy against a matching language, find the "match all" spelling first. In cert-manager's selector language it's the empty selector, and your policy has to treat absent fields as "everything", not as "nothing to check". Hence the first validation, which requires a non-empty `dnsZones` or `dnsNames` on every solver.

`dnsNames` selectors need the same allowlist check as `dnsZones`. The example above only checks zones, so a complete policy adds a third rule for names.

### Layer 3: who can use a ClusterIssuer

Even with every Issuer locked down, there's a bigger gap. Any `Certificate` in any namespace can reference any `ClusterIssuer` by name. If the platform ClusterIssuer can solve for all of `example.com`, then team-a can create:

```yaml
kind: Certificate
metadata:
  namespace: team-a
spec:
  secretName: stolen
  dnsNames: ["payments.example.com"]     # owned by a different team
  issuerRef: {kind: ClusterIssuer, name: letsencrypt-dns}
```

cert-manager will solve the challenge with the platform's credentials and write a valid certificate and key for another team's hostname into team-a's namespace. The CA is satisfied because the TXT record really was published.

Two ways to close this:

1. **Admission policy on `Certificate`** (and on `Ingress`/`Gateway` resources that carry cert-manager's ingress-shim annotations, since those create Certificates for you). Check that every `dnsNames` entry is inside the namespace's allowed domains whenever `issuerRef` is a shared ClusterIssuer.
2. **cert-manager's approver-policy**. Disable the built-in approver (`--controllers=*,-certificaterequests-approver`), install approver-policy, and write `CertificateRequestPolicy` objects that say which namespaces may request which names from which issuers. This works at the `CertificateRequest` level, so it catches every path that creates one, including the CSI driver and direct `CertificateRequest` creation.

```yaml
apiVersion: policy.cert-manager.io/v1alpha1
kind: CertificateRequestPolicy
metadata:
  name: team-a-dns
spec:
  allowed:
    dnsNames:
      values: ["team-a.example.com", "*.team-a.example.com"]
  selector:
    issuerRef:
      kind: ClusterIssuer
      name: letsencrypt-dns
    namespace:
      matchNames: ["team-a"]
```

approver-policy also needs RBAC binding the policy to the requester (the `use` verb on the policy), so a policy existing isn't enough on its own.

### Layer 4: outside the cluster

CAA with `accounturi` and `validationmethods` limits which ACME account can issue at all. Certificate Transparency monitoring (searching CT logs for your domains) tells you when something was issued that you didn't expect. Neither prevents a misbehaving cluster from issuing through the legitimate account, but they catch everything else.

### Putting the layers together

```
  Request for a cert for payments.example.com from namespace team-a
     │
     ├─ RBAC: can team-a create Issuers / Certificates?          (who may ask)
     ├─ Admission on Issuer: are solvers scoped, no empty         (what tenant issuers may do)
     │   selectors, approved servers?
     ├─ Admission on Certificate / approver-policy:               (which names each namespace
     │   is payments.example.com allowed for team-a?               may request)
     ├─ Ambient creds off for Issuers                             (tenants can't borrow the
     │                                                             controller's identity)
     ├─ Challenge-zone credential only writes acme.example.com    (blast radius of the
     │                                                             credential itself)
     └─ CAA accounturi + CT monitoring                            (CA-side and detection)
```

---

## Debugging DNS-01

Follow the chain from the top. The failing object's events and status usually say exactly what's wrong.

```bash
# The whole chain for one namespace
kubectl get certificate,certificaterequest,order,challenge -n app

# The challenge is where DNS-01 problems surface
kubectl describe challenge -n app <name>
#   Status.Presented: true/false   Status.Reason: "Waiting for DNS-01 challenge propagation: ..."

# cmctl summarises the chain for one Certificate
cmctl status certificate app-tls -n app
cmctl renew app-tls -n app        # force a re-issue

# What the CA will see: do the walk yourself
dig +trace _acme-challenge.app.example.com TXT

# What the authoritative servers say, bypassing caches
dig NS acme.example.com +short
dig @ns-b1.provider-b.net app.acme.example.com TXT +norecurse

# What a split-horizon in-cluster resolver says (often the culprit)
kubectl run -it --rm dnscheck --image=busybox:1.36 --restart=Never -- \
  nslookup -type=TXT _acme-challenge.app.example.com
```

| Symptom | Likely cause | Fix |
|---|---|---|
| `Presented: false`, 403 from provider | Credential can't write that zone, or wrote at the CNAME source | Check IAM scope; set `cnameStrategy: Follow` |
| `Waiting for DNS-01 challenge propagation` forever | Self-check queries a private/split-horizon zone, or a resolver cached `NXDOMAIN` | `--dns01-recursive-nameservers` pointing at public resolvers; lower SOA minimum |
| `no such host` / can't find zone | SOA walk fails, often because the in-cluster resolver can't see the public zone | Same as above; or set `hostedZoneName` / `hostedZoneID` |
| CA says `Incorrect TXT record` | Wildcard + apex in one order and the provider replaced the RRset; or stale values | Check RRset has both values; avoid concurrent orders for the same name |
| CA says `NXDOMAIN` but `dig` works | Dangling or wrong NS delegation, or not yet propagated to all anycast nodes | `dig +trace`; compare every authoritative server |
| Works for ClusterIssuer, fails for Issuer with "no credentials" | Ambient credentials are off for namespaced Issuers | Give the Issuer explicit credentials (by design) |
| `urn:ietf:params:acme:error:rateLimited` | Retry loop against production | Switch to staging while debugging; wait out the window |
| `CAA record prevents issuance` | CAA restricts CA, account, or method | Add the right `issue`/`issuewild` entry |

---

## Interview Prep

### Q: Why would you choose DNS-01 over HTTP-01, and what does it cost you?

**A:** DNS-01 is the only ACME challenge that issues wildcards, and it doesn't need the CA to reach your service, so it works for internal services, multi-cluster setups where several load balancers serve one name, and certificates you want before the load balancer exists. HTTP-01 needs port 80 reachable on every endpoint that serves the name, and anything in front (CDN, WAF, HTTPS redirect, a second LB) can eat the request.

The cost is that the client needs DNS write credentials, and those are used on every renewal, so they live in the cluster permanently. DNS write access is powerful: it can redirect traffic and intercept mail, not just issue certificates. Most of the design work in a DNS-01 setup goes into narrowing that credential, usually with a delegated challenge zone, and controlling who in the cluster can make cert-manager use it.

### Q: Walk me through what happens after you `kubectl apply` a Certificate with a DNS-01 ClusterIssuer.

**A:**

```
Certificate ─▶ CertificateRequest ─▶ Order ─▶ Challenge(s) ─▶ TXT ─▶ CA validates ─▶ finalize ─▶ Secret
```

1. The Certificate controller sees no Secret, generates a private key into a temporary Secret, builds a CSR, and creates a CertificateRequest.
2. The request gets approved (by the built-in approver or a policy plugin).
3. The ACME issuer creates an Order, which calls `newOrder` and gets one authorization per name, each offering a dns-01 challenge with a token.
4. For each unfinished authorization, a Challenge is created. Its controller computes `base64url(SHA-256(token + "." + accountKeyThumbprint))`, finds the zone with an SOA walk (following CNAMEs if `cnameStrategy: Follow`), and creates the TXT record through the provider API.
5. It self-checks by querying the zone's authoritative nameservers until the value is visible, then tells the CA to validate.
6. The CA queries from multiple vantage points, marks the authorization valid, and cert-manager deletes the record.
7. The Order finalizes with the CSR, downloads the chain into the CertificateRequest status, and the Certificate controller writes `tls.crt`, `tls.key`, and `ca.crt` into the target Secret.

Renewal repeats this at two thirds of the lifetime by default.

### Q: Explain authoritative nameservers vs recursive resolvers, and how a resolver finds the right authoritative server.

**A:** Authoritative servers hold zone data and answer only for their zones, with the AA bit set. Recursive resolvers hold nothing; they answer clients by walking the tree and caching results. A cold resolver starts from root hints, the list of 13 root server identities, each anycast from many sites. It asks a root server, gets a referral to the TLD servers (from the NS records the root zone holds for `com`), asks those, gets a referral to the zone's nameservers (from the NS records `com` holds for `example.com`, with glue if the nameservers are in-zone), and asks those for the answer. Each hop is cached for its TTL, and negative answers are cached for the SOA minimum (RFC 2308).

The ACME angle: cert-manager's self-check goes straight to the authoritative servers to avoid caches, but the CA goes through its own recursive resolvers, so a stale negative cache or a slow anycast node can fail validation even after the self-check passed.

### Q: How does delegation work, and how would you let cert-manager complete DNS-01 without giving it write access to your main zone?

**A:** Delegation is NS records in the parent zone at the child's name. The parent's servers return a referral, and the child's servers become the source of truth. Nothing syncs between them.

To keep cert-manager away from the main zone, create a separate zone that only holds challenge records, `acme.example.com` at another provider or account, delegate it with NS records in `example.com`, and CNAME each `_acme-challenge.<name>` into it:

```
_acme-challenge.app.example.com.  CNAME  app.acme.example.com.
acme.example.com.                 NS     ns-b1.provider-b.net.
```

Set `cnameStrategy: Follow` on the solver and give cert-manager a credential scoped to `acme.example.com` only. The CA follows the CNAME and reads the TXT from the challenge zone. The alternative is NS-delegating each `_acme-challenge.<name>` as its own zone, which needs no CNAME following but costs a zone per name.

The point is the split between write path and read path. Reads go through DNS and follow CNAMEs and delegations freely. Writes go through the provider API, and the provider's IAM decides scope. Moving the challenge names into their own zone lets you use zone-level IAM to express "can only write challenge records".

### Q: What does that pattern not protect against?

**A:** It doesn't prevent issuance. Anyone holding the challenge-zone credential can still get a valid certificate for every name CNAMEd into it, because publishing the TXT value is the proof of control. What it removes is the ability to change A records, MX, or the delegation itself, and to add names that weren't CNAMEd in. For issuance, the controls are CAA with `accounturi` so only your ACME account can issue, CT log monitoring for detection, and in-cluster policy over who can make cert-manager request which names.

### Q: What are ambient credentials in cert-manager, and why are they disabled by default for namespaced Issuers?

**A:** Ambient credentials are the cert-manager controller's own identity: cloud metadata, workload identity, or environment variables. A solver with no explicit credential falls back to them.

`--cluster-issuer-ambient-credentials` defaults to true and `--issuer-ambient-credentials` defaults to false because of who creates each object. One controller reconciles Issuers in every namespace. If a namespaced Issuer could use ambient credentials, anyone allowed to create an Issuer in any namespace could make the controller act with its own DNS identity, one usually provisioned for a platform ClusterIssuer with access to production zones. That's a confused deputy: the tenant's RBAC never granted that access, but the controller would exercise it for them. With explicit credentials the boundary holds, since a namespaced Issuer can only reference Secrets in its own namespace. ClusterIssuers can only be created by cluster admins, so letting them use the controller's identity doesn't cross a boundary.

If tenants need keyless credentials, use `serviceAccountRef` where the provider supports it. The identity then belongs to a service account in the tenant's namespace, not to the controller.

### Q: You wrote an admission policy that checks every solver's `dnsZones` is in the tenant's allowlist. What's wrong with it?

**A:** A solver with no selector, or with only `matchLabels`, has no `dnsZones` to check, so the rule passes vacuously. But an empty selector matches every name, which makes it the broadest solver possible. `matchLabels` doesn't help either, because the labels are on the Certificate, which the tenant also controls. The policy has to require a non-empty `dnsZones` or `dnsNames` selector on every solver first, then allowlist both fields.

More generally, when you write policy against a matching language, find how that language spells "match everything" and make sure the policy treats absent fields that way.

### Q: Everything about Issuers is locked down. Can a tenant still get a certificate for another team's hostname?

**A:** Yes, through a shared ClusterIssuer. Any Certificate in any namespace can reference any ClusterIssuer. If the platform ClusterIssuer can solve for all of `example.com`, team-a can request `payments.example.com`, cert-manager solves it with the platform credentials, and the cert and key land in team-a's namespace. The CA is fine with it because the TXT record really was published.

To close it, either validate `dnsNames` against per-namespace allowed domains in admission on Certificates (and on Ingress or Gateway objects carrying cert-manager annotations), or disable the built-in approver and use approver-policy, which checks every CertificateRequest regardless of how it was created. Splitting into per-tenant ClusterIssuers with per-zone credentials also limits the blast radius, but it still needs one of those controls, since any namespace can reference any ClusterIssuer.

### Q: The Challenge is stuck on "Waiting for DNS-01 challenge propagation" but `dig` from your laptop shows the TXT record. What's going on?

**A:** The self-check runs from inside the cluster, and the cluster sees a different DNS than your laptop. The usual causes:

1. Split horizon. A private zone for `example.com` exists in the VPC, so the in-cluster resolver answers from it and never sees the public TXT. Fix with `--dns01-recursive-nameservers` pointing at public resolvers.
2. Egress restrictions. The cluster can't reach arbitrary authoritative servers on port 53, so the direct authoritative query times out. Fix with `--dns01-recursive-nameservers-only` and a reachable resolver.
3. Negative caching. Something queried the name before it existed, and a resolver is holding an `NXDOMAIN` for the SOA minimum TTL. Wait or lower the TTL.
4. Wrong zone. The record went into a zone that isn't actually delegated (the SOA walk found something your laptop's `dig` doesn't use). Compare `dig +trace` with what cert-manager logged.

Run `nslookup` from a pod in the cluster to see what cert-manager sees.

### Q: Why does an order for `example.com` and `*.example.com` sometimes fail when each one alone works?

**A:** Both are validated at `_acme-challenge.example.com`, with different tokens, at the same time. The TXT RRset needs both values at once. If the provider integration replaces the RRset instead of appending, the second Present overwrites the first and one validation sees the wrong value. cert-manager's built-in providers handle this, but webhook solvers and custom scripts are where it shows up.

---

## Related Notes

- [[notes/K8s/ingress-vs-gateway-api|Ingress vs Gateway API]] for where cert-manager fits next to ManagedCertificate and cloud certificate managers
- [[notes/K8s/gke-gateway-iap-and-certificate-manager|GKE Gateway with IAP and Certificate Manager]] for the cloud-native alternative with DNS authorization via CNAME
- [[notes/K8s/kubebuilder-controllers-and-webhooks|Kubebuilder Controllers, Webhooks & Extension APIs]] for how reconcile loops and admission webhooks work, and cert-manager-issued webhook certs
- [[notes/K8s/kubernetes-services-dns-and-network-policies|Services, DNS & Network Policies]] for in-cluster DNS, CoreDNS, and why the cluster's view of DNS differs from the internet's
- [[notes/Networking/dns-privacy-and-oblivious-protocols|DNS Privacy & Oblivious Protocols]] for the resolver side of DNS and encrypted transports
- [[notes/Networking/tls-1.3-handshake|TLS 1.3 Handshake Deep Dive]] for what the issued certificate is actually used for
- [[notes/AuthNZ/oauth-oidc-and-workload-identity|OAuth, OIDC & Workload Identity Federation]] for how keyless ambient credentials work underneath
