# OpenTelemetry Maturity Evaluation: Traefik

## Project overview

- **Project**: Traefik — cloud-native HTTP reverse proxy and load balancer, widely used as a Kubernetes Ingress controller
- **Repository**: https://github.com/traefik/traefik
- **Version evaluated**: v3.7.0
- **Evaluation date**: 2026-05-22
- **Cluster**: otel-eval-traefik
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.5

---

## Summary table

| Dimension | Level | Summary |
|-----------|-------|---------|
| 1. Integration Surface | **Level 2 — OpenTelemetry-Native** | OTLP is the sole tracing backend in v3; all three signals flow natively via OTLP gRPC. Prometheus metrics remain co-equal default; OTLP logs are experimental. Standard `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_SERVICE_NAME` env vars not supported. |
| 2. Semantic Conventions | **Level 1 — OpenTelemetry-Aligned** | Traefik's own spans use fully current OTel HTTP semconv. Dual metric naming paradigms (`http.server.request.duration` vs `traefik_*` with non-OTel keys), fully proprietary log attributes, and cross-signal inconsistency prevent Level 2. |
| 3. Resource Attributes & Configuration | **Level 2 — OpenTelemetry-Native** | `service.name` and `service.version` stable and consistent across all three signals. `OTEL_RESOURCE_ATTRIBUTES` documented. Native Kubernetes attribute auto-detection. `OTEL_SERVICE_NAME` undocumented; split `tracing.serviceName` / `metrics.otlp.serviceName` pattern is non-standard. |
| 4. Trace Modeling & Context Propagation | **Level 2 — OpenTelemetry-Native** | W3C Trace Context extraction and injection confirmed. Coherent SERVER→CLIENT→SERVER→INTERNAL topology for all user traffic. Consistent SERVER kind at entry points. No span events; no architectural trace modeling docs. |
| 5. Multi-Signal Observability | **Level 2 — OpenTelemetry-Native** | All three signals via OTLP to same Collector. 100% of log records carry `traceId`/`spanId`. Six shared OTel semconv attribute keys enable metric→trace pivots. Duplicate Prometheus+OTLP metric pipeline; no cross-signal workflow documentation. |
| 6. Audience & Signal Quality | **Level 1 — OpenTelemetry-Aligned** | Strong SLO-relevant metrics. Span naming is operator-unfriendly (`GET` bare method, `ReverseProxy` component name). Critical log severity mismatch (`info` envelope vs `panic` body). Mixed OTel semconv generations within single traces. |
| 7. Stability & Change Management | **Level 2 — OpenTelemetry-Native** | Comprehensive metrics reference docs; dedicated migration guide sections for every telemetry-breaking change. Official Grafana dashboards maintained. No formal change review process; no dual-emission deprecation windows; trace/log scope versions `vunknown`. |

**Overall profile**: Traefik v3.7.0 is a solidly Level 2 project for integration surface, resource attributes, trace modeling, multi-signal observability, and change management. It is held back to Level 1 on semantic conventions and signal quality by dual metric naming paradigms, proprietary log attributes, weak trace span naming, and a log severity bug. The project is on a clear OTel-native trajectory.

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC push, 77 JSONL lines (311 spans, 142 distinct traces). Instrumentation scope: `github.com/traefik/traefik`. Backend spans also present from `@opentelemetry/instrumentation-http` and `@opentelemetry/instrumentation-express`.
- **Metrics**: Flowing — dual pipeline: OTLP gRPC push (96 JSONL lines, 2,742 metric series; scope `github.com/traefik/traefik`) **and** Prometheus scrape via OTel Collector `prometheusreceiver`. 75+ unique metric names.
- **Logs**: Flowing — OTLP gRPC push (4 JSONL lines, 72 log records / 24 access-log records). Requires `--experimental.otlpLogs=true` feature gate. Scope: `traefik`.

### Resource attributes (native, before collector enrichment)

Traefik emits the following resource attributes natively via the OTel Go SDK (v1.43.0):

