# OpenTelemetry Support Maturity Evaluation: Istio

## Project overview

- **Project**: Istio — CNCF-graduated service mesh providing traffic management, mTLS security, and observability for Kubernetes microservices via Envoy sidecar injection
- **Version evaluated**: 1.29.2
- **Evaluation date**: 2026-05-10
- **Cluster**: otel-eval-istio (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP gRPC traces are first-class; metrics are Prometheus-only with no OTLP path; logs are stdout-only |
| Semantic Conventions | 1 | Envoy spans use deprecated HTTP semconv (http.method, http.status_code, http.url, http.target); Istio-specific attributes are non-standard |
| Resource Attributes & Configuration | 2 | Envoy natively emits service.name + telemetry.sdk.* via OTLP; no service.version; not configurable via OTEL_* env vars |
| Trace Modeling & Context Propagation | 3 | Full W3C traceparent propagation confirmed; correct span kinds; coherent multi-hop traces across gateway and sidecar |
| Multi-Signal Observability | 1 | Only traces use OTLP; metrics require Prometheus scrape; logs are stdout with no trace context; no cross-signal correlation |
| Audience & Signal Quality | 2 | Trace span names are operator-legible Envoy routing strings; rich Istio metadata; high-value control plane metrics via Prometheus |
| Stability & Change Management | 1 | No schemaUrl on Envoy OTLP traces; Prometheus metrics have no schema URL; telemetry behavior documented but not as a formal contract |

---

## Telemetry overview

### Signals observed
- **Traces**: Flowing — OTLP gRPC (Envoy → OTel Collector at port 4317), 905 spans across 405 unique trace IDs in 14 export batches
- **Metrics**: Flowing — Prometheus scrape (collector scrapes port 15014 on istiod, port 15020/15090 on sidecars), 352 unique metric names in 751 export batches
- **Logs**: Not flowing via OTLP — Istio access logs are written to `/dev/stdout` in JSON format; 0 records in logs.jsonl

### Resource attributes (native, before collector enrichment)

Envoy's OTLP exporter natively sends only these resource attributes:

| Attribute | Example Value |
|-----------|---------------|
| `service.name` | `istio-ingressgateway.istio-ingress`, `otel-eval-backend.demo` |
| `telemetry.sdk.name` | `envoy` |
| `telemetry.sdk.language` | `cpp` |
| `telemetry.sdk.version` | `af30be60b7c35f2aceaea1b7382c7fbf12aa5e67/1.37.2-dev/Clean/RELEASE/BoringSSL` |

No `service.version`, `service.namespace`, `service.instance.id`, or `host.*` attributes are emitted natively by Envoy.

The Prometheus-scraped metrics carry only collector-defined labels (`app`, `namespace`, `pod`, `cluster_name`) as resource attributes — these are pipeline-derived, not native Istio output.

### Resource attributes (after collector enrichment)

The `k8sattributes` processor adds the following to trace resource attributes:

- `k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.deployment.name`, `k8s.replicaset.name`, `k8s.node.name`
- `k8s.container.name` (value: `istio-proxy`)
- `k8s.pod.label.*` (all pod labels, including Istio canonical labels)
- `k8s.pod.annotation.*` (all pod annotations, including Istio injection metadata)
- `host.name`, `host.arch`, `os.type`, `os.version`
- `process.*` (from the app container, not from Envoy itself)

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OTLP Native (partial)**

#### Evidence

**Traces — OTLP gRPC (Level 3 quality):**
- Istio 1.13+ supports native OTLP trace export via `meshConfig.extensionProviders` with an `opentelemetry` provider type.
- Configured via Helm `meshConfig.extensionProviders` and activated per-namespace with a `Telemetry` CR (`telemetry.istio.io/v1`).
- Envoy sidecars and the ingress gateway push spans directly to the collector's OTLP gRPC endpoint (port 4317).
- 905 spans confirmed flowing across 405 traces, covering both the ingress gateway and the demo service sidecar.
- The `Telemetry` CR supports `randomSamplingPercentage` — set to 100% for this evaluation.

**Metrics — Prometheus scrape only (Level 1):**
- Istio's primary metrics path is Prometheus exposition format on:
  - Port `15090` (`/stats/prometheus`) on each Envoy sidecar — raw Envoy stats
  - Port `15020` (`/stats/prometheus`) on each sidecar — merged endpoint (Envoy + app + istio-agent)
  - Port `15014` (`/metrics`) on istiod — control plane metrics
- No OTLP metrics export is available natively. The `meshConfig.extensionProviders` supports `opentelemetry` for traces and access logs but **not for metrics** as of Istio 1.29.
- The `Telemetry` CR `mesh-metrics` in this cluster uses `prometheus` as the provider, confirming there is no OTLP metrics provider in the extension system.
- 352 unique metrics scraped: `envoy_*`, `istio_agent_*`, `pilot_*`, `citadel_*`, `galley_*`, `istiod_*`, `sidecar_injection_*`.

**Logs — stdout only (Level 0):**
- Envoy access logs are written to `/dev/stdout` in JSON format (configured via `accessLogFile: /dev/stdout`, `accessLogEncoding: JSON`).
- No OTLP log export. While Istio's extension provider system supports an `envoyFileAccessLog` type with OTel formatting, OTLP log export via `extensionProviders` requires additional configuration not in the default path.
- Access log JSON fields: `authority`, `bytes_received`, `bytes_sent`, `duration`, `method`, `path`, `protocol`, `request_id`, `response_code`, `response_flags`, `start_time`, `upstream_cluster`, `upstream_host`, `user_agent`, `x_forwarded_for` — no `traceId` or `spanId` fields by default.
- 0 records in `logs.jsonl`.

#### Checklist assessment

- ✅ Project supports OTLP for at least one signal (traces via gRPC)
- ✅ OTLP is documented as the recommended tracing path for Istio 1.13+
- ❌ OTLP is not available for metrics (Prometheus-only)
- ❌ OTLP is not available for logs (stdout-only)
- ✅ The OTLP endpoint is configurable (service + port in meshConfig)
- ❌ Not all three signals flow via OTLP

#### Rationale

Level 2 reflects that OTLP is a first-class, documented, production-supported integration for traces, but metrics and logs require separate non-OTLP collection mechanisms. The project has a clear architectural reason (Envoy's Prometheus stats filter is deeply embedded), but the result is a split collection model that requires operators to run both OTLP receivers and Prometheus scrapers.

---

### 2. Semantic Conventions

**Level: 1 — Partial / Inconsistent**

#### Evidence

##### Trace attributes (Envoy spans)

All Envoy-generated spans use **deprecated** OpenTelemetry HTTP semantic conventions. The current stable conventions (`http.request.method`, `http.response.status_code`, `url.path`, `url.full`) are not used.

| Attribute found in traces | Status | Current semconv equivalent |
|---------------------------|--------|---------------------------|
| `http.method` | ⚠️ Deprecated | `http.request.method` |
| `http.status_code` | ⚠️ Deprecated | `http.response.status_code` |
| `http.url` | ⚠️ Deprecated | `url.full` |
| `http.target` | ⚠️ Deprecated | `url.path` |
| `http.flavor` | ⚠️ Deprecated | `network.protocol.version` |
| `http.host` | ⚠️ Deprecated | `server.address` |
| `http.scheme` | ⚠️ Deprecated | `url.scheme` |
| `http.user_agent` | ⚠️ Deprecated | `user_agent.original` |
| `net.host.ip` | ⚠️ Deprecated | `server.address` |
| `net.host.name` | ⚠️ Deprecated | `server.address` |
| `net.host.port` | ⚠️ Deprecated | `server.port` |
| `net.peer.ip` | ⚠️ Deprecated | `network.peer.address` |
| `net.peer.name` | ⚠️ Deprecated | `network.peer.address` |
| `net.peer.port` | ⚠️ Deprecated | `network.peer.port` |
| `net.transport` | ⚠️ Deprecated | `network.transport` |

These deprecated attributes originate from Envoy's OTel tracer implementation, which is tied to the upstream Envoy version (`1.37.2-dev`). This is an upstream dependency constraint, not an Istio-specific choice.

**Istio-specific (non-standard) span attributes:**

| Attribute | Description |
|-----------|-------------|
| `istio.mesh_id` | Mesh identifier (e.g., `cluster.local`) |
| `istio.cluster_id` | Cluster name (e.g., `Kubernetes`) |
| `istio.canonical_service` | Canonical service name |
| `istio.canonical_revision` | Service revision |
| `istio.namespace` | Kubernetes namespace |
| `node_id` | Envoy node ID (e.g., `sidecar~10.244.0.9~pod.ns~ns.svc.cluster.local`) |
| `upstream_cluster` | Envoy cluster name |
| `upstream_cluster.name` | Upstream cluster name |
| `upstream_address` | Upstream host:port |
| `peer.address` | Peer address |
| `response_flags` | Envoy response flags (e.g., `-`, `UF`) |
| `component` | Always `proxy` |
| `guid:x-request-id` | Envoy request ID |
| `downstream_cluster` | Downstream cluster |
| `zone` | Availability zone |

These are Istio/Envoy-specific attributes with no OTel semconv equivalent. They provide valuable mesh context but are not portable across observability tools.

##### Metric names and attributes

All metrics use Prometheus naming conventions (`snake_case`, `_total` suffix for counters, `_seconds`/`_bytes` units). No OTel metric naming conventions (e.g., `http.server.request.duration` histogram) are used.

- `envoy_cluster_*` — raw Envoy cluster stats (Prometheus format)
- `istio_agent_*` — istio-agent process and Go runtime metrics
- `pilot_*`, `citadel_*`, `galley_*` — istiod control plane metrics
- No `istio_requests_total` or `istio_request_duration_milliseconds` were present in the scraped data (these require the Envoy stats filter to be enabled with the `envoy.filters.http.wasm` stats plugin; the standard Telemetry v2 metrics were not flowing in this setup)

Metric attributes use non-semconv labels: `cluster_name`, `response_code`, `response_code_class`, `pod`, `namespace`, `app` — all Prometheus-native, not OTel semantic conventions.

##### Log attributes

Access logs are structured JSON to stdout. No OTel log body schema, severity levels, or attribute conventions are used. No `traceId`/`spanId` correlation fields present.

#### Checklist assessment

- ❌ HTTP span attributes do not use current stable semconv (`http.request.method`, `http.response.status_code`, etc.)
- ❌ Metric names do not follow OTel semconv naming
- ❌ No schemaUrl on Envoy OTLP exports (traces)
- ✅ `telemetry.sdk.name`, `telemetry.sdk.language`, `telemetry.sdk.version` are set correctly
- ✅ `service.name` is set (though in `<service>.<namespace>` format, not standard)
- ⚠️ Istio-specific attributes are custom, not mapped to semconv equivalents

#### Rationale

Level 1 reflects that Istio uses some OTel conventions (SDK attributes, service.name, OTLP transport) but the span-level HTTP attributes are uniformly deprecated, and metric names follow Prometheus conventions rather than OTel semconv. The deprecated attribute usage is largely inherited from Envoy's upstream OTel tracer implementation and cannot be easily changed at the Istio layer without upstream Envoy changes.

---

### 3. Resource Attributes & Configuration

**Level: 2 — Consistent Identity**

#### Evidence

##### Native resource attributes

Envoy's OTLP exporter natively emits a minimal but consistent resource attribute set:

| Attribute | Value pattern | Notes |
|-----------|---------------|-------|
| `service.name` | `<canonical_service>.<namespace>` | e.g., `istio-ingressgateway.istio-ingress`, `otel-eval-backend.demo` |
| `telemetry.sdk.name` | `envoy` | Correctly identifies the SDK |
| `telemetry.sdk.language` | `cpp` | Correct for Envoy |
| `telemetry.sdk.version` | Git hash + Envoy version string | Non-standard format (commit hash, not semver) |

The `service.name` format `<service>.<namespace>` is Istio's canonical service naming convention (derived from the pod's `service.istio.io/canonical-name` label). This is consistent across all spans for the same workload.

**Missing standard resource attributes (natively):**
- ❌ `service.version` — not set by Envoy
- ❌ `service.namespace` — embedded in `service.name` but not as a separate attribute
- ❌ `service.instance.id` — not set
- ❌ `host.name` — not set by Envoy (added by k8sattributes)

##### OTEL_* environment variable support

Istio/Envoy does **not** support `OTEL_*` standard environment variables for configuring the OTLP exporter. Instead, configuration is done via:
- `meshConfig.extensionProviders` (Helm values / IstioOperator)
- `Telemetry` CR (`telemetry.istio.io/v1`)
- `meshConfig.defaultProviders.tracing`

No `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`, or similar env vars are observed in the istiod or istio-proxy deployments.

##### Identity consistency across signals

- **Traces**: `service.name` = `<canonical_service>.<namespace>` (Envoy-native)
- **Metrics**: `service.name` = `istio-envoy-demo`, `istio-istiod`, `istio-envoy-sidecars` (collector-defined job names, not workload identity)
- **Logs**: No structured resource identity — stdout only

There is **no consistent identity** across signals. A trace from `otel-eval-backend.demo` cannot be correlated to metrics under `service.name=istio-envoy-demo` without additional pipeline configuration.

#### Checklist assessment

- ✅ `service.name` is set natively on all OTLP traces
- ✅ `telemetry.sdk.*` attributes are set correctly
- ❌ `service.version` is not set
- ❌ `service.namespace` is not set as a separate attribute
- ❌ `OTEL_*` env vars are not supported for configuration
- ❌ `service.name` is inconsistent between traces and metrics (different naming conventions)

#### Rationale

Level 2 because Envoy consistently emits `service.name` on all spans, and the `telemetry.sdk.*` attributes correctly identify the instrumentation. However, the lack of `service.version`, the non-standard `service.name` format (namespace-embedded), the absence of `OTEL_*` env var support, and the identity mismatch between traces and metrics prevent a higher score.

---

### 4. Trace Modeling & Context Propagation

**Level: 3 — Correct and Complete**

#### Evidence

##### Span structure

Traces flow through two Envoy proxies for each request:

1. **Ingress gateway** (`istio-ingressgateway.istio-ingress`):
   - Span: `otel-eval-backend.demo.svc.cluster.local:3000/*` — kind=`SERVER` (2) — root span (no parent)
   - Child span: `router outbound|3000||otel-eval-backend.demo.svc.cluster.local; egress` — kind=`CLIENT` (3)

2. **Sidecar** (`otel-eval-backend.demo`):
   - Span: `otel-eval-backend.demo.svc.cluster.local:3000/*` — kind=`SERVER` (2) — child of the egress span above

This models the request flow correctly: inbound to gateway (SERVER) → outbound routing (CLIENT) → inbound to sidecar (SERVER). Span kinds are semantically correct.

The app container (otel-eval-backend, using Node.js OTel SDK) also generates spans (`GET /`, `middleware - *`, `request handler - /`) as children of the sidecar SERVER span, creating a complete end-to-end trace.

##### Context propagation

**W3C Trace Context propagation confirmed:**

A test request was sent with `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`. Spans with `traceId=4bf92f3577b34da6a3ce929d0e0e4736` were found across:
- Ingress gateway Envoy spans (SERVER + CLIENT)
- Sidecar Envoy spans (SERVER)
- Application-layer spans (GET /, middleware spans)

The injected `parentSpanId=00f067aa0ba902b7` is correctly used as the parent for the ingress gateway's root span. This confirms full W3C `traceparent` propagation through the entire mesh.

##### Trace coherence

Each request generates a coherent 3-layer trace:
```
[ingress-gateway] otel-eval-backend.demo.svc.cluster.local:3000/* (SERVER, root)
  └─ [ingress-gateway] router outbound|...; egress (CLIENT)
       └─ [sidecar] otel-eval-backend.demo.svc.cluster.local:3000/* (SERVER)
            └─ [app] GET / (SERVER)
                 ├─ [app] middleware - query (INTERNAL)
                 ├─ [app] middleware - jsonParser (INTERNAL)
                 └─ [app] request handler - / (INTERNAL)
```

This is a complete, accurate representation of the request path through the service mesh.

#### Checklist assessment

- ✅ W3C Trace Context (`traceparent`) is propagated correctly
- ✅ Span kinds are semantically correct (SERVER for inbound, CLIENT for outbound)
- ✅ Parent-child relationships are correct across service boundaries
- ✅ External `traceparent` headers are honored (trace ID preserved)
- ✅ Traces tell a complete, coherent story of the request path
- ✅ Both gateway and sidecar spans are present in the same trace

#### Rationale

Level 3 is warranted. Istio/Envoy's trace modeling is excellent — correct span kinds, proper parent-child relationships, full W3C Trace Context support including honoring externally-injected trace IDs. The only caveat is that application-layer header forwarding is required (the app must forward incoming trace headers to outgoing requests), which is a known Istio limitation documented in their observability guide.

---

### 5. Multi-Signal Observability

**Level: 1 — Emerging**

#### Evidence

##### Signal availability

| Signal | Available | Export Method | OTLP? |
|--------|-----------|---------------|-------|
| Traces | ✅ Yes | OTLP gRPC (push) | ✅ Yes |
| Metrics | ✅ Yes | Prometheus scrape (pull) | ❌ No |
| Logs | ⚠️ Partial | stdout JSON | ❌ No |

Traces are the only first-class OTLP signal. Metrics require a separate Prometheus scraper. Access logs are unstructured stdout with no OTel transport.

##### Cross-signal correlation

- **Trace-to-log correlation**: ❌ Not possible. Access logs contain `request_id` (an Envoy UUID) but no `traceId` or `spanId`. There is no standard way to correlate an access log entry to a trace.
- **Trace-to-metric correlation**: ❌ Not possible by default. Metrics use Prometheus labels (`pod`, `namespace`, `app`) while traces use `service.name` in a different format. No shared trace context in metrics.
- **Log-to-metric correlation**: ❌ Not applicable (logs are stdout only).

##### Collection model

The collection model requires two separate pipelines:
1. **OTLP receiver** in the collector — receives Envoy trace spans pushed via gRPC
2. **Prometheus receiver** in the collector — scrapes sidecar and control plane metrics on a 15-second interval

This split model means operators must configure and maintain two different collection paths. There is no unified OTLP pipeline for Istio telemetry.

#### Checklist assessment

- ✅ At least one signal (traces) flows via OTLP
- ❌ Metrics do not flow via OTLP
- ❌ Logs do not flow via OTLP
- ❌ No trace context in access logs (no `traceId`/`spanId`)
- ❌ No shared resource identity between traces and metrics
- ❌ No cross-signal correlation possible out of the box

#### Rationale

Level 1 reflects that only traces use OTLP while the other two signals require separate, non-OTLP collection mechanisms. Cross-signal correlation is not possible without significant pipeline customization. This is a fundamental architectural gap in Istio's current telemetry model.

---

### 6. Audience & Signal Quality

**Level: 2 — Operator-Friendly**

#### Evidence

##### Span naming

Envoy span names follow the pattern `<upstream_cluster_fqdn>:<port>/*` for SERVER spans and `router <cluster_name>; egress` for CLIENT spans:

- `otel-eval-backend.demo.svc.cluster.local:3000/*` — meaningful to Kubernetes operators
- `router outbound|3000||otel-eval-backend.demo.svc.cluster.local; egress` — verbose but interpretable
- `GET /` (from app spans) — standard HTTP operation naming

These names are operational (describing the routing context) rather than internal code paths, making them useful for operators debugging traffic issues.

##### Signal-to-noise ratio

**Traces**: The 905 spans across 405 traces include some noise:
- Internal health check spans (`GET /health`) are included
- Node.js filesystem spans (`fs realpathSync`, `fs readFileSync`) from the app SDK add internal detail
- Envoy-to-metadata-service spans (`egress 169.254.169.254`, `egress 100.100.100.200`) are low-value infrastructure noise

**Metrics**: 352 unique metric names is a large surface. The raw `envoy_*` metrics (majority) are low-level Envoy internals that most operators won't need. The `istio_agent_*` metrics are mostly Go runtime stats. The high-value operational metrics (`pilot_*`, `citadel_*`) are a small fraction.

Notably, the standard Istio L7 traffic metrics (`istio_requests_total`, `istio_request_duration_milliseconds`) were **not observed** in the scraped data. These metrics require the Envoy stats filter (Telemetry v2 / WASM stats plugin) to be properly configured and generating traffic through the standard path. Their absence reduces the operational value of the metrics signal.

##### Default usability

**Traces**: Usable out of the box for mesh traffic observability. The Istio-specific attributes (`istio.canonical_service`, `istio.namespace`, `istio.mesh_id`, `response_flags`) provide valuable mesh context beyond standard HTTP attributes. The `guid:x-request-id` attribute enables correlation with Envoy access logs.

**Metrics**: The Prometheus metrics are well-established and have extensive documentation. However, the absence of `istio_requests_total` (the primary SLI metric) in the collected data is a significant gap.

#### Checklist assessment

- ✅ Span names describe logical operations (not internal code paths)
- ✅ Istio-specific attributes add valuable operational context
- ✅ Metrics are well-documented (Prometheus ecosystem)
- ⚠️ Standard L7 traffic metrics (`istio_requests_total`) not observed in collected data
- ⚠️ `telemetry.sdk.version` is a non-standard git hash string, not a semver
- ❌ High volume of low-value raw Envoy stats in metrics

#### Rationale

Level 2 because the trace signal is genuinely useful for operators — the span names are meaningful, the Istio-specific attributes provide rich mesh context, and W3C propagation enables end-to-end tracing. The metrics have broad coverage but the absence of the key L7 traffic metrics (`istio_requests_total`) and the noise from raw Envoy stats reduce overall quality.

---

### 7. Stability & Change Management

**Level: 1 — Basic Documentation**

#### Evidence

##### Documentation of telemetry behavior

Istio has documentation for its telemetry features:
- [Distributed Tracing](https://istio.io/latest/docs/tasks/observability/distributed-tracing/) — covers OTLP configuration
- [Metrics](https://istio.io/latest/docs/reference/config/metrics/) — documents standard metrics
- [Access Logs](https://istio.io/latest/docs/tasks/observability/logs/access-log/) — documents log format

The documentation describes how to configure telemetry but does not treat the telemetry output as a versioned, stable API contract. Attribute names and metric names can change with Envoy upgrades.

##### Change communication

- Istio release notes (e.g., [1.29 release notes](https://istio.io/latest/news/releases/1.29.x/announcing-1.29/)) mention telemetry changes when significant.
- The Telemetry API (`telemetry.istio.io/v1`) was promoted to v1 (stable) in Istio 1.23, providing API stability for the configuration surface.
- However, the telemetry **output** (span attributes, metric names) is not versioned independently of Envoy releases.

##### Schema URL presence

- **Traces (Envoy OTLP)**: ❌ No `schemaUrl` set at the resource or scope level in Envoy's OTLP exports
- **Metrics (Prometheus scrape)**: ❌ No `schemaUrl` from Prometheus-scraped metrics. The `schemaUrl: https://opentelemetry.io/schemas/1.18.0` present in the data comes from the OTel Collector's `k8sclusterreceiver`, not from Istio itself.
- **Logs**: Not applicable (no OTLP)

##### Stability guarantees

- The `Telemetry` CR API (`telemetry.istio.io/v1`) is stable.
- The `meshConfig.extensionProviders` configuration is stable.
- The telemetry output format (span attributes, metric labels) has no explicit stability guarantee separate from Envoy version compatibility.
- Breaking changes to telemetry output can occur with Envoy upgrades (e.g., when Envoy updates its OTel tracer from deprecated to current HTTP semconv).

#### Checklist assessment

- ✅ Telemetry is documented (tracing, metrics, access logs)
- ✅ Configuration API (`Telemetry` CR) is stable (v1)
- ❌ No `schemaUrl` in Envoy OTLP trace exports
- ❌ Telemetry output is not versioned or treated as a stable contract
- ❌ No explicit stability guarantees for span attributes or metric names
- ⚠️ Telemetry changes are communicated in release notes but not as breaking change notifications

#### Rationale

Level 1 reflects that Istio has good documentation and a stable configuration API, but the telemetry output itself has no schema versioning (no `schemaUrl`), no explicit stability contract, and is subject to change with Envoy version upgrades. The deprecated HTTP semconv attributes in use are a concrete example of a known migration that has not yet occurred.

---

## Key findings

### Strengths

1. **Excellent W3C Trace Context propagation**: Istio/Envoy correctly honors externally-injected `traceparent` headers, propagates trace context across gateway and sidecar boundaries, and generates coherent multi-hop traces. External trace IDs are preserved end-to-end, enabling integration with upstream tracing systems.

2. **Native OTLP gRPC trace export with a clean configuration API**: The `Telemetry` CR + `meshConfig.extensionProviders` provides a clean, Kubernetes-native way to configure OTLP tracing. The `Telemetry` CR API is stable (v1), sampling is configurable per-namespace, and the integration works without any sidecar code changes.

3. **Rich operational span attributes**: Envoy spans include Istio-specific metadata (`istio.canonical_service`, `istio.namespace`, `istio.mesh_id`, `response_flags`, `upstream_cluster`, `node_id`) that provides valuable mesh-level context for debugging traffic issues, even if these attributes are not OTel semconv-aligned.

### Areas for improvement

1. **Update Envoy's OTel tracer to use current HTTP semantic conventions**: All HTTP span attributes use deprecated conventions (`http.method`, `http.status_code`, `http.url`, `http.target`, `net.*`). This is an upstream Envoy issue but Istio should track and communicate the migration timeline. The fix would require Envoy to adopt the stable HTTP semconv (`http.request.method`, `http.response.status_code`, `url.path`, etc.).

2. **Add OTLP metrics export capability**: Metrics are Prometheus-only, requiring a separate scrape pipeline. Adding an OTLP metrics provider to `meshConfig.extensionProviders` (similar to the existing trace provider) would enable a unified collection model and allow `service.name` consistency between traces and metrics. This is a known gap in Istio's telemetry roadmap.

3. **Add `traceId`/`spanId` to access logs and set `schemaUrl` on OTLP exports**: Access logs should include trace context fields to enable correlation with distributed traces. The `%REQ(X-B3-TRACEID)%` or `%TRACEID%` Envoy access log formatter can be used. Additionally, Envoy's OTLP exporter should set a `schemaUrl` to enable tooling to understand the semconv version in use.

### Notable observations

- **`istio_requests_total` was absent from collected metrics**: The standard Istio L7 traffic metrics (the primary SLI metrics for mesh traffic) were not present in the scraped data. These require the Envoy stats filter (Telemetry v2 WASM plugin) to generate them. The `mesh-metrics` Telemetry CR was applied, but the metrics did not appear on ports 15020 or 15090 for the demo workload. This may be a configuration timing issue or require traffic to flow after the Telemetry CR is applied.

- **`service.name` format is non-standard**: Envoy sets `service.name` to `<canonical_service>.<namespace>` (e.g., `otel-eval-backend.demo`), embedding the namespace in the service name rather than using the standard `service.namespace` attribute. This deviates from OTel resource semconv and makes it harder to use `service.name` as a direct workload identifier in backends that expect standard `service.name` values.

- **`telemetry.sdk.version` is a git hash, not a semver**: The value `af30be60b7c35f2aceaea1b7382c7fbf12aa5e67/1.37.2-dev/Clean/RELEASE/BoringSSL` is an Envoy build identifier, not a standard version string. This makes it difficult to correlate the SDK version to a specific Envoy release or track semconv migration progress.

- **The Prometheus-scraped metrics have no resource identity consistency with traces**: Metrics arrive with `service.name` values like `istio-envoy-demo` (the collector job name), while traces use `otel-eval-backend.demo`. Without additional pipeline transformations, these signals cannot be joined in a backend.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector (v0.150.1) with file exporters (`file/traces`, `file/metrics`, `file/logs`) in a kind cluster running Istio 1.29.2
- The `k8sattributes` processor was used; native vs. enriched attributes were distinguished by identifying attributes present in Envoy's OTLP resource (`service.name`, `telemetry.sdk.*`) vs. those added by the processor (`k8s.*`, `host.*`, `os.*`, `process.*`)
- Semantic conventions were checked against the current stable OpenTelemetry specification (HTTP semconv v1.24+)
- The Prometheus receiver scraped istiod (port 15014), Envoy sidecars (port 15020 via `prometheus.io/scrape` annotations), and demo namespace pods (port 15090 via direct pod IP)
- Traffic was generated via the Istio ingress gateway with both plain requests and requests with injected `traceparent` headers to test W3C Trace Context propagation
- Documentation review covered istio.io observability guides, Telemetry API reference, and meshConfig reference for Istio 1.29
