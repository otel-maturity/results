# OpenTelemetry Support Maturity Evaluation: Traefik

## Project overview

- **Project**: Traefik — cloud-native reverse proxy and Kubernetes Ingress controller
- **Version evaluated**: v3.7.0 (Helm chart 40.0.0)
- **Evaluation date**: 2026-05-09
- **Cluster**: otel-eval-traefik
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP is the primary path for all three signals; Prometheus remains a first-class peer, not clearly secondary |
| Semantic Conventions | 2 | Traces use current OTel HTTP conventions; `traefik_*` metrics use legacy-style attribute names (`method`, `code`, `protocol`); access logs use proprietary field names in the body |
| Resource Attributes & Configuration | 2 | Rich native resource attributes; `service.name` / `service.version` consistent across all signals; `OTEL_RESOURCE_ATTRIBUTES` documented; no `service.instance.id` emitted |
| Trace Modeling & Context Propagation | 2 | Coherent SERVER→CLIENT span hierarchy; W3C Trace Context propagation confirmed end-to-end; schemaUrl set to `1.40.0` |
| Multi-Signal Observability | 2 | All three signals flow via OTLP; access logs carry `traceId`/`spanId` enabling log–trace correlation; metrics lack trace context linkage |
| Audience & Signal Quality | 1 | Trace span names are generic (`GET`, `ReverseProxy`); access log body uses `"level":"panic"` for normal requests; `process.command_args` exposes full CLI flags as a resource attribute |
| Stability & Change Management | 1 | Telemetry changes appear in changelogs with `[otel]` tags; no formal telemetry stability policy; OTLP logs remain behind an experimental flag |

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC push
- **Metrics**: Flowing — OTLP gRPC push (native) + Prometheus scrape (via collector Prometheus receiver)
- **Logs**: Flowing — OTLP gRPC push (experimental feature, `experimental.otlpLogs: true` required)

### Resource attributes (native, before collector enrichment)

Traefik emits the following resource attributes natively via the OTel Go SDK:

| Attribute | Value (example) |
|-----------|----------------|
| `service.name` | `traefik` |
| `service.version` | `3.7.0` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-5785795bf7-9pt58` (pod name from Go runtime) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |
| `process.command_args` | Full CLI args array (all config flags — very verbose) |

Notable absence: `service.instance.id` is not emitted natively.

### Resource attributes (after collector enrichment)

The k8sattributes processor adds:

- `k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.deployment.name`, `k8s.replicaset.name`, `k8s.node.name`, `k8s.container.name`
- Pod labels (`k8s.pod.label.*`) and annotations (`k8s.pod.annotation.*`)

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

Traefik v3 uses OpenTelemetry as the sole tracing backend (Jaeger, Zipkin, Datadog, and other legacy tracers were removed in the v2→v3 migration). For metrics, both OTLP push and Prometheus scrape are supported simultaneously and documented as co-equal options. For logs, OTLP is the only structured export path, but it requires an experimental feature flag (`experimental.otlpLogs: true`).

Configuration is entirely through Traefik's own CLI flags / YAML config (e.g., `tracing.otlp.grpc.endpoint`, `metrics.otlp.grpc.enabled`). Standard `OTEL_*` environment variables are **not** used for endpoint/protocol configuration; only `OTEL_RESOURCE_ATTRIBUTES` is documented as respected. This means users cannot configure the OTLP exporter endpoint via the standard `OTEL_EXPORTER_OTLP_ENDPOINT` variable.

The docs describe OpenTelemetry as the tracing system ("Traefik uses OpenTelemetry, an open standard designed for distributed tracing") and list OpenTelemetry as the first metrics backend option. However, Prometheus is listed as a fully equal metrics export path with no deprecation signal.

#### Checklist assessment

- ✅ OTLP is supported for all three signals
- ✅ Users can connect to an existing OTel Collector without adapters for traces and metrics
- ✅ For tracing, OpenTelemetry is the only option (legacy tracers removed in v3)
- ⚠️ For metrics, Prometheus and OTLP are co-equal — neither is clearly "the default"
- ⚠️ For logs, OTLP requires an experimental feature flag
- ❌ `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_EXPORTER_OTLP_PROTOCOL` are not respected; only `OTEL_RESOURCE_ATTRIBUTES` is
- ✅ Telemetry configuration is consistent in structure across signals (all use `*.otlp.grpc.endpoint` pattern)

#### Rationale

Traefik comfortably reaches Level 2 for tracing (OTel is the only option) and is close to Level 2 for metrics and logs. The co-equal status of Prometheus metrics and the experimental gate on OTLP logs prevent a clean Level 2 judgment overall, but the primary integration surface is clearly OTLP-based. The lack of standard `OTEL_EXPORTER_OTLP_ENDPOINT` support is a notable gap but does not drop the level, as Traefik's own configuration is consistent and well-documented. Level 2 is the appropriate rating.

---

### 2. Semantic Conventions

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Trace attributes

Traefik's SERVER spans (`GET`) use current OTel HTTP semantic conventions:
- `http.request.method` (not deprecated `http.method`) ✅
- `http.response.status_code` (not deprecated `http.status_code`) ✅
- `url.path`, `url.query`, `url.scheme` (not deprecated `http.target`, `http.url`) ✅
- `network.protocol.version` ✅
- `user_agent.original` ✅
- `server.address`, `client.address`, `client.port` ✅
- `network.peer.address`, `network.peer.port` ✅

Traefik's CLIENT spans (`ReverseProxy`) also use current conventions:
- `http.request.method`, `http.response.status_code`, `url.full`, `url.scheme` ✅
- `network.peer.address`, `network.peer.port`, `server.address`, `server.port` ✅

The instrumentation scope for traces is `github.com/traefik/traefik` with **no version** set (the `version` field is absent in the scope). The schema URL `https://opentelemetry.io/schemas/1.40.0` is correctly set on the resource spans.

