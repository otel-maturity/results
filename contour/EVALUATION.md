# OpenTelemetry Support Maturity Evaluation: Contour

## Project overview

- **Project**: Contour — CNCF Incubating ingress controller for Kubernetes using Envoy as its data plane
- **Version evaluated**: 1.32.1 (Envoy 1.34.1)
- **Evaluation date**: 2025-05-09
- **Cluster**: otel-eval-contour
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 1 | OTLP gRPC for traces; Prometheus-only for metrics; no log export |
| Semantic Conventions | 0 | Deprecated HTTP attributes (`http.method`, `http.url`, `http.status_code`); non-standard custom keys |
| Resource Attributes & Configuration | 1 | `service.name` present but no `service.version`; no `OTEL_*` env var support; identity inconsistent across signals |
| Trace Modeling & Context Propagation | 2 | Coherent ingress→egress→backend trace tree; W3C Trace Context propagated end-to-end |
| Multi-Signal Observability | 1 | Traces (OTLP) and metrics (Prometheus) both flow; no structured logs; no cross-signal correlation |
| Audience & Signal Quality | 1 | Span names reflect Envoy internals; useful operational attributes present; some noise |
| Stability & Change Management | 1 | Tracing documented; no formal telemetry stability contract; changes not explicitly versioned |

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC (Envoy → OTel Collector via `ExtensionService` CRD)
- **Metrics**: Flowing — Prometheus scrape (OTel Collector Prometheus receiver scraping Contour :8000 and Envoy :8002)
- **Logs**: Not flowing — stdout only; no OTLP log export from Contour or Envoy

### Resource attributes (native, before collector enrichment)

Contour/Envoy emits these resource attributes natively (confirmed from OTLP trace payloads before k8sattributes enrichment):

| Attribute | Value (example) |
|-----------|-----------------|
| `service.name` | `contour` (configured in Contour configmap) |
| `telemetry.sdk.language` | `cpp` |
| `telemetry.sdk.name` | `envoy` |
| `telemetry.sdk.version` | `c435eeccd4201f8d6a200922b166f5dcee08272b/1.34.1/Clean/RELEASE/BoringSSL` |

**Absent natively**: `service.version`, `service.instance.id`, `service.namespace`, any Kubernetes resource attributes.

### Resource attributes (after collector enrichment)

After k8sattributes processor enrichment, the following are added:

- `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.daemonset.name` (`contour-envoy`)
- `k8s.namespace.name` (`projectcontour`)
- `k8s.node.name`
- `k8s.pod.label.*` (multiple Helm and app labels)
- `container.id`

---

## Dimension evaluations

### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

