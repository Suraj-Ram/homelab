# One OpenTelemetry collector on a Docker network

## Context

Services on this NAS should be able to send traces, metrics, and logs with OTLP. The host is one Unraid machine (i7-4790, 16 GB, no VMs). Grafana at `grafana.surajram.online` is an Unraid-managed container and does not store telemetry. Glances already shows host CPU, memory, and disks.

`*.surajram.online` resolves to the NAS Tailscale address. A published host port or a Traefik route would be reachable by tailnet peers that can reach the node. Hermes (`nousresearch/hermes-agent:v2026.9.24`) is not on `traefik-net`. Its gateway export is configured in `/mnt/user/appdata/hermes/config.yaml` (`HERMES_HOME=/opt/data`), not by `OTEL_EXPORTER_OTLP_ENDPOINT`. That file is appdata and stays out of git. The image already includes the `otlp` extra. The gateway plane is content-free: health, lifecycle, and bounded diagnostics. It does not export prompts or tool arguments.

## Options

1. One collector on the existing `traefik-net` network, with no published ports. Every container already on that network can send. Hermes has to join `traefik-net`.
2. One collector on a dedicated `otel` network, with no published ports and no Traefik labels. A container sends only after it joins `otel`.
3. One collector with host ports `4317` and `4318`, so processes outside Docker can send too.

For what happens to the data:

1. Accept OTLP and log a batch count with the debug exporter until a store exists.
2. Add Tempo (or another backend) in the same change.

## Choice

Option 2 for the network. Option 1 for the data.

`otel/opentelemetry-collector:0.165.0` runs as `otel-collector` and is attached only to the external Docker network `otel`. Compose does not publish ports. The OTLP receiver listens on `0.0.0.0` inside the container, which is required because the image defaults to localhost and other containers would not be able to connect. From the host, the LAN, and the tailnet there is no listener.

Pipelines for traces, metrics, and logs use `memory_limiter` (256 MiB), then `batch`. The debug exporter verbosity is `basic`, so the log line is a count and not the payload. Docker json logs are capped at 10 MB times 3 files. Unraid on this host does not enforce Compose `deploy.resources` limits, so the memory limiter is the cap.

Hermes joins `otel` and keeps its published host ports. Other Compose services join `otel` the same way, as a second network beside `traefik-net` when they also need Traefik.

## Consequences

Create the network once, then deploy the collector before any service that lists `otel` as external:

```bash
docker network create otel
git -C /mnt/cache/compose/homelab pull --ff-only
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

Metrics and logs are derived from that URL. `hermes monitoring status` inside the container reports whether export is on. Do not set `OTEL_EXPORTER_OTLP_ENDPOINT` on Hermes for this. On this image the gateway does not read it, and a later image can treat that variable as the switch for Relay export, which includes prompt and response text.

Received telemetry is not stored. `docker logs otel-collector` shows batch counts. Replacing the debug exporter is a later choice and should account for whatever a sender actually puts in a span. The health check listens on `otel-collector:13133` on the same network. The image has no shell or curl, so Compose has no HTTP healthcheck.

A second collector waits until another machine sends telemetry. That machine gets an agent; this process becomes the gateway.