##### Metric names and attributes

Two categories of metrics are present in the OTLP stream from Traefik:

**OTel semantic convention metrics** (correct):
- `http.server.request.duration` (histogram) — attributes: `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `url.scheme` ✅
- `http.client.request.duration` (histogram) — same attributes plus `server.port` ✅

**Traefik-proprietary metrics** (non-OTel naming and attribute conventions):
- `traefik_entrypoint_requests_total` — attributes: `code`, `entrypoint`, `method`, `protocol` ⚠️
- `traefik_router_requests_total` — attributes: `code`, `method`, `protocol`, `router`, `service` ⚠️
- `traefik_service_requests_total` — attributes: `code`, `method`, `protocol`, `service` ⚠️
- `traefik_entrypoint_request_duration_seconds` — attributes: `code`, `entrypoint`, `method`, `protocol` ⚠️

The `traefik_*` metrics use short, non-OTel attribute names (`method` instead of `http.request.method`, `code` instead of `http.response.status_code`, `protocol` instead of `network.protocol.name`). These are Prometheus-style label names, not OTel semantic convention attribute names.

The metrics scope version is correctly set to `3.7.0` for `github.com/traefik/traefik`.

##### Log attributes

Access log records carry both:
1. **OTel-standard fields** in the log record: `traceId`, `spanId`, `severityText: "info"`, `severityNumber: 9`
2. **Log record attributes** using Traefik-proprietary names: `ClientAddr`, `ClientHost`, `DownstreamStatus`, `Duration`, `KubernetesIngressName`, `RequestMethod`, `RequestPath`, `RouterName`, `ServiceAddr`, `ServiceName`, `SpanId`, `TraceId`, `entryPointName`, etc.
3. **Log body**: A JSON string serialization of the access log struct — not a structured map

The log attribute names are Traefik-proprietary and do not follow OTel semantic conventions. For example:
- `RequestMethod` instead of `http.request.method`
- `DownstreamStatus` instead of `http.response.status_code`
- `RequestPath` instead of `url.path`

Additionally, the access log body JSON contains `"level":"panic"` for all normal 200-OK access log entries, while the OTel `severityText` is correctly set to `"info"`. This is a semantic inconsistency — the body-level field appears to be a serialization artifact from Traefik's logrus integration where the access log struct is serialized as a complete Go struct including a zero-value `Level` field that maps to `"panic"` (level 0 in logrus). The OTel severity fields are correct; the body content is misleading.

The log scope is `traefik` with no version set.

The schema URL `https://opentelemetry.io/schemas/1.40.0` is set on the resource logs.

