# Static index at the apex

## Context

`surajram.online` and `*.surajram.online` resolve to `10.0.0.51` and stay DNS-only. Traefik publishes 80 and 443 and routes named hostnames: Unraid, Grafana, Dockhand, and Dagu. The apex has no router, so it serves nothing.

Glances is `https://glances.surajram.online`. The Hermes Web UI and the Hermes dashboard answer on host ports and do not have hostnames yet.

## Options

1. Serve a static page from this repo with nginx on `traefik-net`, routed at Host(`surajram.online`).
2. Run Homepage or Homarr and configure the same links in that app.
3. Redirect the apex to `unraid.surajram.online`.

## Choice

Option 1. `compose_stacks/home/` mounts `public/` at the nginx document root and joins `traefik-net`. The router uses the `my-resolver` DNS-01 resolver. The image is `nginx:1.30.5-alpine`. No host port is published.

The page lists Unraid, Grafana, Glances, Dockhand, and Dagu, plus Hermes Web UI (`10.0.0.51:8787`) and the Hermes dashboard (`10.0.0.51:9119`).

## Consequences

Edit the page in this repo. After the owner pushes and the NAS pulls, nginx reads the updated files from the checkout. Recreate the container only when the Compose file changes.

The apex certificate is its own DNS-01 certificate. The page is unreachable when Traefik is down. The host-port links keep working at `10.0.0.51`.

Give Hermes a hostname later by adding a route and changing the matching link in `public/index.html`. Do not publish this route through a router WAN port-forward.

Deploy after the pull:

```bash
git -C /mnt/cache/compose/homelab pull --ff-only
docker compose -f /mnt/cache/compose/homelab/compose_stacks/home/docker-compose.yml up -d
```
