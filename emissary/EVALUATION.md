# OpenTelemetry Support Maturity Evaluation: Emissary-Ingress

## Project overview

- **Project**: Emissary-Ingress — a CNCF Incubating Kubernetes-native API gateway and ingress controller built on top of Envoy Proxy. Configured via CRDs (`Mapping`, `Listener`, `TracingService`, etc.).
- **Version evaluated**: 3.12.2 (Helm chart 8.12.2)
- **Evaluation date**: 2026-05-08
- **Cluster**: otel-eval-emissary
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 1 | Traces via Zipkin (not OTLP); metrics via Prometheus scrape only; no log export |
| Semantic Conventions | 1 | Deprecated HTTP attributes throughout; Envoy-native naming not aligned with OTel semconv |
| Resource Attributes & Configuration | 1 | Only `service.name` emitted natively; no `service.version`, no `OTEL_*` env var support |
| Trace Modeling & Context Propagation | 2 | W3C Trace Context propagation works correctly; span structure is coherent but span kinds are wrong (CLIENT for ingress) |
| Multi-Signal Observability | 1 | Traces and metrics both available but via non-OTLP paths; no logs; no cross-signal correlation |
| Audience & Signal Quality | 1 | Span names are Envoy-internal (`localhost:18080`, `router … egress`); metrics are Envoy-raw with no operator-friendly abstraction |
| Stability & Change Management | 1 | No telemetry documented as a contract; OTel driver is broken in the released version; no schema URLs |

---

## Telemetry overview

### Signals observed
- **Traces**: flowing — Zipkin HTTP (B3/Zipkin v2 format) → OTel Collector port 9411 → OTLP file export
- **Metrics**: flowing — Prometheus scrape of Envoy admin port 8877 → OTel Collector → OTLP file export
- **Logs**: not flowing — stdout plain text only; no OTLP log export

### Resource attributes (native, before collector enrichment)

