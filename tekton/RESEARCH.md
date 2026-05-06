# Tekton Pipeline — OTel Evaluation Research

## What is Tekton?

Tekton is a CNCF-graduated CI/CD framework for Kubernetes. It provides Kubernetes-native primitives for building, testing, and deploying software via CRDs: `Task`, `TaskRun`, `Pipeline`, `PipelineRun`, and `StepAction`. The Tekton controller watches these CRDs and orchestrates Kubernetes Pods to execute pipeline steps.

**GitHub:** https://github.com/tektoncd/pipeline  
**Version evaluated:** v1.12.0 ("Exotic Shorthair Elektrobots LTS")

---

## Installation Method

Tekton Pipeline is installed via plain Kubernetes manifests (no Helm chart for the core pipeline):

```
kubectl apply -f https://infra.tekton.dev/tekton-releases/pipeline/previous/v1.12.0/release.yaml
```

This creates:
- Namespace: `tekton-pipelines`
- Deployments: `tekton-pipelines-controller`, `tekton-pipelines-webhook`, `tekton-events-controller`
- CRDs: Task, TaskRun, Pipeline, PipelineRun, StepAction, CustomRun, etc.
- RBAC, ConfigMaps, ValidatingWebhookConfigurations

---

## Telemetry Capabilities

### Metrics
- **Protocol:** Prometheus (default) OR OTLP gRPC/HTTP (`grpc`, `http/protobuf`)
- **Configuration:** `config-observability` ConfigMap in `tekton-pipelines` namespace
- **Key field:** `metrics-protocol: prometheus` (default is `prometheus` in v1.12.0)
- **Prometheus port:** `:9090` on the controller and webhook pods
- **Services exposing metrics:**
  - `tekton-pipelines-controller` — port 9090 (`http-metrics`)
  - `tekton-pipelines-webhook` — port 9090 (`http-metrics`)
  - `tekton-events-controller` — port 9090 (`http-metrics`)
- **Key metrics exposed:**
  - `tekton_taskrun_duration_seconds` (histogram)
  - `tekton_pipelinerun_duration_seconds` (histogram)
  - `tekton_running_taskruns_count`
  - `tekton_running_pipelineruns_count`
  - `tekton_taskruns_count` (counter with reason label if enabled)
  - `tekton_pipelineruns_count`
  - Go runtime metrics (`go_*`, `process_*`)
  - Kubernetes client metrics (`rest_client_*`)

### Traces
- **Protocol:** OTLP gRPC or HTTP (`grpc`, `http/protobuf`)
- **Configuration:** `config-observability` ConfigMap
  - `tracing-protocol: grpc` (or `http/protobuf`)
  - `tracing-endpoint: <otlp-grpc-endpoint>`
  - `tracing-sampling-rate: "1.0"`
- **Default:** `tracing-protocol: none` — traces are OFF by default
- **What is traced:** TaskRun and PipelineRun lifecycle spans (reconcile loops in the controller)

### Logs
- **Format:** Structured JSON via `zap` logger (standard Kubernetes controller logs)
- **No OTLP log export** — logs are only available via stdout/stderr of the controller pods
- **Log level:** Configurable via `config-logging` ConfigMap

---

## OpenTelemetry Configuration Details

The `config-observability` ConfigMap in `tekton-pipelines` namespace controls all telemetry:

```yaml
data:
  metrics-protocol: prometheus          # prometheus | grpc | http/protobuf | none
  metrics-endpoint: ""                  # for grpc/http protocols
  metrics-export-interval: ""           # e.g. "30s"
  tracing-protocol: grpc               # none | grpc | http/protobuf | stdout
  tracing-endpoint: "host:port"        # OTLP endpoint (no scheme for gRPC)
  tracing-sampling-rate: "1.0"
  metrics.taskrun.level: "task"        # task | taskrun | namespace
  metrics.taskrun.duration-type: "histogram"
  metrics.pipelinerun.level: "pipeline"
  metrics.pipelinerun.duration-type: "histogram"
  metrics.count.enable-reason: "false"
```

For OTLP gRPC tracing, the endpoint should be `host:port` without scheme (e.g., `otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317`).

---

## Context Propagation

- Tekton injects trace context into TaskRun/PipelineRun spans using W3C Trace Context (traceparent/tracestate)
- The controller creates parent spans for PipelineRuns and child spans for TaskRuns within them
- No HTTP proxy/gateway — Tekton does not sit in the request path for external traffic

---

## Special Setup Requirements

- Tekton creates Pods to run Tasks — these Pods need appropriate RBAC and a ServiceAccount
- The `tekton-pipelines` namespace has `pod-security.kubernetes.io/enforce: restricted` — Tasks/Pods need to comply
- No sidecar injection or mesh required
- Traffic generation is done by creating `TaskRun` and `PipelineRun` resources (not HTTP traffic)

