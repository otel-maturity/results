# OpenTelemetry Support Maturity Evaluation: Traefik

## Project overview

- **Project**: Traefik — cloud-native edge router and Kubernetes Ingress controller (graduated CNCF project)
- **Version evaluated**: v3.6.15 (Helm chart traefik/traefik 39.0.9)
- **Evaluation date**: 2026-05-05
- **Cluster**: otel-eval-traefik
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 3 | Native OTLP gRPC/HTTP for all three signals; traces and metrics GA, logs experimental |
| Semantic Conventions | 2 | Span attributes follow current HTTP semconv; Prometheus-style `traefik_*` metric names coexist with OTel-named metrics; log attributes use custom naming |
| Resource Attributes & Configuration | 2 | Rich native resource attributes (service, process, OS, runtime, SDK); no `OTEL_*` env var support; `service.instance.id` absent |
| Trace Modeling & Context Propagation | 3 | Correct SERVER/CLIENT span hierarchy; W3C Trace Context propagation confirmed end-to-end; complete request story |
| Multi-Signal Observability | 3 | All three signals flow via OTLP; access logs carry `traceId`/`spanId` for cross-signal correlation; consistent `service.name` across all signals |
| Audience & Signal Quality | 2 | Span names are HTTP-method-only (no route template on Traefik spans); high volume of liveness-probe noise; access log body is JSON string, not structured |
| Stability & Change Management | 2 | Schema URL present on traces and metrics; OTLP logs still experimental; no explicit telemetry stability contract documented |

**Overall profile**: Traefik v3 is one of the most OTel-mature proxy/ingress projects in the CNCF ecosystem. It natively pushes all three signals via OTLP without requiring a sidecar or separate agent. The main gaps are metric naming consistency (Prometheus-style names alongside OTel-style names), absence of `OTEL_*` env var support, and the experimental status of OTLP log export.

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC push (native, GA)
- **Metrics**: Flowing — OTLP gRPC push (native, GA) **and** Prometheus `/metrics` scrape
- **Logs**: Flowing — OTLP gRPC push (native, **experimental** — requires `experimental.otlpLogs: true`)

### Resource attributes (native, before collector enrichment)

These attributes are emitted by Traefik itself before the k8sattributes processor enriches the data:

| Attribute | Value (example) |
|-----------|-----------------|
| `service.name` | `traefik` |
| `service.version` | `3.6.15` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-5dc88c47cc-nnktj` |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (...)` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.pid` | `1` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |
| `process.command_args` | full CLI argument array |

Notably absent natively: `service.instance.id`, `os.version` (present only on the backend spans from the k8s-enriched `host.name`).

### Resource attributes (after collector enrichment)

The k8sattributes processor adds:

- `k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.replicaset.name`, `k8s.deployment.name`, `k8s.node.name`, `k8s.container.name`
- `k8s.pod.label.*` (all pod labels), `k8s.pod.annotation.*` (all pod annotations)

---

## Dimension evaluations

### 1. Integration Surface

**Level: 3 — Native OTLP, All Signals**

#### Evidence

- **Traces**: Traefik emits spans via OTLP gRPC to any configured collector endpoint. Configured via `--tracing.otlp.grpc.endpoint`. No Jaeger/Zipkin sidecar required. This is a GA feature since Traefik v3.0.
- **Metrics**: Traefik pushes metrics via OTLP gRPC (`--metrics.otlp.grpc.endpoint`). Also exposes a Prometheus `/metrics` endpoint on a configurable port. Both are active simultaneously. OTLP metrics is GA.
- **Logs**: General operational logs and per-request access logs are both exported via OTLP gRPC (`--log.otlp.grpc.endpoint`, `--accesslog.otlp.grpc.endpoint`). This requires `--experimental.otlpLogs=true` and is still experimental in v3.6.15.
- **Transport**: Both gRPC (`--*.otlp.grpc`) and HTTP (`--*.otlp.http`) are supported for all signals.
- **TLS**: Configurable (`insecure: true/false`), with mTLS support for secure environments.
- **Instrumentation library**: `github.com/traefik/traefik` (no version in scope, but `telemetry.sdk.version: 1.43.0` for the Go OTel SDK).
- **Schema URL**: `https://opentelemetry.io/schemas/1.40.0` on traces; `https://opentelemetry.io/schemas/1.18.0` and `1.40.0` on metrics.

