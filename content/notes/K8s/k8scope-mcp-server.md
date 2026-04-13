---
title: "k8scope: Building a Kubernetes MCP Server with OAuth 2.0 and StreamableHTTP"
---

What happens when you give an LLM direct, read-only access to a Kubernetes cluster? Not through a shell that runs `kubectl` commands -- through a purpose-built server that speaks the Model Context Protocol (MCP) and hands back structured data. That is k8scope: a Go server that bridges Claude Code (or any MCP-capable client) to GKE clusters, with OAuth 2.0 authentication that proxies the user's Google identity to the Kubernetes API.

This post is a complete architectural walkthrough of the rearchitected version. The original design used SSE transport and accepted raw tokens per-request. The current system uses StreamableHTTP transport, implements a full OAuth 2.0 authorization server with Dynamic Client Registration (RFC 7591), PKCE (RFC 7636), and server-side session management. Everything changed except the core idea: give LLMs structured, read-only Kubernetes access.

---

## Why Not Just Shell Out to kubectl?

The obvious approach to giving an LLM Kubernetes access is to let it run `kubectl` commands. This works, but it has problems:

1. **Unstructured output.** `kubectl get pods` returns a human-formatted table. The LLM has to parse whitespace-aligned columns. Errors in parsing are silent.
2. **Safety.** If the LLM can run `kubectl`, it can also run `kubectl delete`. You need a separate guardrail layer.
3. **Auth coupling.** kubectl reads `~/.kube/config`, which ties you to a single machine's credential setup.
4. **No schema.** The LLM has no formal description of what parameters each operation accepts.

k8scope solves all four: it returns structured text, it only exposes read operations, it takes auth via OAuth 2.0 (user logs in with Google, server proxies their identity), and it advertises a JSON Schema for every tool.

---

## Architecture Overview

### Request Flow

Every request traverses this middleware chain:

```
HTTP request
  |
  v
recoverMiddleware   (catches panics, returns 500)
  |
  v
corsMiddleware      (rejects OPTIONS, sets nosniff)
  |
  v
http.ServeMux       (routes by path)
  |
  +-- /.well-known/oauth-authorization-server  -->  handleMetadata (RFC 8414)
  +-- POST /register                           -->  handleRegister (RFC 7591) [rate limited]
  +-- GET  /authorize                          -->  handleAuthorize            [rate limited]
  +-- GET  /callback                           -->  handleCallback             [rate limited]
  +-- POST /token                              -->  handleToken                [rate limited]
  +-- GET  /health                             -->  health check
  +-- /mcp, /mcp/                              -->  oauth.Middleware --> StreamableHTTPServer
                                                                           |
                                                                           v
                                                                       MCPServer
                                                                           |
                                                                           v
                                                                     Tool Handler
```

### MCP Protocol Layer -- Two Objects That Look Similar

The MCP library (`mcp-go v0.47`) provides two objects that are easy to confuse:

- **`server.MCPServer`** -- the protocol engine. It understands MCP messages (`initialize`, `tools/list`, `tools/call`), holds the tool registry, and dispatches to handlers. It knows nothing about HTTP.
- **`server.StreamableHTTPServer`** -- the HTTP adapter. It implements `http.Handler`, translates HTTP request bodies into JSON-RPC messages, passes them to `MCPServer`, and writes responses back. It also handles SSE streaming for long-lived connections.

MCP is NOT REST. There is a single endpoint (`/mcp`) that speaks JSON-RPC 2.0. The tool name lives in the JSON body, not in the URL path. This is why there is no `/api/pods` or `/api/events` -- everything goes through `/mcp`.

```
Client                              Server (:8080)
  |                                     |
  |--- POST /mcp (initialize) -------->|
  |<-- {protocolVersion: "2025-03-26"}  |
  |                                     |
  |--- POST /mcp (tools/list) -------->|
  |<-- {tools: [...6 tools]}            |
  |                                     |
  |--- POST /mcp (tools/call) -------->|
  |<-- {content: [{text: "..."}]}       |
```

StreamableHTTP is the successor to the older SSE transport. In the SSE model, clients first established a long-lived `GET /sse` connection to discover the message endpoint. StreamableHTTP eliminates that -- clients POST directly to `/mcp`. The server can still use SSE for streaming responses (which is why `WriteTimeout` is omitted on the HTTP server), but the initial handshake is simpler.

