# OpenTelemetry Support Maturity Evaluation: Emissary Ingress

## Project overview

- **Project**: Emissary Ingress (formerly Ambassador) — a CNCF incubating Kubernetes-native API gateway and ingress controller built on Envoy Proxy, providing HTTP/gRPC routing, TLS termination, rate limiting, and authentication via CRD-based configuration
- **Version evaluated**: 3.12.2 (Helm chart 8.12.2)
- **Evaluation date**: 2026-05-09
- **Cluster**: otel-eval-emissary
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 0 | Metrics via Prometheus scrape only; OTLP not supported; tracing broken in 3.12.2 |
| Semantic Conventions | 0 | Proprietary `envoy_*` / `ambassador_*` naming; deprecated HTTP span attributes from downstream backend |
| Resource Attributes & Configuration | 0 | No native resource attributes; `service.name` and all k8s attributes are collector-derived; no `OTEL_*` support |
| Trace Modeling & Context Propagation | 0 | Emissary emits zero spans; tracing is architecturally broken in current version |
| Multi-Signal Observability | 0 | Only metrics are first-class; traces and logs not flowing from Emissary itself |
| Audience & Signal Quality | 1 | 407 Prometheus metrics provide operational value; Envoy metric labels are Envoy-internal, not OTel-oriented |
| Stability & Change Management | 0 | No telemetry documentation as contract; tracing marked WIP; no schema URL; no change governance |

---

## Telemetry overview

### Signals observed

- **Traces**: ❌ Not flowing from Emissary — TracingService CRD configured (Zipkin driver), but 0 spans sent due to an architectural bug where Emissary's ADS server overrides the static tracing cluster with a dynamic one, preventing Envoy from delivering spans. Emissary sampled 75 requests (`envoy_http_tracing_random_sampling: 75` on `ingress_https`) but `envoy_cluster_upstream_rq_total` for the tracing cluster remains at 0. The OpenTelemetry driver is explicitly marked "work-in-progress" in Emissary's own logs.
- **Metrics**: ✅ Flowing — Prometheus scrape of `/metrics` on port 8877 (diagd admin service), collected via OTel Collector's `prometheus` receiver. 407 unique metric names observed.
- **Logs**: ❌ Not flowing — Emissary/Envoy does not support OTLP log export. The `LogService` CRD uses Envoy's Access Log Service (ALS) gRPC protocol, which is not OTLP and not supported by the standard OTel Collector.

### Resource attributes (native, before collector enrichment)

Emissary emits **no native resource attributes**. The Prometheus `/metrics` endpoint exposes raw Envoy and Ambassador metrics with no OTLP resource envelope. The project does not use an OTel SDK and does not set any `service.*`, `telemetry.sdk.*`, or other resource attributes natively.

Metric-level labels emitted by Emissary/Envoy (as Prometheus labels, not OTel resource attributes):
- `envoy_cluster_name` — internal cluster name (e.g., `otel_eval_backend_demo_3000`)
- `envoy_http_conn_manager_prefix` — connection manager name (e.g., `ingress_https`)
- `envoy_listener_address` — listener socket address
- `envoy_rds_route_config` — RDS route config name
- `envoy_response_code` — HTTP response code
- `envoy_response_code_class` — response code class (2xx, 4xx, 5xx)
- `envoy_worker_id` — Envoy worker thread ID
- `ambassador_id` — Ambassador instance identifier
- `cluster_id` — cluster identifier
- `level` — log level (on `ambassador_log_level`)
- `single_namespace` — namespace scope flag
- `version` — Ambassador version

### Resource attributes (after collector enrichment)

All resource attributes on the emissary metrics are **collector-derived** via the `k8sattributes` processor and Prometheus receiver metadata:

