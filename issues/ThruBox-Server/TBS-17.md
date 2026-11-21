# [FEATURE]: Report the server version (`--version`, startup log, `/health`) and let clients discover `max_payload_size`

## Feature and its Use Cases

**Current behaviour** (`main` @ `f026f22`, `VERSION` = `1.0.0`)

- The binary has no version information. There is no `version` variable and no `-ldflags -X`. The Dockerfile builds with `-ldflags="-s -w"` only. `VERSION` is read only by `version-release.yml`, and `.dockerignore` excludes it from the image build context.
- No flags are parsed. `relay-server --version` ignores the argument and **starts the server** on the configured port (3000 by default), creating `./data/relay.db`:

  ```console
  $ ./relay-server --version
  level=INFO msg="configuration loaded" config_file=config.yaml port=3000 ...
  level=INFO msg="database initialized" path=./data/relay.db
  level=INFO msg="auto-purge enabled" ttl_days=7
  level=INFO msg="relay server starting" addr=0.0.0.0:3000
  ```

- The startup log and `GET /health` (`{"status":"ok","timestamp":"..."}`) do not include a version, so an operator cannot confirm which release a running instance or image is on.
- Clients cannot learn `messages.max_payload_size` before uploading. They find out from a `413` after sending the whole body.
- Because a client cannot detect the server version, the ThruBox-Client README has to say: "Older servers send no `seq` and ignore `after` and `limit`, returning the whole mailbox."

**Proposal**

1. Add `var version = "dev"` in `cmd/relay` and log it in the `configuration loaded` line. Handle `-version` / `--version` by printing the version and exiting **before** loading config or opening the database.
2. Inject the version at build time. `VERSION` is excluded by `.dockerignore`, so `$(cat VERSION)` inside the Dockerfile will not work. Use an `ARG VERSION=dev` plus `-ldflags "-X main.version=${VERSION}"`, and pass it from `docker-publish.yml` (`build-args: VERSION=${{ steps.meta.outputs.version }}`), or stop excluding `VERSION`.
3. Add a `version` field to the `/health` JSON. It is additive, so existing clients are unaffected. The SDK's `HealthResponse` can gain an optional field later.
4. Optional, needs a maintainer decision: also expose the client-relevant limits, either as extra `/health` fields or as `GET /api/info`, for example `{"version":"1.0.0","maxPayloadSize":524288,"maxPageSize":1000}`. This also needs agreement on what may be shown without the API key.

Tests: the `/health` response includes `version`; `--version` exits without opening the database.

## Additional Context

- Relevant files: `cmd/relay/main.go`, `internal/handler/health.go`, `Dockerfile`, `.github/workflows/docker-publish.yml`, README API section.
- Some limits are already discoverable and do not need an endpoint: the page size cap (1000) is a documented constant, the TTL can be read from each message's `expiresAt`, and an API-key requirement shows up as `401`.
- Environment: Linux, built with Go 1.24.6 (`go.mod` says `go 1.23.12`).

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