---

## Authentication -- The Most Interesting Part

This is where k8scope departs from every Kubernetes tool you have used before. Instead of reading kubeconfig or accepting raw tokens, it implements a full OAuth 2.0 authorization server that proxies Google authentication.

### The Mental Model: Two Different OAuth Client IDs

There are two completely separate `client_id` values in this system, and confusing them is the fastest way to misunderstand the architecture:

```
Claude Code ──(MCP client_id)──> k8scope Server ──(Google client_id)──> Google
```

- **MCP `client_id`** -- identifies the application (Claude Code, Cursor, etc.). Obtained via Dynamic Client Registration (`POST /register`). Used at `/authorize` and `/token`. Each installation registers independently.
- **Google `client_id`** -- `GOOGLE_CLIENT_ID` env var. Identifies our server to Google. Used internally when redirecting to Google login. Claude Code never sees this.

A `client_id` identifies the APPLICATION, not the user. Multiple users share the same `client_id`. User identity comes from the Google login that produces a session.

### The Full Auth Flow

```
0. Discovery + Registration (one-time per client installation)

   Claude Code ─── GET /.well-known/oauth-authorization-server ──> k8scope
                <── { authorization_endpoint, token_endpoint,
                      registration_endpoint, ... }

   Claude Code ─── POST /register ──────────────────────────────> k8scope
                   { client_name: "Claude Code",
                     redirect_uris: ["http://127.0.0.1:0/callback"],
                     token_endpoint_auth_method: "none" }
                <── { client_id: "abc123...", client_id_issued_at: ... }

1. Authorization (user-facing, in browser)

   Claude Code opens browser ──> GET /authorize?client_id=abc123
                                    &redirect_uri=http://127.0.0.1:54321/callback
                                    &code_challenge=<SHA256(verifier)>
                                    &code_challenge_method=S256
                                    &state=<csrf_token>

   k8scope validates:
     - client_id exists in registry
     - redirect_uri matches registered URIs (port-agnostic per RFC 8252 s7.3)
     - code_challenge_method is S256
   Stores PendingAuth { client_id, code_challenge, redirect_uri, state }
   Redirects browser ──> Google login (with pendingKey as Google's state param)

2. Google callback (server-to-server)

   Google redirects browser ──> GET /callback?code=<google_auth_code>&state=<pendingKey>

   k8scope:
     - Looks up PendingAuth by pendingKey (one-time use, 5 min TTL)
     - Exchanges google_auth_code for Google access + refresh tokens (server-to-server)
     - Creates Session { email, access_token, refresh_token, expires_at } (24h TTL)
     - Generates internal auth code linked to client_id + session
     - Redirects browser ──> Claude Code's loopback listener with auth code + state

3. Token exchange (server-to-server)

   Claude Code ─── POST /token ─────────────────────────────────> k8scope
                   { grant_type: "authorization_code",
                     client_id: "abc123",
                     code: "<internal_auth_code>",
                     code_verifier: "<original_random_string>" }

   k8scope validates:
     - client_id matches the one from /authorize step
     - client_secret if confidential client
     - PKCE: SHA256(code_verifier) == stored code_challenge
   Returns: { access_token: "<session_id>", token_type: "bearer", expires_in: 86400 }

4. Every subsequent MCP request

   Claude Code ─── POST /mcp ───────────────────────────────────> k8scope
                   Authorization: Bearer <session_id>
                   { "jsonrpc": "2.0", "method": "tools/call",
                     "params": { "name": "list_pods", "arguments": {...} } }

   Middleware:
     - Extracts session_id from Bearer token
     - Looks up Session (checks 24h TTL)
     - Refreshes Google access token if near expiry (singleflight-deduplicated)
     - Injects Session into request context
   Tool handler:
     - Gets Session from context
     - Uses Session.AccessToken to call GKE/K8s APIs as the user
```

### Why OAuth Proxy Instead of Raw Tokens?

The original k8scope design accepted `api_server` and `token` as parameters on every tool call. This was stateless and simple, but it had problems:

1. **Token exposure.** The raw Google access token traveled through the LLM's context window. If the conversation was logged, the token was logged.
2. **No refresh.** GKE tokens expire after ~1 hour. The user had to manually run `gcloud auth print-access-token` and paste a new one.
3. **No multi-user.** Any client could impersonate any user by providing their token.

