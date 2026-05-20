### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

- **Signals flowing via OTLP**: Traces (OTLP gRPC), Metrics (OTLP gRPC push — opt-in; also Prometheus scrape — default-enabled), Logs (OTLP gRPC — experimental feature gate required)
- **Configuration method**: Project-specific flags (`--tracing.otlp.grpc.endpoint`, `--metrics.otlp.grpc.endpoint`, `--accesslog.otlp.grpc.endpoint`, `--experimental.otlpLogs=true`). Standard `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` env vars are respected, but `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** — the collector endpoint must be set via project-specific flags.
- **Documentation stance**: OTLP is listed first and prominently for tracing (sole option in v3); for metrics, OTLP is documented first but co-equal with Datadog, InfluxDB v2, Prometheus, and StatsD as named vendor backends.
- **Legacy exporter status**: For tracing — legacy exporters (Jaeger, Zipkin, Datadog tracing) were removed in v3; OTLP is now the only option. For metrics — Prometheus, Datadog, InfluxDB v2, and StatsD remain fully supported, actively documented, and co-equal alternatives to OTLP. Prometheus is the Helm chart default.
- **Signals requiring adapters/sidecars**: None — all three signals flow natively via OTLP gRPC without sidecars. However, Prometheus metrics are also scraped by the Collector's `prometheusreceiver`, meaning the Prometheus path does not require an adapter either.

**Telemetry file observations:**
- `traces.jsonl` (9 batches): Scope `github.com/traefik/traefik` confirms native OTel Go SDK instrumentation. Span kinds include server (`GET`) and client (`ReverseProxy`) spans with W3C Trace Context propagation.
- `metrics.jsonl` (9 batches): Two distinct scope origins — `github.com/traefik/traefik` (native OTLP push, including `http.server.request.duration` and `http.client.request.duration` OTel semantic convention metrics) and `github.com/open-telemetry/opentelemetry-collector-contrib/receiver/prometheusreceiver` (Prometheus scrape path, carrying `traefik_*` and Go runtime metrics). Both paths are active simultaneously.
- `logs.jsonl` (3 batches): Scope `traefik` (no version), access logs flowing via OTLP gRPC. Log body is a JSON string (not a structured object) — an implementation quirk. The `experimental.otlpLogs: true` feature gate was required to enable this signal.

**Configuration flags observed in the running pod** (`process.command_args`):
```
--metrics.prometheus=true
--metrics.otlp=true
--metrics.otlp.grpc=true
--metrics.otlp.grpc.endpoint=otel-collector-...:4317
--tracing.otlp=true
--tracing.otlp.grpc=true
--tracing.otlp.grpc.endpoint=otel-collector-...:4317
--experimental.otlpLogs=true
--accesslog.otlp=true
--accesslog.otlp.grpc=true
--accesslog.otlp.grpc.endpoint=otel-collector-...:4317
```
All three signals require explicit project-specific flags to direct output to the OTel Collector. There is no "point at a Collector with a single env var" path.

**Source code findings:**
- `resource.WithFromEnv()` is called in both tracing and metrics setup → `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_SERVICE_NAME` are respected for resource attributes.
- `autoprop.NewTextMapPropagator()` is used → `OTEL_PROPAGATORS` env var is respected for propagation format selection.
- `setupHTTPExporter()` and `setupGRPCExporter()` read the endpoint exclusively from Traefik's own config structs — `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** honored.
- OTLP log export is gated behind `experimental.otlpLogs: true` (an `Experimental` struct field), not yet stable.

**Documentation stance:**
- Tracing: OTLP is the sole documented backend in v3 — a strong positive signal.
- Metrics: Docs state "Traefik provides metrics in the OpenTelemetry format **as well as** the following vendor specific backends: Datadog, InfluxDB2, Prometheus, StatsD." OpenTelemetry is listed first but framed as one of five equally-valid options. Prometheus is the Helm chart default (enabled by default; OTLP requires explicit opt-in).
- Logs: `experimental.otlpLogs` is documented as an experimental feature gate, not a stable integration path.
- Only `OTEL_RESOURCE_ATTRIBUTES` is explicitly called out in docs; `OTEL_EXPORTER_OTLP_ENDPOINT` is not mentioned.

#### Checklist assessment

