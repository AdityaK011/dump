---
title: "KubeLens: Building a Kubernetes MCP Server from Scratch"
---

What happens when you give an LLM direct, read-only access to a Kubernetes cluster? Not through a shell that runs `kubectl` commands -- through a purpose-built server that speaks the Model Context Protocol (MCP) and hands back structured data. That is KubeLens: a Go server that bridges Claude (or any MCP-capable LLM client) to the Kubernetes API, with a design that is stateless, multi-cluster, and deliberately read-only.

This post is a full architectural walkthrough. We will cover the MCP protocol mechanics, the authentication flow (especially the non-obvious parts around GKE), the client-go patterns that differ from textbook usage, networking decisions, and the client caching strategy.

## Why Not Just Shell Out to kubectl?

The obvious approach to giving an LLM Kubernetes access is to let it run `kubectl` commands. This works, but it has problems:

1. **Unstructured output.** `kubectl get pods` returns a human-formatted table. The LLM has to parse whitespace-aligned columns. Errors in parsing are silent.
2. **Safety.** If the LLM can run `kubectl`, it can also run `kubectl delete`. You need a separate guardrail layer.
3. **Auth coupling.** kubectl reads `~/.kube/config`, which ties you to a single machine's credential setup.
4. **No schema.** The LLM has no formal description of what parameters each operation accepts.

KubeLens solves all four: it returns JSON, it only exposes read operations, it takes auth credentials per-request, and it advertises a JSON Schema for every tool.

## The MCP Protocol -- How the Handshake Works

MCP (Model Context Protocol) is Anthropic's standard for connecting LLMs to external tools over a network. KubeLens implements the **SSE transport variant**, which works like this:

```
Client                          Server (:8080)
  |                                |
  |--- GET /sse ------------------>|
  |<-- event: endpoint             |
  |    data: /mcp                  |
  |                                |
  |--- POST /mcp (initialize) --->|
  |<-- {protocolVersion, ...}      |
  |                                |
  |--- POST /mcp (tools/list) --->|
  |<-- {tools: [...8 tools]}       |
  |                                |
  |--- POST /mcp (tools/call) --->|
  |<-- {content: [{text: "..."}]}  |
```

The SSE connection (`GET /sse`) is long-lived but carries almost no data -- it just tells the client where to send JSON-RPC requests. The actual work happens via stateless `POST /mcp` calls carrying JSON-RPC 2.0 payloads.

On the server side, the SSE handler is minimal:

```go
func (s *Server) handleSSE(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    // ...
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    fmt.Fprintf(w, "event: endpoint\ndata: /mcp\n\n")
    flusher.Flush()

    <-r.Context().Done() // block until client disconnects
}
```

The `POST /mcp` handler routes four methods:

- **`initialize`** -- returns protocol version `"2024-11-05"`, server info, and capabilities (tools only, no `listChanged`).
- **`notifications/initialized`** -- acknowledgment, returns empty.
- **`tools/list`** -- returns all 8 tool definitions with full JSON Schemas.
- **`tools/call`** -- the actual work. Unpacks `{name, arguments}`, creates a 30-second context timeout, dispatches.

## Authentication -- The Most Interesting Part

This is where KubeLens deviates most sharply from standard client-go usage, and understanding *why* requires understanding how GKE authentication actually works.

### How GKE Auth Works End-to-End

When you run `kubectl get pods` against a GKE cluster, here is what happens under the hood:

1. kubectl reads `~/.kube/config` and finds an `exec`-based auth provider.
2. It shells out to `gke-gcloud-auth-plugin --mode=exec`.
3. The plugin returns a JSON `ExecCredential` containing a short-lived Google OAuth2 access token (typically 60-minute TTL).
4. kubectl sends this token as `Authorization: Bearer <token>` to the GKE API server.

The standard client-go approach mirrors this: `clientcmd.BuildConfigFromFlags()` reads kubeconfig, discovers the exec plugin, and handles token refresh automatically through client-go's auth provider framework.

### Why KubeLens Bypasses All of This

KubeLens takes a fundamentally different approach. Instead of reading kubeconfig, it accepts raw `api_server` and `token` parameters on every single tool call:

