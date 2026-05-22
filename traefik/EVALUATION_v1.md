# OpenTelemetry Maturity Evaluation: Traefik

## Project overview

- **Project**: Traefik — cloud-native HTTP reverse proxy and load balancer; widely used as a Kubernetes Ingress controller
- **Repository**: https://github.com/traefik/traefik
- **Version evaluated**: v3.7.1 (Helm chart `traefik/traefik` version `40.2.0`)
- **Evaluation date**: 2026-05-22
- **Cluster**: otel-eval-traefik (kind cluster)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.5

---

## Summary table

| # | Dimension | Level | Score |
|---|-----------|-------|-------|
| 1 | Integration Surface | OpenTelemetry-Native | **2** |
| 2 | Semantic Conventions | OpenTelemetry-Aligned | **1** |
| 3 | Resource Attributes & Configuration | OpenTelemetry-Native | **2** |
| 4 | Trace Modeling & Context Propagation | OpenTelemetry-Native | **2** |
| 5 | Multi-Signal Observability | OpenTelemetry-Native | **2** |
| 6 | Audience & Signal Quality | OpenTelemetry-Aligned | **1** |
| 7 | Stability & Change Management | OpenTelemetry-Native | **2** |

**Overall profile: Level 2 — OpenTelemetry-Native** (five of seven dimensions at Level 2; two at Level 1)

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP/gRPC push from Traefik to OTel Collector; 243 spans across 28 JSONL batches; instrumentation scope `github.com/traefik/traefik`
- **Metrics**: Flowing — dual pipeline: (1) OTLP/gRPC push (`github.com/traefik/traefik` scope, v3.7.1) every 10 s producing OTel semconv metrics + Traefik-proprietary metrics; (2) Prometheus scrape of port 9100 every 15 s for Go runtime and legacy metrics; 7,603 metric series across 98 JSONL batches
- **Logs**: Flowing — OTLP/gRPC push of access logs via `accesslog.otlp.grpc` (scope `traefik`); 72 log records across 5 JSONL batches; requires `experimental.otlpLogs: true` feature gate

### Resource attributes (native, before collector enrichment)

Traefik v3.7.1 emits the following resource attributes natively via the OTel Go SDK (`resource.New(...)` in `pkg/types/tracing.go`), confirmed identical across traces, metrics, and logs:

| Attribute | Value observed |
|---|---|
| `service.name` | `traefik` |
| `service.version` | `3.7.1` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-6d96b569d5-sthkv` (pod hostname) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-6d96b569d5-sthkv 6.17.0-1013-azure ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.10` |
| `process.runtime.description` | `go version go1.25.10 linux/amd64` |

Additionally, Traefik's built-in Kubernetes detector (`k8sdetector.go`) natively emits `k8s.namespace.name`, `k8s.pod.uid`, and `k8s.pod.name` when running in a Kubernetes cluster.

### Resource attributes (after collector enrichment)

The OTel Collector `k8sattributes` processor added the following (not emitted by Traefik itself), using `k8s.pod.uid` for pod association:

| Attribute | Source |
|---|---|
| `k8s.node.name` | k8sattributes processor |
| `k8s.deployment.name` | k8sattributes processor |
| `k8s.replicaset.name` | k8sattributes processor |
| `k8s.container.name` | k8sattributes processor |
| `k8s.pod.start_time` | k8sattributes processor |
| `k8s.pod.label.*` (all pod labels) | k8sattributes processor |
| `k8s.pod.annotation.*` (all annotations) | k8sattributes processor |
| `service.instance.id` (Prometheus-scraped metrics only) | Prometheus receiver (scrape endpoint URL) |

---

## Installation context summary