- **Traces — OTLP gRPC**: Envoy exports traces via OTLP gRPC to the OTel Collector. Configuration requires:
  1. Creating an `ExtensionService` CRD (projectcontour.io/v1alpha1) pointing at the Collector gRPC endpoint
  2. Adding a `tracing.extensionService` reference to the Contour configmap
  3. Restarting Contour to apply the change
  - This is documented in the official Contour tracing guide and in the design document (PR #5043).

- **Metrics — Prometheus scrape only**: Both Contour (`:8000/metrics`) and Envoy (`:8002/stats/prometheus`) expose Prometheus endpoints. There is no OTLP metrics export path. The OTel Collector must scrape these via its Prometheus receiver. The official guide (`guides/prometheus.md`) describes this as the primary integration path.

- **Logs — stdout only**: No OTLP log export is supported. Contour and Envoy write to stdout in default format.

- **Mixed integration surface**: Traces use OTLP push; metrics use Prometheus pull. These are separate configuration paths with no unified mechanism.

- **ExtensionService pattern**: The `ExtensionService` CRD (normally used for external authorization) is repurposed as the mechanism to reference the OTel Collector for tracing. This is project-specific glue that is not obvious to users unfamiliar with Contour's CRD model.

- **No `OTEL_*` env var support**: The Contour control plane binary does not use the OTel Go SDK and does not respect `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, or other standard OTel environment variables. Configuration is entirely through the Contour configmap or `ContourConfiguration` CRD.

#### Checklist assessment

- ✅ OTLP is supported (for traces)
- ✅ Users can connect to an existing OTel Collector without sidecars (for traces)
- ❌ OTLP is not the default or clearly recommended path for metrics (Prometheus is the only option)
- ❌ No OTLP log export
- ❌ Configuration is not via standard `OTEL_*` environment variables
- ❌ The integration surface is inconsistent across signals (OTLP for traces, Prometheus for metrics, nothing for logs)
- ❌ Legacy Prometheus integration is the only metrics path, not positioned as secondary

#### Rationale

Contour explicitly supports OTLP for traces and has documentation for it. However, metrics remain Prometheus-only (a legacy pull model), logs are not collected, and the configuration mechanism is project-specific (configmap + ExtensionService CRD) rather than OTel-native. This matches Level 1 (OpenTelemetry-Aligned): OTLP is present alongside legacy approaches, and the integration is signal-inconsistent with lingering project-specific patterns.

---

### 2. Semantic Conventions

**Level: 0 — Instrumented**

#### Evidence

##### Trace attributes (Envoy ingress span — `name=ingress`)

Attributes emitted natively by Envoy on the `ingress` span:

| Attribute | Value (example) | OTel Status |
|-----------|-----------------|-------------|
| `http.method` | `"GET"` | **Deprecated** — current: `http.request.method` |
| `http.url` | `"http://otel-eval.local/"` | **Deprecated** — current: `url.full` |
| `http.status_code` | `"200"` (string) | **Deprecated** — current: `http.response.status_code` (int) |
| `http.protocol` | `"HTTP/1.1"` | **Deprecated** — current: `network.protocol.version` |
| `peer.address` | `"127.0.0.1"` | Non-standard (not OTel semconv) |
| `upstream_cluster` | `"demo/otel-eval-backend/3000/da39a3ee5e"` | Envoy-internal format, not semconv |
| `upstream_cluster.name` | `"demo_otel-eval-backend_3000"` | Envoy-internal, not semconv |
| `upstream_address` | `"10.244.0.6:3000"` | Non-standard |
| `component` | `"proxy"` | Deprecated OTel attribute |
| `downstream_cluster` | `"-"` | Envoy-internal |
| `node_id` | `"contour-envoy-jx2v6"` | Envoy-internal |
| `zone` | `""` | Envoy-internal |
| `guid:x-request-id` | UUID | Envoy-internal |
| `user_agent` | `"curl/8.18.0"` | Non-standard key (should be `user_agent.original`) |
| `response_flags` | `"-"` | Envoy-internal |
| `request_size` | `"0"` (string) | Non-standard |
| `response_size` | `"249"` (string) | Non-standard |
| `podName` | `"contour-envoy-jx2v6"` | Non-standard (camelCase, project-specific) |
| `podNamespace` | `"projectcontour"` | Non-standard (project-specific) |

**Key issues**:
- All HTTP attributes use the **deprecated** OTel HTTP semconv (`http.method`, `http.url`, `http.status_code`, `http.protocol`). Current stable conventions are `http.request.method`, `url.full`, `http.response.status_code`, `network.protocol.version`.
- `http.status_code` is emitted as a **string** (`"200"`), not an integer as required by the current semconv.
- The span name `ingress` and `router demo_otel-eval-backend_3000 egress` are Envoy-internal names, not semantic operation names.
- No `http.route` attribute is present on ingress spans, which would be the most useful routing context for an ingress controller.
- Custom attributes (`podName`, `podNamespace`) use camelCase rather than OTel-style dot-notation.

##### Metric names and attributes

Contour and Envoy use Prometheus naming conventions:
- `contour_*` prefix for Contour control plane metrics (e.g., `contour_dagrebuild_total`, `contour_httpproxy_valid`)
- `envoy_*` prefix for Envoy data plane metrics (e.g., `envoy_cluster_upstream_rq`, `envoy_http_downstream_rq_total`)

Metric attribute keys use Envoy/Prometheus conventions (`envoy_cluster_name`, `envoy_response_code`, etc.), not OTel semantic conventions (`net.peer.name`, `http.response.status_code`). These are converted from Prometheus format by the OTel Collector's Prometheus receiver, which adds a schema URL (`https://opentelemetry.io/schemas/1.18.0`) from the receiver, not from the project itself.

##### Log attributes

No structured log export. Not applicable.

#### Checklist assessment

- ❌ Current OTel HTTP semantic conventions are NOT used (`http.request.method` etc. are absent)
- ❌ Deprecated attributes are used: `http.method`, `http.url`, `http.status_code`, `http.protocol`, `component`
- ❌ `http.status_code` is emitted as a string, not an integer
- ❌ No `http.route` attribute despite being an ingress controller (the most relevant routing context)
- ❌ Span names reflect Envoy internals (`ingress`, `router demo_otel-eval-backend_3000 egress`) rather than semantic operations
- ❌ Metric names use Prometheus/Envoy conventions, not OTel semantic conventions
- ❌ No schemaUrl set on OTLP trace exports (the schema URL seen in metrics is from the Prometheus receiver, not the project)
- ⚠️ `peer.address` is used instead of `net.peer.ip`/`net.peer.port`

#### Rationale

The trace attributes are dominated by deprecated OTel HTTP attributes and Envoy-internal keys. The current stable OTel HTTP semantic conventions (`http.request.method`, `url.full`, `http.response.status_code`, `network.protocol.version`) are not used. This is primarily an Envoy limitation — Envoy's built-in OTel tracer uses the older attribute set — but it is the state of what Contour emits. The absence of `http.route`, the use of string-typed status codes, and the non-standard custom attribute naming all contribute to a Level 0 assessment. Users cannot interpret this telemetry using generic OTel knowledge; they need to know Envoy's internal attribute schema.

---

### 3. Resource Attributes & Configuration

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Native resource attributes

From the OTLP trace payloads (Envoy → Collector), the Contour/Envoy source natively emits:

| Attribute | Present | Notes |
|-----------|---------|-------|
| `service.name` | ✅ `"contour"` | Configured via `serviceName` in Contour configmap |
| `service.version` | ❌ | Not emitted |
| `service.instance.id` | ❌ | Not emitted |
| `service.namespace` | ❌ | Not emitted |
| `telemetry.sdk.language` | ✅ `"cpp"` | Envoy-native |
| `telemetry.sdk.name` | ✅ `"envoy"` | Envoy-native |
| `telemetry.sdk.version` | ✅ commit hash + version | Not human-friendly |

**Absent**: `service.version` is not set despite Contour knowing its own version (visible in `contour_build_info` metric with label `version=v1.32.1`). `service.instance.id` is not set, making it impossible to distinguish multiple Envoy pods in traces without relying on collector-added `k8s.pod.name`.

##### OTEL_* environment variable support

The Contour control plane binary does **not** use the OTel Go SDK and does **not** respect `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_EXPORTER_OTLP_ENDPOINT`, or any other standard OTel environment variables. Configuration is exclusively through the Contour configmap (`tracing.serviceName`, `tracing.extensionService`) or the `ContourConfiguration` CRD.

Envoy itself (the actual trace emitter) also does not support `OTEL_*` env vars for its tracing configuration — it receives tracing config dynamically via xDS from Contour.

##### Identity consistency across signals

| Signal | `service.name` | `service.version` | Identity source |
|--------|---------------|-------------------|-----------------|
| Traces | `"contour"` | Absent | Contour configmap `serviceName` field |
| Metrics | `"contour"` / `"envoy"` | Absent | Prometheus receiver job names |
| Logs | N/A | N/A | stdout only |

The `service.name` for metrics is derived from the Prometheus receiver job name (`contour` and `envoy`), not from the project itself. The trace `service.name` is `"contour"` regardless of whether it's the control plane or data plane emitting the telemetry — the Contour control plane doesn't emit OTLP traces at all; only Envoy does, but it uses `service.name=contour`. This creates a conceptual mismatch: the `service.name` in traces refers to Envoy's activity, not Contour's control plane.

#### Checklist assessment

- ✅ `service.name` is present and stable on traces
- ❌ `service.version` is absent (despite being available in `contour_build_info`)
- ❌ `service.instance.id` is absent
- ❌ `OTEL_*` environment variables are not respected
- ❌ Identity is inconsistent: metrics use job-name-derived `service.name` (`contour`/`envoy`), traces use configmap-configured `service.name` (`contour`)
- ❌ `service.name=contour` in traces refers to Envoy's behavior, not the Contour control plane — potentially confusing
- ⚠️ Kubernetes attributes are pipeline-derived (via k8sattributes processor), which is acceptable per the model

#### Rationale

`service.name` is present and stable, which is the minimum for Level 1. However, the absence of `service.version` and `service.instance.id`, the lack of `OTEL_*` env var support, and the identity inconsistency between signals (traces vs. metrics using different derivation mechanisms) place this at Level 1 rather than Level 2. The configuration mechanism is entirely project-specific, not OTel-native.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

A complete trace for a proxied HTTP request forms a coherent parent-child tree:

```
ingress [kind=SERVER, parentSpanId=null]  ← Envoy root span (service.name=contour)
  └── router demo_otel-eval-backend_3000 egress [kind=CLIENT, parentSpanId=ingress.spanId]
        └── GET / [kind=SERVER, parentSpanId=egress.spanId]  ← Backend root span (service.name=otel-eval-backend)
              ├── middleware - query [kind=INTERNAL]
              ├── middleware - jsonParser [kind=INTERNAL]
              ├── middleware - expressInit [kind=INTERNAL]
              ├── middleware - <anonymous> [kind=INTERNAL]
              └── request handler - / [kind=INTERNAL]
```

Verified from trace `2c998507040586548d8c8bedd4631b9a`:
- Envoy `ingress` span: `spanId=1c3767f137bf6d91`, `parentSpanId=null`
- Envoy `egress` span: `spanId=54424e65f9812201`, `parentSpanId=1c3767f137bf6d91`
- Backend `GET /` span: `spanId=d37cea5f43581c8e`, `parentSpanId=54424e65f9812201`

All 20 Envoy traces share trace IDs with corresponding backend traces — **100% of the traffic-generated traces form complete cross-service trees**.

##### Context propagation

- **W3C Trace Context** (`traceparent` header) is propagated by Envoy to upstream services. When a client sends a `traceparent` header, Envoy reads it and uses it as the parent span.
- Verified: trace `4bf92f3577b34da6a3ce929d0e0e4736` (a known injected trace ID) appears in both Envoy and backend spans with the correct parent-child relationship to the injected `parentSpanId=00f067aa0ba902b7`.
- Envoy also propagates `traceparent` downstream (to the backend), enabling the backend to join the trace as a child.

##### Span kinds

- Envoy ingress spans: `kind=2` (SERVER) — correct for an entry-point ingress span
- Envoy egress spans: `kind=3` (CLIENT) — correct for an upstream call
- Backend spans: `kind=2` (SERVER) for root, `kind=1` (INTERNAL) for middleware

The span kind assignments are semantically correct.

##### Trace coherence

The trace topology accurately reflects the request flow: client → Envoy (ingress) → Envoy (egress/router) → backend. This is the expected mental model for an ingress controller. The egress span name encodes the upstream cluster (`router demo_otel-eval-backend_3000 egress`), which provides routing context.

#### Checklist assessment

- ✅ W3C Trace Context is supported and propagated consistently
- ✅ Parent-child relationships are correct for ingress→egress→backend flows
- ✅ Span kinds are semantically correct (SERVER at ingress, CLIENT at egress)
- ✅ Context propagation works end-to-end (client → Envoy → backend)
- ✅ Traces represent meaningful logical operations (ingress request + upstream call)
- ✅ All 20 Envoy traces share trace IDs with backend traces (100% correlation rate)
- ⚠️ Span names are Envoy-internal (`ingress`, `router ... egress`) rather than semantic operation names
- ⚠️ Trace topology for error paths, retries, or TLS termination not verified

#### Rationale

The trace modeling is clearly intentional and correct for the synchronous ingress proxy use case. The ingress→egress→backend parent-child chain is coherent, span kinds are correct, and W3C Trace Context propagation works end-to-end. This is the strongest dimension for Contour. The design document explicitly describes the trace modeling goals, and the implementation delivers on them. The span naming (using Envoy-internal names) prevents reaching Level 3, but the structural correctness and propagation behavior meet Level 2 characteristics.

---

### 5. Multi-Signal Observability

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Signal availability

| Signal | Status | Export method | First-class? |
|--------|--------|---------------|--------------|
| Traces | ✅ Flowing | OTLP gRPC (Envoy → Collector) | Yes (documented) |
| Metrics | ✅ Flowing | Prometheus scrape (Collector → Contour/Envoy) | Yes (documented) |
| Logs | ❌ Not flowing | stdout only | No (no export path) |

**9 trace batches** and **100 metric batches** were observed in the evaluation cluster. Logs file is empty (0 bytes).

##### Cross-signal correlation

- **Traces ↔ Metrics**: No shared attributes between trace spans and metric data points. The `service.name=contour` appears in both, but metric labels use Envoy/Prometheus naming (`envoy_cluster_name`, `envoy_response_code`) while trace attributes use a different schema. There is no `trace_id` or `span_id` on metrics.
- **Traces ↔ Logs**: No correlation possible — logs are not exported.
- **Metrics ↔ Logs**: No correlation possible — logs are not exported.

##### Collection model

- **Traces**: OTLP push (Envoy → Collector gRPC). Configured via Contour configmap + ExtensionService CRD.
- **Metrics**: Prometheus pull (Collector scrapes Contour and Envoy endpoints). Configured via Collector scrape_configs.
- **Logs**: Not collected. Stdout only.

The two working signals use different collection models (push vs. pull) and different configuration mechanisms, requiring users to set up both independently.

##### Version info in metrics

The `contour_build_info` metric carries `version=v1.32.1` as a label, which is a useful operational signal. However, this version is not surfaced in traces as `service.version`, preventing cross-signal version correlation.

#### Checklist assessment

- ✅ Traces and metrics are both present and documented
- ❌ Logs are absent (no OTLP export)
- ❌ No cross-signal correlation (no `trace_id` in metrics, no logs)
- ❌ Signals use different collection models (push vs. pull) with no unified mechanism
- ❌ Users cannot pivot from a metric to a trace without manual effort and domain knowledge
- ❌ Level 2 requires all three signals as first-class outputs — logs absence blocks progression

#### Rationale

Two of three signals are present, which meets Level 1. However, the absence of structured log export, the incompatible collection models, and the lack of any cross-signal correlation mechanism (shared attributes, trace context in metrics) mean signals coexist rather than work together. This is a clear Level 1 profile: "multiple signals are emitted, often through OpenTelemetry, but they are still loosely connected."

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Envoy emits two span types:
- `ingress` — generic name for all inbound requests (no HTTP method, path, or route context in the name)
- `router demo_otel-eval-backend_3000 egress` — encodes the Envoy cluster name in the span name, which is an internal identifier not meaningful to users

The span names reflect Envoy's internal component architecture rather than user-facing operations. A user-oriented name would be `GET /` or `HTTP GET otel-eval.local` (as the backend's Node.js instrumentation produces). The `ingress` span name does not distinguish between routes, methods, or paths without inspecting attributes.

