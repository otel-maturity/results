### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability
- Traces: **flowing** — OTLP/gRPC push from Traefik to OTel Collector (`github.com/traefik/traefik` scope, v3.7.1); backend spans also carried via `@opentelemetry/instrumentation-http` and `@opentelemetry/instrumentation-express`. 243 spans across 28 JSONL lines.
- Metrics: **flowing** — dual pipeline: (1) OTLP/gRPC push from Traefik (`github.com/traefik/traefik` scope) producing OTel semantic-convention metrics (`http.server.request.duration`, `http.client.request.duration`) and legacy Prometheus-prefixed metrics (`traefik_router_*`, `traefik_entrypoint_*`, `traefik_service_*`); (2) Prometheus scrape via `prometheusreceiver` for Go runtime metrics. 7,603 metric series across 98 JSONL lines.
- Logs: **flowing** — OTLP/gRPC push from Traefik access-log exporter (`accesslog.otlp.grpc`) with scope name `traefik`; 72 log records (24 with full trace context) across 5 JSONL lines.

##### Cross-signal correlation
- Log records with traceId: **24 of 24 (100%)** — every access-log record carries a non-zero `traceId` in the OTLP log record envelope field, plus redundant `TraceId`, `trace_id`, and `span_id` attributes in the structured body.
- Log records with spanId: **24 of 24 (100%)** — the `spanId` field matches the root entry-point span for the corresponding request.
- Matching trace IDs across `traces.jsonl` and `logs.jsonl`: **20 distinct trace IDs** confirmed to appear in both files, meaning access logs and traces are jointly queryable by `traceId` without any manual bridging.
- Shared attribute keys (traces ∩ metrics): `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme` — 6 keys shared between Traefik spans and OTLP-push metrics.
- Additional shared dimensions (Traefik-specific): `entrypoint` values (`web`, `traefik`, `metrics`) appear in both span `entry_point` attributes and metric `entrypoint` labels; `router` and `service` values appear in legacy Prometheus metrics and in access-log attributes (e.g. `RouterName`, `ServiceName`).

##### Collection model per signal

| Signal | Export mechanism | Receiver in Collector |
|--------|------------------|-----------------------|
| **Traces** | OTLP/gRPC push (`--tracing.otlp.grpc`) at 100% sample rate | `otlp` receiver |
| **Metrics (OTel)** | OTLP/gRPC push (`--metrics.otlp.grpc`) every 10 s | `otlp` receiver |
| **Metrics (Prometheus)** | Prometheus scrape of `/metrics` on port 9100 every 15 s | `prometheus` receiver |
| **Logs** | OTLP/gRPC push (`--accesslog.otlp.grpc`) — access logs sent as OTLP log records | `otlp` receiver |

All three primary Traefik signals (traces, OTel metrics, access logs) share the same OTLP/gRPC pipeline to the same collector endpoint. The Prometheus scrape is an additional, parallel path for Go runtime metrics and legacy label-based metrics.

##### Investigative workflow assessment

A user can follow the **metric → trace → log** path without manual bridging:

1. **Metric anomaly** — `traefik_router_request_duration_seconds{router="demo-otel-eval-backend@kubernetes"}` spikes, or `http.server.request.duration` shows elevated latency for `http.response.status_code=404`.
2. **Pivot to trace** — The `server.address`, `http.request.method`, and `http.response.status_code` attributes are shared between the metric data points and Traefik spans. A tool like Grafana or Jaeger can filter spans by those dimensions to find the slow request.
3. **Inspect correlated log** — The access-log OTLP record for that request carries the identical `traceId` and `spanId` as the root entry-point span. The log body includes rich request context (`RouterName`, `ServiceName`, `DownstreamStatus`, `Duration`, `OriginDuration`) that is not duplicated in the span, making the log genuinely complementary.

The correlation is **automatic** — Traefik injects trace context into access logs natively (introduced in v3.x via `--accesslog.otlp`). No log parsing, regex extraction, or manual timestamp alignment is required.