**Level 0 — Instrumented**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is telemetry exported only via tool-specific or legacy exporters (Jaeger only, Prometheus scrape only)? | NO | All three signals flow via native OTLP. Prometheus scrape is an additional parallel path for metrics. |
| Is OTLP unsupported or available only indirectly via sidecars/adapters? | NO | OTLP is natively supported for all three signals. |
| Does telemetry configuration rely entirely on project-specific flags? | PARTIAL | Yes, project-specific flags are required (no `OTEL_EXPORTER_OTLP_ENDPOINT` support), but `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are respected. |
| Do users need to adapt their observability stack to fit the project's model? | NO | Standard OTel Collector pipelines work without adapters. |
| Is OpenTelemetry absent from docs or treated as an afterthought? | NO | OTel is prominently documented for all signals. |

Level 0 does not apply — OTLP is well-supported natively.

**Level 1 — OpenTelemetry-Aligned**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP supported alongside equally-promoted legacy exporters (documented side-by-side)? | YES (metrics) | Metrics docs list OTLP alongside Datadog, InfluxDB v2, Prometheus, StatsD as co-equal options. Prometheus is the Helm chart default. |
| Are there multiple overlapping ways to configure telemetry (project flags AND OTEL_* variables)? | YES | Project-specific flags required for endpoint; `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are also respected, creating a mixed configuration model. |
| Does OTLP require disabling legacy behavior or enabling experimental flags? | YES (logs) | OTLP log export requires `experimental.otlpLogs: true` feature gate. Metrics OTLP requires explicit opt-in while Prometheus remains enabled by default. |
| Is OpenTelemetry integration inconsistent across signals? | YES | Traces: OTLP-only (mature). Metrics: OTLP opt-in alongside 4 other backends, Prometheus is default. Logs: OTLP experimental. Three signals, three different maturity levels. |
| Do users need to read multiple docs pages to get a working OTLP integration? | YES | Separate docs pages for tracing, metrics, and logs/access logs; experimental flag must be discovered separately. |

Level 1 characteristics are substantially met.

**Level 2 — OpenTelemetry-Native**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP the default or clearly-recommended export path in docs? | PARTIAL | Yes for tracing (only option). No for metrics (Prometheus is default in Helm chart; OTLP is opt-in). No for logs (experimental). |
| Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` respected end-to-end? | NO | `OTEL_EXPORTER_OTLP_ENDPOINT` is not supported — endpoint must be set via project-specific flags. Only `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are honored. |
| Can users connect to an existing OTel Collector without adapters or glue code? | YES | OTLP gRPC/HTTP to Collector works natively once configured. |
| Are legacy exporters clearly secondary, optional, or deprecated? | NO | For metrics, Datadog/Prometheus/InfluxDB/StatsD are fully supported, actively documented, and Prometheus is the Helm chart default. |
| Is telemetry configuration consistent across all signals? | NO | Tracing: OTLP-only. Metrics: OTLP opt-in + 4 legacy backends (Prometheus default). Logs: OTLP experimental. Significant inconsistency. |

Level 2 is not met — key criteria fail on metrics defaults, `OTEL_EXPORTER_OTLP_ENDPOINT` absence, and cross-signal inconsistency.

#### Rationale

Traefik v3 represents a meaningful step toward OpenTelemetry alignment — it fully migrated tracing away from legacy exporters (Jaeger, Zipkin, Datadog tracing were removed in v3.0), uses the OTel Go SDK natively, and supports W3C Trace Context propagation. All three signals can flow via OTLP gRPC without sidecars or adapters.

However, the project falls short of Level 2 on several concrete grounds:

1. **`OTEL_EXPORTER_OTLP_ENDPOINT` is not honored**: Users cannot configure the collector endpoint via the standard OTel env var. Each signal requires its own project-specific flag (`--tracing.otlp.grpc.endpoint`, `--metrics.otlp.grpc.endpoint`, `--accesslog.otlp.grpc.endpoint`). This is the most significant gap — a hallmark of Level 2 is that standard env vars "just work."

2. **Metrics are not OTLP-primary**: Prometheus is the Helm chart default for metrics. OTLP metrics require explicit opt-in (`--metrics.otlp=true`). Four vendor-specific backends (Datadog, InfluxDB v2, Prometheus, StatsD) are documented as co-equal alternatives, not as legacy options being phased out.

3. **Log OTLP is experimental**: The `experimental.otlpLogs: true` feature gate is required, signaling that log OTLP is not yet a stable, recommended integration path.

4. **Per-signal inconsistency**: The three signals have different maturity levels — tracing is OTLP-only (strong), metrics is OTLP opt-in alongside legacy backends (moderate), and logs are OTLP experimental (nascent). This inconsistency is a defining characteristic of Level 1.

The project clearly exceeds Level 0 — OTLP is natively supported, OTel SDK is used directly, and no adapters or sidecars are required. But the co-equal legacy metric backends, missing `OTEL_EXPORTER_OTLP_ENDPOINT` support, and experimental log OTLP gate place it firmly at **Level 1 — OpenTelemetry-Aligned**.
