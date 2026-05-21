# OTel Maturity Evaluation: Traefik (v1)

> **Evaluation run**: v1  
> **Assembled**: 2026-05-21  
> **Evaluator**: OTel Maturity Evaluation Pipeline  

---

## Project Overview

| Field | Value |
|---|---|
| **Project** | Traefik |
| **CNCF Status** | Incubating |
| **Evaluation Tag** | v1 |
| **Application Version** | v3.7.0 |
| **Helm Chart** | `traefik/traefik` v40.0.0 |
| **Helm Repo** | `https://traefik.github.io/charts` |
| **Namespace** | `traefik` |
| **Kubernetes Context** | `kind-otel-eval-traefik` |
| **OTel SDK** | OTel Go SDK (`opentelemetry-go v1.43.0`) |
| **GitHub** | https://github.com/traefik/traefik |
| **Docs** | https://doc.traefik.io/traefik/ |

Traefik is a cloud-native HTTP reverse proxy and load balancer, widely used as a Kubernetes Ingress controller. It dynamically discovers routing rules from Kubernetes Ingress, IngressRoute (CRD), and Gateway API resources, and forwards traffic to backend services. It acts as the edge proxy between external clients and cluster workloads.

---

## Summary Table

| # | Dimension | Level | Label |
|---|---|---|---|
| 1 | Integration Surface | **1** | OpenTelemetry-Aligned |
| 2 | Semantic Conventions | **1** | OpenTelemetry-Aligned |
| 3 | Resource Attributes & Configuration | **3** | OpenTelemetry-Optimized |
| 4 | Trace Modeling & Context Propagation | **2** | OpenTelemetry-Native |
| 5 | Multi-Signal Observability | **2** | OpenTelemetry-Native |
| 6 | Audience & Signal Quality | **1** | OpenTelemetry-Aligned |
| 7 | Stability & Change Management | **2** | OpenTelemetry-Native |
| | **Overall (mean)** | **1.7** | OpenTelemetry-Aligned → Native |

> Level scale: 0 = Instrumented · 1 = OpenTelemetry-Aligned · 2 = OpenTelemetry-Native · 3 = OpenTelemetry-Optimized

---

## Telemetry Overview

### Signals Observed

| Signal | Transport | Scope | Status | Notes |
|---|---|---|---|---|
| **Traces** | OTLP gRPC | `github.com/traefik/traefik` | ✅ Flowing | 449 spans across 140 trace IDs; 11 batches confirmed |
| **Metrics (OTLP push)** | OTLP gRPC | `github.com/traefik/traefik` | ✅ Flowing | 17 metric names; 10s push interval; 17+ batches |
| **Metrics (Prometheus scrape)** | Prometheus pull | `prometheusreceiver` | ✅ Flowing | 15 `traefik_*` metrics + Go runtime; 15s scrape interval |
| **Logs (access)** | OTLP gRPC | `traefik` | ✅ Flowing (experimental) | Per-request access logs; requires `experimental.otlpLogs: true` |

### Signals Summary

**Traces:** Traefik emits two span types — `GET` (SERVER kind, entry-point) and `ReverseProxy` (CLIENT kind, upstream proxy call). All 155 Traefik entry-point spans are `SERVER` kind; all 52 `ReverseProxy` spans are `CLIENT` kind. W3C Trace Context propagation is fully functional. End-to-end traces span Traefik → backend with shared trace IDs and zero context loss. Instrumentation scope: `github.com/traefik/traefik` (version `unknown` in traces scope, `v3.7.0` in metrics scope).

**Metrics:** Two parallel metric paths operate simultaneously:
- **OTLP push** delivers 2 OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) plus 15 `traefik_*` proprietary metrics.
- **Prometheus scrape** delivers the same 15 `traefik_*` metrics plus ~37 Go runtime/process metrics.
- Exemplars are present in histogram metrics, containing Trace ID + Span ID for metrics-to-traces correlation.

**Logs:** Access logs are exported via OTLP gRPC (behind the `experimental.otlpLogs: true` feature gate). Each record contains Trace ID and Span ID at the OTLP protocol level, enabling direct correlation with traces. Log body is a JSON string (not a structured object). All records carry a `level: panic` serialization bug in the JSON body regardless of HTTP status.

### Resource Attributes

All three OTLP signals carry a consistent, rich set of resource attributes:

| Attribute | Value | Native? |
|---|---|---|
| `service.name` | `traefik` | ✅ Native |
| `service.version` | `3.7.0` | ✅ Native |
| `telemetry.sdk.name` | `opentelemetry` | ✅ Native |
| `telemetry.sdk.language` | `go` | ✅ Native |
| `telemetry.sdk.version` | `1.43.0` | ✅ Native |
| `host.name` | Pod name (e.g., `traefik-7cd47954cc-npktt`) | ✅ Native |
| `os.type` | `linux` | ✅ Native |
| `os.description` | `Alpine Linux 3.23.4 (Linux ...)` | ✅ Native |
| `process.pid` | `1` | ✅ Native |
| `process.executable.name` | `traefik` | ✅ Native |
| `process.executable.path` | `/usr/local/bin/traefik` | ✅ Native |
| `process.owner` | `traefik` | ✅ Native |
| `process.runtime.name` | `go` | ✅ Native |
| `process.runtime.version` | `go1.25.9` | ✅ Native |
| `process.command_args` | Full CLI args array | ✅ Native |
| `k8s.pod.name` | `traefik-7cd47954cc-npktt` | ✅ Native (K8sAttributesDetector) |
| `k8s.pod.uid` | `65fd3a22-73fb-4c3e-a49d-4a5d20397a3d` | ✅ Native (K8sAttributesDetector) |
| `k8s.namespace.name` | `traefik` | ✅ Native (K8sAttributesDetector) |
| `k8s.deployment.name` | `traefik` | ⚙️ Pipeline-derived (k8sattributes processor) |
| `k8s.replicaset.name` | `traefik-7cd47954cc` | ⚙️ Pipeline-derived |
| `k8s.node.name` | Node name | ⚙️ Pipeline-derived |

Schema URL: `https://opentelemetry.io/schemas/1.40.0` on all three OTLP signals.

---

## Installation Context Summary

### Deployment

Traefik v3.7.0 was deployed via Helm chart `traefik/traefik` v40.0.0 into the `traefik` namespace of a kind cluster (`kind-otel-eval-traefik`). The OTel Collector (from `open-telemetry/opentelemetry-collector`) was deployed in the `opentelemetry` namespace and configured to receive both OTLP push and Prometheus scrape from Traefik.