```go
cfg := &rest.Config{
    Host:        apiServer,
    BearerToken: token,
}
```

This is a "bare" rest.Config -- no token rotation, no exec plugins, no auth provider plugins. The token is baked in as a static string, and client-go's transport layer sends it as `Authorization: Bearer <token>` on every HTTP request.

The reasons are deliberate:

1. **Statelessness.** The server holds no user sessions. Each request is self-contained.
2. **Multi-cluster.** Each tool call can target a different cluster. Call `list_pods` on cluster A, then `get_events` on cluster B, in consecutive requests.
3. **Decoupled auth.** The caller (Claude, or whatever MCP client) is responsible for obtaining the token. For GKE, that means running `gcloud auth print-access-token` separately.

The tradeoff is real: tokens expire after roughly an hour for GKE. The caller must handle refresh. But for an interactive LLM session, this is perfectly fine -- the human or orchestrator can refresh the token when calls start failing with 401s.

### TLS Configuration

GKE API servers use Google-signed TLS certificates, so the default system CA bundle works. But for private clusters or custom CAs:

- **`ca_cert`**: accepts a base64-encoded PEM certificate, decoded at client creation time and passed as `rest.TLSClientConfig{CAData: decoded}`.
- **`insecure: true`**: skips TLS verification entirely via `rest.TLSClientConfig{Insecure: true}`. The code uses an `else if` -- if you provide a CA cert, it takes precedence over insecure mode.

## The Kubernetes Client -- Dual Clients and Caching

### Typed vs. Dynamic

KubeLens builds two clients from the same `rest.Config`:

- **`kubernetes.Interface`** (typed client) -- for core resources with generated Go types: Pods, Namespaces, Events, Deployments, Nodes. Type-safe, with compile-time guarantees.
- **`dynamic.Interface`** (dynamic client) -- for CRDs. Can query any GroupVersionResource without generated types, returning `unstructured.Unstructured` objects. This is what powers `list_crds` (which queries the `apiextensions.k8s.io/v1/customresourcedefinitions` GVR) and `get_crd_instances`.

### Client Caching with TTL

Creating a `kubernetes.Clientset` is expensive -- it involves HTTP/2 transport setup and TLS handshake configuration. Doing this on every request would be wasteful. So KubeLens caches clients:

```go
var (
    clientCache   = make(map[string]*cacheEntry)
    clientCacheMu sync.Mutex
    cacheTTL      = 5 * time.Minute
)

func GetOrCreateClient(apiServer, token, caCert string, insecure bool) (*Client, error) {
    h := sha256.Sum256([]byte(token))
    key := apiServer + "|" + fmt.Sprintf("%x", h[:8])
    // ...
}
```

The cache key is `apiServer + "|" + sha256(token)[:8]` -- the first 8 hex characters of the token's SHA-256 hash. This means:

- Same server + same token = cache hit (fast path).
- Same server + new token (after refresh) = new client (correct behavior).
- Different server = different client (multi-cluster support).

The 5-minute TTL is a middle ground. GKE tokens last about an hour, so most requests within a session will hit the cache. But the TTL ensures that stale clients eventually get cleaned up.

The cache is protected by a `sync.Mutex`, not a `sync.RWMutex`. This is fine -- the critical section is tiny (a map lookup and a time comparison), and the write path (creating a new client) is infrequent.

## Networking Decisions

The HTTP server uses deliberately asymmetric timeouts:

```go
httpServer: &http.Server{
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 60 * time.Second,
}
```

`ReadTimeout` is 10 seconds because JSON-RPC request bodies are small. `WriteTimeout` is 60 seconds because some Kubernetes API calls take significant time -- listing 1000+ nodes, for instance, involves the API server serializing a large response.

The router is `http.NewServeMux` from the standard library. No Gorilla, no Chi, no gin. For two endpoints, the standard mux is the right choice.

On the Kubernetes side, client-go's transport uses HTTP/2 by default when talking to the API server. Every tool call results in one or more HTTP requests. An important detail: list operations use `metav1.ListOptions{}` with no pagination -- the full result set comes back in one response. This is fine for debugging-oriented reads but would not scale for production scraping of very large clusters.