Traefik v3.7.1 was installed via the official Helm chart (`traefik/traefik` version 40.2.0) into the `traefik` namespace. All three telemetry signals required explicit opt-in via project-specific Helm values: `tracing.otlp.grpc.enabled: true`, `metrics.otlp.grpc.enabled: true`, and `accesslog.otlp.grpc.enabled: true` — none default to OTLP out of the box. OTLP log export additionally required the `experimental.otlpLogs: true` feature gate, marking it as not yet stable. A parallel Prometheus scrape was also configured alongside OTLP push for metrics, reflecting Traefik's dual-track metrics architecture. The Helm chart enforces strict schema validation (e.g., `deployment.replicas` rather than `replicaCount`), and the default `LoadBalancer` service type required port-forwarding in the kind cluster environment. No sidecars, adapters, or non-standard Collector components were needed — Traefik pushes OTLP/gRPC directly to the standard `otlp` receiver.

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅, Metrics ✅ (OTLP push + Prometheus scrape), Logs ✅ (access + general logs via OTLP gRPC)
- **Configuration method**: Project-specific Helm/YAML flags (`tracing.otlp`, `metrics.otlp`, `logs.access.otlp`); `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` env vars are also respected; `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** documented as supported for endpoint override
- **Documentation stance**: OTLP is the primary and only tracing export path in v3; for metrics, OTLP is listed first/prominently but vendor backends (Datadog, InfluxDB2, Prometheus, StatsD) remain co-equal and fully supported; OTLP log export is behind an experimental feature gate
- **Legacy exporter status**: Tracing — legacy backends (Jaeger, Zipkin) fully removed in v3.0; Metrics — vendor backends (Prometheus, Datadog, InfluxDB2, StatsD) remain active and co-equal, not deprecated; Logs — OTLP is experimental, no legacy log backends
- **Signals requiring adapters/sidecars**: None — all three signals flow natively via OTLP gRPC from Traefik to the OTel Collector without any sidecar or adapter

**Telemetry file observations:**

- **Traces** (28 batches): Scope `github.com/traefik/traefik` (no version tag in scope, version in resource attribute). Spans include `GET` (server, kind=2) and `ReverseProxy` (client, kind=3). Resource attributes include full OTel semantic convention fields: `service.name=traefik`, `service.version=3.7.1`, `telemetry.sdk.name=opentelemetry`, `telemetry.sdk.language=go`, `telemetry.sdk.version=1.43.0`, `host.name`, `os.type`, `os.description`, `process.*`. W3C Trace Context propagation confirmed working.
- **Metrics** (98 batches): Scope `github.com/traefik/traefik v3.7.1` for OTLP push. Includes both OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) and Traefik-native metrics (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`, `traefik_config_*`, `traefik_open_connections`). Prometheus scrape also active — both channels deliver the same metric set.
- **Logs** (5 batches): Scope `traefik` (no version). Access log records flow via OTLP gRPC. Body is a JSON string (not a structured object — a known quirk). Attributes include `trace_id`, `span_id`, `TraceId`, `SpanId` (duplicated in both snake_case and CamelCase), plus rich request/response fields. Log export requires `experimental.otlpLogs: true` feature gate.

**Configuration observations:**

- All three signals required explicit opt-in via project-specific Helm values. There are no defaults that send to OTLP out of the box.
- `OTEL_RESOURCE_ATTRIBUTES` env var is documented and respected. `OTEL_PROPAGATORS` env var is supported (added in v3.0). However, `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** mentioned in docs — the endpoint must be set via `tracing.otlp.grpc.endpoint` / `metrics.otlp.grpc.endpoint` / `logs.access.otlp.grpc.endpoint`.
- No sidecar, adapter, or Collector component was required as a bridge — Traefik pushes directly to the Collector's OTLP gRPC receiver.
- For metrics, Prometheus scrape was also configured alongside OTLP push, showing that both channels are simultaneously active and documented as equivalent options.

**Documentation stance (from official docs):**

- **Tracing**: OTLP (HTTP and gRPC) is the only export path documented. No legacy backends exist in v3.
- **Metrics**: \"Traefik provides metrics in the OpenTelemetry format **as well as** the following vendor specific backends: Datadog, InfluxDB2, Prometheus, StatsD.\" OpenTelemetry is listed first, but vendor backends are co-equal with full documentation.
- **Logs**: OTLP log export is documented under the `experimental.otlpLogs` feature gate.
- **Legacy removal**: The v3.0.0 release removed Jaeger, Zipkin, and InfluxDB v1 tracing backends entirely.

#### Checklist assessment

**Level 2 — OpenTelemetry-Native** is the appropriate assignment.

Traefik v3 represents a genuine architectural commitment to OpenTelemetry as the primary observability integration surface. The strongest evidence is the **complete removal of Jaeger and Zipkin** in v3.0 — OTLP is now the *only* tracing export path. All three signals flow natively via OTLP gRPC without any sidecar or adapter. The OTel Go SDK is embedded directly in the Traefik binary.

However, Level 3 is not reached for several reasons:

1. **Metrics configuration inconsistency**: Prometheus is the default metrics backend; OTLP metrics require explicit opt-in. Vendor backends remain fully co-equal with no deprecation signal.
2. **OTLP log export is experimental**: The `experimental.otlpLogs: true` feature gate is required.
3. **Incomplete standard env var support**: `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, and `OTEL_SERVICE_NAME` are not documented as supported.
4. **No formal stability contract**: No documented commitment to treat the telemetry integration surface as a stable API.

---

### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

All deprecated attributes originate exclusively from the **`@opentelemetry/instrumentation-http`** scope (the Node.js backend auto-instrumentation library), not from Traefik's own instrumentation. Each deprecated attribute appears on exactly 24 spans:

