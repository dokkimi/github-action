# Dokkimi GitHub Action

Run [Dokkimi](https://dokkimi.dev) integration, E2E, and visual regression tests in GitHub Actions. The action sets up a single-node Kubernetes cluster on the runner and executes your test suite — no infrastructure expertise required.

## Quick start

```yaml
# .github/workflows/test.yml
name: Integration Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5

      - name: Build my service
        run: docker build -t my-app:latest .

      - uses: dokkimi/github-action@v1
        with:
          tests: .dokkimi/
```

Build your Docker images before calling the action. Since the action uses the host Docker daemon, images you build are immediately available to Kubernetes pods.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `tests` | *required* | Path to `.dokkimi/` directory or a specific definition file |
| `max-parallel` | `6` | Maximum concurrent test namespaces |
| `max-booting` | `2` | Maximum namespaces booting simultaneously |
| `timeout` | `30000` | HTTP request timeout in milliseconds |
| `viewport-width` | `1280` | Default browser viewport width for UI tests |
| `viewport-height` | `720` | Default browser viewport height for UI tests |
| `dokkimi-version` | `latest` | Dokkimi CLI version to install |

## What the action does

1. Frees disk space on the runner
2. Installs [k3s](https://k3s.io) using the host Docker daemon (`--docker`) as the container runtime
3. Waits for the cluster to be ready
4. Pre-pulls required external images
5. Installs the Dokkimi CLI globally via npm
6. Runs your tests with `dokkimi run --ci`
7. Cleans up (tears down namespaces, uninstalls k3s)

If any test fails, the action exits with a non-zero code and the workflow step is marked as failed.

## Examples

### Multiple services

```yaml
steps:
  - uses: actions/checkout@v5

  - name: Build services
    run: |
      docker build -t api-gateway:latest -f services/api/Dockerfile .
      docker build -t user-service:latest -f services/user/Dockerfile .
      docker build -t web-app:latest -f apps/web/Dockerfile .

  - uses: dokkimi/github-action@v1
    with:
      tests: .dokkimi/
```

### Running a subset of tests

```yaml
# A specific file
- uses: dokkimi/github-action@v1
  with:
    tests: .dokkimi/definitions/checkout.yaml

# A subdirectory
- uses: dokkimi/github-action@v1
  with:
    tests: .dokkimi/api-tests/
```

### Custom configuration

```yaml
- uses: dokkimi/github-action@v1
  with:
    tests: .dokkimi/
    max-parallel: 3
    max-booting: 1
    timeout: 60000
    viewport-width: 1920
    viewport-height: 1080
```

### Uploading failure artifacts

The Dokkimi CLI is installed globally, so it remains available in subsequent steps.

```yaml
- uses: dokkimi/github-action@v1
  with:
    tests: .dokkimi/

- name: Export failures
  if: failure()
  run: dokkimi dump --failed -o test-failures.json

- uses: actions/upload-artifact@v4
  if: failure()
  with:
    name: test-failures
    path: test-failures.json
```

## Requirements

- `ubuntu-latest` runner (or any Linux runner with Docker installed)
- Docker images built before calling the action
- The action frees disk space automatically, but runners with less than 14 GB free may run into issues with large image sets

## Troubleshooting

**k3s fails to start** — Ensure the runner has Docker installed and running. Self-hosted runners may need `--privileged` or equivalent permissions.

**Image pull timeouts** — Large external images (e.g. browser images for UI tests) can be slow on the first run. Increase `timeout` or cache images across runs with a Docker layer cache action.

**Disk space errors** — The action removes common pre-installed toolchains to free space, but very large test suites with many images may still exceed runner capacity. Consider splitting tests across multiple jobs.

**Tests pass locally but fail in CI** — Check that all images referenced in your `.dokkimi/` definitions are built in prior workflow steps. The runner starts with a clean Docker daemon.

## License

MIT