The OAuth proxy model solves all three. The user's Google tokens live server-side only. The client gets an opaque session ID (64-char hex, not a JWT, not reversible). Token refresh happens automatically via singleflight-deduplicated calls to Google's refresh endpoint.

### Dynamic Client Registration (RFC 7591)

MCP clients do not come pre-registered. When Claude Code first connects, it discovers the `registration_endpoint` from the metadata document and calls `POST /register`:

```json
{
  "client_name": "Claude Code",
  "redirect_uris": ["http://127.0.0.1:0/callback"],
  "grant_types": ["authorization_code"],
  "token_endpoint_auth_method": "none"
}
```

The server enforces a security policy: **all redirect URIs must be loopback** (`127.0.0.1`, `::1`, or `localhost`). This prevents open redirect attacks where an attacker registers `https://evil.com` as a redirect URI.

Two client types are supported:

| Type | `token_endpoint_auth_method` | Secret? | Use Case |
|------|------------------------------|---------|----------|
| Public | `"none"` (default) | No | CLI tools (Claude Code, Cursor) -- cannot safely store secrets |
| Confidential | `"client_secret_post"` | Yes, server-generated | Server-side apps |

The `client_id` is reused across logins. Claude Code registers once per installation, then uses the same `client_id` for all subsequent authentications. If the server restarts (in-memory store), clients re-register.

### Loopback Redirect URI Matching (RFC 8252 Section 7.3)

Claude Code opens a temporary HTTP listener on a random port to receive the auth callback. This means the redirect URI changes every login: `http://127.0.0.1:54321/callback`, then `http://127.0.0.1:62018/callback`, etc.

Per RFC 8252 Section 7.3, loopback redirect URIs are matched by **scheme + host + path only**, ignoring port. The `matchesRegisteredURI` helper implements this:

```go
// Compares scheme + host + path. Ignores port for loopback URIs.
func matchesRegisteredURI(requestURI string, registeredURIs []string) bool {
    // ...
}
```

### PKCE (RFC 7636) -- Why It Matters for CLI Auth

PKCE (Proof Key for Code Exchange) prevents authorization code interception. The flow:

1. Claude Code generates a random `code_verifier` (high-entropy string)
2. Computes `code_challenge = BASE64URL(SHA256(code_verifier))`
3. Sends `code_challenge` to `/authorize`
4. Server stores the challenge
5. At `/token`, Claude Code sends the original `code_verifier`
6. Server computes `SHA256(verifier)` and compares to stored challenge

Only the client that generated the verifier can complete the exchange. An attacker who intercepts the auth code cannot use it without the verifier. The server enforces `S256` -- plain challenges (no hash) are rejected.

### Session Management -- The In-Memory Store

All auth state lives in Go maps protected by per-map `sync.RWMutex`:

```go
type Store struct {
    sessions map[string]*Session     // session ID -> Google tokens       (24h TTL)
    pending  map[string]*PendingAuth // pending key -> in-flight OAuth    (5 min TTL)
    codes    map[string]*AuthCode    // auth code -> session ID mapping   (5 min TTL)
    clients  map[string]*Client      // client_id -> registered client    (no TTL)
}
```

Design decisions:
- **Separate mutexes per map** -- looking up a session does not block creating an auth code or registering a client.
- **TTL enforcement** -- checked on read (lazy) + background goroutine sweeps expired entries every 10 minutes (eager).
- **One-time use** -- `PendingAuth` and `AuthCode` are deleted on read. Replay is impossible.
- **`ClientID` flows through the chain** -- stored in `PendingAuth`, copied to `AuthCode`, validated at `/token`. Ensures the client that started the flow finishes it.
- **Session IDs are opaque** -- 64-char hex from `crypto/rand`. Not a JWT, not derived from tokens, cannot be reversed.

### Token Refresh with Singleflight

When a Google access token approaches expiry (within 5 minutes), the middleware refreshes it:

```go
func (g *GoogleOAuth) EnsureFreshToken(sessionID string, session *Session) error {
    if time.Now().Before(session.ExpiresAt.Add(-5 * time.Minute)) {
        return nil // still valid
    }

    _, err, _ := g.refreshGroup.Do(sessionID, func() (interface{}, error) {
        // Re-check inside singleflight -- another goroutine may have refreshed already
        if time.Now().Before(session.ExpiresAt.Add(-5 * time.Minute)) {
            return nil, nil
        }
        // ... call Google's token refresh endpoint ...
    })
    return err
}
```