**Limitation:** The OTel-semantic-convention metrics (`http.server.request.duration`) do **not** carry `router` or `service` labels, only `server.address` and HTTP method/status. The `router`/`service` dimensions are available only in the legacy `traefik_router_*` Prometheus-format metrics. This means metric → trace pivoting using OTel-native metrics requires matching on `server.address` + status code rather than a router name. This is a minor gap but does not break the investigative workflow.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is only one signal treated as first-class (typically metrics only)? | **No** | All three signals are produced natively via OTLP push |
| Are traces or logs absent, experimental, or completely undocumented? | **No** | Traces and OTLP-based access logs are documented and flowing |
| Is there no shared context between any signals? | **No** | 100% of log records carry `traceId`/`spanId`; 20 trace IDs match across signals |
| Do users need to manually correlate timestamps across unrelated tools? | **No** | Trace context is injected automatically into access logs |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all present (all three JSONL files have data)? | **Yes** | 243 spans, 7,603 metric series, 72 log records |
| Do some signals include correlation identifiers while others do not? | **No** | All log records carry `traceId`/`spanId`; metrics do not carry trace IDs (by design) |
| Do logs lack `traceId`/`spanId` even when traces exist? | **No** | 24/24 (100%) of log records carry both `traceId` and `spanId` |
| Are signals produced by different pipelines or configurations (inconsistent)? | **Partial** | Traces and logs use OTLP/gRPC; Prometheus metrics use a separate scrape path (but OTLP metrics also flow) |
| Must users manually bridge signals (e.g. copy a trace ID into a log search)? | **No** | Trace context is natively embedded in OTLP log records |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all first-class (present and documented)? | **Yes** | All three signals documented in Traefik Observe docs; all flowing via OTLP |
| Do log records automatically include `traceId` and `spanId`? | **Yes** | 24/24 records (100%); set by Traefik's native access-log OTLP exporter |
| Do metrics share attribute keys with traces for the same concepts? | **Yes** | 6 shared keys; `entrypoint` values match between metric labels and span `entry_point`; `router`/`service` in legacy Prometheus metrics align with log `RouterName`/`ServiceName` |
| Can users pivot from metric → trace → log without manual correlation? | **Yes** | Shared attributes enable metric→trace pivot; `traceId` in logs enables trace→log pivot |
| Do signals complement rather than duplicate each other? | **Yes** | Metrics = aggregate throughput/latency; traces = per-request path; logs = per-request rich context (router, service, upstream timings) |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume and cardinality managed intentionally across all signals? | **Partial** | Sample rate is configurable; metrics have entrypoint/router/service label toggles; no explicit cardinality budget documentation |
| Are high-cardinality metrics avoided in favor of trace-driven analysis? | **Partial** | OTel-native metrics avoid `router`/`service` in `http.server.request.duration`; but legacy Prometheus metrics still expose high-cardinality `router` labels |
| Are signals shaped for common investigative paths (documented workflows)? | **No** | Traefik docs describe each signal independently; no cross-signal investigative workflow guide exists |
| Is there guidance on when to use which signal? | **No** | The overview page describes each signal's purpose but does not guide users on choosing or combining signals for debugging |
| Is the balance between cost, clarity, and depth explicit? | **No** | No documentation on trade-offs between Prometheus vs OTLP metrics, or when to rely on traces vs access logs |

---

#### Rationale

Traefik v3.7.1 achieves **Level 2 (OpenTelemetry-Native)** because all three signals are first-class OTLP outputs, and crucially, **trace context is automatically injected into access logs** — the strongest cross-signal correlation indicator. Every access-log record carries the same `traceId` and `spanId` as the root entry-point span, confirmed by 20 matching trace IDs observed across `traces.jsonl` and `logs.jsonl`. Metrics share 6 OTel semantic-convention attribute keys with traces (`http.request.method`, `http.response.status_code`, `server.address`, etc.), and the `entrypoint` dimension aligns between metric labels and span attributes. The investigative path metric → trace → log works without manual bridging.

Level 3 is not awarded because: (1) no cross-signal investigative workflow documentation exists — signals are documented individually; (2) the dual metric pipeline (OTLP push + Prometheus scrape) creates redundancy without explicit guidance on when to use each; (3) the OTel-native `http.server.request.duration` metric lacks `router`/`service` dimensions, creating a partial gap in the metric→trace pivot path; and (4) there is no published guidance on signal selection trade-offs or cardinality management strategy.
