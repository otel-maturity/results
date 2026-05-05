# Traefik Installation Plan

## Project Overview

**Traefik** is a cloud-native edge router and Kubernetes Ingress controller. It automatically discovers services and routes HTTP/TCP/UDP traffic using Kubernetes Ingress, IngressRoute CRDs, or the Gateway API. It is a graduated CNCF project.

## Installation Method

| Field | Value |
|-------|-------|
| Method | Helm |
| Chart | `traefik/traefik` |
| Repo URL | https://traefik.github.io/charts |
| Chart Version | `39.0.9` |
| App Version | `v3.6.15` |
| Namespace | `traefik` |
| Release Name | `traefik` |

## Telemetry Configuration

Traefik v3 has **native OTLP support** for all three signals. All signals are sent via **OTLP gRPC** to the in-cluster OTel Collector.

### Traces
- Enable via `tracing.otlp.enabled: true` + `tracing.otlp.grpc.enabled: true`
- Endpoint: `otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317`
- Insecure (no TLS) since it's in-cluster
- `tracing.addInternals: true` to capture internal router/middleware spans

### Metrics
- **OTLP push** via `metrics.otlp.enabled: true` + `metrics.otlp.grpc.enabled: true` (same endpoint)
- **Prometheus scrape** also kept enabled (default) for comparison
  - Prometheus metrics exposed on port 9100 (`metrics` entrypoint)
  - Moved from default 8080 to avoid conflict with Traefik dashboard entrypoint
- Router and service labels enabled: `addRoutersLabels: true`, `addServicesLabels: true`

### Logs
- **General logs** (operational): `logs.general.otlp.enabled: true` + `grpc.enabled: true`
- **Access logs** (per-request): `logs.access.otlp.enabled: true` + `grpc.enabled: true`; format set to JSON
- Requires `experimental.otlpLogs: true` to enable OTLP log export

> **Important**: The Helm chart requires both a top-level `otlp.enabled: true` AND the protocol-level `grpc.enabled: true`. Without the top-level flag, the chart template silently omits the OTLP CLI args.

## Collector Changes

The collector already has OTLP gRPC/HTTP receivers enabled. Added a Prometheus scrape config for Traefik's `/metrics` endpoint:

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: traefik
          scrape_interval: 15s
          static_configs:
            - targets:
                - traefik-metrics.traefik.svc.cluster.local:9100
```

The `prometheus` receiver was added to the metrics pipeline.

## Routing / Ingress Setup

Traefik is the default IngressClass. A Kubernetes `Ingress` resource routes traffic to `otel-eval-backend`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: otel-eval-backend
  namespace: demo
spec:
  ingressClassName: traefik
  rules:
  - host: otel-eval-backend.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: otel-eval-backend
            port:
              number: 3000
```

## Traffic Generation

```bash
# Port-forward Traefik web entrypoint
kubectl port-forward -n traefik svc/traefik 8888:80 &

# Without trace context (Traefik starts new trace)
curl -s -H "Host: otel-eval-backend.local" http://localhost:8888/
curl -s -H "Host: otel-eval-backend.local" http://localhost:8888/health

# With W3C traceparent (Traefik propagates)
curl -s -H "Host: otel-eval-backend.local" \
     -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
     http://localhost:8888/

# Unknown host (Traefik 404)
curl -s http://localhost:8888/
```

## Steps Executed

1. `kubectl create namespace traefik`
2. First install attempt failed — port conflict: Traefik dashboard (`traefik` entrypoint) and `metrics` entrypoint both default to port 8080. Fixed by setting `ports.metrics.port: 9100`.
3. Second install attempt failed — OTLP args not generated. Root cause: Helm chart requires top-level `otlp.enabled: true` in addition to `grpc.enabled: true`. Fixed by adding `enabled: true` to all OTLP sections.
4. `helm upgrade --install traefik traefik/traefik --version 39.0.9 --namespace traefik -f .otel-eval/traefik/traefik/values.yaml --wait`
5. Applied `traefik/routing/ingress.yaml` to route traffic to `otel-eval-backend`
6. Updated `collector/values.yaml` with Prometheus scrape target and upgraded collector
7. Port-forwarded Traefik and generated 40+ requests (with and without traceparent)
8. Verified all three telemetry signals flowing

## Telemetry Status

| Signal | Source | Method | Status |
|--------|--------|--------|--------|
| Traces | Traefik (per-request spans) | OTLP gRPC | Flowing |
| Metrics | Traefik (entrypoint/router/service metrics) | OTLP gRPC push | Flowing |
| Metrics | Traefik `/metrics` endpoint | Prometheus scrape | Flowing |
| Logs | Traefik general logs | OTLP gRPC | Flowing |
| Logs | Traefik access logs (per-request) | OTLP gRPC | Flowing |

## Telemetry Files

```
/tmp/otel-eval-traefik/traces.jsonl   — Traefik + backend spans
/tmp/otel-eval-traefik/metrics.jsonl  — Traefik OTLP + Prometheus + k8s_cluster
/tmp/otel-eval-traefik/logs.jsonl     — Traefik general + access logs
```

## Access

```bash
# Port-forward to Traefik
kubectl port-forward -n traefik svc/traefik 8888:80

# Send requests through Traefik to backend
curl -H "Host: otel-eval-backend.local" http://localhost:8888/
```

## Key Findings

- **Full distributed tracing**: Traefik creates a root `GET` server span and a `ReverseProxy` client span. The backend's server span has `parentSpanId` = Traefik's `ReverseProxy` span, confirming W3C Trace Context propagation.
- **Dual metrics export**: Traefik emits the same `traefik_*` metric names via both OTLP push and Prometheus scrape. OTLP metrics have richer resource attributes (host, process, runtime info).
- **Access log correlation**: Access logs have `traceId` and `spanId` set, linking them to the trace for the same request.
- **Experimental OTLP logs**: OTLP log export requires `experimental.otlpLogs: true` — it is still an experimental feature in Traefik v3.
