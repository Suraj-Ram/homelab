# Hermes dashboard and API through Traefik

## Context

The Hermes agent container serves a dashboard on 9119 and an OpenAI-compatible gateway API on 8642. The dashboard binds all interfaces inside the image and already requires basic auth. The API server binds `127.0.0.1` unless `API_SERVER_HOST` is set, so a published host port and a proxy on another container both miss it. `*.surajram.online` resolves to the NAS Tailscale address. Traefik routes Compose services on `traefik-net` with Docker labels ([0003](0003-traefik-host-ports.md), [0005](0005-dagu-single-node.md), [0008](0008-dns-to-tailscale-ip.md)).

## Options

1. File-provider routes to published host ports `10.0.0.51:9119` and `10.0.0.51:8642`, leaving Hermes off `traefik-net`.
2. Attach Hermes to `traefik-net`, route both listeners with Docker labels, and publish neither port.

## Choice

Option 2.

- `https://hermes.surajram.online` → container port 9119
- `https://hermesapi.surajram.online` → container port 8642

Both routers use the `my-resolver` DNS-01 resolver. `traefik.docker.network=traefik-net` selects that address because the container also stays on `default` and `otel-observability-net`.

`HERMES_DASHBOARD_PUBLIC_URL` is `https://hermes.surajram.online`. The API server is enabled with `API_SERVER_HOST=0.0.0.0`. `API_SERVER_KEY` comes from the untracked `.env`.

## Consequences

Nothing listens on host ports 9119 or 8642. `http://10.0.0.51:9119` stops answering after the container is recreated. Traefik learns the routers from the Docker provider when that container is recreated; the file provider has no Hermes route.

Set `API_SERVER_KEY` in `compose_stacks/hermes-agent/.env` before that recreate. An empty key leaves the API server down, and the hostname returns a gateway error. Clients send it as `Authorization: Bearer`. The API can run the agent's tools, including the terminal. There is no browser CORS allowlist.

The names are covered by the existing wildcard DNS record and are not proxied by Cloudflare. Tailscale ACLs on ports 80 and 443 remain the remote gate. Do not publish either route through a router WAN port-forward.

`dashboard.trusted_proxies` is not set. It lives in appdata `config.yaml`, and Traefik's address on `traefik-net` is not a fixed IP. Basic-auth cookies still work over HTTPS. Add a bounded proxy entry later if those cookies must be `Secure` or if an OAuth provider is added.
