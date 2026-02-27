# Project Index: hyprmcp-gw

Generated: 2026-02-27 | Version: 0.5.1 | 1,691 LOC | 22 Go files | 0 test files

## Project Structure

```
hyprmcp-gw/
├── main.go                    # Entrypoint
├── cmd/
│   ├── root.go                # Cobra root command (serve is default)
│   └── serve.go               # HTTP server, config hot-reload, router
├── config/
│   └── config.go              # YAML config types, parsing, validation
├── oauth/
│   ├── oauth.go               # Manager: JWK cache, JWT middleware
│   ├── context.go             # Context keys (token, rawToken, originalURL)
│   ├── misc.go                # Proxy rewrite helpers
│   ├── protected_resource.go  # RFC 9728 metadata endpoint
│   ├── authorization_server_metadata.go  # Auth server metadata proxy
│   ├── authorization.go       # Authorization redirect with scope injection
│   └── dynamic_client_registration.go    # Dex gRPC client registration
├── proxy/
│   ├── proxy.go               # NewProxyHandler (ReverseProxy factory)
│   ├── transport.go           # mcpAwareTransport (JSON-RPC interception)
│   ├── event_stream.go        # SSE Event type
│   ├── event_stream_reader.go # SSE stream interceptor
│   └── proxyutil/
│       ├── util.go            # RewriteFullFunc, RewriteHostFunc, RemoveCORSHeaders
│       └── chain.go           # RewriteChain, ModifyResponseChain
├── webhook/
│   ├── types.go               # WebhookPayload struct
│   └── webhook.go             # Send()
├── jsonrpc/
│   └── types.go               # Message/Request/Response, ParseMessage
├── log/
│   └── log.go                 # Context-propagated logr/stdr
└── htmlresponse/
    ├── handle.go              # Browser Accept:text/html fallback
    └── template.html          # Embedded HTML template
```

## Entry Points

- **CLI**: `main.go` → `cmd.NewRootCommand()` (Cobra, default=serve)
- **Server**: `cmd/serve.go:38` → `runServe()` starts HTTP on `:9000`
- **Docker**: `Dockerfile` → distroless nonroot, `ENTRYPOINT ["/mcp-gateway"]`

## Core Modules

### cmd (203 LOC)
- `NewRootCommand()` — Cobra root, default serve behavior
- `runServe()` — HTTP server with CORS, config hot-reload via fsnotify
- `newRouter()` — Creates ServeMux with OAuth + proxy routes
- `WatchConfigChanges()` — fsnotify on parent dir (K8s ConfigMap compatible)
- `delegateHandler` — Hot-swappable http.Handler wrapper

### config (212 LOC)
- `Config` — Top-level: Host, Authorization, DexGRPCClient, Proxy[]
- `Proxy` — Per-route: Path, Http.Url, Authentication, Telemetry, Webhook
- `URL` — Custom url.URL with JSON/YAML marshal support
- `ParseFile()/Parse()` — YAML decoding + validation
- `DexGRPCClient.ClientTLSConfig()` — mTLS config (all-or-nothing)

### oauth (576 LOC)
- `Manager` — JWK cache (lestrrat-go/jwx), JWT validation middleware
- `Manager.Register(mux)` — Registers 4 OAuth endpoints
- `Manager.Handler(next)` — JWT Bearer validation middleware
- `NewProtectedResourceHandler` — RFC 9728 metadata, proxies upstream if available
- `NewAuthorizationServerMetadataHandler` — Proxies OIDC/.well-known metadata
- `NewAuthorizationHandler` — Redirect with scope injection (openid, profile, email)
- `NewDynamicClientRegistrationHandler` — Dex gRPC, rate-limited 3/10min/IP
- `GetMedatata()` — Fetches from oauth-authorization-server then openid-configuration
- Context helpers: `TokenContext`, `GetToken`, `GetRawToken`, `WithOriginalURL`, `GetOriginalURL`

### proxy (487 LOC)
- `NewProxyHandler()` — httputil.ReverseProxy with rewrite chain + mcpAwareTransport
- `mcpAwareTransport.RoundTrip()` — Intercepts POST bodies for webhook/telemetry
- `handler.HandleRequestData()` — Parses JSON-RPC request, strips telemetry args from tools/call
- `handler.HandleResponseData()` — Parses JSON-RPC response, injects telemetry fields into tools/list
- `eventStreamReader` — io.ReadCloser SSE parser with per-event mutation
- `Event` — SSE event type (Event, Data, ID, Retry)
- `getTelemetryInputs()` — Returns hyprmcpPromptAnalytics/HistoryAnalytics schema fields

### proxyutil (77 LOC)
- `RewriteFullFunc()` — Full URL rewrite (scheme, host, path, query)
- `RewriteHostFunc()` — Host-only rewrite
- `RemoveCORSHeaders()` — Strips upstream CORS to prevent duplicates
- `RewriteChain()/ModifyResponseChain()` — Composable middleware chains

### webhook (51 LOC)
- `WebhookPayload` — Subject, email, session ID, timing, auth digest, JSON-RPC req/resp
- `Send()` — HTTP POST with JSON body, fires async from transport