#### Checklist assessment

- ✅ Traces use current OTel HTTP semantic conventions throughout (no deprecated attributes observed in Traefik spans)
- ✅ `http.server.request.duration` and `http.client.request.duration` use OTel semantic convention attribute names
- ⚠️ `traefik_*` metrics use proprietary attribute names (`method`, `code`, `protocol`) instead of OTel conventions
- ⚠️ Access log attributes use Traefik-proprietary names rather than OTel semantic conventions
- ⚠️ Log body contains misleading `"level":"panic"` for normal requests (serialization artifact)
- ✅ Schema URL set to `https://opentelemetry.io/schemas/1.40.0` across all signals
- ❌ Instrumentation scope version absent for traces (`github.com/traefik/traefik` has no version in scope)
- ❌ Log scope version absent (`traefik` scope has no version)

#### Rationale

Traefik's traces are strongly aligned with current OTel semantic conventions — no deprecated HTTP attributes are used. The two OTel semantic convention metric histograms (`http.server.request.duration`, `http.client.request.duration`) are a positive signal. However, the bulk of the metrics (`traefik_*`) use Prometheus-style attribute names that don't align with OTel conventions, and access log attributes use proprietary naming. This mixed picture lands at Level 2: OTel conventions are applied intentionally in the areas where Traefik has done the work, but the `traefik_*` metrics and access log attributes represent incomplete alignment. The project does not reach Level 3 because a first-class signal (metrics) uses proprietary attribute naming where OTel conventions exist, without documenting these as explicit semantic extensions.

---

### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes

Traefik emits a rich set of resource attributes natively via the OTel Go SDK's automatic resource detection:

- **Service identity**: `service.name: traefik`, `service.version: 3.7.0` — stable and consistent across traces, metrics, and logs
- **SDK identity**: `telemetry.sdk.name`, `telemetry.sdk.language: go`, `telemetry.sdk.version: 1.43.0`
- **Host/OS**: `host.name` (set to the pod name by the Go runtime), `os.type: linux`, `os.description`
- **Process**: `process.pid`, `process.executable.name`, `process.executable.path`, `process.owner`, `process.runtime.*`
- **Verbose**: `process.command_args` — contains the full CLI argument array including all Traefik configuration flags. This is extremely verbose for a resource attribute and exposes potentially sensitive configuration details (all endpoint addresses, feature flags, etc.)

Notable absence: `service.instance.id` is not emitted. The pod name is available via `host.name` (Go runtime behavior) and later via `k8s.pod.name` (collector enrichment), but there is no explicit `service.instance.id`.

##### OTEL_* environment variable support