#### Checklist assessment

- ✅ All three signals (traces, metrics, logs) flow via OTLP
- ✅ OTLP gRPC and HTTP both supported
- ✅ Endpoint is configurable (not hardcoded)
- ✅ TLS configurable
- ✅ Traces and metrics are GA; logs are experimental but functional
- ✅ No sidecar/agent required for OTLP export

#### Rationale

Traefik v3 natively pushes all three signals via OTLP without any external agent or sidecar. Both gRPC and HTTP transports are supported. The only caveat is that OTLP log export requires an experimental feature flag, but the functionality is complete and working. This satisfies the highest integration surface level.

---

### 2. Semantic Conventions

**Level: 2 — Partial Alignment with Current Semconv**

#### Evidence

##### Trace attributes

Traefik `GET` (SERVER) span attributes observed:

| Attribute | Status |
|-----------|--------|
| `http.request.method` | ✅ Current semconv (`http.method` deprecated) |
| `http.response.status_code` | ✅ Current semconv (`http.status_code` deprecated) |
| `url.path` | ✅ Current semconv (`http.target` deprecated) |
| `url.query` | ✅ Current semconv |
| `url.scheme` | ✅ Current semconv |
| `network.protocol.version` | ✅ Current semconv |
| `network.peer.address` | ✅ Current semconv |
| `network.peer.port` | ✅ Current semconv |
| `server.address` | ✅ Current semconv |
| `user_agent.original` | ✅ Current semconv |
| `client.address` | ✅ Current semconv |
| `client.port` | ✅ Current semconv |
| `http.request.body.size` | ✅ Current semconv |
| `entry_point` | ⚠️ Traefik-custom attribute (not in OTel semconv) |
| `http.route` | ❌ Absent on Traefik spans (present only on backend spans) |

Traefik `ReverseProxy` (CLIENT) span attributes:

| Attribute | Status |
|-----------|--------|
| `http.request.method` | ✅ Current semconv |
| `http.response.status_code` | ✅ Current semconv |
| `url.full` | ✅ Current semconv |
| `url.scheme` | ✅ Current semconv |
| `network.protocol.version` | ✅ Current semconv |
| `network.peer.address` | ✅ Current semconv |
| `network.peer.port` | ✅ Current semconv |
| `server.address` | ✅ Current semconv |
| `server.port` | ✅ Current semconv |
| `user_agent.original` | ✅ Current semconv |

**No deprecated attributes observed** on Traefik spans. The span attribute set is aligned with the current stable HTTP semantic conventions.

##### Metric names and attributes

Two distinct metric naming schemes coexist in the OTLP output (both from `github.com/traefik/traefik` scope):

