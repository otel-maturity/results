# OpenTelemetry Support Maturity Evaluation: Istio

## Project overview

- **Project**: Istio — open-source service mesh providing traffic management, mTLS, and observability for microservices via Envoy proxy sidecar injection
- **Version evaluated**: 1.29.2
- **Evaluation date**: 2026-05-07
- **Cluster**: otel-eval-istio (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP is the documented path for traces and logs; metrics remain Prometheus-only with no OTLP export |
| Semantic Conventions | 1 | Istio-native span attributes use deprecated HTTP conventions (`http.method`, `http.status_code`, `http.url`); Prometheus metrics use proprietary label schemas |
| Resource Attributes & Configuration | 1 | `service.name` is set natively for traces/logs but inconsistent (`otel-eval-backend` vs `otel-eval-backend.demo`); no `service.version`; metrics use scrape-job names; no `OTEL_*` env var support |
| Trace Modeling & Context Propagation | 2 | W3C Trace Context propagated correctly across gateway and sidecar; coherent 4-span trees per request; context flows from Envoy to app SDK spans |
| Multi-Signal Observability | 2 | All three signals flow; logs carry `traceId`+`spanId` enabling trace-log correlation; metrics lack shared `service.name` identity with traces/logs |
| Audience & Signal Quality | 1 | Span names are Envoy-internal (`otel-eval-backend.demo.svc.cluster.local:3000/*`, `router outbound|...; egress`); access logs are unstructured strings; 275 metrics with heavy low-level `envoy_*` noise |
| Stability & Change Management | 1 | Telemetry behavior is documented in Istio's official docs; no explicit telemetry stability contract; no schema URL on OTLP exports; Prometheus label schemas not versioned |

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC (port 4317) via Envoy's native OpenTelemetry tracing extension
- **Metrics**: Flowing — Prometheus scrape (ports 15014, 15020, 15090) via OTel Collector `prometheusreceiver`
- **Logs**: Flowing — OTLP gRPC (port 4317) via Envoy Access Log Service (ALS)

### Resource attributes (native, before collector enrichment)

Envoy emits the following resource attributes natively in OTLP traces:

| Attribute | Value example | Source |
|-----------|--------------|--------|
| `service.name` | `istio-gateway.istio-system`, `otel-eval-backend.demo` | Envoy (from node metadata) |
| `telemetry.sdk.name` | `envoy` | Envoy |
| `telemetry.sdk.language` | `cpp` | Envoy |
| `telemetry.sdk.version` | `af30be60b7c35f2aceaea1b7382c7fbf12aa5e67/1.37.2-dev/Clean/RELEASE/BoringSSL` | Envoy |

For ALS logs, Envoy natively emits only `traceId`, `spanId`, and the raw access log string body. No resource attributes are set by Envoy itself on log records — all resource attributes on logs come from the collector's `k8sattributes` processor.

For Prometheus-scraped metrics, the resource identity is the scrape job name (`service.name=istio-envoy-sidecars`, `service.name=istio-ingressgateway`), not the actual workload identity.

### Resource attributes (after collector enrichment)

The `k8sattributes` processor adds the following to OTLP traces and logs:

- `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.namespace.name`, `k8s.node.name`
- `k8s.deployment.name`, `k8s.replicaset.name`
- `k8s.container.name`
- `k8s.pod.label.*` (all pod labels, including Istio-specific labels)
- `k8s.pod.annotation.*` (all pod annotations)
- `container.id`
- `host.arch`, `host.name`
- `os.type`, `os.version`
- `process.*` (for the Node.js app spans only)

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **Traces via OTLP gRPC**: Confirmed flowing. Istio's `meshConfig.extensionProviders` with `opentelemetry` type sends spans directly to an OTel Collector on port 4317 using OTLP gRPC.
- **Logs via OTLP gRPC (ALS)**: Confirmed flowing. Istio's `envoyOtelAls` extension provider streams Envoy access logs as OTLP LogRecords to the collector.
- **Metrics via Prometheus scrape**: Istio exposes three Prometheus endpoints (istiod on port 15014, Envoy sidecars on port 15090, ingress gateway on port 15020). **No OTLP metrics export exists natively**.
- **Configuration mechanism**: Istio-specific. Traces and logs are enabled via `meshConfig.extensionProviders` in the istiod Helm chart values, plus a `Telemetry` CR (`telemetry.istio.io/v1`). Standard `OTEL_*` environment variables are not supported — Istio uses its own configuration API.
- **Documentation**: Istio's official documentation has a dedicated "OpenTelemetry" page for tracing and describes the `extensionProviders` mechanism clearly. OTel is positioned as the modern recommended tracing backend, explicitly preferred over the older Zipkin/Jaeger providers.

#### Checklist assessment

- ✅ OTLP export is supported for traces and logs
- ✅ Users can connect to an existing OTel Collector without adapters (just configure the service address)
- ✅ OpenTelemetry is documented as the recommended path for traces
- ✅ Legacy Zipkin/Jaeger providers remain available but are not the default recommendation
- ❌ Metrics require Prometheus scraping — no OTLP metrics export path
- ❌ Configuration is not via `OTEL_*` env vars but via Istio's own `meshConfig` API
- ❌ Telemetry configuration is inconsistent across signals (OTLP for traces/logs, Prometheus for metrics)

#### Rationale

Istio has made OpenTelemetry the primary integration path for traces and access logs, with clean OTLP gRPC export that plugs directly into existing collectors. The `Telemetry` CR provides a clean, Kubernetes-native configuration API. However, the complete absence of OTLP metrics export — a fundamental gap — means users must maintain a parallel Prometheus scraping pipeline for metrics. This split integration model is characteristic of Level 2 (OTel-Native for some signals) but falls short of Level 3 because the integration surface is not unified and metrics remain Prometheus-only. The use of Istio-specific `meshConfig` rather than `OTEL_*` env vars is appropriate for a mesh-level infrastructure component and does not lower the score.

---

### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Trace attributes (Envoy spans)

Envoy emits the following span attributes on HTTP spans. All are **deprecated** in current OpenTelemetry semantic conventions:

| Attribute used | Current convention equivalent | Status |
|---------------|------------------------------|--------|
| `http.method` | `http.request.method` | **Deprecated** |
| `http.status_code` | `http.response.status_code` | **Deprecated** |
| `http.url` | `url.full` | **Deprecated** |
| `http.protocol` | `network.protocol.version` | **Deprecated** |
| `net.host.ip` | `server.address` | **Deprecated** |
| `net.host.port` | `server.port` | **Deprecated** |
| `net.peer.ip` | `network.peer.address` | **Deprecated** |
| `net.peer.port` | `network.peer.port` | **Deprecated** |
| `net.transport` | `network.transport` | **Deprecated** |

Envoy-proprietary attributes with no OTel equivalent:
- `component=proxy`
- `downstream_cluster`
- `upstream_cluster` / `upstream_cluster.name`
- `upstream_address`
- `response_flags`
- `node_id`
- `zone`
- `guid:x-request-id`
- `istio.mesh_id`, `istio.canonical_service`, `istio.canonical_revision`, `istio.cluster_id`, `istio.namespace`
- `peer.address` (not a standard OTel attribute)

Span kinds are correctly set:
- Gateway inbound SERVER spans: `kind=2` (SPAN_KIND_SERVER) ✅
- Gateway outbound CLIENT spans: `kind=3` (SPAN_KIND_CLIENT) ✅
- Sidecar inbound SERVER spans: `kind=2` (SPAN_KIND_SERVER) ✅

No `schemaUrl` is set on any OTLP export from Envoy.

##### Metric names and attributes

Istio's Prometheus metrics use a proprietary label schema:
- `istio_requests_total` with labels: `source_workload`, `destination_workload`, `destination_service`, `reporter`, `request_protocol`, `response_code`, `response_flags`, `connection_security_policy`, etc.
- These labels do **not** align with OTel HTTP semantic conventions (`http.request.method`, `http.response.status_code`, `server.address`)
- `envoy_*` metrics (193 total) use Envoy's internal naming conventions, not OTel semconv

The `k8s_cluster` receiver metrics (from the OTel Collector) do use OTel semantic conventions, but those are collector-generated, not Istio-native.

##### Log attributes

- Log body: unstructured Envoy access log format string (e.g., `[2026-05-07T16:06:35.689Z] "GET / HTTP/1.1" 200 - via_upstream...`)
- No structured log attributes on the log record itself
- No severity set (`UNSET` for all 88 records)
- No scope name (empty instrumentation library)
- `traceId` and `spanId` correctly set as OTLP fields (not as attributes)

#### Checklist assessment

- ✅ OpenTelemetry OTLP protocol is used for traces and logs
- ✅ Span kinds are correctly applied (SERVER/CLIENT)
- ✅ `traceId`/`spanId` are set as proper OTLP fields on log records
- ❌ All HTTP span attributes use deprecated conventions (`http.method`, `http.status_code`, `http.url`)
- ❌ No current stable semconv attributes (`http.request.method`, `http.response.status_code`, `url.full`, `url.path`) present
- ❌ Prometheus metric labels do not align with OTel HTTP semantic conventions
- ❌ Log body is unstructured text, not structured OTel log semantic conventions
- ❌ No `schemaUrl` on OTLP exports
- ❌ Istio-proprietary span attributes (`istio.mesh_id`, `upstream_cluster`, `response_flags`) are not documented as semantic extensions

#### Rationale

Istio uses OpenTelemetry protocols (OTLP) and correctly applies span kinds, but the actual attribute semantics are anchored to the older, deprecated OpenTelemetry HTTP conventions that Envoy implemented years ago. The current stable HTTP semconv (`http.request.method`, `http.response.status_code`, `url.full`, `server.address`, `server.port`) are entirely absent. This is a constraint of upstream Envoy's instrumentation, which Istio inherits — Envoy 1.37 still uses the pre-1.21 HTTP semconv. Prometheus metrics use Istio's own label schema with no alignment to OTel conventions. Logs are unstructured strings. The result is partial OTel alignment (protocols, span kinds) with legacy semantics — a clear Level 1 profile. A project cannot reach Level 2 while all HTTP span attributes use deprecated conventions.

---

### 3. Resource Attributes & Configuration

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Native resource attributes

Envoy sets `service.name` natively in OTLP trace resource attributes. However, the naming is inconsistent:
- Gateway spans: `service.name=istio-gateway.istio-system` (workload + namespace suffix)
- Backend sidecar spans: `service.name=otel-eval-backend.demo` (workload + namespace suffix)
- Backend app SDK spans: `service.name=otel-eval-backend` (no namespace suffix — the app SDK sets its own value)

This creates three different `service.name` values for what is conceptually two services. The namespace-appended format (`<workload>.<namespace>`) is Istio-specific and differs from the OTel convention of using `service.namespace` as a separate resource attribute.

No `service.version` is set by Envoy. No `service.instance.id` is set by Envoy.

##### OTEL_* environment variable support

Not applicable. Istio configures telemetry through `meshConfig.extensionProviders` and the `Telemetry` CR, not through `OTEL_*` environment variables. This is by design — Istio operates at the mesh level, not the application level. Users cannot override `service.name` or the OTLP endpoint via `OTEL_SERVICE_NAME` or `OTEL_EXPORTER_OTLP_ENDPOINT`.

##### Identity consistency across signals

| Signal | service.name | service.version | service.instance.id |
|--------|-------------|-----------------|---------------------|
| Traces (Envoy) | `istio-gateway.istio-system` / `otel-eval-backend.demo` | absent | absent |
| Traces (app SDK) | `otel-eval-backend` | absent | absent |
| Logs (ALS) | absent (no service.name on log resource) | absent | absent |
| Metrics (Prometheus) | `istio-envoy-sidecars` / `istio-ingressgateway` (scrape job names) | absent | `10.244.0.8:15090` (IP:port) |

Identity is fundamentally inconsistent across signals. The `service.name` in traces (`otel-eval-backend.demo`) does not match the `service.name` in metrics (`istio-envoy-sidecars`). Logs have no `service.name` at all. Cross-signal correlation by `service.name` is impossible without pipeline-level join logic.

The k8s.pod.name and k8s.namespace.name attributes (added by k8sattributes) provide a workable alternative correlation key across traces and logs, but not metrics.

#### Checklist assessment

- ✅ `service.name` is set on trace resource (by Envoy natively)
- ✅ `telemetry.sdk.name=envoy` correctly identifies the instrumentation
- ❌ `service.name` is inconsistent: namespace-appended for Envoy spans, not appended for app SDK spans
- ❌ `service.name` is absent from log resource attributes
- ❌ `service.name` in metrics uses scrape job names, not workload identity
- ❌ `service.version` absent across all signals
- ❌ `service.instance.id` absent from traces and logs
- ❌ `OTEL_SERVICE_NAME` / `OTEL_RESOURCE_ATTRIBUTES` not respected (by design, but still a gap)
- ❌ Identity cannot be correlated across traces, metrics, and logs by `service.name`

#### Rationale

Istio sets some resource attributes natively (primarily `service.name` and `telemetry.sdk.*` on traces), but the identity model is inconsistent across signals and even within traces (namespace suffix present for Envoy spans, absent for app SDK spans). The lack of `service.version` and `service.instance.id` limits correlation and grouping. The fundamental disconnect between trace identity (`otel-eval-backend.demo`) and metric identity (`istio-envoy-sidecars`) means multi-signal workflows cannot rely on shared identity. This is Level 1 — some resource attributes exist but are not consistently applied across signals.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

A complete end-to-end trace for a single HTTP request through the Istio mesh produces a coherent 9-span tree (verified for trace `427512ae3f1a5f637ef80f232fb29e0e`):

```
[istio-gateway.istio-system] otel-eval-backend.demo.svc.cluster.local:3000/* (SERVER, ROOT)
  └─ [istio-gateway.istio-system] router outbound|...; egress (CLIENT)
       └─ [otel-eval-backend.demo] otel-eval-backend.demo.svc.cluster.local:3000/* (SERVER)
            └─ [otel-eval-backend] GET / (SERVER)
                 ├─ middleware - query (INTERNAL)
                 ├─ middleware - expressInit (INTERNAL)
                 ├─ middleware - jsonParser (INTERNAL)
                 ├─ middleware - <anonymous> (INTERNAL)
                 └─ request handler - / (INTERNAL)
```

The trace spans three services and four instrumentation layers:
1. **Istio Gateway** (Envoy): inbound SERVER span (root)
2. **Istio Gateway** (Envoy): outbound CLIENT span (egress to backend)
3. **Backend Sidecar** (Envoy): inbound SERVER span
4. **Backend App** (Node.js OTel SDK): HTTP server span + Express middleware spans

Parent-child relationships are correct throughout. The Envoy sidecar SERVER span correctly becomes the parent of the app SDK SERVER span, confirming that `traceparent` is propagated by Envoy into the inbound request headers that the application receives.

##### Context propagation

- **W3C Trace Context** (`traceparent`/`tracestate`): Confirmed working. The gateway Envoy injects `traceparent` into outbound requests; the backend sidecar Envoy propagates it; the Node.js OTel SDK picks it up as the parent context.
- **Sampling**: 100% configured via `randomSamplingPercentage: 100` in the `Telemetry` CR — all requests traced as expected.
- **Propagation format**: W3C Trace Context is the default for the `opentelemetry` extension provider (as opposed to B3 used by the legacy Zipkin provider).

##### Trace coherence

Traces tell a complete, interpretable story of request flow through the mesh. The 4-layer span tree accurately reflects: external request → gateway → mesh routing → application. No fragmentation observed for synchronous request paths.

The only notable gap: the gateway SERVER span is the root (no upstream client span), which is expected — it represents the entry point into the mesh.

#### Checklist assessment

- ✅ W3C Trace Context propagated correctly across all mesh boundaries
- ✅ Parent-child relationships correct for gateway → sidecar → app chain
- ✅ Span kinds correctly applied (SERVER for inbound, CLIENT for outbound)
- ✅ Context propagation works across Envoy-to-Envoy and Envoy-to-SDK boundaries
- ✅ 100% sampling configurable via `Telemetry` CR
- ✅ Traces represent a coherent end-to-end view of request flow
- ⚠️ Span names are Envoy-internal (see Audience & Signal Quality dimension)
- ❌ No explicit documentation of trace modeling decisions (e.g., when spans are created, what constitutes a root span)

#### Rationale

Istio's trace modeling is intentional and effective. The four-layer span tree (gateway SERVER → gateway CLIENT → sidecar SERVER → app spans) correctly models the request path through the mesh. W3C Trace Context propagation works seamlessly across all instrumentation boundaries, including from Envoy to the OTel SDK in the application. This is a genuine Level 2 capability — context propagation is designed into the product, not accidental. The main gap for Level 3 would be explicit documentation of trace topology decisions and testing of more complex patterns (retries, circuit breaking, traffic shifting).

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability

All three signals are present and flowing:
- **Traces**: 9 OTLP batches, 9 lines in traces.jsonl, covering 37+ unique traces
- **Metrics**: 24 OTLP batches, 275 unique metric names (193 `envoy_*`, 56 `istio_*`, plus k8s cluster metrics)
- **Logs**: 7 OTLP batches, 88 log records (Envoy access logs via ALS)

##### Cross-signal correlation

**Trace-log correlation**: Confirmed and working. Every log record carries `traceId` and `spanId` as OTLP fields. Verified: log record with `traceId=427512ae3f1a5f637ef80f232fb29e0e, spanId=7d5d2a97633c3f5e` matches exactly to the Envoy sidecar SERVER span in the trace data. This enables users to jump from a specific span to its access log entry and vice versa.

**Trace-metric correlation**: Not achievable via shared identity. Traces use `service.name=otel-eval-backend.demo`; metrics use `service.name=istio-envoy-sidecars`. There is no shared attribute between the two signals that allows direct correlation. Users can correlate via `k8s.pod.name` (present in trace resources after enrichment, present in metric resources after enrichment), but this requires understanding the enrichment pipeline.

**Metric-log correlation**: Not achievable by identity. Metrics have `service.name=istio-envoy-sidecars`; logs have no `service.name`. Correlation requires matching on `k8s.pod.name` (enriched).

##### Collection model

| Signal | Protocol | Direction | Configuration |
|--------|----------|-----------|---------------|
| Traces | OTLP gRPC | Push (Envoy → Collector) | `meshConfig.extensionProviders` + `Telemetry` CR |
| Logs | OTLP gRPC | Push (Envoy → Collector) | `meshConfig.extensionProviders` + `Telemetry` CR |
| Metrics | Prometheus | Pull (Collector → Envoy) | Collector `prometheusreceiver` scrape config |

The split push/pull model is a notable friction point — traces and logs are OTLP push, metrics are Prometheus pull.

#### Checklist assessment

- ✅ All three signals are first-class and flowing
- ✅ Logs carry `traceId` and `spanId` enabling trace-log correlation
- ✅ Trace-log correlation is automatic and verified end-to-end
- ✅ Signals are designed together (the `Telemetry` CR configures both tracing and access logging)
- ❌ Metrics cannot be correlated with traces via `service.name` — identity is fundamentally different
- ❌ Mixed collection model (OTLP push for traces/logs, Prometheus pull for metrics) adds operational complexity
- ❌ No guidance in Istio docs on how to correlate metrics with traces (what attributes to join on)

#### Rationale

Istio achieves strong trace-log correlation out of the box — every access log record carries the exact `traceId` and `spanId` of the corresponding Envoy span. This is a genuine Level 2 capability. The weakness is metrics: the Prometheus scraping model produces metrics with a completely different identity (`service.name=istio-envoy-sidecars`) than traces (`service.name=otel-eval-backend.demo`), making metric-to-trace correlation require external tooling or pipeline logic. This limits the overall multi-signal experience. The result is Level 2 — signals are present and trace-log correlation is excellent, but the metric identity gap prevents the "pivot from metric to trace" workflow from working naturally.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Envoy span names reflect internal Envoy routing concepts, not user-meaningful operations:

- `otel-eval-backend.demo.svc.cluster.local:3000/*` — the inbound SERVER span name encodes the full Kubernetes DNS FQDN and a wildcard path pattern
- `router outbound|3000||otel-eval-backend.demo.svc.cluster.local; egress` — the outbound CLIENT span name encodes Envoy's internal cluster routing syntax

These names are functional for Envoy/Istio operators who understand the mesh internals, but are opaque to application developers and general operators. They do not follow the OTel HTTP span naming convention (`HTTP {method}` for server spans, `HTTP {method}` for client spans).

The app SDK spans (`GET /`, `middleware - query`, `middleware - expressInit`, etc.) provide better names but are generated by the application, not Istio.

##### Signal-to-noise ratio

**Metrics**: 275 unique metric names are collected. Of these:
- 193 are `envoy_*` internal Envoy metrics (circuit breakers, cluster manager internals, HTTP/2 frame counters, etc.) — most are low-level infrastructure counters not directly actionable by operators
- 56 are `istio_*` metrics — these are the operationally relevant ones (`istio_requests_total`, `istio_request_duration_milliseconds`, `istio_request_bytes`, etc.)
- The ratio of noise to signal is high: ~70% of metrics are Envoy internals

**Logs**: Access logs are unstructured strings in Envoy's default format. No severity level is set (all `UNSET`). The log body must be parsed to extract method, path, status code, latency, etc. There are no structured log attributes. The scope name is empty. This is a raw data dump, not structured observability.

**Traces**: 100% sampling in the evaluation environment is appropriate for testing but would be noisy in production. Envoy's span attribute set includes useful data (status codes, upstream cluster, request size) mixed with Envoy-internal identifiers (node_id, zone, response_flags).

##### Default usability

- An operator encountering `router outbound|3000||otel-eval-backend.demo.svc.cluster.local; egress` in a trace needs Istio-specific knowledge to understand this is an outbound call from the gateway to the backend.
- The `istio_*` metrics are well-labeled with source/destination workload information and are genuinely useful for RED (Rate, Errors, Duration) monitoring.
- The `envoy_*` metrics require deep Envoy knowledge to interpret.
- Access logs require regex parsing to extract structured fields.

#### Checklist assessment

- ✅ `istio_requests_total`, `istio_request_duration_milliseconds` are user-oriented operational metrics
- ✅ Span kinds are correct (helps tooling interpret spans)
- ✅ Trace-log correlation works without customization
- ❌ Span names expose Envoy routing internals, not logical operations
- ❌ 193 `envoy_*` metrics create significant noise; no filtering by default
- ❌ Access logs are unstructured strings with no severity, no scope, no attributes
- ❌ No documentation on which metrics are "operational" vs "debug"
- ❌ Defaults produce high-volume, low-signal telemetry in production

#### Rationale

Istio's telemetry is primarily designed for service mesh operators who understand Envoy internals. The `istio_*` metrics are well-designed for RED monitoring. However, span names expose Envoy's internal routing syntax rather than logical HTTP operations, access logs are unstructured, and the volume of `envoy_*` metrics overwhelms the operationally relevant signals. Some effort has been made (the `istio_*` metric family is clearly user-oriented), but the defaults remain noisy and require domain knowledge to interpret. This is Level 1 — some signals are user-oriented, others remain maintainer/operator-focused, and usability is inconsistent across signal types.

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

Istio has dedicated documentation pages for:
- Distributed tracing with OpenTelemetry (`istio.io/docs/tasks/observability/distributed-tracing/opentelemetry/`)
- Metrics (`istio.io/docs/reference/config/metrics/`)
- Access logging (`istio.io/docs/tasks/observability/logs/access-log/`)

The Prometheus metric schema (`istio_requests_total`, etc.) is documented with label definitions. The `Telemetry` CR API is documented with its fields. However, telemetry is not explicitly declared as a stability-guaranteed contract separate from the API surface.

##### Change communication

Istio follows semver and publishes detailed release notes. Telemetry changes are typically mentioned in release notes when they are significant. For example, the migration from per-signal configuration (EnvoyFilter) to the unified `Telemetry` CR was documented and communicated over multiple releases. However:
- There is no explicit "telemetry changelog" section
- Breaking changes to metric labels or span attributes are not always called out explicitly
- No deprecation timeline is published for legacy telemetry approaches

##### Schema URL presence

No `schemaUrl` is set on any OTLP export from Envoy (traces or logs). The k8s_cluster receiver metrics include `schemaUrl: https://opentelemetry.io/schemas/1.18.0` (set by the collector component, not Istio).

##### Stability guarantees

- The `Telemetry` CR API is versioned (`telemetry.istio.io/v1`) and follows Istio's API stability guarantees
- The Prometheus metric schema is de facto stable (operators depend on it for dashboards and alerts)
- Span names and attributes from Envoy are not explicitly guaranteed stable — they can change with Envoy upgrades
- No explicit "telemetry stability" policy exists separate from general API stability

#### Checklist assessment

- ✅ Telemetry configuration API (`Telemetry` CR) is versioned and documented
- ✅ Major telemetry changes (e.g., `Telemetry` CR introduction) were communicated with migration guidance
- ✅ Prometheus metric schema is documented
- ❌ No `schemaUrl` on OTLP trace/log exports
- ❌ No explicit telemetry stability contract separate from API stability
- ❌ Span names and attributes from Envoy can change with Envoy upgrades without explicit notice
- ❌ No distinction between "stable" and "experimental" telemetry fields
- ❌ No dedicated telemetry changelog section in release notes

#### Rationale

Istio has informal but meaningful telemetry stability — the `istio_*` Prometheus metrics have been stable for years and are documented. The `Telemetry` CR API is versioned. However, there is no explicit telemetry stability contract, no `schemaUrl` on OTLP exports, and span attributes from Envoy can change with Envoy upgrades without explicit notice in Istio release notes. The project is aware that users depend on telemetry (hence the documented migration from EnvoyFilter to `Telemetry` CR), but governance is informal. This is Level 1 — intent exists and major changes are communicated, but practices are inconsistent and no formal stability contract exists.

---

## Key findings

### Strengths

1. **Excellent trace-log correlation**: Every Envoy access log record carries `traceId` and `spanId` as native OTLP fields, enabling zero-configuration trace-log pivoting. This is a genuine Level 2 multi-signal capability that works out of the box.

2. **Coherent end-to-end trace topology**: The 4-layer span tree (gateway SERVER → gateway CLIENT → sidecar SERVER → app SDK spans) accurately models request flow through the mesh. W3C Trace Context propagation works correctly across all Envoy-to-Envoy and Envoy-to-SDK boundaries, producing complete distributed traces without application changes.

3. **Clean OTLP integration for traces and logs**: The `meshConfig.extensionProviders` + `Telemetry` CR configuration model is well-designed and integrates cleanly with standard OTel Collectors. No adapters, sidecars, or custom bridges are needed. The `Telemetry` CR provides a Kubernetes-native, namespace-scoped configuration API that is more ergonomic than the older EnvoyFilter approach.

4. **Rich `istio_*` operational metrics**: The `istio_requests_total`, `istio_request_duration_milliseconds`, `istio_request_bytes`, and related metrics provide well-labeled RED metrics with source/destination workload, namespace, response code, and security policy dimensions — directly useful for SLO monitoring and traffic analysis.

### Areas for improvement

1. **Update Envoy span attributes to current HTTP semantic conventions**: All Envoy HTTP span attributes use deprecated conventions (`http.method`, `http.status_code`, `http.url`, `net.host.*`, `net.peer.*`). Migrating to `http.request.method`, `http.response.status_code`, `url.full`, `server.address`, `server.port` would bring Istio to Level 2 on Semantic Conventions. This is an upstream Envoy dependency, but Istio can advocate for or contribute this change.

2. **Unify service identity across signals**: The `service.name` in traces (`otel-eval-backend.demo`) does not match metrics (`istio-envoy-sidecars`) or logs (absent). Adopting a consistent `service.name` convention (e.g., the Kubernetes workload name without namespace suffix, with `service.namespace` as a separate attribute) and propagating it to metrics would enable metric-to-trace correlation. At minimum, documenting the join key (e.g., `k8s.pod.name`) for cross-signal correlation would help users.

3. **Add native OTLP metrics export**: The absence of OTLP metrics export forces a parallel Prometheus scraping pipeline alongside OTLP push for traces and logs. Adding an OTLP metrics exporter (even as an opt-in alongside Prometheus) would unify the integration model and enable `service.name`-based correlation between metrics and traces.

4. **Structure access logs**: The current ALS implementation sends raw Envoy access log strings as unstructured log bodies. Adding structured log attributes (method, path, status code, duration, upstream cluster) as OTLP log record attributes — and setting a meaningful severity level — would transform access logs from a raw data dump into a structured, queryable signal.

### Notable observations

- **Envoy self-tracing**: The Envoy sidecar in the backend pod generates spans for its outbound calls to the OTel Collector itself (`otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318/*`), because the collector is also in the mesh. These "meta-spans" appear in the trace data and can confuse operators. Istio's documentation does not mention this behavior.

- **`service.name` inconsistency within traces**: The same backend service appears with three different `service.name` values in a single trace: `otel-eval-backend.demo` (Envoy sidecar), `otel-eval-backend` (Node.js SDK), and `istio-envoy-sidecars` (Prometheus metrics). This inconsistency is a significant barrier to unified service-level observability.

- **275 metrics, 70% Envoy internals**: The default metric collection captures all Envoy internal stats (`envoy_cluster_http2_*`, `envoy_cluster_circuit_breakers_*`, etc.) alongside the operationally relevant `istio_*` metrics. This creates a high-volume, low-signal-to-noise default that most operators will need to filter. Istio does not provide guidance on which metrics to keep.

- **No `service.version` anywhere**: Despite Istio tracking canonical revision (`istio.canonical_revision` on spans, `source_canonical_revision`/`destination_canonical_revision` on metrics), `service.version` is never set as a resource attribute. This limits version-based traffic analysis in OTel-native tooling.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with `file` exporter in a local kind cluster running Istio 1.29.2
- The `k8sattributes` processor was used; native vs enriched attributes were distinguished by identifying which attributes Envoy sets in OTLP payloads vs which are added post-collection
- Semantic conventions were checked against the current stable OpenTelemetry specification (HTTP semconv 1.24+)
- Istio documentation at `istio.io` was reviewed for integration guidance, metric references, and change management practices
- Traffic was generated via 40 HTTP requests through the Istio Ingress Gateway to the `otel-eval-backend` service
- The test backend (Node.js with OTel SDK) was sidecar-injected, producing a mixed Envoy + OTel SDK trace tree that reveals context propagation behavior
