---
name: gen-test
description: Generate table-driven Go tests for a package with httptest, context helpers, and config parsing patterns
user_invocable: true
---

# Generate Go Tests

Generate comprehensive `_test.go` files for the specified package.

## Usage

`/gen-test <package>` — e.g., `/gen-test config`, `/gen-test proxy`, `/gen-test oauth`

## Instructions

1. Read all `.go` files in the target package (skip any existing `_test.go`)
2. For each exported function/method, generate table-driven tests
3. Follow these patterns:

### Test Structure
- Use `t.Run` with descriptive subtests
- Table-driven tests with `[]struct{ name string; ... }`
- `t.Parallel()` at test and subtest level where safe
- `t.Helper()` on all test helpers

### HTTP Handlers
- Use `net/http/httptest` for handler tests
- Create request with `httptest.NewRequest`
- Record response with `httptest.NewRecorder`
- Assert status code, headers, and body

### Config Parsing
- Test valid YAML configs with expected output
- Test validation failures (missing required fields, invalid combinations)
- Reference `config.ServerMetadataProxyEnabled` + `DynamicClientRegistration` constraint

### Context & Logging
- Use `logr.Discard()` for test loggers
- Propagate context with `log.IntoContext(ctx, logger)`

### Assertions
- Use standard library only: `if got != want { t.Errorf(...) }`
- No external test frameworks (no testify, no gomega)

4. Write the test file as `<package>/<filename>_test.go`
5. Run `go test ./<package>/...` to verify compilation
6. Run `go vet ./<package>/...` for correctness
