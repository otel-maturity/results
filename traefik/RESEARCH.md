# Traefik Research Notes

## What is Traefik?

Traefik is a cloud-native edge router / reverse proxy and Kubernetes Ingress controller. It automatically discovers services and routes traffic to them based on rules defined via Kubernetes Ingress, IngressRoute CRDs, or the Gateway API. It is a graduated CNCF project.

## Installation

- **Method**: Helm chart (`traefik/traefik`)
- **Helm repo**: https://traefik.github.io/charts
- **Latest chart version**: 39.0.9 (app version: v3.6.15)
- **Recommended namespace**: `traefik`

## Telemetry Capabilities

### Traces
- Traefik v3 supports **OTLP tracing natively** (no Jaeger/Zipkin dependency needed)
- Configured via `tracing.otlp.grpc` or `tracing.otlp.http` in Helm values
- Traces each incoming request as a server span, plus upstream calls as client spans
- Supports capturing request/response headers as span attributes
- Sample rate configurable (default 1.0)
- `tracing.addInternals: true` adds spans for internal Traefik routers/middlewares

### Metrics
- **Prometheus** metrics endpoint enabled by default on port 8080 (metrics entrypoint)
- Also supports **OTLP push** for metrics via `metrics.otlp.grpc` or `metrics.otlp.http`
- Metrics include: entry point metrics, router metrics, service metrics
- Labels: entrypoint name, router name, service name, method, status code

### Logs
- **General logs**: Traefik operational logs (startup, config reload, errors)
  - Can be sent via OTLP with `logs.general.otlp` (requires `experimental.otlpLogs: true`)
- **Access logs**: Per-request HTTP access logs
  - Can be sent via OTLP with `logs.access.otlp` (requires `experimental.otlpLogs: true`)
  - Access logs can be JSON-formatted
  - Fields are highly configurable (keep/drop/redact)

### Context Propagation
- Traefik v3 supports **W3C Trace Context** (traceparent/tracestate headers)
- Also supports B3 propagation
- Will propagate incoming trace context to upstream services

## OpenTelemetry Configuration

All OTel config is done via Helm values:

```yaml
tracing:
  otlp:
    grpc:
      enabled: true
      endpoint: "<host>:<port>"
      insecure: true

metrics:
  otlp:
    grpc:
      enabled: true
      endpoint: "<host>:<port>"
      insecure: true

logs:
  general:
    otlp:
      grpc:
        enabled: true
        endpoint: "<host>:<port>"
        insecure: true
  access:
    otlp:
      grpc:
        enabled: true
        endpoint: "<host>:<port>"
        insecure: true

experimental:
  otlpLogs: true  # Required for OTLP log export
```

## Special Setup Notes

- Installs CRDs for `IngressRoute`, `Middleware`, `TraefikService`, etc.
- By default creates an `IngressClass` named `traefik` (set as default)
- Prometheus metrics are enabled by default on the `metrics` entrypoint (port 8080)
- OTLP logs export requires enabling `experimental.otlpLogs: true`
- The `web` entrypoint listens on port 8000 internally (exposed as port 80 on the service)
- Access via `kubectl port-forward` to the Traefik service on port 80

## Routing Setup

Traffic can be routed through Traefik to the test backend using a standard Kubernetes `Ingress` resource (since Traefik is the default IngressClass) or via a `IngressRoute` CRD.

## Actual Observations

### Installation Notes

- **Chart version**: 39.0.9 (app v3.6.15)
- **Gotcha**: The Traefik `traefik` entrypoint defaults to port 8080, which conflicts with the `metrics` entrypoint default (also 8080). Fixed by moving metrics to port 9100.
- **Gotcha**: The Helm chart requires `tracing.otlp.enabled: true`, `metrics.otlp.enabled: true`, `logs.general.otlp.enabled: true`, and `logs.access.otlp.enabled: true` at the top level in addition to `grpc.enabled: true`. Without the top-level `enabled: true`, the chart template silently omits the OTLP CLI args.

