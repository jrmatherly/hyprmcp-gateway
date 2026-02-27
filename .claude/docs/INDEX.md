# hyprmcp-gw Project Index

> Auto-generated project index — ~1,700 LOC Go, 22 source files

## Identity

| Field | Value |
|-------|-------|
| Module | `github.com/hyprmcp/mcp-gateway` |
| Go | 1.24.5 (mise: 1.25.5) |
| Binary | `mcp-gateway` |
| Container | `ghcr.io/jrmatherly/hyprmcp-gateway` |
| License | See `LICENSE` |
| Version | v0.4.0 (release-please) |

---

## Package Map

```
github.com/hyprmcp/mcp-gateway
├── main.go                          # Entrypoint → cmd.NewRootCommand()
├── cmd/
│   ├── root.go                      # Cobra root (serve is default)
│   └── serve.go                     # HTTP server, config hot-reload, router
├── config/
│   └── config.go                    # YAML config types, parsing, validation, TLS
├── oauth/
│   ├── oauth.go                     # Manager: JWK cache, JWT middleware, endpoint registration
│   ├── context.go                   # Context keys: token, rawToken, originalURL
│   ├── misc.go                      # RewriteSetOriginalURL, UpdateWWWAuthenticateHeader
│   ├── protected_resource.go        # RFC 9728 protected resource metadata
│   ├── authorization_server_metadata.go  # Auth server metadata proxy + OIDC discovery
│   ├── authorization.go             # Authorization proxy with scope injection
│   └── dynamic_client_registration.go    # Dex gRPC dynamic client registration
├── proxy/
│   ├── proxy.go                     # NewProxyHandler: ReverseProxy with rewrite chain
│   ├── transport.go                 # mcpAwareTransport: JSON-RPC interception + webhooks
│   ├── event_stream.go              # SSE Event type
│   ├── event_stream_reader.go       # eventStreamReader: SSE stream interceptor
│   └── proxyutil/
│       ├── util.go                  # RewriteFullFunc, RewriteHostFunc, RemoveCORSHeaders
│       └── chain.go                 # RewriteChain, ModifyResponseChain
├── webhook/
│   ├── types.go                     # WebhookPayload struct
│   └── webhook.go                   # Send() function
├── jsonrpc/
│   └── types.go                     # Message/Request/Response aliases, ParseMessage
├── log/
│   └── log.go                       # Context-propagated logr/stdr logging
└── htmlresponse/
    ├── handle.go                    # HTML fallback for browser Accept:text/html
    └── template.html                # Embedded HTML template
```

---

## Request Flow

```
                          ┌──────────────────────────────┐
                          │       HTTP Client / MCP       │
                          └──────────────┬───────────────┘
                                         │
                                    :9000 (default)
                                         │
                          ┌──────────────▼───────────────┐
                          │     CORS (go-chi/cors)        │
                          └──────────────┬───────────────┘
                                         │
                          ┌──────────────▼───────────────┐
                          │      delegateHandler          │
                          │  (hot-swappable via fsnotify) │
                          └──────────────┬───────────────┘
                                         │
                          ┌──────────────▼───────────────┐
                          │       http.ServeMux           │
                          └──┬────────┬────────┬─────────┘
                             │        │        │
              ┌──────────────▼──┐  ┌──▼──────────────┐  ┌──▼──────────────┐
              │  OAuth Endpoints │  │  Proxy Routes    │  │  (more routes)  │
              │  (well-known,    │  │  per config.Proxy│  │                 │
              │   register,      │  │                  │  │                 │
              │   authorize)     │  │                  │  │                 │
              └─────────────────┘  └──┬───────────────┘  └─────────────────┘
                                      │
                          ┌───────────▼───────────────┐
                          │    htmlresponse handler     │
                          │ (Accept:text/html fallback) │
                          └───────────┬───────────────┘
                                      │
                          ┌───────────▼───────────────┐
                          │  OAuth JWT middleware       │
                          │  (if authentication.enabled)│
                          └───────────┬───────────────┘
                                      │
                          ┌───────────▼───────────────┐
                          │  httputil.ReverseProxy      │
                          │  - Rewrite: host + origURL  │
                          │  - ModifyResponse: CORS +   │
                          │    WWW-Authenticate rewrite  │
                          │  - Transport:               │
                          │    mcpAwareTransport        │
                          └───────────┬───────────────┘
                                      │
                          ┌───────────▼───────────────┐
                          │  mcpAwareTransport          │
                          │  (webhook + telemetry)      │
                          │  - Parse JSON-RPC req/resp  │
                          │  - Strip/inject telemetry   │
                          │  - Intercept SSE streams    │
                          │  - Async webhook dispatch   │
                          └───────────┬───────────────┘
                                      │
                          ┌───────────▼───────────────┐
                          │   Upstream MCP Server       │
                          └───────────────────────────┘
```

