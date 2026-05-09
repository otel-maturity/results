# OpenTelemetry Support Maturity Evaluation: Traefik

## Project overview

- **Project**: Traefik — cloud-native HTTP reverse proxy and Kubernetes Ingress controller
- **Version evaluated**: v3.7.0 (Helm chart 40.0.0)
- **Evaluation date**: 2026-05-09
- **Cluster**: otel-eval-traefik (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.2

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP is a first-class, well-documented path; Prometheus scrape remains co-equal |
| Semantic Conventions | 2 | Trace spans use current semconv; Prometheus metrics use legacy labels; access log attributes are Traefik-native CamelCase |
| Resource Attributes & Configuration | 2 | Rich native resource attributes; `OTEL_RESOURCE_ATTRIBUTES` documented and respected; `service.instance.id` absent natively |
| Trace Modeling & Context Propagation | 2 | W3C Trace Context propagation confirmed; intentional SERVER/CLIENT span kinds; span names are HTTP-method-only (not route-templated) |
| Multi-Signal Observability | 2 | All three signals flow via OTLP; access logs carry trace/span IDs; OTLP logs remain behind an experimental feature gate |
| Audience & Signal Quality | 1 | Trace and metrics signals are operator-ready; access log attribute schema is Traefik-native (not OTel semconv); `level: panic` body quirk for all access logs |
| Stability & Change Management | 1 | OTel changes appear in changelog; no formal telemetry stability contract; OTLP logs still experimental; no schema URL on traces |

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC push to collector
- **Metrics (OTLP)**: Flowing — OTLP gRPC push to collector (10 s interval)
- **Metrics (Prometheus)**: Flowing — Prometheus scrape by collector (15 s interval)
- **Logs (access)**: Flowing — OTLP gRPC push to collector (requires `experimental.otlpLogs: true`)
- **Logs (general/application)**: Flowing — OTLP gRPC push to collector (same experimental gate)

### Resource attributes (native, before collector enrichment)

Traefik sets these resource attributes natively via the OTel Go SDK:

| Attribute | Example value |
|---|---|
| `service.name` | `traefik` |
| `service.version` | `3.7.0` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-599d5c8d5-d4vnb` (pod name) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux …)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |
| `process.command_args` | full CLI args array |

**Note**: Traefik also natively detects and sets Kubernetes resource attributes (`k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.deployment.name`, etc.) via a built-in K8s resource detector introduced in recent v3 releases (PR #11906). These appear in the raw OTLP payloads before any collector enrichment. The k8sattributes processor in the collector adds additional label and annotation attributes on top.

### Resource attributes (after collector enrichment)

The k8sattributes processor added: `k8s.pod.label.*`, `k8s.pod.annotation.*`, `k8s.replicaset.name`, `k8s.replicaset.uid`, `k8s.deployment.uid`, `k8s.node.uid`.

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- All three signals (traces, metrics, logs) support OTLP gRPC and OTLP HTTP export natively in Traefik v3.
- Traces are configured via `tracing.otlp.grpc` or `tracing.otlp.http`. OTLP is the **only** supported tracing export path in v3 — legacy Jaeger/Zipkin/Datadog exporters still exist for traces but are separate backend configurations, not the default.
- Metrics support both OTLP push (`metrics.otlp`) and Prometheus scrape (`metrics.prometheus`) simultaneously. Prometheus is the historical default and remains equally prominent in documentation.
- OTLP log export works for both access logs and general logs via `experimental.otlpLogs: true` feature gate.
- Documentation at `doc.traefik.io` clearly describes OTLP as the recommended path for tracing. The Helm chart has first-class `tracing.otlp` and `metrics.otlp` value keys.
- Confirmed: Traefik connects directly to an existing OTel Collector on port 4317 with zero custom adapters or sidecars.

#### Checklist assessment

- ✅ OTLP export is supported for all three signals
- ✅ Users can connect to an existing OTel Collector without adapters
- ✅ Configuration is consistent across traces, metrics, and logs (all use `*.otlp.grpc.*` pattern)
- ✅ OTel Go SDK is used natively — not a wrapper or bridge
- ⚠️ Prometheus scrape remains co-equal with OTLP push for metrics — not clearly secondary
- ⚠️ OTLP log export requires `experimental.otlpLogs: true` — not yet a stable, default path
- ⚠️ Telemetry integration is not yet documented as a versioned, stable contract (no migration guides for telemetry-specific changes)

#### Rationale

Traefik v3 fully supports OTLP across all signals and integrates cleanly into OTel pipelines. The project "speaks OpenTelemetry" fluently. However, Prometheus metrics remain a co-equal first-class option (not clearly deprecated), and OTLP log export is still experimental. This places the project solidly at Level 2 — OTel is the primary integration surface for traces, and increasingly so for metrics and logs, but the integration surface is not yet explicitly governed as a stable contract (Level 3 would require that).

---

### 2. Semantic Conventions

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Trace attributes

Traefik v3 uses **current, stable OTel HTTP semantic conventions** on spans:

| Attribute | Used | Notes |
|---|---|---|
| `http.request.method` | ✅ | Current semconv (not deprecated `http.method`) |
| `http.response.status_code` | ✅ | Current semconv (not deprecated `http.status_code`) |
| `url.path` | ✅ | Current semconv (not deprecated `http.target`) |
| `url.full` | ✅ | Used on CLIENT spans (ReverseProxy) |
| `url.scheme` | ✅ | Current semconv |
| `url.query` | ✅ | Current semconv |
| `user_agent.original` | ✅ | Current semconv |
| `server.address` | ✅ | Current semconv |
| `server.port` | ✅ | Current semconv |
| `network.peer.address` | ✅ | Current semconv |
| `network.peer.port` | ✅ | Current semconv |
| `network.protocol.version` | ✅ | Current semconv |
| `client.address` | ✅ | Current semconv |
| `client.port` | ✅ | Current semconv |
| `http.request.body.size` | ✅ | Current semconv |
| `entry_point` | ⚠️ | Traefik-specific, not in OTel semconv — identifies the Traefik entrypoint (e.g., `web`, `traefik`) |

No deprecated attributes (`http.method`, `http.status_code`, `http.target`, `http.url`, `http.flavor`, `net.host.*`, `net.peer.*`) were observed on Traefik's own spans.

Source confirms: `semconv "go.opentelemetry.io/otel/semconv/v1.26.0"` is used in the tracing implementation.

##### Metric names and attributes

Two metric naming conventions are in use simultaneously:

**OTLP push (OTel semconv):**
- `http.server.request.duration` (histogram) — current OTel HTTP semconv metric
- `http.client.request.duration` (histogram) — current OTel HTTP semconv metric
- Attributes: `http.request.method`, `http.response.status_code`, `url.scheme`, `server.address`, `network.protocol.name`, `network.protocol.version` — all current semconv

**Prometheus scrape (Traefik-native naming):**
- `traefik_entrypoint_requests_total`, `traefik_entrypoint_request_duration_seconds`, etc.
- Attributes: `code`, `entrypoint`, `method`, `protocol`, `router`, `service` — Traefik-specific labels, not OTel semconv

The Prometheus metrics use `code` (not `http.response.status_code`), `method` (not `http.request.method`), and `protocol` (not `network.protocol.name`). This is a semantic inconsistency between the two metrics paths.

##### Log attributes

Access log attributes use **Traefik-native CamelCase schema**, not OTel semantic conventions:

- `RequestMethod`, `RequestPath`, `RequestProtocol`, `DownstreamStatus`, `RouterName`, `ServiceName`, `ServiceAddr`, `KubernetesIngressName`, `Duration`, `Overhead`, etc.
- Trace context is present via both `TraceId`/`SpanId` (CamelCase) and `trace_id`/`span_id` (snake_case) — duplicated in both the log attributes and embedded in the JSON body string.
- The OTLP `traceId` and `spanId` fields are correctly populated.
- Log severity is `info` (severityNumber=9) for all access log records — correct for normal requests.

The log attribute schema does not use OTel semantic conventions (`http.request.method`, `url.path`, etc.) — it uses the Traefik-native access log field names.

#### Checklist assessment

- ✅ Current OTel HTTP semconv used on all trace spans (no deprecated `http.method`, `http.status_code`, etc.)
- ✅ OTLP metrics use OTel semconv metric names and attribute keys
- ✅ Span kinds are correct: SERVER (kind=2) for entry-point spans, CLIENT (kind=3) for ReverseProxy spans
- ⚠️ Prometheus metrics use Traefik-native label names (`code`, `method`, `protocol`) — inconsistent with trace attributes
- ⚠️ Access log attributes use Traefik-native CamelCase schema — not OTel semconv
- ⚠️ `entry_point` span attribute is Traefik-specific with no documented mapping to OTel conventions
- ⚠️ Instrumentation scope version is `vunknown` on traces (scope name is `github.com/traefik/traefik` but no version set)
- ⚠️ No `http.route` attribute on server spans — the route template is not exposed, only the method

#### Rationale

Traefik's trace spans are an excellent example of current OTel semconv adoption — all HTTP attributes use the latest stable conventions with no deprecated fields. The OTLP metrics also use OTel semconv names. However, the Prometheus metrics path uses Traefik-native attribute names, and the access log schema is entirely Traefik-native. The inconsistency between signals prevents Level 3. Level 2 is appropriate: conventions are applied consistently and deliberately on traces, and OTel semconv metrics are present, but the Prometheus label schema and log attribute schema are not aligned.

---

### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes

Traefik sets a rich set of resource attributes natively via the OTel Go SDK's automatic resource detection:

- **Service identity**: `service.name=traefik`, `service.version=3.7.0` — consistent across traces, metrics, and logs
- **SDK info**: `telemetry.sdk.name`, `telemetry.sdk.language`, `telemetry.sdk.version`
- **Process info**: `process.pid`, `process.executable.name`, `process.executable.path`, `process.owner`, `process.runtime.name`, `process.runtime.version`, `process.runtime.description`, `process.command_args`
- **OS info**: `os.type`, `os.description`
- **Host info**: `host.name` (set to pod name)
- **Kubernetes info** (natively detected, not collector-derived): `k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.deployment.name`, `k8s.container.name`, `k8s.node.name`, `k8s.replicaset.name`

The Kubernetes attributes are set natively by Traefik's built-in K8s resource detector (PR #11906 in v3.x). This is a notable strength — Traefik does not require the collector's k8sattributes processor for basic K8s identity.

**Missing**: `service.instance.id` is not set natively. The pod UID is available as `k8s.pod.uid` but is not mapped to `service.instance.id`.

##### OTEL_* environment variable support

Documentation at `doc.traefik.io` explicitly states: "Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes." This is confirmed as a documented, supported mechanism.

Configuration uses Traefik-native CLI flags/config file options (`tracing.otlp.grpc.endpoint`, `metrics.otlp.grpc.endpoint`) rather than `OTEL_EXPORTER_OTLP_ENDPOINT`. The OTel standard env vars are not the primary configuration mechanism for export endpoints — Traefik has its own config path.

##### Identity consistency across signals

`service.name=traefik` and `service.version=3.7.0` are consistent across all three signals (traces, metrics via OTLP, logs). The metrics scope for Traefik's OTLP push correctly reports `github.com/traefik/traefik v3.7.0`.

#### Checklist assessment

- ✅ `service.name` and `service.version` are stable and consistent across all signals
- ✅ Rich process, OS, host, and Kubernetes resource attributes set natively
- ✅ `OTEL_RESOURCE_ATTRIBUTES` is documented and supported
- ✅ Identity is correct across traces, metrics, and logs
- ⚠️ `service.instance.id` is not set natively (pod UID exists as `k8s.pod.uid` but not mapped)
- ⚠️ Export endpoint is configured via Traefik-native flags (`tracing.otlp.grpc.endpoint`), not `OTEL_EXPORTER_OTLP_ENDPOINT`
- ⚠️ Resource attribute behavior is not formally documented as a stable contract with migration guidance

#### Rationale

Traefik provides excellent native resource attribution — service identity, process info, OS info, and even Kubernetes metadata are all set at the source. `OTEL_RESOURCE_ATTRIBUTES` is documented. The main gaps are the absence of `service.instance.id` and the use of Traefik-native config paths rather than standard `OTEL_*` env vars for export endpoints. Level 2 is appropriate: resource identity is stable and consistent, but configuration precedence is not fully documented and `OTEL_EXPORTER_OTLP_ENDPOINT` is not respected.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

Observed span types from Traefik:

| Span name | Kind | Description |
|---|---|---|
| `GET` | SERVER (kind=2) | Entry-point span — one per incoming HTTP request |
| `ReverseProxy` | CLIENT (kind=3) | Upstream backend call span |

Traefik v3 produces a clean two-level trace structure for proxied requests:
1. A SERVER span named `GET` (or the HTTP method) representing the inbound request at the entrypoint
2. A CLIENT span named `ReverseProxy` as a child, representing the outbound call to the backend

The backend (otel-eval-backend) then creates its own child spans (`GET /`, `GET /health`, `request handler - /`, `middleware - query`, etc.) continuing the same trace.

When `tracing.addInternals: true` is set, internal routes (ping, dashboard) also produce spans.

**Span naming observation**: The root SERVER span is named only with the HTTP method (`GET`), not with a route template. There is no `http.route` attribute. For an ingress controller handling many different routes, this means all inbound requests produce spans named `GET` — making it impossible to distinguish routes in a trace waterfall without inspecting `url.path`. This is a known limitation and is a design choice (route templates are in the router/service labels).

##### Context propagation

W3C Trace Context (`traceparent`/`tracestate`) propagation is confirmed working:

- When an incoming request carries `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`, Traefik correctly creates a child SERVER span with `parentSpanId=00f067aa0ba902b7` — preserving the same trace ID and creating a new span ID.
- The ReverseProxy CLIENT span is a child of the SERVER span, with the correct `parentSpanId`.
- The backend receives the updated `traceparent` and continues the trace, with its spans appearing as children of Traefik's CLIENT span.
- Verified in telemetry: spans from `otel-eval-backend` have `parentSpanId` values that match Traefik's ReverseProxy span IDs.

##### Trace coherence

The trace structure tells a coherent story: `[external] → GET (Traefik SERVER) → ReverseProxy (Traefik CLIENT) → GET / (backend SERVER) → [middleware spans]`. Parent-child relationships are correct throughout.

#### Checklist assessment

- ✅ W3C Trace Context propagation is correctly implemented and verified
- ✅ Entry-point spans are consistently SERVER spans (kind=2)
- ✅ Backend proxy calls are consistently CLIENT spans (kind=3)
- ✅ Correct parent-child relationships across Traefik and backend
- ✅ Sampling decision (`ParentBased` sampler) respects incoming trace context
- ⚠️ Span names are HTTP method only (`GET`) — no route template or `http.route` attribute
- ⚠️ Instrumentation scope version is `vunknown` — scope name is set but version is missing
- ⚠️ No documented trace modeling design decisions or architecture docs

#### Rationale

Traefik's trace modeling is intentional and correct for its role as a proxy. W3C Trace Context propagation works end-to-end, span kinds are correct, and the parent-child structure is coherent. The main limitation is that span names use only the HTTP method (`GET`), not a route template, which reduces the usefulness of traces for route-level analysis. This is a known, documented design choice in the context of Traefik's routing model. Level 2 is appropriate: tracing is intentionally designed, propagation works correctly, but the span naming does not yet expose route-level semantics.

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability

All three signals are present and flowing via OTLP:

| Signal | Status | Transport | Notes |
|---|---|---|---|
| Traces | First-class | OTLP gRPC | Stable, documented |
| Metrics (OTLP) | First-class | OTLP gRPC push | Stable, documented |
| Metrics (Prometheus) | First-class | Prometheus scrape | Stable, co-equal with OTLP |
| Logs (access) | Present | OTLP gRPC | Requires `experimental.otlpLogs: true` |
| Logs (general) | Present | OTLP gRPC | Requires `experimental.otlpLogs: true` |

##### Cross-signal correlation

- **Trace context in logs**: All 28 access log records have `traceId` and `spanId` populated in both the OTLP `traceId`/`spanId` fields and as log attributes (`trace_id`, `span_id`, `TraceId`, `SpanId`). Cross-signal correlation from logs to traces works natively.
- **Metrics to traces**: The OTLP metrics (`http.server.request.duration`) share attribute keys with trace spans: `http.request.method`, `http.response.status_code`, `url.scheme`, `server.address`. A user can pivot from a latency metric to traces using these shared dimensions.
- **Prometheus metrics**: The `traefik_*` Prometheus metrics use Traefik-native label names (`code`, `method`, `entrypoint`, `router`, `service`) that do not directly match trace span attributes, making pivot from Prometheus metrics to traces less natural.

##### Collection model

- Traces: OTLP push (Traefik → Collector)
- OTLP Metrics: OTLP push (Traefik → Collector)
- Prometheus Metrics: Prometheus pull (Collector → Traefik `/metrics`)
- Logs: OTLP push (Traefik → Collector)

#### Checklist assessment

- ✅ All three signals are present and flowing
- ✅ Access logs include trace and span IDs — cross-signal correlation works
- ✅ OTLP metrics share attribute keys with trace spans
- ✅ Signals can be correlated without manual stitching (for traces + logs)
- ⚠️ OTLP log export requires `experimental.otlpLogs: true` — not a stable default
- ⚠️ Prometheus metrics use different label names than trace span attributes — pivot from Prometheus to traces requires label remapping
- ⚠️ No documented guidance on when to use which signal or how to navigate between them

#### Rationale

Traefik provides all three signals via OTLP, and the cross-signal correlation story is strong — access logs carry trace context natively, and OTLP metrics share attribute keys with traces. The experimental gate on OTLP logs and the naming inconsistency between Prometheus metrics and trace attributes are the main gaps preventing Level 3. Level 2 is appropriate: signals are intentionally correlated and designed to work together, but the experience is not yet fully optimized.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

Traefik's SERVER spans are named with the HTTP method only: `GET`. This is technically correct per OTel HTTP semconv (the `name` should be `{method}` for server spans when the route is not known at span creation time, or `{method} {route}` when it is). However, for an ingress controller that always knows the matched route (via `RouterName`), the absence of route information in the span name significantly limits trace usability. Every `GET` request to any route produces a span named `GET`, making it impossible to distinguish routes in a trace list view without drilling into attributes.

The `entry_point` attribute (e.g., `web`, `traefik`) provides some context, but the router name (e.g., `demo-otel-eval-backend@kubernetes`) is not present as a span attribute on the SERVER span.

##### Signal-to-noise ratio

- **Traces**: With `addInternals: true`, Traefik traces all internal routes including health checks (`/ping`) and dashboard requests. This can generate significant noise in production. The feature is opt-in, which is appropriate.
- **Metrics**: The Prometheus scrape path produces ~55 metrics including all Go runtime metrics (`go_memstats_*`, `go_gc_*`) which are useful for deep debugging but are noisy for operational dashboards. These are standard Go Prometheus metrics, not Traefik-specific.
- **Logs**: Access logs have a known quirk: `"level":"panic"` appears in the JSON body for all access log records regardless of HTTP status code (e.g., a successful 200 response has `level: panic` in the body). This is a Traefik serialization bug/quirk that can cause alert fatigue if log-level alerting is based on the body content. The OTLP `severityText` is correctly set to `info` (severityNumber=9), but the embedded JSON body says `panic`.
- **Log body**: The OTLP log body is a JSON string (not a structured object). All access log fields appear both as top-level OTLP log attributes AND embedded inside the JSON string body — duplication that increases payload size.
- **Log attributes**: `TraceId` and `trace_id` are both present as log attributes (CamelCase and snake_case duplicates). Similarly `SpanId` and `span_id`.

##### Default usability

- Operators can use Traefik's traces to understand request flow without deep internal knowledge.
- The `traefik_*` Prometheus metrics are operationally useful for dashboards (request rates, durations, open connections).
- The OTLP `http.server.request.duration` and `http.client.request.duration` metrics are directly compatible with OTel-based dashboards.
- Access logs provide rich per-request detail including routing decisions (`RouterName`, `ServiceName`, `KubernetesIngressName`), which is valuable for debugging routing issues.

#### Checklist assessment

- ✅ Trace spans use correct span kinds and current semconv attributes
- ✅ Metrics provide operationally useful signals (request rates, durations, connection counts)
- ✅ Access logs include rich routing context (router, service, Kubernetes metadata)
- ⚠️ Span names are HTTP method only — no route template, limiting trace usability
- ⚠️ `level: panic` in access log bodies for all requests — known serialization quirk
- ⚠️ Log attributes duplicate trace context in both CamelCase and snake_case
- ⚠️ Log body is a JSON string (not structured) — redundant with extracted attributes
- ⚠️ Access log attribute schema uses Traefik-native names, not OTel semconv
- ⚠️ `addInternals: true` generates health check trace noise (opt-in, but default in this evaluation)

#### Rationale

Traefik's traces and OTLP metrics are reasonably operator-ready, but the access log signal has several quality issues (panic-level body, duplicated attributes, non-semconv naming) that require user-side filtering or awareness. The absence of route templates in span names limits trace usability for an ingress controller. Level 1 is appropriate: some signals are user-oriented and noise is reduced in some areas, but signal quality is inconsistent across the three signals and operators need project-specific knowledge to interpret access log data correctly.

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

Traefik documents its OTel configuration options thoroughly at `doc.traefik.io`. The tracing, metrics, and access log pages describe all available options. However, there is no explicit documentation of the telemetry schema as a stable contract — no document stating "these span names, attributes, and metric names are stable and will not change without notice."

The `experimental.otlpLogs: true` feature gate explicitly marks OTLP log export as unstable. No similar stability markers exist for traces or metrics.

##### Change communication

The Traefik CHANGELOG contains an `[otel]` tag for OpenTelemetry-related changes. Examples from recent releases:

- `**[otel]** Update OpenTelemetry to v1.38.0 and semantic conventions to v1.37.0` — semconv version bumps are tracked
- `**[tracing]** Follow OTel semantic conventions for root span naming` — span naming changes are noted
- `**[middleware,tracing]** Introduce trace verbosity config and produce less spans by default` — behavioral changes to span emission are tracked
- `**[metrics,otel]** Rename traefik_tls_certs_not_after_milliseconds to traefik_tls_certs_not_after_seconds` — metric renames are noted
- `**[logs,otel]** Add OTel-conformant trace context attributes to access logs` — log schema additions are noted

Changes are tracked in the changelog but without migration guidance for users. The rename of `traefik_tls_certs_not_after_milliseconds` to `traefik_tls_certs_not_after_seconds` is a breaking change for dashboards and alerts, noted in a single changelog entry without a migration guide.

##### Schema URL presence

- **Traces**: No `schemaUrl` set in OTLP exports from Traefik. The scope is `github.com/traefik/traefik` with version `vunknown`.
- **Metrics (OTLP)**: Schema URL `https://opentelemetry.io/schemas/1.40.0` is present on Traefik's OTLP metrics.
- **Logs**: Schema URL `https://opentelemetry.io/schemas/1.40.0` is present on log exports.

The absence of `schemaUrl` on traces (while present on metrics and logs) is inconsistent.

##### Stability guarantees

No explicit stability guarantees for telemetry are documented. The `experimental.otlpLogs` gate is the only formal stability marker observed. Traefik does not distinguish between stable and experimental telemetry for traces and metrics.

#### Checklist assessment

- ✅ OTel-related changes appear in the changelog with `[otel]` tag
- ✅ Schema URL is set on OTLP metrics and logs
- ✅ OTLP log export is explicitly marked experimental
- ⚠️ No `schemaUrl` on trace exports
- ⚠️ No formal telemetry stability contract or documentation
- ⚠️ Breaking changes (metric renames) noted in changelog but without migration guides
- ⚠️ No distinction between stable and experimental telemetry for traces and metrics
- ⚠️ Instrumentation scope version is `vunknown` on traces — version not set in scope

#### Rationale

Traefik shows awareness that telemetry changes have impact — OTel changes are tracked in the changelog with a dedicated tag, and the experimental log feature is explicitly labeled. However, there is no formal telemetry stability contract, no migration guides for breaking changes, and no explicit stability labels for traces and metrics. Schema URL is inconsistently set (present on metrics and logs, absent on traces). Level 1 is appropriate: intent exists and some changes are communicated, but governance is still emerging.

---

## Key findings

### Strengths

- **Comprehensive native OTLP support**: All three signals (traces, metrics, logs) flow via OTLP gRPC natively, with no sidecars or adapters required. Traefik connects directly to an OTel Collector.
- **Current semantic conventions on traces**: Traefik v3 uses the latest stable OTel HTTP semconv attributes (`http.request.method`, `http.response.status_code`, `url.path`, `url.full`, etc.) with no deprecated attributes observed. The implementation references `semconv/v1.26.0`.
- **Rich native resource attributes including Kubernetes metadata**: Traefik natively sets `service.name`, `service.version`, process, OS, host, and Kubernetes resource attributes (`k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.deployment.name`, etc.) without requiring collector enrichment.
- **Verified W3C Trace Context propagation**: Incoming `traceparent` headers are correctly honored — Traefik preserves the trace ID and creates child spans, enabling end-to-end distributed tracing across the ingress boundary.
- **Trace context in access logs**: All access log records carry `traceId` and `spanId` in OTLP fields, enabling natural cross-signal correlation from logs to traces.

### Areas for improvement

- **Add `http.route` to SERVER spans**: Traefik knows the matched router name at span time. Exposing it as `http.route` (or a Traefik-specific attribute like `traefik.router.name`) would transform span names from generic `GET` to route-aware `GET /api/v1/users/{id}`, dramatically improving trace usability.
- **Fix the `level: panic` access log body quirk**: All access log records embed `"level":"panic"` in the JSON body regardless of HTTP status. This is a serialization bug that causes confusion and potential alert fatigue. The OTLP `severityText` is correctly `info`, but the embedded body is misleading.
- **Stabilize OTLP log export and remove the experimental gate**: OTLP log export is the most natural path for Traefik in OTel-native environments. Stabilizing `otlpLogs` and making it a first-class, documented feature would complete the three-signal story.
- **Set `schemaUrl` on trace exports**: Metrics and logs correctly set `schemaUrl=https://opentelemetry.io/schemas/1.40.0`. Traces should do the same for consistency and tooling compatibility.
- **Align Prometheus metric attribute names with OTel semconv**: The `traefik_*` Prometheus metrics use `code`, `method`, `protocol` labels instead of `http.response.status_code`, `http.request.method`, `network.protocol.name`. Aligning these would enable natural pivot from Prometheus metrics to traces.

### Notable observations

- **Dual metrics paths create semantic inconsistency**: Traefik simultaneously pushes OTel-semconv metrics (`http.server.request.duration` with `http.request.method`) and exposes Prometheus metrics (`traefik_entrypoint_requests_total` with `method`) for the same underlying data. This is a pragmatic choice for backward compatibility but creates a split observability experience.
- **Traefik's built-in K8s resource detector is a standout feature**: Unlike most projects that rely entirely on the collector's k8sattributes processor, Traefik natively detects and sets Kubernetes resource attributes. This reduces pipeline complexity and ensures K8s identity is present even in minimal collector configurations.
- **Access log attributes are rich but non-standard**: The access log schema (`RouterName`, `ServiceName`, `KubernetesIngressName`, `Duration`, `Overhead`, `RetryAttempts`) provides valuable ingress-specific context not available in traces. However, using Traefik-native names instead of OTel semconv means these attributes cannot be interpreted by generic OTel tooling without project-specific knowledge.
- **Instrumentation scope version is `vunknown`**: The trace instrumentation scope (`github.com/traefik/traefik`) does not set a version, making it harder for backends to track semconv version changes.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file exporter in a local kind cluster (`otel-eval-traefik`)
- Traefik v3.7.0 was installed via Helm chart 40.0.0 with OTLP gRPC configured for traces, metrics, and logs
- The k8sattributes processor was used to distinguish native vs enriched resource attributes (Traefik's built-in K8s detector sets core K8s attributes natively)
- Semantic conventions were checked against the latest stable OTel specification (v1.26.0+ for HTTP)
- Traffic generation included normal requests, requests with injected `traceparent` headers, health check requests, and 404 requests
- Documentation reviewed: `doc.traefik.io` tracing reference, metrics reference, access logs reference, and the Traefik GitHub CHANGELOG
- Source code reviewed: `pkg/tracing/tracing.go` for semconv version and attribute implementation