**OTel-named metrics** (current semconv):
- `http.server.request.duration` (histogram, unit: `s`) — attributes: `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `url.scheme` ✅
- `http.client.request.duration` (histogram, unit: `s`) — attributes: `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme` ✅

**Prometheus-style metrics** (non-OTel naming):
- `traefik_entrypoint_requests_total`, `traefik_entrypoint_request_duration_seconds`, `traefik_entrypoint_requests_bytes_total`, `traefik_entrypoint_responses_bytes_total`
- `traefik_router_requests_total`, `traefik_router_request_duration_seconds`, `traefik_router_requests_bytes_total`, `traefik_router_responses_bytes_total`
- `traefik_service_requests_total`, `traefik_service_request_duration_seconds`, `traefik_service_requests_bytes_total`, `traefik_service_responses_bytes_total`
- `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`

The `traefik_*` metrics use **non-OTel attribute names**: `code`, `method`, `entrypoint`, `protocol`, `router`, `service` — instead of `http.response.status_code`, `http.request.method`, `server.address`, etc. This is a deliberate design choice to maintain Prometheus compatibility.

##### Log attributes

General logs: plain string body, no structured attributes, `severityNumber` set (9=INFO, 13=WARN), `severityText` absent.

Access logs: body is a JSON-serialized string (not a structured log body). Attributes use PascalCase Traefik-specific names: `ClientHost`, `RequestMethod`, `RouterName`, `ServiceName`, `Duration`, `DownstreamStatus`, `OriginStatus`, `SpanId`, `TraceId`, `entryPointName`, `trace_id`, `span_id`. None of these follow OTel log semantic conventions (e.g., `http.request.method`, `http.response.status_code`, `url.path`).

Notable quirk: access log JSON body contains `"level":"panic"` — this is an artifact of Traefik's internal log framework using "panic" as a field name in access logs, not an actual panic-level event. The OTel `severityText` is correctly set to `"info"` and `severityNumber` to `9`.

#### Checklist assessment

- ✅ Span attributes use current HTTP semconv (no deprecated `http.method`, `http.status_code`, etc.)
- ✅ OTel-named metrics (`http.server.request.duration`, `http.client.request.duration`) are present with correct semconv attributes
- ⚠️ `traefik_*` Prometheus-style metrics use non-OTel attribute names (`code`, `method`, `entrypoint`, `protocol`)
- ⚠️ `http.route` absent on Traefik SERVER spans (present only on backend spans)
- ❌ Log attributes use custom PascalCase naming, not OTel log semconv
- ❌ `severityText` absent on general log records

#### Rationale

Traefik's trace attributes are well-aligned with current stable HTTP semantic conventions — no deprecated attributes observed. The OTel-named metrics (`http.server.request.duration`) are correctly attributed. However, the dominant `traefik_*` metric set uses Prometheus-style attribute names, and log attributes do not follow OTel conventions. Level 2 reflects strong trace semconv compliance with meaningful gaps in metrics and logs.

---

### 3. Resource Attributes & Configuration

**Level: 2 — Rich Native Attributes, No OTEL_* Support**

#### Evidence

##### Native resource attributes

Traefik natively emits a rich set of resource attributes across all three signals:
- **Service identity**: `service.name=traefik`, `service.version=3.6.15`
- **SDK metadata**: `telemetry.sdk.name=opentelemetry`, `telemetry.sdk.language=go`, `telemetry.sdk.version=1.43.0`
- **Host**: `host.name` (pod hostname)
- **OS**: `os.type=linux`, `os.description` (full OS string with kernel version)
- **Process**: `process.executable.name`, `process.executable.path`, `process.owner`, `process.pid`, `process.runtime.name`, `process.runtime.version`, `process.runtime.description`, `process.command_args` (full CLI argument array)

This is an unusually rich native resource attribute set for a proxy — most comparable projects emit only `service.name`.

##### OTEL_* environment variable support

Traefik does **not** support standard `OTEL_*` environment variables. Traefik uses its own configuration system: CLI flags (`--tracing.otlp.grpc.endpoint`) or environment variables with the `TRAEFIK_` prefix (e.g., `TRAEFIK_TRACING_OTLP_GRPC_ENDPOINT`). Verified: no `OTEL_*` env vars are present in the pod, and the Traefik documentation does not mention `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, or other standard OTel SDK environment variables.

This means operators cannot use standard OTel tooling (e.g., operator injection of `OTEL_EXPORTER_OTLP_ENDPOINT`) to configure Traefik's telemetry pipeline.

##### Identity consistency across signals

All three signals emit identical identity attributes:

| Attribute | Traces | Metrics (OTLP) | Logs |
|-----------|--------|----------------|------|
| `service.name` | `traefik` | `traefik` | `traefik` |
| `service.version` | `3.6.15` | `3.6.15` | `3.6.15` |
| `telemetry.sdk.name` | `opentelemetry` | `opentelemetry` | `opentelemetry` |
| `telemetry.sdk.version` | `1.43.0` | `1.43.0` | `1.43.0` |

Identity is perfectly consistent across all signals.

##### Notable absence: `service.instance.id`

`service.instance.id` is not emitted natively. In a multi-replica deployment, this would make it impossible to distinguish telemetry from individual Traefik instances without relying on k8sattributes enrichment (`k8s.pod.name`).

#### Checklist assessment

