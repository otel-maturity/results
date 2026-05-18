# OpenTelemetry Support Maturity Evaluation: Traefik

## Project overview

- **Project**: Traefik — cloud-native HTTP reverse proxy and Kubernetes Ingress controller (CNCF Incubating)
- **Version evaluated**: v3.7.1 (Helm chart `traefik/traefik` v40.2.0)
- **Evaluation date**: 2026-05-18
- **Evaluation run version**: v1
- **Cluster**: otel-eval-traefik (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.3

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP gRPC is the primary path for all three signals; Prometheus scrape remains as a parallel option |
| Semantic Conventions | 1 | Traces use current OTel HTTP semconv; `traefik_*` metrics use proprietary labels (`code`, `method`, `protocol`); log attributes use PascalCase proprietary names |
| Resource Attributes & Configuration | 2 | Rich native resource identity (`service.name`, `service.version`, `telemetry.sdk.*`, `process.*`, `os.*`); no `service.instance.id` on traces/OTLP metrics |
| Trace Modeling & Context Propagation | 1 | All spans are flat SERVER spans with no child spans; W3C traceparent propagation confirmed; span name is bare `GET` without route |
| Multi-Signal Observability | 1 | All three signals flow via OTLP; access log trace correlation present but application logs lack `severityText`; signals are not yet designed as a correlated system |
| Audience & Signal Quality | 1 | Trace span names are bare HTTP method (`GET`); access log body is a JSON string (not structured); `severityText` missing from application logs; `entry_point` attribute adds useful context |
| Stability & Change Management | 1 | Telemetry changes mentioned in changelogs informally; OTLP log export still behind experimental feature gate; no explicit stability guarantees or migration guides for telemetry |

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC push, native OTel Go SDK (`github.com/traefik/traefik`)
- **Metrics**: Flowing — OTLP gRPC push (native) + Prometheus scrape (via collector)
- **Logs**: Flowing — OTLP gRPC push (experimental feature gate), scope `traefik vunknown`

### Resource attributes (native, before collector enrichment)

Traefik emits these natively via the OTel Go SDK on traces and OTLP metrics:

| Attribute | Value (example) |
|-----------|-----------------|
| `service.name` | `traefik` |
| `service.version` | `3.7.1` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-577cd8ff58-sszqj` (pod name) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.10` |
| `process.runtime.description` | Full Go version string |
| `process.command_args` | Full CLI args array |

**Note**: `service.instance.id` is absent from traces and OTLP metrics. It is present only in Prometheus-scraped metrics (collector-derived, set to `traefik-metrics.traefik.svc.cluster.local:9100`).

### Resource attributes (after collector enrichment)

The k8sattributes processor added:
- `k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.deployment.name`, `k8s.replicaset.name`, `k8s.node.name`, `k8s.container.name`
- `k8s.pod.label.*`, `k8s.pod.annotation.*`

Schema URL present on traces and logs: `https://opentelemetry.io/schemas/1.40.0`

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

Traefik v3.7.1 supports OTLP gRPC natively for all three signals. The configuration is explicit and direct:

- **Traces**: `--tracing.otlp.grpc.endpoint` — OTLP gRPC push, no adapters required
- **Metrics**: `--metrics.otlp.grpc.endpoint` — OTLP gRPC push; Prometheus scrape endpoint also available in parallel
- **Logs**: `--accesslog.otlp.grpc.endpoint` and `--log.otlp.grpc.endpoint` — OTLP gRPC push (behind `--experimental.otlpLogs=true`)

The process command args in the telemetry data confirm all three OTLP endpoints are configured identically, pointing to the same collector endpoint. The collector received data on all three signal pipelines without any adapters or sidecars.

Prometheus scraping remains as a parallel option (not deprecated), which is a practical concession to existing Prometheus-based stacks. Prometheus is documented alongside OTLP but is not the primary recommended path for new deployments.

#### Checklist assessment

- ✅ OTLP is supported natively for traces, metrics, and logs
- ✅ Users can connect to an existing OTel Collector without adapters
- ✅ Configuration is done via Traefik-specific flags (not `OTEL_*` env vars — see Resource Attributes dimension)
- ✅ Prometheus scraping remains available but is secondary
- ⚠️ OTLP log export requires `--experimental.otlpLogs=true` feature gate (not yet stable)
- ✅ All signals flow to the same collector endpoint without custom glue code

#### Rationale

OTLP is the primary and recommended integration surface for all three signals. The project integrates cleanly with existing OTel Collectors. Legacy Prometheus support coexists but is clearly a compatibility option. The experimental feature gate on logs is a minor caveat that does not prevent OTLP from being the primary path. This is solidly Level 2.

---

### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Trace attributes

Trace spans use current OTel HTTP semantic conventions for server spans:

| Attribute | Status |
|-----------|--------|
| `http.request.method` | ✅ Current semconv |
| `http.response.status_code` | ✅ Current semconv |
| `url.path` | ✅ Current semconv |
| `url.query` | ✅ Current semconv |
| `url.scheme` | ✅ Current semconv |
| `server.address` | ✅ Current semconv |
| `network.protocol.version` | ✅ Current semconv |
| `network.peer.address` | ✅ Current semconv |
| `network.peer.port` | ✅ Current semconv |
| `client.address` | ✅ Current semconv |
| `user_agent.original` | ✅ Current semconv |
| `entry_point` | ⚠️ Traefik-specific, not in OTel semconv |
| `http.request.body.size` | ⚠️ Not in stable OTel semconv |
| `client.port` | ⚠️ Not in stable OTel semconv |

No deprecated attributes (`http.method`, `http.status_code`, `http.target`, `http.url`) were found in traces. The schema URL `https://opentelemetry.io/schemas/1.40.0` is set on trace exports.

**Span name**: All spans are named `GET` (bare HTTP method). Per OTel HTTP semconv, the recommended span name is `{method} {http.route}` or `{method}` when route is unknown. Since Traefik knows the route (from `entry_point` and routing config), the span name `GET` is technically compliant but suboptimal — it lacks the route component that would make spans distinguishable.

**Missing**: `http.route` attribute is absent from all spans. This is a notable gap for an ingress controller that has full routing context.

##### Metric names and attributes

Two metric families coexist:

**OTel semconv metrics** (from `github.com/traefik/traefik` OTLP scope):
- `http.server.request.duration` — histogram with `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `url.scheme` ✅
- `http.client.request.duration` — histogram with same attributes ✅

**Traefik-proprietary metrics** (from both OTLP and Prometheus scrape):
- `traefik_entrypoint_requests_total` — uses `code`, `method`, `protocol`, `entrypoint` ❌ non-semconv
- `traefik_entrypoint_request_duration_seconds` — same non-semconv labels ❌
- `traefik_router_requests_total` — uses `code`, `method`, `protocol`, `router`, `service` ❌ non-semconv
- `traefik_service_requests_total` — uses `code`, `method`, `protocol`, `service` ❌ non-semconv

The `traefik_*` metrics use short-form labels (`code`, `method`, `protocol`) instead of OTel semconv (`http.response.status_code`, `http.request.method`, `network.protocol.name`). These metrics are present in both the OTLP push and Prometheus scrape paths — the non-semconv naming is not just a Prometheus legacy issue.

##### Log attributes

Access log records use PascalCase proprietary field names as log attributes:
- `ClientAddr`, `ClientHost`, `ClientPort` — not OTel semconv
- `DownstreamStatus`, `DownstreamContentSize` — not OTel semconv
- `RequestMethod`, `RequestPath`, `RequestProtocol` — not OTel semconv
- `RouterName`, `ServiceName`, `entryPointName` — traefik-specific
- `TraceId`, `SpanId` (PascalCase) AND `trace_id`, `span_id` (snake_case) — duplicated, non-standard naming

The log body is a JSON string (the full access log line), not a structured OTel log body. Log attributes mirror the JSON body fields — a redundant pattern.

Application logs have `severityNumber` set (9=INFO, 13=WARN) but `severityText` is `null` for all application log records. Only the single access log record has `severityText: "info"`.

#### Checklist assessment

- ✅ Current OTel HTTP semconv used in traces (no deprecated `http.method`, `http.status_code`)
- ✅ `http.server.request.duration` and `http.client.request.duration` use semconv labels
- ❌ `traefik_*` metrics use non-semconv labels (`code`, `method`, `protocol`)
- ❌ Log attributes use proprietary PascalCase names, not OTel log semconv
- ❌ `http.route` absent from spans (known routing context not exposed)
- ⚠️ `entry_point` is a traefik-domain concept not documented as a semconv extension
- ❌ `severityText` absent from application log records

#### Rationale

Traces are well-aligned with current OTel HTTP semconv. However, the `traefik_*` metric family (which covers the bulk of operational metrics) uses proprietary labels inconsistent with OTel semconv. Log attributes use a proprietary schema with no mapping to OTel conventions. The mix of semconv and non-semconv naming across signals is characteristic of Level 1: partial adoption with remaining inconsistencies.

---

### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes

Traefik emits a rich set of resource attributes natively via the OTel Go SDK:

- `service.name: traefik` — stable, consistent across all signals
- `service.version: 3.7.1` — present on traces, OTLP metrics, and logs
- `telemetry.sdk.*` — name, language, version all set correctly
- `process.*` — pid, executable name/path, owner, runtime name/version/description, command_args
- `os.*` — os.type, os.description

These attributes are present on traces, OTLP metrics, and logs — consistent across all three signal types.

**Gap**: `service.instance.id` is absent from traces and OTLP metrics. In Prometheus-scraped metrics, the collector sets `service.instance.id: traefik-metrics.traefik.svc.cluster.local:9100` (a scrape-target-derived value, not from Traefik itself). In a multi-replica deployment, this means traces from different Traefik instances cannot be distinguished by instance identity.

##### OTEL_* environment variable support

Traefik uses its own configuration flags (`--tracing.otlp.grpc.endpoint`, `--metrics.otlp.grpc.endpoint`, etc.) rather than standard `OTEL_EXPORTER_OTLP_ENDPOINT` environment variables. The OTel Go SDK is used internally, but the configuration surface is Traefik-specific. This is a design choice — the Helm chart provides a clean configuration path but it is not the standard `OTEL_*` variable approach.

##### Identity consistency across signals

`service.name: traefik` and `service.version: 3.7.1` are consistent across traces, OTLP metrics, and logs. The `telemetry.sdk.*` attributes are also consistent. The `host.name` (pod name) is present on all signals.

#### Checklist assessment

- ✅ `service.name` is stable and consistent across all signals
- ✅ `service.version` is present and consistent
- ✅ `telemetry.sdk.*` attributes correctly set
- ✅ Rich process and OS resource attributes emitted natively
- ❌ `service.instance.id` absent from traces and OTLP metrics
- ⚠️ Configuration uses Traefik-specific flags rather than `OTEL_*` env vars
- ✅ Identity is stable across restarts and scaling (pod name in `host.name`)

#### Rationale

The core identity attributes (`service.name`, `service.version`, `telemetry.sdk.*`) are correctly set natively and consistently across all signals. The project satisfies the Level 2 requirement for stable resource identity. The absence of `service.instance.id` on traces is a gap but not disqualifying for Level 2, as Kubernetes pod identity is available via `host.name` (which equals the pod name) and is enriched by k8sattributes. Configuration uses project-specific flags rather than `OTEL_*` vars, which is consistent with Level 2 (the project has a coherent configuration surface, just not the standard env var path).

---

### 4. Trace Modeling & Context Propagation

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span structure

All 235 spans in the traces data are:
- Kind: `SERVER` (kind=2) ✅
- Name: `GET` (bare HTTP method, no route) ⚠️
- Parent: none (all root spans) ⚠️

There are **no child spans** or **CLIENT spans** in the trace data. Traefik creates a SERVER span for each incoming request but does not create a CLIENT span for the backend call. This means:
- A request through Traefik produces a single span
- The backend service's spans (if any) are independent traces, not children of the Traefik span

This is architecturally significant: Traefik propagates the W3C traceparent header to the backend (confirmed in the INSTALL-PLAN notes), so the backend can continue the same trace. However, Traefik itself does not create a child CLIENT span for the upstream call. The trace tree is thus incomplete from Traefik's perspective — the "hop" through Traefik is visible as a SERVER span, but the backend call is not represented as a child.

##### Context propagation

W3C Trace Context propagation is confirmed:
- Traefik reads incoming `traceparent` headers
- Creates a new SERVER span with the incoming trace ID
- Injects updated `traceparent` into the upstream request
- The `entry_point` attribute on spans shows the named entrypoint (`traefik`, `web`, `metrics`)

##### Trace coherence

Traces are flat (single span per request). For an ingress controller, this is a reasonable design — the proxy itself is one hop. However, the absence of CLIENT spans for backend calls means there is no visibility into the backend call latency as seen by Traefik. The span name `GET` without route information makes it impossible to distinguish spans for different routes without inspecting `url.path`.

#### Checklist assessment

- ✅ W3C Trace Context propagation supported and working
- ✅ Entry-point spans are consistently SERVER spans (kind=2)
- ❌ No CLIENT spans for backend calls — the upstream call is not represented
- ❌ Span name is bare `GET` — `http.route` absent, making spans indistinguishable by route
- ✅ Context is propagated to backends (traceparent forwarded)
- ⚠️ Traces are flat — no parent-child relationships within Traefik's own trace tree

#### Rationale

Context propagation works correctly. SERVER spans are correctly modeled. However, the absence of CLIENT spans for backend calls and the bare `GET` span name without route context represent meaningful gaps. Users cannot see the Traefik→backend call latency in traces, and cannot distinguish spans for different routes without additional attribute inspection. This is characteristic of Level 1: "works for common synchronous flows" but with gaps in completeness.

---

### 5. Multi-Signal Observability

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Signal availability

All three signals are present via OTLP:
- **Traces**: Flowing via OTLP gRPC (native OTel Go SDK)
- **Metrics**: Flowing via OTLP gRPC push + Prometheus scrape
- **Logs**: Flowing via OTLP gRPC (experimental feature gate)

##### Cross-signal correlation

**Trace-log correlation** (access logs):
- One access log record in the dataset has `traceId` and `spanId` set in the OTLP log record fields ✅
- The same IDs appear as log attributes: both `trace_id` and `TraceId` (duplicated)
- This enables correlation between access log records and traces

**Application log-trace correlation**:
- Application logs (13 records) have no `traceId` or `spanId` — no correlation possible ❌

**Metric-trace correlation**:
- Metrics and traces share `service.name` and `service.version` resource attributes ✅
- The `http.server.request.duration` metric shares attribute keys with trace spans (`http.request.method`, `http.response.status_code`, `server.address`, `url.scheme`) ✅
- The `traefik_*` metrics use different label names (`code`, `method`) that do not align with trace span attributes ❌

##### Collection model

| Signal | Transport | Collection model |
|--------|-----------|-----------------|
| Traces | OTLP gRPC | Push from Traefik to collector |
| Metrics (OTLP) | OTLP gRPC | Push from Traefik to collector |
| Metrics (Prometheus) | Prometheus | Pull from collector to Traefik `/metrics` |
| Logs (general) | OTLP gRPC | Push from Traefik to collector |
| Logs (access) | OTLP gRPC | Push from Traefik to collector (batched, ~30s delay) |

#### Checklist assessment

- ✅ All three signals present via OTLP
- ✅ Access log records include trace context (traceId, spanId)
- ❌ Application logs do not include trace context
- ✅ `http.server.request.duration` metric shares attribute keys with trace spans
- ❌ `traefik_*` metrics use different attribute names than trace spans
- ❌ Log body is a JSON string, not structured — requires parsing for correlation
- ⚠️ Duplicate trace ID fields in logs (`trace_id` + `TraceId`) adds noise

#### Rationale

All three signals are present, which satisfies the minimum for Level 1. However, signals are not yet designed as a correlated system. Application logs lack trace context. The `traefik_*` metric labels don't align with trace attributes. The log body format (JSON string) requires parsing for investigation workflows. Users can correlate access logs with traces, but the overall experience is fragmented. This is Level 1: signals coexist but do not reinforce each other consistently.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming

All 235 spans are named `GET` — the bare HTTP method. This is the minimum compliant span name per OTel HTTP semconv (when `http.route` is not known), but Traefik has full routing context and could produce `GET /ping` or route-based names. The bare `GET` name means:
- All spans look identical in a trace list
- Users must inspect `url.path` to distinguish spans for different endpoints
- No route-level aggregation is possible from span names alone

The `entry_point` attribute (e.g., `entry_point=traefik`, `entry_point=web`, `entry_point=metrics`) adds useful context but is not reflected in the span name.

##### Signal-to-noise ratio

**Traces**: The `addInternals: true` flag causes Traefik to trace internal routes (ping, dashboard, metrics scraping). This adds significant noise — 55 spans for `entry_point=metrics` (Prometheus scraping) and 169 for `entry_point=traefik` (internal/dashboard) out of 225 total spans. Only 1 span for `entry_point=web` (actual user traffic) was observed. This ratio is inverted from what operators typically want.

**Logs**:
- Application logs have `severityNumber` set (9=INFO, 13=WARN) but `severityText` is `null` for all 13 records — a quality gap that may affect log routing in backends that rely on `severityText`
- Access log body is a JSON string — operators must parse the body to extract structured fields, even though the same fields are also present as log attributes
- Access log attributes include both `TraceId` and `trace_id` (duplicated), and `SpanId` and `span_id` — unnecessary duplication

**Metrics**: The dual metric families (`http.server.request.duration` with semconv labels vs `traefik_*` with proprietary labels) create confusion. Operators must know which family to use for OTel-native tooling vs Prometheus-native tooling.

##### Default usability

The `addInternals: true` default means internal traffic (healthchecks, dashboard, Prometheus scraping) is traced alongside user traffic. Without filtering, dashboards will show mostly internal traffic. This is a reasonable choice for debugging but adds cognitive overhead for operators.

#### Checklist assessment

- ⚠️ Span names are bare `GET` — technically compliant but lacking route context
- ❌ `http.route` absent — no route-level span aggregation possible
- ❌ `addInternals: true` causes internal traffic to dominate trace volume
- ❌ `severityText` null on application log records
- ❌ Access log body is JSON string, not structured
- ⚠️ Duplicate trace ID fields in access log attributes
- ✅ `entry_point` attribute adds useful operational context
- ✅ OTel semconv metrics (`http.server.request.duration`) are clean and usable

#### Rationale

Some effort is made toward user-oriented telemetry (semconv-aligned metrics, trace context in access logs, structured log fields). However, span names lack route context, internal traffic dominates trace volume, application log severity text is missing, and the access log body format requires parsing. Operators need domain knowledge to filter and interpret the data effectively. This is Level 1: improving but not yet ergonomic.

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

Traefik has dedicated documentation pages for OTel tracing, metrics, and logs at `doc.traefik.io`. The configuration options are documented. However, telemetry is not documented as a stability contract — there is no explicit statement about which telemetry attributes or metric names are stable.

##### Change communication

The Traefik changelog (`CHANGELOG.md`) includes telemetry-related entries with `[metrics,otel]`, `[tracing]`, `[logs,accesslogs]` tags. Examples from recent versions:
- `[metrics,otel]` Change request duration metric unit from millisecond to second (#11523) — a breaking change communicated in the changelog
- `[tracing]` Use ResourceAttributes instead of GlobalAttributes (#11515)
- `[logs,accesslogs]` OpenTelemetry Logs and Access Logs (#11319) — new feature
- `[middleware,metrics,tracing]` Upgrade to OpenTelemetry Semantic Conventions v1.26.0 (#10850)

Changes are tagged and mentioned in changelogs, but without migration guidance for users who depend on specific metric names or attribute values. The metric unit change (ms → seconds) is an example of a breaking change that was communicated but without a deprecation period.

##### Schema URL presence

Schema URL `https://opentelemetry.io/schemas/1.40.0` is set on trace and log exports. This is a positive signal — the project tracks its schema version.

##### Stability guarantees

OTLP log export is explicitly marked as experimental (`--experimental.otlpLogs=true`). There is no documented stability tier for traces or metrics. The instrumentation scope version is `vunknown` for logs — the scope version is not populated.

#### Checklist assessment

- ✅ Telemetry changes mentioned in changelogs with categorical tags
- ✅ Schema URL set on trace and log exports
- ❌ No explicit stability tiers for telemetry (stable vs experimental, except for logs)
- ❌ No migration guides for telemetry-specific changes
- ❌ Instrumentation scope version is `vunknown` for logs
- ⚠️ Breaking changes (metric unit change) communicated in changelog but without deprecation period
- ⚠️ OTLP log export still behind experimental feature gate

#### Rationale

Traefik communicates telemetry changes in changelogs with categorical tags, which is more than Level 0. However, there is no explicit stability contract, no migration guides for telemetry changes, and the experimental log feature gate signals ongoing instability. The changelog entries show awareness of downstream impact but informal governance. This is Level 1: some awareness, informal communication, no formal process.

---

## Key findings

### Strengths

- **Full OTLP coverage across all three signals**: Traefik v3.7 is one of the few CNCF projects that supports OTLP push for traces, metrics, and logs natively, without sidecars or adapters. The integration "just works" with a standard OTel Collector.
- **Rich native resource identity**: `service.name`, `service.version`, `telemetry.sdk.*`, `process.*`, and `os.*` are all emitted natively. Operators get meaningful service identity without any collector enrichment.
- **Current OTel HTTP semconv in traces**: Trace spans use current stable attributes (`http.request.method`, `http.response.status_code`, `url.path`, etc.) with no deprecated fields. The schema URL `https://opentelemetry.io/schemas/1.40.0` is set.
- **W3C Trace Context propagation**: Traefik correctly reads and propagates `traceparent` headers, enabling end-to-end tracing across the full request path even though Traefik itself doesn't create CLIENT spans.
- **Dual OTel semconv metrics**: `http.server.request.duration` and `http.client.request.duration` with correct semconv labels are emitted via OTLP alongside the `traefik_*` family.

### Areas for improvement

- **Add `http.route` to trace spans**: Traefik has full routing context (router name, matched route). Adding `http.route` to spans would enable route-level aggregation and make span names more meaningful (e.g., `GET /api/{id}` instead of `GET`).
- **Align `traefik_*` metric labels with OTel semconv**: The `traefik_*` metric family uses `code`, `method`, `protocol` instead of `http.response.status_code`, `http.request.method`, `network.protocol.name`. Migrating these labels (with a deprecation period) would enable consistent tooling across both metric families.
- **Fix `severityText` on application logs**: All 13 application log records have `severityText: null` despite `severityNumber` being set correctly. This is likely a bug in the OTLP log exporter — the severity text should be populated from the log level.
- **Add trace context to application logs**: Application logs (startup messages, provider events) have no `traceId` or `spanId`. Adding trace context where available would enable log-trace correlation beyond just access logs.
- **Stabilize OTLP log export**: Moving OTLP log export out of the experimental feature gate would signal stability to users. The feature appears production-ready based on the telemetry data observed.
- **Add `service.instance.id` to traces**: In multi-replica deployments, instance-level trace attribution requires `service.instance.id`. This could be set to the pod UID or hostname.

### Notable observations

- **`addInternals: true` inverts the traffic ratio**: With `addInternals: true`, 224 of 225 spans in the evaluation data came from internal traffic (Prometheus scraping, health checks, dashboard) and only 1 from actual user traffic. This is expected behavior but operators should be aware — filtering by `entry_point` is necessary to focus on user-facing traffic.
- **Duplicate trace ID fields in access logs**: Access log records include both `TraceId`/`SpanId` (PascalCase) and `trace_id`/`span_id` (snake_case) as separate attributes, plus the same values in the JSON body. This triplication suggests the OTLP access log exporter is serializing the same data multiple times through different code paths.
- **Prometheus scrape and OTLP push produce different metric sets**: The OTLP push includes `http.server.request.duration` (semconv) while Prometheus scrape does not. The `traefik_service_*` metrics appear in both. Operators using only Prometheus scraping miss the semconv-aligned metrics.
- **Instrumentation scope version**: The trace scope is `github.com/traefik/traefik vunknown` — the version field is not populated. The log scope is `traefik vunknown`. The metrics scope correctly shows `github.com/traefik/traefik v3.7.1`. Consistent scope versioning across signals would improve tooling compatibility.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector with file export in a local kind cluster (`otel-eval-traefik`)
- The k8sattributes processor was used to distinguish native vs enriched resource attributes
- Traefik v3.7.1 was installed via Helm chart `traefik/traefik` v40.2.0 with OTLP enabled for all signals
- Traffic was generated via port-forward to the Traefik web entrypoint; Prometheus scraping was also active
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv v1.26+)
- The Traefik changelog and documentation were reviewed for stability and change management context
- All telemetry evidence is from the JSONL files at `/tmp/otel-eval-traefik/`