One smart optimization: the `get_events` tool uses server-side filtering via `FieldSelector: "type=Warning"`. Instead of fetching all events and filtering client-side, this pushes the filter to the API server, which is far more efficient.

## Tool Design -- Structured Summaries, Not Raw Objects

The tools do not return raw Kubernetes API objects. Instead, each tool extracts a focused summary. For pods:

```go
type podSummary struct {
    Name     string `json:"name"`
    Phase    string `json:"phase"`
    Ready    bool   `json:"ready"`
    Restarts int32  `json:"restarts"`
    Node     string `json:"node"`
    Message  string `json:"message,omitempty"`
}
```

This is a critical design choice. A raw Pod object in Kubernetes is enormous -- dozens of fields, nested specs, status conditions, volume mounts, tolerations. Sending all of that to an LLM wastes context window and makes it harder for the model to find what matters. The summary extracts the signal: is the pod running? Is it ready? How many restarts? What node?

The same pattern applies to deployments (desired/ready/available counts), nodes (condition map), and events (reason/message/object/count).

Every tool shares common auth properties via `mergeProps()`, which combines the four auth fields (`api_server`, `token`, `ca_cert`, `insecure`) with any tool-specific parameters. This keeps the schema DRY and ensures consistent auth handling.

## The 8 Tools

| Tool | Client | Key Detail |
|------|--------|------------|
| `list_namespaces` | typed | Cluster-scoped, no namespace param |
| `list_pods` | typed | Returns podSummary with restart aggregation |
| `get_pod_logs` | typed | TailLines-based, default 100 lines |
| `get_events` | typed | Server-side `type=Warning` filter |
| `list_deployments` | typed | Desired vs ready vs available |
| `list_nodes` | typed | Full condition map per node |
| `list_crds` | dynamic | Queries apiextensions.k8s.io GVR |
| `get_crd_instances` | dynamic | Arbitrary GVR, cluster-wide |

## MCP Client Configuration

For Claude Code, the configuration is a single JSON file:

```json
{
  "mcpServers": {
    "kubelens": {
      "type": "sse",
      "url": "http://localhost:8080/sse"
    }
  }
}
```

Claude connects to the SSE endpoint, receives the `/mcp` endpoint path, then sends all JSON-RPC requests via POST. The LLM sees the 8 tools with their JSON Schemas and can invoke them with the right parameters -- including obtaining a bearer token via `gcloud auth print-access-token`.

## Testing at Scale

All 8 tools were tested successfully against a live GKE cluster with:

- **1,018 nodes** -- `list_nodes` returned all of them within the 30-second tool timeout.
- **245 CRDs** -- `list_crds` via the dynamic client handled the full apiextensions list.
- **537 Istio VirtualServices** -- `get_crd_instances` with `group: networking.istio.io`, `version: v1beta1`, `resource: virtualservices` returned all instances.

Authentication via `gcloud auth print-access-token` worked seamlessly. The token was passed as the `token` parameter on each tool call, and the client cache ensured that repeated calls within the same session reused the same clientset.

## Key Takeaways

1. **MCP SSE transport is simpler than it sounds.** The SSE connection is just a signaling channel; actual RPC is plain POST. You can implement it with the Go standard library in about 20 lines.

2. **Bypassing kubeconfig for stateless multi-cluster access is a legitimate pattern.** The standard client-go flow (kubeconfig + exec plugins + token refresh) is designed for long-lived CLI tools. For a stateless server where each request might target a different cluster, constructing a bare `rest.Config` with `Host` + `BearerToken` is cleaner.

3. **Client caching by token hash solves the refresh problem elegantly.** When the token rotates, the cache key changes, and a new client is created automatically. No explicit invalidation logic needed.

4. **Returning summaries instead of raw API objects is critical for LLM tool design.** The model's context window is finite and its attention is not unlimited. Give it the signal, not the noise.

5. **The dynamic client unlocks CRD access without code generation.** You do not need to run `client-gen` for every CRD in your cluster. `dynamic.Interface` with a `schema.GroupVersionResource` handles arbitrary resources at runtime.

---

## Related Notes

- [[notes/K8s/interactive-containers-piping-and-ttys|Interactive Containers, Piping, and TTYs in Kubernetes]]