Emissary (via Envoy's Zipkin tracer) emits only a single resource attribute natively:

| Attribute | Value |
|-----------|-------|
| `service.name` | `"ambassador-emissary"` |

The Prometheus scrape path emits collector-synthesized resource attributes:

| Attribute | Value |
|-----------|-------|
| `service.name` | `"emissary"` |
| `service.instance.id` | `"emissary-emissary-ingress-admin.emissary.svc.cluster.local:8877"` |
| `server.address` | `"emissary-emissary-ingress-admin.emissary.svc.cluster.local"` |
| `server.port` | `"8877"` |
| `url.scheme` | `"http"` |

### Resource attributes (after collector enrichment)

After the `k8sattributes` processor, Emissary trace spans gain the full Kubernetes identity set:

- `k8s.deployment.name: emissary-emissary-ingress`
- `k8s.namespace.name: emissary`
- `k8s.node.name: otel-eval-emissary-control-plane`
- `k8s.pod.name: emissary-emissary-ingress-558fc6db4c-gpnrl`
- `k8s.pod.uid`, `k8s.replicaset.name`, and numerous pod labels/annotations

None of these are emitted by Emissary itself.

---

## Dimension evaluations

### 1. Integration Surface

**Level: 1 — Partial / Non-native**

#### Evidence

- **Traces**: Emissary does not push OTLP. Traces are emitted in Zipkin HTTP_JSON format (Zipkin v2 API `/api/v2/spans`) via the `TracingService` CRD with `driver: zipkin`. The OTel Collector's Zipkin receiver converts these to OTLP internally. The TracingService CRD does expose a `driver: opentelemetry` option (enum value confirmed in the CRD schema), but this driver is **silently broken** in 3.12.2: the Python config generator (`v3tracing.py`) produces the Envoy tracer name `"envoy.opentelemetry"` instead of the correct `"envoy.tracers.opentelemetry"`, causing Envoy to silently ignore the tracer configuration. Zero connections to the OTel collector were established when using the OTel driver (confirmed via Envoy admin API during installation).
- **Metrics**: Emissary does not push OTLP metrics. Envoy exposes ~413 Prometheus-format metrics at port 8877 (`/metrics`). These must be scraped by an external collector. The Helm chart includes `adminService.create: true` and Prometheus scrape annotations but no built-in OTLP metrics export.
- **Logs**: No OTLP log export. Access logs are written to stdout in a plain-text format (configurable to JSON via the `Module` CRD, but not OTLP).
- **Configuration mechanism**: Tracing is configured via the `TracingService` CRD, not `OTEL_*` environment variables. No OTel SDK is in use; instrumentation is Envoy-native.

#### Checklist assessment
- ✅ Some telemetry can be collected with reasonable effort (Zipkin + Prometheus scrape)
- ✅ Tracing is documented (TracingService CRD reference exists)
- ❌ No OTLP push for any signal
- ❌ Native `driver: opentelemetry` is non-functional in the released version
- ❌ Metrics require pull-based scraping, not OTLP push
- ❌ No log export at all

#### Rationale

Emissary achieves Level 1 because telemetry is collectable with configuration effort, and the project explicitly supports a tracing integration path (even if Zipkin-based in practice). It does not reach Level 2 because no signal uses OTLP as the native transport — traces require a Zipkin-to-OTLP conversion by the collector, metrics require Prometheus scraping, and logs are absent entirely.

---

### 2. Semantic Conventions

**Level: 1 — Partial alignment**

#### Evidence

##### Trace attributes (Emissary/Envoy spans)

Span attributes observed on `localhost:18080` (ingress) and `router … egress` spans:

| Attribute observed | OTel semconv status | Current equivalent |
|--------------------|--------------------|--------------------|
| `http.method` | **DEPRECATED** | `http.request.method` |
| `http.status_code` | **DEPRECATED** | `http.response.status_code` |
| `http.url` | **DEPRECATED** | `url.full` |
| `http.protocol` | **DEPRECATED** | `network.protocol.name` + `network.protocol.version` |
| `peer.address` | Non-standard (Envoy-native) | `net.peer.ip` / `server.address` |
| `downstream_cluster` | Envoy-native, not in OTel semconv | — |
| `upstream_cluster` | Envoy-native, not in OTel semconv | — |
| `upstream_cluster.name` | Envoy-native | — |
| `upstream_address` | Envoy-native | — |
| `response_flags` | Envoy-native | — |
| `response_size` | Envoy-native | `http.response.body.size` (semconv) |
| `request_size` | Envoy-native | `http.request.body.size` (semconv) |
| `node_id` | Envoy-native | — |
| `component` | Old OpenTracing convention | — |
| `guid:x-request-id` | Envoy-native (Envoy request ID) | — |
| `user_agent` | Non-standard | `user_agent.original` |
| `net.host.ip` | **DEPRECATED** | `network.local.address` |

All HTTP span attributes are from the deprecated pre-1.20 OTel HTTP semconv. No current-stable attributes (`http.request.method`, `http.response.status_code`, `url.path`, `url.full`, `network.protocol.name`) are present.

##### Metric names and attributes

All 413 metrics use Envoy-native naming conventions with `envoy_` prefix (e.g., `envoy_cluster_upstream_cx_active`, `envoy_http_downstream_rq_completed`). Ambassador-specific metrics use `ambassador_` prefix (e.g., `ambassador_diagnostics_errors`, `ambassador_reconfiguration_time_seconds`). None follow OTel semantic conventions (which would use dotted namespacing like `http.server.request.duration`). Metric labels use Envoy-native attribute names (`envoy_http_conn_manager_prefix`, `envoy_rds_route_config`, `envoy_response_code_class`).

##### Log attributes

No OTLP logs are emitted. Not applicable.

##### Schema URL

No `schemaUrl` is set on any trace `resourceSpans`. Metrics have a schema URL of `https://opentelemetry.io/schemas/1.18.0` — but this is set by the OTel Collector's Prometheus receiver, not by Emissary itself.

#### Checklist assessment
- ✅ Some HTTP attributes are present and meaningful (method, status code, URL)
- ❌ All HTTP trace attributes use deprecated pre-1.20 semconv names
- ❌ Multiple Envoy-native attributes have no OTel semconv equivalent
- ❌ Metric naming uses Envoy/Prometheus conventions, not OTel dotted namespace
- ❌ No `schemaUrl` set natively
- ❌ Span kind is incorrect for the ingress entry-point span (see Dimension 4)

#### Rationale

Level 1 because the project does emit semantically meaningful HTTP attributes (method, status code, URL are all present and correct in value, just under deprecated key names). It does not reach Level 2 because the attributes are systematically deprecated and the metric names make no attempt to align with OTel semantic conventions.

---

### 3. Resource Attributes & Configuration

**Level: 1 — Minimal identity**

#### Evidence

##### Native resource attributes

Emissary natively emits only `service.name: "ambassador-emissary"` on trace spans. This name is derived from the Envoy cluster name (`ambassador-emissary`) and is not user-configurable via the TracingService CRD. There is no `service.version`, `service.namespace`, `service.instance.id`, `telemetry.sdk.name`, or `telemetry.sdk.version`.

The Prometheus-scraped metrics path produces `service.name: "emissary"` (synthesized by the OTel Collector from the scrape target address), which is different from the trace signal's `service.name: "ambassador-emissary"`. This inconsistency prevents cross-signal correlation by service identity.

##### OTEL_* environment variable support

No `OTEL_*` environment variables are present in the Emissary deployment. The Emissary container only uses:
- `HOST_IP` (Kubernetes field ref)
- `AMBASSADOR_NAMESPACE` (Kubernetes field ref)
- `AGENT_CONFIG_RESOURCE_NAME` (Datawire cloud agent token)

There is no OTel SDK in use; Envoy's built-in tracing subsystem is configured via the `TracingService` CRD. `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, and similar standard OTel env vars have no effect on Emissary.

##### Identity consistency across signals

| Signal | `service.name` | `service.version` | `service.namespace` |
|--------|---------------|-------------------|---------------------|
| Traces | `ambassador-emissary` | absent | absent |
| Metrics | `emissary` | absent | absent |
| Logs | N/A | N/A | N/A |

The `service.name` differs between traces and metrics, making cross-signal correlation by service identity impossible without pipeline-level transforms.

#### Checklist assessment
- ✅ `service.name` is present on traces
- ❌ `service.name` is inconsistent between traces and metrics
- ❌ `service.version` is absent from all signals
- ❌ No `OTEL_*` env var support (no OTel SDK)
- ❌ No `telemetry.sdk.*` attributes (Envoy native instrumentation)
- ❌ No `service.namespace` or `service.instance.id` natively

#### Rationale

Level 1 because a `service.name` is present on traces, providing minimal identity. The project does not reach Level 2 due to inconsistent `service.name` across signals, absent `service.version`, and no support for standard `OTEL_*` configuration.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — Correct propagation with structural issues**

#### Evidence

##### Span structure

Each HTTP request through Emissary produces two Envoy spans:
1. **Ingress span** — name: `"localhost:18080"`, kind: `CLIENT` (3), no parent span ID (root)
2. **Egress span** — name: `"router otel_eval_backend_demo_3000 egress"`, kind: `CLIENT` (3), parent = ingress span ID

The parent-child relationship between ingress and egress spans is correct. However:
- **Span kind is wrong**: The ingress span (which receives the inbound HTTP request) should be `SERVER` (kind=2), not `CLIENT` (kind=3). Envoy models both ingress and egress as CLIENT spans, which is incorrect per OTel semantics.
- **Span names are Envoy-internal**: `"localhost:18080"` encodes the local port, not a logical operation name. `"router otel_eval_backend_demo_3000 egress"` encodes an internal Envoy cluster name. Neither is meaningful to an operator unfamiliar with Envoy internals.
- **No status code on spans**: All Emissary spans have `status: {}` (empty/UNSET). Even 404 responses (confirmed present in the data) do not produce `STATUS_ERROR` spans. The HTTP status code is present as a span attribute (`http.status_code=404`) but the span-level status is never set to ERROR.

##### Context propagation

W3C Trace Context propagation is confirmed working. When a request arrives with `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`, the downstream backend spans appear with `traceId: 4bf92f3577b34da6a3ce929d0e0e4736` and `parentSpanId: 00f067aa0ba902b7` (the injected parent span ID). This demonstrates that Emissary correctly reads and forwards the W3C `traceparent` header, enabling distributed tracing across service boundaries.

However, the Emissary ingress span itself does **not** appear in the injected trace — Emissary does not create a child of the incoming `traceparent`. Instead, the backend spans become direct children of the externally-injected parent span. This means the Emissary proxy hop is invisible in the distributed trace when an inbound `traceparent` is present.

##### Trace coherence

Without an inbound `traceparent`, traces are internally coherent: the ingress span is the root, the egress span is its child, and the backend's `GET /` span is a further child (confirmed by matching trace IDs). The full end-to-end trace (Emissary ingress → Emissary egress → backend server span → backend middleware spans) is visible in a single trace.

#### Checklist assessment
- ✅ W3C Trace Context (`traceparent`) is read and forwarded to downstream services
- ✅ Parent-child span relationships are correct within Emissary (ingress → egress)
- ✅ End-to-end trace coherence across Emissary + backend is present
- ❌ Ingress span kind is `CLIENT` instead of `SERVER`
- ❌ Span names are Envoy-internal, not logical operation names
- ❌ Span status is never set to ERROR even for 4xx/5xx responses
- ❌ When inbound `traceparent` is present, Emissary does not create a child span of the incoming trace (proxy hop is invisible)

#### Rationale

Level 2 because W3C Trace Context propagation is functional and end-to-end traces are coherent. The project falls short of Level 3 due to incorrect span kinds, non-semantic span names, absent error status propagation, and the missing proxy span in the injected-context case.

---

### 5. Multi-Signal Observability

**Level: 1 — Multiple signals available but disconnected**

#### Evidence

##### Signal availability

| Signal | Available | Protocol | First-class? |
|--------|-----------|----------|-------------|
| Traces | ✅ | Zipkin HTTP → collector conversion | Partial (Zipkin workaround required) |
| Metrics | ✅ | Prometheus scrape | Partial (pull-only, no OTLP push) |
| Logs | ❌ | stdout plain text | No |

##### Cross-signal correlation

There is no cross-signal correlation:
- Trace spans do not carry a `traceId` or `spanId` that could be matched to log records (no OTLP logs exist).
- Metrics have `service.name: "emissary"` while traces have `service.name: "ambassador-emissary"` — the two signals cannot be correlated by service identity without a pipeline transform.
- No exemplars are present in the Prometheus metrics (Envoy does not emit Prometheus exemplars linking metric data points to trace IDs).

##### Collection model

- **Traces**: Emissary pushes Zipkin HTTP to the collector. This is a push model, but not OTLP.
- **Metrics**: Collector scrapes Emissary's Envoy admin port 8877. This is a pull model.
- **Logs**: No collection path.

The two active signals use different collection models (push vs. pull) and different protocols (Zipkin vs. Prometheus), requiring separate collector pipeline configurations.

#### Checklist assessment
- ✅ Two signals (traces and metrics) are available
- ✅ Metrics are rich (413 metrics covering connections, requests, latency histograms, circuit breakers, etc.)
- ❌ No OTLP push for any signal
- ❌ No log signal
- ❌ No cross-signal correlation (mismatched `service.name`, no exemplars, no trace context in logs)
- ❌ Inconsistent collection models (push Zipkin vs. pull Prometheus)

#### Rationale

Level 1 because two signals are available and provide meaningful operational coverage. The project does not reach Level 2 because there is no cross-signal correlation capability and no unified OTLP-based collection path.

---

### 6. Audience & Signal Quality

**Level: 1 — Raw infrastructure signals**

#### Evidence

##### Span naming

Emissary span names observed:
- `"localhost:18080"` — encodes the local port where traffic arrived. Not meaningful to an operator; changes with port configuration.
- `"router otel_eval_backend_demo_3000 egress"` — encodes the Envoy cluster name (`otel_eval_backend_demo_3000`), which is derived from the Kubernetes service name with underscores replacing dots. An operator must know Envoy cluster naming conventions to interpret this.

These names are Envoy-internal identifiers, not logical operation descriptions. They do not convey "HTTP request to backend service" in a human-readable way.

##### Signal-to-noise ratio

**Metrics**: 413 metrics are exported, covering every Envoy subsystem (cluster management, HTTP/1, HTTP/2, TLS inspector, overload manager, runtime, etc.). Many of these are zero-valued counters of edge-case error conditions (e.g., `envoy_cluster_http2_inbound_priority_frames_flood`, `envoy_cluster_http2_outbound_control_flood`). There is no filtering or grouping — all Envoy statistics are exported as-is. This creates significant noise for operators who want to monitor basic request health.

**Traces**: Each HTTP request produces exactly 2 Emissary spans (ingress + egress), which is a reasonable volume. However, the span names and missing error status require operator familiarity with Envoy internals to interpret correctly.

##### Default usability

An operator deploying Emissary with default settings would receive:
- No traces (tracing is opt-in via `TracingService` CRD)
- Metrics only if they configure a Prometheus scraper
- No logs in any structured format

Even after configuration, the telemetry requires significant operator knowledge to interpret: Envoy cluster names must be decoded, deprecated HTTP attribute names must be mapped to modern equivalents, and the 400+ metrics must be filtered to find the operationally relevant ones.

#### Checklist assessment
- ✅ Span attributes contain enough information to reconstruct request details (method, URL, status, upstream)
- ✅ Metrics cover the full Envoy operational surface (connections, requests, latency, circuit breakers)
- ❌ Span names are Envoy-internal, not logical operation descriptions
- ❌ No error status on spans (4xx/5xx not reflected in span status)
- ❌ Metric volume (413) includes significant noise with no default filtering
- ❌ Tracing is opt-in with no default configuration
- ❌ Requires deep Envoy knowledge to interpret telemetry correctly

#### Rationale

Level 1 because the telemetry is functional and contains meaningful data, but requires expert-level Envoy knowledge to interpret. The span names encode infrastructure internals, the metric set is unfiltered, and there are no operator-facing abstractions. The project does not reach Level 2 because the signals are not usable out-of-the-box by an operator unfamiliar with Envoy internals.

---

### 7. Stability & Change Management

**Level: 1 — Minimal documentation, active bugs**

#### Evidence

##### Documentation of telemetry behavior

The Emissary documentation describes the `TracingService` CRD and its `driver` options (including `opentelemetry`). However:
- The `driver: opentelemetry` option is documented as available but is silently broken in 3.12.2 due to a code bug (wrong Envoy tracer extension name).
- There is no documentation of what span names, span attributes, or resource attributes Emissary emits.
- There is no documentation of what Prometheus metrics are available or what they mean.
- The tracing documentation does not mention the Zipkin workaround required when using OTel Collector.

##### Change communication

No evidence of telemetry-specific changelog entries. The CHANGELOG does not call out breaking changes to span names, metric names, or attribute sets.

##### Schema URL presence

No `schemaUrl` is set on Emissary trace `resourceSpans`. The schema URL present on metrics (`https://opentelemetry.io/schemas/1.18.0`) is added by the OTel Collector's Prometheus receiver, not by Emissary.

##### Stability guarantees

No explicit stability commitments exist for Emissary's telemetry output. The `driver: opentelemetry` feature is effectively broken in the current release with no documented workaround in the official docs. The Zipkin workaround requires community knowledge (GitHub issues, source code inspection) to discover.

##### Active bug: OTel driver

The `TracingService` with `driver: opentelemetry` silently fails. The Python config generator (`python/ambassador/envoy/v3/v3tracing.py`) generates `"envoy.opentelemetry"` as the Envoy tracer name, but the correct extension name is `"envoy.tracers.opentelemetry"`. This is a regression that renders the advertised native OTel integration non-functional. This was confirmed empirically during installation (Envoy admin API showed 0 connections to the OTel collector endpoint).

#### Checklist assessment
- ✅ TracingService CRD is documented with driver options
- ❌ `driver: opentelemetry` is documented but broken in the released version
- ❌ No documentation of emitted span names, attributes, or resource attributes
- ❌ No documentation of Prometheus metric semantics
- ❌ No `schemaUrl` set natively on any signal
- ❌ No explicit stability guarantees for telemetry output
- ❌ No telemetry-specific changelog entries

#### Rationale

Level 1 because some configuration documentation exists for the tracing integration. The project falls short of Level 2 due to the broken native OTel driver, absent telemetry schema documentation, no schema URL, and no stability commitments. The broken `driver: opentelemetry` is a particularly significant issue as it undermines trust in the advertised OTel support.

---

## Key findings

### Strengths

- **W3C Trace Context propagation works correctly**: When inbound requests carry a `traceparent` header, Emissary correctly reads and forwards the trace context to downstream services. This enables distributed tracing across service boundaries when Emissary is used as an ingress gateway.
- **Rich Envoy metrics coverage**: 413 metrics covering the full Envoy operational surface — upstream/downstream connections, request rates, latency histograms (p50/p75/p90/p95/p99 via Envoy's histogram buckets), circuit breaker states, HTTP/2 stream counts, and more. For operators who understand Envoy, this is comprehensive.
- **Coherent end-to-end traces**: Without an inbound trace context, Emissary produces coherent traces spanning the full request path: ingress span → egress span → backend server span → backend middleware spans. All in a single trace ID.

### Areas for improvement

1. **Fix the `driver: opentelemetry` bug**: The Python config generator in `v3tracing.py` must produce `"envoy.tracers.opentelemetry"` (not `"envoy.opentelemetry"`). This is a one-line fix that would enable native OTLP gRPC trace export, eliminating the need for the Zipkin workaround and immediately improving the Integration Surface score.
2. **Update span attributes to current OTel HTTP semantic conventions**: Replace deprecated attributes (`http.method` → `http.request.method`, `http.status_code` → `http.response.status_code`, `http.url` → `url.full`, `http.protocol` → `network.protocol.name`/`network.protocol.version`, `net.host.ip` → `network.local.address`). Also set span status to ERROR for 4xx/5xx responses. These changes in Envoy's tracing configuration would significantly improve the Semantic Conventions score.
3. **Standardize `service.name` across signals and add `service.version`**: The `service.name` differs between traces (`ambassador-emissary`) and metrics (`emissary`). Emissary should emit a consistent, user-configurable service name across all signals, and include `service.version` (the Emissary version, e.g., `3.12.2`) as a resource attribute. This would enable cross-signal correlation and improve the Resource Attributes score.

### Notable observations

- **Silent failure of the OTel driver**: The `driver: opentelemetry` TracingService produces zero errors — Envoy simply ignores the unknown tracer name and proceeds without tracing. This is extremely difficult to debug without examining the Envoy admin API directly. Users who configure `driver: opentelemetry` will believe tracing is configured when it is not.
- **Proxy hop invisible with injected trace context**: When a request arrives with a `traceparent` header, Emissary does not create a child span of the incoming trace. The backend spans become direct children of the externally-injected parent, making the Emissary proxy hop invisible in the distributed trace. This is a significant observability gap for users who inject trace context from external clients.
- **`service.name` inconsistency is collector-introduced**: The `service.name: "emissary"` on metrics is synthesized by the OTel Collector's Prometheus receiver from the scrape target address — Emissary itself does not set this. The `service.name: "ambassador-emissary"` on traces comes from Envoy's cluster name. Neither is user-configurable through Emissary's configuration surface.
- **Span kind `CLIENT` for ingress**: Both the ingress span (receiving inbound traffic) and the egress span (sending to upstream) are modeled as `CLIENT` spans. The ingress span should be `SERVER` per OTel semantics. This is an Envoy limitation that Emissary inherits.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file export in a local kind cluster (`otel-eval-emissary`)
- The `k8sattributes` processor was used to enrich telemetry; native vs. enriched resource attributes were distinguished by identifying which attributes Emissary/Envoy emits before k8s enrichment
- The `driver: opentelemetry` TracingService was tested and confirmed non-functional; the Zipkin workaround (`driver: zipkin` pointing to the OTel Collector's Zipkin receiver on port 9411) was used to collect trace data
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv v1.20+)
- W3C Trace Context propagation was verified by injecting a known `traceparent` header (`00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`) and confirming the trace ID appeared in downstream backend spans
- 45 HTTP requests were generated (20 without trace context, 20 with injected trace context, 5 error/404 cases) to produce the telemetry data analyzed
- Documentation was reviewed at https://www.getambassador.io/docs/emissary and source code at https://github.com/emissary-ingress/emissary