### jsonrpc (34 LOC)
- `Message = any`, `Request = jsonrpc2.Request`, `Response = jsonrpc2.Response`
- `ParseMessage()` — Probes "method" field to distinguish request vs response

### log (36 LOC)
- `Root()/Get(ctx)/Add(ctx, logger)` — Context-propagated logr/stdr

### htmlresponse (77 LOC)
- `NewHandler().Handler(next)` — Renders HTML for Accept:text/html, passes through otherwise

## Configuration

- `config.yaml` — Local dev config (gitignored)
- `examples/config.yaml` — Example with auth + public proxy routes
- `mise.toml` — Go 1.25.7, golangci-lint 2, tasks: serve/lint/tidy
- `release-please-config.json` — Release automation config
- `renovate.json` — Dependency update automation (`config:best-practices`, `gomodTidy`)
- `.mcp.json` — Project-level MCP server configuration (Docker)
- `.golangci.yml` — golangci-lint v2: standard linters + gosec, per-file exclusions

## Documentation

- `README.md` — Project overview
- `CHANGELOG.md` — Release history (auto-generated)
- `CONTRIBUTING.md` — Contribution guidelines
- `CLAUDE.md` — AI assistant instructions

## Claude Code Automations

### Hooks (`.claude/settings.json`)

| Hook | Trigger | Action |
|------|---------|--------|
| Auto-lint | PostToolUse `Edit\|Write` | Runs `mise run lint`, tails last 20 lines |
| Auto-tidy | PostToolUse `Edit\|Write` | Runs `go mod tidy` when go.mod is edited |
| Sensitive file guard | PreToolUse `Edit\|Write` | Blocks edits to `*.secret.env` and `*.env.local` |
| go.sum guard | PreToolUse `Edit\|Write` | Blocks direct edits to `go.sum` |

### Skills (`.claude/skills/`)

| Skill | Invocation | Purpose |
|-------|-----------|---------|
| gen-test | `/gen-test <package>` | Generate table-driven `_test.go` files (stdlib only) |
| release-notes | `/release-notes` | Draft conventional commit messages from staged changes |
| docker-test | `/docker-test` | Build and smoke-test the Docker image locally |
| config-check | `/config-check [path]` | Validate config.yaml against schema and constraints |
| go-sec-audit | `/go-sec-audit [package]` | Run gosec, cross-reference against exclusions and TODO tracker |
| dep-check | `/dep-check` | Run govulncheck locally and check for outdated dependencies |

### Agents (`.claude/agents/`)

| Agent | Model | Tools | Scope |
|-------|-------|-------|-------|
| security-reviewer | sonnet | Read, Grep, Glob | Audit oauth/, proxy/, webhook/ for security issues |
| test-coverage-planner | sonnet | Read, Grep, Glob | Analyze codebase and prioritize test gaps |
| api-contract-reviewer | sonnet | Read, Grep, Glob, WebFetch | Review oauth/, proxy/, jsonrpc/ for protocol compliance |

### MCP Servers (`.mcp.json`)

| Server | Package | Purpose |
|--------|---------|---------|
| docker | `@modelcontextprotocol/server-docker` | Container management (build, run, inspect, logs) |

## Test Coverage

- Unit tests: **0 files**
- Integration tests: **0 files**
- No test files exist in the codebase

## Key Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| spf13/cobra | 1.10.2 | CLI framework |
| fsnotify/fsnotify | 1.9.0 | Config file watching |
| go-chi/cors | 1.2.2 | CORS middleware |
| go-chi/httprate | 0.15.0 | Rate limiting |
| go-logr/logr + stdr | 1.4.3 | Structured logging |
| lestrrat-go/jwx/v3 | 3.0.12 | JWK/JWT validation |
| lestrrat-go/httprc/v3 | 3.0.2 | HTTP resource cache |
| dexidp/dex/api/v2 | 2.4.0 | Dex gRPC client |
| modelcontextprotocol/go-sdk | 0.5.0 | MCP types |
| sourcegraph/jsonrpc2 | 0.2.1 | JSON-RPC types |
| opencontainers/go-digest | 1.0.0 | Token digest hashing |
| google/jsonschema-go | 0.3.0 | JSON Schema (telemetry) |

## Request Flow

```
HTTP :9000 → CORS → delegateHandler → ServeMux
  ├── /.well-known/oauth-protected-resource/ → RFC9728 metadata
  ├── /.well-known/oauth-authorization-server → auth metadata proxy
  ├── /oauth/register → Dex gRPC client registration (rate-limited)
  ├── /oauth/authorize → redirect with scope injection
  └── /path/from/config → htmlresponse → [JWT auth] → ReverseProxy
        └── mcpAwareTransport → [JSON-RPC intercept] → upstream
              └── async webhook dispatch
```

## Quick Start

```bash
mise install                    # Install Go + golangci-lint
cp examples/config.yaml .      # Copy example config
mise run serve                  # Run on :9000
mise run lint                   # Lint
mise run tidy                   # go mod tidy
```

### CLI Flags
```
--config, -c     Config file (default: config.yaml)
--addr, -a       Listen address (default: :9000)
--auth-proxy-addr  Separate auth proxy listener
--verbosity, -v    Log verbosity level
```
