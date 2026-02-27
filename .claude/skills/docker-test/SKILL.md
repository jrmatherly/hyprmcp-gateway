---
name: docker-test
description: Build and smoke-test the Docker image locally to validate Dockerfile and binary startup
user_invocable: true
---

# Docker Build & Smoke Test

Build the Docker image locally and verify the binary starts correctly.

## Usage

`/docker-test` — no arguments needed

## Instructions

1. Read the `Dockerfile` to understand the build stages and expected binary name
2. Build the image locally:

```bash
docker build -t hyprmcp-gw:test .
```

3. Verify the binary starts and responds to `--help`:

```bash
docker run --rm hyprmcp-gw:test --help
```

4. Verify the expected entrypoint and user:

```bash
docker inspect hyprmcp-gw:test --format '{{.Config.Entrypoint}} | User: {{.Config.User}}'
```

5. Report results:
   - Build success/failure with any error output
   - Binary startup verification
   - Image size (`docker images hyprmcp-gw:test --format '{{.Size}}'`)
   - Expected: distroless base, nonroot user (65532), entrypoint `/mcp-gateway`

6. Clean up the test image:

```bash
docker rmi hyprmcp-gw:test
```

## Expected Output

```
Build:     OK (XX.Xs)
Binary:    OK (--help exits 0)
Image:     XX MB
Base:      distroless nonroot
User:      65532
```
