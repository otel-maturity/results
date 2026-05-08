# OpenTelemetry Support Maturity Evaluation: Contour

## Project overview

- **Project**: Contour — a CNCF Kubernetes ingress controller that uses Envoy as its data plane. The Contour control plane translates Kubernetes Ingress, HTTPProxy (CRD), and Gateway API resources into Envoy xDS configuration; Envoy handles all data-plane traffic.
- **Version evaluated**: v1.33.4 (official rendered manifests from `ghcr.io/projectcontour/contour:v1.33.4`, Envoy `distroless-v1.35.10`)
- **Evaluation date**: 2026-05-08
- **Cluster**: otel-eval-contour (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 1 | OTLP traces via Envoy's native OTel exporter; metrics only via Prometheus scrape; no OTLP metrics or logs |
| Semantic Conventions | 0 | Deprecated OpenTracing-style HTTP attributes throughout; Envoy-specific non-standard keys; no current OTel semconv |
| Resource Attributes & Configuration | 1 | `service.name` set via project config (not `OTEL_SERVICE_NAME`); no `service.version`; OTEL_* env vars not respected |
| Trace Modeling & Context Propagation | 2 | W3C Trace Context propagated correctly; coherent ingress→egress parent-child structure; E2E traces verified |
| Multi-Signal Observability | 1 | Traces (OTLP) and metrics (Prometheus scrape) present; logs absent; no cross-signal correlation context |
| Audience & Signal Quality | 1 | Span names are Envoy-internal (`ingress`, `router … egress`); Envoy-specific attributes require domain knowledge |
| Stability & Change Management | 1 | Tracing documented; no telemetry-specific changelog entries; no schema URL; no stability guarantees stated |

---

## Telemetry overview

### Signals observed
- **Traces**: Flowing — OTLP gRPC (Envoy → OTel Collector via `ExtensionService` CRD), `service.name=contour`
- **Metrics**: Flowing — Prometheus scrape via collector `prometheusreceiver`; two sources: `contour-control-plane` (port 8000) and `envoy-data-plane` (port 8002/`/stats/prometheus`)
- **Logs**: Not flowing — stdout only; no OTLP log export supported

### Resource attributes (native, before collector enrichment)

From trace data (`service.name=contour` resource block, pre-k8sattributes):

```
service.name:          contour
telemetry.sdk.name:    envoy
telemetry.sdk.language: cpp
telemetry.sdk.version: 3542e3464a2662423065a6ec854905b25955a09e/1.35.10/Clean/RELEASE/BoringSSL
```

No `service.version`, `service.instance.id`, or `service.namespace` are emitted natively. The `telemetry.sdk.version` is an Envoy build hash, not a semantic version.

For metrics, the resource is constructed entirely by the Prometheus receiver and k8sattributes processor — there are no native OTLP resource attributes from the project itself.

### Resource attributes (after collector enrichment)

After k8sattributes processing, the trace resource gains:

```
k8s.pod.name:                    envoy-r5dcc
k8s.pod.uid:                     3973cb14-962d-4f29-bb7c-0aacfb79d1c1
k8s.pod.start_time:              2026-05-08T10:01:53Z
k8s.pod.label.app:               envoy
k8s.pod.label.controller-revision-hash: 678db7cb94
k8s.pod.label.pod-template-generation: 2
k8s.pod.annotation.kubectl.kubernetes.io/restartedAt: 2026-05-08T10:01:52Z
k8s.namespace.name:              projectcontour
k8s.node.name:                   otel-eval-contour-control-plane
k8s.daemonset.name:              envoy
```

Metrics resource attributes are entirely collector-derived (Prometheus receiver + k8sattributes): `server.address`, `server.port`, `service.instance.id`, `service.name` (set by scrape job name), plus all `k8s.*` attributes.

---

## Dimension evaluations

### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

- **Traces**: Envoy natively exports OTLP gRPC spans when configured via the `ExtensionService` CRD and a `tracing:` block in the Contour ConfigMap. This is a first-class, documented feature. The OTLP transport is h2c (plaintext gRPC) to a user-provided collector endpoint. Confirmed flowing: 4 JSONL lines of trace batches observed.
- **Metrics**: No OTLP metrics export exists. Both Contour and Envoy expose only Prometheus `/metrics` (Contour, port 8000) and `/stats/prometheus` (Envoy, port 8002) endpoints. The collector must scrape these via the Prometheus receiver. This is the only supported metrics integration path, and it requires non-trivial configuration (separate services, non-default metrics path for Envoy).
- **Logs**: No OTLP log export. Logs go to stdout only. No structured log export pipeline is documented or supported.
- **Configuration path**: Tracing is configured via Contour-specific constructs (`ExtensionService` CRD + ConfigMap `tracing:` block), not via `OTEL_*` environment variables. There is no OTLP endpoint configuration via standard OTel mechanisms.
- **Documentation**: The [Tracing Support](https://projectcontour.io/docs/1.33/config/tracing/) page explicitly covers OTel integration. The [Prometheus guide](https://projectcontour.io/docs/1.33/guides/prometheus/) covers metrics scraping. OpenTelemetry is acknowledged as the recommended tracing path, but metrics remain Prometheus-only.
- **Setup friction**: Significant — requires creating an `ExtensionService` CRD, a Contour ConfigMap update, and restarting both the Contour deployment and Envoy DaemonSet. Cross-namespace routing requires a manual `ClusterIP` Service + `Endpoints` workaround because `ExtensionService` does not support cross-namespace references.

#### Checklist assessment

- ✅ OTLP is supported for traces
- ✅ OpenTelemetry is documented as the tracing integration path
- ❌ OTLP is not the default — tracing is off by default; must be explicitly enabled
- ❌ Metrics are Prometheus-only; no OTLP metrics path
- ❌ Logs have no OTLP path
- ❌ `OTEL_EXPORTER_OTLP_ENDPOINT` and other standard OTel env vars are not respected
- ❌ Users cannot connect to an existing OTel Collector without project-specific CRD configuration
- ❌ Integration surface is inconsistent across signals (OTLP for traces, scrape for metrics, nothing for logs)

#### Rationale

OTLP traces are supported and documented, placing the project clearly above Level 0. However, OTLP is not the default or primary integration surface — it requires explicit, project-specific CRD configuration. Metrics remain Prometheus-scrape-only with no OTLP path. Logs have no OTel integration at all. The integration surface is signal-inconsistent, which is a defining characteristic of Level 1 ("OpenTelemetry is supported, but not central"). The project does not yet reach Level 2 because OTLP is not the default, `OTEL_*` env vars are not respected, and users cannot reuse a single collector configuration across signals without significant custom work.

---

### 2. Semantic Conventions

**Level: 0 — Instrumented**

#### Evidence

##### Trace attributes (Envoy spans — `service.name=contour`)

All HTTP span attributes observed on `ingress` and `router … egress` spans use **deprecated OpenTracing-style attribute names**, not the current stable OTel semantic conventions:

| Attribute observed | Current OTel semconv equivalent | Status |
|---|---|---|
| `http.url` | `url.full` | ❌ Deprecated |
| `http.method` | `http.request.method` | ❌ Deprecated |
| `http.status_code` | `http.response.status_code` | ❌ Deprecated |
| `http.protocol` | `network.protocol.name` + `network.protocol.version` | ❌ Deprecated |
| `peer.address` | `network.peer.address` | ❌ Non-standard / OpenTracing |
| `upstream_cluster` | No OTel equivalent (Envoy-internal) | ❌ Proprietary |
| `upstream_cluster.name` | No OTel equivalent (Envoy-internal) | ❌ Proprietary |
| `upstream_address` | No OTel equivalent (Envoy-internal) | ❌ Proprietary |
| `response_flags` | No OTel equivalent (Envoy-internal) | ❌ Proprietary |
| `request_size` | `http.request.body.size` | ❌ Non-standard |
| `response_size` | `http.response.body.size` | ❌ Non-standard |
| `user_agent` | `user_agent.original` | ❌ Deprecated form |
| `node_id` | No OTel equivalent (Envoy-internal) | ❌ Proprietary |
| `podName` | `k8s.pod.name` (camelCase is non-standard) | ❌ Non-standard casing |
| `podNamespace` | `k8s.namespace.name` (camelCase is non-standard) | ❌ Non-standard casing |
| `component` | No OTel equivalent | ❌ OpenTracing legacy |
| `downstream_cluster` | No OTel equivalent | ❌ Envoy-internal |
| `guid:x-request-id` | No OTel equivalent | ❌ Envoy-internal |
| `zone` | No OTel equivalent | ❌ Envoy-internal |

Not a single attribute on Envoy's OTel spans uses a current stable OTel semantic convention. All HTTP attributes use the deprecated pre-1.0 OpenTracing-era names. Several attributes are entirely Envoy-proprietary with no OTel mapping.

The instrumentation scope name is `envoy` (version = build hash), not an OTel-conventional scope name.

##### Metric names and attributes

Metrics are scraped via the Prometheus receiver. The 655 unique metric names observed use Prometheus naming conventions (`contour_*`, `envoy_*`), not OTel semantic conventions. No OTel metric naming (`http.server.request.duration`, `http.server.active_requests`, etc.) is used. Metric attribute keys (`envoy_response_code`, `envoy_cluster_name`, etc.) are Envoy-internal Prometheus labels, not OTel semconv.

The Prometheus receiver sets no `schemaUrl` on metrics. The k8sclusterreceiver (which produces cluster-level k8s metrics) does set `schemaUrl: https://opentelemetry.io/schemas/1.18.0`, but this is collector-side, not project-native.

##### Log attributes

No OTLP logs are emitted. Not applicable.

#### Checklist assessment

- ❌ No current OTel semantic conventions used in traces (`http.request.method`, `url.full`, `http.response.status_code` are all absent)
- ❌ All HTTP attributes use deprecated OpenTracing-era names
- ❌ Multiple Envoy-proprietary attributes with no OTel mapping (`upstream_cluster`, `response_flags`, `node_id`, `guid:x-request-id`)
- ❌ No OTel metric naming conventions
- ❌ No schema URL on trace or metric data from the project itself
- ❌ Users need Envoy-specific knowledge to interpret most span attributes

#### Rationale

This is a clear Level 0. Every HTTP attribute on Envoy's OTel spans uses deprecated pre-1.0 attribute names, and several attributes are entirely Envoy-proprietary with no OTel semantic mapping. This is not a partial migration — there is no use of current stable OTel HTTP semantic conventions at all. The root cause is architectural: Envoy's built-in OTel tracer was implemented when the OpenTracing-to-OTel migration was incomplete, and it has not been updated to the stable OTel HTTP semconv. This is a known upstream Envoy limitation, not a Contour-specific choice, but it nonetheless means the telemetry does not align with OTel semantic conventions.

---

### 3. Resource Attributes & Configuration

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Native resource attributes

Envoy emits these resource attributes natively on trace data:
- `service.name: contour` — set via `serviceName` in the Contour ConfigMap `tracing:` block
- `telemetry.sdk.name: envoy`
- `telemetry.sdk.language: cpp`
- `telemetry.sdk.version: 3542e3464a2662423065a6ec854905b25955a09e/1.35.10/Clean/RELEASE/BoringSSL` (Envoy build hash)

Missing from native resource attributes:
- `service.version` — absent; no Contour/Envoy version is emitted as a resource attribute
- `service.instance.id` — absent
- `service.namespace` — absent

##### OTEL_* environment variable support

Contour's configuration reference documents only one environment variable: `CONTOUR_NAMESPACE`. There is no mention of `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_EXPORTER_OTLP_ENDPOINT`, or any other standard OTel env var in the Contour documentation. The `service.name` for traces is configured via the project-specific `serviceName` field in the ConfigMap `tracing:` block, not via `OTEL_SERVICE_NAME`. The Contour deployment manifest only sets `CONTOUR_NAMESPACE` and `POD_NAME` environment variables — no OTel env vars.

##### Identity consistency across signals

Identity is **inconsistent across signals**:
- Traces: `service.name=contour` (configured in Contour ConfigMap)
- Metrics (Contour control plane): `service.name=contour-control-plane` (set by Prometheus scrape job name in collector config)
- Metrics (Envoy data plane): `service.name=envoy-data-plane` (set by Prometheus scrape job name in collector config)

The `service.name` values differ between traces and metrics, and the metrics `service.name` is entirely collector-derived (from the scrape job name), not emitted by the project. This means there is no shared identity across signals that would enable natural cross-signal correlation.

#### Checklist assessment

- ✅ `service.name` is set as a resource attribute on traces
- ✅ `telemetry.sdk.*` attributes are present
- ❌ `service.version` is absent
- ❌ `service.instance.id` is absent
- ❌ `OTEL_SERVICE_NAME` is not respected
- ❌ `OTEL_RESOURCE_ATTRIBUTES` is not respected
- ❌ Identity is inconsistent across signals (traces vs metrics use different `service.name` values)
- ❌ Metrics resource identity is entirely collector-derived, not project-native

#### Rationale

The project sets `service.name` as a resource attribute on traces (satisfying the minimum for Level 1), but `service.version` is absent, `OTEL_*` env vars are not respected, and identity is inconsistent across signals. The `service.name` for traces is configured through a project-specific mechanism (ConfigMap `tracing.serviceName`), not standard OTel configuration. This is characteristic of Level 1: resource attributes exist but are not consistently applied, and configuration precedence between project settings and `OTEL_*` variables is unclear (in fact, `OTEL_*` variables are simply not supported). Level 2 requires stable identity across all signals and respect for `OTEL_SERVICE_NAME`/`OTEL_RESOURCE_ATTRIBUTES`, neither of which is present.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

Each HTTP request through Envoy produces exactly two spans with a correct parent-child relationship:

1. **`ingress`** (kind=`SERVER`/2) — root span representing the inbound HTTP request entering Envoy. This is the entry-point span.
2. **`router demo_otel-eval-backend_3000 egress`** (kind=`CLIENT`/3) — child span representing the upstream request to the backend service. Parent span ID matches the `ingress` span ID.

This two-span structure is correct and intentional: SERVER span for inbound, CLIENT span for outbound. The span kinds are semantically correct per OTel conventions.

##### Context propagation

W3C Trace Context (`traceparent` header) propagation is confirmed working end-to-end:

**Without inbound traceparent**: Envoy creates a new root `ingress` span with no `parentSpanId`. The `router … egress` span is a child of `ingress`. The backend receives the `traceparent` injected by Envoy, and the backend's `GET /` span has `parentSpanId` matching the Envoy egress span. Full trace verified for `traceId=9c401bbf69e50ccf7c53415e688bce63`:

```
ingress (Envoy, SERVER, root)  spanId=92aad59b30dfb15d
  └── router demo_otel-eval-backend_3000 egress (Envoy, CLIENT)  spanId=fa5106a03ed26abe
        └── GET / (backend, SERVER)  spanId=c215ff2f34c0745c
              ├── middleware - query
              ├── middleware - expressInit
              ├── middleware - jsonParser
              ├── middleware - <anonymous>
              └── request handler - /
```

**With inbound traceparent** (`traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`): Envoy's `ingress` span correctly sets `parentSpanId=00f067aa0ba902b7`, continuing the caller's trace. Multiple `ingress` spans all show `parentSpanId=00f067aa0ba902b7` and `traceId=4bf92f3577b34da6a3ce929d0e0e4736`, confirming W3C Trace Context is honored.

##### Trace coherence

Traces are coherent and tell a complete story from Envoy ingress through backend processing. The E2E trace connects the ingress controller and the upstream service in a single trace tree. This is the core value proposition of distributed tracing for an ingress controller.

#### Checklist assessment

- ✅ W3C Trace Context (`traceparent`) is supported and propagated correctly — both inbound (continuing external traces) and outbound (injecting into upstream requests)
- ✅ Entry-point spans are consistently `SERVER` (kind=2) spans
- ✅ Upstream forwarding spans are consistently `CLIENT` (kind=3) spans
- ✅ Parent-child relationships are correct and consistent
- ✅ E2E traces span Envoy and upstream services coherently
- ✅ Context propagation works across service boundaries
- ✅ The two-span model (ingress + egress) is intentional and documented
- ⚠️ Span names (`ingress`, `router demo_otel-eval-backend_3000 egress`) reflect Envoy internals rather than logical operations, but the structure is correct
- ⚠️ No schema URL on trace data; trace modeling is not explicitly documented as a design contract

#### Rationale

This is the strongest dimension for Contour. W3C Trace Context propagation works correctly in both directions (inbound context continuation and outbound context injection). The span structure is intentional: a SERVER span for the inbound request and a CLIENT span for the upstream forwarding, with a correct parent-child relationship. E2E traces are coherent and span multiple services. This meets Level 2 characteristics: trace modeling is intentional, context propagation is explicit, and traces represent meaningful logical operations (even if span names could be improved). The project does not reach Level 3 because trace modeling is not explicitly documented as a design contract, and there is no evidence of trace modeling being reviewed architecturally.

---

### 5. Multi-Signal Observability

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Signal availability

| Signal | Status | Transport | Notes |
|--------|--------|-----------|-------|
| Traces | First-class | OTLP gRPC | Envoy spans via ExtensionService |
| Metrics | Present | Prometheus scrape | 655 unique metrics (470 `envoy_*`, 14 `contour_*`, plus Go runtime) |
| Logs | Absent | — | stdout only; no OTLP path |

Logs are not flowing and are not supported via OTLP. This is a hard blocker for Level 2, which requires all three signals to be first-class.

##### Cross-signal correlation

There is **no cross-signal correlation** between traces and metrics:
- Traces use `service.name=contour` (Envoy pods)
- Metrics use `service.name=contour-control-plane` and `service.name=envoy-data-plane` (collector-derived scrape job names)
- No shared attribute keys exist between trace spans and metric data points
- Logs do not exist as an OTLP signal

A user cannot pivot from a latency metric to a specific trace, or from a trace to related logs. The signals are completely isolated from each other.

##### Collection model

- Traces: OTLP push (Envoy → Collector)
- Metrics: Prometheus pull (Collector scrapes Envoy and Contour)
- Logs: Not collected via OTel

The mixed push/pull model with different `service.name` values makes cross-signal correlation impossible without significant downstream pipeline work.

#### Checklist assessment

- ✅ Traces are present and flowing via OTLP
- ✅ Metrics are present (rich set: 655 unique metric names)
- ❌ Logs are absent as an OTLP signal
- ❌ No trace context in logs (logs not available)
- ❌ Metrics do not share attribute keys with traces
- ❌ `service.name` differs between traces and metrics
- ❌ Users cannot pivot between signals without manual correlation effort
- ❌ Signals are designed and collected independently

#### Rationale

Two of three signals are present (traces and metrics), which places the project above Level 0. However, logs are absent as an OTLP signal, and there is no cross-signal correlation between traces and metrics. The model's Level 1 description — "multiple signals exist but are loosely connected; signals coexist rather than working together" — accurately describes the current state. Level 2 requires all three signals to be first-class and correlation to not depend on ad-hoc parsing; neither condition is met.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Envoy span names reflect internal Envoy concepts rather than logical HTTP operations:

- `ingress` — generic; does not communicate HTTP method, route, or virtual host
- `router demo_otel-eval-backend_3000 egress` — encodes the Envoy cluster name (`demo_otel-eval-backend_3000`) in the span name; this is an internal Envoy routing concept, not a user-meaningful operation name

Compare with OTel HTTP semconv best practice: `GET /path` or `HTTP GET`. An operator unfamiliar with Envoy's cluster naming scheme would need to decode `demo_otel-eval-backend_3000` to understand what the span represents (namespace `demo`, service `otel-eval-backend`, port `3000`).

##### Signal-to-noise ratio

**Traces**: Two spans per request is lean and focused. No excessive internal spans. The sampling rate is configurable (`overallSampling: 100` by default). The span set is appropriate for an ingress controller.

**Metrics**: 655 unique metric names is very high. The Envoy metrics set includes extensive circuit breaker, HTTP/2 framing, SSL cipher, and internal state metrics that are rarely needed for operational use. Many metrics have low operational value by default (e.g., `envoy_cluster_http2_inbound_empty_frames_flood`, `envoy_cluster_http2_inbound_priority_frames_flood`). There is no guidance on which metrics are most operationally relevant.

**Logs**: Not applicable (no OTLP logs).

##### Default usability

- Span attributes require Envoy-specific knowledge to interpret (`upstream_cluster`, `response_flags`, `node_id`, `guid:x-request-id`, `zone`)
- The `response_flags` attribute (e.g., `-` for no flags) uses Envoy's internal flag notation, which is not self-explanatory
- `upstream_cluster` value (`demo/otel-eval-backend/3000/da39a3ee5e`) encodes Envoy's internal cluster naming scheme
- Contour-specific customization options (`customTags`, `includePodDetail`) allow operators to add context, but defaults require domain knowledge to use effectively

#### Checklist assessment

- ✅ Trace volume is lean (2 spans per request)
- ✅ Sampling is configurable
- ❌ Span names reflect Envoy internals, not logical operations
- ❌ Multiple span attributes require Envoy-specific knowledge
- ❌ Metric volume is very high (655 metrics) with no guidance on which are operationally relevant
- ❌ No documentation on which signals to use for common investigative workflows
- ❌ `response_flags` and `upstream_cluster` values require Envoy documentation to interpret

#### Rationale

Some noise reduction is present (2 spans per request is lean; sampling is configurable), but telemetry is still shaped primarily by Envoy's internal model rather than operator needs. Span names encode Envoy cluster names, and attributes require Envoy-specific knowledge. This matches Level 1: "some effort is made to reduce noise and improve clarity, but telemetry is still largely shaped by internal perspectives." The project does not reach Level 2 because span names do not describe logical operations and operators need domain knowledge to interpret most attributes.

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

The [Tracing Support](https://projectcontour.io/docs/1.33/config/tracing/) page documents the tracing configuration options (`serviceName`, `overallSampling`, `includePodDetail`, `customTags`, `maxPathTagLength`, `extensionService`). The [Prometheus guide](https://projectcontour.io/docs/1.33/guides/prometheus/) documents the metrics endpoints. However:
- No telemetry is documented as a **stable contract** with defined stability guarantees
- No distinction between stable and experimental telemetry
- Span attribute names and metric names are not documented as a reference (no attribute catalog)

##### Change communication

Changelogs for v1.33.0 and v1.32.0 were checked. No telemetry-specific changelog entries were found. There is no dedicated section for observability or telemetry changes in release notes. Telemetry changes (if any) would be buried in general commit messages.

##### Schema URL presence

- **Traces**: No `schemaUrl` is set at the `resourceSpans` or `scopeSpans` level. Confirmed by `jq` query returning empty.
- **Metrics**: No `schemaUrl` is set by the Prometheus receiver (confirmed: `schemaUrl=none` for `prometheusreceiver` scope). The k8sclusterreceiver sets `schemaUrl: https://opentelemetry.io/schemas/1.18.0`, but this is collector-side, not project-native.

##### Stability guarantees

No explicit stability guarantees are stated for telemetry in the Contour documentation. The project does not distinguish between stable and experimental telemetry. There is no deprecation policy for telemetry.

#### Checklist assessment

- ✅ Tracing configuration is documented
- ✅ Metrics endpoints are documented
- ❌ No `schemaUrl` on trace or metric data
- ❌ No telemetry-specific changelog sections
- ❌ No distinction between stable and experimental telemetry
- ❌ No explicit stability guarantees for telemetry
- ❌ Telemetry is not treated as part of the public contract
- ❌ No deprecation policy for telemetry

#### Rationale

The project documents how to configure tracing and where to scrape metrics, which demonstrates awareness of telemetry as a user-facing concern (above Level 0). However, there are no stability guarantees, no `schemaUrl` in emitted data, no telemetry-specific changelog communication, and no distinction between stable and experimental telemetry. This matches Level 1: "maintainers are aware that telemetry changes have impact, but handling is informal and inconsistent." Level 2 requires telemetry to be treated as part of the public contract with explicit change communication; this is not yet present.

---

## Key findings

### Strengths

- **W3C Trace Context propagation is correct and complete.** Envoy correctly reads inbound `traceparent` headers (continuing external traces) and injects `traceparent` into upstream requests. E2E traces spanning Envoy and backend services are coherent and verified. This is the most important capability for an ingress controller.
- **Rich Envoy metrics set.** 470+ `envoy_*` metrics and 14 `contour_*` control-plane metrics provide deep visibility into proxy behavior, circuit breakers, upstream health, and xDS control plane state. The Prometheus metrics endpoint is well-established and documented.
- **Tracing is documented and first-class.** The [Tracing Support](https://projectcontour.io/docs/1.33/config/tracing/) page provides a complete guide for enabling OTel tracing via the `ExtensionService` CRD. OpenTelemetry is explicitly positioned as the supported tracing integration, not an afterthought.

### Areas for improvement

1. **Adopt current OTel HTTP semantic conventions in Envoy's OTel tracer.** All HTTP span attributes use deprecated OpenTracing-era names (`http.method`, `http.status_code`, `http.url`). Migrating to `http.request.method`, `http.response.status_code`, `url.full`, `url.path`, `network.protocol.name` would immediately improve interoperability with OTel-native tooling. This requires upstream Envoy changes, but Contour could advocate for or contribute this work.

2. **Add `service.version` and respect `OTEL_SERVICE_NAME` / `OTEL_RESOURCE_ATTRIBUTES`.** The trace resource lacks `service.version` (the Envoy version is available as `telemetry.sdk.version` in a non-semantic build hash format). Supporting standard OTel env vars for service identity would allow operators to configure telemetry consistently without project-specific ConfigMap changes.

3. **Unify service identity across signals.** Traces use `service.name=contour` while metrics use collector-derived names (`contour-control-plane`, `envoy-data-plane`). Establishing a consistent `service.name` (or at minimum documenting the naming scheme) would enable cross-signal correlation in backends like Grafana or Jaeger.

4. **Add OTLP metrics export.** The metrics-only-via-Prometheus-scrape model prevents cross-signal correlation and requires users to maintain a separate scrape pipeline. OTLP metrics export (even as an opt-in alongside Prometheus) would significantly improve the integration surface and enable `service.name` consistency between traces and metrics.

5. **Introduce telemetry stability documentation and changelog sections.** Adding a dedicated "Observability" or "Telemetry" section to release notes, and documenting which span attributes and metric names are considered stable, would allow operators to build reliable dashboards and alerts without fear of silent breakage.

### Notable observations

- **Architectural constraint: Envoy owns the trace semantics.** Contour's role in tracing is as a configuration layer — it programs Envoy's OTel tracer via xDS. The span attributes, span names, and resource attributes are entirely determined by Envoy's built-in OTel tracer implementation, not by Contour code. Improvements to semantic conventions require upstream Envoy changes. This is a significant architectural constraint that limits how quickly Contour can improve its semconv compliance.

- **`ExtensionService` cross-namespace limitation is a real operator pain point.** The `ExtensionService` CRD cannot reference services in other namespaces and does not support `ExternalName` services. This forces operators who run their OTel Collector in a separate namespace (common in production) to create manual `ClusterIP` Service + `Endpoints` objects in the `projectcontour` namespace — a non-obvious workaround that is not documented in the tracing guide.

- **Envoy metrics path is non-standard (`/stats/prometheus`).** The official Contour Prometheus guide does not prominently highlight that Envoy's metrics endpoint is at `/stats/prometheus`, not `/metrics`. The Prometheus receiver's default `metrics_path=/metrics` silently returns 404, resulting in no Envoy metrics being scraped. This is a common operator mistake.

- **`telemetry.sdk.version` is a build hash, not a semantic version.** The value `3542e3464a2662423065a6ec854905b25955a09e/1.35.10/Clean/RELEASE/BoringSSL` is not useful as a version identifier for filtering or grouping telemetry. The actual Envoy version (`1.35.10`) is embedded in the string but requires parsing to extract.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file exporter in a local kind cluster (`otel-eval-contour`)
- The k8sattributes processor was used to distinguish native vs enriched resource attributes
- Traffic was generated both without inbound trace context (to observe Envoy-initiated traces) and with W3C `traceparent` headers (to verify context propagation)
- Semantic conventions were checked against the current stable OpenTelemetry specification (HTTP semconv v1.x stable)
- Documentation reviewed: Contour v1.33 Tracing Support page, Configuration Reference, Prometheus Metrics guide
- Source code not reviewed directly; architectural constraints inferred from documentation and telemetry data
- The Bitnami Helm chart was unavailable (images moved off Docker Hub); official rendered manifests from `v1.33.4` were used instead
