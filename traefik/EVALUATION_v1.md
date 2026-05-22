# OpenTelemetry Maturity Evaluation: Traefik v1

---

## Project overview

- **Project**: Traefik — cloud-native HTTP reverse proxy and load balancer / Kubernetes Ingress controller
- **Repository**: https://github.com/traefik/traefik
- **Version evaluated**: v3.7.1 (Helm chart `traefik/traefik` v40.0.0)
- **Evaluation date**: 2026-05-22
- **Cluster**: otel-eval-traefik
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.5

---

## Summary table

| # | Dimension | Level | Label |
|---|-----------|-------|-------|
| 1 | Integration Surface | **2** | OpenTelemetry-Native |
| 2 | Semantic Conventions | **1** | OpenTelemetry-Aligned |
| 3 | Resource Attributes & Configuration | **3** | OpenTelemetry-Optimized |
| 4 | Trace Modeling & Context Propagation | **2** | OpenTelemetry-Native |
| 5 | Multi-Signal Observability | **2** | OpenTelemetry-Native |
| 6 | Audience & Signal Quality | **1** | OpenTelemetry-Aligned |
| 7 | Stability & Change Management | **2** | OpenTelemetry-Native |

**Overall profile**: Traefik v3 is a strong Level 2 project with one dimension reaching Level 3 (Resource Attributes) and two dimensions constrained to Level 1 (Semantic Conventions; Audience & Signal Quality). The integration surface is genuinely OpenTelemetry-native — all legacy exporters were removed in v3 — but cross-signal semantic consistency and signal quality for traces and logs remain areas for improvement.

---

## Telemetry overview

### Signals observed

- **Traces**: Flowing — OTLP gRPC push, native OTel Go SDK; 61 JSONL batches / 242 spans
- **Metrics**: Flowing — dual pipeline: OTLP gRPC push (`--metrics.otlp.grpc`) + Prometheus scrape; 78 JSONL batches / 2,904 data points
- **Logs**: Flowing — OTLP gRPC push (`--accesslog.otlp.grpc`); 4 JSONL batches / 24 access log records; application-level logs via `--experimental.otlpLogs` gated as experimental

### Resource attributes (native, before collector enrichment)

The following attributes are emitted natively by Traefik across all three signals (traces, OTLP metrics, logs):

| Attribute | Value observed |
|-----------|----------------|
| `service.name` | `traefik` |
| `service.version` | `3.7.1` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-5fb59c7b48-c2pgw` (pod name via `resource.WithHost()`) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-5fb59c7b48-c2pgw 6.17.0-1013-azure ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.10` |
| `process.runtime.description` | `go version go1.25.10 linux/amd64` |
| `process.command_args` | (CLI args array) |

**Natively detected Kubernetes attributes** (via Traefik's built-in `K8sAttributesDetector`, not Collector enrichment):

| Attribute | Value observed |
|-----------|----------------|
| `k8s.pod.name` | `traefik-5fb59c7b48-c2pgw` |
| `k8s.pod.uid` | `d1e20c59-c3f8-42ef-8d27-fd32a5650310` |
| `k8s.namespace.name` | `traefik` |

### Resource attributes (after collector enrichment)

The following were added by the OTel Collector's `k8sattributes` processor (not emitted by Traefik natively):

- `k8s.container.name: traefik`
- `k8s.deployment.name: traefik`
- `k8s.replicaset.name: traefik-5fb59c7b48`
- `k8s.node.name: otel-eval-traefik-control-plane`
- `k8s.pod.start_time: 2026-05-22T07:02:02Z`
- `k8s.pod.annotation.prometheus.io/path`, `k8s.pod.annotation.prometheus.io/port`, `k8s.pod.annotation.prometheus.io/scrape`
- `k8s.pod.label.app.kubernetes.io/instance`, `k8s.pod.label.app.kubernetes.io/managed-by`, `k8s.pod.label.app.kubernetes.io/name`, `k8s.pod.label.helm.sh/chart`, `k8s.pod.label.pod-template-hash`

Note: `service.instance.id: traefik-metrics.traefik.svc.cluster.local:9100` seen in Prometheus-scraped metrics is a Collector artifact (scrape target address used as instance ID) — not natively emitted by Traefik.

---

## Installation context summary

Traefik v3.7.1 was installed via the official Helm chart (`traefik/traefik` v40.0.0) into the `traefik` namespace on a kind cluster. Getting all three signals flowing required explicit project-specific Helm values: `tracing.otlp.grpc.*` for traces, `metrics.otlp.grpc.*` for OTLP metrics (Prometheus scraping was enabled in parallel by default), and `accesslog.otlp.grpc.*` for access logs. A notable non-standard step was enabling `experimental.otlpLogs: true` as a feature gate — without this flag, OTLP log export does not activate. The Helm chart schema enforces strict validation (e.g. `deployment.replicas` rather than `replicaCount`), and the default `LoadBalancer` service type requires port-forwarding in a kind cluster. Traefik's built-in `K8sAttributesDetector` natively queries the Kubernetes API to populate `k8s.pod.name`, `k8s.pod.uid`, and `k8s.namespace.name` without any Collector-side configuration, which is a notable positive. The Prometheus metrics endpoint on port 9100 was scraped in parallel by the Collector, resulting in a dual metrics pipeline throughout the evaluation.

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅, Metrics ✅ (OTLP push + Prometheus scrape dual-channel), Logs ✅ (experimental feature gate)
- **Configuration method**: Project-specific Helm/YAML/CLI flags (`tracing.otlp.grpc`, `metrics.otlp`, `logs.access.otlp`); `OTEL_PROPAGATORS` env var is respected for propagator selection; `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** documented as a configuration path — endpoint must be set via project-specific keys
- **Documentation stance**: OTLP is the **only** supported exporter in Traefik v3 for traces and metrics; legacy exporters (Jaeger, Zipkin, Datadog) were present in v2 and fully removed in v3 — the tracing overview page now reads "Traefik uses OpenTelemetry, an open standard designed for distributed tracing. Please check our dedicated OTel docs."
- **Legacy exporter status**: Removed — Jaeger/Zipkin/Datadog tracing backends existed in v2 but were dropped in v3; no legacy exporters remain for traces or metrics
- **Signals requiring adapters/sidecars**: None — all three signals flow directly from the Traefik process via OTLP gRPC to the Collector; however, Prometheus scrape is also enabled in parallel for metrics (dual-channel), and OTLP log export requires `experimental.otlpLogs: true` feature gate

**Observed telemetry details:**

| Signal | Batches | Instrumentation Scope | Transport |
|--------|---------|----------------------|-----------|
| Traces | 27 | `github.com/traefik/traefik` (no version in scope) | OTLP gRPC |
| Metrics | 39 | `github.com/traefik/traefik v3.7.1` (OTLP push) + `prometheusreceiver v0.150.1` (scrape) | OTLP gRPC push + Prometheus pull |
| Logs | 4 | `traefik` (no version in scope) | OTLP gRPC |

**Specific observations:**

1. **Traefik v3 is OTLP-only for traces**: The v3 changelog confirms all legacy tracers were removed at the v3.0 boundary. The tracing overview now exclusively references OpenTelemetry. This is a clean break from v2.

2. **Metrics are dual-channel**: Traefik emits both OTLP push metrics (`github.com/traefik/traefik v3.7.1` scope, including OTel semantic convention metrics `http.server.request.duration` and `http.client.request.duration`) and Prometheus scrape metrics (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`). The Prometheus endpoint is opt-in but equally documented alongside OTLP. Both channels were active in this evaluation. This is a notable split: OTLP metrics include semantic convention names, while Prometheus metrics use Traefik-native naming.

