# OpenTelemetry Support Maturity Evaluation: kgateway

## Project overview

- **Project**: kgateway — Envoy-based API gateway and Kubernetes Gateway API control plane (formerly Gloo, CNCF Sandbox)
- **Version evaluated**: v2.2.4
- **Evaluation date**: 2026-05-10
- **Cluster**: otel-eval-kgateway
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 1 | OTLP supported for traces and logs (data plane); metrics are Prometheus-only with no OTLP push path |
| Semantic Conventions | 0 | Envoy emits deprecated HTTP attributes (`http.method`, `http.url`, `http.status_code`) with wrong types; no current semconv alignment |
| Resource Attributes & Configuration | 1 | `service.name` present but inconsistent across signals; `service.version` absent; no `OTEL_*` env var support |
| Trace Modeling & Context Propagation | 2 | Coherent ingress/egress span tree; W3C Trace Context propagated end-to-end with confirmed backend correlation |
| Multi-Signal Observability | 2 | All three signals present and correlated; logs carry `traceId`/`spanId` matching trace spans |
| Audience & Signal Quality | 1 | Span names (`ingress`, `router kube_demo_otel-eval-backend_3000 egress`) partially useful; Envoy-internal naming leaks into span names and attributes |
| Stability & Change Management | 1 | Observability documented in official docs with OTel guide; no explicit telemetry stability contract or changelog sections for telemetry changes |

---

## Telemetry overview

### Signals observed
- **Traces**: Flowing — OTLP gRPC to collector port 4317 (from Envoy data plane)
- **Metrics**: Flowing — Prometheus scrape (controller on port 9092, Envoy proxy on port 9091)
- **Logs**: Flowing — OTLP gRPC to collector port 4317 (Envoy OTel ALS access logs)

### Resource attributes (native, before collector enrichment)

