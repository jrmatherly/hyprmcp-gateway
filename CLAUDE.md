# CLAUDE.md — hyprmcp-gateway

Go module: `github.com/hyprmcp/mcp-gateway` (upstream path retained — standard for forks) | Go 1.24.5
Fork repo: `github.com/jrmatherly/hyprmcp-gateway`
Container: `ghcr.io/jrmatherly/hyprmcp-gateway`
Branding: **Apollos AI** / `apollosai.dev`

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

## Commit Conventions

- **Release-please** uses [Conventional Commits](https://www.conventionalcommits.org/) — commit prefixes determine version bumps
- `feat:` → minor bump, `fix:` → patch bump, `feat!:` / `BREAKING CHANGE:` → major bump
- `ci:`, `docs:`, `chore:`, `style:` → **no release** — use these for non-code changes
- Release-please opens a PR; merging the PR creates the GitHub Release and triggers Docker build
- Always merge release-please PRs with **merge commit** (not squash/rebase)

## Codebase Notes

- **Fork of hyprmcp/mcp-gateway** — Go module path deliberately kept as upstream; do NOT rename `go.mod` or import paths
- **Container image** is `ghcr.io/jrmatherly/hyprmcp-gateway` (not the upstream `ghcr.io/hyprmcp/mcp-gateway`)
- **GitHub Actions** must be SHA-pinned (no bare `@v3` tags); see `.github/workflows/`
- **RELEASE_PAT secret** required in GitHub repo settings for release-please (not GITHUB_TOKEN — tag pushes must trigger docker workflow)
- **`htmlresponse/template.html`** loads `@hyprmcp/mcp-install-instructions-generator` via CDN — upstream dependency, needs forking/vendoring (see `.scratchpad/TODO.md`)
- **Outstanding TODOs** tracked in `.scratchpad/TODO.md` (Dex branding fork, npm package, hosted assets)
- **No tests exist** — there are zero `_test.go` files in the project
- **config.yaml is gitignored** — copy from `examples/config.yaml` for local dev
- **Config validation**: `serverMetadataProxyEnabled` must be true when `dynamicClientRegistration.enabled` is true; `dexGRPCClient.addr` is also required
- **Config hot-reload** watches the parent directory (not the file directly) for K8s ConfigMap/Secret symlink compatibility
- **Code style**: Go files use tabs (indent 4); all other files use 2-space indent (`.editorconfig`)
- **Telemetry system**: When `telemetry.enabled`, the proxy injects `hyprmcpPromptAnalytics`/`hyprmcpHistoryAnalytics` fields into `tools/list` responses and strips them from `tools/call` requests before forwarding upstream
- **Project indexes maintained in 3 files** — keep in sync: `PROJECT_INDEX.md`, `PROJECT_INDEX.json`, `.claude/docs/INDEX.md`
- **golangci-lint v2** config at `.golangci.yml` — `version: "2"` header required; `default: standard` + gosec enabled
- **Lint exclusions** reference TODO items — suppressed upstream issues tracked in `.scratchpad/TODO.md` (G114, errcheck)
- **GitHub Actions path filters** — Docker and security workflows only trigger on Go/Dockerfile changes; tag pushes are never filtered
- **Concurrency groups** — all workflows cancel stale runs; Docker workflow protects tag pushes (`cancel-in-progress: false` for tags)

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
- **Auto-tidy after go.mod edit**: `go mod tidy` runs after go.mod changes (PostToolUse)
- **Sensitive file guard**: Blocks edits to `*.secret.env` and `*.env.local` (PreToolUse)
- **go.sum guard**: Blocks direct edits to `go.sum` — must use `go mod tidy` (PreToolUse)

### Skills
- `/gen-test <package>` — Generate table-driven `_test.go` files (stdlib only, no testify)
- `/release-notes` — Draft conventional commit messages from staged changes
- `/docker-test` — Build and smoke-test the Docker image locally
- `/config-check [path]` — Validate config.yaml against schema and constraint rules

### Agents
- `security-reviewer` — Read-only sonnet agent auditing oauth/, proxy/, webhook/ for security issues
- `test-coverage-planner` — Analyzes codebase to identify and prioritize test gaps

### MCP Servers (`.mcp.json`)
- **Docker** — Container management (build, run, inspect, logs) via `@modelcontextprotocol/server-docker`

## Security Scanning

- **CodeQL** — enabled via GitHub default setup (no workflow file); deep semantic SAST for Go
- **govulncheck** — `.github/workflows/security.yaml`; SARIF upload to GitHub Security tab; weekly scheduled scan
- **gosec** — enabled in golangci-lint (`.golangci.yml`); runs locally via `mise run lint` and in auto-lint hook
- All three feed into GitHub Security tab → Code scanning alerts
- **Renovate** — `config:best-practices` with `gomodTidy`, grouped GHA updates, self-image excluded