---

## Configuration Schema

```yaml
# config.Config
host: http://localhost:9000/            # Required: public-facing URL

authorization:
  server: http://localhost:5556/         # Required: OAuth2/OIDC server
  serverMetadataProxyEnabled: true       # Proxy /.well-known/oauth-authorization-server
  authorizationProxyEnabled: true        # Proxy /oauth/authorize with scope injection
  dynamicClientRegistration:             # Dynamic client registration via Dex gRPC
    enabled: true
    publicClient: true                   # Create public (no secret) clients

dexGRPCClient:                           # Required when dynamicClientRegistration enabled
  addr: localhost:5557
  tlsCert: /path/to/cert.pem            # Optional: mTLS
  tlsKey: /path/to/key.pem
  tlsClientCA: /path/to/ca.pem

proxy:                                   # Array of proxy routes
  - path: /weather/mcp                   # URL path to register on mux
    http:
      url: http://upstream:8000/mcp/     # Upstream MCP server URL
    authentication:
      enabled: true                      # Enable JWT validation for this route
    telemetry:
      enabled: true                      # Inject hyprmcpPrompt/HistoryAnalytics fields
    webhook:                             # Optional: async webhook per request
      url: http://hooks:8080/webhook
      method: POST                       # Default: POST
```

---

## Key Types

### `config.Config` → `config/config.go:16`
Top-level YAML config: Host, Authorization, DexGRPCClient, Proxy[].

### `oauth.Manager` → `oauth/oauth.go:23`
Central OAuth component. Manages JWK cache, JWT middleware, endpoint registration.

### `proxy.mcpAwareTransport` → `proxy/transport.go:25`
Custom `http.RoundTripper`. Intercepts JSON-RPC request/response bodies for webhook
assembly and telemetry field injection. Handles both `application/json` and `text/event-stream`.

### `proxy.eventStreamReader` → `proxy/event_stream_reader.go:10`
`io.ReadCloser` that parses SSE streams event-by-event, applying a mutation function
to each event's data before forwarding.

### `webhook.WebhookPayload` → `webhook/types.go:10`
Full request context: subject, email, session ID, timing, auth token digest, JSON-RPC
request/response, user agent, HTTP status.

### `jsonrpc.ParseMessage` → `jsonrpc/types.go:14`
Probes for `"method"` field to distinguish Request vs Response.

---

## OAuth Endpoints

| Path | Handler | Description |
|------|---------|-------------|
| `/.well-known/oauth-protected-resource/{path}` | `NewProtectedResourceHandler` | RFC 9728 metadata; proxies upstream if available |
| `/.well-known/oauth-authorization-server` | `NewAuthorizationServerMetadataHandler` | Proxies auth server metadata; injects registration + authorization endpoints |
| `/oauth/register` | `NewDynamicClientRegistrationHandler` | Creates OAuth clients via Dex gRPC; rate-limited 3/10min per IP |
| `/oauth/authorize` | `NewAuthorizationHandler` | Redirects to auth server with injected scopes (openid, profile, email) |

---

