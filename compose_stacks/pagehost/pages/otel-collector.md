## What this is

One OpenTelemetry collector runs on the NAS. It is the only ingest for logs and metrics from Docker services. Grafana reads both stores. The collector does not take traces.

The collector container is `otel-collector`. It joins the Docker network `otel-observability-net` and publishes no ports. From the host, the LAN, and the tailnet there is nothing to connect to. A container on `otel-observability-net` reaches it by name.

| Signal | Path | Grafana |
|---|---|---|
| Logs | collector, then Loki | Loki datasource `http://10.0.0.51:3100` |
| Metrics | collector, then your Prometheus | the Prometheus datasource you already have |
| Traces | refused | none |

Glances still shows host CPU, memory, and disks. This collector is for what services emit.

## Send from a service

Add the `otel-observability-net` network to the service. If Traefik already routes it, keep `traefik-net` as well. Then set:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_SERVICE_NAME=the-service
```

gRPC is `otel-collector:4317` with no URL scheme. HTTP is port 4318.

## Run Loki

Loki is its own Compose stack, `compose_stacks/loki`. The image is `grafana/loki:3.7.9`. It stores chunks on `/mnt/user/appdata/loki` and deletes them after 7 days. The process runs as Unraid `nobody:users` (`99:100`). Anonymous usage reporting is off.

The collector writes to `http://loki:3100/otlp` on `otel-observability-net`. Grafana is not on that network, so the query port is published on the LAN address only: `10.0.0.51:3100`. It is not a Traefik route and it is not bound to the Tailscale address. That published port also accepts log pushes from the LAN. The collector does not use it.

Create the network and the data directory once. Start Loki before the collector.

```bash
docker network inspect otel-observability-net >/dev/null 2>&1 || docker network create otel-observability-net
mkdir -p /mnt/user/appdata/loki
chown 99:100 /mnt/user/appdata/loki
git -C /mnt/cache/compose/homelab pull --ff-only
docker compose -f /mnt/cache/compose/homelab/compose_stacks/loki/docker-compose.yml up -d
```

In Grafana, add a Loki datasource at `http://10.0.0.51:3100`. No auth.

## Run the collector

The stack is `compose_stacks/otel-collector`. The image is `otel/opentelemetry-collector:0.165.0`. Logs go to Loki. Metrics are remote-written to Prometheus. There is no traces pipeline.

```bash
docker compose -f /mnt/cache/compose/homelab/compose_stacks/otel-collector/docker-compose.yml up -d
```

The default Prometheus URL is `http://10.0.0.51:9090/api/v1/write`. To point it somewhere else, copy `compose_stacks/otel-collector/.env.example` to `.env` in that directory and set `PROMETHEUS_REMOTE_WRITE_URL`. `.env` stays untracked.

Prometheus has to be started with `--web.enable-remote-write-receiver`, or it will refuse the push. The collector sends one write at a time so a stock Prometheus does not drop samples as out of order. Each series is labeled `host_name=s-nas`. `service.name` from the sender is kept as a label.

Grafana keeps using the Prometheus datasource you already configured. This setup does not run Prometheus.

## Order

:::callout Start Loki first
Create the `otel-observability-net` network, start Loki, then start the collector. A service can publish only after it has joined `otel-observability-net`.
:::

A second collector waits until another machine sends telemetry. That machine would run an agent, and this collector would become the gateway.
