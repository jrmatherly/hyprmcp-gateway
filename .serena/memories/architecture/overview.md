# hyprmcp-gw Architecture Overview

## Identity
Module: github.com/hyprmcp/mcp-gateway | Go 1.24.5 | ~1700 LOC | 17 source files
Binary: mcp-gateway | Container: ghcr.io/jrmatherly/hyprmcp-gateway

## Package Map
- cmd/ → Cobra CLI, serve command, config hot-reload (fsnotify), router creation
- config/ → YAML config types (Config, Authorization, Proxy, DexGRPCClient, URL), validation, TLS
- oauth/ → Manager (JWK cache + JWT middleware), RFC9728 metadata, Dex gRPC client reg, auth proxy
- proxy/ → ReverseProxy with mcpAwareTransport (JSON-RPC interception), SSE stream reader
- proxy/proxyutil/ → Rewrite/ModifyResponse chain utilities
- webhook/ → WebhookPayload type, async Send() function
- jsonrpc/ → Message type aliases, ParseMessage (probes "method" to distinguish req vs resp)
- log/ → Context-propagated logr/stdr logging
- htmlresponse/ → Browser fallback (Accept:text/html) with embedded template

## Request Flow
HTTP → CORS → delegateHandler (hot-swappable) → ServeMux → [htmlresponse] → [oauth JWT middleware] → ReverseProxy(mcpAwareTransport) → upstream

## Key Features
1. Config hot-reload via fsnotify (watches parent dir for K8s ConfigMap support)
2. Per-route auth: authentication.enabled toggles JWT validation per proxy entry
3. Telemetry: injects hyprmcpPromptAnalytics/HistoryAnalytics into tools/list, strips from tools/call
4. Webhook: async POST with full JSON-RPC context, timing, auth digest, subject info
5. SSE interception: eventStreamReader parses SSE events for mutation before forwarding
6. Dynamic client registration: Dex gRPC, rate-limited 3/10min, public or confidential clients
7. OAuth metadata: RFC9728 protected resource metadata, auth server metadata proxy

## OAuth Endpoints
- /.well-known/oauth-protected-resource/ → RFC9728 resource metadata
- /.well-known/oauth-authorization-server → Auth server metadata proxy
- /oauth/register → Dynamic client registration (Dex gRPC)
- /oauth/authorize → Authorization redirect with scope injection

## CI/CD
- release-please on push to main → version bump PRs
- Docker multi-arch (amd64+arm64), cosign signing, distroless nonroot base