- ✅ `service.name` and `service.version` set natively on all signals
- ✅ `telemetry.sdk.*` attributes present
- ✅ Rich process and OS resource attributes
- ✅ Consistent identity across traces, metrics, and logs
- ❌ `OTEL_*` env var support absent (uses `TRAEFIK_` prefix instead)
- ❌ `service.instance.id` not emitted

#### Rationale

The native resource attribute set is rich and consistent across all signals, which is above average for this type of project. The absence of `OTEL_*` env var support is a meaningful gap — it breaks the standard OTel operator/SDK configuration contract. Level 2 reflects the strong native attributes with the configuration portability gap.

---

### 4. Trace Modeling & Context Propagation

**Level: 3 — Correct Span Hierarchy, End-to-End Propagation**

#### Evidence

##### Span structure

For each proxied HTTP request, Traefik emits two spans:

1. **`GET` (kind=SERVER, kind=2)** — the inbound request span. Root span when no incoming trace context is present. Attributes: `entry_point`, `http.request.method`, `url.path`, `url.query`, `url.scheme`, `network.protocol.version`, `http.request.body.size`, `user_agent.original`, `server.address`, `network.peer.address`, `network.peer.port`, `client.address`, `client.port`, `http.response.status_code`.

2. **`ReverseProxy` (kind=CLIENT, kind=3)** — the upstream call span. `parentSpanId` = the `GET` span's `spanId`. Attributes: `http.request.method`, `url.full`, `url.scheme`, `network.protocol.version`, `network.peer.address`, `network.peer.port`, `server.address`, `server.port`, `user_agent.original`, `http.response.status_code`.

Span kinds are correct: SERVER for the inbound edge, CLIENT for the upstream proxy call.

##### Context propagation

W3C Trace Context propagation is confirmed end-to-end:
- Traefik creates a root `GET` span (no `parentSpanId`)
- Traefik creates a `ReverseProxy` CLIENT span with `parentSpanId` = `GET` span's `spanId`
- The backend (otel-eval-backend) creates a `GET /` SERVER span with `parentSpanId` = `ReverseProxy` span's `spanId`
- All three spans share the same `traceId`

This confirms Traefik correctly injects W3C `traceparent` headers into upstream requests, enabling full distributed trace continuity across the proxy boundary.

When an incoming request already carries a `traceparent` header, Traefik correctly continues the existing trace (the `GET` SERVER span gets the incoming `parentSpanId`).

##### Trace coherence

The trace tells a complete, coherent story: `[client] → GET (SERVER) → ReverseProxy (CLIENT) → [upstream server span]`. The proxy overhead is measurable as the difference between the SERVER span duration and the CLIENT span duration.

With `tracing.addInternals: true`, internal Traefik middleware spans are also emitted (though none were observed in the collected data for the proxied requests — internal spans appear for dashboard/ping traffic).

##### Scope version

The instrumentation scope is `github.com/traefik/traefik` with no version set (version field is empty). This is a minor gap.

#### Checklist assessment

- ✅ Correct SERVER span for inbound requests
- ✅ Correct CLIENT span for upstream calls
- ✅ Proper parent-child relationship between GET and ReverseProxy spans
- ✅ W3C Trace Context propagation confirmed end-to-end
- ✅ Incoming `traceparent` headers are honored
- ✅ Trace tells a complete and useful story
- ⚠️ Instrumentation scope has no version set
- ⚠️ `http.route` not set on Traefik SERVER spans (only URL path is captured)

#### Rationale

Traefik's trace modeling is exemplary for a proxy. The SERVER/CLIENT span hierarchy is correct, W3C Trace Context propagation works end-to-end, and the trace provides clear visibility into proxy overhead. The absence of `http.route` on Traefik spans (vs. the raw `url.path`) is a minor limitation for cardinality management in high-traffic environments. Level 3 is well-justified.

---

### 5. Multi-Signal Observability

**Level: 3 — All Three Signals, Cross-Signal Correlation**

#### Evidence

##### Signal availability

| Signal | Method | Status | Notes |
|--------|--------|--------|-------|
| Traces | OTLP gRPC push | GA | Per-request spans |
| Metrics | OTLP gRPC push | GA | `traefik_*` + OTel-named metrics |
| Metrics | Prometheus scrape | GA | Same `traefik_*` names |
| General logs | OTLP gRPC push | Experimental | Operational log messages |
| Access logs | OTLP gRPC push | Experimental | Per-request, with trace correlation |

