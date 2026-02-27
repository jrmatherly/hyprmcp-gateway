---
name: config-check
description: Validate a config.yaml file against the Config schema and constraint rules from config/config.go
user_invocable: true
---

# Config Validator

Validate a gateway config file against the schema and constraint rules defined in `config/config.go`.

## Usage

`/config-check [path]` — defaults to `config.yaml` if no path given

## Instructions

1. Read `config/config.go` to understand the current `Config` struct and `validate()` method
2. Read the target config file (argument or `config.yaml`)
3. Check all validation rules:

### Required Fields
- `host` must be a valid URL
- Each `proxy[]` entry must have `path` and `http.url`

### Constraint Rules
- If `authorization.dynamicClientRegistration.enabled` is true:
  - `authorization.serverMetadataProxyEnabled` must also be true
  - `dexGRPCClient.addr` must be set
- If `dexGRPCClient` TLS fields are set, all three must be present (`tlsCert`, `tlsKey`, `tlsClientCA`)

### URL Validation
- `host` must parse as a valid URL
- `authorization.server` must parse as a valid URL
- Each `proxy[].http.url` must parse as a valid URL
- Each `proxy[].webhook.url` (if set) must parse as a valid URL

### Structural Checks
- `proxy[].path` values should not conflict (no duplicates)
- Webhook method defaults to POST if not specified

4. Report findings:

```
Config:     [path]
Status:     VALID / INVALID
Errors:     [list any validation failures]
Warnings:   [list any potential issues]
Routes:     [count] proxy routes configured
Auth:       [enabled/disabled for each route]
Telemetry:  [enabled/disabled for each route]
Webhooks:   [count] webhook endpoints configured
```
