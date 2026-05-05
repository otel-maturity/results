# Tekton Pipeline — Installation Plan

## Project Overview

**Tekton Pipeline** is a CNCF-graduated Kubernetes-native CI/CD framework. It provides CRDs for
defining and running CI/CD pipelines as Kubernetes resources. Each TaskRun creates a Pod; the
controller reconciles CRD objects and manages their lifecycle.

- **GitHub:** https://github.com/tektoncd/pipeline
- **Version:** v1.12.0 ("Exotic Shorthair Elektrobots LTS", released 2026-05-04)
- **Install namespace:** `tekton-pipelines` (hardcoded in the release manifest)
- **TaskRun namespace:** `tekton-runs` (created separately; needs privileged PSA)

---

## Step 1 — Install Tekton Pipeline

**Method:** Apply the official release manifest (no official Helm chart available).

```bash
kubectl apply --filename \
  https://github.com/tektoncd/pipeline/releases/download/v1.12.0/release.yaml
kubectl wait --for=condition=Available \
  deployment/tekton-pipelines-controller \
  deployment/tekton-pipelines-webhook \
  deployment/tekton-events-controller \
  -n tekton-pipelines --timeout=120s
```

**What gets installed:**
- Namespaces: `tekton-pipelines`, `tekton-pipelines-resolvers`
- CRDs: `tasks`, `pipelines`, `taskruns`, `pipelineruns`, `stepactions`, `verificationpolicies`,
  `resolutionrequests`, `customruns`
- Deployments: `tekton-pipelines-controller`, `tekton-pipelines-webhook`,
  `tekton-events-controller`, `tekton-pipelines-remote-resolvers`
- ConfigMaps: `config-observability`, `config-tracing`, `config-defaults`, `feature-flags`
- RBAC: ClusterRoles, ClusterRoleBindings, ServiceAccounts

---

## Step 2 — Create TaskRun Namespace

The `tekton-pipelines` namespace has `pod-security.kubernetes.io/enforce: restricted`. Tekton's
internal init containers (`prepare`, `place-scripts`) do NOT meet restricted PSA requirements.
A separate namespace with `privileged` PSA is needed for TaskRun pod execution.

```bash
kubectl create namespace tekton-runs
kubectl label namespace tekton-runs \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged
```

---

## Step 3 — Configure Telemetry

Tekton v1.12.0 has **two** telemetry configuration mechanisms:

### 3a. `config-observability` — Metrics + Tracing (new unified OTel config)

> **Important:** OTLP gRPC (`metrics-protocol: grpc`) fails with TLS handshake errors against
> a plaintext collector. Use `http/protobuf` (OTLP HTTP) instead.

```bash
kubectl patch configmap config-observability -n tekton-pipelines --type merge -p '{
  "data": {
    "metrics-protocol": "http/protobuf",
    "metrics-endpoint": "http://otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318",
    "metrics-export-interval": "15s",
    "tracing-protocol": "http/protobuf",
    "tracing-endpoint": "http://otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318",
    "tracing-sampling-rate": "1.0",
    "metrics.taskrun.level": "task",
    "metrics.taskrun.duration-type": "histogram",
    "metrics.pipelinerun.level": "pipeline",
    "metrics.pipelinerun.duration-type": "histogram",
    "metrics.count.enable-reason": "false"
  }
}'
```

The `metrics-endpoint` for `http/protobuf` is the **base URL** (no path). Tekton appends
`/v1/metrics` automatically.

### 3b. `config-tracing` — Tracing (legacy, HTTP-only)

```bash
kubectl patch configmap config-tracing -n tekton-pipelines --type merge -p '{
  "data": {
    "enabled": "true",
    "endpoint": "http://otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4318/v1/traces"
  }
}'
```

The `config-tracing` endpoint requires the **full path** including `/v1/traces`.

Both tracing paths are active simultaneously. `config-tracing` produces traces with
`service.name` = `taskrun-reconciler` / `pipelinerun-reconciler`.

### 3c. Restart controllers to pick up config

```bash
kubectl rollout restart \
  deployment/tekton-pipelines-controller \
  deployment/tekton-events-controller \
  deployment/tekton-pipelines-webhook \
  -n tekton-pipelines
kubectl rollout status \
  deployment/tekton-pipelines-controller \
  deployment/tekton-events-controller \
  deployment/tekton-pipelines-webhook \
  -n tekton-pipelines --timeout=90s
```

### 3d. Logs

Tekton does **not** support OTLP log export. Logs are structured JSON (zap/knative) written to
stdout. They are not captured in `logs.jsonl`. This is a documented limitation.

---

## Step 4 — Collector Changes

No changes to the OTel Collector are needed. The collector already accepts OTLP HTTP on port 4318.

Tekton pushes metrics and traces via OTLP HTTP to the collector directly — no Prometheus scraping
is configured (Tekton is switched from the default Prometheus mode to OTLP HTTP mode).

---

## Step 5 — Traffic Generation