The Traefik documentation states: "Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes." This was confirmed in the research notes. However, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME`, and other standard OTel SDK env vars are **not** respected for exporter configuration — Traefik uses its own `tracing.otlp.grpc.endpoint` etc. configuration path.

##### Identity consistency across signals

`service.name: traefik` and `service.version: 3.7.0` are present and consistent across traces, metrics, and logs. The same `host.name` (pod name) is present across all signals. Identity is stable.

#### Checklist assessment

- ✅ `service.name` is stable and consistent across all three signals
- ✅ `service.version` is present and consistent
- ✅ `OTEL_RESOURCE_ATTRIBUTES` is respected and documented
- ⚠️ `OTEL_SERVICE_NAME` and `OTEL_EXPORTER_OTLP_ENDPOINT` are not respected
- ⚠️ `process.command_args` exposes full CLI flags as a resource attribute — overly verbose and potentially sensitive
- ❌ `service.instance.id` is not emitted
- ✅ Resource attributes are placed correctly (identity on resource, not on spans)
- ✅ Consistent identity across restarts and signals

#### Rationale

Traefik meets the core Level 2 requirements: stable `service.name`/`service.version` consistent across all signals, correct placement of identity on the resource, and partial respect for `OTEL_*` variables (`OTEL_RESOURCE_ATTRIBUTES` works). The lack of `service.instance.id` and the incomplete `OTEL_*` support are gaps, but the overall resource modeling is sound and predictable. The `process.command_args` verbosity is a quality concern (addressed in Audience & Signal Quality) but not a correctness issue. Level 2 is appropriate.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

Each HTTP request through Traefik produces a coherent two-span trace:

1. **Entry-point SERVER span** (`GET`, kind=`SERVER`): Represents the incoming request to Traefik. Attributes include `http.request.method`, `url.path`, `url.scheme`, `http.response.status_code`, `entry_point`, `user_agent.original`, `server.address`, `client.address`, `network.protocol.version`.

2. **ReverseProxy CLIENT span** (`ReverseProxy`, kind=`CLIENT`): Represents the outbound request to the backend. Attributes include `http.request.method`, `url.full`, `url.scheme`, `http.response.status_code`, `network.peer.address`, `network.peer.port`, `server.address`, `server.port`. The `ReverseProxy` span has the `GET` SERVER span as its parent (`parentSpanId` matches the SERVER span's `spanId`).

This is the correct and expected pattern for a reverse proxy: SERVER span for ingress, CLIENT span for egress, with proper parent-child relationship.

With `tracing.addInternals: true`, additional internal spans are emitted for Traefik's own routing (ping endpoint, dashboard), which increases span volume but provides visibility into internal behavior.

##### Context propagation

W3C Trace Context propagation is confirmed working:
- Requests sent **without** a `traceparent` header: Traefik creates a new root SERVER span (no `parentSpanId`).
- Requests sent **with** `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`: Traefik's `GET` SERVER span has `parentSpanId: 00f067aa0ba902b7`, correctly joining the external trace. Multiple requests with the same `traceparent` all share `traceId: 4bf92f3577b34da6a3ce929d0e0e4736`.
- Traefik injects `traceparent` into forwarded requests, so the backend (otel-eval-backend) receives the trace context and its spans appear as children of Traefik's `ReverseProxy` CLIENT span.

The schema URL `https://opentelemetry.io/schemas/1.40.0` is set on all resource spans.

##### Trace coherence

Traces tell a clear story: incoming request → Traefik processing → backend call. The span names (`GET`, `ReverseProxy`) are somewhat generic (see Audience & Signal Quality), but the structure is correct and coherent. The `entry_point` attribute on SERVER spans identifies which Traefik entrypoint handled the request, providing routing context.

#### Checklist assessment

- ✅ W3C Trace Context (`traceparent`) supported and propagated correctly
- ✅ SERVER span for ingress, CLIENT span for egress — correct span kinds
- ✅ Parent-child relationship between SERVER and CLIENT spans is correct
- ✅ External trace context is respected (parent-based sampling)
- ✅ Context is propagated to downstream backends
- ✅ Schema URL set correctly
- ⚠️ Span names are generic (`GET`, `ReverseProxy`) rather than operation-descriptive (see Audience dimension)
- ✅ Trace topology is stable and predictable

#### Rationale

Traefik's trace modeling is intentional and correct for its role as a reverse proxy. The SERVER→CLIENT hierarchy accurately models the proxy's function. W3C Trace Context propagation works end-to-end, confirmed by matching `traceId` across Traefik and backend spans. This is a solid Level 2 implementation. Level 3 would require evidence of architectural review of trace modeling decisions and explicit documentation of the trace model, which is not observed.

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability

All three signals are available as first-class OTLP outputs:
- **Traces**: OTLP gRPC, flowing
- **Metrics**: OTLP gRPC, flowing (plus Prometheus scrape as a co-equal path)
- **Logs**: OTLP gRPC, flowing (behind `experimental.otlpLogs: true`)

The experimental gate on OTLP logs is a caveat — it is not enabled by default and requires explicit opt-in. However, once enabled, logs are a fully functional first-class signal.

##### Cross-signal correlation