### Telemetry Configuration Method

All telemetry configuration uses **Traefik-specific CLI flags** (no standard `OTEL_*` env vars for endpoint configuration):

```
--tracing.otlp=true
--tracing.otlp.grpc=true
--tracing.otlp.grpc.endpoint=otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317
--tracing.otlp.grpc.insecure=true
--metrics.otlp=true
--metrics.otlp.grpc=true
--metrics.otlp.grpc.endpoint=otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317
--metrics.otlp.grpc.insecure=true
--metrics.prometheus=true                    # also active simultaneously
--accesslog.otlp=true
--accesslog.otlp.grpc=true
--accesslog.otlp.grpc.endpoint=otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317
--experimental.otlpLogs=true                 # feature gate required for log export
--tracing.sampleRate=1                       # 100% sampling for evaluation
--tracing.addInternals=true                  # trace internal routes (/ping, /metrics, dashboard)
```

### Collector Pipeline

The OTel Collector was configured with:
- **Receivers**: OTLP gRPC/HTTP (from Traefik push), Prometheus (scraping Traefik at `traefik-metrics.traefik.svc.cluster.local:9100`), k8s_cluster
- **Processors**: k8sattributes (enrichment), batch, memory_limiter
- **Exporters**: debug/detailed (for verification), file (JSONL output to `/tmp/otel-eval-traefik/`)

### Traffic Generation

Test traffic was generated via port-forward to Traefik's web entrypoint:
- 10 normal `GET /` requests
- 5 requests with injected `traceparent` header (context propagation test)
- 5 `GET /health` requests
- 3 `GET /nonexistent` requests (404 error path)

The test backend (`otel-eval-backend`) is a Node.js Express app with OTel auto-instrumentation (`@opentelemetry/instrumentation-http` v0.48.0, `@opentelemetry/instrumentation-express` v0.35.0).

### Known Quirks

| Issue | Impact |
|---|---|
| `level: panic` in access log JSON body | All access log records show `panic` severity in body regardless of HTTP status — serialization bug |
| `experimental.otlpLogs: true` required | OTLP log export is not yet GA; instability risk for production |
| Prometheus metrics active by default | Both OTLP push and Prometheus scrape run simultaneously; disabling Prometheus requires explicit config |
| `OTEL_EXPORTER_OTLP_ENDPOINT` not honored | Standard OTel env vars do not configure Traefik's OTLP endpoints |
| Trace scope version missing | `github.com/traefik/traefik` scope has `vunknown` in traces; `v3.7.0` only in metrics scope |
| `url.full` contains pod IPs | `ReverseProxy` CLIENT spans carry full upstream URL with pod IP — high-cardinality in production |

---

## Dimension Evaluations

---

### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅ (OTLP gRPC, scope `github.com/traefik/traefik`), Metrics ✅ (OTLP gRPC push, scope `github.com/traefik/traefik`; also via Prometheus scrape), Logs ✅ (OTLP gRPC, scope `traefik` — access logs confirmed flowing)
- **Configuration method**: Traefik-specific flags (`--tracing.otlp.grpc.endpoint`, `--metrics.otlp.grpc.endpoint`, `--accesslog.otlp.grpc.endpoint`, `--experimental.otlpLogs=true`). No `OTEL_EXPORTER_OTLP_ENDPOINT` or standard OTel SDK env vars respected for configuration.
- **Documentation stance**: OTLP is documented alongside Prometheus (metrics) and legacy Zipkin/Jaeger exporters (traces). Multiple backends are presented as co-equal options. The v3 docs dedicate distinct sections to each exporter type.
- **Legacy exporter status**: Co-equal — Prometheus is still the default/primary metrics path (enabled by default in the Helm chart); Zipkin and Jaeger trace exporters remain documented alongside OTLP. There is no deprecation notice on legacy exporters.
- **Signals requiring adapters/sidecars**: None — all three signals flow natively from Traefik to the OTel Collector via OTLP gRPC without any adapter or sidecar.

**Observed telemetry (confirmed from JSONL files):**

| Signal | Transport | Scope | Lines/Batches |
|--------|-----------|-------|---------------|
| Traces | OTLP gRPC | `github.com/traefik/traefik` | 1 batch, 31 Traefik spans (`GET`, `ReverseProxy`) |
| Metrics (OTLP push) | OTLP gRPC | `github.com/traefik/traefik` | 17 metric batches; 17 metric names including `http.server.request.duration`, `http.client.request.duration` (OTel semantic conventions) + 15 `traefik_*` names |
| Metrics (Prometheus) | Prometheus scrape | `prometheusreceiver` | Same 15 `traefik_*` metrics + Go runtime metrics |
| Logs (access) | OTLP gRPC | `traefik` | 5 batches, 30+ log records with `trace_id`/`span_id` correlation |

**Configuration required (from process args observed in telemetry):**

```
--tracing.otlp=true
--tracing.otlp.grpc=true
--tracing.otlp.grpc.endpoint=<collector>:4317
--tracing.otlp.grpc.insecure=true
--metrics.otlp=true
--metrics.otlp.grpc=true
--metrics.otlp.grpc.endpoint=<collector>:4317
--metrics.otlp.grpc.insecure=true
--metrics.prometheus=true                          # also still enabled simultaneously
--accesslog.otlp=true
--accesslog.otlp.grpc=true
--accesslog.otlp.grpc.endpoint=<collector>:4317
--experimental.otlpLogs=true                       # feature gate required for log export
```

**Key observations:**

1. **OTLP is supported but not default**: All three signals require explicit opt-in via Traefik-specific flags. Prometheus metrics are enabled by default in the Helm chart; OTLP metrics must be explicitly added.

2. **Dual metrics paths run simultaneously**: Both Prometheus scrape and OTLP push are active at the same time, producing overlapping metric data. The OTLP push adds OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) that are not available via Prometheus scrape.

3. **Log export behind experimental feature gate**: OTLP log export requires `--experimental.otlpLogs=true` (or `experimental.otlpLogs: true` in Helm values). This is explicitly marked as non-stable, meaning it is not part of the stable integration surface.

