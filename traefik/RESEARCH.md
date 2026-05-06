# Traefik Research Notes

## What is Traefik?

Traefik (pronounced "traffic") is a modern HTTP reverse proxy and load balancer designed for deploying microservices. It is a CNCF graduated project and serves as a Kubernetes Ingress Controller, supporting:
- Kubernetes Ingress (networking.k8s.io/v1)
- Kubernetes Gateway API (gateway.networking.k8s.io)
- Native Traefik IngressRoute CRDs

It automatically discovers services and routes traffic to them. Traefik v3 (current) has deep native OpenTelemetry support for all three signal types.

## Installation

- **Method:** Helm chart from `https://traefik.github.io/charts`
- **Chart:** `traefik/traefik`
- **Latest version:** 39.0.9 (App version: v3.6.15)
- **Namespace:** `traefik`

## Telemetry Capabilities

### Traces
- **Native OTLP support** via `tracing.otlp` in Helm values
- Supports both gRPC (`tracing.otlp.grpc`) and HTTP (`tracing.otlp.http`)
- Traces HTTP requests flowing through Traefik routers/services
- W3C Trace Context propagation supported natively
- Configurable: `addInternals`, `capturedRequestHeaders`, `capturedResponseHeaders`, `sampleRate`
- Verbosity levels per entrypoint: `minimal` (default) or `detailed`

### Metrics
- **Prometheus endpoint** (default, port 9100 via `metrics` entrypoint)
- **Native OTLP metrics** via `metrics.otlp` (push-based, default interval 10s)
- Supports both gRPC and HTTP OTLP export
- Metrics include: entry points, routers, services
- Labels: `addEntryPointsLabels`, `addRoutersLabels`, `addServicesLabels`

### Logs
- **Structured general logs** (JSON format available via `logs.general.format: json`)
- **Access logs** (JSON format available via `logs.access.format: json`)
- **OTLP logs** (experimental feature, requires `experimental.otlpLogs: true`)
  - Both general logs and access logs can be sent via OTLP
  - Supports gRPC and HTTP OTLP export

## OpenTelemetry Configuration

All three signals can be configured via Helm values:

```yaml
# Traces
tracing:
  otlp:
    enabled: true
    grpc:
      enabled: true
      endpoint: "<host>:<port>"
      insecure: true

# Metrics
metrics:
  otlp:
    enabled: true
    grpc:
      enabled: true
      endpoint: "<host>:<port>"
      insecure: true

# Logs (experimental)
experimental:
  otlpLogs: true
logs:
  general:
    otlp:
      enabled: true
      grpc:
        enabled: true
        endpoint: "<host>:<port>"
        insecure: true
  access:
    enabled: true
    otlp:
      enabled: true
      grpc:
        enabled: true
        endpoint: "<host>:<port>"
        insecure: true
```

## Context Propagation

- Traefik v3 natively supports **W3C Trace Context** (traceparent/tracestate headers)
- When tracing is enabled, Traefik injects/propagates trace context to upstream services
- The `capturedRequestHeaders` / `capturedResponseHeaders` can add headers as span attributes

## Special Setup Requirements

- Traefik installs CRDs (IngressRoute, Middleware, etc.) — handled by Helm
- Gateway API CRDs may be needed if using Gateway API provider
- The `web` entrypoint (port 80) is exposed by default for HTTP traffic
- The `metrics` entrypoint (port 9100) serves Prometheus metrics
- The `traefik` entrypoint (port 8080) serves the dashboard/API

## Routing to Test Backend

Use Kubernetes Ingress (simplest approach) or Traefik IngressRoute CRD:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: otel-eval-backend
  namespace: demo
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: otel-eval-backend
            port:
              number: 3000
