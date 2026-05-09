# OpenTelemetry Support Maturity Evaluation: Emissary-Ingress

## Project overview

- **Project**: Emissary-Ingress (formerly Ambassador) — a CNCF Incubating Kubernetes-native API Gateway and Ingress controller built on Envoy Proxy
- **Version evaluated**: 3.12.2 (Helm chart 8.12.2)
- **Evaluation date**: 2026-05-09
- **Cluster**: otel-eval-emissary
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 1 | OTLP driver exists but is non-functional; working path uses Zipkin protocol via a separate CRD |
| Semantic Conventions | 0 | Deprecated HTTP attributes throughout; proprietary Envoy-internal attribute names; no OTel semconv alignment |
| Resource Attributes & Configuration | 0 | No `service.version`; `service.name` differs between traces (`ambassador-emissary`) and metrics (`emissary`); no `OTEL_*` env var support |
| Trace Modeling & Context Propagation | 1 | Coherent parent-child structure within Emissary, but B3 propagation prevents end-to-end correlation with W3C-native downstream services |
| Multi-Signal Observability | 1 | Traces and metrics both flow; logs are stdout-only with no OTLP export; no cross-signal correlation |
| Audience & Signal Quality | 1 | Span names are host:port strings and internal cluster identifiers; metrics are comprehensive but use Envoy-internal naming |
| Stability & Change Management | 1 | Some telemetry changes appear in release notes; no formal telemetry stability contract; `opentelemetry` driver documented but non-functional |

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — Zipkin v2 JSON HTTP export to OTel Collector port 9411 (via `TracingService` CRD with `driver: zipkin`)
- **Metrics**: Flowing — Prometheus scrape at `:8877/metrics` (443 unique metric names; 375 `envoy_*`, 16 `ambassador_*`, plus collector self-metrics)
- **Logs**: Not flowing via OTLP — Envoy access logs written to stdout in text format only

### Resource attributes (native, before collector enrichment)

