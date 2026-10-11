# Traefik on host ports

## Context

Unraid's nginx listens on ports 80 and 443 of `10.0.0.51` and of the Tailscale address. The UI is HTTP because Use SSL/TLS is No. Port 443 is still taken by the Tailscale certificate for `s-nas.liger-interval.ts.net`. `surajram.online` is delegated to Cloudflare. Its apex A record is `10.0.0.51`, DNS only. There is no wildcard record.

## Options

1. Give Traefik its own address on the `br0` macvlan, and leave the Unraid UI on 80 and 443.
2. Move the Unraid UI to 8080 and 8443, publish Traefik on 80 and 443, and issue a DNS-01 certificate for `surajram.online` and `*.surajram.online`.
3. Run Traefik inside a Tailscale container so it has its own tailnet address.

## Choice

Option 2. The UI moves in **Settings → Management Access** before this stack starts: HTTP port `8080`, HTTPS port `8443`, Use SSL/TLS left at No. Traefik then publishes 80 and 443. `https://surajram.online` proxies to `http://10.0.0.51:8080`.

One certificate covers the apex and the wildcard. The Cloudflare token is `CF_DNS_API_TOKEN` in the untracked `.env`. Propagation checks use `1.1.1.1` and `8.8.8.8`.

Add a DNS-only A record `*.surajram.online` to the same address as the apex. Both records can later change from `10.0.0.51` to `100.81.134.74` without a Traefik change. They stay DNS only. Backends in the file provider stay on `10.0.0.51`, because Traefik is on the NAS.

## Consequences

`https://s-nas.liger-interval.ts.net` is served by Unraid on port 8443. Let's Encrypt cannot issue a certificate for that name. `http://10.0.0.51:8080` still reaches the UI without Traefik.

The Docker network `traefik-net` was created on the host with `docker network create traefik-net`. Compose stacks join that existing network. The hermes and hermes-webui stacks join it and label their routers. Their published host ports stay until each name works over HTTPS. Golink, iperf3, and the hermes gateway on container port 8642 are not routed. The gateway binds `127.0.0.1` inside the container.

Grafana, Dockhand, and Glances are reached through the file provider at their current host ports. Grafana may need its root URL set to `https://grafana.surajram.online` in the Unraid template before links stay on that name.

Certificates live in `/mnt/user/appdata/traefik/acme.json`. The dashboard user file is `/mnt/user/appdata/traefik/dashboard-users`, one `htpasswd` line, bcrypt. Create both before the first start. `acme.json` must be mode 600, and an empty file is enough.

```bash
mkdir -p /mnt/user/appdata/traefik
touch /mnt/user/appdata/traefik/acme.json
chmod 600 /mnt/user/appdata/traefik/acme.json
docker run --rm httpd:2.4 htpasswd -nbB admin 'choose-a-password' > /mnt/user/appdata/traefik/dashboard-users
chmod 600 /mnt/user/appdata/traefik/dashboard-users
```

Copy `compose_stacks/traefik/.env.example` to `.env` and set `ACME_EMAIL` and `CF_DNS_API_TOKEN`. Then:

```bash
docker compose -f /mnt/cache/compose/homelab/compose_stacks/traefik/docker-compose.yml up -d
```