| Attribute | Value (example) | Source |
|-----------|-----------------|--------|
| `service.name` | `emissary-ingress` | Collector-derived (from Prometheus job/pod annotation) |
| `service.instance.id` | `10.244.0.7:8877` | Collector-derived (Prometheus target address) |
| `k8s.container.name` | `emissary-ingress` | Collector-derived (k8sattributes) |
| `k8s.pod.name` | `emissary-ingress-d8b6dc584-7r4xh` | Collector-derived (k8sattributes) |
| `k8s.namespace.name` | `emissary` | Collector-derived (k8sattributes) |
| `k8s.node.name` | `otel-eval-emissary-control-plane` | Collector-derived (k8sattributes) |
| `k8s.deployment.name` | `emissary-ingress` | Collector-derived (k8sattributes) |
| `k8s.pod.uid` | `e4d6403e-a514-4b6a-b512-1602c5403b75` | Collector-derived (k8sattributes) |
| `k8s.replicaset.name` | `emissary-ingress-d8b6dc584` | Collector-derived (k8sattributes) |
| `container.id` | `7a807b7f924...` | Collector-derived (k8sattributes) |
| `container.image.name` | `docker.io/datawire/emissary` | Collector-derived (k8sattributes) |
| `container.image.tag` | `3.12.2` | Collector-derived (k8sattributes) |
| `server.address` | `10.244.0.7` | Collector-derived (Prometheus receiver) |
| `server.port` | `8877` | Collector-derived (Prometheus receiver) |
| `url.scheme` | `http` | Collector-derived (Prometheus receiver) |
| `k8s.pod.annotation.*` | Helm checksum, prometheus scrape annotations | Collector-derived (k8sattributes) |
| `k8s.pod.label.*` | Helm labels, product, profile | Collector-derived (k8sattributes) |

**Notably absent natively**: `service.version`, `telemetry.sdk.*`, `service.namespace`, any OTLP-native resource attributes.

---

## Dimension evaluations

### 1. Integration Surface

**Level: 0 — Instrumented**

#### Evidence

Emissary's sole telemetry integration mechanism is a **Prometheus `/metrics` endpoint** on port 8877 (the diagd admin service). There is no OTLP export capability in the project itself. Users must deploy an OTel Collector with a `prometheus` receiver and scrape configuration to ingest Emissary's metrics into an OpenTelemetry pipeline.

Tracing is configured via a **`TracingService` CRD** that supports four drivers: `lightstep`, `zipkin`, `datadog`, and `opentelemetry`. The OpenTelemetry driver is explicitly marked "work-in-progress, not for production use" in Emissary 3.12.2. All tracing drivers are non-functional in 3.12.2 due to an architectural bug (ADS overrides static tracing cluster). The Zipkin driver was configured pointing to the OTel Collector's Zipkin receiver (port 9411), and while the endpoint is reachable (manual curl returns 202), zero spans were delivered (`envoy_cluster_upstream_rq_total` for the tracing cluster = 0 across all scrape intervals).

Logs are not exported at all — the `LogService` CRD uses Envoy's ALS (Access Log Service) gRPC protocol, which is incompatible with OTLP.

#### Checklist assessment

- ❌ OTLP export not supported natively (metrics: Prometheus only; traces: broken/WIP; logs: not supported)
- ❌ OpenTelemetry is not the primary integration surface — Prometheus scraping is the only working mechanism
- ❌ Users must adapt their observability stack (add OTel Collector with Prometheus receiver)
- ❌ No `OTEL_*` environment variables respected
- ✅ The `opentelemetry` tracing driver exists (though WIP/broken) — shows intent but not working implementation
- ❌ No OTLP endpoint configuration; no SDK integration

#### Rationale

Emissary's telemetry integration is entirely tool-specific: Prometheus scraping for metrics, a proprietary CRD-based tracing configuration that doesn't work, and no log export. Users must deploy and configure an OTel Collector as an adapter. OpenTelemetry is mentioned in the `TracingService` CRD but is explicitly labeled as non-production. This is a clear Level 0 profile: telemetry exists, but the integration surface is not OpenTelemetry-oriented.

