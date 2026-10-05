# Installed on the NAS

Newest first. Prepend an entry when something is installed, updated, or removed on the host. Do not rewrite older entries.

## 2026-10-04 — glances

- Action: installed
- Kind: container
- Persistence: `/mnt/user/compose/glances/docker-compose.yml`. The repo copy is `compose_stacks/glances/docker-compose.yml`. No appdata volume.
- What: Glances 4.5.7 web UI from `nicolargo/glances:4.5.7-full`, host network and host PID. Listening on port 61208. `/api/4/status` returned `{"version": "4.5.7"}` and the container health check is healthy.
- Remove: `docker compose -f /mnt/user/compose/glances/docker-compose.yml down` and delete `/mnt/user/compose/glances`

## 2026-10-04 — docker compose

- Action: installed
- Kind: binary
- Persistence: `/boot/config/binaries/docker-compose`, copied at boot by `/boot/config/go`
- What: Docker Compose v5.6.0 for linux/x86_64, checksum `40343e21ca777173e69cff5dbafeb37c6f81f3b0d57d9e597f036e95eb63e76a`. Installed on 2026-10-04. `docker compose` and `docker-compose` both report v5.6.0. The previous `go` file is `/boot/config/go.bak.20261004201526`.
- Remove: delete `/boot/config/binaries/docker-compose`, restore the previous `/boot/config/go`, and delete `/usr/local/bin/docker-compose` and `/usr/local/lib/docker/cli-plugins/docker-compose`

The live `/boot/config/go` on 2026-10-04 is the stock script, with CRLF line endings and no `&`:

```bash
#!/bin/bash
# Start the Management Utility
/usr/local/sbin/emhttp
```

`/etc/rc.d/rc.local` removes a trailing `&` from that `emhttp` line, then runs `go`. `/usr/local/sbin/emhttp` daemonizes; the running process is `/usr/libexec/unraid/emhttpd`, parented to init, and it stays up for the life of the boot. Leave `emhttp` as the last line, with no `&`, so the management utility still starts and boot is not blocked.

`/boot/config/go` after this install:

```bash
#!/bin/bash
# Start the Management Utility

if [ -d /boot/config/binaries ]; then
  for binary in /boot/config/binaries/*; do
    [ -f "$binary" ] || continue
    name="$(basename "$binary")" || continue
    if cp -f "$binary" "/usr/local/bin/$name"; then
      chmod +x "/usr/local/bin/$name" || echo "go: chmod failed: $name" >&2
    else
      echo "go: copy failed: $name" >&2
    fi
  done
else
  echo "go: /boot/config/binaries is missing; skipping binary restore" >&2
fi

if [ -f /usr/local/bin/docker-compose ]; then
  plugin_dir="/usr/local/lib/docker/cli-plugins"
  if mkdir -p "$plugin_dir" && cp -f /usr/local/bin/docker-compose "$plugin_dir/docker-compose"; then
    chmod +x "$plugin_dir/docker-compose" || echo "go: chmod failed: docker compose plugin" >&2
  else
    echo "go: docker compose plugin install failed" >&2
  fi
fi

# Start the management utility. No trailing &. rc.local strips one,
# and emhttp daemonizes so emhttpd keeps running under init.
/usr/local/sbin/emhttp
```
