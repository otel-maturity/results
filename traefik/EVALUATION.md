# OpenTelemetry Support Maturity Evaluation: Traefik

## Project overview

- **Project**: Traefik — a CNCF graduated cloud-native HTTP reverse proxy and Kubernetes Ingress Controller that automatically discovers services and routes traffic to them
- **Version evaluated**: v3.6.15 (Helm chart 39.0.9)
- **Evaluation date**: 2026-05-06
- **Cluster**: otel-eval-traefik (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP is the primary path for all three signals; Prometheus retained as secondary |
| Semantic Conventions | 2 | Current OTel HTTP conventions used in traces; `traefik_*` metrics and access log attributes diverge |
| Resource Attributes & Configuration | 2 | Strong native identity (`service.name`, `service.version`, process/OS/host attributes); configuration via Traefik-specific flags, not `OTEL_*` env vars |
| Trace Modeling & Context Propagation | 2 | Coherent SERVER→CLIENT trace structure; W3C Trace Context propagated to upstreams; `entry_point` is a useful custom attribute |
| Multi-Signal Observability | 2 | All three signals flow via OTLP; access logs carry trace/span IDs; signals are functionally correlated |
| Audience & Signal Quality | 2 | Span names and attributes are user-oriented and operationally useful; access log attributes use PascalCase rather than OTel conventions |
| Stability & Change Management | 1 | OTel-related changes appear in changelogs; no explicit telemetry stability contract or deprecation policy; OTLP logs remain experimental |

---

## Telemetry overview

### Signals observed
- **Traces**: Flowing — OTLP gRPC push to collector at `:4317`
- **Metrics**: Flowing — OTLP gRPC push to collector at `:4317` (every 10s); Prometheus endpoint also available at `:9100/metrics`
- **Logs**: Flowing (experimental) — OTLP gRPC push; both general daemon logs and per-request access logs

### Resource attributes (native, before collector enrichment)

Traefik emits the following resource attributes natively via the embedded OTel Go SDK and its own resource detectors:

| Attribute | Example Value |
|-----------|--------------|
| `service.name` | `traefik` |
| `service.version` | `3.6.15` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.pid` | `1` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |
| `process.command_args` | `[traefik, --entryPoints.web.address=:8000/tcp, ...]` |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (...)` |
| `host.name` | `traefik-5999d49978-h8zpk` |

**Note on k8s attributes**: Traefik v3 includes a native Kubernetes resource detector (added in PR #11906) that sets `k8s.pod.uid`, `k8s.pod.name`, and `k8s.namespace.name` natively. The collector's `k8sattributes` processor then enriches further with additional metadata.

### Resource attributes (after collector enrichment)

The `k8sattributes` processor adds the following on top of Traefik's native attributes:

- `k8s.deployment.name`, `k8s.replicaset.name`, `k8s.node.name`, `k8s.container.name`
- `k8s.pod.start_time`
- `k8s.pod.label.*` (all pod labels, e.g., `k8s.pod.label.app.kubernetes.io/name`)
- `k8s.pod.annotation.*` (all pod annotations)

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **OTLP gRPC is the primary export path** for all three signals (traces, metrics, logs), configured via `tracing.otlp`, `metrics.otlp`, and `logs.*.otlp` Helm values.
- **Prometheus endpoint retained** at `:9100/metrics` as a secondary option — it is clearly positioned alongside OTLP, not as the default.
- **No adapters or sidecars required**: Traefik pushes directly to the OTel Collector at `otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317`.
- **OTLP logs are experimental** (`experimental.otlpLogs: true` required), which slightly limits the integration surface for logs, but traces and metrics are fully supported without experimental flags.
- **Configuration is Traefik-specific**: Users configure the OTLP endpoint via `--tracing.otlp.grpc.endpoint`, `--metrics.otlp.grpc.endpoint`, and `--log.otlp.grpc.endpoint` — not via `OTEL_EXPORTER_OTLP_ENDPOINT`. This is a project-specific configuration path rather than the standard OpenTelemetry SDK mechanism.
- **Documentation** covers OTel as the recommended observability path for v3, with dedicated pages for tracing, metrics, and logs via OpenTelemetry.

#### Checklist assessment

- ✅ OTLP export supported for traces, metrics, and logs
- ✅ Users can connect to an existing OTel Collector without adapters
- ✅ Prometheus scraping retained as a secondary/optional path
- ✅ Documentation positions OTLP as the recommended path
- ⚠️ Configuration uses project-specific flags, not `OTEL_*` environment variables
- ⚠️ OTLP log export requires `experimental.otlpLogs: true` feature flag
- ✅ Integration works cleanly with a standard OTel Collector pipeline

#### Rationale

Traefik v3 clearly positions OTLP as the primary integration surface for all three signals, and the integration works without glue code or custom adapters. The Prometheus endpoint remains available but is secondary. The use of project-specific configuration flags (rather than `OTEL_*` env vars) and the experimental status of OTLP logs are the main gaps from Level 3. Level 2 is appropriate: OpenTelemetry is the "happy path," but the integration surface is not yet fully documented as a stable contract with explicit change governance.

---

### 2. Semantic Conventions

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Trace attributes (Traefik-native spans)

Traefik's entry-point spans (`GET`, `POST`, etc., kind=SERVER) use **current, stable OTel HTTP semantic conventions**:

| Attribute | Value (example) | Convention Status |
|-----------|----------------|-------------------|
| `http.request.method` | `GET` | ✅ Current |
| `http.response.status_code` | `200` | ✅ Current |
| `url.path` | `/` | ✅ Current |
| `url.query` | `` | ✅ Current |
| `url.scheme` | `http` | ✅ Current |
| `url.full` | `http://10.244.0.6:3000/` | ✅ Current (on CLIENT spans) |
| `client.address` | `127.0.0.1` | ✅ Current |
| `client.port` | `50242` | ✅ Current |
| `network.peer.address` | `10.244.0.6` | ✅ Current |
| `network.peer.port` | `3000` | ✅ Current |
| `network.protocol.version` | `1.1` | ✅ Current |
| `user_agent.original` | `curl/8.5.0` | ✅ Current |
| `server.address` | `10.244.0.6` | ✅ Current |
| `server.port` | `3000` | ✅ Current |

**No deprecated attributes found** in Traefik's own spans. The deprecated attributes (`http.method`, `http.status_code`, `http.target`, `http.url`) that appear in the traces data come from the backend Node.js application (using an older OTel SDK), not from Traefik itself.

**Custom attribute**: `entry_point` (e.g., `web`, `traefik`) is a Traefik-specific attribute identifying which entry point handled the request. This is not an OTel semantic convention attribute, but it is operationally useful and documented in Traefik's configuration reference.

**Scope**: `github.com/traefik/traefik` (version not set — `vunknown`).

**Schema URL**: `https://opentelemetry.io/schemas/1.40.0` set on trace data — indicating deliberate alignment with a specific schema version.

##### Metric names and attributes

Traefik emits **two parallel sets of metrics**:

1. **OTel semantic convention metrics** (aligned with current conventions):
   - `http.server.request.duration` (histogram) — attributes: `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `url.scheme`, `server.address`, `server.port`, `error.type`
   - `http.client.request.duration` (histogram) — same attribute set

2. **Traefik-proprietary metrics** (non-OTel naming and attributes):
   - `traefik_entrypoint_requests_total`, `traefik_entrypoint_request_duration_seconds`, etc.
   - Attributes: `code`, `method`, `protocol`, `entrypoint`, `router`, `service` — these are **non-OTel attribute names** (e.g., `code` instead of `http.response.status_code`, `method` instead of `http.request.method`)
   - Schema URL: `https://opentelemetry.io/schemas/1.18.0` on the Traefik scope (older schema version)

The dual metric set is a positive sign of intentional OTel alignment, but the `traefik_*` metrics with non-OTel attribute names create inconsistency.

##### Log attributes

Access log records use **PascalCase Traefik-proprietary attribute names** (`RequestMethod`, `RequestPath`, `DownstreamStatus`, `RouterName`, `ServiceName`, `TraceId`, `SpanId`, etc.) rather than OTel semantic convention names. The trace/span IDs are present in both the OTLP record fields (as `traceId`/`spanId` hex strings) and as log attributes (`TraceId`, `SpanId`, `trace_id`, `span_id` — duplicated in multiple formats).

General logs lack `severityText` at the OTLP record level (it is `null`); only `severityNumber` is set (9 = INFO). Access logs have `severityText: "info"` but the log body contains `"level": "panic"` — a known bug in the experimental OTLP log implementation.

#### Checklist assessment

- ✅ Current OTel HTTP semantic conventions used in trace spans (no deprecated `http.method`, `http.status_code`, etc.)
- ✅ OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) emitted alongside proprietary metrics
- ✅ Schema URL set on trace and log data (`https://opentelemetry.io/schemas/1.40.0`)
- ⚠️ Traefik-proprietary metrics (`traefik_*`) use non-OTel attribute names (`code`, `method`, `entrypoint`)
- ⚠️ Access log attributes use PascalCase proprietary names, not OTel conventions
- ⚠️ Instrumentation scope version not set (`vunknown`)
- ⚠️ Access log severity mapping has a bug (`"panic"` in body, `info` in OTLP field)
- ✅ `entry_point` custom attribute is operationally meaningful (though undocumented as a semantic extension)

#### Rationale

Traefik's trace spans are a strong example of current OTel semantic convention alignment — no deprecated attributes, correct attribute scoping, and a schema URL. The dual metric approach (OTel semconv + proprietary) shows intentional alignment but creates inconsistency. Access logs use proprietary naming. The overall picture is Level 2: conventions are applied deliberately in traces and partially in metrics, but inconsistency remains across signals. The project has not reached Level 3 because the proprietary metric and log attribute schemas are not explicitly documented as semantic extensions with defined mappings to OTel conventions.

---

### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes

Traefik natively emits a rich set of resource attributes via its embedded OTel Go SDK:
- **Service identity**: `service.name=traefik`, `service.version=3.6.15` — stable and consistent across all three signals
- **SDK metadata**: `telemetry.sdk.name=opentelemetry`, `telemetry.sdk.language=go`, `telemetry.sdk.version=1.43.0`
- **Process attributes**: `process.executable.name`, `process.executable.path`, `process.pid`, `process.owner`, `process.runtime.*`, `process.command_args`
- **OS attributes**: `os.type=linux`, `os.description` (full Alpine Linux version string)
- **Host**: `host.name` (pod hostname)
- **Kubernetes** (via native k8s detector, PR #11906): `k8s.pod.uid`, `k8s.pod.name`, `k8s.namespace.name`

`service.name` and `service.version` are consistent across traces, metrics, and logs — identity is stable and predictable.

##### OTEL_* environment variable support

Traefik does **not** use `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, or `OTEL_RESOURCE_ATTRIBUTES` standard environment variables. All OTel configuration is done via Traefik-specific CLI flags or Helm values (e.g., `--tracing.otlp.grpc.endpoint`, `--metrics.otlp.grpc.endpoint`). This is a notable gap: users cannot configure Traefik's OTel export using the standard OTel SDK environment variable mechanism that works across the ecosystem.

##### Identity consistency across signals

| Attribute | Traces | Metrics | Logs |
|-----------|--------|---------|------|
| `service.name` | `traefik` ✅ | `traefik` ✅ | `traefik` ✅ |
| `service.version` | `3.6.15` ✅ | `3.6.15` ✅ | `3.6.15` ✅ |
| `telemetry.sdk.*` | ✅ | ✅ | ✅ |
| `process.*` | ✅ | ✅ | ✅ |
| `os.*` | ✅ | ✅ | ✅ |

Identity is fully consistent across all three signals.

#### Checklist assessment

- ✅ `service.name` and `service.version` set natively and consistently across all signals
- ✅ Identity is stable across restarts (tied to the binary version, not runtime state)
- ✅ Resource attributes placed correctly (not on spans)
- ✅ Rich process, OS, and host attributes emitted natively
- ✅ Kubernetes pod identity emitted natively (k8s.pod.uid, k8s.pod.name, k8s.namespace.name)
- ❌ `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES` not respected — configuration uses Traefik-specific flags only
- ⚠️ `service.instance.id` not set (no per-instance differentiation beyond `host.name`)

#### Rationale

Traefik's native resource attribute set is excellent: service identity, SDK metadata, process/OS/host details, and Kubernetes pod identity are all emitted natively and consistently across signals. The main gap is the absence of `OTEL_*` environment variable support — users cannot configure Traefik's telemetry using the standard OTel SDK mechanism. This is a meaningful friction point for operators who manage observability configuration uniformly across services. The level is 2 rather than 3 because the configuration behavior is not documented as a stable contract, and `OTEL_*` env vars are not respected.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

Traefik produces a clean two-span structure per routed HTTP request:

1. **Root span**: `GET` (or `POST`, etc.) — kind=`SERVER` (kind=2)
   - Attributes: `entry_point`, `http.request.method`, `url.path`, `url.query`, `url.scheme`, `user_agent.original`, `client.address`, `client.port`, `network.peer.address`, `network.peer.port`, `network.protocol.version`, `http.response.status_code`, `server.address`, `http.request.body.size`
   - Optionally: captured request headers as `http.request.header.*` attributes

2. **Child span**: `ReverseProxy` — kind=`CLIENT` (kind=3)
   - Attributes: `http.request.method`, `url.full`, `url.scheme`, `user_agent.original`, `network.peer.address`, `network.peer.port`, `server.address`, `server.port`, `http.response.status_code`, captured headers
   - `parentSpanId` set to the root span's ID — correct parent-child relationship

Span kinds are correct: `SERVER` for the entry point, `CLIENT` for the upstream proxy call.

Span names (`GET`, `POST`) follow the OTel HTTP semantic convention for server spans (method-based naming). The changelog confirms this was an intentional decision: "[tracing] Follow OTel semantic conventions for root span naming" (PR #11673).

##### Context propagation

W3C Trace Context propagation is confirmed working:
- When an incoming request includes a `traceparent` header, Traefik continues the trace (uses the same trace ID, creates a child span)
- When no `traceparent` is present, Traefik starts a new trace and injects `traceparent` into the upstream request
- Full distributed trace topology confirmed: `GET (traefik, SERVER)` → `ReverseProxy (traefik, CLIENT)` → `GET / (otel-eval-backend, SERVER)`
- The `traceparent` header itself appears as a captured span attribute (`http.request.header.traceparent`) when configured

##### Trace coherence

- Traces tell a complete story: entry-point span → proxy span → backend span
- Internal health check requests (ping endpoint) produce their own traces when `addInternals: true` is set
- Error spans are correctly modeled with `status.code=2` (ERROR) for failed requests
- No fragmented or disconnected spans observed in the test data

#### Checklist assessment

- ✅ W3C Trace Context (`traceparent`/`tracestate`) propagated to upstream services
- ✅ SERVER spans for entry points, CLIENT spans for upstream proxy calls
- ✅ Correct parent-child relationships between entry-point and proxy spans
- ✅ Span names follow OTel HTTP semantic conventions (method-based)
- ✅ Distributed traces confirmed end-to-end (Traefik → backend)
- ✅ Error status correctly set on failed spans
- ⚠️ No span links used (not needed for this synchronous proxy use case)
- ⚠️ Trace behavior for async/background work (middleware chains, retries) not explicitly documented

#### Rationale

Traefik's trace modeling is intentional and well-executed for its core use case as a reverse proxy. The SERVER→CLIENT span structure is correct, W3C propagation works end-to-end, and span names follow OTel conventions. The `entry_point` attribute is a useful domain-specific addition. Trace behavior is stable and predictable. This is solidly Level 2. Level 3 would require explicit architectural documentation of trace modeling decisions and validation/testing of trace behavior over time.

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability

All three signals are first-class OTLP outputs in Traefik v3:
- **Traces**: Fully supported, stable
- **Metrics**: Fully supported, stable (both `traefik_*` and OTel semconv metrics)
- **Logs**: Supported via OTLP but **experimental** (requires `experimental.otlpLogs: true`)

##### Cross-signal correlation

Access logs include trace and span context at the **OTLP record level** (`traceId` and `spanId` fields), confirmed in the telemetry data:

```json
{
  "traceId": "474aa57b3d0f20dc2934a8623888f0f1",
  "spanId": "9d93be3ed31f6340",
  "severityText": "info",
  "body": "{\"ClientAddr\":\"127.0.0.1:50242\", ..., \"TraceId\":\"474aa57b3d0f20dc2934a8623888f0f1\", \"SpanId\":\"9d93be3ed31f6340\", ...}"
}
```

The trace/span IDs appear both in the OTLP record fields (enabling native log-to-trace linking in backends like Grafana) and as log attributes (`TraceId`, `SpanId`, `trace_id`, `span_id`). This enables users to pivot from access logs to traces without manual correlation.

Metrics share the same `service.name=traefik` resource identity as traces and logs, enabling filtering by service across all signals.

##### Collection model

| Signal | Method | Protocol | Push/Pull |
|--------|--------|----------|-----------|
| Traces | OTLP | gRPC/4317 | Push |
| Metrics | OTLP | gRPC/4317 | Push (every 10s) |
| Metrics | Prometheus | HTTP/:9100 | Pull (secondary) |
| Logs | OTLP | gRPC/4317 | Push |

All signals use a consistent OTLP push model, which simplifies pipeline configuration.

#### Checklist assessment

- ✅ All three signals emitted via OTLP
- ✅ Access logs include `traceId` and `spanId` at the OTLP record level — native log-to-trace correlation
- ✅ Consistent `service.name` and `service.version` across all signals
- ✅ Metrics and traces complement each other (aggregate counts/latencies vs. per-request detail)
- ⚠️ OTLP log export is experimental — not yet a stable first-class signal
- ⚠️ General daemon logs do not include trace context (only access logs do)
- ⚠️ Metric attribute names (`code`, `method`) differ from trace attribute names (`http.response.status_code`, `http.request.method`) for the `traefik_*` metric family — cross-signal correlation requires mapping

#### Rationale

Traefik achieves Level 2 multi-signal observability: all three signals are present and functionally correlated. Access logs include trace context at the OTLP record level, enabling direct log-to-trace linking. The experimental status of OTLP logs and the attribute naming inconsistency between `traefik_*` metrics and traces prevent a Level 3 rating. Level 3 would require explicit signal design documentation, cardinality management guidance, and resolution of the attribute naming inconsistency.

---

### 6. Audience & Signal Quality

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span naming

Traefik uses HTTP method names (`GET`, `POST`) as span names for entry-point spans, following the OTel HTTP semantic convention for server spans. This is user-oriented: operators can immediately understand what type of request was handled. The `ReverseProxy` span name clearly describes the operation.

The changelog entry "[tracing] Follow OTel semantic conventions for root span naming" (PR #11673) confirms this was an intentional quality improvement.

##### Signal-to-noise ratio

- **Traces**: Configurable verbosity. Default mode (`minimal`) produces two spans per request (entry-point + proxy). `addInternals: true` adds internal health check traces. The `detailed` verbosity level adds middleware spans. Configurable sampling rate (`sampleRate`). This is well-designed: operators can choose their verbosity level.
- **Metrics**: Rich set covering entry points, routers, and services. Both aggregate (`traefik_*`) and OTel semconv (`http.server.request.duration`) metrics are emitted — some redundancy, but the OTel semconv metrics are more useful for cross-project tooling.
- **Logs**: General logs are INFO-level by default and focused on configuration events. Access logs are per-request and include all relevant operational fields. The experimental severity bug (`"panic"` in body) is a quality issue.

##### Default usability

- Entry-point spans include all information needed to understand a request: method, path, status code, client address, user agent, and the upstream backend called.
- The `entry_point` attribute identifies which Traefik entrypoint handled the request — useful for multi-entrypoint deployments.
- Access logs include router name, service name, and upstream address — operators can understand routing decisions from logs alone.
- Metrics cover the full request lifecycle at three levels of granularity (entry point, router, service).

#### Checklist assessment

- ✅ Span names follow OTel conventions (HTTP method-based)
- ✅ Span attributes provide complete operational context without internal implementation details
- ✅ Configurable trace verbosity (minimal/detailed) — sensible defaults
- ✅ Metrics cover operationally relevant dimensions (entry point, router, service, latency, bytes)
- ✅ Access logs include trace context for correlation
- ⚠️ Access log attribute names use PascalCase (`RequestMethod`, `DownstreamStatus`) rather than OTel conventions — requires mapping for generic tooling
- ⚠️ Access log severity bug: `severityText="info"` at OTLP level but body contains `"level":"panic"` — incorrect severity signal
- ⚠️ General logs lack `severityText` at the OTLP record level (only `severityNumber` set)
- ⚠️ Duplicate trace/span ID fields in access logs (`TraceId`, `SpanId`, `trace_id`, `span_id`) — redundancy without clear purpose

#### Rationale

Traefik's telemetry is clearly designed with operators in mind. Span names are logical, attributes are operationally relevant, and configurable verbosity gives operators control over signal volume. The access log attribute naming and severity issues are the main quality gaps — they reduce the out-of-the-box usability of log data in generic OTel tooling. Overall this is Level 2: telemetry is intentionally designed for its audience with sensible defaults, but some signals still require project-specific interpretation knowledge.

---

### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Documentation of telemetry behavior

Traefik's official documentation covers OTel configuration for traces, metrics, and logs with dedicated pages. However, the documentation describes *how to configure* OTel, not *what telemetry is emitted* or *what stability guarantees apply*. There is no explicit telemetry contract, no list of stable vs. experimental telemetry, and no schema documentation for span attributes or metric dimensions.

##### Change communication

The Traefik changelog includes OTel-related entries tagged with `[otel]`, `[tracing]`, `[metrics]`, `[logs]`, and `[accesslogs]`. Recent examples from the evaluated version range:

- `[logs, otel]` Add OTel-conformant trace context attributes to access logs (#12801)
- `[otel]` Update OpenTelemetry to v1.38.0 and semantic conventions to v1.37.0 (#12099)
- `[tracing]` Follow OTel semantic conventions for root span naming (#11673)
- `[metrics,otel]` Rename `traefik_tls_certs_not_after_milliseconds` to `traefik_tls_certs_not_after_seconds` (#12141)
- `[logs,tracing,k8s,otel]` Add k8s resource attributes automatically (#11906)

Changes are mentioned in release notes, but without explicit "breaking change" flags for telemetry changes or migration guidance. The metric rename (`traefik_tls_certs_not_after_milliseconds` → `traefik_tls_certs_not_after_seconds`) is a breaking change for users with dashboards or alerts on that metric — it is listed in the changelog but without migration guidance.

##### Schema URL presence

Schema URL is set on trace data (`https://opentelemetry.io/schemas/1.40.0`) and log data (`https://opentelemetry.io/schemas/1.40.0`). The `traefik_*` metric scope uses an older schema URL (`https://opentelemetry.io/schemas/1.18.0`), suggesting the proprietary metrics were not updated alongside the semantic convention metrics.

##### Stability guarantees

- **OTLP logs** are explicitly marked as **experimental** (`experimental.otlpLogs: true`), which is a positive sign of stability awareness — the project distinguishes experimental from stable features.
- **OTLP traces and metrics** have no explicit stability label but have been available since Traefik v3.0.
- No formal policy for telemetry evolution or deprecation timelines exists in the documentation.

#### Checklist assessment

- ✅ OTel-related changes appear in changelogs with relevant tags
- ✅ Experimental features explicitly labeled (`experimental.otlpLogs`)
- ✅ Schema URL set on trace and log data
- ⚠️ No explicit telemetry stability contract or documentation
- ⚠️ Breaking telemetry changes (e.g., metric renames) listed in changelog but without migration guidance
- ⚠️ No distinction between stable and experimental telemetry beyond the `experimental.otlpLogs` flag
- ❌ No defined review process for telemetry changes
- ❌ No deprecation timelines or migration paths for telemetry changes

#### Rationale

Traefik is at Level 1 for stability and change management. The project is aware that OTel changes have impact — they tag changelog entries and label experimental features. However, there is no formal telemetry stability contract, no explicit policy for what constitutes a breaking telemetry change, and no migration guidance when telemetry changes. Users who depend on specific metric names or span attributes must monitor the changelog carefully. Level 2 would require telemetry changes to be documented as breaking changes with migration guidance, and a clear distinction between stable and experimental telemetry.

---

## Key findings

### Strengths

- **Excellent trace semantic convention alignment**: Traefik's entry-point spans use current, stable OTel HTTP conventions (`http.request.method`, `http.response.status_code`, `url.path`, etc.) with no deprecated attributes. Schema URL is set.
- **Rich native resource attributes**: `service.name`, `service.version`, process/OS/host attributes, and Kubernetes pod identity are all emitted natively and consistently across all three signals — no collector enrichment needed for service identity.
- **Full distributed tracing with W3C context propagation**: The complete trace chain (Traefik entry-point → proxy → backend) works out of the box, with correct span kinds and parent-child relationships.
- **Access logs carry trace context at the OTLP record level**: `traceId` and `spanId` are set on log records, enabling native log-to-trace correlation in backends like Grafana without additional processing.
- **Dual metric strategy**: Traefik emits both OTel semantic convention metrics (`http.server.request.duration`) and proprietary `traefik_*` metrics — showing active alignment with OTel conventions while maintaining backward compatibility.

### Areas for improvement

- **Adopt `OTEL_*` environment variable support**: Traefik should respect `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, and `OTEL_RESOURCE_ATTRIBUTES` alongside its own configuration flags, enabling standard OTel SDK configuration patterns.
- **Align access log attribute names with OTel conventions**: Replace PascalCase proprietary names (`RequestMethod`, `DownstreamStatus`, `RouterName`) with OTel semantic convention names (`http.request.method`, `http.response.status_code`) or explicitly document the proprietary schema as a semantic extension.
- **Fix access log severity mapping**: The `"level": "panic"` in the log body for access logs is a known bug in the experimental OTLP log implementation. Access logs should use the correct severity level (typically INFO for successful requests, ERROR for 5xx).
- **Graduate OTLP logs from experimental**: The OTLP log export feature works reliably in v3.6.15. Removing the `experimental.otlpLogs` requirement would make logs a first-class signal.
- **Establish a telemetry stability contract**: Document which telemetry is stable, explicitly flag breaking telemetry changes in release notes, and provide migration guidance when metric names or span attributes change.

### Notable observations

- **Traefik has a native Kubernetes resource detector** (added in PR #11906) that sets `k8s.pod.uid`, `k8s.pod.name`, and `k8s.namespace.name` natively — this is unusual and impressive for an ingress controller. Most projects rely entirely on collector enrichment for k8s attributes.
- **The `traefik_*` metrics use an older schema URL** (`https://opentelemetry.io/schemas/1.18.0`) while the OTel semconv metrics and traces use `1.40.0` — suggesting the proprietary metrics were not updated when the schema was bumped.
- **Instrumentation scope version is not set** (`vunknown`) for the `github.com/traefik/traefik` scope — this makes it impossible to correlate telemetry with a specific Traefik version from the scope metadata alone (though `service.version` in the resource serves this purpose).
- **The `process.command_args` resource attribute includes the full Traefik command line**, which exposes the OTel endpoint configuration in the telemetry data. This is technically correct (it's process metadata) but may expose internal configuration details in shared observability backends.
- **Traefik actively tracks OTel semantic convention updates**: The changelog shows regular bumps of `go.opentelemetry.io/otel` and semantic convention versions, indicating active maintenance of OTel alignment.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector (contrib v0.150.1) with file export in a local kind cluster (`otel-eval-traefik`)
- The `k8sattributes` processor was used to enrich telemetry with Kubernetes metadata; native vs. enriched attributes were distinguished based on Traefik's source code (PR #11906) and the collector configuration
- Semantic conventions were checked against the latest stable OpenTelemetry specification (HTTP semconv v1.40.0)
- Documentation was reviewed at `https://doc.traefik.io/traefik/` and source code was inspected via the GitHub API
- The Traefik changelog was reviewed for OTel-related changes in the v3.x series
- Traffic was generated via `curl` to exercise HTTP request traces, and the cluster was observed for metric push batches and log records
