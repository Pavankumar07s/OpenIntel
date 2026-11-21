# Docker image is built with end-of-life Go 1.23.12 and runs on end-of-life Alpine 3.20

## Bug Description

The published image is built and run on versions that no longer get security fixes:

- `Dockerfile:2` builds with `FROM golang:1.23.12-alpine`, and `go.mod:3` declares `go 1.23.12`. Go 1.23 went out of support on 2025-08-12, when Go 1.25 came out. The supported releases are now 1.27.4 and 1.26.11.
- `Dockerfile:18` runs on `FROM alpine:3.20`. Alpine 3.20 went out of support on 2026-04-01, and the current release is 3.24.

`govulncheck` run with the shipped toolchain reports **32 standard-library vulnerabilities** and exits with code 3. It counts all 32 as "called" by our code. That count comes from static analysis of the call graph, so it overstates what is exploitable today. I checked each one against how the relay runs (plain `srv.ListenAndServe()`, no TLS, no outgoing HTTP calls):

- **18 do not apply.** They are in `crypto/tls`, `crypto/x509`, `encoding/asn1` and `encoding/pem`, and the relay neither serves nor makes TLS connections.
- **5 do not apply.** They are HTTP/2 server bugs (GO-2026-6603, -6611, -6612, -6617, -6089), and the relay does not speak HTTP/2 on its plain-HTTP listener. `curl --http2-prior-knowledge` fails with `PROTOCOL_ERROR`.
- **GO-2026-6859** ("HTTP/1 server crash due to MaxBytesReader header race") does not apply as the code stands. The crash needs a handler that reads the body while another goroutine writes response headers. `HandleCreate` reads the whole body with `io.ReadAll` before it writes anything. I sent 4,800 oversized POSTs over 32 concurrent connections to a `-race` build. Every one got 413, with no race report and no crash.
- **GO-2026-6613** (CONNECT desync) does not apply, because no route returns 2xx to `CONNECT` (it gets 404). **GO-2026-4971** affects Windows only. **GO-2026-4602** (`os.Root`) is in code the relay does not use.
- **Some are reachable, with low impact.** For example GO-2026-4341: `GET /api/messages/{address}?…` with a ~1 MiB query string of ~160k distinct keys returns 200, and the `r.URL.Query()` call in that handler allocates about 26 MiB for each such request.

So I found no crash or takeover reachable today. The problem is that the shipped binary and base image are frozen on unsupported versions, so fixes for future `net/http` or Alpine security bugs will never reach deployed relays.

Here is how it drifted. Dependabot opened golang bumps #15, #33, #40, #47, #65 and #71, and an alpine bump #16. None was merged. The stale workflow closed #16, #47 and #71, and Dependabot closed the others when it opened a newer bump.

Two details matter for the fix:

1. **1.27.3 is not enough.** The last Dependabot PR (#71) moved to 1.27.3, and GO-2026-6859 is still open in 1.27.3. It was fixed in **1.27.4** and 1.26.11.
2. **Changing only the Dockerfile is not enough.** Go chooses GODEBUG defaults from the `go` line in `go.mod`. With `go 1.23.12` kept there, a binary built with Go 1.27.4 keeps the old Go 1.23 behaviour for settings that changed later. One example is `urlmaxqueryparams=0`, which means no limit on query parameters, so the fix for GO-2026-4341 stays off. I checked this with a minimal module that parses that ~1 MiB query using Go 1.27.4:
   - with `go 1.23` in `go.mod`: 159,769 parameters parsed, 26.5 MiB allocated
   - with `go 1.27` in `go.mod`: `number of URL query parameters exceeded limit`, 0 MiB allocated

## Steps to Reproduce

1. On `main`, scan with the shipped toolchain:
   ```sh
   GOTOOLCHAIN=go1.23.12 govulncheck ./...
   ```
   Result: `Your code is affected by 32 vulnerabilities from the Go standard library.` (exit code 3)
2. Apply the proposed fix and check it:
   ```sh
   go mod edit -go=1.27.4
   GOTOOLCHAIN=go1.27.4 go vet ./... && GOTOOLCHAIN=go1.27.4 go test ./...
   GOTOOLCHAIN=go1.27.4 CGO_ENABLED=1 go build -o relay ./cmd/relay
   govulncheck -mode=binary ./relay
   ```
   Result: all tests pass, then `No vulnerabilities found.`

   A `govulncheck` binary built with an older Go cannot load the packages once `go.mod` says `go 1.27.4`, so step 2 uses binary mode.

## Logs and Screenshots

```text
$ GOTOOLCHAIN=go1.23.12 govulncheck ./...
Vulnerability #1: GO-2026-6859  HTTP/1 server crash due to MaxBytesReader header race in net/http
Vulnerability #2: GO-2026-6809  Distrust compromised intermediate certificates in crypto/x509
Vulnerability #3: GO-2026-6617  HTTP/2 server crash due to HPACK encoder race in net/http
...
Your code is affected by 32 vulnerabilities from the Go standard library.

$ govulncheck -mode=binary ./relay      # built with go1.27.4
No vulnerabilities found.
```

## Environment Details

- ThruBox-Server `main` at `f026f22`
- Linux amd64. Go 1.23.12 and 1.27.4 via `GOTOOLCHAIN`, govulncheck v1.10.0 (vuln DB updated 2026-11-11)
- Affects the published `ghcr.io/aossie-org/thrubox-server` image, which `docker-publish.yml` builds from the Dockerfile

## Impact

Medium - Feature works but has issues

## Proposed fix

- `go.mod`: `go 1.27.4`. It must be at least 1.27.4, or 1.26.11 on the 1.26 line.
- `Dockerfile`: `FROM golang:1.27.4-alpine` and `FROM alpine:3.24`. Pinning both by digest would be better, so Dependabot proposes explicit bumps.
- Update the prerequisites in `README.md` ("Go 1.22+", twice) and `CONTRIBUTING.md` ("Go 1.23+") to match.
- Possible follow-ups, each in its own issue: add a `govulncheck` step to CI, and keep the stale workflow from closing Dependabot PRs, so this cannot drift unnoticed again.