| Deprecated Attribute | Count | OTel Replacement |
|---|---|---|
| `http.method` | 24 | `http.request.method` |
| `http.status_code` | 24 | `http.response.status_code` |
| `http.url` | 24 | `url.full` |
| `http.target` | 24 | `url.path` + `url.query` |
| `http.host` | 24 | `server.address` + `server.port` |
| `http.scheme` | 24 | `url.scheme` |
| `http.flavor` | 24 | `network.protocol.version` |
| `http.user_agent` | 24 | `user_agent.original` |
| `net.host.name` | 24 | `server.address` |
| `net.host.port` | 24 | `server.port` |
| `net.peer.port` | 24 | `network.peer.port` |

Additional non-standard attribute: `http.status_text` (e.g., `"OK"`, `"NOT FOUND"`) — not part of OTel semantic conventions, emitted by the Node.js backend instrumentation.

##### Current OTel attributes found on spans

Traefik's own instrumentation scope (`github.com/traefik/traefik`) exclusively uses **current, stable** OTel semantic conventions:

| Attribute | Example Value | OTel Semconv |
|---|---|---|
| `http.request.method` | `GET` | ✅ Current |
| `http.response.status_code` | `200`, `404` | ✅ Current |
| `url.full` | `http://10.244.0.6:3000/` | ✅ Current |
| `url.path` | `/` | ✅ Current |
| `url.scheme` | `http` | ✅ Current |
| `server.address` | `10.244.0.6` | ✅ Current |
| `server.port` | `3000` | ✅ Current |
| `client.address` | `127.0.0.1` | ✅ Current |
| `network.peer.address` | `10.244.0.6` | ✅ Current |
| `network.protocol.version` | `1.1` | ✅ Current |
| `user_agent.original` | `curl/8.18.0` | ✅ Current |

Traefik-specific (domain) attributes on spans:
- `entry_point` — Traefik entrypoint name (e.g., `web`, `traefik`, `metrics`); not in OTel semconv, not documented as an extension

##### Metric names and conventions

The metric corpus contains three distinct categories:

**1. OTel-compliant metrics (current semconv names):**
- `http.server.request.duration` — ✅ Matches OTel HTTP server semconv exactly; attributes use current keys
- `http.client.request.duration` — ✅ Matches OTel HTTP client semconv exactly

**2. Traefik-proprietary metrics (non-OTel naming, non-OTel attribute keys):**
- `traefik_entrypoint_request_duration_seconds`, `traefik_entrypoint_requests_total`, etc.
- `traefik_router_request_duration_seconds`, `traefik_router_requests_total`, etc.
- `traefik_service_request_duration_seconds`, `traefik_service_requests_total`, etc.
- `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`

Proprietary metric attribute keys (not OTel semconv): `code`, `method`, `protocol`, `entrypoint`, `router`, `service`

**3. Infrastructure metrics (from OTel collector scrapers):**
- `go_*`, `process_*` — Prometheus-format Go runtime metrics
- `k8s.*` — OTel Kubernetes semantic conventions ✅
- `scrape_*` — Prometheus scrape metadata

##### Log attributes

Log records use **Traefik-proprietary PascalCase attribute names** — not aligned with OTel semantic conventions:

| Log Attribute | Semantic Meaning | OTel Equivalent |
|---|---|---|
| `RequestMethod` | HTTP method | `http.request.method` |
| `DownstreamStatus` | Response status code | `http.response.status_code` |
| `RequestPath` | URL path | `url.path` |
| `RequestScheme` | URL scheme | `url.scheme` |
| `ClientAddr` | Client address+port | `client.address` + `client.port` |
| `RouterName` | Traefik router | — (domain-specific) |
| `ServiceName` | Traefik service | — (domain-specific) |
| `TraceId` / `trace_id` | Trace ID (duplicated!) | OTel `traceId` field |
| `SpanId` / `span_id` | Span ID (duplicated!) | OTel `spanId` field |

Notable issues:
- Log body is a **JSON-encoded string blob** rather than structured attributes
- `TraceId`/`SpanId` are duplicated (both PascalCase and snake_case variants present)
- Severity is `"info"` for all records but the embedded JSON shows `"level": "panic"` — a mismatch

##### Cross-signal consistency

Shared attribute keys between traces and OTel metrics (✅ consistent naming):
- `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`

However, the **Traefik proprietary metrics** use completely different names for the same concepts:
- Traces: `http.request.method` → Traefik metrics: `method`
- Traces: `http.response.status_code` → Traefik metrics: `code`
- Traces: `network.protocol.version` → Traefik metrics: `protocol`

**No shared attribute keys between traces/metrics and logs.** Logs use proprietary PascalCase names while traces use OTel semconv.

##### Schema URL

Schema URLs present on all three signals (`https://opentelemetry.io/schemas/1.40.0`), demonstrating intent toward semconv governance.

#### Rationale

Traefik is assigned **Level 1 — OpenTelemetry-Aligned**.

Traefik's own tracer exclusively uses current, stable OTel HTTP semantic conventions on spans. The two OTel-named metrics are correctly named and use correct OTel attribute keys. Schema URLs are present on all three signals. However, three significant inconsistencies prevent Level 2:

1. **Proprietary metric family:** The `traefik_*` metric family (13 metrics) uses Prometheus-style naming and non-OTel attribute keys (`code`, `method`, `protocol`, `entrypoint`, `router`, `service`).
2. **Log attributes are entirely proprietary:** All 36 log attributes use Traefik's own PascalCase naming — none align with OTel semantic conventions.
3. **Deprecated attributes co-exist in the pipeline:** From the Node.js backend's instrumentation, but present in the same telemetry dataset.

**Cross-signal inconsistency** is the clearest Level 1 indicator: the concept of \"HTTP method\" has three different names — `http.request.method` (traces), `method` (Traefik metrics), and `RequestMethod` (logs).

---

### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik v3.7.1 uses the OTel Go SDK directly and emits a rich set of resource attributes natively via its own `resource.New(...)` call in `pkg/types/tracing.go`:

| Attribute | Value observed |
|---|---|
| `service.name` | `traefik` |
| `service.version` | `3.7.1` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-6d96b569d5-sthkv` (pod hostname) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.10` |
| `process.runtime.description` | `go version go1.25.10 linux/amd64` |
| `process.command_args` | (CLI args array) |

Additionally, Traefik's built-in Kubernetes resource detector (`k8sdetector.go`) natively emits `k8s.namespace.name`, `k8s.pod.uid`, and `k8s.pod.name` when running in a Kubernetes cluster.

##### Pipeline-derived resource attributes (added by Collector enrichment)

The OTel Collector's `k8sattributes` processor enriches telemetry using `k8s.pod.uid` for pod association. These attributes are **not** emitted by Traefik itself: `k8s.node.name`, `k8s.deployment.name`, `k8s.replicaset.name`, `k8s.container.name`, `k8s.pod.start_time`, `k8s.pod.label.*`, `k8s.pod.annotation.*`.

##### service.name consistency across signals

- Traces: `traefik` ✅
- Metrics (OTLP push): `traefik` ✅
- Metrics (Prometheus scrape): `traefik` ✅
- Logs: `traefik` ✅
- **Consistent: Yes** — `service.name` and `service.version: 3.7.1` are identical across all signal types and both metrics collection methods.

##### OTEL_* env var support

**Partially documented:**

- **`OTEL_RESOURCE_ATTRIBUTES`**: Explicitly documented in the Traefik tracing reference: *\"Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes.\"* Also documented in the metrics OTLP section.
- **`OTEL_SERVICE_NAME`**: **Not explicitly documented** in Traefik docs. However, it is **functionally supported** because the OTel Go SDK's `resource.WithFromEnv()` detector reads `OTEL_SERVICE_NAME` and the SDK's merge order gives env vars precedence over config-file values.
- **Configuration precedence**: The precedence order between `tracing.serviceName`, `tracing.resourceAttributes`, and `OTEL_RESOURCE_ATTRIBUTES`/`OTEL_SERVICE_NAME` is not explicitly documented.

##### Identity misplacement

**None observed.** No `service.name`, `service.version`, or other identity attributes were found as span-level attributes. All identity attributes are correctly placed in the resource scope across all signals.

#### Rationale

Traefik v3.7.1 achieves **Level 2 (OpenTelemetry-Native)** for resource attributes and configuration.

The project emits a comprehensive and consistent set of native resource attributes across all three signal types using the OTel Go SDK directly. `service.name` and `service.version` are stable, identical across signals, and correctly placed in the resource scope. Traefik also natively detects three Kubernetes resource attributes (`k8s.namespace.name`, `k8s.pod.uid`, `k8s.pod.name`) via its own `k8sdetector`.

The remaining Level 3 gaps are: (1) `OTEL_SERVICE_NAME` override behavior is undocumented, (2) configuration precedence rules are not explicitly stated, and (3) there is no documentation about identity behavior in multi-tenant or shared-cluster deployments.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure
- Total root spans: 73
- Total child spans: 170
- Total spans (all services): 243
- Multi-span traces: 20
- Single-span traces: 54 (all `/ping` or `/metrics` — legitimate terminal endpoints)
- Span kind distribution: SERVER=102, INTERNAL=117, CLIENT=24

##### Context propagation

W3C Trace Context (`traceparent`) is documented as the **default** propagator in Traefik's OpenTelemetry documentation. This is confirmed in source code (`pkg/tracing/tracing.go`) via `autoprop.NewTextMapPropagator()` from `go.opentelemetry.io/contrib/propagators/autoprop`.

**Observed evidence:** Trace `4bf92f3577b34da6a3ce929d0e0e4736` (40 spans) shows 5 Traefik `GET` (SERVER) spans all correctly parented to external span `00f067aa0ba902b7` — a span ID not present in the collected JSONL, meaning it was injected via an incoming `traceparent` header. Context is also correctly propagated **outbound**: the `ReverseProxy` (CLIENT) spans inject the `traceparent` header into the upstream request, causing the backend to produce SERVER spans parented to Traefik's CLIENT span within the same trace ID.

