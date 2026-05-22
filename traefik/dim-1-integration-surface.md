### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅, Metrics ✅ (OTLP push + Prometheus scrape), Logs ✅ (access + general logs via OTLP gRPC)
- **Configuration method**: Project-specific Helm/YAML flags (`tracing.otlp`, `metrics.otlp`, `logs.access.otlp`); `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` env vars are also respected; `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** documented as supported for endpoint override
- **Documentation stance**: OTLP is the primary and only tracing export path in v3; for metrics, OTLP is listed first/prominently but vendor backends (Datadog, InfluxDB2, Prometheus, StatsD) remain co-equal and fully supported; OTLP log export is behind an experimental feature gate
- **Legacy exporter status**: Tracing — legacy backends (Jaeger, Zipkin) fully removed in v3.0; Metrics — vendor backends (Prometheus, Datadog, InfluxDB2, StatsD) remain active and co-equal, not deprecated; Logs — OTLP is experimental, no legacy log backends
- **Signals requiring adapters/sidecars**: None — all three signals flow natively via OTLP gRPC from Traefik to the OTel Collector without any sidecar or adapter

**Telemetry file observations:**

- **Traces** (28 batches): Scope `github.com/traefik/traefik` (no version tag in scope, version in resource attribute). Spans include `GET` (server, kind=2) and `ReverseProxy` (client, kind=3). Resource attributes include full OTel semantic convention fields: `service.name=traefik`, `service.version=3.7.0`, `telemetry.sdk.name=opentelemetry`, `telemetry.sdk.language=go`, `telemetry.sdk.version=1.43.0`, `host.name`, `os.type`, `os.description`, `process.*`. W3C Trace Context propagation confirmed working.
- **Metrics** (98 batches): Scope `github.com/traefik/traefik v3.7.1` for OTLP push (20 metric series). Includes both OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) and Traefik-native metrics (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`, `traefik_config_*`, `traefik_open_connections`). Prometheus scrape also active (43 series from `prometheusreceiver` scope) — both channels deliver the same metric set.
- **Logs** (5 batches): Scope `traefik` (no version). Access log records flow via OTLP gRPC. Body is a JSON string (not a structured object — a known quirk). Attributes include `trace_id`, `span_id`, `TraceId`, `SpanId` (duplicated in both snake_case and CamelCase), plus rich request/response fields. General logs also flow via the same OTLP channel. Log export requires `experimental.otlpLogs: true` feature gate.

**Configuration observations:**

- All three signals required explicit opt-in via project-specific Helm values (e.g., `tracing.otlp.enabled: true`, `metrics.otlp.enabled: true`, `logs.access.otlp.enabled: true`). There are no defaults that send to OTLP out of the box.
- `OTEL_RESOURCE_ATTRIBUTES` env var is documented and respected for resource attributes. `OTEL_PROPAGATORS` env var is supported (added in v3.0). However, `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** mentioned in docs as a way to configure the export endpoint — the endpoint must be set via `tracing.otlp.grpc.endpoint` / `metrics.otlp.grpc.endpoint` / `logs.access.otlp.grpc.endpoint`.
- No sidecar, adapter, or Collector component was required as a bridge — Traefik pushes directly to the Collector's OTLP gRPC receiver.
- For metrics, Prometheus scrape was also configured alongside OTLP push, showing that both channels are simultaneously active and documented as equivalent options.

**Documentation stance (from official docs):**

- **Tracing**: The tracing docs page is titled "Traefik uses OpenTelemetry, an open standard designed for distributed tracing." OTLP (HTTP and gRPC) is the only export path documented. No legacy backends exist in v3.
- **Metrics**: The metrics page states "Traefik provides metrics in the OpenTelemetry format **as well as** the following vendor specific backends: Datadog, InfluxDB2, Prometheus, StatsD." OpenTelemetry is listed first, but vendor backends are co-equal with full documentation. The default OTLP protocol is HTTP to `https://localhost:4318/v1/metrics`.
- **Logs**: OTLP log export is documented under the `experimental.otlpLogs` feature gate. The access logs OTLP configuration is a first-class option once the gate is enabled.
- **Legacy removal**: The v3.0.0 release removed Jaeger, Zipkin, and InfluxDB v1 tracing backends entirely (`[tracing,otel] Migrate to opentelemetry`). This is a strong signal of intentional OTel-first design for traces.

#### Checklist assessment

**Level 0 — Instrumented**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is telemetry exported only via tool-specific or legacy exporters (Jaeger only, Prometheus scrape only)? | **NO** | OTLP is the primary tracing path; OTLP push is available for metrics and logs |
| Is OTLP unsupported or available only indirectly via sidecars/adapters? | **NO** | Traefik pushes OTLP natively for all 3 signals |
| Does telemetry configuration rely entirely on project-specific flags? | **PARTIAL** | Project-specific YAML/Helm flags are required, but OTEL_RESOURCE_ATTRIBUTES and OTEL_PROPAGATORS env vars are also respected |
| Do users need to adapt their observability stack to fit the project's model? | **NO** | Standard OTel Collector with OTLP gRPC receiver works out of the box |
| Is OpenTelemetry absent from docs or treated as an afterthought? | **NO** | OTel is central to tracing docs; prominently featured in metrics and logs docs |

→ Project clearly exceeds Level 0.