Since Tekton is a CI/CD engine, "traffic" = creating `TaskRun` and `PipelineRun` CRD objects.

**Tasks created in `tekton-runs` namespace:**
- `call-backend` — single step, calls `otel-eval-backend` via curl
- `backend-integration` — multi-step: health check, POST, trace-context test

**Pipeline created:**
- `backend-pipeline` — 3 tasks: `call-backend` (GET), `call-backend` (POST), `backend-integration`

**Runs executed:**
- 20 `TaskRun` objects for `call-backend` (10 initial + 10 more)
- 5 `TaskRun` objects for `backend-integration`
- 6 `PipelineRun` objects for `backend-pipeline` (3 initial + 3 more)
- Total: 43 TaskRuns, 6 PipelineRuns — all succeeded

**Manifests:** `.otel-eval/tekton/tekton/tekton-task-runs.yaml`

---

## Step 6 — Verification

```bash
# Check pods
kubectl get pods -n tekton-pipelines
kubectl get pods -n tekton-runs

# Check runs
kubectl get taskruns -n tekton-runs
kubectl get pipelineruns -n tekton-runs

# Check telemetry files
wc -l /tmp/otel-eval-tekton/traces.jsonl
wc -l /tmp/otel-eval-tekton/metrics.jsonl
wc -l /tmp/otel-eval-tekton/logs.jsonl
```

---

## Actual Telemetry Results

| Signal  | Protocol       | Service Name(s)                        | Status   |
|---------|---------------|----------------------------------------|----------|
| Traces  | OTLP HTTP      | `taskrun-reconciler`, `pipelinerun-reconciler` | **Flowing** |
| Metrics | OTLP HTTP      | `tekton-pipelines-controller`          | **Flowing** |
| Logs    | (none)         | —                                      | Not supported |

**Telemetry file counts (after traffic generation):**
- `traces.jsonl`: 18 lines
- `metrics.jsonl`: 60 lines
- `logs.jsonl`: 0 lines (expected — no OTLP log export)

### Key Metrics

| Metric Name | Type | Labels |
|-------------|------|--------|
| `tekton_pipelines_controller_taskrun_duration_seconds` | Histogram | namespace, status, task |
| `tekton_pipelines_controller_pipelinerun_duration_seconds` | Histogram | namespace, status, pipeline |
| `tekton_pipelines_controller_pipelinerun_taskrun_duration_seconds` | Histogram | namespace, status, pipeline, task |
| `tekton_pipelines_controller_taskrun_total` | Counter | namespace, status, task |
| `tekton_pipelines_controller_pipelinerun_total` | Counter | namespace, status, pipeline |
| `tekton_pipelines_controller_running_taskruns` | Gauge | namespace |
| `tekton_pipelines_controller_running_pipelineruns` | Gauge | namespace |
| `tekton_pipelines_controller_taskruns_pod_latency_milliseconds` | Histogram | namespace |
| `go.goroutine.count`, `go.memory.*`, `go.processor.limit` | Gauge | — |
| `kn.workqueue.*` | Histogram/Gauge | — |
| `http.client.request.duration` | Histogram | — |

**Notable:** Histogram metrics include **exemplars** with `traceId` and `spanId` linking to
specific reconciliation traces. This enables metrics→traces correlation.

### Key Trace Spans

| Span Name | Service | Description |
|-----------|---------|-------------|
| `TaskRun:Reconciler` | `taskrun-reconciler` | Top-level reconcile entry |
| `TaskRun:ReconcileKind` | `taskrun-reconciler` | Main reconciliation logic |
| `prepare` | `taskrun-reconciler` | Parameter/workspace resolution |
| `createPod` | `taskrun-reconciler` | Pod creation |
| `stopSidecars` | `taskrun-reconciler` | Sidecar cleanup |
| `finishReconcileUpdateEmitEvents` | `taskrun-reconciler` | Status update + events |
| `durationAndCountMetrics` | `taskrun-reconciler` | Metric recording |
| `PipelineRun:Reconciler` | `pipelinerun-reconciler` | Top-level PipelineRun reconcile |
| `resolvePipelineState` | `pipelinerun-reconciler` | Pipeline task state resolution |
| `runNextSchedulableTask` | `pipelinerun-reconciler` | Next task scheduling |

---

## Issues Encountered

| Issue | Root Cause | Resolution |
|-------|-----------|------------|
| TaskRuns failing with `PodAdmissionFailed` | `tekton-pipelines` namespace has `restricted` PSA; Tekton internal init containers don't meet restricted profile | Created `tekton-runs` namespace with `privileged` PSA |
| OTLP gRPC metrics export failing with TLS error | Tekton gRPC exporter defaults to TLS; OTel Collector uses plaintext | Switched to `http/protobuf` (OTLP HTTP) in `config-observability` |
| `service.name` mismatch between traces and metrics | `config-tracing` sets `service.name` = reconciler name; `config-observability` sets it = controller name | Documented as a Tekton observability limitation; both paths active simultaneously |
