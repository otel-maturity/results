# Traefik Install Plan

## Project Overview

**Traefik** is a CNCF graduated cloud-native HTTP reverse proxy and Kubernetes Ingress Controller. It automatically discovers services and routes traffic to them. Traefik v3 (current) has first-class, native OpenTelemetry support for all three telemetry signals (traces, metrics, logs) via OTLP.

---

## 1. Installation Method

| Field         | Value                                  |
|---------------|----------------------------------------|
| Helm repo     | `https://traefik.github.io/charts`     |
| Chart         | `traefik/traefik`                      |
| Chart version | `39.0.9`                               |
| App version   | `v3.6.15`                              |
| Namespace     | `traefik`                              |
| Release name  | `traefik`                              |
| Values file   | `.otel-eval/traefik/traefik/values.yaml` |

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update traefik
kubectl create namespace traefik
helm upgrade --install traefik traefik/traefik \
    --namespace traefik \
    --version 39.0.9 \
    -f .otel-eval/traefik/traefik/values.yaml \
    --wait --timeout 5m
```

---

## 2. Telemetry Configuration

Traefik v3 supports **native OTLP export** for all three signals. All are configured to push via **OTLP gRPC** to the in-cluster OTel Collector.

### Collector endpoint

```
otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317
```

### Traces (OTLP gRPC)

```yaml
tracing:
  addInternals: true
  capturedRequestHeaders: [traceparent, tracestate, x-forwarded-for]
  sampleRate: 1.0
  otlp:
    enabled: true
    grpc:
      enabled: true
      endpoint: "otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317"
      insecure: true
```

- Traefik creates a **server span** for each routed HTTP request
- W3C Trace Context headers (`traceparent` / `tracestate`) are propagated to upstream services
- `addInternals: true` also traces Traefik-internal requests (health checks, dashboard)

### Metrics (OTLP gRPC push + Prometheus scrape)

```yaml
metrics:
  addInternals: true
  prometheus:
    entryPoint: metrics
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
  otlp:
    enabled: true
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
    pushInterval: "10s"
    grpc:
      enabled: true
      endpoint: "otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317"
      insecure: true
```

- Traefik **pushes OTLP metrics** every 10 seconds
- Prometheus endpoint also available at `:9100/metrics` (kept for reference)
- Metrics cover: entry points, routers, services (request counts, latencies, etc.)

### Logs (OTLP gRPC — experimental)

```yaml
experimental:
  otlpLogs: true

logs:
  general:
    format: json
    otlp:
      enabled: true
      grpc:
        enabled: true
        endpoint: "otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317"
        insecure: true
  access:
    enabled: true
    format: json
    otlp:
      enabled: true
      grpc:
        enabled: true
        endpoint: "otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317"
        insecure: true
```

- OTLP log export is an **experimental feature** in Traefik v3 (requires `experimental.otlpLogs: true`)
- Both **general logs** (Traefik daemon logs) and **access logs** (per-request HTTP logs) can be sent via OTLP
- Access logs are the most interesting: they include request method, path, status code, duration, etc.

---

## 3. Collector Changes Needed

The existing collector config already has:
- OTLP gRPC receiver on `:4317`
- OTLP HTTP receiver on `:4318`
- Pipelines for traces, metrics, and logs

**No collector changes are required.** Traefik pushes all signals via OTLP gRPC, which the collector already accepts. The Prometheus endpoint will be left available but not scraped (OTLP push is preferred for this evaluation).

---

## 4. Routing / Ingress Setup

A Kubernetes Ingress resource routes all traffic (`/`) through Traefik to the `otel-eval-backend` service in the `demo` namespace.

```yaml
# .otel-eval/traefik/traefik/routing/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: otel-eval-backend
  namespace: demo
  annotations:
    kubernetes.io/ingress.class: traefik
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

Apply after Traefik is running:
```bash
kubectl apply -f .otel-eval/traefik/traefik/routing/ingress.yaml
```

---

## 5. Traffic Generation

Traefik's `web` entrypoint listens on port 8000 internally (exposed as port 80 via ClusterIP service). Access via port-forward:

```bash
kubectl port-forward -n traefik svc/traefik 8080:80 &
```

Then generate traffic:

```bash
# Plain requests (Traefik starts new traces)
for i in $(seq 1 10); do
  curl -s http://localhost:8080/ | jq .
done

# Requests with W3C Trace Context (Traefik propagates)
for i in $(seq 1 10); do
  curl -s \
    -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
    http://localhost:8080/ | jq .
done

# 404 request (exercises error path)
curl -sv http://localhost:8080/not-found 2>&1 | tail -5
```

---

## 6. Verification Steps

```bash
# Check Traefik pods
kubectl get pods -n traefik

# Check Ingress is admitted
kubectl get ingress -n demo

# Check telemetry files
echo "=== Traces ===" && wc -l /tmp/otel-eval-traefik/traces.jsonl
echo "=== Metrics ===" && wc -l /tmp/otel-eval-traefik/metrics.jsonl
echo "=== Logs ===" && wc -l /tmp/otel-eval-traefik/logs.jsonl

# Sanity check first record of each
head -1 /tmp/otel-eval-traefik/traces.jsonl | jq '.resourceSpans[0].resource'
head -1 /tmp/otel-eval-traefik/metrics.jsonl | jq '.resourceMetrics[0].resource'
head -1 /tmp/otel-eval-traefik/logs.jsonl | jq '.resourceLogs[0].resource'
```

---

## 7. Expected Telemetry Summary

| Signal  | Method        | Protocol   | Expected Data                                           |
|---------|---------------|------------|---------------------------------------------------------|
| Traces  | OTLP push     | gRPC/4317  | One span per HTTP request through Traefik               |
| Metrics | OTLP push     | gRPC/4317  | Entry point / router / service request metrics          |
| Logs    | OTLP push     | gRPC/4317  | General daemon logs + per-request access logs           |

---

## 8. Actual Results

All three signals confirmed flowing. See `RESEARCH.md` for full post-install observations.

| Signal  | Status   | Protocol       | Service Name | Notes                                                     |
|---------|----------|----------------|--------------|-----------------------------------------------------------|
| Traces  | FLOWING  | OTLP gRPC/4317 | `traefik`    | Span per request; W3C propagation confirmed; distributed traces with backend |
| Metrics | FLOWING  | OTLP gRPC/4317 | `traefik`    | 17 metric names; both `traefik_*` and OTel semantic convention names |
| Logs    | FLOWING  | OTLP gRPC/4317 | `traefik`    | General + access logs; access logs include TraceId/SpanId; severity quirk |

**Telemetry files:**
- `/tmp/otel-eval-traefik/traces.jsonl` — 15+ batches
- `/tmp/otel-eval-traefik/metrics.jsonl` — 27+ batches
- `/tmp/otel-eval-traefik/logs.jsonl` — 6+ batches