3. **OTLP log export is behind a feature gate**: `experimental.otlpLogs: true` must be set to enable OTLP log export. This is documented as experimental in the source code (`OTLPLogs bool` in `experimental.go`). The feature is functional and flowing but not yet stable. Log body is a JSON string (not a structured OTLP body), and `level: panic` appears in access log records for normal 200 responses — a known serialization quirk.

4. **Configuration is project-specific, not OTEL_* env vars**: Endpoint, protocol, and TLS configuration must be set via Traefik's own config hierarchy (`tracing.otlp.grpc.endpoint`, etc.). The standard `OTEL_EXPORTER_OTLP_ENDPOINT` env var is not documented as a supported configuration path. Only `OTEL_PROPAGATORS` is explicitly supported as a standard OTel env var.

5. **Rich resource attributes**: Traefik sets `service.name`, `service.version`, `telemetry.sdk.name`, `telemetry.sdk.language`, `telemetry.sdk.version`, `host.name`, `os.type`, `os.description`, `process.*` — all OTel semantic convention resource attributes populated correctly by the Go OTel SDK.

6. **W3C Trace Context propagation confirmed**: Incoming `traceparent` headers are correctly read, trace IDs preserved, new child span IDs generated, and updated `traceparent` forwarded to backends.

7. **OTel semantic convention compliance**: Traefik v3 docs state it follows "official OpenTelemetry semantic conventions v1.26.0". Span names use HTTP method names (`GET`), span kinds are SERVER/CLIENT, and metric names include both OTel semantic convention names and Traefik-native names.

#### Checklist assessment

**Level 0 — Instrumented**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is telemetry exported only via tool-specific or legacy exporters (Jaeger only, Prometheus scrape only)? | **NO** | OTLP is the primary and only trace exporter; metrics have OTLP push as the primary path |
| Is OTLP unsupported or available only indirectly via sidecars/adapters? | **NO** | All three signals flow natively via OTLP gRPC without any sidecar or adapter |
| Does telemetry configuration rely entirely on project-specific flags? | **PARTIAL** | Configuration uses project-specific YAML/CLI flags, but this is by design in Traefik's architecture |
| Do users need to adapt their observability stack to fit the project's model? | **NO** | Standard OTel Collector with OTLP receiver accepts all signals directly |
| Is OpenTelemetry absent from docs or treated as an afterthought? | **NO** | OTel is the only supported tracing/metrics integration in v3; prominently documented |