##### Trace coherence

A complete proxied request produces the following coherent chain:

```
GET (traefik, SERVER, kind=2)                    ← entry point, extracts incoming traceparent
  └─ ReverseProxy (traefik, CLIENT, kind=3)      ← outgoing call to upstream, injects traceparent
       └─ GET / (otel-eval-backend, SERVER, kind=2)   ← upstream receives context
            ├─ middleware - query (INTERNAL)
            ├─ middleware - expressInit (INTERNAL)
            ├─ middleware - jsonParser (INTERNAL)
            ├─ middleware - <anonymous> (INTERNAL)
            └─ request handler - / (INTERNAL)
```

##### Span status

3 error spans (all `ReverseProxy` for 404 upstream responses) correctly set `status.code=2`. No status messages on errors.

##### Span links

No span links present — appropriate for Traefik's synchronous proxy model.

#### Rationale

Traefik achieves **Level 2 (OpenTelemetry-Native)**. The evidence is strong and consistent:

1. **W3C Trace Context is the default propagator** — documented explicitly and confirmed by observed trace data where an external `traceparent` was extracted and honored.
2. **Outbound context injection works correctly** — The `ReverseProxy` (CLIENT) span injects context into upstream requests, causing the backend to produce spans within the same trace.
3. **Span kinds are intentional and correct** — Entry points are consistently `SERVER` (kind=2), outbound proxy calls are `CLIENT` (kind=3), internal middleware spans are `INTERNAL` (kind=1).
4. **Trace topology is logically meaningful** — The `GET → ReverseProxy → GET /` chain represents what a user cares about.
5. **Single-span traces are not fragmentation** — The 54 single-span traces are for `/ping` and `/metrics`, which are terminal endpoints at Traefik itself.

Level 3 is not reached because: (a) no span events are used, (b) no evidence of architectural trace modeling review or trace structure tests, and (c) the project's synchronous proxy nature limits the opportunity to demonstrate advanced async/fan-out trace modeling.

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability

- **Traces**: Flowing — OTLP/gRPC push from Traefik; 243 spans across 28 JSONL lines
- **Metrics**: Flowing — dual pipeline (OTLP/gRPC push + Prometheus scrape); 7,603 metric series across 98 JSONL lines
- **Logs**: Flowing — OTLP/gRPC push of access logs; 72 log records across 5 JSONL lines

##### Cross-signal correlation

- Log records with traceId: **24 of 24 (100%)** — every access-log record carries a non-zero `traceId` in the OTLP log record envelope field
- Log records with spanId: **24 of 24 (100%)**
- Matching trace IDs across `traces.jsonl` and `logs.jsonl`: **20 distinct trace IDs** confirmed to appear in both files
- Shared attribute keys (traces ∩ OTel metrics): `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme` — 6 keys shared
- Additional shared dimensions: `entrypoint` values (`web`, `traefik`, `metrics`) appear in both span `entry_point` attributes and metric `entrypoint` labels

##### Collection model per signal

| Signal | Export mechanism | Receiver in Collector |
|--------|------------------|-----------------------|
| **Traces** | OTLP/gRPC push (`--tracing.otlp.grpc`) at 100% sample rate | `otlp` receiver |
| **Metrics (OTel)** | OTLP/gRPC push (`--metrics.otlp.grpc`) every 10 s | `otlp` receiver |
| **Metrics (Prometheus)** | Prometheus scrape of `/metrics` on port 9100 every 15 s | `prometheus` receiver |
| **Logs** | OTLP/gRPC push (`--accesslog.otlp.grpc`) | `otlp` receiver |

##### Investigative workflow

A user can follow the **metric → trace → log** path without manual bridging:

1. **Metric anomaly** — `http.server.request.duration` shows elevated latency for `http.response.status_code=404`
2. **Pivot to trace** — 6 shared OTel attribute keys enable filtering spans by the same dimensions
3. **Inspect correlated log** — The access-log OTLP record carries the identical `traceId` and `spanId` as the root entry-point span, with rich request context (`RouterName`, `ServiceName`, `DownstreamStatus`, `Duration`) not duplicated in the span

**Limitation:** The OTel-semantic-convention metrics (`http.server.request.duration`) do **not** carry `router` or `service` labels — these dimensions are only available in legacy `traefik_router_*` Prometheus-format metrics.

#### Key findings table

| Criterion | Finding |
|-----------|---------|
| All 3 signals present | ✅ Traces, metrics, logs all flowing via OTLP |
| Log-trace correlation | ✅ 100% of access logs carry `traceId`/`spanId` automatically |
| Metric-trace attribute overlap | ✅ 6 shared keys; `entrypoint` dimension aligns |
| Metric→trace→log pivot | ✅ Possible without manual bridging |
| Cross-signal workflow docs | ❌ Not documented as a unified workflow |
| Cardinality/cost guidance | ❌ No explicit guidance |