##### Useful attributes present

Despite naming issues, the span attributes provide useful operational context:
- `upstream_cluster.name` (`demo_otel-eval-backend_3000`) — identifies the backend cluster
- `http.status_code` (`"200"`) — response status (though as a string, not int)
- `upstream_address` (`10.244.0.6:3000`) — upstream pod IP:port
- `response_flags` — Envoy response flags (e.g., `-` for no flags, `UH` for upstream unhealthy)
- `podName` / `podNamespace` — identifies which Envoy pod handled the request
- `peer.address` — client IP

These attributes are useful for debugging but require knowledge of Envoy's attribute schema to interpret.

##### Signal-to-noise ratio

The ingress span has 18 attributes, several of which are low-value in normal operation:
- `zone: ""` — empty string
- `downstream_cluster: "-"` — Envoy default when no downstream cluster is configured
- `node_id` — duplicates `podName`
- `guid:x-request-id` — Envoy-internal request ID (useful for log correlation but not standard)

The metric set is large (554 unique metric names total: 14 `contour_*`, 469 `envoy_*`). The Envoy metrics are comprehensive but noisy for typical operational use — most users need only a small subset (request rates, error rates, latency histograms).

##### Default usability

An operator familiar with OTel but not Envoy would need to learn Envoy's attribute schema to interpret traces. The span names and attribute keys are not self-explanatory. The metrics require knowing which `envoy_*` metrics correspond to which operational concerns.

