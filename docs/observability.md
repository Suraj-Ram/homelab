# Sending logs and metrics

Internal applications send logs and metrics with OTLP to the collector on this NAS. The collector writes logs to Loki and metrics to Prometheus. It has no traces pipeline yet. The network and the pipelines are recorded in `docs/decisions/0011-otel-collector-docker-network.md`.

## Reach the collector

`otel-collector` publishes no ports. A container on the Docker network `otel-observability-net` reaches it at:

- HTTP: `http://otel-collector:4318`
- gRPC: `otel-collector:4317`

Use HTTP with protobuf. gRPC is the same collector for an SDK that only speaks gRPC.

A process on the Unraid host, the LAN, or the tailnet has no collector address. Put the application on `otel-observability-net`.

```yaml
services:
  app:
    environment:
      TZ: America/New_York
      OTEL_SERVICE_NAME: my-app
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4318
      OTEL_EXPORTER_OTLP_PROTOCOL: http/protobuf
      OTEL_LOGS_EXPORTER: otlp
      OTEL_METRICS_EXPORTER: otlp
      # Remove this when the traces pipeline exists. Until then /v1/traces is 404.
      OTEL_TRACES_EXPORTER: none
    networks:
      - otel-observability-net

networks:
  otel-observability-net:
    external: true
    name: otel-observability-net
```

An application that Traefik already routes keeps `traefik-net` and adds `otel-observability-net`. The collector is not on `traefik-net`.

`OTEL_EXPORTER_OTLP_ENDPOINT` is the base URL. The SDK appends `/v1/logs` and `/v1/metrics`. A base URL that already ends in `/v1/metrics` sends logs to the wrong path.

`OTEL_SERVICE_NAME` is required. It sets the resource attribute `service.name`. Use one stable short name for every replica of that application.

Install the OpenTelemetry SDK for the language. Logs also need that language's log bridge. The environment variables do not attach an existing logger by themselves.

The collector batches for up to 5 seconds (`compose_stacks/otel-collector/config.yaml`) before it writes to Loki or Prometheus.

## Logs

Logs go to Loki (`compose_stacks/loki`). Loki deletes them after 7 days (`retention_period: 168h`).

Grafana reads Loki at `http://10.0.0.51:3100`. Applications send OTLP to the collector. They do not push to that URL.

Query by the service name:

```logql
{service_name="my-app"}
```

`service.name` is stored as the index label `service_name`. `deployment.environment`, when the application sets it, is stored as `deployment_environment`. Those are the resource attributes this Loki indexes. Other fields, including severity and log attributes, are structured metadata. Filter them in the pipeline:

```logql
{service_name="my-app"} | severity_text="ERROR"
```

`severity_text` inside the braces matches nothing. It is not an index label. Loki also sets `detected_level` (`info`, `error`, and so on) from the severity.

A one-shot log, run on the NAS:

```bash
NOW="$(date +%s)000000000"
docker run --rm -i --network otel-observability-net curlimages/curl:8.11.1 \
  -sS -w '\nlogs %{http_code}\n' \
  -X POST http://otel-collector:4318/v1/logs \
  -H 'Content-Type: application/json' \
  --data-binary @- <<JSON
{"resourceLogs":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"my-app"}}]},"scopeLogs":[{"logRecords":[{"timeUnixNano":"$NOW","severityNumber":9,"severityText":"INFO","body":{"stringValue":"hello from my-app"}}]}]}]}
JSON
```

HTTP 200 and `{"partialSuccess":{}}` mean the collector accepted the log. It shows up in Loki after the batch flush:

```logql
{service_name="my-app"}
```

## Metrics

Metrics are remote-written to the Prometheus container at `http://10.0.0.51:9090/api/v1/write`. That container is started with `--web.enable-remote-write-receiver` and currently keeps samples for 15 days. Applications send OTLP to the collector on `/v1/metrics`. They do not remote-write, and Prometheus does not scrape them.

Every series gets these labels:

| Label | Value |
|---|---|
| `job` | `service.name` |
| `service_name` | `service.name` |
| `host_name` | `s-nas`, added by the collector |
| `otel_scope_name` | instrumentation scope name, when the SDK sets one |

Attributes on the data point become labels. A `.` in the key is stored as `_`. Resource attributes other than `service.name` are not copied onto the series, so `deployment.environment` is a Loki label and not a Prometheus label.

The collector translates names on the way into Prometheus:

- A gauge keeps its name.
- A monotonic cumulative sum gains a `_total` suffix. Name the instrument `requests` and query `requests_total`.
- A cumulative histogram is stored as `_bucket`, `_sum`, and `_count`.

Leave the SDK on cumulative temporality. Prometheus rejects a second sample for the same series at the same timestamp, so a hand-built payload uses a new timestamp for each point.

Query in the existing Prometheus datasource:

```promql
{job="my-app"}
```

A one-shot gauge, run on the NAS:

```bash
NOW="$(date +%s)000000000"
docker run --rm -i --network otel-observability-net curlimages/curl:8.11.1 \
  -sS -w '\nmetrics %{http_code}\n' \
  -X POST http://otel-collector:4318/v1/metrics \
  -H 'Content-Type: application/json' \
  --data-binary @- <<JSON
{"resourceMetrics":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"my-app"}}]},"scopeMetrics":[{"metrics":[{"name":"my_app_ready","gauge":{"dataPoints":[{"asDouble":1,"timeUnixNano":"$NOW"}]}}]}]}]}
JSON
```

HTTP 200 and an empty `partialSuccess` mean the collector accepted the sample. After the batch flush, `{job="my-app"}` returns `my_app_ready`.

## Traces

Traces are next. The collector has no traces pipeline, and `POST http://otel-collector:4318/v1/traces` returns 404.

Keep `OTEL_TRACES_EXPORTER=none`. When a trace store is added, set that variable to `otlp`. The endpoint stays `http://otel-collector:4318`.

A log record can already carry `traceId` and `spanId`. Loki stores them with the line. Sending those fields does not require the traces exporter.

## When data does not arrive

The collector's memory limiter is 256 MiB. Above that it refuses the export and the SDK retries. A refused write, a full Loki disk, or Prometheus rejecting a sample is in `docker logs otel-collector`. A 200 from the collector means it accepted the request, not that the store has flushed the batch yet.