Access logs carry explicit trace correlation:
- **OTel log record fields**: `traceId` and `spanId` are set as proper OTel log record fields (not just embedded in the body)
- **Log record attributes**: `TraceId`, `SpanId`, `trace_id`, `span_id` are also present as log attributes (redundant but functional)
- The `traceId` in access logs matches the `traceId` of the corresponding Traefik SERVER span, enabling log-to-trace navigation

This means a user can: start from a latency metric → identify a slow request → find the corresponding trace → inspect the access log for that trace.

Metrics do not carry trace context (this is expected for aggregate metrics), but the metric attributes (`entrypoint`, `router`, `service`) align with the `entry_point` span attribute, enabling metric-to-trace correlation by service/router name.

##### Collection model

| Signal | Method | Notes |
|--------|--------|-------|
| Traces | OTLP gRPC push | Native, always-on when configured |
| Metrics | OTLP gRPC push | Native; also Prometheus pull |
| Logs | OTLP gRPC push | Experimental feature flag required |

#### Checklist assessment

- ✅ All three signals flow via OTLP
- ✅ Access logs include `traceId` and `spanId` as proper OTel log record fields
- ✅ Metrics and traces share conceptual dimensions (`entrypoint`, `router`, `service`)
- ⚠️ OTLP logs require experimental feature flag — not enabled by default
- ⚠️ Metrics do not include trace context (expected for aggregates, but limits drill-down)
- ✅ Signals are designed to complement each other (metrics for aggregate view, traces for individual requests, logs for request details)

#### Rationale

Traefik provides a coherent multi-signal observability story once all three signals are enabled. The trace context in access logs is particularly valuable, enabling direct log-to-trace correlation. The experimental gate on OTLP logs is a friction point but does not prevent the signal from working. Level 2 is appropriate: all three signals are present and intentionally correlated. Level 3 would require evidence of signal cardinality management, explicit guidance on investigative workflows, and signal design evolution based on user feedback.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Traefik's span names are generic and reflect the HTTP method rather than a logical operation:
- Entry-point spans are named `GET` (or `POST`, etc.) — the HTTP method only, with no route or operation context
- The upstream forwarding span is named `ReverseProxy` — an internal component name

For an ingress controller, a more user-oriented span name would be something like `GET /path` or include the router name (e.g., `GET demo-otel-eval-backend@kubernetes`). The current names require inspecting span attributes (specifically `url.path` and `entry_point`) to understand what the span represents. The `http.route` attribute is absent from Traefik's own SERVER spans (it is present in the backend's spans).

##### Signal-to-noise ratio

Several quality concerns:

1. **`process.command_args` as resource attribute**: The full CLI argument array (40+ flags including all endpoint addresses, feature flags, and configuration details) is emitted as a resource attribute on every span, metric data point, and log record. This is unusual for a resource attribute, adds significant payload size, and may expose sensitive configuration data (endpoint URLs, feature flags).

2. **Access log body `"level":"panic"` for normal requests**: The access log body is a JSON string containing `"level":"panic"` for every 200-OK request. This is a serialization artifact (logrus Level 0 = "panic" in Go) but creates a confusing signal for operators who might interpret this as an error condition. The OTel `severityText` field is correctly set to `"info"`, but the body content is misleading.

3. **Internal spans with `addInternals: true`**: When enabled, Traefik emits spans for liveness probes (`/ping`), metrics scrape requests (`/metrics`), and dashboard requests. These internal requests generate significant trace volume and noise in production environments. The `addInternals` flag allows this to be disabled, which is a positive design choice, but the default behavior when enabled includes all internal traffic.

4. **Duplicate trace context in access logs**: The access log attributes contain both `TraceId`/`SpanId` (PascalCase, Traefik-native) and `trace_id`/`span_id` (snake_case, OTel-conformant) as separate attributes, alongside the OTel-standard `traceId`/`spanId` log record fields. This redundancy suggests an evolving implementation rather than a clean design.

##### Default usability

The default configuration (OTLP disabled, requires explicit opt-in) means operators must actively configure telemetry. Once configured, the traces are structurally correct and useful. The metrics provide clear operational signals (request counts, latency histograms by entrypoint/router/service). Access logs provide per-request detail with trace correlation.