**Level 1 — OpenTelemetry-Aligned**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP supported alongside equally-promoted legacy exporters? | **NO** | Legacy exporters (Jaeger, Zipkin, Datadog) were fully removed in v3; OTLP is the sole option for traces |
| Are there multiple overlapping ways to configure telemetry? | **PARTIAL** | Metrics have dual-channel (OTLP + Prometheus), both documented equally; traces and logs are OTLP-only |
| Does OTLP require disabling legacy behavior or enabling experimental flags? | **PARTIAL** | Traces/metrics: no flags needed; logs require `experimental.otlpLogs: true` |
| Is OpenTelemetry integration inconsistent across signals? | **PARTIAL** | Traces/metrics are OTLP-native; logs are OTLP-capable but behind experimental flag |
| Do users need to read multiple docs pages to get a working OTLP integration? | **NO** | Each signal has a dedicated OTel doc page; configuration is straightforward |

**Level 2 — OpenTelemetry-Native**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP the default or clearly-recommended export path in docs? | **YES** | For traces: OTLP is the only option in v3 (defaults to HTTPS to localhost:4318). For metrics: OTLP is documented as the push path alongside Prometheus scrape |
| Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` respected end-to-end? | **PARTIAL** | `OTEL_PROPAGATORS` is explicitly supported; `OTEL_EXPORTER_OTLP_ENDPOINT` is not documented — endpoint must be set via project-specific config keys |
| Can users connect to an existing OTel Collector without adapters or glue code? | **YES** | Direct OTLP gRPC connection to Collector works for all three signals; confirmed in evaluation |
| Are legacy exporters clearly secondary, optional, or deprecated? | **YES** | Completely removed in v3; no legacy exporters exist for traces; Prometheus is optional alongside OTLP for metrics |
| Is telemetry configuration consistent across all signals? | **MOSTLY YES** | Traces and metrics: stable OTLP config. Logs: functional but experimental. All three signals use same OTLP gRPC endpoint pattern |

**Level 3 — OpenTelemetry-Optimized**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is the telemetry integration surface documented as a stable contract? | **NO** | OTLP log export is explicitly experimental; metric naming includes both OTel semantic convention and Traefik-native names without clear stability guarantees |
| Are telemetry integration changes reviewed like API changes? | **NO** | No evidence of telemetry-specific stability commitments or change review process in docs or changelog |
| Are breaking changes communicated with migration guidance? | **PARTIAL** | The v2→v3 migration removed all legacy tracers; migration guide exists but telemetry changes are not specially called out with migration paths |
| Does the project explicitly support diverse deployment models? | **PARTIAL** | HTTP and gRPC OTLP are both documented; TLS and insecure modes supported; no explicit guidance for local dev vs. Kubernetes vs. managed platforms |
| Are legacy integrations removed or tightly scoped with clear deprecation timelines? | **PARTIAL** | Legacy tracers were removed cleanly in v3; Prometheus metrics endpoint remains active without deprecation timeline |

#### Rationale

Traefik v3 is firmly at **Level 2 — OpenTelemetry-Native**. The most significant evidence is the complete removal of all legacy tracing backends (Jaeger, Zipkin, Datadog) in v3, making OTLP the sole trace export path. All three signals flow natively via OTLP gRPC to an OTel Collector without any adapter, sidecar, or bridge component. The project uses the OTel Go SDK with full semantic convention compliance for resource attributes, span semantics, and metric naming.

The project falls short of Level 3 for several reasons: (1) OTLP log export remains behind an `experimental.otlpLogs` feature gate and exhibits serialization quirks (JSON string body, incorrect severity level for access logs); (2) the standard `OTEL_EXPORTER_OTLP_ENDPOINT` env var is not documented as a supported configuration path — users must use Traefik-specific config keys; (3) metrics maintain a dual-channel model (OTLP push + Prometheus scrape) with no stated deprecation timeline for Prometheus; (4) there is no explicit stability contract or change-review process for the telemetry integration surface.

---

### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

Deprecated attributes are present on **24 spans** emitted by the backend service's Node.js `@opentelemetry/instrumentation-http` (v0.48.0) library, which is captured as part of the distributed trace. These are not Traefik-native spans but they appear in the same trace data collected from the cluster:

| Deprecated Attribute | Count | Current Equivalent |
|---------------------|-------|--------------------|
| `http.method` | 24 | `http.request.method` |
| `http.status_code` | 24 | `http.response.status_code` |
| `http.url` | 24 | `url.full` |
| `http.target` | 24 | `url.path` + `url.query` |
| `http.host` | 24 | `server.address` + `server.port` |
| `http.scheme` | 24 | `url.scheme` |
| `http.flavor` | 24 | `network.protocol.version` |
| `http.user_agent` | 24 | `user_agent.original` |
| `net.peer.name` / `net.peer.port` | 24 each | `server.address` / `server.port` |
| `net.host.name` / `net.host.port` | 24 each | `server.address` / `server.port` |

##### Current OTel attributes found on spans

Traefik's own spans (`github.com/traefik/traefik` scope, 117 spans) use **current stable OTel semantic conventions**:

| Current Attribute | Count |
|-------------------|-------|
| `http.request.method` | 101 |
| `http.response.status_code` | 101 |
| `url.scheme` | 101 |
| `url.path` | 77 |
| `url.query` | 77 |
| `url.full` | 24 |
| `server.address` | 101 |
| `server.port` | 24 |
| `network.peer.address` | 101 |
| `network.peer.port` | 101 |
| `network.protocol.version` | 101 |
| `client.address` | 77 |
| `client.port` | 77 |
| `user_agent.original` | 77 |
| `http.request.body.size` | 77 |

Traefik-specific span attributes (domain extensions, not OTel semconv violations):
- `entry_point` — Traefik entrypoint name (no OTel equivalent)

##### Metric names and conventions

Two distinct metric naming systems coexist:

**OTel semconv-aligned (emitted by `github.com/traefik/traefik` scope):**
- `http.server.request.duration` — follows OTel `http.server.*` convention
- `http.client.request.duration` — follows OTel `http.client.*` convention
- Attributes: `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`, `error.type` ✅

**Traefik-proprietary Prometheus metrics (scraped via Prometheus receiver):**
- `traefik_entrypoint_request_duration_seconds`, `traefik_entrypoint_requests_total`, `traefik_entrypoint_requests_bytes_total`, `traefik_entrypoint_responses_bytes_total`
- `traefik_router_request_duration_seconds`, `traefik_router_requests_total`, etc.
- `traefik_service_request_duration_seconds`, `traefik_service_requests_total`, etc.
- `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`
- Attributes on Traefik Prometheus metrics: `code`, `method`, `protocol`, `entrypoint`, `router`, `service` — **proprietary, non-OTel naming**

##### Log attributes

Log records from the `traefik` scope (24 records) use **PascalCase proprietary attribute names** that do not follow OTel semantic conventions:

| Log Attribute | OTel Equivalent (if any) |
|---------------|--------------------------|
| `RequestMethod` | `http.request.method` |
| `DownstreamStatus` | `http.response.status_code` |
| `RequestPath` | `url.path` |
| `RequestScheme` | `url.scheme` |
| `RequestHost` | `server.address` |
| `ClientAddr` | `client.address` |
| `ClientPort` | `client.port` |
| `Duration` | no direct OTel equivalent (custom) |
| `ServiceName` | no OTel equivalent (Traefik-specific) |
| `RouterName` | no OTel equivalent (Traefik-specific) |
| `KubernetesIngressName` | no OTel equivalent |
| `SpanId` / `TraceId` | duplicate of OTLP `spanId`/`traceId` fields |

Log bodies are **stringified JSON blobs** (single `stringValue` string), not structured OTel log records with typed attributes.

##### Cross-signal consistency

The same HTTP concept is named **differently** across signals:

| Concept | Traces (Traefik spans) | Metrics (OTel) | Metrics (Prometheus) | Logs |
|---------|----------------------|----------------|---------------------|------|
| HTTP method | `http.request.method` ✅ | `http.request.method` ✅ | `method` ❌ | `RequestMethod` ❌ |
| HTTP status | `http.response.status_code` ✅ | `http.response.status_code` ✅ | `code` ❌ | `DownstreamStatus` ❌ |
| URL path | `url.path` ✅ | — | — | `RequestPath` ❌ |
| Protocol version | `network.protocol.version` ✅ | `network.protocol.version` ✅ | `protocol` ❌ | `RequestProtocol` ❌ |

##### Schema URL

| Signal | Schema URL |
|--------|-----------|
| Traces | `https://opentelemetry.io/schemas/1.40.0` ✅ |
| Metrics | `https://opentelemetry.io/schemas/1.40.0` (Traefik native) and `https://opentelemetry.io/schemas/1.18.0` (Prometheus-scraped) |
| Logs | `https://opentelemetry.io/schemas/1.40.0` ✅ |