---

### 2. Semantic Conventions

**Level: 0 — Instrumented**

#### Evidence

##### Metric names and attributes

All 407 metrics use **Emissary/Envoy-proprietary naming conventions**:

- `envoy_cluster_upstream_rq_total` — Envoy's internal counter name
- `envoy_http_downstream_cx_active` — Envoy connection manager naming
- `ambassador_aconf_time_seconds` — Emissary control plane naming
- `ambassador_reconfiguration_time_seconds` — Emissary-specific
- `envoy_http_tracing_random_sampling` — Envoy tracing subsystem counter

Metric labels use Envoy-internal names:
- `envoy_cluster_name` (not `network.destination.name` or similar OTel semconv)
- `envoy_http_conn_manager_prefix` (not `http.server_name` or `server.address`)
- `envoy_response_code` (not `http.response.status_code`)
- `envoy_response_code_class` (not OTel-standard)
- `envoy_worker_id` (internal Envoy threading concept)

There is no alignment with OTel semantic conventions for metrics. The naming follows Envoy's internal stats subsystem naming, not the OTel `http.*`, `rpc.*`, `network.*`, or other established metric namespaces.

##### Trace attributes

Emissary itself emits **no spans**. The traces in the telemetry files are from the `otel-eval-backend` Node.js service, which uses deprecated OTel HTTP semantic conventions:

- `http.method` ← deprecated; should be `http.request.method`
- `http.status_code` ← deprecated; should be `http.response.status_code`
- `http.target` ← deprecated; should be `url.path`
- `http.url` ← deprecated; should be `url.full`
- `http.host` ← deprecated; should be `server.address`
- `http.flavor` ← deprecated; should be `network.protocol.version`
- `http.scheme` ← deprecated; should be `url.scheme`

These are from the backend test service, not Emissary itself, but they confirm that no OTel-conformant tracing data exists in the pipeline from the Emissary component.

##### Log attributes

No structured log data flows through the OTel pipeline from Emissary.

#### Checklist assessment

- ❌ No OTel semantic conventions used in metric names or labels
- ❌ Proprietary `envoy_*` naming throughout — mirrors Envoy's internal stats system
- ❌ Metric labels use Envoy-internal names (`envoy_cluster_name`, `envoy_http_conn_manager_prefix`) rather than OTel equivalents
- ❌ No schema URL on metrics (the `https://opentelemetry.io/schemas/1.18.0` schema URL seen in the data is from the `k8sclusterreceiver`, not from Emissary)
- ❌ No traces from Emissary to evaluate span attribute conventions
- ❌ No documentation referencing OTel semantic conventions

#### Rationale

Emissary's metric naming is entirely derived from Envoy's internal stats subsystem, which predates OTel and uses a completely different naming philosophy. There is no alignment with OTel semantic conventions for HTTP metrics, network metrics, or any other domain. The metric label names are Envoy-specific internal identifiers. This is firmly Level 0: telemetry meaning is implicit and tied to Envoy/Emissary internals, requiring project-specific knowledge to interpret.

---

### 3. Resource Attributes & Configuration

**Level: 0 — Instrumented**

#### Evidence

##### Native resource attributes

Emissary emits **zero native resource attributes**. The project does not use an OTel SDK and does not set any resource attributes in its telemetry output. The Prometheus `/metrics` endpoint is a flat text format with no concept of resource identity.

Confirmed: the emissary container resource in the metrics JSONL has **no `service.name` set natively**. The `service.name: emissary-ingress` value observed in the telemetry is **collector-derived** — the OTel Collector's Prometheus receiver infers the service name from the pod annotations and scrape job configuration. Similarly, all `k8s.*` attributes are added by the `k8sattributes` processor.

##### OTEL_* environment variable support

