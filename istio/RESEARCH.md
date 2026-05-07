# Istio — Research Notes

## What is Istio?

Istio is an open-source service mesh that provides a uniform way to secure, connect, and observe microservices. It operates by injecting an Envoy proxy sidecar into each application pod. The sidecar intercepts all inbound and outbound traffic, giving Istio control over routing, load balancing, retries, circuit breaking, mTLS, and telemetry — all without changes to application code.

Key components:
- **istiod**: The control plane (Pilot, Citadel, Galley merged into one). Manages configuration, certificate issuance, and service discovery.
- **Envoy sidecar (istio-proxy)**: The data plane. Injected into every application pod. Generates all telemetry.
- **Ingress Gateway**: An Envoy-based ingress controller for external traffic.

## Installation Method

**Official method**: Helm charts from `https://istio-release.storage.googleapis.com/charts`

Three charts installed in order:
1. `istio/base` — CRDs and cluster-scoped resources
2. `istio/istiod` — Control plane (istiod)
3. `istio/gateway` — Ingress gateway (optional, but needed for traffic routing)

**Latest stable version**: `1.29.2`

**Namespace**: `istio-system` (standard)

## Telemetry Produced

### Traces
- **Protocol**: OTLP (gRPC, port 4317) — native support via `extensionProviders` in meshConfig
- **Mechanism**: Envoy sidecars emit spans for every request they handle (inbound + outbound). The istiod control plane configures the sidecars with tracing settings.
- **Configuration**: Two-step process:
  1. Configure `meshConfig.extensionProviders` in istiod to point to an OTel collector
  2. Apply a `Telemetry` CR (telemetry.istio.io/v1) to enable tracing with a sampling rate
- **Sampling**: Configurable via `randomSamplingPercentage` (0–100) in the Telemetry CR
- **Propagation**: W3C Trace Context (traceparent/tracestate) — the default for the OTel provider

### Metrics
- **Protocol**: Prometheus (pull-based) — no native OTLP metrics export
- **Endpoints**:
  - `istiod`: port 15014 (`/metrics`) — control plane metrics
  - Each Envoy sidecar: port 15090 (`/stats/prometheus`) — per-pod data plane metrics
  - Ingress gateway: port 15020 (`/stats/prometheus`) — gateway metrics
- **Key metric families**: `istio_requests_total`, `istio_request_duration_milliseconds`, `istio_request_bytes`, `istio_response_bytes`, `istio_tcp_*`, `pilot_*`, `citadel_*`
- **enablePrometheusMerge**: When enabled (default), istiod merges app metrics with Envoy metrics on port 15020

### Logs (Access Logs)
- **Protocol**: OTLP (gRPC) — via the Access Log Service (ALS) extension provider
- **Mechanism**: Envoy sidecars can emit access logs as OTLP LogRecords for every request
- **Configuration**: Configure `extensionProviders` with `envoyOtelAls` type, then apply a Telemetry CR with `accessLogging` enabled
- **Default**: Access logs are disabled by default in Istio 1.x (they were enabled in older versions)

## OpenTelemetry Configuration

Istio's OTel integration is configured via **meshConfig** in the istiod Helm values:

```yaml
meshConfig:
  extensionProviders:
  - name: otel-tracing
    opentelemetry:
      port: 4317
      service: <otel-collector-svc>.<namespace>.svc.cluster.local
  - name: otel-als
    envoyOtelAls:
      port: 4317
      service: <otel-collector-svc>.<namespace>.svc.cluster.local
```

Then activated by a `Telemetry` CR (can be namespace- or mesh-scoped):

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system   # mesh-wide when in istio-system
spec:
  tracing:
  - providers:
    - name: otel-tracing
    randomSamplingPercentage: 100
  accessLogging:
  - providers:
    - name: otel-als