`singleflight.Group` keyed by session ID ensures that when 5 concurrent MCP requests arrive for the same user whose token is expiring, only ONE goroutine calls Google. The other 4 wait and get the same result. Once the call completes, the key is forgotten -- future calls execute fresh.

---

## The Kubernetes Client Layer

### How Cluster Selection Works

There is no pre-configured cluster. Every tool call includes `project`, `location`, and `cluster` as parameters. The server calls the GKE API to get the cluster's endpoint + CA certificate, builds a `kubernetes.Clientset` using the user's Google token, and makes the K8s API call.

```go
func NewClientForUser(ctx context.Context, accessToken string, cluster ClusterInfo) (*kubernetes.Clientset, error) {
    endpoint, ca, err := getCachedClusterDetails(ctx, accessToken, cluster)

    config := &rest.Config{
        Host:        "https://" + endpoint,
        BearerToken: accessToken,       // user's Google OAuth token
        TLSClientConfig: rest.TLSClientConfig{
            CAData: ca,                 // cluster's CA cert
        },
    }
    return kubernetes.NewForConfig(config)
}
```

This is a "bare" `rest.Config` -- no token rotation, no exec plugins, no auth provider plugins. The token is baked in as a static string. This works because token refresh happens at the OAuth middleware layer, not at the K8s client layer.

### Cluster Details Caching (10-Minute TTL)

The expensive operation is calling the GKE API to resolve a cluster name into an endpoint IP and CA certificate. This is cached for 10 minutes, keyed by `project/location/cluster`:

```go
var (
    clusterCacheMu  sync.RWMutex
    clusterCache    = make(map[string]*cachedCluster)
    clusterCacheTTL = 10 * time.Minute
)
```

The cache uses a read lock for lookup and a write lock for update -- multiple goroutines can check the cache simultaneously, but only one can write. This maximizes throughput on cache hits.

### K8s Client Connection Pooling (The Non-Obvious Part)

`kubernetes.NewForConfig` is called per-request, but this is cheap -- it just creates wrapper structs. The actual HTTP/2 connections and TLS state are pooled by client-go's internal `tlsTransportCache` (`k8s.io/client-go/transport/cache.go`).

The critical insight: **`BearerToken` is NOT part of the transport cache key.** Different tokens targeting the same cluster (same CA cert, same server name) share the same connection pool and TLS state. This means a multi-user server does not open a new TLS connection per user.

```
Transport cache key = (CA cert, client cert, server name, ...)
                      NOT (bearer token)

User A's token ─┐
                ├──> same http.Transport, same TCP connections
User B's token ─┘
```

---

## The 6 Tools

| Tool | Scope | Key Detail |
|------|-------|------------|
| `list_clusters` | GKE API (not K8s) | Lists all clusters in a GCP project. Uses `container.Service` directly, not `kubernetes.Clientset` |
| `list_pods` | Namespace or cluster-wide | Returns formatted table: name, status, reason, restarts, age. **500 pod limit** with truncation indicator |
| `describe_pod` | Single pod | Detailed view: conditions, container statuses (waiting/running/terminated), resource requests/limits |
| `get_pod_logs` | Single pod/container | Tail-based (default 100 lines). **60s timeout** (vs 30s for others). **1 MB cap** via `io.LimitReader` |
| `get_events` | Namespace or cluster-wide | **200 event limit**, sorted by `LastTimestamp` (most recent first), displays top 50. Uses `sort.Slice` |
| `get_nodes` | Cluster-wide | Status, kubelet version, CPU/memory capacity, topology zone. **500 node limit** |

### Tool Design Philosophy

Tools return structured text summaries, not raw Kubernetes API objects. A raw Pod object is enormous -- dozens of fields, nested specs, status conditions, volume mounts, tolerations. Sending all of that to an LLM wastes context window and makes it harder for the model to find what matters.

Each handler shares the same pattern:

```go
func handleListPods(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)  // deadline
    defer cancel()

    session, err := auth.SessionFromContext(ctx)              // get user's Google token
    if err != nil { return errResult("auth: %v", err) }

    ci, err := clusterInfo(req.GetArguments())                // validate + extract cluster params
    if err != nil { return errResult("validation: %v", err) }

    client, err := k8sClient.NewClientForUser(ctx, session.AccessToken, ci)  // cached cluster details
    if err != nil { return errResult("auth/connect: %v", err) }

    // ... K8s API call with pagination limits ...
    // ... format as text table ...
    return mcp.NewToolResultText(sb.String()), nil
}
```

