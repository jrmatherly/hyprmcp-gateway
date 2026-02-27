---
name: release-notes
description: Draft conventional commit messages from staged changes for release-please
user_invocable: true
---

# Release Notes Drafter

Analyze staged git changes and draft a conventional commit message compatible with release-please.

## Instructions

1. Run `git diff --cached --stat` to see staged files
2. Run `git diff --cached` to read the full diff
3. Determine the change type:
   - `feat`: new feature or capability
   - `fix`: bug fix
   - `chore`: maintenance, deps, CI
   - `refactor`: code restructuring without behavior change
   - `docs`: documentation only
4. Determine the scope from modified packages:
   - `oauth` — oauth/ package
   - `proxy` — proxy/ package
   - `config` — config/ package
   - `cmd` — cmd/ package
   - `webhook` — webhook/ package
   - `jsonrpc` — jsonrpc/ package
   - `log` — log/ package
   - `htmlresponse` — htmlresponse/ package
   - Omit scope if changes span 3+ packages
5. Check if the change is breaking (API or config schema change) — add `!` suffix
6. Draft the commit message:

```
<type>(<scope>): <short description>

<body explaining what and why>

BREAKING CHANGE: <if applicable>
```

7. Present the message for user review before committing