## Telemetry System

When `telemetry.enabled` is true for a proxy route:

1. **tools/list response**: Injects `hyprmcpPromptAnalytics` and `hyprmcpHistoryAnalytics` string fields into each tool's input schema
2. **tools/call request**: Strips those telemetry fields from arguments before forwarding to upstream

This allows MCP clients to voluntarily include prompt/history context in tool calls, which
gets captured in the webhook payload but never reaches the upstream server.

---

## CI/CD

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `release.yaml` | push to main/v*.x | release-please: changelog + version bump PRs |
| `docker.yaml` | push/PR/tag | Multi-arch Docker build (amd64+arm64), push on tag, cosign signing |

**Container**: `ghcr.io/jrmatherly/hyprmcp-gateway` — distroless base, nonroot user (65532).

---

## Dependencies (direct)

| Dependency | Purpose |
|------------|---------|
| `spf13/cobra` | CLI framework |
| `fsnotify/fsnotify` | Config file watching |
| `go-chi/cors` | CORS middleware |
| `go-chi/httprate` | Rate limiting |
| `go-logr/logr` + `stdr` | Structured logging |
| `lestrrat-go/jwx/v3` | JWK/JWT parsing and validation |
| `lestrrat-go/httprc/v3` | HTTP resource cache (for JWK refresh) |
| `dexidp/dex/api/v2` | Dex gRPC client for dynamic client registration |
| `modelcontextprotocol/go-sdk` | MCP types (CallToolParams, ListToolsResult) |
| `sourcegraph/jsonrpc2` | JSON-RPC 2.0 Request/Response types |
| `opencontainers/go-digest` | Auth token digest hashing |
| `google/jsonschema-go` | JSON Schema types for telemetry field injection |

---

## Claude Code Automations

### Hooks (`.claude/settings.json`)
| Hook | Trigger | Action |
|------|---------|--------|
| Auto-lint | PostToolUse `Edit\|Write` | Runs `mise run lint`, tails last 20 lines (15s timeout) |
| Auto-tidy | PostToolUse `Edit\|Write` | Runs `go mod tidy` when go.mod is edited (30s timeout) |
| Sensitive file guard | PreToolUse `Edit\|Write` | Blocks edits to `*.secret.env` and `*.env.local` (exit 2) |
| go.sum guard | PreToolUse `Edit\|Write` | Blocks direct edits to `go.sum` (exit 2) |

### Skills (`.claude/skills/`)
| Skill | Invocation | Purpose |
|-------|-----------|---------|
| gen-test | `/gen-test <package>` | Generate table-driven `_test.go` files (stdlib only, httptest, logr) |
| release-notes | `/release-notes` | Draft conventional commit messages from `git diff --cached` |
| docker-test | `/docker-test` | Build and smoke-test the Docker image locally |
| config-check | `/config-check [path]` | Validate config.yaml against schema and constraint rules |

### Agents (`.claude/agents/`)
| Agent | Model | Tools | Scope |
|-------|-------|-------|-------|
| security-reviewer | sonnet | Read, Grep, Glob | Audit oauth/, proxy/, webhook/ for JWT, SSRF, header injection, rate-limit bypass |
| test-coverage-planner | sonnet | Read, Grep, Glob | Analyze codebase to identify and prioritize test gaps |

### MCP Servers (`.mcp.json`)
| Server | Package | Purpose |
|--------|---------|---------|
| docker | `@modelcontextprotocol/server-docker` | Container management (build, run, inspect, logs) |

---

## Dev Quick Reference

```bash
mise install           # Install Go + golangci-lint
mise run serve         # go run . --config config.yaml
mise run lint          # golangci-lint run
mise run tidy          # go mod tidy
```

### CLI Flags
```
--config, -c   Config file path (default: config.yaml)
--addr, -a     Listen address (default: :9000)
--auth-proxy-addr  Separate auth proxy listener (advanced)
--verbosity, -v    Log verbosity (0 = default)
```
