# [BUG]: Docker image sets `RELAY_STORAGE_PATH`, so `storage.path` in a mounted config.yaml is ignored (and the README `docker run` config example mounts no `/data` volume)

## Bug Description

The README says the image reads `/etc/relay/config.yaml` and that you can mount your own file over it (README "4. Docker (Alternative)"). That works for every setting except `storage.path`.

The runtime stage of the `Dockerfile` sets:

```dockerfile
ENV RELAY_STORAGE_PATH=/data/relay.db   # Dockerfile:40
```

`applyEnvOverrides` lets that variable replace the YAML value (`internal/config/config.go:144-146`). Because the image always sets it, a `storage.path` in a mounted config never takes effect. The README says that "an environment variable always wins over the file", but it never says the image sets one. The Configuration table also lists the storage path default as `./data/relay.db` with no Docker note, while the config-path row does have one. The startup log shows the effective `storage_path`, but nothing warns that the file's value was overridden.

The `ENV` line isn't needed for the default setup. The runtime stage has no `WORKDIR`, so the process runs from `/`, and the bundled `storage.path: "./data/relay.db"` (and the built-in default) already resolves to `/data/relay.db`.

A smaller docs issue in the same section: the config-mount example

```bash
docker run -v ./my-config.yaml:/etc/relay/config.yaml:ro ghcr.io/aossie-org/thrubox-server
```

mounts nothing at `/data`. Because of `VOLUME ["/data"]`, the database goes into an anonymous volume. If you recreate the container with the same command (for example after `docker pull`), Docker creates a new, empty volume. The relay then starts with no messages and mailbox `seq` starts again at 1. The old volume is left behind unused. The compose file and the "Step 0" `docker run` example both use a named `relay-data:/data` volume and don't have this problem.

## Steps to Reproduce

1. Create `my-config.yaml`:
   ```yaml
   server:
     port: 3000
     host: "0.0.0.0"
   storage:
     path: "/data/custom.db"
   messages:
     ttl_days: 3
   ```
2. Run the published image as the README shows, with a port added:
   ```bash
   docker run -d --name relay-a -p 127.0.0.1:3000:3000 \
     -v ./my-config.yaml:/etc/relay/config.yaml:ro ghcr.io/aossie-org/thrubox-server:latest
   ```
3. `docker logs relay-a | grep 'configuration loaded'` and `docker exec relay-a ls /data`.
4. POST a few messages, then recreate the container with the same command:
   ```bash
   docker rm -f relay-a
   docker run -d --name relay-b -p 127.0.0.1:3000:3000 \
     -v ./my-config.yaml:/etc/relay/config.yaml:ro ghcr.io/aossie-org/thrubox-server:latest
   curl http://127.0.0.1:3000/api/messages/0xabc
   ```

**Expected:** `ttl_days=3` and `storage_path=/data/custom.db` are both taken from the mounted file. Following the documented `docker run` example keeps the data when the container is recreated.

**Actual:** `ttl_days=3` is applied but `storage_path` stays `/data/relay.db`. After recreating the container the mailbox is empty and the next message gets `seq: 1`.

Clearing the variable shows the `ENV` line isn't needed. `docker run --rm -e RELAY_STORAGE_PATH= ...` uses `/data/custom.db` with the mounted file, and creates `/data/relay.db` with the bundled file.

## Logs and Screenshots

Step 3 (mounted config):
```
level=INFO msg="configuration loaded" config_file=/etc/relay/config.yaml port=3000 storage_driver=sqlite storage_path=/data/relay.db ttl_days=3 ...
level=INFO msg="database initialized" path=/data/relay.db
$ docker exec relay-a ls /data
relay.db  relay.db-shm  relay.db-wal
```

Step 4 (before: three POSTs returned seq 1, 2, 3):
```
$ curl http://127.0.0.1:3000/api/messages/0xabc
[]
$ curl -X POST ... '{"to":"0xabc","from":"0xdef","payload":"m4"}'
{"seq":1,"payload":"m4", ...}
```

With `-e RELAY_STORAGE_PATH=` (variable cleared):
```
bundled config -> storage_path=./data/relay.db   (/data contains relay.db)
mounted config -> storage_path=/data/custom.db   (/data contains custom.db)
```

## Environment Details

- Image `ghcr.io/aossie-org/thrubox-server:latest`, `org.opencontainers.image.revision=f026f228ac9e50cb9bd4bc80b2536a4f7f880cb0` (current `main`). Image env: `RELAY_STORAGE_PATH=/data/relay.db`, `RELAY_CONFIG_PATH=/etc/relay/config.yaml`, WorkingDir `/`, `VOLUME /data`
- Docker on Linux
- The same precedence shows with a local build of `main` (`RELAY_CONFIG_PATH=<file> RELAY_STORAGE_PATH=<other> ./relay-server` uses `<other>`)

## Impact

Low - Minor inconvenience

## Proposed fix

- Remove `ENV RELAY_STORAGE_PATH=/data/relay.db` from the `Dockerfile`, so the image follows `storage.path` like every other setting. To make the `/data` resolution explicit and not depend on the implicit working directory, add `WORKDIR /` to the runtime stage with a comment. Optionally remove the matching `environment:` entry from `docker-compose.yml` and the README compose snippets as well.
- Add a CHANGELOG note: anyone who mounted a config with a different `storage.path` will switch to that file after upgrading, and should move `/data/relay.db` first or set `RELAY_STORAGE_PATH` explicitly.
- If you'd rather keep the `ENV`, document it instead. Add "(`/data/relay.db` in the Docker image)" to the Storage path row, and say next to the mount example that `storage.path` needs `-e RELAY_STORAGE_PATH=...` (or an empty value) to change.
- Add a `/data` volume to the config-mount example, e.g. `docker run -d -p 3000:3000 -v relay-data:/data -v ./my-config.yaml:/etc/relay/config.yaml:ro ghcr.io/aossie-org/thrubox-server`.

## Code of Conduct

- [ ] I have joined the [Discord server](https://discord.gg/hjUhu33uAn) and will post updates there
- [ ] I have searched existing issues to avoid duplicates