| Attribute | Value |
|-----------|-------|
| `service.name` | `traefik` |
| `service.version` | `3.7.0` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-7cd47954cc-rrd6j` (pod name) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-7cd47954cc-rrd6j 6.17.0-1013-azure ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |
| `process.command_args` | Full CLI args array |

Traefik also natively auto-detects `k8s.namespace.name`, `k8s.pod.uid`, and `k8s.pod.name` when running in Kubernetes.

### Resource attributes (after collector enrichment)

The OTel Collector `k8sattributes` processor added the following (pipeline-derived, not project-native):

- `k8s.container.name`: `traefik`
- `k8s.deployment.name`: `traefik`
- `k8s.node.name`: `otel-eval-traefik-control-plane`
- `k8s.replicaset.name`: `traefik-7cd47954cc`
- `k8s.pod.start_time`: `2026-05-22T11:25:03Z`
- `k8s.pod.label.app.kubernetes.io/name`: `traefik`
- `k8s.pod.label.app.kubernetes.io/instance`: `traefik-traefik`
- `k8s.pod.label.helm.sh/chart`: `traefik-40.0.0`
- `k8s.pod.annotation.prometheus.io/scrape`: `true`
- `k8s.pod.annotation.prometheus.io/port`: `9100`
- `container.id`: container runtime hash
- `os.version`: `6.17.0-1013-azure`

For Prometheus-scraped metrics, the Collector additionally set `service.instance.id` to the scrape target address (`traefik-metrics.traefik.svc.cluster.local:9100`) — a Collector-derived value, not a pod identity.

---

## Installation context summary

Traefik v3.7.0 was installed via the official Helm chart (`traefik/traefik`, chart version `40.0.0`) into the `traefik` namespace. Getting all three telemetry signals flowing required explicit opt-in beyond the Helm defaults: OTLP tracing and metrics push were enabled via `--tracing.otlp=true` and `--metrics.otlp=true` CLI flags, while OTLP log export required the `--experimental.otlpLogs=true` feature gate (not yet stable). Prometheus metrics are enabled by default and remain active alongside OTLP push, creating a dual-pipeline for metrics. All telemetry configuration uses Traefik-specific CLI flags rather than standard `OTEL_*` environment variables — the deployment carries 40+ `--tracing.*`, `--metrics.*`, and `--accesslog.*` args with no `OTEL_EXPORTER_OTLP_ENDPOINT` or `OTEL_SERVICE_NAME` in the environment. The Helm chart schema enforces strict validation (e.g., `deployment.replicas` not `replicaCount`), and the default `LoadBalancer` service type required port-forwarding for access in the kind cluster. Once configured, all signals flowed reliably to the OTel Collector without adapters or sidecars.

---

## Dimension evaluations

### Dimension 1: Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅ (77 JSONL lines), Metrics ✅ (96 JSONL lines — both OTLP push and Prometheus scrape), Logs ✅ (4 JSONL lines — access logs and general logs via OTLP)
- **Configuration method**: Project-specific CLI flags (`--tracing.otlp.grpc.*`, `--metrics.otlp.*`, `--accesslog.otlp.*`, `--experimental.otlpLogs=true`) — `OTEL_*` standard env vars are **not** the primary configuration surface; only `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are supported as supplementary env vars
- **Documentation stance**: OTLP is the **only** documented tracing backend in v3.x. For metrics, OTLP is listed first and given equal documentation alongside Prometheus, Datadog, InfluxDB2, and StatsD. Logs OTLP export is documented but behind an experimental feature gate.
- **Legacy exporter status**: For tracing — **removed** (Jaeger/Zipkin direct exporters were removed in v3.0; only OTLP remains). For metrics — Prometheus is **co-equal** (enabled by default in the Helm chart). For logs — OTLP is **experimental** (requires `experimental.otlpLogs: true` flag).
- **Signals requiring adapters/sidecars**: None — all signals flow natively via OTLP gRPC to the OTel Collector.

**Detailed observations:**

**Traces (OTLP — fully native):**
- Traefik v3 uses the OTel Go SDK directly (`go.opentelemetry.io/otel`) and emits spans via `otlptracehttp`/`otlptracegrpc` exporters natively.
- Instrumentation scope: `github.com/traefik/traefik` (no version in scope attribute — minor quirk).
- Span types observed: `GET` (server spans, kind=2) and `ReverseProxy` (client spans) — consistent with OTel HTTP semantic conventions v1.26.0.
- W3C Trace Context propagation verified end-to-end: `traceparent` correctly propagated to backend with same trace ID and new child span ID.
- `OTEL_PROPAGATORS` env var is supported for selecting propagators (tracecontext, baggage, b3, jaeger, xray, ottrace).
- Resource attributes set natively: `service.name=traefik`, `service.version=3.7.0`, `telemetry.sdk.name=opentelemetry`, `telemetry.sdk.language=go`, `telemetry.sdk.version=1.43.0`, plus full process/OS attributes.

