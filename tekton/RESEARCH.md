# Tekton Pipeline — Research Notes

## What is Tekton?

Tekton is a CNCF project (graduated) that provides a Kubernetes-native CI/CD framework. It defines
Kubernetes Custom Resource Definitions (CRDs) for CI/CD primitives:

- **Task** / **TaskRun** — a single unit of work (steps run as containers in a Pod)
- **Pipeline** / **PipelineRun** — an ordered graph of Tasks
- **StepAction** — a reusable step definition
- **Resolver** — resolves remote Task/Pipeline definitions (Git, Bundle, Cluster, etc.)

Tekton runs entirely inside Kubernetes: each TaskRun creates a Pod; Pipelines orchestrate those Pods
via the controller. There is no long-lived "agent" per pipeline; the controller watches CRD objects
and reconciles them.

## Installation Method

**Official method:** Apply a single YAML manifest from the release bucket.

- Release v1.12.0 (latest as of 2026-05-04, "Exotic Shorthair Elektrobots LTS")
- Manifest URL: `https://github.com/tektoncd/pipeline/releases/download/v1.12.0/release.yaml`
- Installs into namespace `tekton-pipelines` (hardcoded in the manifest)
- Also creates `tekton-pipelines-resolvers` namespace for the resolvers component

No official Helm chart exists. The community chart (`cdf/tekton-pipeline`) is outdated. The
canonical install is always the upstream release manifest.

## Telemetry Capabilities

### Metrics

Tekton v1.12.0 supports **OpenTelemetry-native metrics** via the `config-observability` ConfigMap.

**Configuration key:** `metrics-protocol` in `config-observability` ConfigMap in `tekton-pipelines`

Supported values:
- `prometheus` (default) — exposes a Prometheus scrape endpoint on port **9090** at `/metrics`
- `grpc` — OTLP gRPC export
- `http/protobuf` — OTLP HTTP export
- `none` — disabled

**Metrics endpoint** (for grpc/http): configured via `metrics-endpoint` key

**Prometheus scrape targets** (when `metrics-protocol: prometheus`):
- `tekton-pipelines-controller.tekton-pipelines.svc:9090/metrics`
- `tekton-events-controller.tekton-pipelines.svc:9090/metrics`
- `tekton-pipelines-webhook.tekton-pipelines.svc:9090/metrics` (port 9090)

**Available Tekton-specific metrics:**
- `tekton_taskrun_duration_seconds` — histogram of TaskRun durations
- `tekton_pipelinerun_duration_seconds` — histogram of PipelineRun durations
- `tekton_running_taskruns_count` — gauge of currently running TaskRuns
- `tekton_running_pipelineruns_count` — gauge of currently running PipelineRuns
- `tekton_taskrun_count` — counter of TaskRuns by status
- `tekton_pipelinerun_count` — counter of PipelineRuns by status
- `tekton_reconcile_count` — controller reconciliation count
- Plus standard Go runtime and controller-runtime metrics

### Traces

Tekton v1.12.0 supports **OTLP tracing** via the `config-tracing` ConfigMap in `tekton-pipelines`.

**Configuration keys:**
- `enabled: "true"` — enables trace export
- `endpoint` — OTLP HTTP endpoint (e.g., `http://collector:4318/v1/traces`)

Tekton traces cover the **controller reconciliation loop**: each TaskRun/PipelineRun reconciliation
creates spans. The spans capture:
- PipelineRun reconciliation
- TaskRun reconciliation
- Pod creation and status updates

**Important:** Tekton uses `config-tracing` (legacy, HTTP-only OTLP) AND `config-observability`
(newer, supports both grpc and http/protobuf for traces via `tracing-protocol` key).

In v1.12.0, `config-observability` has `tracing-protocol` as an `_example` field (not active data),
while `config-tracing` is the active tracing configuration.

### Logs

