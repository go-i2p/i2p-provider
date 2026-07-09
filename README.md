# I2P Provider Test Harness

This repository provides GitHub Actions building blocks for starting an I2P router inside a job container so application tests in the *same job* can talk to it over localhost.

The building blocks are two composite actions:

- [.github/actions/i2p-router](.github/actions/i2p-router/action.yml) — starts the router as a background process and waits for it to become ready.
- [.github/actions/i2p-router-stop](.github/actions/i2p-router-stop/action.yml) — stops the router cleanly.

[.github/workflows/i2p-router.yml](.github/workflows/i2p-router.yml) is this repository's own self-test harness for those actions and its config fixtures; it is **not** meant to be called as a reusable workflow by other repositories (see [Important: use the composite action, not the workflow](#important-use-the-composite-action-not-the-workflow) below).

## What it does

- Starts an I2P router in hidden / non-transit mode.
- Binds I2CP to `127.0.0.1:7654`.
- Enables SAM on `127.0.0.1:7656` only for backends that support it in this harness.
- Exposes I2PControl for Java on `http://127.0.0.1:7657/jsonrpc/` and for i2pd on `https://127.0.0.1:7650/`.
- Sends `TERM` on cleanup and waits for graceful shutdown.

## Supported backends

The `backend` input defaults to `java`.

- `java`: the standard Java router implementation.
- `i2pd`: the C++ backend.
- `emissary`: the Rust backend, launched through override hooks if needed.
- `go-i2p`: the Go backend.
- `i2pcontrol` smoke tests run on the backends that expose the control endpoint.

## Config files

Router-specific config lives under [.github/i2p-router/](.github/i2p-router/) in this repository and is fetched by the composite action at run time (via its `fixtures_ref` input, defaulting to `main`) - consumers do not need to copy these files into their own repo.

- [.github/i2p-router/java/router.config](.github/i2p-router/java/router.config)
- [.github/i2p-router/java/webapps.config](.github/i2p-router/java/webapps.config)
- [.github/i2p-router/java/01-net.i2p.sam.SAMBridge-clients.config](.github/i2p-router/java/01-net.i2p.sam.SAMBridge-clients.config)
- [.github/i2p-router/i2pd/i2pd.conf](.github/i2p-router/i2pd/i2pd.conf)

## Important: use the composite action, not the workflow

Each GitHub Actions job runs on its own isolated runner/container. A background router process started in one job **cannot** be reached from a different job, even if that job depends on it via `needs:`. Because of this, calling `.github/workflows/i2p-router.yml` with `uses:` (i.e. as a reusable workflow) starts the router in its own isolated job - your own test steps, defined in a different job, will never be able to connect to it.

Instead, add the composite action as a **step** in the same job as your tests:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: ubuntu:24.04
      options: --init
    steps:
      - uses: actions/checkout@v4

      - name: Start I2P router
        uses: go-i2p/i2p-provider/.github/actions/i2p-router@main
        with:
          backend: go-i2p

      - name: Run your tests
        run: go test ./...

      - name: Stop I2P router
        if: always()
        uses: go-i2p/i2p-provider/.github/actions/i2p-router-stop@main
```

If you need a different backend, set `backend` to `java`, `i2pd`, or `emissary`.

### Action inputs

`go-i2p/i2p-provider/.github/actions/i2p-router`:

- `backend` (default `java`): `java`, `i2pd`, `go-i2p`, or `emissary`.
- `fixtures_ref` (default `main`): git ref of `go-i2p/i2p-provider` to pull router config fixtures from.

### Action outputs and environment variables

The action exposes outputs (`i2cp-host`, `i2cp-port`, `sam-enabled`, `sam-host`, `sam-tcp-port`, `sam-udp-port`, `i2pcontrol-enabled`, `i2pcontrol-url`, `backend`) and also sets the equivalent `I2P_ROUTER_*` environment variables for the rest of the job, so subsequent steps can use either form.

## Caveats

- `java` and `i2pd` have checked-in config files.
- `java` and `i2pd` expose I2PControl locally; `go-i2p` and `emissary` are left to their backend defaults or overrides.
- `go-i2p` currently uses command-line flags rather than a separate repo-owned config file in this harness.
- `emissary` is launched through `EMISSARY_START_COMMAND`, `EMISSARY_BINARY`, or `EMISSARY_ARGS` if you need to supply a backend-specific launch command.