**Metrics (dual-path — OTLP push + Prometheus scrape):**
- OTLP push is opt-in (`--metrics.otlp=true`); Prometheus is **enabled by default** in the Helm chart.
- Both were active in this evaluation; 17 OTLP-native metric names observed from the `github.com/traefik/traefik` scope, including OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) alongside Traefik-specific names (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`).
- The Prometheus endpoint is exposed on a dedicated port (9100) via a separate ClusterIP service — it is a first-class citizen, not deprecated.
- Helm chart docs explicitly show how to **disable Prometheus** to use OTLP-only: `metrics: { prometheus: null, otlp: { enabled: true } }`.

**Logs (OTLP — experimental):**
- Both general logs and access logs can be exported via OTLP, but this requires `--experimental.otlpLogs=true` feature gate.
- The docs carry an explicit `!!! warning: This is an experimental feature.` notice.
- Access log OTLP export has a known quirk: the log body is a raw JSON string, and `level: panic` appears in the JSON body for normal 200 responses — a serialization bug.
- Log records include rich attributes: `TraceId`, `SpanId`, `trace_id`, `span_id` (duplicated in both camelCase and snake_case), `RouterName`, `ServiceName`, `KubernetesIngressName`, etc.

**Configuration surface analysis:**
- Configuration is entirely via Traefik-native CLI flags / YAML/TOML config — **not** via `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, or `OTEL_SERVICE_NAME`.
- `OTEL_PROPAGATORS` (propagator selection) and `OTEL_RESOURCE_ATTRIBUTES` (additional resource attributes) are the only standard OTel env vars supported.
- No adapters, sidecars, or bridge components are needed — Traefik connects directly to the OTel Collector.

#### Checklist assessment

**Level 2 — OpenTelemetry-Native**

| Criterion | Met? | Evidence |
|-----------|------|----------|
| OTLP is default/clearly-recommended for traces | ✅ Yes | Sole tracing backend in v3.x |
| OTLP is default/clearly-recommended for metrics | ⚠️ Partial | OTLP opt-in; Prometheus is Helm default |
| OTLP is default/clearly-recommended for logs | ⚠️ Partial | Experimental feature gate required |
| `OTEL_EXPORTER_OTLP_ENDPOINT` respected | ❌ No | Not supported |
| `OTEL_SERVICE_NAME` respected | ❌ No | Not supported |
| No adapters/sidecars required | ✅ Yes | Direct OTLP gRPC to Collector |
| Legacy exporters clearly secondary/deprecated | ✅ Traces, ❌ Metrics | Jaeger/Zipkin removed; Prometheus co-equal |

#### Rationale

Traefik v3 is assigned **Level 2 — OpenTelemetry-Native**. The tracing surface is unambiguously OTel-native — no adapters, no sidecars, no bridges, and all legacy backends removed. All three signals flow via OTLP gRPC to a standard OTel Collector. The project substantially exceeds Level 1's "OTLP supported alongside equally-promoted legacy exporters" characterization for tracing. Level 3 is not reached due to: (1) no standard `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_SERVICE_NAME` support; (2) Prometheus metrics remains the Helm default; (3) OTLP logs require an experimental feature gate.

---

### Dimension 2: Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

All 11 deprecated HTTP/network attributes appear exclusively on spans emitted by the **backend service's Node.js auto-instrumentation** (`@opentelemetry/instrumentation-http`), not by Traefik itself. Each deprecated attribute appears exactly **24 times**:

| Deprecated Attribute | Count | Source Scope |
|---|---|---|
| `http.method` | 24 | `@opentelemetry/instrumentation-http` |
| `http.status_code` | 24 | `@opentelemetry/instrumentation-http` |
| `http.url` | 24 | `@opentelemetry/instrumentation-http` |
| `http.target` | 24 | `@opentelemetry/instrumentation-http` |
| `http.host` | 24 | `@opentelemetry/instrumentation-http` |
| `http.scheme` | 24 | `@opentelemetry/instrumentation-http` |
| `http.flavor` | 24 | `@opentelemetry/instrumentation-http` |
| `http.user_agent` | 24 | `@opentelemetry/instrumentation-http` |
| `net.host.name` | 24 | `@opentelemetry/instrumentation-http` |
| `net.host.port` | 24 | `@opentelemetry/instrumentation-http` |
| `net.peer.port` | 24 | `@opentelemetry/instrumentation-http` |

**Traefik's own spans** (`github.com/traefik/traefik` scope) do **not** use these deprecated attributes.

##### Current OTel attributes found on Traefik spans

| Current Attribute | Count | Notes |
|---|---|---|
| `http.request.method` | 98 | ✅ Current semconv |
| `http.response.status_code` | 98 | ✅ Current semconv |
| `url.scheme` | 98 | ✅ Current semconv |
| `server.address` | 98 | ✅ Current semconv |
| `network.protocol.version` | 98 | ✅ Current semconv |
| `network.peer.address` | 98 | ✅ Current semconv |
| `network.peer.port` | 98 | ✅ Current semconv |
| `client.address` | 74 | ✅ Current semconv |
| `url.path` | 74 | ✅ Current semconv |
| `url.query` | 74 | ✅ Current semconv |
| `url.full` | 24 | ✅ Current semconv |
| `user_agent.original` | present | ✅ Current semconv |

Traefik-specific span attribute: `entry_point` (non-OTel, proprietary domain label).

##### Metric names and conventions

**OTel semantic convention metric names (current):**
- `http.server.request.duration` — ✅ OTel semconv histogram with current attribute keys
- `http.client.request.duration` — ✅ OTel semconv histogram with current attribute keys

**Traefik-proprietary metric names (Prometheus-style, non-OTel semconv naming):**
- `traefik_entrypoint_request_duration_seconds`, `traefik_entrypoint_requests_total`, `traefik_entrypoint_requests_bytes_total`, `traefik_entrypoint_responses_bytes_total`
- `traefik_router_request_duration_seconds`, `traefik_router_requests_total`, `traefik_router_requests_bytes_total`, `traefik_router_responses_bytes_total`
- `traefik_service_request_duration_seconds`, `traefik_service_requests_total`, `traefik_service_requests_bytes_total`, `traefik_service_responses_bytes_total`
- `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`

**Traefik-specific metric attribute labels (non-OTel semconv):** `method`, `code`, `protocol`, `entrypoint`, `router`, `service` — bare abbreviations instead of OTel semconv keys.

##### Log attributes

Log records carry Traefik's access log fields as structured attributes — entirely proprietary, non-OTel naming: `ClientAddr`, `ClientHost`, `ClientPort`, `DownstreamStatus`, `Duration`, `RequestMethod`, `RequestPath`, `RouterName`, `ServiceName`, `KubernetesIngressName`, etc. Additionally, duplicate trace context keys (`TraceId`/`SpanId` in PascalCase and `trace_id`/`span_id` in snake_case) appear in the same record.

##### Cross-signal consistency

| Concept | Traces (Traefik scope) | Metrics (OTel) | Metrics (Traefik) | Logs |
|---|---|---|---|---|
| HTTP method | `http.request.method` ✅ | `http.request.method` ✅ | `method` ❌ | `RequestMethod` ❌ |
| HTTP status code | `http.response.status_code` ✅ | `http.response.status_code` ✅ | `code` ❌ | `DownstreamStatus` ❌ |
| URL scheme | `url.scheme` ✅ | `url.scheme` ✅ | `protocol` ❌ | `RequestScheme` ❌ |
| Entry point | `entry_point` (custom) | `entrypoint` (custom) | `entrypoint` | `entryPointName` |

##### Schema URL

Present across all three signals (`https://opentelemetry.io/schemas/1.40.0`), indicating governance intent.

#### Rationale

Traefik is assigned **Level 1 — OpenTelemetry-Aligned**. Traefik's own spans use fully current OTel HTTP semconv attributes and the OTel metrics `http.server.request.duration` / `http.client.request.duration` use correct current keys. Schema URLs are present on all signals. However, Level 2 is not reached due to: (1) dual metric naming paradigms with non-OTel attribute keys in Traefik's Prometheus metrics; (2) fully proprietary log attributes with zero OTel semconv alignment; (3) cross-signal inconsistency for the same concepts; and (4) deprecated attributes from the backend auto-instrumentation visible within the same trace pipeline.

---

### Dimension 3: Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes

Traefik v3.7.0 uses the OTel Go SDK (v1.43.0) and emits a rich set of resource attributes natively via OTLP push for all three signals:

| Attribute | Value observed |
|---|---|
| `service.name` | `traefik` |
| `service.version` | `3.7.0` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-7cd47954cc-rrd6j` (pod name) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |
| `process.command_args` | CLI args array |

Traefik also natively auto-detects `k8s.namespace.name`, `k8s.pod.uid`, and `k8s.pod.name` when running in Kubernetes.

##### service.name and service.version consistency

- **Traces**: `service.name=traefik`, `service.version=3.7.0` ✅
- **Metrics** (OTLP push): `service.name=traefik`, `service.version=3.7.0` ✅
- **Logs**: `service.name=traefik`, `service.version=3.7.0` ✅
- **Consistent**: ✅ Yes — stable and identical across all three signals

##### OTEL_* env var support

- **`OTEL_RESOURCE_ATTRIBUTES`**: Explicitly documented and supported. Official docs state: *"Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes."*
- **`OTEL_SERVICE_NAME`**: Not explicitly documented. Traefik uses `tracing.serviceName` and `metrics.otlp.serviceName` as separate config options (both defaulting to `"traefik"`). The relationship between these and `OTEL_SERVICE_NAME` is undocumented.

##### Identity misplacement

✅ **No misplacement detected.** All identity attributes are correctly placed in resource scope. No `service.*`, `deployment.*`, or `cloud.*` attributes found on spans.

##### service.instance.id

Absent from traces and logs natively. Present in Prometheus-scraped metrics only, set to the scrape target address (`traefik-metrics.traefik.svc.cluster.local:9100`) — a Collector-derived value, not a pod identity.

#### Rationale

Traefik earns **Level 2 — OpenTelemetry-Native** because `service.name` and `service.version` are present, stable, and consistent across all three signals; identity is correctly placed in resource scope; `OTEL_RESOURCE_ATTRIBUTES` is documented; and Kubernetes attributes are partially auto-detected natively. Level 3 is not reached due to: the undocumented `OTEL_SERVICE_NAME` behavior; the split `tracing.serviceName` / `metrics.otlp.serviceName` config pattern (non-standard, could lead to identity divergence); no resource attribute versioning policy; and absent `service.instance.id` from traces/logs.

---

### Dimension 4: Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

- Total spans: 311
- Root spans: 141 | Child spans: 170
- Distinct trace IDs: 142
- Multi-span traces (user traffic): 20 (7–40 spans each)
- Single-span traces: 122 — **all are Kubernetes liveness probes (`/ping`) or Prometheus scrapes (`/metrics`); no user-traffic fragmentation**
- Span kind distribution: INTERNAL=117, SERVER=170, CLIENT=24

Trace topology for a standard user request (8-span trace):
```
ROOT → GET [SERVER, github.com/traefik/traefik, entry_point=web]
         └─ ReverseProxy [CLIENT, github.com/traefik/traefik]
               └─ GET / [SERVER, @opentelemetry/instrumentation-http]
                     ├─ middleware - query      [INTERNAL]
                     ├─ middleware - expressInit [INTERNAL]
                     ├─ middleware - jsonParser  [INTERNAL]
                     ├─ middleware - <anonymous> [INTERNAL]
                     └─ request handler - /     [INTERNAL]
```

The 40-span trace demonstrates fan-out: 5 parallel `GET` (SERVER) spans all share parent `00f067aa0ba902b7`, which is **not present** in the dataset — confirming Traefik correctly extracted and continued an incoming external W3C `traceparent` header.

##### Context propagation

- **Inbound extraction**: `ExtractCarrierIntoContext(req.Context(), req.Header)` called before creating root SERVER span. Confirmed in telemetry: 40-span trace has Traefik SERVER spans parented to external span ID `00f067aa0ba902b7`.
- **Outbound injection**: `InjectContextIntoCarrier(req)` called before forwarding to backends. Confirmed: backend spans appear as children of Traefik `ReverseProxy` CLIENT spans.
- **Propagator**: `autoprop.NewTextMapPropagator()` — defaults to W3C Trace Context + Baggage; `OTEL_PROPAGATORS` env var supported.
- **Parent-based sampling**: Documented — child spans inherit parent sampling decision.

##### Span status

Error spans (`status.code=ERROR`) correctly set on `ReverseProxy` CLIENT spans for 404 responses, with `http.response.status_code=404` attribute.

##### Integration test coverage

`integration/tracing_test.go` includes `TestOpenTelemetryRetry`, `TestOpenTelemetryAuth`, `TestOpenTelemetryRateLimit`, `TestOpenTelemetrySafeURL`, and others that validate span topology assertions.

#### Rationale

Traefik v3.7.0 is assigned **Level 2 — OpenTelemetry-Native** because W3C Trace Context propagation is correct and complete; span kinds are intentional (SERVER at entry points, CLIENT for proxy calls, INTERNAL for middleware); traces represent logical HTTP operations; single-span traces are by design (health/metrics endpoints); and fan-out/retry topology is validated by integration tests. Level 3 is not assigned due to: absent span events; no architectural trace modeling documentation; and no evidence of active trace topology refinement beyond integration test coverage.

---

### Dimension 5: Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability

- Traces: **flowing** — OTLP gRPC push. 239 spans across 88 unique trace IDs.
- Metrics: **flowing** — dual pipeline: OTLP gRPC push (OTel semconv + Traefik-style metrics) + Prometheus scrape. 2,742 metric series total.
- Logs: **flowing** — OTLP gRPC push of access logs (`--experimental.otlpLogs=true`). 72 log records across 4 JSONL lines.

##### Cross-signal correlation

- Log records with `traceId`: **24 of 24 (100%)**
- Log records with `spanId`: **24 of 24 (100%)**
- Matching trace IDs (logs ∩ traces): **20 of 20 unique log trace IDs found in trace data (100%)**
- Shared attribute keys (traces ∩ metrics): `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`

##### Collection model

| Signal | Export mechanism | Scope |
|--------|-----------------|-------|
| Traces | OTLP gRPC push → OTel Collector | `github.com/traefik/traefik` |
| Metrics | OTLP gRPC push **and** Prometheus scrape | `github.com/traefik/traefik` + `prometheusreceiver` |
| Logs | OTLP gRPC push (experimental) | `traefik` |

All three signals export to the same OTel Collector endpoint (`otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317`).

##### Investigative workflow

A user **can** follow the metric → trace → log path:
1. **Metric anomaly** (e.g., spike in `http.server.request.duration` or `traefik_router_requests_total` with `code=500`): filterable by `router`, `service`, `http.request.method`, `http.response.status_code`.
2. **Pivot to trace**: 6 shared OTel semconv attribute keys allow metric-to-trace correlation without manual ID copying.
3. **Pivot to log**: 100% of access log records carry OTLP-native `traceId` and `spanId` — verified against actual spans.

#### Rationale

Traefik meets **Level 2 — OpenTelemetry-Native** cleanly. All three signals are first-class and exported via a unified OTLP pipeline. Correlation quality is excellent: 100% of access log records carry both `traceId` and `spanId`, and all 20 unique trace IDs from logs match verified spans. Six shared attribute keys enable metric-to-trace pivots. Level 3 is not reached because: (1) dual Prometheus+OTLP metric pipeline creates naming inconsistencies without documented rationale; (2) trace sampling is 100% with no cost/cardinality management; (3) no cross-signal investigative workflow documentation exists.

---

### Dimension 6: Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming assessment

**Traefik-native spans (scope: `github.com/traefik/traefik`):**
- **`GET`** (35 occurrences) — Bare HTTP method only, no route template. All requests collapse onto a single span name, making trace-based analysis difficult. An operator cannot distinguish `/ping` health-check traffic from `/` application traffic at the span-name level.
- **`ReverseProxy`** (24 occurrences) — Internal component name. Describes Traefik's mechanism rather than a logical operation.

**Backend spans (scope: `@opentelemetry/instrumentation-http`):**
- `GET /`, `GET /health` — More logical (method + route).
- `middleware - *`, `request handler - /` — Internal Express naming.

##### Log severity mismatch (critical defect)

All 24 log records carry `severityText: "info"` in the OTel envelope, but the embedded structured JSON body contains `"level": "panic"`. This severity mismatch means any OTel-native log routing, filtering, or alerting based on OTel severity will silently misclassify these records.

##### Metric quality

**Traefik-native SLO-relevant metrics (15 metrics):**
- Request rate, latency histograms, byte throughput, open connections, config reload health at entrypoint, router, and service levels.
- `traefik_entrypoint_requests_total`, `traefik_entrypoint_request_duration_seconds`, `traefik_router_requests_total`, `traefik_router_request_duration_seconds`, `traefik_service_requests_total`, `traefik_service_request_duration_seconds`, `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`
- OTel semconv: `http.server.request.duration`, `http.client.request.duration`

**Runtime/infra metrics (lower operator value):** 29 `go_*`, 8 `process_*`, 11 `k8s.*`, 4 `scrape_*` metrics.

##### High-cardinality attribute concerns

- `url.full` in `ReverseProxy` spans contains raw pod IP addresses (`http://10.244.0.6:3000/`) — change on pod restarts
- `network.peer.port`: 73 unique ephemeral port values in spans
- `server.address` in OTel histogram metrics contains raw IPs in some cases

##### No operator dashboards or alert templates documented

No public operator dashboards, alert templates, or sampling/filtering guidance found in official documentation.

#### Rationale

Traefik is awarded **Level 1 — OpenTelemetry-Aligned**. The project has made genuine progress: OTel semantic conventions in its own spans, well-structured access logs with trace correlation, and a rich set of SLO-relevant metrics. However, three issues prevent Level 2: (1) **span naming is not operator-friendly** — `GET` (bare method, no route) and `ReverseProxy` (component name) require Traefik architecture knowledge to interpret; (2) **log severity mismatch is a production defect** — `severityText: "info"` for what Traefik internally labels `panic` breaks OTel-native log routing and alerting; (3) **mixed OTel semconv generations** within a single distributed trace (Traefik uses current conventions, backend uses deprecated) reduces usability.

---

### Dimension 7: Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence

- Traces: `https://opentelemetry.io/schemas/1.40.0` ✅
- Metrics: `https://opentelemetry.io/schemas/1.40.0` (Traefik scope) + `https://opentelemetry.io/schemas/1.18.0` (Collector scopes) ✅
- Logs: `https://opentelemetry.io/schemas/1.40.0` ✅

##### Telemetry documentation

- **Metrics reference**: Full tables of all emitted metric names, types, labels, and descriptions per backend. Explicitly pins OTel semantic conventions to `v1.23.1`. Separate sections for OTel semconv metrics, entrypoint/router/service metrics.
- **Official Grafana dashboards** maintained (IDs 17346 and 17347) for on-premises and Kubernetes deployments — demonstrating commitment to stable metric names.
- **Tracing reference**: Full configuration option table; documents span verbosity levels (`minimal` vs `detailed`).

##### Release note quality for telemetry changes

Dedicated per-version sections in the migration guide (`docs/content/migrate/v3.md`):

| Version | Change |
|---------|--------|
| v3.5.4 | `traefik_tls_certs_not_after_milliseconds` renamed to `traefik_tls_certs_not_after_seconds` |
| v3.5.0 | New `traceVerbosity` option; `k8s.pod.name`/`k8s.pod.uid` injected automatically |
| v3.3.4 | OTel Request Duration metric unit changed from milliseconds to seconds |
| v3.3 | `tracing.globalAttributes` renamed to `tracing.resourceAttributes` |
| v3.0 | Full tracing overhaul — all legacy exporters (Jaeger, Zipkin, Instana, etc.) removed; OTLP-only |

Migration notes explicitly describe behavioral impact (e.g., *"Existing configurations will default to `minimal` unless overridden, which will result in fewer spans being generated than before."*).

##### Instrumentation scope versions

| Signal | Scope | Version |
|--------|-------|---------|
| Traces | `github.com/traefik/traefik` | `vunknown` ⚠️ |
| Metrics | `github.com/traefik/traefik` | `v3.7.0` ✅ |
| Logs | `traefik` | `vunknown` ⚠️ |

Traefik's own metrics scope carries version `v3.7.0`, but trace and log scopes show `vunknown` — a notable inconsistency.

##### Stable vs experimental labeling

No per-signal stability badges on individual metrics or spans. Experimental features are gated via `experimental:` config blocks. OTel semconv version is pinned to `v1.23.1` in docs.

#### Rationale

Traefik scores **Level 2 — OpenTelemetry-Native**. The project treats telemetry as part of its public contract with comprehensive reference documentation, dedicated migration guide sections for every telemetry-breaking change, and explicit user-impact warnings. Level 3 is not reached due to: no formal telemetry change review process (no TEPs/RFCs); no dual-emission deprecation windows for metric renames (changes are immediate); missing trace/log scope versions (`vunknown`); and no automated telemetry regression detection.

---

## Key findings

### Top 3 strengths

1. **OTLP-native tracing with correct end-to-end propagation**: Traefik v3 removed all legacy tracing exporters (Jaeger, Zipkin) and made OTLP the sole backend. W3C Trace Context extraction and injection are implemented correctly — confirmed by a 40-span trace that continues an externally-initiated trace ID across 5 parallel backend calls. The trace topology (SERVER→CLIENT→SERVER→INTERNAL) is coherent, intentional, and validated by integration tests.

2. **Excellent multi-signal correlation**: All three signals flow via OTLP to the same Collector. 100% of access log records carry both `traceId` and `spanId` in the OTLP envelope (verified against actual spans). Six shared OTel semconv attribute keys (`http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`) enable metric→trace pivots without manual bridging. The observability overview documentation presents all three signals together with a unified configuration example.

3. **Proactive, comprehensive change management**: Traefik maintains a dedicated per-version migration guide with named sections for every telemetry-breaking change — metric renames (v3.5.4), unit changes (v3.3.4), config renames (v3.3), span verbosity changes (v3.5.0), and the full tracing overhaul in v3.0. Official Grafana dashboards are maintained and referenced in docs, demonstrating a commitment to stable metric names that users can build on.

### Top 3 areas for improvement

1. **Span naming needs route context**: Traefik's entry-point spans are named `GET` (bare HTTP method) without a route template, collapsing all requests onto a single span name. The `ReverseProxy` span exposes an internal component name rather than a logical operation. Adding the matched route template to span names (e.g., `GET /api/v1/{resource}`) would make traces immediately useful for operators without requiring Traefik architecture knowledge. This is the primary blocker for Dimension 6 Level 2.

2. **Log severity mapping is broken**: All access log records carry `severityText: "info"` in the OTel envelope but `"level": "panic"` in the structured JSON body. This breaks any OTel-native log routing, filtering, or alerting based on severity. Fixing the severity mapping from Traefik's internal log levels to OTel `SeverityNumber` is a relatively contained change with significant impact on production usability.

3. **Unify metric naming and adopt standard OTEL_* env vars**: The dual metric naming paradigm (`http.server.request.duration` with OTel semconv keys vs `traefik_*` with non-OTel abbreviated keys like `method`, `code`, `protocol`) creates cross-signal inconsistency and requires project-specific knowledge to correlate. Aligning the `traefik_*` metric attribute keys with OTel semconv, and supporting `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_SERVICE_NAME` as primary configuration env vars, would bring Traefik to Level 2 on Semantic Conventions and Level 3 on Integration Surface.

### Notable observations

- **Logs OTLP is experimental but functional**: The `--experimental.otlpLogs=true` feature gate is required, but log export works reliably in practice. The known `level: panic` serialization bug in the access log body is a separate issue from the experimental status. As this feature stabilizes, Traefik's multi-signal story will strengthen.

- **`service.instance.id` is absent from OTLP signals**: Traefik does not emit `service.instance.id` natively on traces or logs. It is present in Prometheus-scraped metrics only (set to the scrape target address by the Collector), which is not a meaningful pod identity. In multi-replica deployments, this makes it impossible to distinguish telemetry from individual Traefik pods without relying on pipeline-enriched `k8s.pod.name`.

- **Instrumentation scope version inconsistency**: Traefik's metrics scope carries `v3.7.0` (the project version), but trace and log scopes show `vunknown`. This inconsistency means the instrumentation library version cannot be used to detect telemetry changes for traces/logs from the scope alone — a gap for automated telemetry regression detection.

- **Prometheus metrics is the Helm default**: For new users deploying Traefik via Helm, Prometheus scrape is enabled by default and OTLP metrics push requires explicit opt-in. This means a significant portion of Traefik deployments in production are not using the OTel-native metrics path. The path to OTLP-only is documented (`metrics: { prometheus: null, otlp: { enabled: true } }`) but requires deliberate action.

---

## Methodology notes

- **Evaluation cluster**: kind cluster `otel-eval-traefik`, Kubernetes v1.31.0
- **Traefik version**: v3.7.0 (Helm chart `traefik/traefik` v40.0.0)
- **OTel Collector**: `otel/opentelemetry-collector-contrib` v0.150.1 with `k8sattributes` processor
- **Backend service**: `otel-eval-backend` (Node.js/Express with OTel auto-instrumentation)
- **Telemetry capture**: OTLP gRPC → OTel Collector → file exporter → `/tmp/otel-eval-traefik/*.jsonl`
- **Signal volumes captured**: traces.jsonl (77 lines / 311 spans), metrics.jsonl (96 lines / 2,742 series), logs.jsonl (4 lines / 72 records)
- **Key principle**: Native project telemetry is evaluated separately from Collector-enriched attributes. Pipeline-derived attributes (from `k8sattributes`, Prometheus receiver) are noted as mitigations but do not raise maturity levels.
- **Skill version**: evaluate-otel-maturity v0.0.5
- **Evaluation run**: v1