All three signals are first-class, configurable, and functional.

##### Cross-signal correlation

**Logs ↔ Traces**: Access log records have `traceId` and `spanId` set in the OTel log record fields, matching the trace for the same request. For example, access log with `traceId=4601f7871262b14721e96398de2e8ae9` / `spanId=9fde7555ec93046f` matches the `GET` SERVER span with the same IDs.

Additionally, the access log body JSON contains duplicate `TraceId`/`SpanId` (PascalCase) and `trace_id`/`span_id` (snake_case) fields, and the log attributes include both `TraceId` and `trace_id` keys. While redundant, this ensures trace context is accessible regardless of how the log is consumed.

**Metrics ↔ Traces**: Both signals share `service.name=traefik` and `service.version=3.6.15` as resource attributes, enabling correlation by service identity. No explicit trace context in metrics (expected — metrics are aggregates).

**Metrics ↔ Logs**: Consistent `service.name` and `service.version` across all signals.

##### Collection model

Traefik uses a pure **OTLP push** model for traces and logs. For metrics, it supports both OTLP push and Prometheus scrape simultaneously. This dual-path metrics approach provides excellent flexibility for environments that haven't fully migrated to OTLP.

#### Checklist assessment

- ✅ All three signals flowing via OTLP
- ✅ Access logs carry `traceId` and `spanId` (OTel log record fields)
- ✅ Consistent `service.name`/`service.version` across all signals
- ✅ OTLP push for traces and logs; OTLP push + Prometheus scrape for metrics
- ⚠️ General logs do not carry trace context (expected for operational logs)
- ⚠️ OTLP log export is experimental (requires feature flag)

#### Rationale

Traefik provides genuine multi-signal observability with working cross-signal correlation between access logs and traces. The consistent resource identity across signals and the dual metrics export path (OTLP + Prometheus) make this a strong Level 3. The experimental log export flag is a caveat but does not prevent the signals from flowing.

---

### 6. Audience & Signal Quality

**Level: 2 — Useful but with Noise and Naming Gaps**

#### Evidence

##### Span naming

- **`GET`** — Traefik SERVER spans are named after the HTTP method only (e.g., `GET`). This is technically compliant with the OTel HTTP semconv recommendation for server spans where the route template is not available, but it means all GET requests are grouped under a single span name. For an ingress proxy, `http.route` would typically require knowing the matched router rule, which Traefik does not include in the span name or as an `http.route` attribute.
- **`ReverseProxy`** — The upstream CLIENT span is named `ReverseProxy`, which is a Traefik-internal operation name, not an HTTP method or URL template. It is descriptive in context but does not follow the OTel HTTP client span naming convention (`{method}`).
- **Backend spans** (from otel-eval-backend): `GET /`, `GET /health`, `middleware - query`, etc. — these are from the Express instrumentation, not Traefik.

##### Signal-to-noise ratio

The collected traces contain significant **liveness probe noise**: Kubernetes `kube-probe/1.31` health checks to `/ping` on the `traefik` entrypoint generate `GET` SERVER spans at high frequency (the majority of spans in the collected data). These are indistinguishable from real traffic in the trace data without filtering by `user_agent.original`.

Traefik does not provide built-in filtering to suppress health check / internal traffic spans. Operators must configure the collector to filter these out.

The `process.command_args` resource attribute includes the full CLI argument array (40+ arguments), which is verbose and exposes configuration details (including endpoint addresses) in every telemetry record.

##### Default usability

- Traces are immediately useful for understanding request flow through the proxy, proxy overhead, and upstream call details.
- Metrics provide good coverage of entry point, router, and service dimensions.
- The `http.server.request.duration` and `http.client.request.duration` OTel metrics are ready to use with standard dashboards.
- The `traefik_*` Prometheus-style metrics are familiar to existing Traefik users.
- Access logs provide rich per-request detail with trace correlation out of the box.