The main usability friction is the span naming (generic HTTP method names), the misleading `"level":"panic"` in access log bodies, and the verbose `process.command_args` resource attribute.

#### Checklist assessment

- ⚠️ Span names are generic (`GET`, `ReverseProxy`) — reflect HTTP method and internal component, not logical operations
- ❌ Access log body contains `"level":"panic"` for normal requests — misleading
- ❌ `process.command_args` exposes full CLI flags as a resource attribute — noisy and potentially sensitive
- ✅ Metrics provide clear operational signals at entrypoint/router/service granularity
- ✅ `addInternals` flag allows control over internal span volume
- ⚠️ Duplicate trace context fields in access log attributes
- ✅ Log severity levels are correct at the OTel record level (`severityText: "info"`, `severityNumber: 9`)

#### Rationale

Traefik's telemetry is functional and increasingly user-oriented, but several quality issues prevent a Level 2 rating. The generic span names, misleading `"level":"panic"` in access log bodies, and verbose `process.command_args` resource attribute indicate that telemetry quality has not been fully optimized for operator consumption. These are not fundamental architectural problems — they are refinement opportunities. Level 1 is appropriate: noise has been reduced from what it could be (e.g., `addInternals` is opt-in), but defaults and naming still reflect internal perspectives in key areas.

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

Traefik has dedicated documentation pages for each telemetry signal:
- `reference/install-configuration/observability/tracing/` — covers OTLP configuration, sampling, resource attributes
- `reference/install-configuration/observability/metrics/` — covers OTLP, Prometheus, and other backends
- `reference/install-configuration/observability/logs/` — covers general and access log configuration

The documentation describes configuration options but does not treat telemetry as a stable contract. There is no explicit stability guarantee for span names, metric names, or attribute keys.

##### Change communication

