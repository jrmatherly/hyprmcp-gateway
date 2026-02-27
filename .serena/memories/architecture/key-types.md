# Key Types and Interfaces

## config.Config
Top-level config: Host, Authorization, DexGRPCClient, Proxy[]

## config.Proxy
Per-route config: Path, Http (upstream URL), Authentication, Telemetry, Webhook

## oauth.Manager
Holds JWK set, config, auth server metadata. Methods:
- NewManager(ctx, config) → creates JWK cache from auth server
- Register(mux) → registers OAuth endpoints
- Handler(next) → JWT validation middleware
- UpdateWWWAuthenticateHeader(resp) → rewrites upstream 401 headers

## proxy.mcpAwareTransport
Custom http.RoundTripper that intercepts JSON-RPC bodies for webhooks.
- Reads request body → parses JSON-RPC → strips telemetry args
- Reads response body → parses JSON-RPC → injects telemetry fields
- Handles both regular JSON and SSE streams

## proxy.eventStreamReader
io.ReadCloser that intercepts SSE events with a mutateFunc callback.
Parses event/data/id/retry fields per SSE spec.

## webhook.WebhookPayload
Contains: Subject, SubjectEmail, MCPSessionID, StartedAt, Duration,
AuthTokenDigest, MCPRequest, MCPResponse, UserAgent, HttpStatusCode, HttpError

## jsonrpc types
- `Message = any`, `Request = jsonrpc2.Request`, `Response = jsonrpc2.Response`
- `ParseMessage(data)` → probes for "method" field to distinguish req vs resp

## Context Keys (oauth/context.go)
- tokenKey{} → jwt.Token
- rawTokenKey{} → string (raw bearer token)
- originalURLKey{} → *url.URL (original request URL before proxy rewrite)