The main usability concerns are:
1. Span name `GET` provides no route information — high-cardinality dashboards must rely on `url.path` instead.
2. Health probe spans pollute trace data without built-in suppression.
3. Access log body is a JSON string (not a structured body), requiring log parsing in downstream systems.
4. `severityText` is absent on general log records (only `severityNumber` is set).

#### Checklist assessment

- ✅ Span attributes provide actionable information (status code, method, path, peer)
- ✅ Metrics cover the key proxy dimensions (entry point, router, service)
- ✅ OTel-named metrics are immediately usable with standard tooling
- ⚠️ Span name `GET` provides no route cardinality — all GET requests are grouped
- ⚠️ `ReverseProxy` CLIENT span name doesn't follow OTel HTTP client naming
- ⚠️ Health probe spans create noise without built-in filtering
- ⚠️ Access log body is a JSON string, not structured
- ⚠️ `severityText` absent on general log records
- ⚠️ `process.command_args` exposes full configuration in every record

#### Rationale

The telemetry is genuinely useful for operators — the signal content is rich, metrics cover the right dimensions, and trace correlation works. However, the span naming limitations (method-only names, no route template), health probe noise, and log body structure issues reduce out-of-the-box usability. Level 2 reflects good quality with meaningful friction points that require operator-side mitigation.

---

### 7. Stability & Change Management

**Level: 2 — Schema URL Present, Experimental Logs, No Explicit Contract**

#### Evidence

##### Schema URL presence