#### Rationale

Traefik v3.7.1 achieves **Level 2 (OpenTelemetry-Native)** because all three signals are first-class OTLP outputs, and **trace context is automatically injected into access logs** — the strongest cross-signal correlation indicator.

Level 3 is not awarded because: (1) no cross-signal investigative workflow documentation exists; (2) the dual metric pipeline (OTLP push + Prometheus scrape) creates redundancy without explicit guidance; (3) the OTel-native `http.server.request.duration` metric lacks `router`/`service` dimensions; and (4) there is no published guidance on signal selection trade-offs or cardinality management strategy.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming assessment

- **Good (logical/user-relevant)**:
  - `GET` — HTTP method as span name is semantically aligned with OTel HTTP conventions; interpretable when combined with `url.path` and `entry_point` attributes, though ambiguous alone
  - `GET /health`, `GET /` — from backend test service; method + path is the correct OTel pattern

- **Bad / Neutral (internal/implementation detail)**:
  - `ReverseProxy` — Traefik internal component name; an operator unfamiliar with Traefik internals would not immediately understand this span
  - `middleware - query`, `middleware - expressInit`, `middleware - jsonParser`, `middleware - <anonymous>` — from backend's Express instrumentation; expose framework internals
  - Bare `GET` for Traefik's own entry-point spans — no routing context in the span name; router/service context absent from trace spans entirely

##### Log verbosity

| Severity Text | Severity Number | Count |
|---------------|-----------------|-------|
| `info` | 9 | 24 |

- **Critical anomaly**: The OTel `severityText` field is `info` (severityNumber=9), but the JSON body embedded inside the log record contains `"level":"panic"`. This is a **severity mismatch bug** — alerting or filtering systems relying on OTel severity levels would be misled.
- **Log body is a JSON string**: The log body is a JSON-encoded string rather than structured OTel attributes, which means log backends cannot natively filter individual fields without additional parsing.

##### Metric quality

| Metric Category | Assessment |
|---|---|
| `traefik_entrypoint_*` (requests_total, request_duration_seconds, bytes) | ✅ SLO-relevant — request rate and latency by entry point |
| `traefik_router_*` (requests_total, request_duration_seconds, bytes) | ✅ SLO-relevant — request rate and latency by router+service |
| `traefik_service_*` (requests_total, request_duration_seconds, bytes) | ✅ SLO-relevant — request rate and latency per backend service |
| `traefik_open_connections`, `traefik_config_*` | ✅ Operational — connection count and config health |
| `http.server.request.duration`, `http.client.request.duration` | ✅ OTel semantic convention metrics |
| `go_*`, `process_*` | ⚠️ Runtime internals — useful for infra teams, not SLO-relevant |
| `scrape_*` | ⚠️ Prometheus scrape metadata — not operational signal |

- **High-cardinality concerns**: Metric labels are well-controlled (`code`, `entrypoint`, `method`, `protocol`, `router`, `service` — all bounded). `url.full` in `ReverseProxy` spans contains raw backend IP+port URLs, which is a minor concern.

##### Default production usability

**Mixed — partially usable without customization, but with notable gaps:**

1. **Traces**: `GET` (ambiguous without attributes) and `ReverseProxy` (internal name). **Router and service context is absent from trace spans** — only available in metrics and access logs.
2. **Logs**: Rich content (`RouterName`, `ServiceName`, `Duration`), but **severity mismatch bug** (`info` OTel envelope vs. `panic` in body) is a production-quality defect. JSON-as-string body limits native field querying.
3. **Metrics**: Well-designed for operator use — three layers of granularity (entrypoint → router → service). The strongest signal quality area.
4. **No sampling/filtering guidance**: Kubernetes liveness probe traffic (`/ping`) and metrics scrape traffic (`/metrics`) are included in traces by default.

#### Rationale

Traefik is assessed at **Level 1 — OpenTelemetry-Aligned**.

The strongest evidence for this level is the **inconsistency across signal types**: metrics are genuinely operator-focused and SLO-ready, but the trace and log signals have notable usability gaps.

**Key factors preventing Level 2:**

1. **`ReverseProxy` span name** is an internal Traefik component name, not a logical operation name.
2. **Router and service context is absent from trace spans** — the primary way operators understand Traefik traffic flow is not present on spans; it appears only in metrics and access logs.
3. **Severity mismatch in logs**: All 24 log records have OTel `severityText=info` but the JSON body contains `"level":"panic"`. This is a production-quality defect.
4. **Liveness probe and metrics scrape noise**: Included in traces by default without filtering guidance.

The metrics signal alone would qualify for Level 2, but the overall telemetry package is better characterized as Level 1: some effort toward operator usability, but signals are still shaped by internal implementation perspectives and have quality defects requiring operator workarounds.