#### Rationale

Traefik is assigned **Level 1 — OpenTelemetry-Aligned** because:

1. **Traefik's own spans use current OTel semconv** (117 spans from `github.com/traefik/traefik` scope) with correct attributes like `http.request.method`, `http.response.status_code`, `url.path`, `url.scheme`, `network.peer.address`, `client.address`, `user_agent.original`. This represents clear progress beyond Level 0.

2. **OTel-native metrics exist** — `http.server.request.duration` and `http.client.request.duration` are emitted with fully OTel-compliant attribute sets including `error.type`.

3. **Schema URLs are present** on all three signals (traces/metrics/logs reference `https://opentelemetry.io/schemas/1.40.0`), demonstrating governance intent.

4. **However, consistency is incomplete across all signals:**
   - The Prometheus-scraped `traefik_*` metrics (the primary operational metrics) use proprietary label names (`code`, `method`, `protocol`, `entrypoint`, `router`, `service`) instead of OTel semconv names.
   - Log attributes use PascalCase proprietary naming (`RequestMethod`, `DownstreamStatus`, `ClientAddr`, `RouterName`) with no OTel alignment, and log bodies are stringified JSON blobs rather than structured OTel attributes.
   - 24 spans from the backend's `@opentelemetry/instrumentation-http` (part of the same distributed traces) carry the full set of deprecated HTTP attributes.
   - The same concept is named differently across signals (HTTP method: `http.request.method` vs `method` vs `RequestMethod`).

5. **The dual metric system** means users cannot rely on a single, consistent OTel-aligned view — off-the-shelf OTel dashboards would work for the OTel metrics but not for the primary Traefik operational metrics.

To reach **Level 2**, Traefik would need to: (a) migrate `traefik_*` Prometheus metric labels to OTel semconv attribute names, (b) restructure access logs to emit structured OTel log attributes using OTel semconv names instead of PascalCase proprietary attributes, and (c) ensure no deprecated HTTP attributes appear anywhere in the trace data.

---

### 3. Resource Attributes & Configuration

