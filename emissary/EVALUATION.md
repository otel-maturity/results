# OpenTelemetry Support Maturity Evaluation: Emissary-ingress

## Project overview

- **Project**: Emissary-ingress — a CNCF Incubating Kubernetes-native API Gateway and Ingress controller built on the Envoy proxy, formerly known as Ambassador API Gateway
- **Version evaluated**: 3.12.2 (Helm chart 8.12.2)
- **Evaluation date**: 2026-05-09
- **Cluster**: otel-eval-emissary (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 1 | OTLP tracing driver exists but is WIP; metrics only via Prometheus scrape; no OTLP log export |
| Semantic Conventions | 1 | Envoy Prometheus metrics use proprietary `envoy_*` naming; no OTel semconv alignment; traces are backend-only with deprecated HTTP attributes |
| Resource Attributes & Configuration | 1 | No native `service.name` from Emissary itself; identity is pipeline-derived; no `OTEL_*` env var support |
| Trace Modeling & Context Propagation | 0 | Emissary emits no spans in v3.12.2 (OTel driver WIP); no trace context propagation to upstream services |
| Multi-Signal Observability | 0 | Only Prometheus metrics are effectively usable; traces absent; logs absent; no cross-signal correlation |
| Audience & Signal Quality | 1 | 407 Prometheus metric types provide operational coverage; naming is Envoy-internal (`envoy_cluster_*`); no user-oriented signal design |
| Stability & Change Management | 1 | OTel driver WIP status is disclosed in logs but not prominently in docs; telemetry changes not treated as a contract; no schema URL on Prometheus metrics |

---

## Telemetry overview

### Signals observed

- **Traces**: Not flowing from Emissary — the `opentelemetry` TracingService driver is explicitly WIP in v3.12.2 and emits no spans. Backend-only traces (from `otel-eval-backend` Node.js service) are present via OTLP/HTTP push.
- **Metrics**: Flowing — 407 unique Prometheus metric names scraped from Emissary's admin service at port 8877 (`/metrics`) by the OTel Collector's Prometheus receiver. Converted to OTLP metrics by the collector pipeline.
- **Logs**: Not flowing — Emissary does not emit OTLP logs. The `LogService` CRD uses Envoy's gRPC Access Log Service (ALS) protocol, which is incompatible with the standard OTLP receiver. Pod stdout logs were not collected (logsCollection disabled in cluster).

### Resource attributes (native, before collector enrichment)

Emissary itself sets **no OTLP resource attributes**. The Prometheus scrape target produces these resource attributes (set by the OTel Collector's Prometheus receiver, not by Emissary):

```
server.address: emissary-emissary-ingress-admin.emissary.svc.cluster.local
server.port: 8877
service.instance.id: emissary-emissary-ingress-admin.emissary.svc.cluster.local:8877
service.name: emissary-ingress   ← set by the collector from the scrape job_name, NOT by Emissary
url.scheme: http
```

Emissary does not use the OTel SDK and does not set `service.name`, `service.version`, `telemetry.sdk.*`, or any other resource attributes natively.

### Resource attributes (after collector enrichment)

The k8sattributes processor added Kubernetes metadata to the Emissary pod's metrics (k8s_cluster receiver path):

```
k8s.pod.uid, k8s.pod.name, k8s.pod.start_time
k8s.namespace.name: emissary
k8s.node.name, k8s.replicaset.name, k8s.deployment.name: emissary-emissary-ingress
k8s.pod.label.app.kubernetes.io/name: emissary-ingress
k8s.pod.label.app.kubernetes.io/instance: emissary
k8s.pod.label.helm.sh/chart: emissary-ingress-8.12.2
k8s.pod.annotation.prometheus.io/scrape: "true"
k8s.pod.annotation.prometheus.io/port: "8877"
container.id, container.image.name: docker.io/datawire/emissary, container.image.tag: 3.12.2
```

The Prometheus-scraped metrics resource did **not** receive k8sattributes enrichment (the scrape target is a Service DNS name, not a pod IP, so association failed).

---

## Dimension evaluations

### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

- **Traces**: The `TracingService` CRD supports `driver: opentelemetry` (alongside `zipkin` and `datadog`). This is the mechanism for OTLP trace export. However, in v3.12.2 the driver is explicitly marked WIP in Emissary's own logs: *"The OpenTelemetry tracing driver is work-in-progress. Functionality is incomplete and it is not intended for production use."* The bootstrap Envoy config is generated correctly (gRPC cluster pointing to port 4317), but the HTTP Connection Manager tracing config in each listener remains empty `{}`, so no spans are emitted and no trace context is propagated. The tracing cluster shows `cx_total: 0` — no connections were ever made.
- **Metrics**: Exposed exclusively via Prometheus `/metrics` on the admin service port 8877. No OTLP metrics emission. Users must configure a Prometheus receiver in their OTel Collector (or a Prometheus server) to consume these metrics. This is a Prometheus-first, not OTLP-first, integration model.
- **Logs**: The `LogService` CRD offers Envoy gRPC Access Log Service (ALS) integration, but this uses Envoy's proprietary ALS gRPC protocol — not OTLP. The standard OTel Collector OTLP receiver does not implement this protocol. There is no OTLP log export path from Emissary.
- **Configuration**: There is no `OTEL_EXPORTER_OTLP_ENDPOINT` or `OTEL_*` environment variable support. All telemetry configuration is done through Emissary-specific CRDs (`TracingService`, `LogService`) and project-specific Helm values (`adminService.create`, `podAnnotations`).
- **v4 improvement note**: The v4.0.1 release (post-evaluation) includes a "Distributed Tracing with OpenTelemetry and Dash0" how-to guide in the official docs, suggesting the OTel tracing driver has been stabilized in v4. The evaluated version (v3.12.2) predates this.

#### Checklist assessment

- ❌ Is OTLP the default or clearly recommended export path? — No. Prometheus scraping is the only working path for metrics; the OTel tracing driver is WIP.
- ✅ Is OTLP supported, even if alongside legacy exporters? — Yes, `driver: opentelemetry` exists for traces; Prometheus scraping can be consumed by the OTel Collector.
- ❌ Are standard `OTEL_*` environment variables respected? — No. Emissary uses its own CRD-based configuration model.
- ❌ Can users connect the project to an existing OTel Collector without adapters or glue code? — Partially. Metrics require a Prometheus receiver; traces require the WIP OTel driver; logs have no path.
- ❌ Is OpenTelemetry the "happy path"? — No. Prometheus scraping is the only reliably working telemetry integration.

#### Rationale

Emissary has explicitly added the `opentelemetry` driver to its `TracingService` CRD, which demonstrates intentional movement toward OTLP. This places it at Level 1 (OpenTelemetry-Aligned). However, the driver is WIP and non-functional in v3.12.2, and metrics remain Prometheus-only with no OTLP emission. The integration surface is still primarily Prometheus-centric, with OTLP as an aspirational but not yet functional path for traces, and absent for metrics and logs. This is clearly Level 1 — OTLP is present but not central.

---

### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Trace attributes

Traces in the evaluation are exclusively from `otel-eval-backend` (Node.js), not from Emissary itself. Emissary emits no spans. The backend spans use **deprecated** OpenTelemetry HTTP semantic conventions (from `@opentelemetry/instrumentation-http v0.48.0`):

```
http.method          ← deprecated (current: http.request.method)
http.status_code     ← deprecated (current: http.response.status_code)
http.url             ← deprecated (current: url.full)
http.target          ← deprecated (current: url.path)
http.host            ← deprecated
http.scheme          ← deprecated (current: url.scheme)
http.flavor          ← deprecated (current: network.protocol.version)
net.host.name        ← deprecated (current: server.address)
net.host.port        ← deprecated (current: server.port)
net.peer.ip          ← deprecated (current: network.peer.address)
net.peer.port        ← deprecated (current: network.peer.port)
net.transport        ← deprecated
http.client_ip       ← deprecated (current: client.address)
```

Root spans (`GET /`) have `kind=2` (SERVER) — correct. Express middleware spans have `kind=1` (INTERNAL) — correct. No schema URL is set on traces.

##### Metric names and attributes

Emissary/Envoy exposes 407 unique Prometheus metric names. These follow Envoy's internal naming conventions:

- `envoy_cluster_*` — cluster-level stats (upstream connections, requests, circuit breakers)
- `envoy_http_*` — HTTP connection manager stats (downstream requests, response codes)
- `envoy_listener_*` — listener-level stats
- `envoy_server_*` — server health and config stats
- `envoy_tls_inspector_*` — TLS inspection stats
- `ambassador_*` — Emissary-specific process metrics (`ambassador_aconf_time_seconds`, `ambassador_process_cpu_seconds_total`, etc.)

Data point labels use Envoy-internal names:
```
envoy_cluster_name
envoy_http_conn_manager_prefix
envoy_listener_address
envoy_rds_route_config
envoy_response_code
envoy_response_code_class
envoy_worker_id
ambassador_id
cluster_id
```

These are **not** OTel semantic convention names. There is no use of `http.request.method`, `http.response.status_code`, `server.address`, `network.protocol.version`, or other current OTel HTTP/network conventions on metrics. The metric naming convention (`envoy_*`) is Envoy's proprietary Prometheus exposition format.

##### Log attributes

No logs are flowing. Cannot assess.

#### Checklist assessment

- ❌ Are current, stable OTel semantic conventions used? — No. Envoy metric names are proprietary. Backend trace attributes use deprecated HTTP conventions.
- ❌ Are deprecated attributes removed? — No. Backend spans use `http.method`, `http.status_code`, `http.url`, `http.target`, etc.
- ❌ Are attributes aligned across traces, metrics, and logs? — Cannot assess fully (no logs; Emissary emits no traces). Metric labels do not align with trace attributes.
- ❌ Can telemetry be interpreted using generic OTel knowledge without project-specific mapping? — No. The `envoy_*` metric naming requires Envoy-specific knowledge.
- ✅ Some OTel alignment exists — the `TracingService` CRD references `TRACE_CONTEXT` (W3C) and `B3` propagation modes; the `opentelemetry` driver is named correctly.

#### Rationale

The Prometheus metrics follow Envoy's internal naming conventions, not OTel semantic conventions. The backend traces use deprecated OTel HTTP attributes. Emissary itself emits no spans, so there is no opportunity to evaluate Emissary's own trace semantic convention alignment. The partial adoption of OTel concepts (W3C propagation modes, `opentelemetry` driver name) is consistent with Level 1 (partial alignment), but there is no systematic application of current OTel semantic conventions to the actual telemetry data.

---

### 3. Resource Attributes & Configuration

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Native resource attributes

Emissary does not use the OpenTelemetry SDK and sets **no resource attributes** natively in its OTLP output. The project does not emit OTLP at all for metrics or logs; for traces, the WIP driver is non-functional.

The `service.name: emissary-ingress` visible in the metrics data is set by the OTel Collector's Prometheus receiver from the scrape job's `job_name` configuration — it is **pipeline-derived**, not emitted by Emissary.

##### OTEL_* environment variable support

No `OTEL_*` environment variable support. Emissary's telemetry is configured exclusively through:
- `TracingService` CRD — tracing endpoint, driver, service name, propagation modes
- `LogService` CRD — access log forwarding
- Admin service Helm values — Prometheus `/metrics` exposure
- Pod annotations — Prometheus scrape discovery

There is no documented support for `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, or any other standard OTel environment variable.

##### Identity consistency across signals

No consistent identity across signals:
- **Metrics**: `service.name: emissary-ingress` is set by the collector from `job_name`; no `service.version`, no `service.instance.id` from Emissary
- **Traces**: No spans from Emissary; backend traces have `service.name: otel-eval-backend` (unrelated)
- **Logs**: Not flowing

The `TracingService` CRD has a `config.service_name` field (`emissary-ingress`), which would set the service name in Envoy's bootstrap tracing config — but since the OTel driver is WIP and emits no spans, this is never observed in practice.

#### Checklist assessment

- ❌ Is `service.name` set as a resource attribute by the project? — No (pipeline-derived only)
- ❌ Is `service.version` present? — No
- ❌ Is `service.instance.id` present? — No (pipeline-derived `service.instance.id` from scrape target URL)
- ❌ Are `OTEL_*` environment variables respected? — No
- ❌ Is identity stable and consistent across signals? — No (different sources, no unified identity)
- ✅ Some identity exists via pipeline enrichment — k8s.deployment.name, k8s.namespace.name, etc. are added by k8sattributes

#### Rationale

Emissary has no OTel SDK integration and therefore no native resource attribute emission. The `TracingService` CRD's `service_name` field is a step toward identity awareness, but it is not surfaced in working telemetry. Configuration is entirely CRD-based with no `OTEL_*` support. This is Level 1: some identity concepts exist (the `service_name` in the CRD), but behavior is inconsistent across signals and configuration is project-specific rather than OTel-native.

---

### 4. Trace Modeling & Context Propagation

**Level: 0 — Instrumented**

#### Evidence

##### Span structure

Emissary emits **no spans** in v3.12.2. The `opentelemetry` TracingService driver is WIP:
- Envoy bootstrap config correctly references the OTel tracing cluster (`envoy.opentelemetry`, gRPC to port 4317)
- HTTP Connection Manager (HCM) tracing config in each listener is empty `{}` — Envoy does not enable per-request tracing
- The tracing cluster shows `cx_total: 0` — no connections to the collector were ever made
- No `tracing.*` stats appear in Envoy's stats endpoint

The only traces in the evaluation are from `otel-eval-backend` (Node.js):
- Root spans: `GET /` with `kind=2` (SERVER) — correct kind, but no parent from Emissary
- Child spans: `middleware - query`, `middleware - expressInit`, `middleware - jsonParser`, `middleware - <anonymous>`, `request handler - /` — all with `kind=1` (INTERNAL), correctly parented to the root HTTP span

##### Context propagation

Emissary does **not** inject or propagate trace context headers to upstream services. This was confirmed by:
1. Sending requests with explicit `traceparent` headers (W3C Trace Context): `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
2. Observing that the backend's root `GET /` spans have `parentSpanId: null` — Emissary did not forward the `traceparent` header

The `TracingService` CRD was configured with `propagation_modes: [TRACE_CONTEXT, B3]`, but since the HCM tracing hook is not active, no propagation occurs.

##### Trace coherence

No coherent distributed traces are possible. Each backend request appears as an isolated trace with no connection to the ingress layer.

#### Checklist assessment

- ❌ Does Emissary emit spans? — No (WIP driver)
- ❌ Is context propagation working? — No (HCM tracing not enabled)
- ❌ Are parent-child relationships correct for request flows? — Cannot assess (no Emissary spans)
- ❌ Is W3C Trace Context propagated to upstream services? — No
- ❌ Do traces represent meaningful logical operations? — No Emissary spans exist

#### Rationale

This is clearly Level 0. The tracing driver exists in the CRD schema but is non-functional in v3.12.2. No spans are emitted, no context is propagated, and traces cannot be followed through the gateway layer. The project is aware of this limitation (explicit WIP warning in logs), which shows intent to progress, but the current state is Level 0 for this dimension.

**Important context**: GitHub issue [#5573](https://github.com/emissary-ingress/emissary/issues/5573) ("Opentelemetry driver missing spans", opened 2024-02-15, last updated 2026-04-14) confirms this is a known open issue. The v4 documentation includes a working "Distributed Tracing with OpenTelemetry and Dash0" how-to, suggesting the driver was fixed in v4.

---

### 5. Multi-Signal Observability

**Level: 0 — Instrumented**

#### Evidence

##### Signal availability

| Signal | Status | Method | Notes |
|--------|--------|--------|-------|
| Traces | Not flowing from Emissary | — | OTel driver WIP in v3.12.2 |
| Metrics | Flowing | Prometheus scrape | 407 metric types from Envoy |
| Logs | Not flowing | — | ALS protocol mismatch with OTLP |

Only one signal (metrics) is effectively usable. Traces and logs are absent.

##### Cross-signal correlation

No cross-signal correlation is possible:
- Metrics have no trace IDs or span IDs
- No logs are present
- Backend traces have no connection to Emissary metrics

The `x-request-id` header is mentioned in the TracingService documentation as a mechanism for request correlation in logs, but this is not OTLP-based and was not observed in the evaluation.

##### Collection model

- **Metrics**: Prometheus scrape (pull model) — requires a Prometheus receiver in the OTel Collector
- **Traces**: OTLP gRPC push (WIP, non-functional in v3.12.2)
- **Logs**: Envoy ALS gRPC (incompatible with OTLP receiver)

The collection models differ per signal, requiring different receiver configurations. There is no unified OTLP push model.

#### Checklist assessment

- ❌ Are traces, metrics, and logs all first-class signals? — No. Only metrics work; traces are WIP; logs have no OTLP path.
- ❌ Do signals include correlation context? — No. Metrics have no trace IDs.
- ❌ Can users pivot naturally between signals? — No. Only one signal is usable.
- ❌ Are signals designed to work together? — No evidence of intentional multi-signal design.

#### Rationale

Level 0. Only one signal (Prometheus metrics) is effectively usable. The model requires all three signals to be present and first-class for Level 1. Emissary's tracing is WIP and logs have no OTLP path. This is a single-signal observability model in practice, with aspirational support for additional signals.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Emissary emits no spans. The backend spans use framework-generated names:
- `GET /` — logical HTTP operation name (good)
- `middleware - query`, `middleware - expressInit`, `middleware - <anonymous>` — Express.js internal middleware names (implementation-centric, not user-relevant)
- `request handler - /` — reasonable

##### Signal-to-noise ratio

The 407 Prometheus metric types are comprehensive but include many internal Envoy counters that are unlikely to be useful to most operators:
- `envoy_cluster_http2_inbound_empty_frames_flood` — very internal
- `envoy_cluster_http2_inbound_priority_frames_flood` — very internal
- `envoy_cluster_http2_inbound_window_update_frames_flood` — very internal
- `envoy_cluster_circuit_breakers_default_cx_pool_open` — useful
- `envoy_http_downstream_rq_total` — useful
- `envoy_cluster_upstream_rq_total` — useful

The `ambassador_*` metrics provide useful process-level signals:
- `ambassador_aconf_time_seconds` — config processing time
- `ambassador_process_cpu_seconds_total` — CPU usage
- `ambassador_process_resident_memory_bytes` — memory usage

##### Default usability

The Prometheus metrics are usable for basic monitoring without customization — standard Envoy dashboards exist (e.g., in Grafana's dashboard library). However, the metric names require Envoy-specific knowledge to interpret. Without tracing or structured logs, deep investigation of individual requests is not possible.

The absence of tracing significantly limits the operational value of the telemetry. Operators can see aggregate metrics (total requests, error rates, latency histograms) but cannot drill into specific failing requests.

#### Checklist assessment

- ✅ Are some signals usable without extensive customization? — Yes, Prometheus metrics support basic monitoring
- ❌ Do span names describe logical, user-relevant operations? — Cannot assess (no Emissary spans); backend middleware span names are implementation-centric
- ❌ Are metrics focused on operational signals rather than raw counters? — Mixed; many useful operational metrics but also many internal Envoy counters
- ❌ Can operators move from symptoms to causes efficiently? — No. Without traces, root-cause analysis is limited to aggregate metrics.
- ✅ Obvious noise is somewhat controlled — the 407 metrics are comprehensive but not gratuitously verbose

#### Rationale

Level 1. The Prometheus metrics provide real operational value for basic monitoring and alerting, which is a meaningful improvement over nothing. However, the naming is Envoy-internal (not user-oriented), the absence of traces prevents deep investigation, and there is no evidence of intentional signal quality design. The telemetry serves basic monitoring needs but requires significant domain knowledge to interpret.

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

The TracingService documentation (v3.9) describes the CRD fields and supported drivers (`zipkin`, `datadog`, `opentelemetry`) but does not explicitly call out the WIP status of the `opentelemetry` driver in the documentation itself. The WIP warning appears only in Emissary's runtime logs:

```
"The OpenTelemetry tracing driver is work-in-progress. Functionality is incomplete and it is not intended for production use."
```

This is a meaningful disclosure, but it is not surfaced in the documentation or the CRD schema where users configure the driver.

The v3.9 CHANGELOG contains this entry when the OTel driver was introduced:
> "Feature: In Envoy 1.24, experimental support for a native OpenTelemetry tracing driver was introduced... Emissary-ingress now supports setting the TracingService.spec.driver=opentelemetry to export spans in otlp format."

This was presented as a feature, not as experimental/WIP. The WIP warning was added later.

##### Change communication

The CHANGELOG documents major telemetry changes (e.g., removal of LightStep driver, introduction of OTel driver, deprecation of `HTTP_JSON_V1` for Zipkin). However:
- The WIP status of the OTel driver is not reflected in release notes
- Telemetry changes are not treated with the same rigor as API changes
- No migration guides for telemetry changes were found
- No distinction between stable and experimental telemetry in the CRD schema or docs

##### Schema URL presence

- **Traces**: No schema URL (no Emissary spans; backend spans also have no `schemaUrl`)
- **Metrics (k8s_cluster receiver)**: `schemaUrl: https://opentelemetry.io/schemas/1.18.0` — set by the OTel Collector, not by Emissary
- **Metrics (Prometheus receiver)**: No `schemaUrl`

##### Stability guarantees

No explicit stability guarantees for telemetry. The `TracingService` CRD API itself has an inconsistency between `v3alpha1` (`propagation_modes` field) and `v1` (`v3PropagationModes` field) — the webhook conversion handles this transparently but it indicates the API is still evolving.

#### Checklist assessment

- ✅ Some telemetry changes are mentioned in release notes — Yes, major changes (LightStep removal, OTel driver introduction) are documented
- ❌ Is there a distinction between stable and experimental telemetry? — Not in the CRD schema or docs; WIP warning only in runtime logs
- ❌ Are breaking changes called out explicitly? — Partially; the LightStep removal was documented, but the WIP status of OTel driver is not
- ❌ Is migration guidance provided? — Not for telemetry-specific changes
- ❌ Are telemetry changes reviewed with downstream impact in mind? — No evidence of this practice
- ❌ Is schema URL present on Emissary-emitted telemetry? — No (Emissary emits no OTLP telemetry natively)

#### Rationale

Level 1. There is some awareness of stability (major changes like LightStep removal are documented), but handling is informal and inconsistent. The WIP status of the OTel driver is disclosed at runtime but not in documentation or CRD schema. There is no clear policy for telemetry evolution, no distinction between stable and experimental telemetry in the public-facing API, and no migration guides for telemetry changes. This matches Level 1 characteristics: "some telemetry changes are mentioned in release notes; breaking changes discovered reactively; no clear distinction between stable and experimental telemetry."

---

## Key findings

### Strengths

1. **Rich Prometheus metrics coverage**: 407 unique Envoy metric types provide comprehensive operational visibility into cluster-level, HTTP, listener, server, and TLS behavior. These are well-established Envoy metrics that many operators already know and for which community dashboards (Grafana) exist.

2. **OTel tracing architecture is in place**: The `TracingService` CRD with `driver: opentelemetry`, W3C Trace Context propagation mode, 128-bit trace IDs, and gRPC export to an OTel Collector endpoint is correctly designed. The Envoy bootstrap config is generated correctly. The infrastructure is ready; only the HCM hook is missing. This is a fixable gap, not a fundamental design problem.

3. **Kubernetes-native CRD configuration model**: Emissary's CRD-based configuration (`TracingService`, `LogService`, `Mapping`) integrates naturally with Kubernetes workflows and GitOps practices. The `TracingService` CRD provides a clean, declarative way to configure tracing without requiring application-level changes.

4. **v4 shows significant improvement**: The v4.0.1 release (post-evaluation) includes a documented, working "Distributed Tracing with OpenTelemetry and Dash0" integration, indicating the OTel tracing driver was fixed. This suggests the project is actively progressing toward OTel maturity.

### Areas for improvement

1. **Fix the OTel tracing driver (critical)**: The HCM tracing configuration must be populated for the `opentelemetry` driver to emit spans and propagate trace context. This is the single most impactful improvement. The fix in v4 should be backported documentation-wise so users on v3.x know the limitation clearly.

2. **Add OTLP metrics export**: Emissary should support emitting metrics via OTLP push (in addition to or instead of Prometheus scraping). This would enable a unified OTLP pipeline for all signals and eliminate the need for a separate Prometheus receiver configuration. The `service.name`, `service.version`, and `service.instance.id` resource attributes should be set natively.

3. **Implement OTLP log export**: The `LogService` CRD's Envoy ALS integration should be complemented or replaced with an OTLP log export path. Emissary is well-positioned to emit structured access logs as OTLP log records with trace context (once tracing is working), enabling true multi-signal correlation.

4. **Align metric naming with OTel semantic conventions**: While the `envoy_*` naming convention is deeply embedded in Envoy's Prometheus exposition, Emissary could provide an OTLP metrics path that uses OTel HTTP semantic convention names (e.g., `http.server.request.duration`, `http.server.active_requests`) alongside the raw Envoy stats. This would enable off-the-shelf OTel dashboards and alerts.

5. **Document WIP status prominently**: The `opentelemetry` driver's WIP status should be clearly documented in the `TracingService` reference docs and CRD schema (e.g., via a `description` annotation), not just in runtime logs. Users configuring `driver: opentelemetry` in v3.x should see an immediate warning.

### Notable observations

1. **WIP driver is the key gap**: The evaluated version (v3.12.2) has all the infrastructure for OTel tracing except the final HCM hook. This is a surprisingly small gap given the overall design. The v4 fix confirms this was a targeted, fixable issue.

2. **Prometheus receiver auto-sets `service.name`**: The OTel Collector's Prometheus receiver sets `service.name: emissary-ingress` from the scrape job's `job_name`. This creates an illusion of proper resource attribution, but it is pipeline-derived — Emissary itself does not set this attribute.

3. **k8sattributes enrichment did not apply to Prometheus metrics**: Because the Prometheus receiver scrapes a Service DNS name (not a pod IP), the k8sattributes processor could not associate the scrape target with a pod and did not enrich the Prometheus metrics resource with Kubernetes metadata. The k8s_cluster receiver metrics (which use pod UIDs for association) did receive enrichment.

4. **CRD API version inconsistency**: The `propagation_modes` field name differs between `v3alpha1` (`propagation_modes`) and `v1` (`v3PropagationModes`). The webhook conversion handles this transparently, but it indicates the CRD API is still evolving and not fully stabilized.

5. **Backend traces use deprecated semconv**: The `otel-eval-backend` Node.js service uses `@opentelemetry/instrumentation-http v0.48.0`, which emits deprecated HTTP attributes (`http.method`, `http.status_code`, `http.url`, etc.) instead of the current conventions (`http.request.method`, `http.response.status_code`, `url.full`). This is a backend issue, not an Emissary issue, but it affects the overall trace quality in the evaluation.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector (contrib, v0.150.1) with file exporters in a kind cluster (`otel-eval-emissary`)
- The k8sattributes processor was used to distinguish native vs enriched resource attributes
- Emissary v3.12.2 (Helm chart 8.12.2) was installed with `createDefaultListeners: true` and `adminService.create: true`
- A `TracingService` CRD with `driver: opentelemetry` was applied; the WIP status was confirmed by examining Emissary logs and Envoy's bootstrap/HCM config
- Prometheus metrics were scraped from the admin service at port 8877 via the OTel Collector's Prometheus receiver
- ~75 test requests were sent through Emissary to `otel-eval-backend`, including requests with explicit `traceparent` headers to test context propagation
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv v1.23+)
- Documentation was reviewed at emissary-ingress.dev for v3.9, v3.10, and v4.0; the v4 docs show significant OTel improvements post-evaluation
- GitHub issues were searched to confirm the OTel driver WIP status as a known open issue (#5573)