---

### 7. Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence

- Traces: **Present** — `https://opentelemetry.io/schemas/1.40.0`
- Metrics: **Present** — `https://opentelemetry.io/schemas/1.40.0` (Traefik-side), `https://opentelemetry.io/schemas/1.18.0` (collector-side)
- Logs: **Present** — `https://opentelemetry.io/schemas/1.40.0`

All three signals carry schema URLs, signaling intentional versioning of semantic conventions.

##### Telemetry documentation

Traefik has a dedicated **Observability** documentation section:
- Metrics reference page — lists all metric names, types, labels, and descriptions for both OTel and Prometheus formats
- Tracing reference page — documents span configuration options, verbosity levels, and resource attributes
- Observability overview page — links all signals

No explicit \"stable\" vs. \"experimental\" stability markers are applied to individual metrics or spans.

##### Release note quality for telemetry changes

Telemetry changes are consistently tagged with `[otel]`, `[metrics]`, `[tracing]`, or `[metrics,otel]` labels in the CHANGELOG. Examples:

- **v3.5.4**: `[metrics,otel]` Rename `traefik_tls_certs_not_after_milliseconds` to `traefik_tls_certs_not_after_seconds`
- **v3.5.0**: `[middleware,tracing]` Introduce trace verbosity config and produce less spans by default. **Impact:** Existing configurations will default to `minimal`, resulting in **fewer spans being generated than before**
- **v3.3.4**: `[metrics,otel]` Change request duration metric unit from millisecond to second — **Old Unit:** Milliseconds → **New Unit:** Seconds
- **v3.3**: `[tracing]` Rename `tracing.globalAttributes` → `tracing.resourceAttributes`
- **v3.7.0-ea.2**: `[logs,otel]` Add OTel-conformant trace context attributes to access logs

Breaking changes include a \"**Migration Required**\" or \"**Impact**\" section with old/new values and migration steps.

##### Stable vs experimental labeling

No per-metric or per-span stability label exists. Traefik uses a feature-level \"experimental\" concept for providers, but this pattern does not extend to individual telemetry signals.

##### Instrumentation scope versions

| Signal | Scope | Version |
|--------|-------|---------|
| Traces | `github.com/traefik/traefik` | `unknown` ⚠️ |
| Metrics | `github.com/traefik/traefik` | `v3.7.1` ✅ |
| Logs | `traefik` | `unknown` ⚠️ |

The metrics scope correctly carries `v3.7.1`, tying emitted metrics to the Traefik release. The traces scope version is `unknown` — trace consumers cannot determine the instrumentation version from the telemetry data itself.

#### Rationale

Traefik is assessed at **Level 2 — OpenTelemetry-Native** for the following reasons:

**Strengths supporting Level 2:**
1. **Schema URLs present on all signals** — demonstrates intentional alignment with OTel versioning conventions.
2. **Comprehensive telemetry reference documentation** — a dedicated metrics reference page lists all metric names, types, dimensions, and descriptions.
3. **Proactive communication of breaking telemetry changes** — the versioned migration guide (`migrate/v3.md`) has dedicated per-version sections for telemetry changes with \"Migration Required\" / \"Impact\" sections.
4. **User impact awareness** — the v3.5.0 traceVerbosity change explicitly warns users that fewer spans will be produced.
5. **Official Grafana dashboards** — Traefik maintains and publishes official Grafana dashboards, creating accountability for metric stability.

**Gaps preventing Level 3:**
1. **No per-signal stability labels** — users cannot tell which telemetry signals are safe to build long-lived SLOs on.
2. **Some breaking changes are immediate** — the request duration unit change (ms→s in v3.3.4) was applied without a dual-emit deprecation period.
3. **Traces scope version is `unknown`** — limiting consumers' ability to correlate telemetry shape with a specific release.
4. **No formal telemetry change governance** — changes go through standard PRs without a dedicated telemetry impact review checklist.
5. **No proactive regression detection** — no evidence of automated telemetry contract testing or CI-based checks.

---

## Key findings

### Top 3 strengths

1. **All three signals via native OTLP/gRPC, zero adapters required.** Traefik v3 removed Jaeger and Zipkin entirely — OTLP is now the only tracing path. Traces, metrics, and access logs all push directly to a standard OTel Collector via OTLP/gRPC without any sidecar, bridge, or protocol adapter. The OTel Go SDK is embedded natively in the Traefik binary with rich resource attributes (`service.name`, `service.version`, `telemetry.sdk.*`, `process.*`, `os.*`) consistent across all signals.

2. **Automatic trace-to-log correlation at 100% coverage.** Every access-log record (24/24) carries `traceId` and `spanId` matching the corresponding entry-point span, injected natively by Traefik's OTLP access-log exporter. Twenty distinct trace IDs were confirmed to appear in both `traces.jsonl` and `logs.jsonl`, enabling metric→trace→log pivoting without manual bridging or timestamp alignment.