### Telemetry Signals Actually Flowing

#### Traces (OTLP gRPC) — Flowing ✓
- **service.name**: `traefik`
- **telemetry.sdk.name**: `opentelemetry`
- **telemetry.sdk.language**: `go`
- **telemetry.sdk.version**: `1.43.0`
- **service.version**: `3.6.15`
- **Span names**: `GET` (server span, entry point), `ReverseProxy` (client span, upstream call)
- **Span attributes** (project-native): `entry_point`, `http.request.method`, `url.path`, `url.scheme`, `url.query`, `user_agent.original`, `server.address`, `network.peer.address`, `client.address`, `client.port`, `http.response.status_code`, `network.protocol.version`, `http.request.body.size`
- **Context propagation**: W3C Trace Context confirmed — Traefik creates root span (`GET`), then client span (`ReverseProxy`), and the backend receives the correct `parentSpanId` pointing to `ReverseProxy`.
- **Resource attributes** (project-native): `host.name`, `os.type`, `os.description`, `process.executable.name`, `process.executable.path`, `process.owner`, `process.pid`, `process.command_args`, `process.runtime.name`, `process.runtime.version`, `process.runtime.description`
- **Resource attributes** (collector-enriched via k8sattributes): `k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.replicaset.name`, `k8s.deployment.name`, `k8s.node.name`, `k8s.container.name`, `k8s.pod.label.*`, `k8s.pod.annotation.*`, `k8s.pod.start_time`

#### Metrics — Flowing ✓ (both OTLP push AND Prometheus scrape)

**OTLP push metrics** (service.name=traefik, no server.address attribute):
- Same `traefik_*` metric names as Prometheus (Traefik uses the same metric names for both)
- Resource attributes: same as traces (host.name, process.*, os.*, service.*)

**Prometheus scrape metrics** (service.name=traefik, server.address=traefik-metrics.traefik.svc.cluster.local):
- `traefik_config_last_reload_success`
- `traefik_config_reloads_total`
- `traefik_entrypoint_request_duration_seconds`
- `traefik_entrypoint_requests_bytes_total`
- `traefik_entrypoint_requests_total`
- `traefik_entrypoint_responses_bytes_total`
- `traefik_open_connections`
- `traefik_router_request_duration_seconds`
- `traefik_router_requests_bytes_total`
- `traefik_router_requests_total`
- `traefik_router_responses_bytes_total`
- `traefik_service_request_duration_seconds`
- `traefik_service_requests_bytes_total`
- `traefik_service_requests_total`
- `traefik_service_responses_bytes_total`
- Plus Go runtime metrics (`go_*`) and process metrics

#### Logs (OTLP gRPC) — Flowing ✓

**General logs** (operational):
- service.name=traefik, telemetry.sdk.name=opentelemetry
- Plain string body (Traefik log messages)
- No traceId/spanId on general logs

**Access logs** (per-request):
- JSON body with full request details (RouterName, ServiceName, Duration, Status, etc.)
- **traceId and spanId are set** on access log records — correlates with traces
- Attributes include: SpanId, StartLocal, ClientHost, OriginStatus, RequestMethod, etc.
- Access logs include `TraceId` and `SpanId` fields in the JSON body as well

### Surprises / Deviations from Documentation

1. **Dual `enabled` flags required**: The Helm chart has both a top-level `otlp.enabled` and a `grpc.enabled` / `http.enabled`. Both must be set to `true`. The documentation examples often only show the inner flag.
2. **OTLP metrics use Prometheus metric names**: Traefik's OTLP metrics export uses the same `traefik_*` naming convention as Prometheus, not the OTel semantic convention naming (e.g., `traefik.entrypoint.requests`). This is a deliberate design choice.
3. **Access logs body is JSON string, not structured**: The access log body is a JSON-serialized string, not a structured log body. The fields are also available as log record attributes.
4. **Experimental flag required for OTLP logs**: `experimental.otlpLogs: true` must be set separately from the log OTLP config itself.