```

## Context Propagation

- **Default for OTel provider**: W3C Trace Context (`traceparent`, `tracestate`)
- Also supports B3 (single/multi header) via Zipkin provider
- Envoy propagates headers automatically; no application changes needed

## Special Setup Requirements

1. **Sidecar injection**: Namespaces must be labeled `istio-injection: enabled` for automatic sidecar injection
2. **CRDs**: Installed via `istio/base` chart
3. **OTel collector service port naming**: The gRPC port on the collector service must be named with a `grpc-` prefix (e.g., `grpc-otlp`) for Istio's Envoy proxies to use HTTP/2 — otherwise traces may silently fail
4. **Kind cluster**: No special requirements; Istio works on kind with standard CNI

## Prometheus Scraping

For the OTel collector to scrape Istio's Prometheus metrics, the following scrape configs are needed:

- **istiod**: `istio-system` namespace, port 15014, path `/metrics`
- **Envoy sidecars**: All pods with `istio-proxy` container, port 15090, path `/stats/prometheus`
- **Ingress gateway**: `istio-system` namespace, port 15020, path `/stats/prometheus`

The collector's Prometheus receiver needs these scrape configs added.

## Key Observations from Docs

- Istio's Envoy-based tracing is **project-native** (no app instrumentation needed)
- Metrics are Prometheus-only — OTLP metrics export is not supported natively
- Access logs via OTLP ALS are a relatively new feature (stable in 1.18+)
- The `Telemetry` API (telemetry.istio.io/v1) is the modern way to configure telemetry (replaces older EnvoyFilter/MeshConfig approaches)
- Sampling at 100% is needed in a test environment to capture all traces

## Actual Observations (Post-Installation v1.29.2)

### Traces — FLOWING (OTLP gRPC)
- **SDK**: `telemetry.sdk.name=envoy`, `telemetry.sdk.language=cpp` — **project-native**, not OTel SDK
- **Service names**: Envoy sets `service.name` as `<workload>.<namespace>` (e.g., `otel-eval-backend.demo`, `istio-gateway.istio-system`)
- **Span names**: Envoy uses URL-pattern style names like `otel-eval-backend.demo.svc.cluster.local:3000/*` for inbound, and `router outbound|...; egress` for outbound
- **W3C Trace Context**: Confirmed — Istio injects `traceparent` header into requests flowing through the mesh, visible in backend response
- **Span kinds**: OTLP span kinds present (SERVER for inbound, CLIENT for outbound)
- **100% sampling**: Working correctly with `randomSamplingPercentage: 100`

### Metrics — FLOWING (Prometheus scrape)
- **istio_requests_total**: ✅ Present (labeled with source/destination workload, namespace, method, response_code)
- **istio_request_duration_milliseconds**: ✅ Present (histogram)
- **istio_request_bytes / istio_response_bytes**: ✅ Present
- **istio_tcp_***: ✅ Present
- **istio_agent_***: ✅ Present (from istiod sidecar agent)
- **istio_build**: ✅ Present
- **envoy_cluster_*, envoy_listener_*, envoy_server_***: ✅ Present (raw Envoy internal metrics)
- **pilot_* metrics**: Not scraped in this config (istiod port 15014 scrape only captured istio_agent_* metrics from the sidecar agent, not the pilot control plane metrics directly)
- **Note**: All metrics are Prometheus-scraped — no OTLP metrics export exists natively

### Logs — FLOWING (OTLP gRPC via Envoy ALS)
- **Format**: Envoy access log format as string body: `[timestamp] "METHOD PATH PROTOCOL" STATUS_CODE ... "traceparent_header" ...`
- **Trace correlation**: Log records include `traceId` and `spanId` fields — **full trace-log correlation**
- **Resource**: Same resource attributes as traces (pod labels, namespace, k8s metadata enriched by collector)
- **Scope**: Empty scope (no instrumentation library name/version for ALS)

### Context Propagation
- W3C Trace Context confirmed: `traceparent` header injected by Envoy gateway sidecar, propagated to backend

### Native vs Collector-Derived
- **Project-native**: `service.name`, `telemetry.sdk.name=envoy`, `telemetry.sdk.version`, `telemetry.sdk.language=cpp`, span names, trace/log correlation IDs
- **Collector-derived (k8sattributes)**: `k8s.pod.name`, `k8s.namespace.name`, `k8s.deployment.name`, `k8s.pod.uid`, `k8s.node.name`, pod labels/annotations

### Surprises
- The Envoy sidecar in the demo pod also generated spans for its outbound call to the OTel collector itself (`otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318/*`) — this is because the collector is also in the mesh path
- `service.name` format differs between gateway (`istio-gateway.istio-system`) and sidecar (`otel-eval-backend.demo`) — namespace is appended
- The `otel-eval-backend` service name also appeared without namespace suffix in some spans — inconsistency in how Envoy sets service.name depending on the context
