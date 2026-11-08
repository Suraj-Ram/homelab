# One OpenTelemetry collector on a Docker network

## Context

Services on this NAS should send logs and metrics with OTLP. Traces wait until there is a trace store. The host is one Unraid machine (i7-4790, 16 GB, no VMs). Grafana at `grafana.surajram.online` is an Unraid-managed container. Prometheus is already running for that Grafana. Glances shows host CPU, memory, and disks.

`*.surajram.online` resolves to the NAS Tailscale address. A published collector port or a Traefik route would be reachable by tailnet peers that can reach the node. Hermes (`nousresearch/hermes-agent:v2026.9.24`) is not on `traefik-net`. Its gateway export is configured in `/mnt/user/appdata/hermes/config.yaml` (`HERMES_HOME=/opt/data`), not by `OTEL_EXPORTER_OTLP_ENDPOINT`. That file is appdata and stays out of git. The image already includes the `otlp` extra. The gateway plane is content-free: health, lifecycle, and bounded diagnostics. It does not export prompts or tool arguments.

## Options

1. One collector on the existing `traefik-net` network, with no published ports. Every container already on that network can send. Hermes has to join `traefik-net`.
2. One collector on a dedicated `otel` network, with no published ports and no Traefik labels. A container sends only after it joins `otel`.
3. One collector with host ports `4317` and `4318`, so processes outside Docker can send too.

For the signals:

1. Logs go to Loki. Metrics are remote-written to the existing Prometheus. Traces have no pipeline, so the collector refuses them.
2. Accept traces as well and hold them until a trace store exists.

For Loki:

1. Loki is its own Compose stack on the `otel` network. Grafana reaches its query port on the LAN address.
2. Loki stays in the collector's Compose file.

## Choice

Option 2 for the network. Option 1 for the signals. Option 1 for Loki.

`otel/opentelemetry-collector:0.165.0` runs as `otel-collector` and is attached only to the external Docker network `otel`. Compose does not publish collector ports. The OTLP receiver listens on `0.0.0.0` inside the container, which is required because the image defaults to localhost and other containers would not be able to connect. From the host, the LAN, and the tailnet there is no collector listener.

The logs pipeline is `memory_limiter` (256 MiB), then `batch`, then `otlp_http` to `http://loki:3100/otlp`. The metrics pipeline uses the same processors, then `prometheus_remote_write` to `PROMETHEUS_REMOTE_WRITE_URL`, defaulting to `http://10.0.0.51:9090/api/v1/write`. One remote-write consumer keeps sample order for a stock Prometheus. Prometheus must be started with `--web.enable-remote-write-receiver`. Unraid on this host does not enforce Compose `deploy.resources` limits, so the memory limiter is the collector's cap.

Grafana Loki `3.7.9` is `compose_stacks/loki`. It stores chunks on `/mnt/user/appdata/loki`, deletes them after 7 days, and runs as Unraid `nobody:users` (`99:100`). Anonymous usage reporting is off. Its HTTP port is published as `10.0.0.51:3100` because Grafana is not on `otel`. The collector does not use that published port.

Hermes joins `otel` and keeps its published host ports. Other Compose services join `otel` the same way, as a second network beside `traefik-net` when they also need Traefik.

The operator page for this layout is `compose_stacks/pagehost/pages/otel-collector.md`. Pagehost renders it. The page id is assigned at publish time.

## Consequences

Create the network and the Loki data directory once. Start Loki, then the collector:

```bash
docker network inspect otel >/dev/null 2>&1 || docker network create otel
mkdir -p /mnt/user/appdata/loki
chown 99:100 /mnt/user/appdata/loki
git -C /mnt/cache/compose/homelab pull --ff-only
docker compose -f /mnt/cache/compose/homelab/compose_stacks/loki/docker-compose.yml up -d
docker compose -f /mnt/cache/compose/homelab/compose_stacks/otel-collector/docker-compose.yml up -d
```

Endpoints, from a container on `otel`:

- gRPC: `otel-collector:4317`
- HTTP: `http://otel-collector:4318`

Standard SDK variables:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_SERVICE_NAME=<service>
```

Hermes gateway export is a separate switch in the appdata config. Add this to `/mnt/user/appdata/hermes/config.yaml`, keeping the rest of the file, then recreate the Hermes container after the collector is up:

```yaml
monitoring:
  gateway_health_export:
    enabled: true
  export:
    otlp:
      enabled: true
      endpoint: http://otel-collector:4318/v1/traces
```

Hermes derives its metrics and logs URLs from that traces URL. The collector stores the log records in Loki and the metrics in Prometheus, and refuses the trace export. Hermes treats a refused export as fail-open, so the gateway keeps running. Do not set `OTEL_EXPORTER_OTLP_ENDPOINT` on Hermes for this. On this image the gateway does not read it, and a later image can treat that variable as the switch for Relay export, which includes prompt and response text.

In Grafana, add a Loki datasource at `http://10.0.0.51:3100`. Leave the existing Prometheus datasource as it is. Copy `compose_stacks/otel-collector/.env.example` to `.env` when Prometheus remote write is not on port 9090.

The collector health check listens on `otel-collector:13133` on the `otel` network. Neither the collector image nor the Loki image has a shell or curl, so Compose has no HTTP healthcheck.

A second collector waits until another machine sends telemetry. That machine gets an agent; this process becomes the gateway.