Emissary (via Envoy's Zipkin exporter) natively emits only one resource attribute:

| Attribute | Value | Source |
|-----------|-------|--------|
| `service.name` | `ambassador-emissary` | Derived from Envoy node cluster name; not configurable via TracingService in Zipkin mode |

No `service.version`, `service.instance.id`, `telemetry.sdk.*`, or any other resource attributes are emitted natively by Emissary.

For metrics, the Prometheus scrape target produces no resource attributes natively. The `service.name: emissary` and `service.instance.id: emissary-emissary-ingress-admin.emissary.svc.cluster.local:8877` values are set by the OTel Collector's Prometheus receiver based on the scrape job configuration.

### Resource attributes (after collector enrichment)

After k8sattributes processing, the following attributes appear on trace data:

```
service.name: ambassador-emissary
k8s.namespace.name: emissary
k8s.pod.name: emissary-emissary-ingress-68dcbc4ddf-8zr5q
k8s.pod.uid: 4599fcbc-3805-4986-b20e-172b183f789e
k8s.pod.start_time: 2026-05-09T17:39:11Z
k8s.deployment.name: emissary-emissary-ingress
k8s.replicaset.name: emissary-emissary-ingress-68dcbc4ddf
k8s.node.name: otel-eval-emissary-control-plane
k8s.pod.label.app.kubernetes.io/name: emissary-ingress
k8s.pod.label.app.kubernetes.io/instance: emissary
k8s.pod.label.helm.sh/chart: emissary-ingress-8.12.2
k8s.pod.label.product: aes
k8s.pod.annotation.prometheus.io/scrape: true
k8s.pod.annotation.prometheus.io/port: 8877
```

All `k8s.*` attributes are collector-derived. No `service.version` is present even after enrichment.

---

## Dimension evaluations

### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

Emissary provides a `TracingService` CRD that nominally supports an `opentelemetry` driver (OTLP gRPC) alongside `zipkin`, `lightstep`, and `datadog`. However, the `opentelemetry` driver is explicitly marked "work-in-progress" and does not function in practice — it logs a warning and produces no spans. The only working trace export path is via the `zipkin` driver, which sends Zipkin v2 JSON over HTTP to port 9411. This is a legacy protocol that requires the OTel Collector to be configured with a Zipkin receiver.

Metrics are exposed only via a Prometheus endpoint at `:8877/metrics`. There is no OTLP metrics export path.

Configuration is entirely through Emissary-specific CRDs (`TracingService`, `LogService`) and Helm chart values, not through `OTEL_*` environment variables or OTel SDK configuration.

Observed working integration:
- Traces: Zipkin v2 JSON → OTel Collector Zipkin receiver → traces pipeline
- Metrics: Prometheus scrape → OTel Collector Prometheus receiver → metrics pipeline
- Logs: stdout text → not captured via OTLP

#### Checklist assessment

- ❌ OTLP is not the default or working export mechanism (the `opentelemetry` driver is non-functional)
- ❌ Standard `OTEL_*` environment variables are not respected
- ❌ Users cannot connect to an existing OTel Collector using standard OTLP configuration without adapters
- ✅ OTLP is nominally supported (the CRD field exists and is documented)
- ✅ Zipkin export works as an alternative path to an OTel Collector
- ✅ Prometheus scraping is well-supported and documented
- ❌ OpenTelemetry is not the "happy path" — it requires significant workarounds

#### Rationale

Level 1 (OpenTelemetry-Aligned) is appropriate. OpenTelemetry is explicitly referenced in the `TracingService` CRD schema and in the changelog, and the intent to support OTLP is clear. However, the `opentelemetry` driver is non-functional, leaving users to rely on the Zipkin protocol as an indirect path to OTel pipelines. Metrics use Prometheus scraping exclusively. The integration "works" but requires adapters and project-specific knowledge. This is characteristic of Level 1: OpenTelemetry is supported but not central, and legacy integrations remain the only reliable paths.

---

### 2. Semantic Conventions

**Level: 0 — Instrumented**

#### Evidence

##### Trace attributes

Emissary (via Envoy's Zipkin tracing) emits the following span attributes, all of which are deprecated or non-standard relative to current OpenTelemetry semantic conventions:

**Ingress span (`localhost:8090`):**
| Attribute | Value | Status |
|-----------|-------|--------|
| `http.url` | `http://localhost:8090/` | **Deprecated** — current semconv uses `url.full` |
| `http.method` | `GET` | **Deprecated** — current semconv uses `http.request.method` |
| `http.status_code` | `200` | **Deprecated** — current semconv uses `http.response.status_code` |
| `http.protocol` | `HTTP/1.1` | **Deprecated** — current semconv uses `network.protocol.name` + `network.protocol.version` |
| `user_agent` | `curl/8.18.0` | **Non-standard** — current semconv uses `user_agent.original` |
| `peer.address` | `127.0.0.1` | **Non-standard** — current semconv uses `network.peer.address` |
| `downstream_cluster` | `-` | **Envoy-internal** — no OTel equivalent |
| `node_id` | `test-id` | **Envoy-internal** — no OTel equivalent |
| `request_size` | `0` | **Non-standard** — current semconv uses `http.request.body.size` |
| `response_size` | `161` | **Non-standard** — current semconv uses `http.response.body.size` |
| `response_flags` | `-` | **Envoy-internal** — no OTel equivalent |
| `guid:x-request-id` | UUID | **Envoy-internal** — no OTel equivalent |
| `component` | `proxy` | **Deprecated** — legacy attribute |
| `net.host.ip` | IP address | **Deprecated** — current semconv uses `network.local.address` |
| `upstream_cluster` | Internal cluster name | **Envoy-internal** |
| `upstream_cluster.name` | DNS-style cluster name | **Envoy-internal** |

**Egress span (`router ... egress`):**
| Attribute | Value | Status |
|-----------|-------|--------|
| `upstream_address` | `10.96.157.165:3000` | **Non-standard** |
| `upstream_cluster` | Internal cluster name | **Envoy-internal** |
| `upstream_cluster.name` | DNS-style name | **Envoy-internal** |
| `http.protocol` | `HTTP/1.1` | **Deprecated** |
| `http.status_code` | `200` | **Deprecated** |
| `response_flags` | `-` | **Envoy-internal** |
| `peer.address` | IP:port | **Non-standard** |
| `component` | `proxy` | **Deprecated** |

**Span kinds:** Both ingress and egress spans use `kind=3` (CLIENT). The ingress span that receives external traffic should be `kind=2` (SERVER) per OTel semantic conventions for HTTP server spans.

**Instrumentation scope:** The scope name and version are empty (`{}`). OTel requires a non-empty scope name identifying the instrumentation library.

##### Metric names and attributes

Emissary metrics use Envoy's native Prometheus naming (`envoy_*` prefix) and Ambassador-specific naming (`ambassador_*` prefix). These are entirely proprietary and do not align with OTel semantic conventions:

- `envoy_cluster_upstream_rq_total` — not `http.server.request.duration` or similar
- `envoy_http_downstream_rq_2xx` — not `http.server.response.status_code`
- `ambassador_reconfiguration_time_seconds` — proprietary control-plane metric
- Metric label keys: `envoy_cluster_name`, `envoy_response_code`, `envoy_response_code_class` — Envoy-internal naming, not OTel semconv

##### Log attributes

No OTLP log export. Stdout access logs use Envoy's text format — not OTel structured log format.

#### Checklist assessment

- ❌ Current OTel HTTP semantic conventions are not used (`http.request.method`, `http.response.status_code`, `url.full`, `url.path`, `network.protocol.*`)
- ❌ Deprecated attributes are used throughout (`http.method`, `http.status_code`, `http.url`, `http.target`, `peer.address`, `component`, `net.host.ip`)
- ❌ Proprietary/Envoy-internal attributes dominate (`downstream_cluster`, `node_id`, `response_flags`, `guid:x-request-id`, `upstream_cluster`, `upstream_cluster.name`)
- ❌ Span kinds are incorrect (ingress SERVER spans reported as CLIENT)
- ❌ Instrumentation scope is empty — no library name or version
- ❌ Metric names do not follow OTel semantic conventions
- ❌ No schema URL set on any telemetry

#### Rationale

Level 0 (Instrumented) is the correct assessment. While some attribute names superficially resemble OTel conventions (e.g., `http.method` and `http.status_code` are OTel-ish), they are deprecated versions of current conventions and appear to be inherited from Envoy's pre-OTel instrumentation era rather than from intentional OTel adoption. The majority of attributes are Envoy-internal proprietary names with no OTel equivalent. Metric naming is entirely Envoy-native. No schema URL is set. Span kinds are incorrect. Telemetry meaning is implicit and requires Envoy-specific knowledge to interpret.

---

### 3. Resource Attributes & Configuration

**Level: 0 — Instrumented**

#### Evidence

##### Native resource attributes

Emissary natively emits only `service.name: ambassador-emissary` on trace data. This value is derived from the Envoy node cluster name, which is an internal Envoy concept, not a user-configured service identity. It cannot be overridden via TracingService configuration in Zipkin mode.

For metrics, the `service.name: emissary` is set by the OTel Collector's Prometheus receiver job configuration — it is not emitted by Emissary itself.

**Critical inconsistency:** `service.name` differs between signals:
- Traces: `ambassador-emissary` (Envoy node cluster name)
- Metrics: `emissary` (Prometheus job_name set in collector config)
- Logs: absent (no OTLP export)

**Missing attributes:**
- No `service.version` in any signal (not even after collector enrichment)
- No `service.instance.id` emitted natively
- No `telemetry.sdk.name`, `telemetry.sdk.version`, or `telemetry.sdk.language`
- No `service.namespace`

##### OTEL_* environment variable support

Emissary does not use the OTel SDK and does not respect `OTEL_*` environment variables. Configuration is exclusively through:
- `TracingService` CRD (tracing endpoint and driver)
- Helm chart values (admin service, service type)
- Ambassador environment variables (`AMBASSADOR_*`)

Setting `OTEL_SERVICE_NAME` or `OTEL_RESOURCE_ATTRIBUTES` has no effect.

##### Identity consistency across signals

| Signal | `service.name` | `service.version` | `service.instance.id` |
|--------|---------------|-------------------|----------------------|
| Traces | `ambassador-emissary` | absent | absent |
| Metrics | `emissary` | absent | `emissary-emissary-ingress-admin.emissary.svc.cluster.local:8877` (collector-derived) |
| Logs | N/A | N/A | N/A |

The `service.name` inconsistency means traces and metrics from the same Emissary pod cannot be correlated by service identity without custom mapping.

#### Checklist assessment

- ❌ `service.name` is inconsistent across signals (different values in traces vs metrics)
- ❌ `service.version` is absent from all signals
- ❌ `service.instance.id` is absent from native trace data
- ❌ `OTEL_*` environment variables are not respected
- ❌ Identity is derived from internal Envoy concepts (node cluster name), not user-configurable
- ❌ No `telemetry.sdk.*` attributes (Emissary does not use the OTel SDK)

#### Rationale

Level 0 (Instrumented) is the correct assessment. Resource identity is incomplete and inconsistent. The `service.name` differs between traces and metrics, making cross-signal correlation by identity impossible without custom pipeline transforms. There is no `service.version`. The OTel SDK is not used, so standard configuration mechanisms (`OTEL_*`) have no effect. This is characteristic of Level 0: identity exists implicitly but is not treated as a coherent system.

---

### 4. Trace Modeling & Context Propagation

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span structure

Emissary produces two spans per request:

1. **Ingress span** (`localhost:8090`): Root span (no parent) representing the inbound HTTP request. Contains client-facing attributes: `http.url`, `http.method`, `http.status_code`, `user_agent`, `request_size`, `response_size`.

2. **Egress span** (`router otel_eval_backend_demo_svc_cluster_local_3000 egress`): Child of the ingress span (correct parent-child relationship). Contains upstream routing attributes: `upstream_address`, `upstream_cluster.name`, `http.status_code`, `response_flags`.

The parent-child relationship between these two spans is correct and coherent within Emissary's own trace tree:
```
localhost:8090 (kind=3, root)
└── router otel_eval_backend_demo_svc_cluster_local_3000 egress (kind=3, child)
```

##### Context propagation

Emissary (via Envoy's Zipkin driver) injects **B3 headers** (`x-b3-traceid`, `x-b3-spanid`, `x-b3-sampled`) into upstream requests. The downstream backend (`otel-eval-backend`) uses the OTel Node.js SDK with W3C Trace Context (`traceparent`).

**Confirmed broken propagation:** Analysis of trace IDs shows zero overlap between Emissary trace IDs and backend trace IDs:
- Emissary trace IDs: `883c13ef...`, `74a997df...`, `374961c3...`, `4196d20e...`, `44da71a8...`, `617bb0dd...`
- Backend trace IDs: `586a8fc6...`, `7562c4e2...`, `ac262fe8...`, `c77385a0...`, `ddf82cb5...`, `f4dafd0c...`

No shared trace IDs exist. Backend `GET /` spans have no parent span ID (they start fresh traces). The B3 headers injected by Emissary are not recognized by the backend's W3C Trace Context propagator, resulting in **completely separate, uncorrelated trace trees** for every request.

##### Trace coherence

Within Emissary, the two-span structure is internally coherent. However, the trace does not extend to the backend service, making end-to-end distributed tracing impossible in the default configuration.

Span names use internal Envoy representations:
- `localhost:8090` — the port-forwarded address (not a meaningful operation name)
- `router otel_eval_backend_demo_svc_cluster_local_3000 egress` — the full Kubernetes DNS cluster name (not a meaningful operation name)

Both spans use `kind=3` (CLIENT). The ingress span that receives external traffic should be `kind=2` (SERVER).

#### Checklist assessment

- ✅ Parent-child relationship within Emissary's own spans is correct
- ✅ Sampling is configurable (100% in this evaluation)
- ❌ W3C Trace Context (`traceparent`) is not propagated — Zipkin driver uses B3 only
- ❌ Emissary and backend traces are completely uncorrelated (different trace IDs)
- ❌ Span kinds are incorrect (ingress SERVER span reported as CLIENT)
- ❌ Span names reflect internal Envoy infrastructure, not logical operations
- ❌ No `traceparent` header injection for W3C-compatible downstream services

#### Rationale

Level 1 (OpenTelemetry-Aligned) is the correct assessment. The internal parent-child span structure is coherent for synchronous request/response flows, which is characteristic of Level 1. However, the B3 propagation protocol is incompatible with W3C Trace Context used by modern OTel-instrumented services, causing broken end-to-end traces. This is the defining limitation of Level 1: "works for common paths but breaks down in more complex cases." The span kind incorrectness and non-meaningful span names prevent reaching Level 2.

---

### 5. Multi-Signal Observability

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Signal availability

| Signal | Status | Method | Protocol |
|--------|--------|--------|----------|
| Traces | Flowing | TracingService CRD | Zipkin v2 JSON HTTP |
| Metrics | Flowing | Prometheus scrape at :8877 | Prometheus text |
| Logs | Not flowing (OTLP) | stdout text | N/A |

Traces and metrics are both available but use completely different export mechanisms (Zipkin vs Prometheus scrape). Logs are not available via OTLP; Envoy access logs go to stdout in text format only. The `LogService` CRD can forward to a gRPC access log service, but this is not OTLP.

##### Cross-signal correlation

There is no cross-signal correlation:

1. **Traces ↔ Metrics**: `service.name` differs (`ambassador-emissary` in traces vs `emissary` in metrics). No shared attribute allows joining traces to metrics for the same Emissary instance.

2. **Traces ↔ Logs**: No OTLP log export. Stdout access logs contain no trace or span IDs.

3. **Emissary traces ↔ Backend traces**: Completely separate trace trees due to B3/W3C propagation mismatch (confirmed: 0 shared trace IDs).

##### Collection model

Each signal requires a separate collector configuration:
- Traces: Zipkin receiver on port 9411 must be explicitly added to the traces pipeline
- Metrics: Prometheus scrape job must be configured with the admin service address
- Logs: Not collected via OTLP

#### Checklist assessment

- ✅ Two of three signals (traces, metrics) are available
- ❌ Logs are not available via OTLP (only stdout text)
- ❌ Traces and metrics cannot be correlated by service identity (`service.name` differs)
- ❌ No trace context in logs
- ❌ Emissary traces and backend traces are uncorrelated
- ❌ Investigation requires switching between completely separate signal streams
- ❌ All three signals must be reached Level 2: logs are not first-class

#### Rationale

Level 1 (OpenTelemetry-Aligned) is the correct assessment. Two signals (traces and metrics) are present and flowing, which exceeds Level 0. However, the signals are loosely connected at best: different `service.name` values prevent trace-metric correlation, logs are not available via OTLP, and the propagation mismatch prevents end-to-end trace correlation with downstream services. The spec requires all three signals to be first-class for Level 2, and logs do not qualify here.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Emissary span names reflect internal Envoy infrastructure rather than logical operations:

- `localhost:8090` — the port-forward address used during testing; in production this would be the listener address. Not a meaningful operation name for users.
- `router otel_eval_backend_demo_svc_cluster_local_3000 egress` — the full Kubernetes DNS name of the upstream cluster. Useful for debugging routing, but verbose and implementation-specific.

These span names would not be usable with off-the-shelf dashboards or alerts. A user-oriented name would be something like `HTTP GET /` (for the ingress span) and `upstream otel-eval-backend` (for the egress span).

##### Signal-to-noise ratio

**Metrics**: 443 unique metric names are scraped. This includes 375 `envoy_*` metrics covering low-level Envoy internals (HTTP/2 frame counters, circuit breaker states per connection type, etc.) and 16 `ambassador_*` control-plane metrics. The volume is very high and includes many metrics that will be zero in most deployments. However, the metrics include operationally useful signals:
- `envoy_cluster_upstream_rq_*` — per-cluster request counters
- `envoy_http_downstream_rq_*` — downstream HTTP stats
- `envoy_tracing_zipkin_spans_sent` — tracing pipeline health
- `ambassador_reconfiguration_time_seconds` — config reload latency
- `ambassador_process_resident_memory_bytes` — memory usage

The high volume requires users to know which metrics matter, but the useful signals are present.

**Traces**: Two spans per request with meaningful HTTP attributes (method, status, URL, size). The attribute set is useful for debugging, though deprecated naming makes tool integration harder.

##### Default usability

The default configuration requires significant operator knowledge:
- `createDefaultListeners: true` must be set explicitly or Envoy never opens port 8080
- The `opentelemetry` driver is documented but non-functional
- `config.service_name` in TracingService causes a crash loop
- Pod restart required after TracingService changes
- Zipkin receiver must be manually added to the collector traces pipeline

#### Checklist assessment

- ❌ Span names expose internal infrastructure details, not logical operations
- ❌ Defaults are not usable without significant project-specific knowledge
- ✅ Metrics include operationally useful signals (request rates, latency, error rates)
- ✅ Traces include useful HTTP attributes for debugging
- ❌ High metric volume (443 metrics) without guidance on which matter
- ❌ Deprecated attribute naming requires custom dashboard normalization
- ❌ Several documented features are non-functional or have undocumented gotchas

#### Rationale

Level 1 (OpenTelemetry-Aligned) is the correct assessment. Some effort has been made toward usability — the metrics set is comprehensive and operationally useful, and trace attributes cover the key HTTP dimensions. However, span names are not user-oriented, metric volume is high without curation guidance, and the integration has significant undocumented gotchas that make defaults unsuitable for production without substantial operator knowledge. This is characteristic of Level 1: "telemetry is improving, but usability is inconsistent."

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

Emissary documents tracing via the `TracingService` CRD reference and a tracing guide on getambassador.io. The `opentelemetry` driver is listed in the CRD schema and mentioned in the changelog. However, the driver is non-functional in 3.12.2 and the documentation does not clearly communicate this limitation.

Known undocumented behaviors discovered during installation:
1. `opentelemetry` driver logs "work-in-progress" warning and produces no spans
2. `config.service_name` in ZipkinConfig causes Envoy to crash on startup
3. `createDefaultListeners: true` is required for Envoy to bind HTTP port
4. TracingService changes require pod restart (bootstrap config, not xDS)

##### Change communication

The CHANGELOG does reference telemetry-related changes:
- "Feature: In Envoy 1.24, experimental support for a native OpenTelemetry tracing driver was added" (noting it as experimental)
- "Feature: It is now possible to set `custom_tags` in the `TracingService`"
- "Feature: It is now possible to set `propagation_modes` in the `TracingService` config"
- "BREAKING CHANGE: Support for the extra metrics endpoint has been removed"
- "Fix: The Helm chart now correctly restores the `HOST_IP` value for tracing providers"

Telemetry changes are mentioned in release notes, but inconsistently — the `opentelemetry` driver's non-functional status in 3.12.2 is not clearly documented in the release notes or docs.

##### Schema URL presence

No schema URL is set on any telemetry signal. The OTLP file export shows `schemaUrl` is absent on all `resourceSpans`.

##### Stability guarantees

No explicit stability guarantees for telemetry are documented. There is no distinction between stable and experimental telemetry signals (despite the `opentelemetry` driver being effectively experimental).

#### Checklist assessment

- ✅ Some telemetry changes are mentioned in release notes
- ✅ Breaking changes to telemetry-adjacent features are called out (e.g., metrics endpoint removal)
- ❌ No schema URL on telemetry
- ❌ Non-functional features (`opentelemetry` driver) are not clearly flagged as broken in docs
- ❌ No formal stability/experimental distinction for telemetry signals
- ❌ Undocumented breaking behaviors (crash on `service_name`, bootstrap restart requirement)
- ❌ No migration guidance for users moving from deprecated features

#### Rationale

Level 1 (OpenTelemetry-Aligned) is the correct assessment. The project is aware that telemetry changes have impact and includes some telemetry-related entries in changelogs. However, handling is informal and inconsistent — the `opentelemetry` driver's non-functional state is not communicated, several breaking behaviors are undocumented, and there is no formal telemetry stability contract. Users cannot safely build long-lived dashboards or alerts on Emissary's telemetry without risking silent breakage.

---

## Key findings

### Strengths

- **Rich Prometheus metrics coverage**: 443 unique metric names covering Envoy cluster health, HTTP request/response rates, circuit breaker states, and Ambassador control-plane timing. This is a comprehensive operational dataset for monitoring Emissary's behavior.
- **Functional trace export path**: Despite the `opentelemetry` driver being non-functional, the `zipkin` driver reliably exports traces to an OTel Collector, enabling trace-based debugging of request routing and upstream behavior.
- **Internal trace coherence**: Within Emissary's own spans, the parent-child relationship between the ingress span and egress span is correct, making it possible to see the full request lifecycle within Emissary.
- **OTel Collector compatibility**: Emissary's Zipkin and Prometheus outputs are compatible with standard OTel Collector receivers, allowing integration into existing OTel pipelines without custom adapters.

### Areas for improvement

1. **Fix or remove the `opentelemetry` TracingService driver**: The OTLP gRPC driver is listed in the CRD schema and changelog but does not function in 3.12.2. It should either be fixed to actually configure Envoy's tracing provider, or clearly marked as unsupported/removed. This is the single highest-impact improvement for integration surface maturity.

2. **Align span attributes with current OTel HTTP semantic conventions**: Replace deprecated attributes (`http.method` → `http.request.method`, `http.status_code` → `http.response.status_code`, `http.url` → `url.full`, `peer.address` → `network.peer.address`, `http.protocol` → `network.protocol.name`/`network.protocol.version`) and fix span kinds (ingress span should be `SPAN_KIND_SERVER = 2`, not `CLIENT = 3`).

3. **Consistent `service.name` across signals**: The `service.name` should be the same value in both traces (`ambassador-emissary`) and metrics. Currently these differ, making cross-signal correlation by service identity impossible. Consider supporting W3C Trace Context propagation (or at minimum B3+W3C dual propagation) to enable end-to-end distributed tracing with OTel-native downstream services.

4. **Add `service.version` to resource attributes**: The Emissary version (3.12.2) should be emitted as `service.version` in trace resource attributes, enabling version-based analysis of telemetry changes.

5. **Support OTLP log export or structured access logs**: The current stdout text format for access logs cannot be correlated with traces or metrics. Either implement OTLP log export or use the `LogService` CRD to forward structured access logs (with trace context headers) to an OTel Collector.

### Notable observations

1. **The `opentelemetry` TracingService driver is non-functional despite being in the CRD schema**: This is the most significant finding. Users who follow the documentation and configure `driver: opentelemetry` will get no traces. The driver logs "The OpenTelemetry tracing driver is work-in-progress" and does not configure Envoy's tracing bootstrap. This has been the case since the driver was introduced in Envoy 1.24 and has not been resolved in Emissary 3.12.2.

2. **`config.service_name` in ZipkinConfig causes an Envoy crash loop**: Emissary 3.12.2 generates a `service_name` field in the Zipkin config that Envoy 1.31.4 rejects as invalid. Users who include `config.service_name` in their `TracingService` will find their Emissary pods crash-looping. This is a bug in Emissary's config generation that is not documented.

3. **B3 propagation creates a hard boundary in distributed traces**: Because Envoy's Zipkin driver injects B3 headers while modern OTel-instrumented services use W3C Trace Context, every request through Emissary creates two completely separate, uncorrelated trace trees. This is confirmed by the telemetry data: 0 shared trace IDs between Emissary and the backend service across 6 requests. This is a fundamental limitation of using the Zipkin driver.

4. **`createDefaultListeners: true` is required but not prominently documented**: Without this Helm value, Envoy never opens port 8080 and all traffic is refused. This is a significant usability gap for new users.

5. **Tracing bootstrap changes require a full pod restart**: The `TracingService` configuration is written to `bootstrap-ads.json`, which Envoy reads only at startup. Changes to `TracingService` require `kubectl rollout restart` — they are not applied via xDS dynamic configuration. This is not documented in the TracingService reference.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file exporter in a local kind cluster (`otel-eval-emissary`)
- The k8sattributes processor was used to enrich telemetry with Kubernetes metadata; only attributes present before this enrichment are counted as "native"
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv v1.23+)
- The Zipkin receiver was required in the OTel Collector traces pipeline; the OTLP gRPC receiver was not usable for Emissary traces due to the non-functional `opentelemetry` driver
- Prometheus scrape was configured targeting `emissary-emissary-ingress-admin.emissary.svc.cluster.local:8877`
- 6 HTTP requests were sent through Emissary to generate trace data
- Source code and changelog reviewed at https://github.com/emissary-ingress/emissary
- Emissary version 3.12.2 / Helm chart 8.12.2 / Envoy 1.31.4-dev
