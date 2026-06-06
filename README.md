# Dokkimi GitHub Action

Run [Dokkimi](https://dokkimi.dev) integration, E2E, and visual regression tests in GitHub Actions. The action uses Docker on the runner to execute your test suite — no infrastructure expertise required.

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

Build your Docker images before calling the action. Since the action uses the host Docker daemon, images you build are immediately available to your test environments.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `tests` | *required* | Path to `.dokkimi/` directory or a specific definition file |
| `max-parallel` | `6` | Maximum concurrent test environments |
| `max-booting` | `2` | Maximum environments booting simultaneously |
| `timeout` | `30000` | HTTP request timeout in milliseconds |
| `viewport-width` | `1280` | Default browser viewport width for UI tests |
| `viewport-height` | `720` | Default browser viewport height for UI tests |
| `dump-retention-days` | `1` | Days to retain the failed-run dump artifact (0 to disable) |
| `dokkimi-version` | `latest` | Dokkimi CLI version to install |

## What the action does

1. Frees disk space on the runner
2. Pre-pulls required sidecar images
3. Installs the Dokkimi CLI globally via npm
4. Runs your tests with `dokkimi run --ci`
5. Uploads a dump artifact if any test fails (configurable via `dump-retention-days`)
6. Cleans up test environments

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

## Requirements

- `ubuntu-latest` runner (or any Linux runner with Docker installed)
- Docker images built before calling the action
- The action frees disk space automatically, but runners with less than 14 GB free may run into issues with large image sets

## Troubleshooting

**Docker not available** — Ensure the runner has Docker installed and running. Self-hosted runners may need `--privileged` or equivalent permissions.

**Image pull timeouts** — Large external images (e.g. browser images for UI tests) can be slow on the first run. Increase `timeout` or cache images across runs with a Docker layer cache action.

**Disk space errors** — The action removes common pre-installed toolchains to free space, but very large test suites with many images may still exceed runner capacity. Consider splitting tests across multiple jobs.

**Tests pass locally but fail in CI** — Check that all images referenced in your `.dokkimi/` definitions are built in prior workflow steps. The runner starts with a clean Docker daemon.

## License

MIT