4. **No standard OTel env var support**: Traefik does not respect `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, or `OTEL_SERVICE_NAME`. Each signal requires separate, Traefik-specific endpoint configuration flags. This is a significant departure from OTel SDK conventions.

5. **Per-signal configuration inconsistency**: Traces, metrics, and logs each have their own separate OTLP endpoint configuration blocks (`tracing.otlp`, `metrics.otlp`, `accesslog.otlp`). There is no unified OTLP exporter configuration.

6. **Prometheus still co-equal**: The Traefik Helm chart enables Prometheus metrics by default (`metrics.prometheus.entryPoint: metrics`). The OTLP metrics path is additive, not a replacement. Users must explicitly decide to use OTLP and may end up with both paths active.

7. **Rich OTel SDK integration**: Despite the configuration friction, Traefik v3 uses the OTel Go SDK natively (`telemetry.sdk.name: opentelemetry`, `telemetry.sdk.version: 1.43.0`). Resource attributes (`service.name`, `service.version`, `host.name`, `os.type`, `process.*`, `process.runtime.*`) are rich and OTel-compliant.

8. **W3C Trace Context propagation confirmed**: Traefik correctly reads incoming `traceparent` headers, preserves the trace ID, and creates a child span, forwarding the updated context to backends.

9. **Log body quirk**: Access log records embed the full JSON as a string in the OTLP log body, rather than as a structured object. The `level: panic` field appears in all access log records regardless of HTTP status — a known Traefik serialization bug in v3.

#### Checklist Assessment

**Level 1 — OpenTelemetry-Aligned** — confirmed. All five Level 1 characteristics are present:
- OTLP is supported alongside equally-promoted legacy exporters ✅
- Multiple overlapping configuration methods exist (Traefik flags only; OTEL_* not respected) ✅
- OTLP requires disabling legacy behavior or enabling experimental flags ✅
- OTel integration is inconsistent across signals ✅
- Users need multiple docs pages for a working OTLP integration ✅

**Level 2 — OpenTelemetry-Native** — not met:
- OTLP is not the default/clearly-recommended export path ❌
- `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL` are not respected ❌
- Legacy exporters are not clearly secondary or deprecated ❌
- Telemetry configuration is not consistent across all signals ❌

#### Rationale

Traefik v3 is solidly at **Level 1 — OpenTelemetry-Aligned**. The project has made genuine and substantial investment in OTel: it uses the OTel Go SDK natively, supports OTLP for all three signals, implements W3C Trace Context propagation, emits OTel semantic convention metrics, and requires no adapters or sidecars for an OTel Collector integration. However, the complete absence of standard OTel env var support, Prometheus as the default metrics path, log export behind an experimental feature gate, and per-signal configuration fragmentation prevent Level 2.

---

### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

Deprecated attributes are present exclusively on spans emitted by the **backend demo service** (`@opentelemetry/instrumentation-http` / `@opentelemetry/instrumentation-express` scopes), **not** on Traefik's own spans. These originate from an outdated Node.js OTel auto-instrumentation library bundled with the backend workload, not from Traefik itself:

| Deprecated Attribute | Count | Scope |
|---|---|---|
| `http.method` | 34 | `@opentelemetry/instrumentation-http` |
| `http.status_code` | 34 | `@opentelemetry/instrumentation-http` |
| `http.url` | 34 | `@opentelemetry/instrumentation-http` |
| `http.target` | 34 | `@opentelemetry/instrumentation-http` |
| `http.host` | 34 | `@opentelemetry/instrumentation-http` |
| `http.scheme` | 34 | `@opentelemetry/instrumentation-http` |
| `http.flavor` | 34 | `@opentelemetry/instrumentation-http` |
| `http.user_agent` | 34 | `@opentelemetry/instrumentation-http` |
| `http.status_text` | 34 | `@opentelemetry/instrumentation-http` |
| `http.client_ip` | 34 | `@opentelemetry/instrumentation-http` |
| `net.host.name` | 34 | `@opentelemetry/instrumentation-http` |
| `net.host.ip` | 34 | `@opentelemetry/instrumentation-http` |
| `net.host.port` | 34 | `@opentelemetry/instrumentation-http` |
| `net.peer.ip` | 34 | `@opentelemetry/instrumentation-http` |
| `net.peer.port` | 34 | `@opentelemetry/instrumentation-http` |

**Traefik's own spans (`github.com/traefik/traefik`) contain zero deprecated HTTP/net attributes.**

##### Current OTel attributes found on Traefik spans

| Attribute | Spans |
|---|---|
| `http.request.method` | `GET`, `ReverseProxy` |
| `http.response.status_code` | `GET`, `ReverseProxy` |
| `http.request.body.size` | `GET` (entrypoint) |
| `url.path` | `GET` (entrypoint) |
| `url.query` | `GET` (entrypoint) |
| `url.scheme` | `GET`, `ReverseProxy` |
| `url.full` | `ReverseProxy` |
| `network.protocol.version` | `GET`, `ReverseProxy` |
| `network.peer.address` | `GET`, `ReverseProxy` |
| `network.peer.port` | `GET`, `ReverseProxy` |
| `server.address` | `GET`, `ReverseProxy` |
| `server.port` | `ReverseProxy` |
| `client.address` | `GET` (entrypoint) |
| `client.port` | `GET` (entrypoint) |
| `user_agent.original` | `GET`, `ReverseProxy` |

Additionally, Traefik adds one custom attribute: `entry_point` (Traefik-specific, naming the entrypoint such as `web`, `traefik`, `metrics`). This is a domain concept with no OTel semconv equivalent, but it is not namespaced (should ideally be `traefik.entry_point`).

Span event attributes (`exception.message`, `exception.stacktrace`, `exception.type`) follow OTel semantic conventions correctly.

##### Metric names and conventions

Traefik emits metrics from **two parallel systems simultaneously**:

**OTel-native metrics (OTLP push) — current semconv:**

| Metric Name | Type | Attributes |
|---|---|---|
| `http.server.request.duration` | histogram | `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `url.scheme`, `error.type` |
| `http.client.request.duration` | histogram | `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`, `error.type` |

**Traefik-proprietary metrics (OTLP push + Prometheus scrape) — non-OTel naming:**

