# I2P Provider Test Harness

This repository provides a GitHub Actions harness for starting an I2P router inside the job container so application tests in the same job can talk to it over localhost.

The main entry point is [.github/workflows/i2p-router.yml](.github/workflows/i2p-router.yml).

The workflow includes a startup smoke check and a separate endpoint matrix job.

## What it does

- Starts an I2P router in hidden / non-transit mode.
- Binds I2CP to `127.0.0.1:7654`.
- Enables SAM on `127.0.0.1:7656` only for backends that support it in this harness.
- Exposes I2PControl for Java on `http://127.0.0.1:7657/jsonrpc` and for i2pd on `https://127.0.0.1:7650/`.
- Sends `TERM` on cleanup and waits for graceful shutdown.

## Supported backends

The workflow backend is configurable through the `backend` input and defaults to `java`.

- `java`: the standard Java router implementation.
- `i2pd`: the C++ backend.
- `emissary`: the Rust backend, launched through override hooks if needed.
- `go-i2p`: the Go backend.
- `i2pcontrol` smoke tests run on the backends that expose the control endpoint.

## Config files

The workflow keeps router-specific config out of the YAML and stores it under [.github/i2p-router/](.github/i2p-router/).

- [.github/i2p-router/java/router.config](.github/i2p-router/java/router.config)
- [.github/i2p-router/java/01-net.i2p.sam.SAMBridge-clients.config](.github/i2p-router/java/01-net.i2p.sam.SAMBridge-clients.config)
- [.github/i2p-router/i2pd/i2pd.conf](.github/i2p-router/i2pd/i2pd.conf)

## Usage

From GitHub Actions, call the workflow manually or reuse it from another workflow:

```yaml
jobs:
  router:
    uses: go-i2p/i2p-provider/.github/workflows/i2p-router.yml@main
    with:
      backend: java
```

If you need a different backend, set `backend` to `i2pd`, `emissary`, or `go-i2p`.

The workflow also runs a separate endpoint matrix job that checks `i2cp`, `sam`, and `i2pcontrol` where applicable.

## Caveats

- `java` and `i2pd` have checked-in config files.
- `java` and `i2pd` expose I2PControl locally; `go-i2p` and `emissary` are left to their backend defaults or overrides.
- `go-i2p` currently uses command-line flags rather than a separate repo-owned config file in this harness.
- `emissary` is launched through `EMISSARY_START_COMMAND`, `EMISSARY_BINARY`, or `EMISSARY_ARGS` if you need to supply a backend-specific launch command.
