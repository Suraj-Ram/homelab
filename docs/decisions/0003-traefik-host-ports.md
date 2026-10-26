# Traefik on host ports

## Context

Unraid's nginx originally listened on ports 80 and 443 of `10.0.0.51` and the Tailscale address. Traefik needs host ports 80 and 443 for normal HTTP and HTTPS URLs.

`surajram.online` is delegated to Cloudflare. At the time of this decision, the apex and wildcard DNS records resolved to `10.0.0.51` and remained DNS-only so access stayed on the LAN. Public DNS later moved to the NAS Tailscale address; see [0008-dns-to-tailscale-ip.md](0008-dns-to-tailscale-ip.md).

## Options

1. Give Traefik its own address on the `br0` macvlan, and leave the Unraid UI on 80 and 443.
2. Move the Unraid UI to 8080 and 8443, publish Traefik on 80 and 443, and issue DNS-01 certificates for routed `surajram.online` hostnames.
3. Run Traefik inside a Tailscale container so it has its own tailnet address.

## Choice

Option 2. In **Settings → Management Access**, the Unraid HTTP port is `8080`, the HTTPS port is `8443`, and Use SSL/TLS remains No. Traefik publishes ports 80 and 443 and redirects HTTP to HTTPS.

`https://unraid.surajram.online` is a file-provider route to `http://10.0.0.51:8080`. The file provider is required because the Unraid WebGUI is a host service rather than a Docker container. The HTTPS service on port 8443 is not proxied; it remains direct recovery and Tailscale access at `https://s-nas.liger-interval.ts.net:8443`.

Each TLS router requests its own certificate through the `my-resolver` ACME resolver. It uses Cloudflare DNS-01 with `CF_DNS_API_TOKEN` from the untracked `.env`; challenge TXT records are temporary and are deleted after validation. Propagation checks use `1.1.1.1:53` and `8.8.8.8:53`. The ACME account email is configured in `traefik.yml`.

File-provider backends continue to use `10.0.0.51`, because Traefik runs on the NAS. Where the public names resolve is defined in [0008-dns-to-tailscale-ip.md](0008-dns-to-tailscale-ip.md).

## Consequences

Unraid does not officially support putting its WebGUI behind a reverse proxy. It ignores `X-Forwarded-Proto` when generating some redirects, so it can redirect an HTTPS request to an HTTP URL. Traefik's entry-point redirect sends that request back to HTTPS, at the cost of an extra round trip. Direct access through port 8443 remains the recovery path. Unraid serves a Let's Encrypt certificate obtained through Tailscale for `s-nas.liger-interval.ts.net`.

The proposed `management-access` IP allowlist remains commented out. Before enabling it, investigate identity-aware access and verify how source addresses are preserved for LAN and Tailscale clients. Do not expose the Unraid route through a router WAN port-forward.

The external Docker network `traefik-net` was created on the host with `docker network create traefik-net`. Traefik and Docker services it proxies must explicitly join that network. At present, only the example web application is routed this way.

The file provider routes the Unraid WebGUI and the Unraid-managed Grafana container through their published host ports. Grafana uses `http://10.0.0.51:3003` as its backend and must set `GF_SERVER_ROOT_URL=http://grafana.surajram.online` in its Unraid template so generated links and redirects use the public URL.

Traefik writes ACME state inside the container to `/var/traefik/certs/cloudflare-acme.json`; the file is mode 600. The current Compose file mounts `./data/certs` there, so the live host file is:

```text
/mnt/cache/compose/homelab/compose_stacks/traefik/data/certs/cloudflare-acme.json
```

This survives reboot because the checkout is on the cache-backed `compose` share, but it conflicts with the repository policy that runtime data belongs under `/mnt/user/appdata`. The Compose mount and existing ACME file still need to be migrated to `/mnt/user/appdata/traefik/certs` in a separate change.

Copy `compose_stacks/traefik/.env.example` to `.env` and set `CF_DNS_API_TOKEN`. Then deploy with:

```bash
docker compose -f /mnt/cache/compose/homelab/compose_stacks/traefik/docker-compose.yml up -d
```