### Input Validation

All GCP identifiers are validated with compiled regexes before reaching any API:

```go
var (
    projectRe  = regexp.MustCompile(`^[a-z][a-z0-9-]{4,28}[a-z0-9]$`)
    locationRe = regexp.MustCompile(`^[a-z]+-[a-z]+\d+(-[a-z])?$`)
    nameRe     = regexp.MustCompile(`^[a-z][a-z0-9-]{0,38}[a-z0-9]$`)
)
```

These are compiled once at startup (`MustCompile` panics on invalid regex -- caught at startup, not runtime). They prevent injection-style attacks and catch typos before they result in confusing GKE API errors.

---

## HTTP Server Configuration

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           recoverMiddleware(corsMiddleware(mux)),
    ReadHeaderTimeout: 10 * time.Second,  // slowloris protection
    IdleTimeout:       120 * time.Second, // keep-alive cleanup
    // WriteTimeout intentionally omitted — MCP uses SSE streams
}
```

### Why WriteTimeout Is Omitted

MCP's StreamableHTTP transport uses SSE (Server-Sent Events) for streaming responses. SSE connections stay open indefinitely. A `WriteTimeout` would kill these connections after the deadline, breaking streaming tool responses. The tradeoff: a misbehaving client could hold a connection open forever, but the `IdleTimeout` (120s) handles idle connections.

### Middleware Nesting Order

The `Handler` field defines execution order through nesting:

```
recoverMiddleware (outermost -- catches panics from everything below)
  └── corsMiddleware (rejects OPTIONS, sets nosniff)
        └── mux (routes by URL path)
              └── route-specific handler
                    └── oauth.Middleware (for /mcp only -- validates Bearer token)
                          └── StreamableHTTPServer (parses JSON-RPC)
                                └── MCPServer (dispatches to tool handler)
```

### Graceful Shutdown

```
SIGTERM/SIGINT
  |
  v
signal.Notify catches it
  |
  v
srv.Shutdown(15s context)
  |-- stops accepting new connections
  |-- waits for in-flight requests (including SSE streams) to complete
  |-- exits cleanly after all done or 15s deadline
  |
  v
stopCleanup()       // kills session cleanup goroutine
stopRateLimiter()   // kills rate limiter cleanup goroutine
```

`ListenAndServe` runs in a goroutine; the main thread blocks on the signal channel. When `Shutdown` is called, `ListenAndServe` returns `http.ErrServerClosed` -- this is expected, not an error.

---

## Security Hardening

The server implements 10 security fixes and 13 bug fixes. The important ones:

### Redirect URI Validation
Only loopback URIs allowed (`127.0.0.1`, `::1`, `localhost`). Prevents open redirect attacks.

### Rate Limiting (Token Bucket)
10 requests/minute per IP across all auth endpoints (`/authorize`, `/callback`, `/token`, `/register`). Token bucket algorithm: burst of 5, refill 1 every 6 seconds. Per-IP limiters with background cleanup of stale entries.

```go
limiter := rate.NewLimiter(rate.Every(6*time.Second), 5)
```

### CORS Rejection
OPTIONS preflight requests are rejected with 403. The server is not designed to be called from browser JavaScript. `X-Content-Type-Options: nosniff` prevents MIME sniffing.

### Panic Recovery
`recoverMiddleware` catches panics in any handler and returns a 500 instead of crashing the entire process. This is critical for a long-running server.

### fetchEmail Security
Uses `Authorization: Bearer` header instead of query parameter to avoid leaking tokens in access logs.

### Scheme Detection Behind Proxy
When behind a TLS-terminating load balancer, `r.TLS` is always nil. The server checks `X-Forwarded-Proto` header, but only when `TRUST_PROXY=true` -- prevents an attacker from spoofing the header.

---

## Deployment

### Dockerfile (Multi-Stage Build)

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /k8scope ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /k8scope /k8scope
EXPOSE 8080
ENTRYPOINT ["/k8scope"]
```