Tekton does not export logs via OTLP. Logs are written to stdout/stderr by the controller,
webhook, and events controller pods in structured format. They are not pushed to an OTLP endpoint.

Step/Task logs are streamed via the Kubernetes API (pod logs) — no OTLP log export.

## Context Propagation

Tekton propagates W3C Trace Context headers when creating child spans during reconciliation.
The `traceparent` header is used internally between controller components.

Tekton does **not** inject trace context into the TaskRun Pods (the user's CI steps). The tracing
covers the Tekton control plane only, not the user workloads.

## Special Setup Requirements

- Requires CRDs: `tasks.tekton.dev`, `pipelines.tekton.dev`, `taskruns.tekton.dev`,
  `pipelineruns.tekton.dev`, `stepactions.tekton.dev`, `verificationpolicies.tekton.dev`, etc.
- Requires `tekton-pipelines` namespace with `pod-security.kubernetes.io/enforce: restricted`
- The `config-tracing` ConfigMap must be patched **after** installation (not included as active data)
- No sidecar injection; no service mesh required
- Kind clusters need no special configuration for Tekton

## Traffic Generation Strategy

Since Tekton is a CI/CD engine (not an HTTP proxy/gateway), "traffic" means creating TaskRun and
PipelineRun objects. The test backend (`otel-eval-backend` in `demo`) can be called from a TaskRun
step using `curl`.

Traffic generation plan:
1. Create a Task that calls the test backend via `curl`
2. Create multiple TaskRuns (10+) to exercise the metrics and traces
3. Create a Pipeline with multiple Tasks and run it via PipelineRun
4. Include both successful and failed runs to exercise different metric labels

## Actual Observations (filled in post-install)

_To be completed after installation._

---

## Actual Observations (Post-Installation)

### Installation

- **Version:** v1.12.0 installed via `kubectl apply` from GitHub releases
- **Namespace:** `tekton-pipelines` (hardcoded in manifest)
- **Additional namespace:** `tekton-runs` (created for TaskRun execution, with privileged PSA)

### PSA (Pod Security Admission) Discovery

The `tekton-pipelines` namespace has `pod-security.kubernetes.io/enforce: restricted` label.
Tekton's internal init containers (`prepare`, `place-scripts`) do NOT meet the restricted PSA
profile — they require `allowPrivilegeEscalation` and don't set `runAsNonRoot`.

**Mitigation:** Created `tekton-runs` namespace with `privileged` PSA label for TaskRun execution.
This is the standard workaround for Tekton on PSA-enabled clusters.

### Telemetry Configuration Discovery

In v1.12.0, Tekton has **two** telemetry configuration mechanisms:

1. **`config-observability` ConfigMap** (new, unified OTel config):
   - `metrics-protocol`: `prometheus` | `grpc` | `http/protobuf` | `none`
   - `metrics-endpoint`: OTLP endpoint URL (without path for HTTP)
   - `tracing-protocol`: `grpc` | `http/protobuf` | `none` | `stdout`
   - `tracing-endpoint`: OTLP endpoint URL (without path)
   - `tracing-sampling-rate`: 0.0–1.0

2. **`config-tracing` ConfigMap** (legacy, HTTP-only):
   - `enabled: "true"`
   - `endpoint`: full OTLP HTTP URL including path (e.g., `.../v1/traces`)

Both are active. The `config-tracing` path sets `service.name` to `taskrun-reconciler` /
`pipelinerun-reconciler`. The `config-observability` metrics path sets `service.name` to
`tekton-pipelines-controller` with `service.version` = git commit hash.

### OTLP gRPC Failure

Initial attempt to use `metrics-protocol: grpc` failed with:
```
transport: authentication handshake failed: tls: first record does not look like a TLS handshake
```
The Tekton OTLP gRPC exporter defaults to TLS. The OTel Collector in this cluster uses plaintext.
**Fix:** Use `http/protobuf` (OTLP HTTP) instead, which works without TLS by default.

### Telemetry Signals Actually Flowing

| Signal  | Protocol       | Service Name              | Source                         |
|---------|---------------|---------------------------|-------------------------------|
| Traces  | OTLP HTTP      | `taskrun-reconciler`      | `config-tracing` ConfigMap    |
| Traces  | OTLP HTTP      | `pipelinerun-reconciler`  | `config-tracing` ConfigMap    |
| Metrics | OTLP HTTP      | `tekton-pipelines-controller` | `config-observability`    |
| Logs    | (none)         | —                         | Not supported via OTLP        |

### Metrics Details

Tekton emits the following metric categories via OTLP HTTP:

**Tekton-specific:**
- `tekton_pipelines_controller_taskrun_duration_seconds` (histogram, with exemplars!)
- `tekton_pipelines_controller_pipelinerun_duration_seconds` (histogram)
- `tekton_pipelines_controller_pipelinerun_taskrun_duration_seconds` (histogram)
- `tekton_pipelines_controller_taskrun_total` (counter)
- `tekton_pipelines_controller_pipelinerun_total` (counter)
- `tekton_pipelines_controller_running_taskruns` (gauge)
- `tekton_pipelines_controller_running_pipelineruns` (gauge)
- `tekton_pipelines_controller_running_*_waiting_on_*` (gauges)
- `tekton_pipelines_controller_taskruns_pod_latency_milliseconds` (histogram)

**Go runtime (via OTel Go SDK):**
- `go.goroutine.count`, `go.memory.used`, `go.memory.allocated`, `go.processor.limit`, etc.

**Knative framework (workqueue):**
- `kn.workqueue.depth`, `kn.workqueue.adds`, `kn.workqueue.process.duration`, etc.
- `kn.k8s.client.http.response.status_code`
- `http.client.request.duration`

**Key finding:** Tekton metrics include **exemplars** that link histogram data points to specific
trace IDs and span IDs. This enables metrics-to-traces correlation.

### Trace Details

Tekton traces cover the **controller reconciliation loop**:

- `TaskRun:Reconciler` — top-level reconcile span
- `TaskRun:ReconcileKind` — main reconciliation logic
- `prepare` — preparing the TaskRun (resolving params, workspaces)
- `updateTaskRunWithDefaultWorkspaces` — applying workspace defaults
- `createPod` — creating the TaskRun Pod
- `stopSidecars` — stopping sidecars after completion
- `finishReconcileUpdateEmitEvents` — updating status and emitting events
- `durationAndCountMetrics` — recording duration metrics
- `PipelineRun:Reconciler` — top-level PipelineRun reconcile
- `PipelineRun:ReconcileKind` — main PipelineRun logic
- `resolvePipelineState` — resolving pipeline task state
- `runNextSchedulableTask` — scheduling next task in pipeline
- `updatePipelineRunStatusFromInformer` — updating status

Trace span attributes include `taskrun` / `pipelinerun` name and `namespace`.

**Missing from traces:** No `service.version` in trace resource. The `service.name` values
(`taskrun-reconciler`, `pipelinerun-reconciler`) don't match the metrics `service.name`
(`tekton-pipelines-controller`), making cross-signal correlation harder.

### Logs

Tekton logs are structured JSON (using `zap` logger via `knative.dev/pkg`). They include:
- `severity`, `timestamp`, `logger`, `caller`, `message`
- `knative.dev/traceid` — a trace ID (but this is the knative internal trace ID, not OTel)
- `knative.dev/controller`, `knative.dev/kind`, `knative.dev/key`

Logs are **not** exported via OTLP. They are only available via `kubectl logs`.

### Traffic Generation

43 TaskRuns and 6 PipelineRuns were created in the `tekton-runs` namespace. All succeeded.
TaskRun pods ran `curlimages/curl:8.11.1` to call the `otel-eval-backend` service.