3. **Proactive telemetry change management with migration guidance.** Breaking telemetry changes (metric renames, unit changes, span verbosity changes) are consistently documented in a versioned migration guide with explicit \"Migration Required\" and \"Impact\" sections. Schema URLs (`https://opentelemetry.io/schemas/1.40.0`) are present on all three signals, and official Grafana dashboards create accountability for metric stability.

### Top 3 areas for improvement

1. **Semantic convention fragmentation across signals.** The concept of \"HTTP method\" has three different names: `http.request.method` (traces), `method` (Traefik proprietary metrics), and `RequestMethod` (access logs). The `traefik_*` metric family (13 metrics) uses non-OTel attribute keys (`code`, `method`, `protocol`) for concepts where OTel semconv exists. Migrating proprietary metric attribute keys to OTel semconv names and adopting OTel attribute naming in access log records would elevate Dimension 2 from Level 1 to Level 2.

2. **Signal quality defects requiring operator workarounds.** Three issues prevent the telemetry package from being production-ready out of the box: (a) a severity mismatch bug where all 24 access-log records have OTel `severityText=info` but the JSON body contains `"level":"panic"`, which would cause alerting systems to miss severity-based rules; (b) router and service context is absent from trace spans — operators cannot identify which Traefik router matched a request from trace data alone; and (c) Kubernetes liveness probe and Prometheus scrape traffic is included in traces by default, adding noise without operational value.

3. **Incomplete standard OTel environment variable support and documentation.** `OTEL_EXPORTER_OTLP_ENDPOINT` is not documented as supported (endpoint configuration requires project-specific Helm flags), `OTEL_SERVICE_NAME` is functionally supported via the OTel SDK but undocumented, and configuration precedence between project config and `OTEL_*` env vars is not explained. Additionally, the traces instrumentation scope version is `unknown` (while the metrics scope correctly carries `v3.7.1`), and there are no per-signal stability labels to help users identify which telemetry signals are safe for long-lived SLOs.

### Notable observations

- **Dual metrics pipeline creates ambiguity.** Traefik simultaneously pushes OTLP metrics and exposes a Prometheus scrape endpoint, with both paths delivering overlapping metric sets. The OTLP path includes OTel semconv metrics (`http.server.request.duration`) that the Prometheus path does not, while the Prometheus path includes Go runtime metrics. There is no documentation guiding users on which pipeline to use or how to avoid double-counting.

- **OTLP log export behind experimental gate.** The `experimental.otlpLogs: true` feature gate is required to enable log export, signaling that this signal is not yet considered stable by the project. This is the only signal still gated, making the log integration surface the least mature of the three.

- **Traefik's native Kubernetes resource detection is underappreciated.** Unlike many projects that rely entirely on the OTel Collector's `k8sattributes` processor for Kubernetes identity, Traefik natively detects `k8s.namespace.name`, `k8s.pod.uid`, and `k8s.pod.name` via its own `k8sdetector.go`. This means Kubernetes identity is available even without collector enrichment, which is a notable capability for bare-metal or non-standard collector deployments.

---

## Methodology notes

- **Evaluation cluster**: kind cluster (`otel-eval-traefik`), single-node, Kubernetes v1.31
- **Traefik version**: v3.7.1 (Helm chart `traefik/traefik` version 40.2.0, namespace `traefik`)
- **Telemetry collection window**: Approximately 11 minutes of live traffic; backend test application (`otel-eval-backend`) generated HTTP requests to exercise the proxy path
- **OTel Collector version**: `otel/opentelemetry-collector-contrib` v0.150.1
- **Collector pipeline**: OTLP gRPC receiver → `k8sattributes` processor → file exporter (JSONL)
- **Prometheus scrape**: Also configured to capture Traefik's `/metrics` endpoint on port 9100 every 15 s
- **Signal files**: `traces.jsonl` (28 lines / 243 spans), `metrics.jsonl` (98 lines / 7,603 metric series), `logs.jsonl` (5 lines / 72 records)
- **Key analytical distinction**: All dimension evaluations distinguish between project-native telemetry and collector-derived enrichment. Resource attributes added by the `k8sattributes` processor are explicitly marked as pipeline-derived and not credited to the project.
- **Backend test app**: `otel-eval-backend` (Node.js, auto-instrumented with `@opentelemetry/instrumentation-http` v0.48.0 and `@opentelemetry/instrumentation-express` v0.35.0) was deployed as a backend service to exercise Traefik's proxy path and context propagation. Backend spans appear in the trace data but are excluded from Traefik-specific dimension assessments.
- **Maturity model**: OpenTelemetry Support Maturity Model for CNCF Projects (draft), four levels: 0 (Instrumented), 1 (OpenTelemetry-Aligned), 2 (OpenTelemetry-Native), 3 (OpenTelemetry-Optimized)