Key decisions:
- **`CGO_ENABLED=0`** -- static binary, no glibc dependency. Required for distroless.
- **`distroless/static-debian12:nonroot`** -- no shell, no package manager, runs as non-root. Minimal attack surface.
- **Separate `go mod download`** layer -- Docker caches dependencies, rebuilds only when go.mod/go.sum change.

### Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `GOOGLE_CLIENT_ID` | Yes | Google OAuth client ID (from GCP Console) |
| `GOOGLE_CLIENT_SECRET` | Yes | Google OAuth client secret |
| `REDIRECT_URL` | Yes | Server's `/callback` URL (registered in Google Console) |
| `PORT` | No (default: 8080) | Listen port |
| `TRUST_PROXY` | No (default: false) | Trust `X-Forwarded-Proto` header |

### MCP Client Configuration (Claude Code)

```json
{
  "mcpServers": {
    "k8scope": {
      "type": "streamable-http",
      "url": "https://k8scope.example.com/mcp"
    }
  }
}
```

Claude Code discovers the OAuth endpoints via `/.well-known/oauth-authorization-server`, registers via `POST /register`, and handles the OAuth flow automatically. The user sees a browser popup for Google login.

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `mcp-go` | v0.47.0 | MCP protocol library (JSON-RPC + StreamableHTTP transport) |
| `golang.org/x/oauth2` | v0.16.0 | Google OAuth 2.0 flow + token refresh |
| `google.golang.org/api/container/v1` | v0.126.0 | GKE API for cluster discovery and details |
| `k8s.io/client-go` | v0.29.3 | Kubernetes client with TLS transport caching |
| `golang.org/x/time/rate` | v0.5.0 | Token bucket rate limiting |
| `golang.org/x/sync/singleflight` | v0.20.0 | Deduplicate concurrent token refreshes |

---

## Key File Map

| File | Purpose |
|------|---------|
| `cmd/server/main.go` | Entry point: wires mux + middleware + MCP server, graceful shutdown, HTTP timeouts |
| `internal/auth/oauth.go` | OAuth endpoints, Dynamic Client Registration, Google token exchange, PKCE, singleflight refresh, scheme detection |
| `internal/auth/middleware.go` | Bearer token validation, session injection into context |
| `internal/auth/session.go` | In-memory store with TTLs (24h sessions, 5 min codes/pending), client registry, cleanup goroutine |
| `internal/auth/ratelimit.go` | Per-IP rate limiter (10 req/min) for auth + registration endpoints |
| `internal/tools/tools.go` | MCP tool definitions + handlers with input validation, context timeouts, pagination limits |
| `internal/k8s/client.go` | GKE API calls with cluster details cache (10 min TTL), builds `kubernetes.Clientset` per user |

---

## What Changed From v1

| Aspect | v1 (Original) | v2 (Current) |
|--------|---------------|--------------|
| Transport | SSE (`GET /sse` + `POST /mcp`) | StreamableHTTP (`POST /mcp` directly) |
| Auth | Raw `api_server` + `token` per request | OAuth 2.0 proxy with sessions |
| Client registration | None | Dynamic Client Registration (RFC 7591) |
| PKCE | None | S256 required |
| Token management | Caller's problem (manual `gcloud auth print-access-token`) | Server-side refresh with singleflight |
| Token exposure | Token in LLM context window | Token server-side only; client gets opaque session ID |
| Tools | 8 (including CRD tools) | 6 (cluster-focused, no CRD tools) |
| K8s clients | Typed + Dynamic | Typed only |
| Client caching | By `sha256(token)[:8]` with 5 min TTL | Cluster details cache (endpoint + CA) with 10 min TTL |
| Rate limiting | None | Token bucket, 10 req/min per IP |
| Graceful shutdown | None | SIGTERM/SIGINT with 15s drain |
| Pagination | None | 500 pods, 200 events, 500 nodes |
| Log safety | No limit | 1 MB cap via `io.LimitReader` |
| Panic recovery | None | `recoverMiddleware` |

---

## Interview Prep

### Q: Why implement an OAuth 2.0 authorization server instead of just accepting raw tokens?

**A:** Three reasons. First, token exposure -- with raw tokens, the Google access token passes through the LLM's context window and could end up in conversation logs. The OAuth proxy keeps Google tokens server-side; the client only sees an opaque session ID. Second, token refresh -- GKE tokens expire in ~1 hour. With raw tokens, the user has to manually run `gcloud auth print-access-token` when calls start failing. The OAuth proxy refreshes automatically using the stored refresh token, with singleflight to deduplicate concurrent refreshes. Third, multi-user support -- the OAuth model gives each user their own session with proper identity (email from Google's userinfo endpoint), while raw tokens provide no user attribution.