**Level 1 — OpenTelemetry-Aligned**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP supported alongside equally-promoted legacy exporters (documented side-by-side)? | **PARTIAL** | Tracing: OTLP only (legacy removed). Metrics: OTLP + Datadog/Prometheus/InfluxDB2/StatsD co-equal. Logs: OTLP experimental only. |
| Are there multiple overlapping ways to configure telemetry (project flags AND OTEL_* variables)? | **PARTIAL** | OTEL_RESOURCE_ATTRIBUTES and OTEL_PROPAGATORS are supported; OTEL_EXPORTER_OTLP_ENDPOINT is not — endpoint must use project-specific flags |
| Does OTLP require disabling legacy behavior or enabling experimental flags? | **PARTIAL** | Tracing: no. Metrics: Prometheus is enabled by default; OTLP must be explicitly enabled. Logs: requires `experimental.otlpLogs: true` |
| Is OpenTelemetry integration inconsistent across signals? | **PARTIAL** | Traces: OTLP-only, fully stable. Metrics: OTLP push available but Prometheus scrape is the default. Logs: OTLP is experimental |
| Do users need to read multiple docs pages to get a working OTLP integration? | **YES** | Tracing, metrics, and logs each have separate docs pages; logs require an additional experimental flags page |

→ Project exceeds Level 1 for traces; partially Level 1 for metrics and logs.

**Level 2 — OpenTelemetry-Native**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP the default or clearly-recommended export path in docs? | **PARTIAL** | Tracing: YES — OTLP is the only option. Metrics: OTLP listed first but Prometheus is the default (enabled without config). Logs: OTLP is experimental |
| Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` respected end-to-end? | **PARTIAL** | `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are documented; `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_SERVICE_NAME` are **not** documented as supported — endpoint config requires project-specific flags |
| Can users connect to an existing OTel Collector without adapters or glue code? | **YES** | All 3 signals push directly via OTLP gRPC to the Collector; no adapter needed |
| Are legacy exporters clearly secondary, optional, or deprecated? | **PARTIAL** | Tracing: fully removed (strongest signal). Metrics: Prometheus/Datadog/StatsD/InfluxDB2 remain co-equal and undeprecated. Logs: no legacy path |
| Is telemetry configuration consistent across all signals? | **PARTIAL** | Traces and metrics use similar `otlp.grpc.endpoint` config patterns. Logs require an extra `experimental.otlpLogs: true` gate. Prometheus metrics enabled by default but OTLP metrics require opt-in |

→ Most Level 2 criteria are substantially met for traces; partially met for metrics and logs.

**Level 3 — OpenTelemetry-Optimized**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is the telemetry integration surface documented as a stable contract? | **NO** | OTLP log export is explicitly experimental; no formal stability contract documented |
| Are telemetry integration changes reviewed like API changes? | **NO** | No evidence of formal telemetry API governance |
| Are breaking changes communicated with migration guidance? | **PARTIAL** | v2→v3 migration guide exists; Jaeger/Zipkin removal was documented; but no formal telemetry-specific change policy |
| Does the project explicitly support diverse deployment models? | **YES** | Docs cover Kubernetes, Docker, Docker Swarm, bare-metal; Helm chart is the primary Kubernetes delivery vehicle |
| Are legacy integrations removed or tightly scoped with clear deprecation timelines? | **PARTIAL** | Tracing legacy backends fully removed in v3. Metrics legacy backends (Prometheus, Datadog, StatsD) remain with no deprecation timeline |

→ Level 3 criteria are not substantially met.

#### Rationale

**Level 2 — OpenTelemetry-Native** is the appropriate assignment.

Traefik v3 represents a genuine architectural commitment to OpenTelemetry as the primary observability integration surface. The strongest evidence is the **complete removal of Jaeger and Zipkin** in v3.0 — OTLP is now the *only* tracing export path, not one among many. All three signals (traces, metrics, logs) flow natively via OTLP gRPC without any sidecar, adapter, or Collector bridge component. The OTel Go SDK is embedded directly in the Traefik binary, and resource attributes conform to OTel semantic conventions (`service.name`, `service.version`, `telemetry.sdk.*`, `process.*`, `os.*`).

However, Level 3 is not reached for several reasons:

1. **Metrics configuration inconsistency**: Prometheus is the default metrics backend (enabled without any configuration), while OTLP metrics require explicit opt-in. Vendor backends (Datadog, InfluxDB2, StatsD, Prometheus) remain fully co-equal with no deprecation signal — the docs explicitly frame OTLP as "in addition to" these backends.

2. **OTLP log export is experimental**: The `experimental.otlpLogs: true` feature gate is required for log export, indicating this signal is not yet considered stable.

3. **Incomplete standard env var support**: `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, and `OTEL_SERVICE_NAME` are not documented as supported. Endpoint configuration requires project-specific Helm/YAML flags. Only `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are explicitly supported.

4. **No formal stability contract**: There is no documented commitment to treat the telemetry integration surface as a stable API, and no formal change governance process for telemetry configuration.

The project clearly exceeds Level 1 (no legacy tracing backends at all; no adapters needed; OTel SDK embedded natively) and meets the core Level 2 criteria (OTLP is the primary/recommended path for traces, OTLP is available without adapters for all signals, direct Collector connectivity). The partial gaps in metrics defaults and experimental log export prevent Level 3.