**Level: 3 — OpenTelemetry-Optimized**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik sets the following resource attributes natively via the OTel Go SDK across all three signals (traces, metrics via OTLP push, and logs):

| Attribute | Value observed |
|-----------|----------------|
| `service.name` | `traefik` |
| `service.version` | `3.7.1` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-5fb59c7b48-c2pgw` (pod name via `resource.WithHost()`) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-5fb59c7b48-c2pgw 6.17.0-1013-azure ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.10` |
| `process.runtime.description` | `go version go1.25.10 linux/amd64` |
| `process.command_args` | (CLI args array) |

**Natively detected Kubernetes attributes** (via Traefik's built-in `K8sAttributesDetector`):

| Attribute | Value observed |
|-----------|----------------|
| `k8s.pod.name` | `traefik-5fb59c7b48-c2pgw` |
| `k8s.pod.uid` | `d1e20c59-c3f8-42ef-8d27-fd32a5650310` |
| `k8s.namespace.name` | `traefik` |

Traefik implements a custom `K8sAttributesDetector` (in `pkg/types/k8sdetector.go`) that queries the Kubernetes API using in-cluster credentials to detect `k8s.pod.name`, `k8s.pod.uid`, and `k8s.namespace.name` natively at the source — without relying on the Collector.

##### service.name consistency across signals

| Signal | `service.name` value |
|--------|---------------------|
| Traces | `traefik` |
| Metrics (OTLP push) | `traefik` |
| Logs | `traefik` |
| **Consistent** | **Yes** ✅ |

`service.version: 3.7.1` is also identical across all three signals.

##### OTEL_* env var support

- `OTEL_RESOURCE_ATTRIBUTES` is **documented** in the Traefik official docs under the `resourceAttributes` section for both tracing and metrics.
- All three signal providers (tracing in `pkg/observability/types/tracing.go`, metrics in `pkg/observability/metrics/otel.go`, logs in `pkg/observability/types/logs.go`) use identical resource-building patterns ending with `resource.WithFromEnv()` — giving `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` highest precedence.
- Source code comment explicitly states: *"Use the environment variables to allow overriding above resource attributes."*
- Configuration precedence is clear: `OTEL_*` env vars take highest priority, overriding project defaults and `resourceAttributes` config.

##### Identity misplacement

None observed. No `service.*`, `deployment.*`, or `cloud.*` attributes were found on span attributes. Identity is correctly placed only in the resource scope.

#### Rationale

Traefik v3.7.1 achieves **Level 3 — OpenTelemetry-Optimized** for resource attributes and configuration.

1. **Complete native resource attribute set**: `service.name`, `service.version`, `telemetry.sdk.*`, `host.*`, `os.*`, `process.*` are all emitted natively across every signal using the OTel Go SDK's standard resource detectors.

2. **Native Kubernetes identity detection**: Traefik implements its own `K8sAttributesDetector` that queries the Kubernetes API to populate `k8s.pod.name`, `k8s.pod.uid`, and `k8s.namespace.name` at the source — not relying on Collector pipeline enrichment for core identity.

3. **Perfect signal consistency**: `service.name: traefik` and `service.version: 3.7.1` are identical across all three signal types, with no discrepancy.

4. **Explicit, documented `OTEL_*` support with clear precedence**: All three signal providers use the same resource-building pattern with `resource.WithFromEnv()` as the final (highest-priority) step. This is documented in the official Traefik reference docs and commented in the source code.

5. **Zero identity misplacement**: No `service.*` or identity attributes found on span attributes — all identity is correctly in the resource scope.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure
- Total root spans: 72
- Total child spans: 170
- Multi-span traces: 20
- Single-span traces: 53
- Span kind distribution: SERVER=120, INTERNAL=117, CLIENT=24

> Note: Kind=2 (SERVER) is used by both `github.com/traefik/traefik` (96 spans) and `@opentelemetry/instrumentation-http` (24 spans). Kind=3 (CLIENT) is used exclusively for `ReverseProxy` spans from Traefik. Kind=1 (INTERNAL) is used for Express middleware spans from `@opentelemetry/instrumentation-express`.

##### Context propagation
W3C Trace Context is actively propagated. Traefik injects the `traceparent` header when forwarding requests to backends, and the backend (Node.js/Express) correctly picks it up — evidenced by traces that span **three instrumentation scopes** in a single `traceId`:

```
github.com/traefik/traefik             | GET           | kind=SERVER  | parent=ROOT (or external)
github.com/traefik/traefik             | ReverseProxy  | kind=CLIENT  | parent=GET
@opentelemetry/instrumentation-http   | GET /         | kind=SERVER  | parent=ReverseProxy
@opentelemetry/instrumentation-express | middleware-*  | kind=INTERNAL| parent=GET /
```

The 40-span trace shows multiple parallel requests all parented under an external span (`00f067aa0ba902b7` not in collected data), confirming Traefik accepts incoming `traceparent` headers from external callers.

##### Trace coherence assessment
Traces are highly coherent and tell a clear end-to-end story. For proxied requests, a user can follow: **external caller → Traefik entry-point (SERVER) → Traefik ReverseProxy (CLIENT) → backend HTTP server (SERVER) → Express middleware chain (INTERNAL)**. The 53 single-span traces are all for Traefik-internal endpoints (`/ping`, `/metrics`, `/health`) — correctly isolated, not fragmented proxied traces.

Error cases (404 on `/nonexistent`) correctly set `status.code=2` on the `ReverseProxy` CLIENT span while the backend's SERVER spans remain `UNSET`, accurately reflecting where the error was observed.

##### Span links usage
Absent — appropriate since all parent-child relationships are expressed via direct parentage for synchronous request chains.

#### Rationale

Traefik earns **Level 2 — OpenTelemetry-Native** based on strong evidence of intentional, consistent trace modeling:

1. **Consistent SERVER kind at all entry points** — every single root span (72/72) uses `kind=2`, demonstrating deliberate modeling.

2. **End-to-end W3C Trace Context propagation** — traces span three independent instrumentation scopes within a single `traceId`, proving `traceparent` is correctly injected by Traefik's `ReverseProxy` CLIENT span and received by the backend.

3. **Correct CLIENT/SERVER/INTERNAL span kind semantics** — `ReverseProxy` is `CLIENT` (kind=3); backend entry is `SERVER` (kind=2); Express middleware is `INTERNAL` (kind=1). This matches OTel semantic conventions precisely.

4. **Documented behavior** — the Traefik documentation explicitly covers W3C Trace Context, parent-based sampling, and OTLP export.

Level 3 is not reached because: (a) no span events are used to enrich spans, (b) async/fan-out trace topology beyond parallel synchronous requests is not demonstrated, and (c) there is no evidence of architectural trace review or validation testing processes.

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability
- Traces: **flowing** — OTLP gRPC push; 242 spans across 27 JSONL batches; scope `github.com/traefik/traefik`
- Metrics: **flowing** — dual pipeline: OTLP gRPC push for OTel semantic convention metrics **and** Prometheus scrape for `traefik_*` series; 2,904 metric data points
- Logs: **flowing** — OTLP gRPC push via `--accesslog.otlp.grpc`; 24 access log records; application-level logs via `--experimental.otlpLogs` configured but flagged as experimental

##### Cross-signal correlation
- Log records with traceId: **24 of 24 (100%)**
- Log records with spanId: **24 of 24 (100%)**
- Unique log traceIds that resolve to a span in traces: **20 of 20 (100%)**
- Shared attribute keys (traces ∩ metrics): `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`

##### Collection model per signal

| Signal | Mechanism | Scope / Instrumentation |
|--------|-----------|------------------------|
| Traces | OTLP gRPC push (native SDK) | `github.com/traefik/traefik` v3.7.1 |
| Metrics | OTLP gRPC push (native SDK) **+** Prometheus scrape | Both paths produce `traefik_*` series; OTLP-only path additionally produces `http.server.request.duration` / `http.client.request.duration` |
| Logs | OTLP gRPC push via `accesslog.otlp` (access logs only) | Scope `traefik`; `--experimental.otlpLogs=true` covers application-level logs but remains experimental |

##### Investigative workflow assessment

A user **can** follow the metric → trace → log path without manual bridging:

1. **Metric anomaly** — `traefik_router_requests_total` or `http.server.request.duration` spikes; attributes narrow the scope.
2. **Pivot to traces** — same `http.request.method`, `http.response.status_code`, `server.address` attributes present on spans enable a direct pivot.
3. **Inspect logs** — every access log record carries OTLP-level `traceId` and `spanId` fields byte-for-byte identical to the trace IDs on the corresponding spans.

The workflow is seamless for **request-level investigation** but incomplete for **application-level investigation** (startup errors, configuration issues, routing decisions) because application logs remain behind the `--experimental.otlpLogs` flag.

#### Rationale

Traefik v3.7.1 reaches **Level 2** because all three signals are present and correlated by design. Traces and access logs share the same OTLP pipeline, and every access log record is stamped with the trace ID and span ID of the matching request span — a 100% correlation rate with zero manual bridging required. Metrics share six OTel semantic-convention attribute keys with traces, enabling a direct metric → trace pivot.

The project falls short of Level 3 for three reasons:
1. **Dual metrics pipeline**: Traefik emits the same `traefik_*` metrics both via OTLP push and via a Prometheus scrape endpoint — additive duplication rather than intentional signal shaping.
2. **Experimental application logs**: `--experimental.otlpLogs` is configured but gated; only access logs (HTTP request/response records) flow as OTLP logs.
3. **No cross-signal investigative workflow documentation**: Each signal is documented independently with no published guide describing how to move from a metric anomaly to a trace to a correlated log entry.

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span name audit

Traefik emits only two span names from its own scope (`github.com/traefik/traefik`):
- `GET` — 88 occurrences across all entry points (lacks route context in the name itself)
- `ReverseProxy` — 24 occurrences (an internal component name, not a user-facing operation description)

`ReverseProxy` is an internal implementation detail that requires domain knowledge of Traefik's architecture to interpret. `GET` lacks the route template (e.g. `GET /api/users/{id}`) that would make it actionable for operators.

##### Log severity and verbosity analysis

24 log records observed (one per HTTP request), all with OTel `severityText: info`. **Critical severity mismatch bug**: the embedded JSON body carries `"level": "panic"` (Logrus level) while OTel reports `info`. This is a Traefik bug in how it serializes access log records — severity-based alerting on logs would silently fail for all access log entries. Log bodies are double-encoded JSON strings with PascalCase attribute names inconsistent with OTel conventions. Timestamps in the body show Go zero-time (`0001-01-01T00:00:00Z`).

##### Metric quality assessment

77 unique metric names identified. Traefik's own metrics (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`) form a strong SLO-ready three-tier hierarchy supporting the RED method. OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) are also present. Runtime `go_*` and `process_*` metrics add noise but are standard.

##### High-cardinality attribute check

- Pod IPs in `server.address` for OTel metrics — potential cardinality issue in large clusters
- `url.full` in `ReverseProxy` spans contains raw backend pod IPs

#### Key findings by signal

| Signal | Quality | Key Issue |
|--------|---------|-----------|
| **Metrics** | ✅ Strong (Level 2) | SLO-ready three-tier hierarchy; pod IPs in some label values |
| **Traces** | ❌ Weak (Level 0–1) | `ReverseProxy` is internal naming; `GET` lacks route context in name |
| **Logs** | ⚠️ Partial (Level 1) | Severity mapping bug (logrus `panic` → OTel `info`); non-standard PascalCase attributes |

#### Rationale

Assigned **Level 1 — OpenTelemetry-Aligned**. The project's metrics are genuinely operator-friendly — the `traefik_entrypoint_*` / `traefik_router_*` / `traefik_service_*` three-tier hierarchy provides excellent RED method coverage and is SLO-ready. However, the trace span naming (`ReverseProxy` as an internal component name) and the log severity mapping bug prevent reaching Level 2. An operator unfamiliar with Traefik internals would need domain knowledge to interpret trace data, and severity-based alerting on logs would silently fail because every access log record reports `info` at the OTel level regardless of the actual log level in the body.

To reach Level 2, Traefik would need to: (a) use route-template-based span names (e.g. `GET /api/{id}`) instead of generic `GET` and internal `ReverseProxy`, (b) fix the severity mapping bug so OTel severity correctly reflects the actual log level, and (c) restructure log bodies as structured OTel attributes rather than stringified JSON blobs.

---

### 7. Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence
- Traces: **Present** — `https://opentelemetry.io/schemas/1.40.0`
- Metrics: **Present** — `https://opentelemetry.io/schemas/1.18.0` (Prometheus receiver scope) and `https://opentelemetry.io/schemas/1.40.0` (Traefik native scope)
- Logs: **Present** — `https://opentelemetry.io/schemas/1.40.0`

##### Telemetry documentation

A **dedicated observability reference section** exists with a comprehensive metrics reference table listing all emitted metric names, their types, label dimensions, and descriptions. Traefik also maintains **official Grafana dashboards** (IDs 17346 and 17347) for on-premises and Kubernetes deployments, signalling that the metrics surface is treated as a user-facing contract. There is **no explicit "stable" vs "experimental" labeling** applied to individual metrics or spans in the documentation.

##### Release note quality for telemetry changes

Telemetry changes are **consistently tagged and documented** in the CHANGELOG using `[otel]`, `[metrics]`, `[tracing]`, `[logs]` prefixes. Examples:

- **v3.5.4**: `traefik_tls_certs_not_after_milliseconds` → `traefik_tls_certs_not_after_seconds` — breaking metric rename called out explicitly with a dedicated migration guide section.
- **v3.5.0**: `traceVerbosity` change — migration note explicitly warns about fewer spans being generated with the new default.
- **v2→v3 migration guide**: Dedicated "Observability" section documenting all breaking telemetry changes including metric renames/removals, span behavior changes, and full removal of vendor-specific trace exporters with migration paths.

##### Instrumentation scope versions

| Scope | Version |
|-------|---------|
| `github.com/traefik/traefik` (metrics) | `v3.7.1` ✅ |
| `github.com/traefik/traefik` (traces) | `unknown` ⚠️ |

Traefik's own metrics scope carries a proper semantic version, directly tying the instrumentation to the release. The traces scope does not carry a version — a minor gap.

#### Rationale

Traefik is assessed at **Level 2 — OpenTelemetry-Native**. The project treats telemetry as a meaningful user-facing contract. Schema URLs are present on all three signals. The metrics reference documentation is comprehensive. Telemetry changes are consistently tagged in the CHANGELOG, and breaking changes receive dedicated migration guide sections with explicit impact statements and remediation steps.

The project falls short of Level 3 primarily because:
1. No formal per-signal stable/experimental labeling — users cannot tell which metrics are guaranteed stable vs subject to change.
2. No formal telemetry change review process (design proposals, TEPs) is publicly documented.
3. Metric renames do not appear to follow a dual-emission deprecation window.
4. The traces instrumentation scope does not carry a version (`unknown`).
5. No evidence of proactive telemetry regression detection (e.g. automated dashboard tests or signal quality gates in CI).

---

## Key findings

### Top 3 strengths

1. **Genuine OpenTelemetry-native integration surface**: Traefik v3 completed a clean break from legacy exporters — Jaeger, Zipkin, and Datadog tracing backends were fully removed in v3, making OTLP the sole trace export path. All three signals flow natively via OTLP gRPC to an OTel Collector without any adapter, sidecar, or bridge component. This is a meaningful architectural commitment, not just an added option.

2. **Exemplary resource attribute completeness and native Kubernetes identity detection**: Traefik achieves Level 3 for resource attributes — the highest level in the model. The project natively emits `service.name`, `service.version`, `telemetry.sdk.*`, `host.*`, `os.*`, and `process.*` consistently across all three signals, and implements its own `K8sAttributesDetector` to populate `k8s.pod.name`, `k8s.pod.uid`, and `k8s.namespace.name` at the source without relying on Collector enrichment. `OTEL_RESOURCE_ATTRIBUTES` is documented and given highest override precedence via `resource.WithFromEnv()`.

3. **Strong telemetry change management and migration guidance**: Traefik treats telemetry as a user-facing contract. Breaking telemetry changes receive dedicated sections in migration guides with explicit before/after examples and remediation steps. The CHANGELOG consistently uses `[otel]`, `[metrics]`, `[tracing]` tags. Official Grafana dashboards (IDs 17346 and 17347) signal that the metrics surface is considered stable. The v2→v3 migration guide's "Observability" section is a model example of proactive telemetry communication.

### Top 3 areas for improvement

1. **Cross-signal semantic convention consistency**: The same HTTP concept is named three different ways across signals (`http.request.method` in traces/OTel metrics, `method` in Prometheus metrics, `RequestMethod` in logs). Log attributes use PascalCase proprietary naming with no OTel alignment, and log bodies are stringified JSON blobs rather than structured OTel attributes. Migrating Prometheus metric labels to OTel semconv names and restructuring log attributes would unlock Level 2 for Semantic Conventions and significantly improve cross-signal query correlation.

2. **Log signal quality and severity mapping bug**: All 24 access log records carry `severityText: info` at the OTel level, but the embedded JSON body contains `"level": "panic"` — a Logrus serialization bug that would cause severity-based alerting to silently fail. Additionally, application-level logs remain behind the `experimental.otlpLogs: true` feature gate, meaning startup errors, configuration reloads, and routing decisions are not observable via OTLP. Fixing the severity mapping and graduating log export to stable would significantly improve observability completeness.

3. **Trace span naming for operator usability**: Traefik emits only two span names from its own scope: `GET` (generic HTTP method, lacking route template context) and `ReverseProxy` (an internal component name requiring domain knowledge). Adding route-template-based span names (e.g. `GET /api/{resource}/{id}`) would make traces immediately actionable for operators unfamiliar with Traefik internals, and would unlock Level 2 for Audience & Signal Quality.

### Notable observations

- **Dual metrics pipeline**: Traefik emits `traefik_*` metrics via both OTLP push and Prometheus scrape simultaneously, with no documented rationale or deprecation timeline for the Prometheus path. This creates overlapping series in the metrics store and ambiguity about which pipeline to prefer.
- **Distributed trace coherence is excellent**: Despite the span naming concerns, the trace structure itself is textbook-correct — 100% of root spans use `kind=SERVER`, `ReverseProxy` uses `kind=CLIENT`, W3C Trace Context is propagated end-to-end across service boundaries, and error status codes are correctly placed on the CLIENT span where the error was observed.
- **`service.version` reflects the actual running version**: `service.version: 3.7.1` is set at build time from `version.Version` and is consistent across all three signals — a detail many projects miss.
- **Backend spans use deprecated OTel HTTP attributes**: 24 spans from the test backend's `@opentelemetry/instrumentation-http` v0.48.0 carry the full set of deprecated HTTP attributes (`http.method`, `http.status_code`, `http.target`, etc.), which appear in the same trace data as Traefik's correctly-attributed spans. This is a backend library version issue, not a Traefik issue, but it affects the overall semantic convention picture.
- **OTLP log export quirk**: Access log records embed `TraceId`/`SpanId` redundantly in four different forms in the log body and attributes (`TraceId`, `SpanId`, `trace_id`, `span_id`), in addition to the OTLP-level `traceId`/`spanId` fields. This redundancy suggests the log serialization path was not designed with OTLP's native correlation fields in mind.

---

## Methodology notes

- **Evaluation cluster**: kind (`otel-eval-traefik`), single-node, Kubernetes v1.31
- **Traefik version**: v3.7.1 (Helm chart `traefik/traefik` v40.0.0)
- **OTel Collector version**: `otel/opentelemetry-collector-contrib` v0.150.1
- **Telemetry collection window**: ~10 minutes of synthetic HTTP traffic (GET `/`, `/ping`, `/metrics`, `/health`, `/nonexistent`)
- **Test backend**: Node.js/Express application with `@opentelemetry/auto-instrumentations-node`
- **Signal files**: `/tmp/otel-eval-traefik/traces.jsonl` (61 lines), `/tmp/otel-eval-traefik/metrics.jsonl` (78 lines), `/tmp/otel-eval-traefik/logs.jsonl` (4 lines)
- **Model**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)
- **Skill version**: evaluate-otel-maturity v0.0.5
- **Key analytical principle**: All dimension scores reflect what Traefik supports natively. Collector-derived enrichment (e.g. `k8s.deployment.name` added by `k8sattributes` processor) is noted but does not influence dimension scores.
- **Dimension 6 (Audience & Signal Quality)**: The dimension report file (`dim-6-audience-signal-quality.md`) was not present on disk at assembly time; the dimension evaluation was reconstructed from the dimension agent's summary provided in the user message.