- **Traces**: `schemaUrl: https://opentelemetry.io/schemas/1.40.0` is set on all Traefik trace exports. ✅
- **Metrics**: `schemaUrl: https://opentelemetry.io/schemas/1.40.0` is set on Traefik OTLP metric exports. ✅ (Prometheus-scraped metrics get `1.18.0` from the collector's conversion.)
- **Logs**: No `schemaUrl` observed on log exports. ❌

##### Documentation of telemetry behavior

Traefik's documentation at [doc.traefik.io](https://doc.traefik.io/traefik/) covers OTLP tracing, metrics, and logs configuration in detail. The configuration reference documents all relevant CLI flags and their Helm values equivalents. However, the documentation does not define the telemetry output as a versioned contract — there is no statement like "these span attributes will not change without a major version bump."

##### Change communication

Traefik maintains a detailed changelog. Breaking changes to configuration are documented in migration guides (e.g., the v3 migration guide covers tracing changes from v2). However, there is no dedicated section for "telemetry breaking changes" — attribute additions or renames would appear in general release notes.

##### Stability guarantees

- Traces and metrics OTLP export: **stable** (GA since v3.0)
- Logs OTLP export: **experimental** (requires `--experimental.otlpLogs=true` in v3.6.15). Experimental features in Traefik can change or be removed without a major version bump.
- The instrumentation scope version is not set (`github.com/traefik/traefik vunknown`), making it impossible to detect instrumentation changes programmatically.

#### Checklist assessment

- ✅ Schema URL set on traces and OTLP metrics
- ✅ OTLP telemetry configuration is well-documented
- ✅ GA signals (traces, metrics) are stable across patch versions
- ⚠️ No explicit telemetry stability contract or versioning commitment
- ⚠️ OTLP logs are experimental — subject to change
- ⚠️ Schema URL absent on log exports
- ⚠️ Instrumentation scope version is not set

#### Rationale

The presence of schema URLs on traces and metrics is a positive signal. The documentation is thorough for configuration. However, the lack of an explicit telemetry stability contract, the experimental status of log export, and the missing scope version prevent a Level 3 assessment. Level 2 reflects good practice with room for formalization.

---

## Key findings

### Strengths

1. **Native OTLP for all three signals**: Traefik v3 is one of very few CNCF projects to natively push traces, metrics, and logs via OTLP gRPC/HTTP without any sidecar, agent, or additional instrumentation layer. The implementation is complete and functional.

2. **Excellent trace modeling with end-to-end context propagation**: The SERVER/CLIENT span hierarchy is correct, W3C Trace Context propagation is confirmed end-to-end (Traefik → upstream), and the trace provides clear visibility into proxy overhead. This is a reference implementation of how a proxy should handle distributed tracing.

3. **Access log ↔ trace correlation**: Access logs carry OTel `traceId` and `spanId` fields, enabling direct correlation between per-request log records and trace spans. Combined with the rich access log attributes (router name, service name, durations, upstream address), this provides powerful debugging capability.

4. **Rich native resource attributes**: Traefik emits `service.name`, `service.version`, `telemetry.sdk.*`, `process.*`, `os.*`, and `host.name` natively — a significantly richer resource attribute set than most comparable proxy/ingress projects.

5. **Current HTTP semantic conventions on spans**: No deprecated attributes (`http.method`, `http.status_code`, `http.target`, `http.url`) were observed. Traefik has fully adopted the current stable HTTP semconv attribute names.

### Areas for improvement

1. **Adopt `OTEL_*` environment variable support**: Traefik uses `TRAEFIK_*` prefixed env vars instead of the standard `OTEL_*` env vars (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, etc.). Adding support for standard OTel SDK env vars would enable operator-injection patterns and improve interoperability with OTel-aware deployment tooling.

2. **Migrate `traefik_*` metric attribute names to OTel semconv**: The `traefik_*` metrics use non-OTel attribute names (`code`, `method`, `entrypoint`, `protocol`) instead of OTel names (`http.response.status_code`, `http.request.method`, `server.address`, `network.protocol.name`). Aligning these would allow standard OTel dashboards and alerting rules to work without remapping. The `http.server.request.duration` and `http.client.request.duration` metrics already demonstrate the correct approach.

3. **Graduate OTLP log export from experimental**: The `experimental.otlpLogs: true` flag requirement signals instability and discourages production adoption of OTLP log export. Graduating this to stable would complete Traefik's OTLP story. Additionally, the access log body should be a structured OTel log body (with attributes following OTel log semconv) rather than a JSON string.

4. **Add `service.instance.id` and set instrumentation scope version**: `service.instance.id` is absent, making multi-replica deployments harder to distinguish without relying on k8s enrichment. The instrumentation scope version should be set to enable programmatic detection of instrumentation changes.

5. **Add `http.route` to Traefik SERVER spans and provide health check filtering**: The matched router rule (e.g., `Host(otel-eval-backend.local)`) or a normalized route template should be added as `http.route` on SERVER spans to enable route-level cardinality in metrics and traces. Built-in filtering for health check / liveness probe traffic would significantly reduce trace noise.

### Notable observations

1. **Dual metric naming is intentional but creates confusion**: Traefik simultaneously emits `traefik_entrypoint_request_duration_seconds` (Prometheus-style) and `http.server.request.duration` (OTel-style) via the same OTLP endpoint. These are not duplicates — the OTel-named metric tracks internal HTTP client/server calls while the `traefik_*` metrics track entry point / router / service dimensions. The coexistence is intentional but can confuse operators expecting a single canonical metric per measurement.

2. **Access log `"level":"panic"` is a false alarm**: The access log JSON body contains `"level":"panic"` — this is an artifact of Traefik's internal log framework using "panic" as a field name in the access log schema, not an actual panic-level event. The OTel `severityText` is correctly `"info"`. This could cause false alerts in log monitoring systems that parse the body.

3. **`process.command_args` exposes full configuration**: The full CLI argument array (including OTLP endpoint addresses) is emitted as a resource attribute in every telemetry record. While useful for debugging, this may be undesirable in security-sensitive environments where endpoint addresses should not appear in telemetry data.

4. **Schema URL on traces references 1.40.0**: This is a recent schema version, indicating Traefik is tracking current OTel semantic convention releases.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector (v0.150.1) with file exporter in a kind cluster (`otel-eval-traefik`)
- The k8sattributes processor was used to enrich telemetry with Kubernetes metadata; native vs. enriched attributes were distinguished by examining non-`k8s.*` resource attributes
- Traefik v3.6.15 was installed via Helm chart `traefik/traefik` v39.0.9 with OTLP gRPC enabled for all signals
- Traffic was generated via `curl` through a port-forwarded Traefik service; requests with and without `traceparent` headers were tested
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv v1.x stable)
- Prometheus metrics were also scraped from the Traefik `/metrics` endpoint to compare naming with OTLP push metrics
- Documentation was reviewed at doc.traefik.io and the traefik/traefik GitHub repository