Emissary does not support any `OTEL_*` environment variables. The project uses a Python-based control plane (diagd) and the Envoy proxy data plane, neither of which is instrumented with an OTel SDK. There is no `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, or `OTEL_RESOURCE_ATTRIBUTES` support.

##### Identity consistency across signals

- **Metrics**: `service.name = emissary-ingress` (collector-derived from Prometheus job)
- **Traces**: No traces from Emissary
- **Logs**: No logs from Emissary
- **`service.version`**: Absent from all signals (even collector-enriched)
- **`service.instance.id`**: Collector-derived as `10.244.0.7:8877` (Prometheus target address) — not a stable pod identity

There is no cross-signal identity consistency because only metrics are flowing, and that identity is entirely pipeline-derived.

#### Checklist assessment

- ❌ `service.name` is not set at the source — it is collector-derived
- ❌ `service.version` is absent from all signals
- ❌ No `OTEL_*` environment variable support
- ❌ Resource attributes are not the source of identity — Prometheus scrape metadata is
- ❌ Identity is not stable across environments (changes based on collector configuration)
- ❌ No OTel SDK usage in the project

#### Rationale

Emissary has no concept of OTel resource identity. It is a Prometheus-native application from an observability perspective. All identity attributes (service name, Kubernetes metadata, container info) are injected by the OTel Collector pipeline. This is Level 0: resource identity is implicit and entirely dependent on downstream enrichment.

---

### 4. Trace Modeling & Context Propagation

**Level: 0 — Instrumented**

#### Evidence

##### Span structure

Emissary emits **zero spans**. Despite a `TracingService` CRD being applied (Zipkin driver pointing to the OTel Collector), the tracing cluster received 0 requests across all scrape intervals:

```
envoy_cluster_upstream_rq_total{envoy_cluster_name="otel_collector_opentelemetry_collector_opentelemetry_9411"} = 0
```

Meanwhile, Envoy correctly sampled requests:
```
envoy_http_tracing_random_sampling{envoy_http_conn_manager_prefix="ingress_https"} = 75
```

This confirms that Envoy is sampling requests and intending to send spans, but the span export is silently failing due to the architectural bug where Emissary's ADS server overrides the static tracing cluster with a dynamic one.

##### Context propagation

Emissary **does pass through** the `traceparent` W3C Trace Context header to upstream services. Evidence: the backend (`otel-eval-backend`) traces show parent span IDs (e.g., `parentSpanId=4f21ca495756ad33`), indicating that when a request arrives with a `traceparent` header, the backend receives it and creates child spans. However, these parent spans do not originate from Emissary — they would originate from the test client. Emissary acts as a transparent pass-through for trace headers, not as a trace-generating participant.

##### Trace coherence

No Emissary-generated traces exist. The traces in the telemetry files are exclusively from the `otel-eval-backend` Node.js service (service.name = `otel-eval-backend`), representing the backend's internal span tree (HTTP server span + Express middleware spans). There is no Emissary span as a parent.

#### Checklist assessment

- ❌ No spans emitted by Emissary (0 traces from the project itself)
- ❌ Tracing is architecturally broken in 3.12.2 (ADS cluster override bug)
- ❌ OpenTelemetry tracing driver explicitly marked WIP/non-production
- ✅ W3C `traceparent` headers are passed through to upstream services (passive propagation)
- ❌ Emissary does not inject its own span into the trace chain
- ❌ No parent-child relationship between Emissary and upstream service spans

#### Rationale

Emissary's tracing support is non-functional in the evaluated version. The project samples requests and attempts to export spans, but the span export fails silently due to a known architectural bug in xDS cluster handling. The OpenTelemetry driver is explicitly labeled as "work-in-progress." Header pass-through (W3C `traceparent`) is a positive finding, but it is passive forwarding, not active trace participation. This is Level 0: spans are not emitted, and trace structure is non-existent.

---

### 5. Multi-Signal Observability

**Level: 0 — Instrumented**

#### Evidence

##### Signal availability

| Signal | Status | Method | Notes |
|--------|--------|--------|-------|
| Metrics | ✅ Flowing | Prometheus scrape (port 8877) | 407 unique metric names; envoy_* + ambassador_* |
| Traces | ❌ Not flowing | TracingService CRD (broken) | 0 spans sent despite 75 requests sampled |
| Logs | ❌ Not flowing | None (ALS not OTLP) | No OTLP log export capability |

Only one signal (metrics) is first-class and functional. Traces are architecturally broken. Logs are not supported via OTLP.

##### Cross-signal correlation

There is no cross-signal correlation from Emissary:
- Metrics have no trace context (no `trace_id`, `span_id` on metric data points)
- No logs to correlate
- No traces from Emissary itself

The only traces in the system come from the `otel-eval-backend` service, which has its own independent OTel SDK instrumentation. These backend traces cannot be correlated with Emissary's Prometheus metrics because:
1. Emissary metrics have no trace context
2. There is no shared `service.name` or resource attribute linking them
3. Emissary does not generate a parent span that the backend traces could attach to

##### Collection model

- **Metrics**: Prometheus scrape (pull model) — requires OTel Collector with `prometheus` receiver
- **Traces**: Would require Zipkin or OTel push from Emissary — currently broken
- **Logs**: No OTLP support — would require log file collection (not configured)

#### Checklist assessment

- ❌ Only one signal (metrics) is first-class and functional
- ❌ Traces are absent from Emissary
- ❌ Logs are absent from Emissary
- ❌ No shared correlation context between signals
- ❌ Users cannot pivot from metrics to traces or logs
- ❌ Investigation requires entirely separate tooling for each signal

#### Rationale

Emissary is firmly at Level 0 for multi-signal observability. Only metrics are available, and they are exposed via a legacy Prometheus endpoint. Traces and logs are non-functional. There is no cross-signal correlation. Users investigating an issue using Emissary's metrics have no way to drill into traces or correlate with logs from the same request path. This is the classic "single-signal focus" pattern described at Level 0.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Metric signal quality

Emissary exposes 407 unique Prometheus metrics covering the full Envoy proxy and Ambassador control plane. Despite non-OTel naming, the metrics provide genuine operational value:

**Operational metrics (useful for operators):**
- `envoy_http_downstream_rq_total` — total downstream requests (with `envoy_http_conn_manager_prefix` label for per-listener breakdown)
- `envoy_cluster_upstream_rq_total` — per-cluster upstream request counts
- `envoy_http_downstream_cx_active` — active connections
- `envoy_cluster_circuit_breakers_*` — circuit breaker state
- `ambassador_reconfiguration_time_seconds` — control plane config reload latency
- `ambassador_diagnostics_errors` — configuration error count

**Internal/debug metrics (less useful for operators):**
- `envoy_worker_id`-dimensioned metrics — per-thread metrics that most operators don't need
- `ambassador_fetcher_time_seconds` — internal control plane fetch timing
- `envoy_cluster_lb_subsets_*` — subset load balancing internals

The metrics are comprehensive but not curated. All Envoy stats are exposed, including many that are only relevant for deep debugging of Envoy internals. There is no tiering between "operational" and "debug" metrics.

##### Metric label usability

The metric labels use Envoy-internal naming that requires Envoy/Emissary knowledge to interpret:
- `envoy_cluster_name` values like `otel_eval_backend_demo_3000` (derived from service address, not a human-readable name)
- `envoy_http_conn_manager_prefix` values like `ingress_https`, `ready_http`, `admin` (Envoy-internal names)
- `envoy_response_code` is a numeric HTTP status code (usable but not labeled semantically)

##### Default usability

The 407-metric Prometheus endpoint is usable for operators familiar with Envoy, but requires domain knowledge. Off-the-shelf OTel dashboards would not work — users need Envoy-specific dashboards. The breadth is impressive, but the signal-to-noise ratio is low for operators unfamiliar with Envoy internals.

#### Checklist assessment

- ✅ Metrics provide genuine operational value (request rates, error rates, latency, circuit breakers)
- ✅ Some noise reduction compared to raw Envoy stats (stats are grouped by subsystem)
- ❌ Metric names require Envoy-specific knowledge to interpret
- ❌ No tiering between operational and debug metrics
- ❌ Metric labels use internal Envoy naming (not human-readable or OTel-aligned)
- ❌ No traces or logs to evaluate for quality
- ❌ Off-the-shelf OTel dashboards do not work without normalization

#### Rationale

Emissary scores Level 1 here — the metrics provide real operational value and are comprehensive, showing deliberate effort to expose the full Envoy stats surface. However, they are shaped by Envoy's internal naming conventions rather than user-oriented design. Operators familiar with Envoy can use them effectively, but operators without Envoy knowledge face a steep learning curve. The lack of traces and logs prevents assessing signal quality across the full observability surface.

---

### 7. Stability & Change Management

**Level: 0 — Instrumented**

#### Evidence

##### Documentation of telemetry behavior

Emissary's documentation does not treat telemetry as a stable contract:
- The `TracingService` CRD documentation lists supported drivers but does not document which are stable vs experimental
- The OpenTelemetry tracing driver is labeled "work-in-progress, not for production use" in the runtime logs but this is not prominently documented in official docs
- There is no telemetry reference page documenting metric names, labels, and their meanings
- No changelog sections specifically tracking telemetry changes

##### Schema URL presence

- Metrics: **No schema URL** on the Prometheus receiver scope (the `https://opentelemetry.io/schemas/1.18.0` observed is from the `k8sclusterreceiver`, not from Emissary)
- Traces: No traces from Emissary
- Logs: No logs from Emissary

