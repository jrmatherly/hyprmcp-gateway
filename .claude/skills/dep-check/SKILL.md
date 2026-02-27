---
name: dep-check
description: Run govulncheck locally and check for outdated Go dependencies
user_invocable: true
---

# Dependency Health Check

Run govulncheck and check for outdated dependencies without waiting for CI.

## Usage

`/dep-check` — no arguments needed

## Prerequisites

- `govulncheck` installed (`go install golang.org/x/vuln/cmd/govulncheck@latest`)
- If missing, the skill will install it automatically

## Instructions

1. Verify govulncheck is available:

```bash
which govulncheck || go install golang.org/x/vuln/cmd/govulncheck@latest
```

2. Run govulncheck on all packages:

```bash
govulncheck ./... 2>&1
```

3. Check for available dependency updates:

```bash
go list -m -u all 2>&1 | grep '\[' | head -30
```

4. Report results:

```
Vulnerabilities:  <count> (or "None found")
Outdated deps:    <count>

Vulnerabilities:
  - <vuln-id>: <package> — <description>
    Fixed in: <version>
    Current:  <version>

Available updates:
  - <module> <current> → <available>
  - ...
```

5. For any vulnerabilities found:
   - Check if the vulnerable code path is actually called (govulncheck does this by default)
   - Suggest `go get <module>@<fixed-version>` commands for fixes
   - Note if the vulnerability is in a direct vs. indirect dependency

6. For outdated dependencies:
   - Flag security-critical ones (crypto, net, jwx, grpc) as higher priority
   - Note that Renovate handles automated updates — only flag urgent ones
