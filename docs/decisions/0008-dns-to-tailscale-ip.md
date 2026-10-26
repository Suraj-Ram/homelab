# DNS A records to the Tailscale IP

## Context

Traefik publishes host ports 80 and 443 and routes `*.surajram.online` with Cloudflare DNS-01 certificates, as in [0003-traefik-host-ports.md](0003-traefik-host-ports.md). That decision left apex and wildcard records on `10.0.0.51` (DNS-only) so names were LAN-reachable, and noted they could later point at the NAS Tailscale address.

Remote access should use the same hostnames. The NAS is already on the tailnet as `s-nas.liger-interval.ts.net`. Traefik binds all interfaces, so it already answers on the Tailscale address. Unraid recovery stays on `https://s-nas.liger-interval.ts.net:8443`, not through Traefik.

## Options

1. Keep Cloudflare on `10.0.0.51` and advertise a Tailscale subnet route for `10.0.0.51/32` (or the LAN `/24`) so remote clients reach the LAN address over the tailnet.
2. Point Cloudflare apex and wildcard A records at the NAS Tailscale IPv4, keep DNS-only, and gate access with Tailscale ACLs.
3. Use split DNS (LAN resolves to `10.0.0.51`, tailnet resolves to `100.x`) so home clients stay on the LAN path.

## Choice

Option 2. In Cloudflare, `surajram.online` and `*.surajram.online` A records are the NAS Tailscale IPv4 from `tailscale ip -4` on Unraid. Proxy status stays DNS-only. Do not orange-cloud a Tailscale address.

Tailscale ACL grants are the access control for who may reach that node on ports 80 and 443. App auth on individual services is unchanged. Tailscale Funnel stays off. There is no router WAN port-forward for 80 or 443.

This supersedes the public DNS posture in 0003 (records on `10.0.0.51`). Traefik Compose, file-provider backends (`http://10.0.0.51:…`), and DNS-01 issuance are unchanged: backends are still the host LAN address because Traefik runs on the NAS; ACME does not need the A record to be publicly reachable.

## Consequences

Anyone who resolves the public names learns the NAS Tailscale address. That is accepted. Random internet hosts still cannot route to `100.64.0.0/10`; membership in the tailnet plus ACL remains the gate. Misconfiguration risk is higher than LAN-only DNS: enabling Funnel, or opening ACLs too wide, makes discovery of management UIs trivial via the existing hostnames.

Clients on the LAN use the same names and reach Traefik over Tailscale. If Tailscale is down on the client, those hostnames fail even at home. Direct access by LAN IP (`http://10.0.0.51:…` where a host port is published) is unaffected. Off-tailnet, names resolve but TCP to `100.x` fails.

Subnet routing (option 1) and split DNS (option 3) are rejected for this goal so there is a single public DNS story and no advertised LAN routes. The commented `management-access` IP allowlist in `compose_stacks/traefik/dynamic/routes.yml` stays off; Tailscale ACL is the remote control, not Traefik `ipAllowList`.