##### Stability guarantees

- The `opentelemetry` tracing driver is explicitly marked non-production in the current stable release (3.12.2)
- The tracing bug (ADS cluster override) is a regression/architectural issue with no documented workaround in official docs
- Metric names follow Envoy's internal stats naming, which can change when Envoy is upgraded (Emissary upgrades Envoy as a dependency)
- No explicit stability guarantees for any telemetry signal

##### Change communication

Review of Emissary release notes and changelogs does not show dedicated sections for telemetry changes. Metric changes that result from Envoy version bumps are not called out as telemetry breaking changes. The tracing WIP status is not prominently surfaced in the documentation.

#### Checklist assessment

- ❌ No telemetry documentation as a stable contract
- ❌ No schema URL on any Emissary-native telemetry
- ❌ No distinction between stable and experimental telemetry (beyond runtime WIP warning)
- ❌ No changelog sections for telemetry changes
- ❌ Metric stability depends on Envoy upstream (undocumented dependency)
- ❌ Tracing driver marked WIP with no documented timeline for stabilization
- ❌ No migration guidance for telemetry changes

#### Rationale

Emissary's telemetry is treated as an implementation detail, not a user-facing contract. There is no schema URL, no documented telemetry stability policy, and no process for communicating telemetry changes. The fact that the OpenTelemetry tracing driver is broken in the current stable release with no documented workaround or timeline is a strong indicator of Level 0 stability maturity. Users building dashboards or alerts on Emissary's Prometheus metrics are implicitly accepting that metric names may change with Envoy upgrades.