#### Checklist assessment

- ✅ Some user-relevant information is present (status codes, upstream info, pod identity)
- ✅ Obvious noise is partially reduced (Contour configmap `includePodDetail: true` adds useful context)
- ❌ Span names reflect internal Envoy components, not user-facing operations
- ❌ Users need Envoy-specific knowledge to interpret attribute names and values
- ❌ `http.status_code` as a string requires type conversion in dashboards
- ❌ No `http.route` attribute on ingress spans (the most operationally useful routing context)
- ❌ 469 Envoy metrics with no guidance on which are SLO-relevant

#### Rationale

Some effort has been made to include useful operational context (pod identity via `includePodDetail`, upstream cluster info), but the telemetry is primarily shaped by Envoy's internal perspective. Span names are internal component names, deprecated attribute keys require OTel knowledge to recognize as deprecated, and the metric volume is high without guidance. This matches Level 1: "some effort is made to reduce noise and improve clarity, but telemetry is still largely shaped by internal perspectives."

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

Contour has a dedicated [Tracing Support](https://projectcontour.io/docs/1.32/config/tracing/) documentation page and a [Prometheus metrics guide](https://projectcontour.io/docs/main/guides/prometheus/). The tracing configuration options are documented (serviceName, overallSampling, customTags, etc.). A [design document](https://github.com/projectcontour/contour/blob/main/design/tracing-design.md) exists describing the architectural decisions.

However, the telemetry behavior itself (span names, attribute names, metric names) is **not documented as a contract**. The documentation describes how to configure tracing, not what telemetry will be emitted.

##### Change communication

Release notes do not include dedicated telemetry change sections. The recent releases (v1.32.x, v1.33.x) reviewed do not contain telemetry-specific changelog entries. Telemetry changes from Envoy upgrades (e.g., new attributes, changed attribute semantics) are not explicitly called out.

##### Schema URL presence

No `schemaUrl` is set on OTLP trace exports from Envoy. The schema URL `https://opentelemetry.io/schemas/1.18.0` seen in metrics is added by the OTel Collector's Prometheus receiver, not by the project.

##### Stability guarantees

There are no explicit stability commitments for telemetry. The [Contour Deprecation Policy](https://projectcontour.io/resources/deprecation-policy/) covers API and configuration, but does not mention telemetry signals.

#### Checklist assessment

- ✅ Tracing configuration is documented
- ✅ A design document exists describing the tracing architecture
- ❌ Span names, attribute names, and metric names are not documented as a contract
- ❌ No schemaUrl on OTLP trace exports
- ❌ Telemetry changes are not explicitly called out in release notes
- ❌ No distinction between stable and experimental telemetry
- ❌ The deprecation policy does not cover telemetry signals
- ❌ Users cannot safely build dashboards and alerts without risk of breakage on upgrade

#### Rationale

Contour has made an intentional effort to document tracing configuration and has a design document, which shows awareness that telemetry is a user-facing concern. However, the emitted telemetry behavior is not treated as a contract — there are no commitments about span names, attribute stability, or metric naming. This matches Level 1: "maintainers are aware that telemetry changes have impact, but handling is informal and inconsistent."

---

## Key findings

### Strengths

1. **Excellent trace correlation**: The ingress→egress→backend trace tree is coherent and complete. W3C Trace Context propagation works end-to-end, with 100% of traffic-generated traces forming proper parent-child chains across the ingress boundary. This is the strongest OTel capability Contour offers.

2. **Correct span kinds**: Envoy correctly emits `SERVER` spans for inbound requests and `CLIENT` spans for upstream calls, which is semantically correct and enables proper trace visualization in observability backends.

3. **Documented tracing architecture**: Contour has a dedicated tracing documentation page, a design document, and configurable options (service name, sampling rates, custom tags, pod detail). The ExtensionService-based configuration, while unusual, is documented.

4. **Rich Prometheus metrics**: Both Contour (14 control-plane metrics) and Envoy (469 data-plane metrics) expose comprehensive Prometheus endpoints. The `contour_build_info` metric includes version information, and Envoy metrics provide detailed request/connection/circuit-breaker statistics.

5. **Custom tag support**: The tracing configuration supports adding custom span tags from literal values or request headers, enabling users to enrich traces with routing-specific context without modifying Contour itself.

### Areas for improvement

1. **Adopt current OTel HTTP semantic conventions**: Replace deprecated attributes (`http.method` → `http.request.method`, `http.url` → `url.full`, `http.status_code` → `http.response.status_code` as integer, `http.protocol` → `network.protocol.version`). Add `http.route` to ingress spans. This is primarily an Envoy upstream change, but Contour could advocate for or configure it.

2. **Add `service.version` and `service.instance.id` to trace resource attributes**: Contour already knows its version (visible in `contour_build_info`). Emitting `service.version=v1.32.1` as a resource attribute would enable version-based filtering and correlation. `service.instance.id` (using pod UID) would distinguish individual Envoy pods.

3. **Add OTLP metrics export or document the Prometheus→OTLP pipeline as a supported integration**: Currently there is no OTLP metrics path. Either adding native OTLP metrics export or explicitly documenting the Prometheus receiver pipeline as the supported integration contract would improve the integration surface.

4. **Structured log export**: Envoy access logs could be configured in JSON format and collected via the OTel Collector filelog receiver, enabling log-trace correlation. Contour's control-plane logs could similarly be structured for collection.

5. **Treat telemetry as a versioned contract**: Document the emitted span names, attribute names, and metric names. Add telemetry-specific sections to release notes. Set `schemaUrl` on OTLP exports. Consider extending the deprecation policy to cover telemetry signals.

### Notable observations

1. **Tracing via xDS (not bootstrap)**: Contour configures Envoy's tracing dynamically via xDS (Listener Discovery Service), not via the Envoy bootstrap JSON. The bootstrap file has no tracing section. This is an unusual and sophisticated pattern that enables runtime reconfiguration but makes the tracing setup non-obvious.

2. **ExtensionService repurposed for tracing**: The `ExtensionService` CRD (normally for external authorization backends) is repurposed as the mechanism to reference the OTel Collector endpoint for tracing. This is the only supported mechanism. Users unfamiliar with Contour's CRD ecosystem may find this confusing.

3. **`service.name=contour` from Envoy**: Traces come from Envoy (the data plane), but the `service.name` is configured as `contour` in the Contour configmap. The Contour control plane itself emits no OTLP traces. This means `service.name=contour` in traces represents Envoy's activity, not the control plane's — a conceptual mismatch that could confuse users.

4. **Bitnami chart configmap vs. CRD mode**: The Bitnami Helm chart uses `--config-path` (configmap mode), not the `ContourConfiguration` CRD. The CRD exists but is not read at startup. Tracing configuration must go in the configmap. This distinction is not obvious from the documentation, which primarily describes the CRD approach.

5. **`http.status_code` as string**: Envoy emits the HTTP status code as a string value (`"200"`) rather than an integer. The current OTel semantic convention specifies `http.response.status_code` as an integer. This type mismatch will cause issues with tools that expect numeric status codes for range queries or aggregations.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file export (`fileexporter`) in a local kind cluster (`otel-eval-contour`)
- The k8sattributes processor was used to enrich telemetry with Kubernetes metadata; native vs. enriched resource attributes were distinguished by comparing trace resource attributes against what the project emits before enrichment
- Semantic conventions were checked against the current stable OpenTelemetry specification (HTTP semconv v1.21+)
- Documentation was reviewed from `projectcontour.io/docs/1.32/` and the GitHub repository (`projectcontour/contour`)
- The Contour source code (`go.mod`, `internal/metrics/metrics.go`) was reviewed to confirm the absence of OTel Go SDK usage in the control plane
- Traffic was generated with and without injected `traceparent` headers to verify W3C Trace Context propagation
- 9 trace batches (20 Envoy spans + 244 backend spans) and 100 metric batches were analyzed
