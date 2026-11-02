# ntopng on host network with Traefik file route

## Context

The NAS should run [ntopng](https://www.ntop.org/products/traffic-analysis/ntop/) for traffic visibility: hosts, flows, and applications. Stacks in this repo deploy from `/mnt/cache/compose/homelab` with appdata under `/mnt/user/appdata`, and HTTPS UIs go through Traefik on `*.surajram.online` ([0002](0002-compose-from-checkout.md), [0003](0003-traefik-host-ports.md), [0008](0008-dns-to-tailscale-ip.md)).

ntopng needs a packet-visible interface and Redis for users and preferences. The official image starts an in-container Redis that does not persist cleanly on Unraid appdata mounts, so settings and the admin password reset on every recreate unless Redis is external and durable. Host port `3000` is already used by OpenSpeedTest (`speed.surajram.online` → `http://10.0.0.51:3000`).

Capturing on `br0` (or `eth0` / `bond0` if bridging is off) only shows traffic that touches the NAS. Full LAN visibility needs a switch SPAN/mirror into a NIC, or NetFlow/IPFIX/sFlow from the gateway into ntopng. That is a later phase, not a blocker for bringing the UI up.

## Options

1. **Unraid Community Apps template** for `ntop/ntopng` plus a separate Redis app. Fast, but drifts from compose-from-checkout and leaves Traefik routing undocumented in this repo.
2. **Compose stack, bridge network only**, publish ports. Matches other services' Traefik Docker labels, but the container cannot see host interfaces; capture is useless for NAS traffic.
3. **Compose stack, `network_mode: host`**, Community edition, external Redis on a localhost-only published port, Web UI on a free host port, Traefik **file provider** to that port (same pattern as Grafana / Dockhand / OpenSpeedTest). Optional later: NetFlow collector or a mirrored NIC.
4. **Install ntopng on the Unraid host** from packages or a binary under `/boot/config/binaries`. Unraid's root is a RAM disk; packages and modules do not survive reboot the way appdata does, and this fights the compose model.
5. **PF_RING on the host** for high-rate capture. Overkill for this hardware and awkward on Unraid's RAM OS; libpcap via host networking is enough for home traffic involving the NAS.

## Choice

Option 3.

- Image: `ntop/ntopng:latest` pinned by digest (Hub's stable channel; there is no `:stable` tag), Community edition (`--community`), no license mount.
- Capture: host networking, `NET_ADMIN` + `NET_RAW`, interface hardcoded as `br0` in Compose (confirm with `ip -br link` on the NAS before first start; edit `-i` if needed).
- Web UI: bind `127.0.0.1:3009` so the UI is not on the LAN/Tailscale address directly; Traefik on the same host proxies it.
- Redis: `redis:7-alpine` in the same Compose project, AOF on `/mnt/user/appdata/ntopng/redis`, published only as `127.0.0.1:16379:6379` so host-networked ntopng reaches it without exposing Redis on the LAN.
- Data: `/mnt/user/appdata/ntopng/data` → `/var/lib/ntopng`.
- HTTPS: file-provider route `ntop.surajram.online` → `http://127.0.0.1:3009` in `compose_stacks/traefik/dynamic/routes.yml`. Docker labels cannot attach this container to `traefik-net` while it uses host networking.
- Local networks: `10.0.0.0/24` to match the NAS LAN address `10.0.0.51`.

## Consequences

ntopng sees only traffic on the chosen host interface(s). Without SPAN or flow export, that is mostly NAS-sourced and NAS-destined traffic, not every client on the LAN. Adding NetFlow/IPFIX later is compatible: keep this stack, open a collector port (or another `-i` for a mirror NIC), and point the gateway at the NAS.

Redis must stay up for preferences and auth. Publishing Redis only on loopback avoids a LAN-open Redis while still working with host-networked ntopng. If host port `16379` is already taken, change the left-hand publish port and the `-r` argument together.

Default login is `admin` / `admin`; ntopng forces a password change on first visit. Change it immediately. Access remains gated by Tailscale ACLs on ports 80/443 like other Traefik hostnames ([0008](0008-dns-to-tailscale-ip.md)).

Do not use host port `3000`. Do not load PF_RING on Unraid for this stack. Do not deploy until this change is pushed and the NAS has `git pull --ff-only`.

Deploy after appdata exists and the capture interface is confirmed:

```bash
ip -br link
mkdir -p /mnt/user/appdata/ntopng/{data,redis}
git -C /mnt/cache/compose/homelab pull --ff-only
docker compose -f /mnt/cache/compose/homelab/compose_stacks/ntopng/docker-compose.yml up -d
# Traefik watches dynamic/; pull alone updates the ntop route if Traefik is already running
```

UI: `https://ntop.surajram.online` after the Traefik route is live.