---

## Key findings

### Strengths

- **Comprehensive Prometheus metrics**: 407 unique Envoy + Ambassador metrics provide deep observability into the proxy data plane and control plane. Coverage includes request rates, error rates, circuit breaker states, connection counts, and config compilation timings — genuinely useful for operators familiar with Envoy.
- **W3C Trace Context pass-through**: Emissary correctly forwards `traceparent` headers to upstream services, enabling downstream services to participate in distributed traces initiated by clients. This is correct behavior for a transparent proxy.
- **Control plane metrics (`ambassador_*`)**: Unique metrics like `ambassador_reconfiguration_time_seconds`, `ambassador_ir_time_seconds`, and `ambassador_diagnostics_errors` expose Emissary-specific control plane internals that are valuable for diagnosing configuration issues.

### Areas for improvement

1. **Fix tracing before adding features**: The ADS cluster override bug that prevents all tracing drivers from working is the highest-priority issue. Until spans can actually be delivered, all tracing configuration is theater. The fix requires ensuring the tracing cluster is not overridden by dynamic cluster discovery.
2. **Adopt OTel SDK for OTLP-native metrics export**: Replace or supplement the Prometheus endpoint with OTLP metrics export using an OTel SDK. This would enable native resource attributes (`service.name`, `service.version`), proper metric naming aligned with OTel semantic conventions, and direct integration with OTel Collectors without requiring a Prometheus receiver.
3. **Align metric naming with OTel semantic conventions**: Metric names like `envoy_http_downstream_rq_total` should be mapped to or replaced with OTel-standard names (e.g., `http.server.request.duration` as a histogram, `http.server.active_requests` as a gauge). Metric labels like `envoy_response_code` should become `http.response.status_code`. This would enable off-the-shelf OTel dashboards and alerts.
4. **Add structured OTLP log export**: Implement OTLP log export for Emissary's access logs and control plane logs. This would complete the three-signal observability picture and enable log-trace correlation.
5. **Document telemetry as a stable contract**: Create a telemetry reference page documenting all metric names, labels, and their stability status. Introduce schema URLs. Track telemetry changes in changelogs with explicit breaking change notices.