| Metric Name | Attributes Used |
|---|---|
| `traefik_entrypoint_requests_total` | `code`, `entrypoint`, `method`, `protocol` |
| `traefik_entrypoint_request_duration_seconds` | `code`, `entrypoint`, `method`, `protocol` |
| `traefik_entrypoint_requests_bytes_total` | `code`, `entrypoint`, `method`, `protocol` |
| `traefik_entrypoint_responses_bytes_total` | `entrypoint`, `protocol` |
| `traefik_router_requests_total` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_router_request_duration_seconds` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_router_requests_bytes_total` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_router_responses_bytes_total` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_service_requests_total` | `code`, `method`, `protocol`, `service` |
| `traefik_service_request_duration_seconds` | `code`, `method`, `protocol`, `service` |
| `traefik_service_requests_bytes_total` | `code`, `method`, `protocol`, `service` |
| `traefik_service_responses_bytes_total` | `code`, `method`, `protocol`, `service` |
| `traefik_config_reloads_total` | _(none)_ |
| `traefik_config_last_reload_success` | _(none)_ |
| `traefik_open_connections` | `entrypoint`, `protocol` |

The `traefik_*` metrics use Prometheus-style attribute names (`code`, `method`, `protocol`) instead of OTel semconv (`http.response.status_code`, `http.request.method`, `network.protocol.name`).

##### Log attributes

Log records (access logs via OTLP, `traefik` scope) use **PascalCase proprietary attribute names** with no alignment to OTel semantic conventions:

| Log Attribute | OTel Semconv Equivalent |
|---|---|
| `RequestMethod` | `http.request.method` |
| `RequestPath` | `url.path` |
| `RequestScheme` | `url.scheme` |
| `DownstreamStatus` | `http.response.status_code` |
| `ClientHost` | `client.address` |
| `ClientPort` | `client.port` |
| `Duration` | _(no direct equivalent; `http.server.request.duration` is a metric)_ |
| `ServiceAddr` | `server.address` |
| `ServiceURL` | `url.full` |
| `RouterName` | _(Traefik-specific, no semconv)_ |
| `entryPointName` | _(Traefik-specific, no semconv)_ |
| `KubernetesIngressName` | _(Traefik-specific, no semconv)_ |
| `TraceId` / `trace_id` | _(duplicated — both camelCase and snake_case present)_ |
| `SpanId` / `span_id` | _(duplicated — both camelCase and snake_case present)_ |

The log record fields `traceId` and `spanId` are correctly populated in the OTLP log record envelope (not just as attributes), enabling trace correlation. However, the attribute keys themselves are not OTel semconv.

##### Cross-signal consistency

The same HTTP concept is named differently across signals:

| Concept | Traces (Traefik) | Metrics (`traefik_*`) | Metrics (`http.*`) | Logs |
|---|---|---|---|---|
| HTTP method | `http.request.method` ✅ | `method` ❌ | `http.request.method` ✅ | `RequestMethod` ❌ |
| HTTP status | `http.response.status_code` ✅ | `code` ❌ | `http.response.status_code` ✅ | `DownstreamStatus` ❌ |
| Client address | `client.address` ✅ | _(absent)_ | _(absent)_ | `ClientHost` ❌ |
| URL scheme | `url.scheme` ✅ | `protocol` ❌ | `url.scheme` ✅ | `RequestScheme` ❌ |
| Network protocol | `network.protocol.version` ✅ | `protocol` ❌ | `network.protocol.name/version` ✅ | `RequestProtocol` ❌ |

##### Schema URL

| Signal | Schema URL |
|---|---|
| Traces | `https://opentelemetry.io/schemas/1.40.0` ✅ |
| Metrics (OTLP) | `https://opentelemetry.io/schemas/1.40.0` ✅ |
| Metrics (Prometheus) | `https://opentelemetry.io/schemas/1.18.0` ✅ |
| Logs | `https://opentelemetry.io/schemas/1.40.0` ✅ |

#### Rationale

Traefik v3.7.0 is assigned **Level 1 — OpenTelemetry-Aligned** because OTel semantic conventions are partially and inconsistently adopted across signals.

**What earns Level 1 (above Level 0):** Traefik's own trace spans use exclusively current, stable OTel HTTP semantic conventions (zero deprecated attributes). The OTLP-pushed `http.server.request.duration` and `http.client.request.duration` metrics use fully current OTel semconv attributes. A schema URL is declared on all three signals.

**Why it cannot reach Level 2:**
1. The `traefik_*` family of metrics use Prometheus-style shorthand attribute names (`code`, `method`, `protocol`) rather than OTel semconv names for the same concepts.
2. All access log attributes use PascalCase Traefik-internal names with no alignment to OTel semantic conventions.
3. HTTP method is `http.request.method` in traces, `method` in `traefik_*` metrics, and `RequestMethod` in logs — three different names for the same concept.
4. The custom attribute `entry_point` is not namespaced (should be `traefik.entry_point`) and is not documented as an OTel extension.

---

### 3. Resource Attributes & Configuration

**Level: 3 — OpenTelemetry-Optimized**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik v3.7.0 sets the following resource attributes natively via the OTel Go SDK across **all three signals** (traces, metrics, logs):

| Attribute | Value observed | Signal(s) |
|---|---|---|
| `service.name` | `traefik` | Traces, Metrics (OTLP), Logs |
| `service.version` | `3.7.0` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.name` | `opentelemetry` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.language` | `go` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.version` | `1.43.0` | Traces, Metrics (OTLP), Logs |
| `host.name` | `traefik-7cd47954cc-npktt` (pod name) | Traces, Metrics (OTLP), Logs |
| `os.type` | `linux` | Traces, Metrics (OTLP), Logs |
| `os.description` | `Alpine Linux 3.23.4 (Linux ...)` | Traces, Metrics (OTLP), Logs |
| `process.pid` | `1` | Traces, Metrics (OTLP), Logs |
| `process.executable.name` | `traefik` | Traces, Metrics (OTLP), Logs |
| `process.executable.path` | `/usr/local/bin/traefik` | Traces, Metrics (OTLP), Logs |
| `process.owner` | `traefik` | Traces, Metrics (OTLP), Logs |
| `process.runtime.name` | `go` | Traces, Metrics (OTLP), Logs |
| `process.runtime.version` | `go1.25.9` | Traces, Metrics (OTLP), Logs |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` | Traces, Metrics (OTLP), Logs |
| `process.command_args` | Full CLI args array | Traces, Metrics (OTLP), Logs |
| `k8s.pod.name` | `traefik-7cd47954cc-npktt` | Traces, Metrics (OTLP), Logs |
| `k8s.pod.uid` | `65fd3a22-73fb-4c3e-a49d-4a5d20397a3d` | Traces, Metrics (OTLP), Logs |
| `k8s.namespace.name` | `traefik` | Traces, Metrics (OTLP), Logs |