---

## Collector Changes Needed

Since Tekton exposes Prometheus metrics on port 9090 of the controller/webhook/events-controller services, the OTel Collector needs a Prometheus receiver with scrape configs targeting:

- `tekton-pipelines-controller.tekton-pipelines.svc.cluster.local:9090`
- `tekton-pipelines-webhook.tekton-pipelines.svc.cluster.local:9090`

For OTLP traces, the collector already has the OTLP receiver enabled on port 4317 (gRPC).

---

## Actual Observations (post-install)

### Critical Config Discovery

Tekton v1.12.0 has **two separate tracing configurations** that are easy to confuse:

1. **`config-observability`** (in `tekton-pipelines`) — Controls metrics protocol and high-level flags.
   The `_example` section shows `tracing-protocol`, `tracing-endpoint` etc., but these are **not used by the controller's OTLP exporter**. The actual tracing config is in a different ConfigMap.

2. **`config-tracing`** (in `tekton-pipelines`) — The actual tracing configuration used by the OTLP exporter. Keys:
   - `enabled: "true"` — must be explicitly set
   - `endpoint: "http://<host>:<port>/v1/traces"` — full URL including path

The `tracing.go` source confirms the controller uses `otlptracehttp` (HTTP, not gRPC), and parses the endpoint as a full URL including scheme and path. Default: `http://jaeger-collector.jaeger.svc.cluster.local:4318/v1/traces`.

### Telemetry Actually Flowing

**Traces — OTLP HTTP — CONFIRMED FLOWING**
- Service names: `taskrun-reconciler`, `pipelinerun-reconciler`
- Span names observed: `TaskRunReconciler`, `TaskRun:Reconciler`, `TaskRun:ReconcileKind`, `PipelineRun:ReconcileKind`, `updateTaskRunWithDefaultWorkspaces`, `prepare`, `createPod`, `reconcile`, `updateLabelsAndAnnotations`, `finishReconcileUpdateEmitEvents`, `durationAndCountMetrics`, `resolvePipelineState`, `runNextSchedulableTask`, `updatePipelineRunStatusFromInformer`, `createTaskRun`, `stopSidecars`
- Proper trace IDs and parent-child span relationships
- W3C Trace Context propagation (`otel.SetTextMapPropagator(propagation.TraceContext{})`)
- Resource attributes: `service.name` set natively; k8s attributes enriched by collector k8sattributes processor

**Metrics — Prometheus scrape — CONFIRMED FLOWING**
- Source: `:9090` on controller, webhook, events-controller pods
- Key metrics: `tekton_pipelines_controller_taskrun_duration_seconds` (histogram), `tekton_pipelines_controller_pipelinerun_duration_seconds`, `tekton_pipelines_controller_taskrun_total`, `tekton_pipelines_controller_pipelinerun_total`, `tekton_pipelines_controller_running_taskruns`, `tekton_pipelines_controller_running_pipelineruns`, `tekton_pipelines_controller_taskruns_pod_latency_milliseconds`, `tekton_pipelines_controller_pipelinerun_taskrun_duration_seconds`
- Also includes Go runtime metrics (`go_memory_*`, `go_processor_*`) and OpenTelemetry SDK metrics
- Resource attributes from Prometheus scrape include `service.name`, `service.instance.id`, `k8s_pod_name`, `telemetry_sdk_*`

**Logs — stdout only — NOT FLOWING via OTLP**
- Tekton controller logs are structured JSON via `zap` logger
- No OTLP log export capability in v1.12.0
- Logs contain `knative.dev/traceid` field (internal reconcile trace ID, not OTLP trace ID)
- Log format: `{"severity":"info","timestamp":"...","logger":"tekton-pipelines-controller","caller":"...","message":"..."}`

### Project-Native vs Collector-Derived

**Project-native (set by Tekton itself):**
- `service.name` on traces: `taskrun-reconciler`, `pipelinerun-reconciler`
- All span names and span attributes
- Trace IDs and parent-child span relationships
- `service.name` on metrics: `tekton-pipelines-controller`, `tekton-pipelines-webhook`, `tekton-events-controller`
- Tekton-specific metric labels: `namespace`, `reason`, `status`, `type`

**Collector-derived (added by k8sattributes processor):**
- `k8s.pod.name`, `k8s.pod.uid`, `k8s.namespace.name`, `k8s.deployment.name`
- `k8s.pod.label.*`, `k8s.pod.annotation.*`
- `k8s.node.name`, `k8s.pod.start_time`

**Prometheus-receiver-derived (from scrape metadata):**
- `server.address`, `server.port`, `url.scheme`, `service.instance.id`
- `k8s_namespace_name`, `k8s_pod_name` (underscore format, from Prometheus labels)
- `telemetry_sdk_language`, `telemetry_sdk_name`, `telemetry_sdk_version`
