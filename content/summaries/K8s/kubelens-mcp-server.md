---
title: "Summary: KubeLens MCP Server"
---

> **Full notes:** [[notes/K8s/kubelens-mcp-server|KubeLens: Building a Kubernetes MCP Server from Scratch -->]]

## Key Concepts

### Why Not kubectl?
- kubectl output is unstructured (human-formatted tables)
- No safety boundary (LLM could run `kubectl delete`)
- Auth coupled to single machine's kubeconfig
- No formal schema for LLM tool discovery
- KubeLens: JSON responses, read-only, per-request auth, JSON Schema per tool

### MCP Protocol (SSE Transport)
- `GET /sse` -- long-lived SSE connection, signals where to POST
- `POST /mcp` -- stateless JSON-RPC 2.0 calls (actual work)
- Methods: `initialize`, `notifications/initialized`, `tools/list`, `tools/call`
- Protocol version: `2024-11-05`
- Tool call timeout: **30 seconds**

### Authentication Design
- Bypasses kubeconfig entirely -- accepts raw `api_server` + `token` per request
- Bare `rest.Config{Host, BearerToken}` -- no token rotation, no exec plugins
- **Stateless**: no user sessions, each request self-contained
- **Multi-cluster**: consecutive calls can target different clusters
- Caller responsible for token refresh (GKE tokens expire ~1 hour)
- TLS: `ca_cert` (base64 PEM) takes precedence over `insecure: true`

### Client Caching
- Cache key: `apiServer + "|" + sha256(token)[:8]`
- TTL: **5 minutes**
- Token rotation = new cache key = new client automatically (no explicit invalidation)
- Protected by `sync.Mutex` (not RWMutex -- critical section is tiny)

### Dual Client Pattern
- **Typed client** (`kubernetes.Interface`): core resources (Pods, Namespaces, Events, Deployments, Nodes) -- compile-time type safety
- **Dynamic client** (`dynamic.Interface`): CRDs via arbitrary GVR, returns `unstructured.Unstructured` -- no code generation needed

### Tool Design Philosophy
- Return **structured summaries**, not raw K8s API objects
- Pod summary: name, phase, ready, restarts, node, message (vs dozens of raw fields)
- `mergeProps()` for DRY auth field injection across all tool schemas
- `get_events` uses server-side `FieldSelector: "type=Warning"` for efficiency

### The 8 Tools
- `list_namespaces` -- cluster-scoped
- `list_pods` -- podSummary with restart aggregation
- `get_pod_logs` -- tail-based, default 100 lines
- `get_events` -- server-side Warning filter
- `list_deployments` -- desired/ready/available counts
- `list_nodes` -- full condition map
- `list_crds` -- dynamic client, apiextensions.k8s.io GVR
- `get_crd_instances` -- dynamic client, arbitrary GVR

### HTTP Server Config
- ReadTimeout: **10s** (small JSON-RPC bodies)
- WriteTimeout: **60s** (large K8s API responses)
- Router: standard `http.NewServeMux` (2 endpoints, no framework needed)
- No pagination on list operations (fine for debugging, not production scraping)

## Quick Reference

```
MCP SSE Handshake:
  Client --GET /sse--> Server
  Server --SSE event: endpoint /mcp--> Client
  Client --POST /mcp (initialize)--> Server
  Client --POST /mcp (tools/list)--> Server  (gets 8 tools)
  Client --POST /mcp (tools/call)--> Server  (actual work)

Auth Flow (per request):
  Caller: gcloud auth print-access-token -> token
  Tool call: {api_server, token, ...params}
  Server: rest.Config{Host: api_server, BearerToken: token}
  Cache: sha256(token)[:8] as key, 5min TTL

Client config (Claude Code):
  {
    "mcpServers": {
      "kubelens": { "type": "sse", "url": "http://localhost:8080/sse" }
    }
  }

Scale tested:
  1,018 nodes        (list_nodes)
  245 CRDs           (list_crds)
  537 VirtualServices (get_crd_instances)
```