```

## Actual Observations (Post-Install)

### Installation

Traefik v3.6.15 (chart 39.0.9) installed cleanly via Helm into the `traefik` namespace. All configuration applied correctly via Helm values — no manual patching required.

### Traces — FLOWING ✅

- **Protocol:** OTLP gRPC, pushed to collector at `:4317`
- **Service name:** `traefik` (set natively in resource attributes)
- **SDK:** `opentelemetry` / `go` / `1.43.0` — Traefik embeds the OTel Go SDK directly
- **Resource attributes set natively:** `service.name`, `service.version`, `telemetry.sdk.name`, `telemetry.sdk.language`, `telemetry.sdk.version`, `process.*`, `os.*`, `host.name`
- **Span structure per HTTP request:**
  - Root span: `GET` (entry point span, kind=SERVER) with attributes: `entry_point`, `http.request.method`, `url.path`, `url.scheme`, `http.response.status_code`, `client.address`, `network.peer.address`, `user_agent.original`
  - Child span: `ReverseProxy` (outgoing proxy request to backend)
- **W3C Trace Context propagation confirmed:** Traefik injects `traceparent` header into upstream requests. Backend spans appear as children of Traefik's `ReverseProxy` span — full distributed trace confirmed.
- **`addInternals: true`** also traces internal health check requests (ping endpoint on `traefik` entrypoint)
- **Captured request headers** (`traceparent`, `tracestate`, `x-forwarded-for`) appear as span attributes

### Metrics — FLOWING ✅

- **Protocol:** OTLP gRPC push every 10s to collector at `:4317`
- **Service name:** `traefik` (set natively)
- **Metric names observed:**
  - `traefik_entrypoint_requests_total` — request count per entry point
  - `traefik_entrypoint_request_duration_seconds` — latency histogram per entry point
  - `traefik_entrypoint_requests_bytes_total` / `traefik_entrypoint_responses_bytes_total`
  - `traefik_router_requests_total` / `traefik_router_request_duration_seconds`
  - `traefik_router_requests_bytes_total` / `traefik_router_responses_bytes_total`
  - `traefik_service_requests_total` / `traefik_service_request_duration_seconds`
  - `traefik_service_requests_bytes_total` / `traefik_service_responses_bytes_total`
  - `traefik_open_connections`
  - `traefik_config_reloads_total` / `traefik_config_last_reload_success`
  - `http.server.request.duration` (OTel semantic convention metric)
  - `http.client.request.duration` (OTel semantic convention metric)
- **Note:** Traefik emits **both** traditional `traefik_*` named metrics AND the standard OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`)
- Prometheus endpoint also available at `:9100/metrics` (kept for reference)

### Logs — FLOWING ✅ (experimental)

- **Protocol:** OTLP gRPC push to collector at `:4317`
- **Service name:** `traefik` (set natively)
- **Two log streams:**
  1. **General logs** — Traefik daemon/configuration logs (e.g., "No domain found in rule..."), structured with `severityText` and log attributes
  2. **Access logs** — Per-request HTTP access logs sent as OTLP log records, with rich structured attributes including: `RequestMethod`, `RequestPath`, `DownstreamStatus`, `Duration`, `RouterName`, `ServiceName`, `ServiceAddr`, `TraceId`, `SpanId`, `entryPointName`, etc.
- **Access logs include trace/span IDs** — correlating logs to traces is possible natively
- **Note:** OTLP log export is an **experimental feature** in Traefik v3, requiring `experimental.otlpLogs: true`. The `severityText` field for access logs is incorrectly set to `"panic"` (a known quirk of the experimental implementation — the log level mapping is not yet correct for access logs).

### Context Propagation — CONFIRMED ✅

- Traefik correctly propagates **W3C Trace Context** (`traceparent` header) to upstream services
- When an incoming request has a `traceparent` header, Traefik uses the same trace ID and creates a child span
- When no `traceparent` is present, Traefik starts a new trace and injects `traceparent` into the upstream request
- Full distributed trace topology confirmed: `GET (traefik)` → `ReverseProxy (traefik)` → `GET / (otel-eval-backend)`

### Surprises / Deviations from Documentation

1. **OTLP logs are experimental** but worked correctly in v3.6.15 with `experimental.otlpLogs: true`
2. **Access log severity is incorrectly set to "panic"** — this is a known quirk of the experimental OTLP log implementation; the log body contains the correct level field
3. **Traefik emits OTel semantic convention metrics** (`http.server.request.duration`, `http.client.request.duration`) in addition to its traditional `traefik_*` metrics — this is a positive finding showing alignment with OTel conventions
4. **`service.version`** is set natively in resource attributes (e.g., `"3.6.15"`) — excellent practice
5. **The Ingress annotation** `kubernetes.io/ingress.class` is deprecated; use `spec.ingressClassName: traefik` in production

### Native vs. Collector-Derived Attributes

**Project-native (set by Traefik itself):**
- `service.name`, `service.version`
- `telemetry.sdk.name`, `telemetry.sdk.language`, `telemetry.sdk.version`
- `process.executable.name`, `process.executable.path`, `process.pid`, `process.owner`, `process.runtime.*`, `process.command_args`
- `os.type`, `os.description`
- `host.name`
- All span attributes: `entry_point`, `http.request.method`, `url.*`, `http.response.status_code`, `client.address`, `network.peer.*`, `user_agent.original`

**Collector-derived (added by k8sattributes processor):**
- `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.namespace.name`, `k8s.node.name`
- `k8s.deployment.name`, `k8s.replicaset.name`
- `k8s.container.name`
- `k8s.pod.label.*`, `k8s.pod.annotation.*`
