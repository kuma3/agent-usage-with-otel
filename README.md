# Codex OTel Local Demo

Local OpenTelemetry demo for Codex telemetry. Codex sends OTLP logs and metrics to a local OpenTelemetry Collector, then:

- logs go to Loki and are visible in Grafana Explore
- metrics are exposed by the Collector and scraped by Prometheus
- Grafana is provisioned with Loki and Prometheus datasources

The demo does not edit `~/.codex/config.toml`.

New to OTel/Grafana? Read [docs/explanation.md](docs/explanation.md) for a beginner-friendly walkthrough of what each piece does and what to look at.

## Requirements

- Docker Desktop
- Docker Compose
- Codex CLI with OTel support

This workspace was planned against:

- Docker `28.0.4`
- Docker Compose `v2.34.0-desktop.1`
- Codex CLI `0.124.0-alpha.2`

## Start the Stack

```sh
docker compose up -d
```

Open Grafana:

```text
http://localhost:3000
```

Anonymous admin access is enabled for the local demo. The `Codex OTel Overview` dashboard is provisioned under the `Codex OTel` folder.

## Run Codex With Temporary OTel Settings

The safest path is to pass one-off config overrides and environment variables when starting Codex. This avoids changing your normal Codex config.

```sh
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs \
OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4318/v1/metrics \
OTEL_METRIC_EXPORT_INTERVAL=5000 \
codex \
  -c 'otel.environment="local"' \
  -c 'otel.exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/logs", protocol = "binary" } }' \
  -c 'otel.log_user_prompt=false' \
  -c 'otel.metrics_exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/metrics", protocol = "binary" } }'
```

Run a small prompt, for example:

```text
Say hello, then run `pwd`.
```

Codex batches telemetry asynchronously and flushes on shutdown, so exit the Codex session before checking the final log/metric set.

## Optional Config Snippet

`codex-config.sample.toml` contains the persistent config snippet if you want to copy it into your own Codex config later:

```toml
[otel]
environment = "local"
exporter = { otlp-http = { endpoint = "http://localhost:4318/v1/logs", protocol = "binary" } }
log_user_prompt = false
metrics_exporter = { otlp-http = { endpoint = "http://localhost:4318/v1/metrics", protocol = "binary" } }
```

The sample keeps `log_user_prompt = false`, so raw prompt text is redacted. Only prompt length and related metadata should be exported.

## Inspect Logs

In Grafana, open Explore and select the `Loki` datasource.

Try these LogQL queries:

```logql
{service_name="codex-local-demo"}
```

```logql
{service_name="codex-local-demo"} |= "codex."
```

You can also watch the Collector debug exporter:

```sh
docker compose logs -f otel-collector
```

Representative Codex log event names include `codex.conversation_starts`, `codex.api_request`, `codex.user_prompt`, `codex.tool_decision`, and `codex.tool_result`.

## Inspect Metrics

In Grafana, open Explore and select the `Prometheus` datasource.

Try:

```promql
{service_name="codex-local-demo"}
```

Or open Prometheus directly:

```text
http://localhost:9090
```

The exact metric names depend on your Codex version and which path the run uses. Useful families to look for include:

- `codex_api_request`
- `codex_api_request_duration_ms`
- `codex_tool_call`
- `codex_tool_call_duration_ms`
- `codex_turn_*`

OpenTelemetry metric names containing dots are converted to Prometheus-friendly names with underscores.

## Troubleshooting

Check that all containers are running:

```sh
docker compose ps
```

Validate the Compose file:

```sh
docker compose config
```

You can send a tiny OTLP smoke-test log without starting Codex:

```sh
curl -X POST http://localhost:4318/v1/logs \
  -H 'Content-Type: application/json' \
  -d '{"resourceLogs":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"manual-otel-smoke"}}]},"scopeLogs":[{"logRecords":[{"severityText":"INFO","body":{"stringValue":"codex otel smoke test"},"attributes":[{"key":"event.name","value":{"stringValue":"codex.otel_smoke_test"}}]}]}]}]}'
```

Then query Loki:

```sh
curl --get http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={service_name="codex-local-demo"} |= "codex otel smoke test"' \
  --data-urlencode 'limit=5'
```

You can also send a tiny OTLP metric:

```sh
curl -X POST http://localhost:4318/v1/metrics \
  -H 'Content-Type: application/json' \
  -d '{"resourceMetrics":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"manual-otel-smoke"}}]},"scopeMetrics":[{"metrics":[{"name":"codex.otel_smoke_metric","description":"OTLP smoke-test metric","unit":"1","sum":{"aggregationTemporality":2,"isMonotonic":true,"dataPoints":[{"asInt":"1","attributes":[{"key":"source","value":{"stringValue":"manual"}}]}]}}]}]}]}'
```

Then query Prometheus:

```sh
curl --get http://localhost:9090/api/v1/query \
  --data-urlencode 'query=codex_otel_smoke_metric_total'
```

If logs do not appear:

- Confirm Codex was started with the temporary OTel command above.
- Exit the Codex session so the async exporter can flush.
- Check `docker compose logs otel-collector` for export errors.
- Check `docker compose logs loki` for OTLP ingestion errors.

If metrics do not appear:

- Confirm `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4318/v1/metrics` is set.
- Confirm `otel.metrics_exporter={ otlp-http = { endpoint = "http://localhost:4318/v1/metrics", protocol = "binary" } }` is passed to Codex.
- Check `http://localhost:9464/metrics` for Collector-exported Prometheus metrics.
- Check Prometheus target health at `http://localhost:9090/targets`.
- Metric export behavior can vary by Codex version and config. Logs are the primary signal in this demo; metrics are enabled and wired through when Codex emits them.

## Stop the Stack

```sh
docker compose down
```

To remove local demo data:

```sh
docker compose down -v
```

## References

- [Codex Advanced Configuration: Observability and telemetry](https://developers.openai.com/codex/config-advanced#observability-and-telemetry)
- [Codex Configuration Reference](https://developers.openai.com/codex/config-reference)