**Note on native K8s attributes**: Traefik implements its own `K8sAttributesDetector` (in `pkg/types/k8sdetector.go`) that queries the Kubernetes API at startup to discover `k8s.pod.uid`, `k8s.pod.name`, and `k8s.namespace.name`. These three attributes are **project-native** — emitted by Traefik itself without any Collector enrichment.

##### OTEL_* env var support

**Documented and implemented end-to-end for all three signals.** All three signal providers use `resource.WithFromEnv()` placed **last** in the resource chain, giving `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_SERVICE_NAME` the highest precedence:

```go
resource.New(ctx,
    resource.WithContainer(),
    resource.WithHost(),
    resource.WithOS(),
    resource.WithProcess(),
    resource.WithTelemetrySDK(),
    resource.WithDetectors(K8sAttributesDetector{}),
    resource.WithAttributes(semconv.ServiceName(serviceName), semconv.ServiceVersion(version.Version)),
    resource.WithAttributes(resAttrs...),  // config-level resourceAttributes
    resource.WithFromEnv(),                // OTEL_* env vars — highest precedence
)
```

The official Traefik docs explicitly state: *"Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes."*

##### Identity misplacement

**None.** No `service.name`, `service.version`, `deployment.*`, or `cloud.*` attributes were found on spans. All identity attributes are correctly in the resource scope.

##### service.name / service.version consistency

- Traces: `service.name=traefik`, `service.version=3.7.0`
- Metrics (OTLP): `service.name=traefik`, `service.version=3.7.0`
- Logs: `service.name=traefik`, `service.version=3.7.0`
- **Perfectly consistent across all three signals.**

#### Rationale

Traefik v3.7.0 reaches **Level 3 — OpenTelemetry-Optimized**:

1. **Consistent identity across all signals**: `service.name: traefik` and `service.version: 3.7.0` are present and identical on traces, OTLP metrics, and logs.
2. **OTEL_* env vars respected with highest precedence**: `resource.WithFromEnv()` is explicitly placed last in all three signal providers, with source code comments confirming intentional override semantics.
3. **Documented configuration**: `tracing.serviceName`, `tracing.resourceAttributes`, and `OTEL_RESOURCE_ATTRIBUTES` are documented in the official reference docs.
4. **Stable identity across versions**: The `service.name` default is stable; `GlobalAttributes` was migrated to `ResourceAttributes` with backward-compat code.
5. **Native K8s resource detection**: Traefik's own `K8sAttributesDetector` natively emits `k8s.pod.uid`, `k8s.pod.name`, and `k8s.namespace.name`.
6. **No identity misplacement**: All identity attributes are strictly in the resource scope.

Minor gaps: `OTEL_SERVICE_NAME` is not explicitly documented (only `OTEL_RESOURCE_ATTRIBUTES`); no formal precedence table in docs; multi-tenancy guidance absent. These do not materially affect practical behavior.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure

- Total spans observed: 449
- Total root spans (no parentSpanId): 139
- Total child spans (have parentSpanId): 310
- Distinct trace IDs: 140
- Multi-span traces: 53 (38% of all traces)
- Single-span traces: 87 (62% of all traces)
- Traefik-native span kinds: SERVER=155, CLIENT=52, INTERNAL=0

##### Span names observed

- Traefik scope (`github.com/traefik/traefik`): `GET` (SERVER, entry-point), `ReverseProxy` (CLIENT, upstream call)
- Backend scope (`@opentelemetry/instrumentation-http`): `GET /`, `GET /health` (SERVER)
- Backend scope (`@opentelemetry/instrumentation-express`): `middleware - query`, `middleware - expressInit`, `middleware - jsonParser`, `middleware - <anonymous>`, `request handler - /`, `request handler - /health` (INTERNAL)

##### Trace topology

A complete end-to-end request through Traefik produces a coherent 8-span trace:

```
GET (SERVER, Traefik)                            ← entry-point root span
  └── ReverseProxy (CLIENT, Traefik)             ← upstream call to backend
        └── GET / (SERVER, backend)              ← backend receives propagated context
              ├── middleware - query (INTERNAL)
              ├── middleware - expressInit (INTERNAL)
              ├── middleware - jsonParser (INTERNAL)
              ├── middleware - <anonymous> (INTERNAL)
              └── request handler - / (INTERNAL)
```

All 50 backend trace IDs are shared with Traefik trace IDs, confirming end-to-end context propagation across service boundaries.

##### Context propagation

W3C Trace Context (`traceparent`/`tracestate`) is fully supported and verified empirically. When a request arrives with `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`, Traefik correctly:
1. Continues the trace (preserves `traceId=4bf92f3577b34da6a3ce929d0e0e4736`)
2. Creates a new child `GET` SERVER span as a child of the injected `parentSpanId=00f067aa0ba902b7`
3. Propagates the updated `traceparent` to the backend via `ReverseProxy`
4. The backend receives and continues the trace with its own child spans

The 40-span trace (`4bf92f3577b34da6a3ce929d0e0e4736`) contains 5 complete request chains, all correctly parented under the externally injected span ID `00f067aa0ba902b7`. Propagation is documented in Traefik v3 official docs as W3C Trace Context by default.

##### Span links usage

No span links observed. This is appropriate — Traefik's request model is purely synchronous and does not require cross-trace linking.

##### Trace coherence assessment

Single-span traces fall into two categories:
1. **Legitimate single-span traces**: `/ping` (Kubernetes health probes handled internally), `/metrics` (Prometheus scrape handled internally) — correctly have no downstream calls.
2. **Batching artifacts**: Some `/` path traces appear as single Traefik spans because the backend OTLP export batches arrived in different JSONL records. Shared trace IDs confirm these are part of coherent multi-service traces.

Error handling is correct: 404 responses from the backend cause `ReverseProxy` CLIENT spans to be marked `status.code=2 (ERROR)` with `http.response.status_code=404`, while the parent `GET` SERVER span remains `UNSET` (correct — the proxy itself succeeded).

#### Rationale

Traefik v3 earns **Level 2 (OpenTelemetry-Native)** for trace modeling and context propagation.

