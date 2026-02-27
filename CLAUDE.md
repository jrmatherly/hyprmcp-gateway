# CLAUDE.md — hyprmcp-gw

Go module: `github.com/hyprmcp/mcp-gateway` | Go 1.24.5

## Common Development Commands

### Quick Start
```bash
mise install                     # Install Go + golangci-lint
cp examples/config.yaml .        # config.yaml is gitignored; create from example
mise run serve                   # Starts on :9000
```

### Build and Run
```bash
mise run serve                   # go run . --config config.yaml
go build -o mcp-gateway .       # Compile binary
```

### CLI Flags
```
--config, -c     Config file path (default: config.yaml)
--addr, -a       Listen address (default: :9000)
--verbosity, -v  Log verbosity (higher = more)
```

### Lint and Tidy
```bash
mise run lint    # golangci-lint
mise run tidy    # go mod tidy
```

## Codebase Notes

- **No tests exist** — there are zero `_test.go` files in the project
- **config.yaml is gitignored** — copy from `examples/config.yaml` for local dev
- **Config validation**: `serverMetadataProxyEnabled` must be true when `dynamicClientRegistration.enabled` is true; `dexGRPCClient.addr` is also required
- **Config hot-reload** watches the parent directory (not the file directly) for K8s ConfigMap/Secret symlink compatibility
- **Code style**: Go files use tabs (indent 4); all other files use 2-space indent (`.editorconfig`)
- **Telemetry system**: When `telemetry.enabled`, the proxy injects `hyprmcpPromptAnalytics`/`hyprmcpHistoryAnalytics` fields into `tools/list` responses and strips them from `tools/call` requests before forwarding upstream

## Architecture Overview

MCP-Gateway is an HTTP reverse proxy for MCP (Model Context Protocol) servers with OAuth authentication and webhook-based observability.

### Key Components

1. **HTTP Server** (`cmd/serve.go`): Cobra CLI with hot-reloading config via fsnotify
2. **OAuth Manager** (`oauth/`): JWT validation using JWK sets, integrates with Dex identity provider
3. **Proxy Handler** (`proxy/`):
   - Reverse proxy with MCP-aware transport
   - Intercepts JSON-RPC messages for session tracking
   - Supports SSE (Server-Sent Events) for streaming
4. **Webhook System** (`webhook/`): Async notifications with full request/response context


### Important Design Patterns
- **Config Hot-Reload**: Uses fsnotify to watch config file changes
- **Context Propagation**: Structured logging with request context
- **Transport Interception**: Custom RoundTripper for MCP protocol awareness
- **Per-Route Auth**: Authentication can be disabled for specific routes (e.g., public endpoints)

## Claude Code Automations

### Hooks (`.claude/settings.json`)
- **Auto-lint on edit**: `mise run lint` runs after every Edit/Write (PostToolUse)
- **Sensitive file guard**: Blocks edits to `*.secret.env` and `*.env.local` (PreToolUse)

### Skills
- `/gen-test <package>` — Generate table-driven `_test.go` files (stdlib only, no testify)
- `/release-notes` — Draft conventional commit messages from staged changes

### Agents
- `security-reviewer` — Read-only sonnet agent auditing oauth/, proxy/, webhook/ for security issues