### Notable observations

1. **Tracing is fundamentally broken in 3.12.2**: The combination of Emissary's ADS server pushing the tracing cluster as a dynamic cluster (overriding the static bootstrap definition) makes all four tracing drivers non-functional. Envoy samples requests correctly (`envoy_http_tracing_random_sampling: 75`) but delivers zero spans (`upstream_rq_total: 0` for the tracing cluster). This is a silent failure — no error is logged, no metric indicates the problem.
2. **OpenTelemetry driver explicitly WIP**: Emissary's own runtime logs warn: "The OpenTelemetry tracing driver is work-in-progress. Functionality is incomplete and it is not intended for production use." This appears in the stable 3.12.2 release, suggesting the project has not yet committed to a stable OTel tracing integration timeline.
3. **407 metrics with no OTel alignment**: The sheer breadth of Prometheus metrics is impressive and provides real operational value. However, this creates a large surface area of non-OTel-aligned telemetry that would require significant renaming effort to align with semantic conventions.
4. **v4.x exists but no Helm chart**: The latest GitHub release (v4.0.1) is not available in the datawire Helm repository. v4 appears to be a major rewrite, potentially with improved OTel support, but is not yet accessible via standard installation methods.
5. **`service.name` is entirely collector-derived**: There is no `service.name` set by Emissary itself. The value `emissary-ingress` in the telemetry is inferred by the OTel Collector from the Prometheus scrape job and pod annotations. This means users who change their collector configuration could inadvertently change the service identity.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file export in a local kind cluster (`otel-eval-emissary`)
- The `k8sattributes` processor was used; all resource attributes on Emissary metrics are pipeline-derived, not project-native
- The Prometheus receiver scraped Emissary's `/metrics` endpoint on port 8877 via pod annotation discovery
- A `TracingService` CRD was applied with both OpenTelemetry (gRPC/4317) and Zipkin (HTTP/9411) drivers — both produced 0 spans
- Traffic was generated via port-forward to the Emissary service (75+ requests for the `ingress_https` listener)
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv 1.23+)
- Documentation review covered the Emissary Ingress official docs at getambassador.io and the GitHub repository
- The `otel-eval-backend` Node.js service traces (which are present in the telemetry files) are from the downstream test service, not from Emissary itself, and use deprecated OTel HTTP semantic conventions (`http.method`, `http.status_code`, etc.)