### Q: Explain the two different client_ids in this system.

**A:** There are two completely separate client_id values. The MCP client_id identifies the application -- Claude Code, Cursor, etc. It is obtained via Dynamic Client Registration (`POST /register`) and used at `/authorize` and `/token`. Each installation registers independently and gets its own client_id. The Google client_id (`GOOGLE_CLIENT_ID` env var) identifies the k8scope server to Google. It is used internally when redirecting to Google's login page. Claude Code never sees the Google client_id. A key concept: client_id identifies the APPLICATION, not the user. Multiple users share the same client_id. User identity comes from the Google login.

### Q: How does singleflight work for token refresh, and why is the double-check inside the callback important?

**A:** `singleflight.Group.Do` takes a key (session ID) and a function. If multiple goroutines call `Do` with the same key concurrently, only one executes the function -- the others block and receive the same result. This prevents thundering herd on Google's token refresh endpoint when multiple MCP requests arrive simultaneously for the same user whose token is near expiry.

The double-check inside the callback (re-checking `ExpiresAt` before calling Google) handles a subtle race: goroutine A enters `Do`, goroutine B calls `Do` with the same key and blocks. But goroutine C, which arrived just before A, already completed a refresh in a previous singleflight cycle. A's callback re-checks and sees the token is now fresh, so it returns without calling Google. Without the double-check, A would make a redundant refresh call.

Important: singleflight only deduplicates in-flight calls. Once `Do` returns, the key is forgotten. Future calls execute fresh -- there is no caching.

### Q: Why is WriteTimeout omitted on the HTTP server?

**A:** MCP uses SSE (Server-Sent Events) for streaming responses. SSE connections are long-lived -- the server keeps the response open and sends events as they become available. A `WriteTimeout` would kill these connections after the deadline. Since some streaming responses can take an indefinite amount of time, any fixed timeout would be wrong. Protection against truly idle connections comes from `IdleTimeout: 120s`, which closes keep-alive connections that have no active request.

### Q: How does K8s client-go's transport cache interact with a multi-user server?

**A:** `kubernetes.NewForConfig` is cheap -- it just creates wrapper structs. The expensive work (TLS handshake, HTTP/2 connection setup) is done by `http.Transport`, which is cached by client-go's internal `tlsTransportCache`. The critical detail: **BearerToken is NOT part of the cache key.** The cache key consists of TLS configuration (CA cert, client cert, server name). This means all users targeting the same cluster share the same transport and TCP connections, with only the `Authorization` header differing per request. This is why creating a new `Clientset` per request is not wasteful -- the heavy lifting is shared.

### Q: Walk through what happens when a user's session expires mid-conversation.

**A:** Sessions have a 24-hour TTL (checked at `GetSession` on every request). When a request arrives with an expired session ID, the middleware returns HTTP 401 before the MCP handler ever sees the request. Claude Code receives the 401 and triggers a re-authentication flow: it already has the client_id from the initial registration, so it skips registration, opens the browser for Google login, receives a new auth code, exchanges it for a new session ID via `/token`, and resumes sending MCP requests with the new Bearer token. The Google refresh token from the expired session is lost (it was in-memory only), so the user must log in to Google again.

### Q: Explain the port-agnostic loopback redirect URI matching and why it is necessary.

**A:** Claude Code opens a temporary HTTP listener on a random port to receive the OAuth callback (`http://127.0.0.1:54321/callback`). The port changes every login because the OS assigns it. RFC 8252 Section 7.3 specifies that for native apps using loopback redirects, the redirect URI match must compare scheme + host + path only, ignoring port. The `matchesRegisteredURI` helper implements this. Without port-agnostic matching, the user would need to re-register a new redirect URI for every login attempt, which defeats the purpose of reusable client registration.

---

## Related Notes

- [[notes/AuthNZ/oauth-oidc-and-workload-identity|OAuth, OIDC & Workload Identity Federation]]
- [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping, and TTYs in Kubernetes]]
- [[notes/AI-Tooling/claude-code-internals|Claude Code Internals -- Skills, Agents, Hooks, and the Plugin System]]