The trace structure is clearly intentional and semantically correct: entry-point spans are consistently `SERVER` kind, upstream proxy calls are `CLIENT` kind, and the parent-child hierarchy accurately represents the logical request flow. W3C Trace Context propagation is fully functional — empirically verified. The span naming (`GET`, `ReverseProxy`) represents logical operations rather than internal implementation details.

Level 3 is not awarded because there is no evidence of architectural trace review processes, explicit trade-off documentation between trace completeness and cost, or automated trace shape validation in the project's CI/CD. The `addInternals` configuration option shows some awareness of trace scope control, but falls short of the active refinement and validation expected at Level 3.

---

### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal Coverage

**Metrics — PRESENT** (via OTLP gRPC + Prometheus scrape)

| Property | Value |
|---|---|
| OTLP Endpoint | `otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317` |
| Protocol | gRPC (insecure) |
| Push Interval | 10s |
| OTel SDK | `opentelemetry-go v1.43.0` |
| Schema URL | `https://opentelemetry.io/schemas/1.40.0` |
| Batches observed | 50+ |

**OTLP-native metrics (OTel Semantic Conventions):**
- `http.server.request.duration` (Histogram, cumulative)
- `http.client.request.duration` (Histogram, cumulative)

**Prometheus-scraped Traefik metrics (15 unique metrics):**
- `traefik_entrypoint_requests_total`, `traefik_entrypoint_request_duration_seconds`
- `traefik_router_requests_total`, `traefik_router_request_duration_seconds`
- `traefik_service_requests_total`, `traefik_service_request_duration_seconds`
- `traefik_config_reloads_total`, `traefik_open_connections`, and more

**Exemplars:** ✅ Present in histogram metrics, containing Trace ID and Span ID for metrics-to-traces correlation.

**Traces — PRESENT** (via OTLP gRPC)

| Property | Value |
|---|---|
| OTLP Endpoint | `otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317` |
| Protocol | gRPC (insecure) |
| Instrumentation Scope | `github.com/traefik/traefik` (v3.7.0) |
| Sample Rate | 100% (`--tracing.sampleRate=1`) |
| Schema URL | `https://opentelemetry.io/schemas/1.40.0` |
| Batches observed | 44+ |

**Logs — PRESENT** (via OTLP gRPC, experimental feature)

| Property | Value |
|---|---|
| Feature Flag | `--experimental.otlpLogs=true` |
| OTLP Endpoint | `otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317` |
| Protocol | gRPC (insecure) |
| Instrumentation Scope | `traefik` |
| Log Type | Access logs (`--accesslog.otlp=true`) |
| Schema URL | `https://opentelemetry.io/schemas/1.40.0` |
| Batches observed | 6+ |

**Trace-Log Correlation:** ✅ Each log record has Trace ID and Span ID set at the OTLP protocol level, enabling direct correlation with traces.

##### Cross-Signal Correlation

| Correlation | Mechanism | Status |
|---|---|---|
| Metrics → Traces | Exemplars with Trace ID + Span ID in histogram data points | ✅ |
| Logs → Traces | OTLP `Trace ID` + `Span ID` fields set on every log record | ✅ |
| All signals → Resource | Identical `service.name`, `service.version`, `k8s.*`, `telemetry.sdk.*` | ✅ |

##### Notable Issues

1. **Logs are experimental**: OTLP log export requires `--experimental.otlpLogs=true` flag — not yet GA.
2. **Log body format**: Access log bodies are serialized as a JSON string rather than fully structured OTel log attributes.
3. **Dual metric paths**: Both OTLP-push (OTel semantic conventions) and Prometheus-scrape (Traefik-specific naming) are active simultaneously, providing redundancy but also duplication.
4. **Log severity quirk**: The embedded JSON body in access logs shows `"level":"panic"` — this is an internal Traefik access log formatting artifact; the OTLP SeverityText is correctly set to `info`.

##### Summary Scores

| Criterion | Result |
|---|---|
| Metrics signal present | ✅ Yes |
| Traces signal present | ✅ Yes |
| Logs signal present | ✅ Yes (experimental) |
| All signals exported via OTLP | ✅ Yes |
| Cross-signal correlation (metrics↔traces) | ✅ Yes (exemplars) |
| Cross-signal correlation (logs↔traces) | ✅ Yes (OTLP trace context) |
| Consistent resource attributes across signals | ✅ Yes |
| OTel semantic conventions followed | ✅ Yes (traces + OTLP metrics) |

#### Rationale

Traefik v3.7.0 achieves **Level 2 — OpenTelemetry-Native** for multi-signal observability. All three signals (traces, metrics, logs) are present and flowing via OTLP. Cross-signal correlation is fully functional: exemplars in histogram metrics link to specific traces, and every log record carries OTLP-level Trace ID and Span ID fields. Resource attributes are consistent across all signals, enabling unified filtering by `service.name`, `service.version`, and Kubernetes identity.

The level is capped at 2 (rather than 3) primarily because OTLP log export is behind an experimental feature gate, the log body is not fully structured, and there is attribute naming inconsistency across signals (PascalCase in logs vs. OTel semconv in traces/metrics).

---

### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming assessment

Traefik emits exactly **two span types** from its own instrumentation scope (`github.com/traefik/traefik`):

| Span Name | Kind | Count | Assessment |
|-----------|------|-------|------------|
| `GET` | Server (2) | 115 | **Neutral-to-Bad** — HTTP method alone with no path template; ambiguous without attributes |
| `ReverseProxy` | Client (3) | 44 | **Bad** — internal component name, not a user-facing operation |

- `GET` is technically OTel-conformant for HTTP server spans (the spec allows method-only names when no route template is available), but the absence of a route template (e.g. `GET /api/{resource}`) means every unique path creates a distinct-looking span only distinguishable by attributes.
- `ReverseProxy` is a Go struct/component name exposed directly as a span name. An operator unfamiliar with Traefik internals would not know this represents "outbound call to upstream backend."

##### Log verbosity

- **Total log records**: 44 (all from access logs; one per HTTP request)
- **Volume assessment**: Low — appropriate operational practice (logs fire on meaningful request-completion boundaries)
- **Attributes per record**: 36 fields including `RouterName`, `ServiceName`, `KubernetesIngressName`, `Duration`, `DownstreamStatus`, `TraceId`/`SpanId` — rich and operationally useful
- **Critical bug**: All access log records carry `level: panic` in the JSON body regardless of HTTP status — a known Traefik v3 serialization bug. This would trigger false alerts in any monitoring system watching log severity.
- **Structural quirk**: Log body is a raw JSON string; fields are duplicated as OTLP attributes but body is opaque to OTLP-native consumers without JSON parsing.