**Envoy data plane (traces and logs):**
- `service.name: kgateway-envoy.demo` (traces, configurable via `ListenerPolicy.tracing.serviceName`)
- `service.name: demo-gateway.demo` (logs, Envoy's default cluster name format — differs from traces)
- `telemetry.sdk.name: envoy`
- `telemetry.sdk.language: cpp`
- `telemetry.sdk.version: e0ac044aa6c7f599bdd83caabb19c009b77a613a/1.36.5/Distribution/RELEASE/BoringSSL`
- `log_name: envoy-access-log` (logs)
- `cluster_name: demo-gateway.demo` (logs)
- `node_name: demo-gateway-7689fbcf7-mc2rc.demo` (logs)
- `zone_name: ""` (logs)

**kgateway controller (metrics, via Prometheus scrape — collector-assigned `service.name`):**
- `service.name: kgateway-controller` (assigned by Prometheus receiver, not the controller itself)
- `service.name: kgateway-envoy-proxy` (assigned by Prometheus receiver for Envoy metrics)

### Resource attributes (after collector enrichment)

The k8sattributes processor added the following to traces and logs:
- `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.namespace.name`, `k8s.node.name`
- `k8s.deployment.name`, `k8s.replicaset.name`
- `k8s.container.name`
- `k8s.pod.label.*`, `k8s.pod.annotation.*`
- `container.id` (traces only)

For metrics, the Prometheus receiver added:
- `service.instance.id` (from scrape target IP:port)
- `server.address`, `server.port`, `url.scheme`
- All k8s attributes via k8sattributes

---

## Dimension evaluations

### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

kgateway supports OTLP gRPC for two of its three signals natively:

- **Traces (OTLP gRPC)**: Envoy data plane sends traces via OTLP gRPC to a configurable BackendRef (Kubernetes Service). Confirmed flowing to collector port 4317. Configured via `ListenerPolicy.spec.default.httpSettings.tracing.provider.openTelemetry.grpcService`.
- **Logs (OTLP gRPC)**: Envoy access logs sent via OTel ALS (OTLP gRPC) to the same collector endpoint. Configured via `ListenerPolicy.spec.default.httpSettings.accessLog[].openTelemetry.grpcService`.
- **Metrics (Prometheus scrape only)**: Neither the kgateway controller nor the Envoy data plane supports OTLP metrics export. Both expose Prometheus endpoints (`/metrics` on port 9092 for controller, `/metrics` on port 9091 for Envoy proxy). The official docs acknowledge this and recommend using the OTel Collector as a Prometheus scraper to bridge into the OTel ecosystem.

Configuration is done via kgateway-specific CRDs (`ListenerPolicy`, `GatewayParameters`), not via standard `OTEL_*` environment variables. There is no single unified configuration path — traces and logs are configured via `ListenerPolicy`, metrics are scraped passively.

The official documentation has a dedicated "Set up the OpenTelemetry stack" guide (`/docs/envoy/latest/observability/otel-stack/`) that positions OTel as the recommended observability integration path, covering all three signals. However, the guide itself uses the Collector as a Prometheus scraper for metrics, making OTLP a partial story.

A `ReferenceGrant` is required for cross-namespace BackendRefs (e.g., when the `ListenerPolicy` in `demo` namespace references a collector Service in `opentelemetry` namespace). This is standard Gateway API policy but adds integration friction.

#### Checklist assessment

- ✅ OTLP is supported for traces and logs
- ✅ OTel Collector is the documented integration path
- ✅ Official docs have a dedicated OTel integration guide
- ❌ Metrics have no OTLP push path — Prometheus scrape only
- ❌ Configuration is via project-specific CRDs, not `OTEL_*` environment variables
- ❌ No single unified telemetry configuration surface across all three signals
- ❌ Cross-namespace setup requires `ReferenceGrant` (additional friction)

#### Rationale

kgateway clearly supports OpenTelemetry and positions it as the recommended path, earning it above Level 0. However, the absence of OTLP for metrics — the highest-volume signal — means OTel is not yet the *primary* integration surface for all signals. The Prometheus-only metrics path is explicitly called out in the official docs as the default, with OTel acting as a collection bridge. Configuration is via project-specific CRDs rather than standard `OTEL_*` mechanisms. This is characteristic of **Level 1 (OTel-Aligned)**: OTel works and is recommended, but legacy approaches (Prometheus scraping) remain first-class for a key signal.

---

### 2. Semantic Conventions

**Level: 0 — Instrumented**

#### Evidence

##### Trace attributes (Envoy data plane)

Envoy's OTel tracer emits the following span attributes on `ingress` and `router ... egress` spans:

| Attribute | Current OTel Semconv | Status |
|-----------|---------------------|--------|
| `http.url` | `url.full` | **Deprecated** (replaced in HTTP semconv v1.23+) |
| `http.method` | `http.request.method` | **Deprecated** (replaced in HTTP semconv v1.23+) |
| `http.status_code` | `http.response.status_code` | **Deprecated** AND wrong type: Envoy sends as `stringValue: "200"` instead of `intValue: 200` |
| `http.protocol` | `network.protocol.name` + `network.protocol.version` | **Non-standard** |
| `component` | No equivalent | **Proprietary** (value: `"proxy"`) |
| `upstream_cluster` | No equivalent | **Proprietary** (Envoy-internal cluster name) |
| `upstream_cluster.name` | No equivalent | **Proprietary** (duplicate of above) |
| `node_id` | No equivalent | **Proprietary** (Envoy node identifier) |
| `zone` | No equivalent | **Proprietary** |
| `downstream_cluster` | No equivalent | **Proprietary** |
| `peer.address` | `network.peer.address` | **Deprecated** (old OpenTracing convention) |
| `guid:x-request-id` | No equivalent | **Proprietary** (Envoy request ID header) |
| `response_flags` | No equivalent | **Proprietary** (Envoy-specific) |
| `user_agent` | `user_agent.original` | **Non-standard key** |

**Key finding**: All HTTP attributes use deprecated pre-v1.23 OpenTelemetry semantic conventions. `http.status_code` is additionally emitted as a string type instead of an integer. None of the current stable HTTP semantic conventions (`http.request.method`, `http.response.status_code`, `url.full`, `url.path`, `network.protocol.name`) are used.

##### Metric names and attributes

The kgateway controller emits `kgateway_*` prefixed metrics using Prometheus naming conventions (underscores, not dots). Examples:
- `kgateway_controller_reconcile_duration_seconds`
- `kgateway_translator_translations_total`
- `kgateway_xds_snapshot_resources`

These follow Prometheus naming conventions, which are reasonable but do not align with OTel metric naming conventions (which use dots as separators: `kgateway.controller.reconcile.duration`).

Envoy proxy metrics use `envoy_*` prefix with Prometheus naming. No OTel metric semantic conventions apply here as these are Envoy-native stats.

Metric attribute keys use Prometheus-style naming: `envoy_cluster_name`, `envoy_response_code`, `envoy_response_code_class`, `controller`, `gateway`, `namespace`, `resource`, `result`, `syncer`, `translator` — all project-specific, not aligned with OTel semconv.

##### Log attributes

Log records from Envoy OTel ALS have:
- Empty body (`{}`)
- No log attributes (empty attributes array)
- Trace correlation via `traceId`/`spanId` fields (correct OTel log data model)
- Resource attributes: `log_name`, `cluster_name`, `node_name`, `zone_name`, `service.name` — all Envoy-proprietary

No OTel log semantic conventions are applied (e.g., no `log.record.uid`, no severity mapping, no structured body).

#### Checklist assessment

- ❌ Current OTel HTTP semantic conventions not used (`http.request.method`, `http.response.status_code`, `url.full`)
- ❌ Deprecated attributes used: `http.method`, `http.url`, `http.status_code`, `peer.address`
- ❌ `http.status_code` emitted as string type instead of integer
- ❌ Metric names follow Prometheus convention (underscores), not OTel convention (dots)
- ❌ Log body is empty; no log semantic conventions applied
- ✅ Trace correlation fields (`traceId`, `spanId`) correctly placed in log records
- ❌ No documentation referencing OTel semantic conventions for telemetry design

#### Rationale

The use of deprecated HTTP semantic conventions is upstream Envoy behavior (Envoy's OTel tracer hasn't been updated to current semconv), and kgateway inherits this directly. However, kgateway does not document this deviation, does not provide any mapping to current conventions, and does not offer a migration path. The proprietary attributes (`component`, `upstream_cluster`, `node_id`, `response_flags`) are Envoy-internal and have no OTel semconv equivalents. The incorrect type for `http.status_code` (string vs. integer) is an additional quality issue. This pattern — implicit, ad-hoc, and primarily reflecting internal Envoy implementation details — is characteristic of **Level 0 (Instrumented)**.

---

### 3. Resource Attributes & Configuration

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Native resource attributes

- **Traces**: `service.name: kgateway-envoy.demo` — configurable via `ListenerPolicy.tracing.serviceName`. Present and meaningful. `telemetry.sdk.*` attributes correctly identify Envoy as the SDK.
- **Logs**: `service.name: demo-gateway.demo` — Envoy's default cluster name format. **Different from the trace `service.name`** for the same Envoy proxy pod. This is a significant identity inconsistency.
- **Metrics**: `service.name` is not emitted by the controller or proxy natively. The Prometheus receiver in the collector assigns `service.name: kgateway-controller` and `service.name: kgateway-envoy-proxy` based on scrape target labels — these are collector-derived, not project-native.
- **`service.version`**: Absent from all three signals. The Envoy proxy version is embedded in `telemetry.sdk.version` (e.g., `e0ac044aa6c7f599bdd83caabb19c009b77a613a/1.36.5/...`) but not expressed as `service.version`.
- **`service.instance.id`**: Not set natively. The Prometheus receiver derives `service.instance.id` from the scrape target IP:port (e.g., `10.244.0.8:9091`), which is a non-standard form.

##### OTEL_* environment variable support

No `OTEL_*` environment variable support was found or documented for kgateway. Telemetry configuration is exclusively through:
- kgateway CRDs (`ListenerPolicy`, `GatewayParameters`) for data plane telemetry
- Helm chart values for controller deployment settings

Standard variables like `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, and `OTEL_RESOURCE_ATTRIBUTES` have no effect on kgateway's telemetry behavior.

##### Identity consistency across signals

| Attribute | Traces (Envoy) | Metrics (Controller) | Metrics (Envoy) | Logs (Envoy) |
|-----------|---------------|---------------------|----------------|--------------|
| `service.name` | `kgateway-envoy.demo` (configurable) | `kgateway-controller` (collector-derived) | `kgateway-envoy-proxy` (collector-derived) | `demo-gateway.demo` (Envoy default, different from traces) |
| `service.version` | absent | absent | absent | absent |
| `service.instance.id` | absent | `10.244.0.7:9092` (collector-derived) | `10.244.0.8:9091` (collector-derived) | absent |

The `service.name` for the Envoy proxy differs between traces (`kgateway-envoy.demo`) and logs (`demo-gateway.demo`), making cross-signal correlation by service identity unreliable without manual knowledge of this discrepancy.

#### Checklist assessment

- ✅ `service.name` present in traces (configurable, meaningful)
- ❌ `service.name` inconsistent between traces and logs for the same Envoy process
- ❌ `service.name` not emitted natively by the controller (collector-derived)
- ❌ `service.version` absent from all signals
- ❌ `OTEL_*` environment variables not respected
- ❌ Configuration is via project-specific CRDs, not standard OTel mechanisms
- ✅ K8s attributes available via collector enrichment (standard pipeline)

#### Rationale

Some resource attributes exist and are meaningful (`service.name` in traces, `telemetry.sdk.*`), but behavior is inconsistent across signals. The `service.name` difference between traces and logs for the same Envoy pod is a concrete correlation failure. The absence of `service.version` and `OTEL_*` support are gaps. This is characteristic of **Level 1 (OTel-Aligned)**: resource attributes are present but not consistently applied, and configuration precedence is unclear or project-specific.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

Each HTTP request through the Envoy gateway produces the following trace tree (confirmed from telemetry data):

```
ingress (kind=SERVER, traceId=d374625072e87cb0d55ddab5c20eca14, spanId=565049b8d78c1488)
└── router kube_demo_otel-eval-backend_3000 egress (kind=CLIENT, parentSpanId=565049b8d78c1488)
    └── GET / (kind=SERVER, from otel-eval-backend, parentSpanId=c5d2265f209b85a9)
        ├── middleware - query (kind=INTERNAL)
        ├── middleware - expressInit (kind=INTERNAL)
        ├── middleware - jsonParser (kind=INTERNAL)
        ├── middleware - <anonymous> (kind=INTERNAL)
        └── request handler - / (kind=INTERNAL)
```

Key observations:
- **`ingress` span**: Root span, `kind=2` (SERVER) — correct for an entry-point HTTP listener span
- **`router ... egress` span**: Child of `ingress`, `kind=3` (CLIENT) — correct for an upstream connection span
- **Backend spans**: Correctly parented to the egress span; the backend `GET /` span has `parentSpanId` matching the egress span's `spanId`
- **Span links**: Not used (not needed for this synchronous flow)
- Enabled via `spawnUpstreamSpan: true` in `ListenerPolicy` — without this, only the `ingress` span is generated

##### Context propagation

**Confirmed end-to-end W3C Trace Context propagation:**
- Envoy injects `traceparent` header into upstream requests
- Backend (Node.js Express with OTel SDK) correctly reads and continues the trace
- All spans share the same `traceId` (e.g., `d374625072e87cb0d55ddab5c20eca14`)
- The backend's `GET /` span has `parentSpanId` matching the Envoy egress span's `spanId`

This is confirmed by the telemetry data: backend spans appear in the same trace tree as Envoy spans, with correct parent-child linkage across the proxy boundary.

##### Trace coherence

The trace structure clearly tells the story of a request: it enters the gateway (`ingress`), is forwarded upstream (`router ... egress`), and is handled by the backend (`GET /` + middleware spans). The trace is complete, coherent, and interpretable without internal knowledge.

#### Checklist assessment

- ✅ W3C Trace Context (`traceparent`) supported and propagated
- ✅ Parent-child relationships are correct for synchronous request flows
- ✅ Entry-point spans use SERVER kind; upstream spans use CLIENT kind
- ✅ Trace continues into downstream backend services
- ✅ `ingress` (root) → `router ... egress` (child) → backend spans (grandchildren)
- ✅ Traces tell a coherent, interpretable story
- ⚠️ `spawnUpstreamSpan: true` must be explicitly configured (not default) to get the egress span
- ⚠️ No schema URL set on trace exports

#### Rationale

Trace modeling is intentional and correct for the gateway use case. The parent-child relationships accurately reflect the request flow through the proxy and into the backend. W3C Trace Context propagation works end-to-end with confirmed evidence from the telemetry data. The trace structure is stable and interpretable. The need to explicitly enable `spawnUpstreamSpan` is a minor friction point, but the overall trace modeling is well-designed. This clearly meets **Level 2 (OTel-Native)**: trace modeling is intentional, context propagation works end-to-end, and traces represent meaningful logical operations.

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability

All three signals are present and flowing:

| Signal | Source | Protocol | Volume |
|--------|--------|----------|--------|
| Traces | Envoy data plane | OTLP gRPC | 2 trace batches (2 requests) |
| Metrics | kgateway controller | Prometheus scrape | 21 `kgateway_*` metrics + Go runtime |
| Metrics | Envoy data plane | Prometheus scrape | 153+ `envoy_*` metrics |
| Logs | Envoy OTel ALS | OTLP gRPC | 2 log records (1 per request) |

The controller's structured JSON logs (stdout) are not pushed to the collector — they remain in Kubernetes log streams only.

##### Cross-signal correlation

**Trace-log correlation confirmed:**
- Log record 1: `traceId=d374625072e87cb0d55ddab5c20eca14`, `spanId=565049b8d78c1488`
- Trace ingress span 1: `traceId=d374625072e87cb0d55ddab5c20eca14`, `spanId=565049b8d78c1488`
- ✅ Exact match — log records carry the correct `traceId` and `spanId` from the corresponding ingress span

**Trace-metric correlation:**
- `envoy_tracing_opentelemetry_spans_sent` counter confirms OTel tracing is active
- `envoy_access_logs_open_telemetry_access_log_logs_written` counter confirms OTel ALS is active
- Metric labels include `k8s.deployment.name: demo-gateway` and `k8s.namespace.name: demo` — same as trace resource attributes (after enrichment)
- However, native metric labels use Prometheus-style keys (`envoy_cluster_name`, etc.) not matching trace span attribute keys

##### Collection model

| Signal | Collection method | OTLP native? |
|--------|------------------|--------------|
| Traces | OTLP push (Envoy → Collector) | ✅ Yes |
| Logs | OTLP push (Envoy → Collector) | ✅ Yes |
| Metrics | Prometheus pull (Collector scrapes Envoy + Controller) | ❌ No |

The asymmetry in collection model (OTLP push for traces/logs, Prometheus pull for metrics) is a design constraint. The official docs explicitly acknowledge this and recommend using the OTel Collector as a Prometheus scraper to normalize the metrics into the OTel pipeline.

#### Checklist assessment

- ✅ All three signals are present and flowing
- ✅ Logs include `traceId` and `spanId` matching trace spans (confirmed exact match)
- ✅ Metrics confirm OTel trace and log activity (`envoy_tracing_opentelemetry_spans_sent`, `envoy_access_logs_open_telemetry_access_log_logs_written`)
- ✅ Official docs describe all three signals together in the OTel guide
- ⚠️ Metrics use Prometheus pull, not OTLP push — asymmetric collection model
- ⚠️ `service.name` differs between traces and logs for the same Envoy process (see Dimension 3)
- ⚠️ Controller stdout logs are not collected into the OTel pipeline
- ⚠️ Log body is empty — all access log data is in resource attributes and trace correlation fields

#### Rationale

kgateway achieves genuine multi-signal observability: all three signals flow, and trace-log correlation is confirmed with exact `traceId`/`spanId` matching. The official documentation treats observability as a cohesive system across all three signals. The metrics-via-Prometheus gap is a real limitation but is bridged by the OTel Collector in the documented setup. The log body being empty is a quality issue (Dimension 6) but doesn't break the multi-signal story. This meets **Level 2 (OTel-Native)**: signals are intentionally correlated, logs include trace context, and the observability system is designed as a coherent whole.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Envoy span names:
- `ingress` — generic, not informative about the specific request (no HTTP method or path)
- `router kube_demo_otel-eval-backend_3000 egress` — exposes Envoy-internal cluster naming format (`kube_<namespace>_<service>_<port>`), not user-friendly

Comparison with current OTel HTTP semconv recommendations:
- Current convention for server spans: `{http.request.method} {http.route}` → e.g., `GET /`
- Envoy emits `ingress` regardless of method or route — no request-level information in the span name

The backend (otel-eval-backend) correctly uses `GET /` as the span name, following OTel conventions. The contrast with Envoy's `ingress` is stark.

##### Signal-to-noise ratio

**Metrics**: The Envoy proxy emits 153+ unique metric names. This is extremely comprehensive — covering circuit breakers, cluster health, HTTP stats, connection stats, xDS stats, runtime stats, and more. For operators, this is powerful but requires significant domain knowledge to navigate. The kgateway controller emits 21 `kgateway_*` metrics that are well-named and operationally focused (reconciliation counts/durations, XDS snapshot stats, resource sync stats).

**Traces**: Low volume (2 spans per request with `spawnUpstreamSpan: true`). The trace structure is clear, but span attributes include low-value fields: `downstream_cluster: "-"` (always a dash for direct clients), `zone: ""` (empty in kind), `response_flags: "-"` (no flags set). The `guid:x-request-id` attribute uses a non-standard key format with a colon.

**Logs**: Log body is always empty (`{}`). All access log data is in resource attributes (`cluster_name`, `node_name`) and trace correlation fields. There are no log attributes at all — the structured access log data that Envoy collects (HTTP method, path, status, latency) is not included in the OTel log record body or attributes. This makes OTel access logs significantly less useful than Envoy's traditional text-based or JSON access logs.

##### Default usability

- Operators can identify request flow from traces (ingress → egress → backend)
- Span names require Envoy knowledge to interpret (`ingress` = inbound listener, `router ... egress` = upstream cluster selection)
- The empty log body means OTel access logs cannot be used for request-level analysis without combining with traces
- Metric cardinality is very high (153+ metrics) but well-organized under Envoy's stat namespace hierarchy

#### Checklist assessment

- ✅ Traces tell a coherent story of request flow
- ✅ kgateway controller metrics are operationally focused and well-named
- ❌ Span names (`ingress`, `router kube_demo_otel-eval-backend_3000 egress`) expose Envoy internals
- ❌ No HTTP method or route in the ingress span name
- ❌ Log body is always empty — OTel access logs carry no request data in the body
- ❌ Low-value span attributes present: `downstream_cluster: "-"`, `zone: ""`, `response_flags: "-"`
- ❌ Non-standard attribute key format: `guid:x-request-id`
- ⚠️ 153+ Envoy metrics require domain expertise to interpret

#### Rationale

kgateway has made real effort in observability design — the multi-signal setup is documented, the trace structure is coherent, and controller metrics are operationally meaningful. However, the Envoy data plane's telemetry quality is constrained by upstream Envoy defaults: span names don't include request context, log bodies are empty, and several span attributes expose internal Envoy naming conventions. Operators need Envoy-specific knowledge to interpret the telemetry. This is characteristic of **Level 1 (OTel-Aligned)**: some signals are user-oriented (controller metrics, trace structure), while others remain shaped by internal implementation (Envoy span names, empty log bodies, internal cluster naming in egress spans).

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

kgateway has a dedicated observability section in its official documentation:
- `/docs/envoy/latest/observability/otel-stack/` — "Set up the OpenTelemetry stack" guide
- `/docs/envoy/latest/observability/control-plane-metrics/` — Control plane metrics reference
- `/docs/envoy/latest/observability/gateway-metrics/` — Gateway proxy metrics reference

The docs describe *how* to configure OTel telemetry but do not establish telemetry as a *stable contract*. There is no explicit statement about which telemetry fields are stable vs. experimental, no versioning of the telemetry schema, and no schema URL set in OTLP exports.

##### Change communication

The v2.2.4 release notes (reviewed) contain no mention of telemetry-specific changes. The breaking change in v2.2.4 is about Gateway API version compatibility, not telemetry. Prior releases mention observability improvements (e.g., "Add `trackRemaining` field to `BackendConfigPolicy` circuit breakers... enabling observability of remaining capacity") but these are feature additions, not telemetry contract changes.

The developer architecture docs (`devel/architecture/metrics.md`) describe the internal metrics implementation in detail, including naming conventions and best practices, but this is contributor-facing documentation, not a user-facing stability contract.

##### Schema URL presence

No `schemaUrl` is set on any OTLP export:
- Traces: `schemaUrl` field is empty/absent
- Metrics: Schema URL is set to `https://opentelemetry.io/schemas/1.18.0` — but this is added by the Prometheus receiver in the OTel Collector (collector-derived), not by kgateway itself
- Logs: `schemaUrl` field is empty/absent

##### Stability guarantees

No explicit stability guarantees for telemetry were found in the documentation or source code. Telemetry is not explicitly labeled as stable or experimental. The project is a CNCF Sandbox project, which implies the overall project API is still evolving.

#### Checklist assessment

- ✅ Observability is documented with a dedicated section in official docs
- ✅ OTel integration is positioned as the recommended path
- ❌ No schema URL set in native OTLP exports (traces, logs)
- ❌ No explicit stability classification for telemetry (stable vs. experimental)
- ❌ Release notes do not include dedicated telemetry change sections
- ❌ No migration guidance for telemetry changes
- ❌ Telemetry not treated as a public contract with versioning

#### Rationale

kgateway is aware that observability matters — it has dedicated documentation and an OTel integration guide. However, telemetry is not yet treated as a formal public contract. There is no schema URL, no stability labeling, and no process for communicating telemetry changes. The v2.2.4 release notes show that breaking changes are communicated for API/behavior changes but not for telemetry. This is characteristic of **Level 1 (OTel-Aligned)**: intent exists and documentation is present, but governance and stability practices are still emerging.

---

## Key findings

### Strengths

1. **Genuine multi-signal observability with confirmed trace-log correlation**: All three signals flow, and log records carry exact `traceId`/`spanId` values matching the corresponding ingress trace spans — enabling drill-down from logs to traces in supported backends.

2. **End-to-end W3C Trace Context propagation**: Envoy correctly injects `traceparent` headers into upstream requests, and the backend continues the trace with correct parent-child relationships. The full trace tree (Envoy ingress → Envoy egress → backend handler) is visible in a single trace.

3. **Rich Envoy data plane metrics with OTel-confirming counters**: 153+ `envoy_*` metrics provide deep operational visibility, including `envoy_tracing_opentelemetry_spans_sent` and `envoy_access_logs_open_telemetry_access_log_logs_written` which confirm the OTel pipeline is active and healthy.

4. **Well-documented OTel integration path**: The official docs have a dedicated "Set up the OpenTelemetry stack" guide covering all three signals with concrete configuration examples. OpenTelemetry is clearly positioned as the recommended observability integration path.

5. **Intentional trace span structure**: The `ingress` (SERVER) → `router ... egress` (CLIENT) span hierarchy correctly models the gateway's role as a proxy, with appropriate span kinds for entry-point and upstream connection spans.

### Areas for improvement

1. **Adopt current HTTP semantic conventions**: Envoy's OTel tracer emits deprecated HTTP attributes (`http.method`, `http.url`, `http.status_code`) instead of current ones (`http.request.method`, `url.full`, `http.response.status_code`). Additionally, `http.status_code` is emitted as a string instead of an integer. kgateway should track upstream Envoy's semconv migration and document the current state. Consider contributing to Envoy's OTel tracer to accelerate this update.

2. **Fix `service.name` inconsistency between traces and logs**: The Envoy proxy emits `service.name: kgateway-envoy.demo` in traces but `service.name: demo-gateway.demo` in logs. This breaks cross-signal correlation by service identity. The log `service.name` should match the configured trace `service.name` (or vice versa) so both signals identify the same service consistently.

3. **Populate OTel access log body and attributes**: Envoy OTel ALS log records have an empty body (`{}`) and no attributes, making them significantly less useful than traditional access logs. The HTTP request method, path, status code, latency, and other access log fields should be included in the log body or attributes. This would make OTel access logs a first-class signal for request-level analysis.

4. **Add OTLP metrics export path**: Neither the kgateway controller nor the Envoy data plane supports pushing metrics via OTLP. Adding an OTLP metrics exporter to the controller and documenting how to configure it would complete the OTel integration surface and eliminate the Prometheus-scrape dependency for metrics.

5. **Set `schemaUrl` in OTLP exports and add `service.version`**: Native OTLP exports (traces, logs) do not include a `schemaUrl`, making it impossible for consumers to know which semantic convention version is in use. Adding `service.version` to resource attributes (using the kgateway version) would improve identity completeness across signals.

### Notable observations

1. **Envoy metrics port is 9091, not 9901**: kgateway's deployer overrides Envoy's default admin port (9901) with a custom stats endpoint at port 9091 with Prometheus-format metrics. This is automatically annotated on the pod and works correctly, but differs from Envoy's documented default and may surprise operators familiar with standard Envoy deployments.

2. **`spawnUpstreamSpan: true` is not the default**: Without this `ListenerPolicy` setting, only the `ingress` span is generated — the upstream egress span is omitted. This means the default trace structure does not show the proxy-to-backend connection, limiting trace utility. The default should arguably be `true` for a proxy that exists specifically to forward requests.

3. **`service.name` in logs uses Envoy cluster name format**: The log `service.name: demo-gateway.demo` is derived from Envoy's internal cluster name format (`<gateway-name>.<namespace>`), not from any user-configurable field. This is an Envoy implementation detail leaking into the OTel resource identity.

4. **Metrics schema URL is collector-derived**: The `schemaUrl: https://opentelemetry.io/schemas/1.18.0` visible in metrics data is added by the OTel Collector's Prometheus receiver, not by kgateway. This gives the appearance of schema awareness that is actually collector-provided.

5. **Cross-namespace `ReferenceGrant` required**: When the `ListenerPolicy` and the OTel Collector are in different namespaces (a common production setup), a `ReferenceGrant` must be created in the collector's namespace. This is standard Gateway API policy but is an additional operational step that should be prominently documented.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file exporter in a local kind cluster (`otel-eval-kgateway`)
- The k8sattributes processor was used to distinguish native vs. enriched resource attributes; native attributes are those present before k8sattributes enrichment
- Semantic conventions were checked against the OpenTelemetry HTTP semantic conventions specification (stable, post-v1.23 migration)
- kgateway documentation at `kgateway.dev/docs/envoy/latest/` was reviewed for integration guidance and stability commitments
- GitHub release notes for v2.2.4 and recent releases were reviewed for telemetry-related change communication
- The kgateway developer architecture docs (`devel/architecture/metrics.md`) were reviewed for internal metrics design intent
- Traffic was generated via `kubectl port-forward` to the Envoy gateway service, producing 2 traced requests with full end-to-end trace propagation confirmed
