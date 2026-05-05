# Tekton Pipeline — Install Plan

## Project Overview

**Tekton Pipeline** (https://github.com/tektoncd/pipeline) is a CNCF graduated project providing Kubernetes-native CI/CD primitives. It defines CRDs (`Task`, `Pipeline`, `TaskRun`, `PipelineRun`) that run as Kubernetes pods, orchestrated by the Tekton controller.

## 1. Installation Method

| Field | Value |
|---|---|
| **Version** | v1.12.0 |
| **Method** | `kubectl apply` — official GitHub release manifest |
| **Manifest URL** | `https://github.com/tektoncd/pipeline/releases/download/v1.12.0/release.yaml` |
| **Namespaces** | `tekton-pipelines`, `tekton-pipelines-resolvers` |
| **No Helm chart** | Official install is manifest-only; community Helm charts lag behind |

```bash
kubectl apply --filename https://github.com/tektoncd/pipeline/releases/download/v1.12.0/release.yaml
```

**Note**: `https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml` resolves to v1.6.0, not v1.12.0. Use the GitHub release URL for the latest version.

## 2. Telemetry Configuration

Tekton uses **two separate ConfigMaps** for telemetry:

| ConfigMap | Namespace | Controls |
|---|---|---|
| `config-observability` | `tekton-pipelines` | Metrics protocol and endpoint |
| `config-tracing` | `tekton-pipelines` | Trace enable/endpoint |

### Metrics — OTLP HTTP (`config-observability`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-observability
  namespace: tekton-pipelines
data:
  metrics-protocol: http/protobuf
  metrics-endpoint: "http://otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318"
  metrics-export-interval: "30s"
  metrics.taskrun.level: "task"
  metrics.taskrun.duration-type: "histogram"
  metrics.pipelinerun.level: "pipeline"
  metrics.pipelinerun.duration-type: "histogram"
  metrics.count.enable-reason: "false"
```

**Important**: Use `http/protobuf` (not `grpc`) — the `grpc` protocol requires TLS and fails with plaintext collectors. The `http://` prefix in the endpoint URL enables insecure (plaintext) mode.

### Traces — OTLP HTTP (`config-tracing`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-tracing
  namespace: tekton-pipelines
data:
  enabled: "true"
  endpoint: "http://otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318/v1/traces"
```

**Note**: Tekton traces use the `otlptracehttp` package (HTTP only, not gRPC). The `http://` scheme triggers insecure mode.

## 3. Collector Changes

Since Tekton uses OTLP HTTP export, the existing OTLP receiver handles both metrics and traces. **No Prometheus scrape config is needed** for primary telemetry.

**Important finding**: When `metrics-protocol` is not `prometheus`, Tekton does **not** start its Prometheus HTTP server on port 9090. Prometheus scrape attempts will fail (`up=0`).

The collector values add a Prometheus scrape config for documentation purposes, but it will show `up=0` as expected behavior when OTLP mode is active.

## 4. Routing / Ingress Setup

Tekton is a control-plane component — it does not sit in the HTTP request path. There is no ingress or routing needed. Traffic generation is done by creating Kubernetes CRs (`TaskRun`, `PipelineRun`).

## 5. Traffic Generation

Create TaskRun and PipelineRun resources to exercise the controller and generate telemetry:

```bash
# 8 TaskRuns
for i in $(seq 1 8); do kubectl create -f taskrun.yaml; done

# 4 PipelineRuns
for i in $(seq 1 4); do kubectl create -f pipelinerun.yaml; done
```

### TaskRun manifest
```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  generateName: hello-taskrun-
  namespace: default
spec:
  taskSpec:
    steps:
      - name: hello
        image: alpine:3.19
        script: |
          #!/bin/sh
          echo "Hello from Tekton TaskRun!"
          sleep 2
```

### PipelineRun manifest (2-task sequential pipeline)
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: hello-pipeline-run-
  namespace: default
spec:
  pipelineSpec:
    tasks:
      - name: task-a
        taskSpec:
          steps:
            - name: step-a
              image: alpine:3.19
              script: echo "Task A"
      - name: task-b
        runAfter: [task-a]
        taskSpec:
          steps:
            - name: step-b
              image: alpine:3.19
              script: echo "Task B"
```

## 6. Restart After Config Change

After patching the ConfigMaps, restart the controller to pick up changes:

```bash
kubectl rollout restart deployment/tekton-pipelines-controller -n tekton-pipelines
kubectl rollout restart deployment/tekton-events-controller -n tekton-pipelines
```

## 7. Verification

```bash
# Check Tekton pods are running
kubectl get pods -n tekton-pipelines

# Check runs completed
kubectl get taskruns,pipelineruns -n default

# Check telemetry files
wc -l /tmp/otel-eval-tekton/traces.jsonl
wc -l /tmp/otel-eval-tekton/metrics.jsonl
```

## 8. Telemetry Results

| Signal | Source | Protocol | Status | Notes |
|---|---|---|---|---|
| Metrics | `tekton-pipelines-controller` | OTLP HTTP | **Flowing** | 11 Tekton-specific metrics + infrastructure |
| Metrics | `tekton-events-controller` | OTLP HTTP | **Flowing** | Infrastructure metrics only |
| Traces | `pipelinerun-reconciler` | OTLP HTTP | **Flowing** | Per-reconcile spans with parent-child hierarchy |
| Traces | `taskrun-reconciler` | OTLP HTTP | **Flowing** | Per-reconcile spans |
| Logs | Tekton pods | stdout JSON | **Not OTLP** | No OTLP log export available |

### Key Metrics Observed
- `tekton_pipelines_controller_pipelinerun_duration_seconds` (histogram)
- `tekton_pipelines_controller_pipelinerun_taskrun_duration_seconds` (histogram)
- `tekton_pipelines_controller_pipelinerun_total` (counter)
- `tekton_pipelines_controller_running_pipelineruns` (gauge)
- `tekton_pipelines_controller_taskrun_duration_seconds` (histogram)
- `tekton_pipelines_controller_taskrun_total` (counter)
- `tekton_pipelines_controller_taskruns_pod_latency_milliseconds` (histogram)

### Key Spans Observed
- `PipelineRun:Reconciler`, `PipelineRun:ReconcileKind`
- `TaskRun:Reconciler`, `TaskRun:ReconcileKind`
- `createPod`, `createTaskRun`, `createTaskRuns`
- `durationAndCountMetrics`, `finishReconcileUpdateEmitEvents`
- `prepare`, `reconcile`, `resolvePipelineState`
- `runNextSchedulableTask`, `stopSidecars`