##### Metric quality

| Category | Metrics | Assessment |
|----------|---------|------------|
| **OTel semantic convention** | `http.server.request.duration`, `http.client.request.duration` | ✅ SLO-relevant — latency histograms with status, method, route |
| **Traefik-native (entrypoint)** | `traefik_entrypoint_*` (4 metrics) | ✅ SLO-relevant — per-entrypoint RED metrics |
| **Traefik-native (router)** | `traefik_router_*` (4 metrics) | ✅ SLO-relevant — per-router RED metrics |
| **Traefik-native (service)** | `traefik_service_*` (4 metrics) | ✅ SLO-relevant — per-backend-service RED metrics |
| **Traefik-native (config)** | `traefik_config_*`, `traefik_open_connections` | ✅ Operational signals |
| **Go runtime** | `go_goroutines`, `go_memstats_*`, etc. (~28 metrics) | ⚠️ Runtime internals — useful for debugging, not SLO alerting |
| **Process** | `process_cpu_seconds_total`, etc. (~9 metrics) | ⚠️ Infrastructure-level |
| **Scrape metadata** | `scrape_duration_seconds`, `up`, etc. | ℹ️ Collector/Prometheus housekeeping |

##### High-cardinality concerns

- `url.full` on `ReverseProxy` spans contains pod IPs (e.g., `http://10.244.0.6:3000/`) — high-cardinality in production with pod churn; each pod restart generates a new IP creating unbounded unique span attribute values.
- `client.port` and `network.peer.port` (ephemeral ports) present on server spans — noise with no diagnostic value.
- `http.server.request.duration` uses `server.address` as a label (includes port) — potential cardinality concern in multi-tenant or high-route environments.

#### Rationale

Traefik v3.7.0 is assessed at **Level 1 (OpenTelemetry-Aligned)**. The metric layer is genuinely strong — a well-structured three-layer hierarchy (entrypoint/router/service) covering the full RED pattern plus OTel semantic convention metrics. However, the trace and log signal quality issues prevent Level 2:

1. **`ReverseProxy` span name** — an internal Go component name used as the client span name for upstream backend calls. Requires Traefik-specific knowledge to interpret.
2. **`GET` span name without route template** — all requests share the same span name, reducing trace usability for "which route was slow" diagnostics.
3. **High-cardinality `url.full` with pod IPs** — structurally problematic in production.
4. **Ephemeral port attributes** — `client.port` and `network.peer.port` add noise without diagnostic value.
5. **`level: panic` in access logs** — critical serialization bug that would trigger false severity alerts.
6. **OTLP log export is experimental** — not production-stable.

---

### 7. Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence

- Traces: `https://opentelemetry.io/schemas/1.40.0` ✅
- Metrics (OTLP): `https://opentelemetry.io/schemas/1.40.0` ✅
- Metrics (Prometheus): `https://opentelemetry.io/schemas/1.18.0` ✅
- Logs: `https://opentelemetry.io/schemas/1.40.0` ✅

##### Telemetry documentation

A dedicated observability reference section exists at `doc.traefik.io/traefik/reference/install-configuration/observability/` with subsections for Metrics, Tracing, and Logs & AccessLogs. The metrics reference page lists all emitted metric names with types, label dimensions, and descriptions. OTel semantic convention metrics are listed with their attribute sets. The tracing reference lists span attributes and resource attributes (including K8s resource attributes auto-detected since v3.5.0).

##### Release note quality for telemetry changes

Telemetry changes are **consistently surfaced** in both the CHANGELOG and the dedicated v3 migration guide. CHANGELOG uses consistent `[metrics,otel]`, `[tracing,otel]`, `[logs,otel]` tags. Examples:

- `[metrics,otel] Rename traefik_tls_certs_not_after_milliseconds to traefik_tls_certs_not_after_seconds` (v3.5.4)
- `[metrics,otel] Change request duration metric unit from millisecond to second` (v3.3.4)
- `[tracing,otel] Use ParentBased sampler to respect parent span sampling decision` (v3.7.0-ea)
- `[logs,otel] Add OTel-conformant trace context attributes to access logs` (v3.6.11/v3.7.0-ea.2)
- `[middleware,metrics,tracing] Upgrade to OpenTelemetry Semantic Conventions v1.26.0`
- `[middleware,tracing] Introduce trace verbosity config and produce less spans by default`

**Migration guide entries** provide old/new values for every breaking change:
- v3.5.4: "Certificate Metric Renamed with OpenTelemetry" — `traefik_tls_certs_not_after_milliseconds` → `traefik_tls_certs_not_after_seconds`
- v3.5.0: "Observability / TraceVerbosity on Routers and Entrypoints" — explicit impact warning about fewer spans
- v3.3.4: "OpenTelemetry Request Duration Metric" — unit changed ms→s, old and new unit documented
- v3.2→v3.3: "Tracing Global Attributes" — `tracing.globalAttributes` renamed to `tracing.resourceAttributes`
- v2→v3: Open connections metric restructuring, vendor tracing backend removal — full migration strategies provided

##### Stable vs experimental labeling

**Partially present.** OTLP log export is explicitly gated behind `experimental.otlpLogs: true`. No systematic per-metric stable/experimental labels in reference documentation.

##### Instrumentation scope versions

| Signal | Scope | Version |
|---|---|---|
| Traces | `github.com/traefik/traefik` | **unknown** (`vunknown`) |
| Metrics | `github.com/traefik/traefik` | `v3.7.0` ✅ |

Notable discrepancy: metrics scope carries `v3.7.0` but traces scope has `vunknown` — trace instrumentation scope versioning is not fully wired up.

#### Rationale

Traefik reaches **Level 2 (OpenTelemetry-Native)** for Stability & Change Management.

**Evidence supporting Level 2:**
1. Schema URLs are present in all three signals.
2. Telemetry is treated as part of the public contract — every breaking telemetry change found across v3.x has a dedicated migration guide entry with old/new values.
3. Breaking changes are explicitly called out with named subsections in the migration guide.
4. Migration guidance is provided for every breaking telemetry change found.
5. OTLP logs are explicitly marked experimental via the feature gate.

