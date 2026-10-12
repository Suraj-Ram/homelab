# Dockhand through Traefik

## Context

Dockhand is already running as the Unraid Docker Manager container `Dockhand` (`fnsys/dockhand:latest`). The template publishes host port 3001 to container port 3000, stores data at `/mnt/user/appdata/dockhand`, and mounts the Docker socket read-only. It is on the default bridge network, not `traefik-net`.

`*.surajram.online` already resolves to `10.0.0.51`. Traefik publishes 80 and 443. Unraid-managed containers that stay on a host port, such as Grafana, are routed by the file provider. Compose services on `traefik-net`, such as Dagu, are routed by Docker labels.

## Options

1. Add a file-provider route for `docker.surajram.online` to `http://10.0.0.51:3001`, and leave the Unraid template in place.
2. Stop the template container and run Dockhand from a Compose file on `traefik-net`, with no host port.

## Choice

Option 1. The UI is `https://docker.surajram.online`. The router uses the `my-resolver` DNS-01 resolver, the same way `grafana.surajram.online` does. The backend keeps the published host port because Unraid Docker Manager owns the container.

## Consequences

Direct access remains `http://10.0.0.51:3001`. If the template's host port changes, the backend URL in `compose_stacks/traefik/dynamic/routes.yml` has to change with it. The file provider watches that directory, so a pull on the NAS updates the route without recreating Traefik.

The container can read the Docker socket. The hostname is covered by the existing wildcard DNS record, which resolves to the LAN address and is not proxied by Cloudflare. Do not publish this route through a router WAN port-forward.

Moving Dockhand into Compose later means removing the Unraid template as well. Otherwise Docker Manager can recreate `Dockhand` on the next boot and the two definitions will fight over the name and the appdata directory.
