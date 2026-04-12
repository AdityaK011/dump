---
title: "Summary: KubeLens MCP Server"
---

> **Full notes:** [[notes/K8s/kubelens-mcp-server|KubeLens: Building a Kubernetes MCP Server with OAuth 2.0 and StreamableHTTP -->]]

## Key Concepts

### Architecture
- **MCP over StreamableHTTP** -- single `POST /mcp` endpoint, JSON-RPC 2.0 (NOT REST)
- Two MCP objects: `MCPServer` (protocol engine, tool registry) + `StreamableHTTPServer` (HTTP adapter, `http.Handler`)
- Middleware chain: `recoverMiddleware` -> `corsMiddleware` -> `mux` -> `oauth.Middleware` (for /mcp) -> `StreamableHTTPServer`
- **6 tools**, all read-only: `list_clusters`, `list_pods`, `describe_pod`, `get_pod_logs`, `get_events`, `get_nodes`

### Auth: OAuth 2.0 Proxy with Dynamic Client Registration
- **Two client_ids**: MCP client_id (app identity, from `POST /register`) vs Google client_id (server identity, env var)
- `client_id` identifies the APP, not the user. User identity from Google login -> session
- **Dynamic Client Registration (RFC 7591)**: `POST /register` -> returns `client_id`. Reused across logins
- **PKCE (RFC 7636)**: `S256` enforced. Prevents auth code interception
- **Loopback redirect URIs only** -- port-agnostic matching per RFC 8252 s7.3
- **Public clients** (`token_endpoint_auth_method: "none"`) = no secret. CLI tools like Claude Code
- Session ID = opaque 64-char hex. NOT a JWT. Google tokens live server-side only

### Session Management (In-Memory Store)
- 4 maps, each with own `sync.RWMutex`: sessions (24h), pending (5 min), codes (5 min), clients (no TTL)
- **One-time use**: pending auth + auth codes deleted on read
- **`ClientID` flows through chain**: PendingAuth -> AuthCode -> validated at `/token`
- Background cleanup goroutine every **10 minutes**
- Token refresh: `singleflight.Group` keyed by session ID -- 1 goroutine calls Google, others wait

### K8s Client Layer
- No pre-configured cluster. Tool params: `project`, `location`, `cluster`
- **Cluster details cache**: endpoint + CA, keyed by `project/location/cluster`, **10 min TTL**, `sync.RWMutex`
- `kubernetes.NewForConfig` per-request is cheap -- just wrapper structs
- **Transport cache**: client-go's `tlsTransportCache` pools HTTP/2 connections. `BearerToken` NOT in cache key -- multi-user shares connections

### Security Hardening
- **Rate limiting**: 10 req/min per IP, token bucket (burst 5, refill 1/6s). Shared across all auth endpoints
- **Redirect URI validation**: loopback only (`127.0.0.1`, `::1`, `localhost`)
- **Input validation**: compiled regex for project ID, location, cluster name
- **Log cap**: `io.LimitReader(stream, 1MB)` prevents OOM
- **Panic recovery**: `recoverMiddleware` catches panics, returns 500
- **CORS rejection**: OPTIONS -> 403. `X-Content-Type-Options: nosniff`
- **Scheme detection**: `X-Forwarded-Proto` only when `TRUST_PROXY=true`

### HTTP Server Config
- `ReadHeaderTimeout: 10s` (slowloris protection)
- `IdleTimeout: 120s` (keep-alive cleanup)
- **WriteTimeout omitted** -- SSE streams stay open indefinitely
- Graceful shutdown: SIGTERM/SIGINT -> 15s drain via `srv.Shutdown`

### Pagination Limits
- Pods: **500**, Events: **200** (display top 50), Nodes: **500**
- Log streaming: **60s timeout**, **1 MB cap**
- Default tool timeout: **30s**

## Quick Reference

```
Auth Flow:
  1. GET /.well-known/oauth-authorization-server  (discovery)
  2. POST /register  { redirect_uris, client_name }  -> client_id
  3. GET /authorize?client_id=...&code_challenge=...  -> Google login
  4. GET /callback  (Google redirects back, server creates session)
  5. POST /token  { code, code_verifier, client_id }  -> session_id
  6. POST /mcp  Authorization: Bearer <session_id>    -> tool calls

Two client_ids:
  Claude Code --(MCP client_id)--> KubeLens --(Google client_id)--> Google

In-Memory Store:
  sessions:  session_id   -> Google tokens      (24h TTL)
  pending:   pending_key  -> PKCE + redirect    (5 min, one-time)
  codes:     auth_code    -> session_id         (5 min, one-time)
  clients:   client_id    -> registered client  (no TTL)

v1 -> v2 Changes:
  SSE transport         -> StreamableHTTP
  Raw token per request -> OAuth 2.0 proxy + sessions
  8 tools               -> 6 tools
  No rate limiting      -> 10 req/min per IP
  No shutdown handling  -> SIGTERM + 15s drain
  Token in LLM context  -> Token server-side only

Deployment:
  distroless/static-debian12:nonroot
  CGO_ENABLED=0 (static binary)
  Env: GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, REDIRECT_URL
```
