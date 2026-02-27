---
name: go-sec-audit
description: Run gosec on a specific package and cross-reference findings against suppression list and TODO tracker
user_invocable: true
---

# Go Security Audit

Run gosec directly on a specific package and report findings with remediation guidance.

## Usage

`/go-sec-audit <package>` — e.g., `/go-sec-audit oauth`, `/go-sec-audit proxy`, `/go-sec-audit cmd`

`/go-sec-audit` — (no args) audits all packages

## Instructions

1. Read `.golangci.yml` to understand current exclusions
2. Read `.scratchpad/TODO.md` to understand tracked suppressions
3. Run golangci-lint with only gosec on the target package:

```bash
golangci-lint run --enable-only gosec ./<package>/... 2>&1
```

Or for all packages:

```bash
golangci-lint run --enable-only gosec ./... 2>&1
```

4. For each finding, classify it:
   - **New** — not in `.golangci.yml` exclusions; needs triage
   - **Suppressed** — already excluded in `.golangci.yml`; verify the exclusion comment references a TODO item
   - **Tracked** — listed in `.scratchpad/TODO.md` with a remediation plan

5. Report results:

```
Package:     <package>
New issues:  <count>
Suppressed:  <count> (tracked in TODO.md)

New findings:
  - G<code> at <file>:<line> — <description>
    Remediation: <guidance>

Tracked suppressions:
  - G<code> at <file> — TODO #<N>: <summary>
```

6. If new findings exist, suggest whether to:
   - Fix immediately (if straightforward)
   - Add to `.golangci.yml` exclusions with a new TODO item
   - Investigate further (if the finding may be a false positive)