The CHANGELOG contains `[otel]`-tagged entries that track OpenTelemetry-related changes:
- `[logs, otel]` Add OTel-conformant trace context attributes to access logs (#12801)
- `[tracing, otel]` Use ParentBased sampler to respect parent span sampling decision (#12403)
- `[accesslogs, otel]` Allow Stdio access logs alongside OTLP logging (#12307)
- `[logs, metrics, tracing]` Bump go.opentelemetry.io/otel (#13100)

This shows awareness that telemetry changes should be tracked. However, the communication is at the feature/PR level, not at the "here is what changed in the telemetry schema" level. There is no dedicated "Telemetry Changes" section in release notes, and no migration guidance for telemetry-consuming users (dashboards, alerts).

##### Schema URL presence

Schema URL `https://opentelemetry.io/schemas/1.40.0` is set on resource spans, resource metrics (Traefik scope), and resource logs. This is a positive signal — it indicates awareness of schema versioning. However, the schema URL is not bumped when Traefik changes its telemetry (e.g., adding new attributes), suggesting it is set by the OTel SDK rather than actively managed by Traefik.

##### Stability guarantees

- OTLP logs are explicitly marked experimental (`experimental.otlpLogs: true`) — this is a clear stability signal
- No other telemetry features are marked as experimental or stable
- No explicit deprecation policy for telemetry attributes or metric names
- The v2→v3 migration removed all legacy tracers (Jaeger, Zipkin, etc.) — a significant breaking change that was documented in the migration guide

#### Checklist assessment

- ✅ Telemetry-related changes appear in changelog with `[otel]` tags
- ✅ OTLP logs clearly marked experimental
- ✅ Schema URL set on all signals
- ❌ No formal telemetry stability policy or explicit stability guarantees for span/metric names
- ❌ No "Telemetry Changes" section in release notes
- ❌ No migration guidance for telemetry consumers when attributes change
- ⚠️ Schema URL appears SDK-managed rather than actively maintained
- ✅ v2→v3 breaking telemetry changes (legacy tracer removal) were documented in migration guide

#### Rationale

Traefik shows awareness that telemetry changes matter — the `[otel]` changelog tags and the experimental flag on OTLP logs are evidence of this. However, there is no formal policy for telemetry stability, no explicit contract for span/metric names, and no migration guidance for users who depend on specific telemetry shapes. The v2→v3 breaking changes were documented, but this was at a major version boundary. Level 1 is appropriate: informal, inconsistent communication exists, but governance is still emerging.

---

## Key findings

### Strengths

1. **Complete OTLP coverage for all three signals**: Traefik v3 supports OTLP push for traces, metrics, and logs. Traces use current OTel HTTP semantic conventions throughout (no deprecated attributes). The OTel SDK is deeply embedded (opentelemetry-go v1.43.0) and used consistently.

2. **Excellent trace modeling and W3C context propagation**: The SERVER→CLIENT span hierarchy is correct for a reverse proxy. W3C Trace Context propagation works end-to-end — Traefik correctly joins external traces and injects context into forwarded requests. The schema URL is set on all signals.

3. **Strong log-trace correlation**: Access logs carry `traceId` and `spanId` as proper OTel log record fields, enabling direct log-to-trace navigation. The trace IDs in access logs match the corresponding Traefik SERVER spans.

### Areas for improvement

1. **Standardize `traefik_*` metric attribute names to OTel conventions**: The `traefik_*` metrics use Prometheus-style attribute names (`method`, `code`, `protocol`) instead of OTel semantic convention names (`http.request.method`, `http.response.status_code`, `network.protocol.name`). Aligning these would enable off-the-shelf OTel dashboards and reduce the semantic gap between the `traefik_*` metrics and the `http.server.request.duration` / `http.client.request.duration` histograms.

2. **Fix access log body `"level":"panic"` for normal requests**: The access log JSON body contains `"level":"panic"` for every successful request due to a logrus Level 0 serialization artifact. This is misleading for operators and should be fixed (e.g., omit the `level` field from access log body serialization, or set it to `"info"`).

3. **Reduce `process.command_args` verbosity and promote OTLP logs to stable**: The `process.command_args` resource attribute exposes all CLI flags (40+ entries) on every telemetry record, adding payload size and potentially exposing sensitive configuration. This should be opt-in or filtered. Additionally, OTLP logs have been functional since Traefik v3 and should be promoted out of experimental status with explicit stability guarantees.

### Notable observations

- **Span names reflect HTTP method only** (`GET`, `ReverseProxy`): Unlike many frameworks that include the route template in the span name (e.g., `GET /api/users/{id}`), Traefik only uses the HTTP method. This is a deliberate choice (avoids high-cardinality span names from unmatched routes), but it reduces the immediate readability of traces.

- **`"level":"panic"` in access log bodies**: All 14 access log records observed had `"level":"panic"` in the body JSON while `severityText` was correctly `"info"`. This is a known serialization artifact where logrus's Level type serializes its zero value as `"panic"`, but it is surprising and potentially alarming for operators.

- **Dual trace context in access log attributes**: Access log records contain both Traefik-native (`TraceId`, `SpanId` in PascalCase) and OTel-conformant (`trace_id`, `span_id` in snake_case) attribute names, plus the proper OTel log record `traceId`/`spanId` fields — three representations of the same data. This redundancy reflects the incremental evolution of the feature (PR #12801 added OTel-conformant attributes alongside the existing ones).

- **Instrumentation scope version absent for traces**: The trace scope `github.com/traefik/traefik` has no version set (the `version` field is absent). The metrics scope correctly sets `version: "3.7.0"`. This inconsistency means trace data cannot be filtered by instrumentation library version.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file export in a local kind cluster (kind-otel-eval-traefik)
- The k8sattributes processor enriched telemetry with Kubernetes metadata; this enrichment is noted and not credited to Traefik's native capabilities
- Traefik v3.7.0 was deployed via Helm chart 40.0.0 with OTLP traces, metrics, and logs all configured to push to the in-cluster OTel Collector
- Traffic was generated via `curl` with and without `traceparent` headers to verify context propagation
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv v1.x stable)
- Documentation at `doc.traefik.io/traefik/` and the GitHub CHANGELOG were reviewed for context
- The Prometheus receiver in the collector was also configured to scrape Traefik's `/metrics` endpoint; Prometheus-scraped metrics are noted as a separate export path