**Why Level 3 is not awarded:**
- No formal telemetry governance process (no TEPs or design proposals for telemetry changes).
- The v3.3.4 duration unit change was immediately breaking with no deprecation period.
- Trace instrumentation scope lacks a version (`vunknown` in telemetry data).
- No evidence of proactive telemetry regression detection or signal quality evaluation frameworks.

---

## Key Findings

### Top 3 Strengths

1. **Excellent resource attribute design (Dim 3: Level 3)**: Traefik v3.7.0 demonstrates an intentional, well-governed resource attribute design. `service.name` and `service.version` are consistent across all three signals, `OTEL_RESOURCE_ATTRIBUTES` is documented and respected with highest precedence (via `resource.WithFromEnv()` placed last in the resource chain), and Traefik implements its own native `K8sAttributesDetector` that emits `k8s.pod.name`, `k8s.pod.uid`, and `k8s.namespace.name` without any Collector enrichment. This is the strongest dimension in the evaluation.

2. **Telemetry treated as a public contract (Dim 7: Level 2)**: Every breaking telemetry change across v3.x minor versions has a dedicated entry in the official migration guide with old/new values and explicit migration instructions. The CHANGELOG uses consistent `[metrics,otel]`/`[tracing,otel]`/`[logs,otel]` tags. Users are not surprised by silent metric renames or span behavior changes — they are documented proactively.

3. **Complete multi-signal coverage with full cross-signal correlation (Dim 5: Level 2)**: All three signals (traces, metrics, logs) flow via OTLP to the same collector. Exemplars in histogram metrics link to specific traces. Every log record carries OTLP-level Trace ID and Span ID. Resource attributes are consistent across all signals. The three-layer metric hierarchy (entrypoint/router/service) covers the full RED pattern and directly supports SLO alerting.

### Top 3 Areas for Improvement

1. **No standard OTel env var support for endpoint configuration (Dim 1: Level 1)**: Traefik does not respect `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, or related standard OTel SDK env vars. Users must configure three separate Traefik-specific endpoint blocks (`tracing.otlp`, `metrics.otlp`, `accesslog.otlp`) instead of a single unified OTLP endpoint. This is the most significant gap for operators familiar with the standard OTel SDK configuration model.

2. **Semantic convention inconsistency across signals (Dim 2: Level 1)**: HTTP method is `http.request.method` in traces, `method` in `traefik_*` metrics, and `RequestMethod` in logs — three different names for the same concept. The `traefik_*` metric family (the primary operational metrics for entrypoints, routers, and services) uses Prometheus-style shorthand attribute names (`code`, `method`, `protocol`) instead of OTel semconv names. Access log attributes are entirely proprietary PascalCase names. This prevents generic OTel tooling from correlating across signals without custom mapping.

3. **Signal quality issues in traces and logs (Dim 6: Level 1)**: The `ReverseProxy` span name exposes an internal Go component name, requiring Traefik-specific knowledge to interpret. The `level: panic` serialization bug in access log records would trigger false severity alerts in production monitoring systems. The `url.full` attribute on `ReverseProxy` spans contains pod IPs, creating high-cardinality issues in production. OTLP log export remains behind an experimental feature gate. These issues collectively reduce the out-of-the-box usefulness of the telemetry for operators.

### Notable Observations

- **Dual metrics paths are a double-edged sword**: Having both Prometheus scrape and OTLP push active simultaneously provides redundancy and enables gradual migration, but creates metric duplication and naming inconsistency. The OTLP push path provides OTel semconv metrics (`http.server.request.duration`) that the Prometheus path does not, but the `traefik_*` metrics on both paths use non-OTel attribute names. Operators must understand both paths to use the telemetry effectively.

- **Traefik is ahead of its OTel maturity score on resource attributes**: The Level 3 score on Dimension 3 reflects that Traefik's resource attribute design is genuinely excellent — better than most CNCF projects. The native `K8sAttributesDetector`, consistent `service.name`/`service.version` across all signals, and documented `OTEL_RESOURCE_ATTRIBUTES` support are all production-grade behaviors.

- **The experimental log gate is the right call**: Marking OTLP log export as experimental (`experimental.otlpLogs: true`) is the correct stability signal — it prevents users from depending on an unstable API. However, the `level: panic` serialization bug in the experimental log records suggests the experimental label is warranted and should not be removed until the bug is fixed.

- **Trace scope version inconsistency**: The `github.com/traefik/traefik` instrumentation scope carries `v3.7.0` in metrics but `vunknown` in traces. This is a minor but notable inconsistency that affects tooling that uses scope versions for filtering or routing.

- **W3C Trace Context propagation is production-ready**: The empirical verification (injecting `traceparent` and confirming end-to-end trace ID preservation across Traefik → backend) is one of the strongest signals in the evaluation. Zero context loss across 50 backend spans is a meaningful quality indicator.

---

## Methodology Notes

- **Evaluation run**: v1
- **Cluster**: `kind-otel-eval-traefik` (kind v0.x, Kubernetes 1.x)
- **Traefik version**: `v3.7.0` (Helm chart `traefik/traefik` v40.0.0)
- **OTel Collector version**: `open-telemetry/opentelemetry-collector` v0.150.1 (contrib)
- **Telemetry collection**: JSONL files written by OTel Collector file exporter to `/tmp/otel-eval-traefik/` on the kind node
- **Telemetry files analyzed**:
  - `traces.jsonl` — 337KB, 29 JSONL lines, 449 total spans
  - `metrics.jsonl` — 2.7MB, 26+ batches, 77 unique metric names
  - `logs.jsonl` — 202KB, 5-6 batches, 44+ log records
- **Traffic generation**: Synthetic curl traffic via port-forward (10 normal requests, 5 with injected `traceparent`, 5 health checks, 3 404 requests)
- **Source code reviewed**: `pkg/observability/types/tracing.go`, `pkg/observability/metrics/otel.go`, `pkg/observability/types/logs.go`, `pkg/types/k8sdetector.go`
- **Documentation reviewed**: `doc.traefik.io/traefik/` (observability reference, migration guides v2→v3 and v3.x minor), `CHANGELOG.md` on GitHub
- **Level scale**: 0 (Instrumented) → 1 (OpenTelemetry-Aligned) → 2 (OpenTelemetry-Native) → 3 (OpenTelemetry-Optimized)
- **Dimensions evaluated**: 7 (Integration Surface, Semantic Conventions, Resource Attributes & Configuration, Trace Modeling & Context Propagation, Multi-Signal Observability, Audience & Signal Quality, Stability & Change Management)
