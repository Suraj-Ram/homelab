# Dagu as a single node

## Context

The NAS needs a scheduler for YAML workflows: cron, retries, logs, and a Web UI, without a database or message broker. [Dagu](https://github.com/dagucloud/dagu) v2.19.2 is one binary that can run the Web UI, scheduler, and an optional coordinator together. Community self-hosting does not need a license key. Dagu Console, the separate multi-server monitor, is not part of this stack.

Unraid's WebGUI already listens on host port 8080. Traefik publishes 80 and 443, redirects HTTP to HTTPS, and routes Docker services on the external `traefik-net` network. `*.surajram.online` already resolves to `10.0.0.51`.

## Options

1. One container running `dagu start-all`, with the coordinator left on. Distributed workers would later need mTLS certificates and a reachable coordinator port.
2. One container running `dagu start-all` with the coordinator disabled. Add workers only if a job must run somewhere other than this NAS.
3. The production Compose layout: separate server, scheduler, coordinator, and worker services, plus an OpenTelemetry collector.

For workflow files:

1. Store them in `/mnt/user/appdata/dagu` so the Web UI can save edits. They stay off the NAS git checkout and out of this repo.
2. Store them in this repo and mount the directory read-write on the NAS checkout. A UI save dirties that checkout, and the next `git pull --ff-only` can fail.
3. Store them in this repo and mount the directory read-only. Edits happen on the workstation.

For Docker:

1. Mount `/var/run/docker.sock` so workflows can start containers. That socket can control every container on the host.
2. Leave the socket unmounted. Shell, HTTP, and SSH steps still run. Container steps can be added later.

## Choice

Option 2 for the process, option 3 for workflow files, and option 2 for Docker.

`ghcr.io/dagucloud/dagu:2.19.2` runs `dagu start-all` with `DAGU_COORDINATOR_ENABLED=false`. Builtin auth is on. The first visit to `/setup` creates the admin account, and Dagu stores the signing secret under the data directory. The UI is `https://dagu.surajram.online` through Traefik. Port 8080 is not published on the host.

`compose_stacks/dagu/dags/` is the workflow directory, mounted read-only. Runtime state uses `/mnt/user/appdata/dagu`, owned by Unraid `nobody:users` (`99:100`). Schedules use `America/New_York`.

## Consequences

The container can run commands, HTTP calls, and SSH from workflow YAML. It cannot start sibling containers until the Docker socket is mounted and `DOCKER_GID` is set to the host Docker group. Do that only for workflows that should control the host Docker daemon.

A read-only DAG mount means the Web UI can run workflows and read logs, and it cannot save YAML. Change a workflow in this repo, then pull on the NAS. Dagu creates a wiki directory at startup under the DAG directory, so `DAGU_WIKI_DIR` points at `/var/lib/dagu/wiki` on the appdata mount.

There is no host port for Dagu. If Traefik is down, the UI is down with it. `docker exec` into the container can still run the `dagu` CLI against `/var/lib/dagu`.

Deploy after the appdata directory exists:

```bash
mkdir -p /mnt/user/appdata/dagu
chown 99:100 /mnt/user/appdata/dagu
git -C /mnt/cache/compose/homelab pull --ff-only
docker compose -f /mnt/cache/compose/homelab/compose_stacks/dagu/docker-compose.yml up -d
```
