# OpenTelemetry Support Maturity Evaluation: Tekton Pipeline

## Project overview

- **Project**: Tekton Pipeline — CNCF-graduated Kubernetes-native CI/CD framework providing CRD-based primitives (Task, TaskRun, Pipeline, PipelineRun) for building, testing, and deploying software
- **Version evaluated**: v1.12.0 ("Exotic Shorthair Elektrobots LTS")
- **Evaluation date**: 2026-05-05
- **Cluster**: otel-eval-tekton (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)

---

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP traces (HTTP) and Prometheus/OTLP metrics supported; no OTLP log export |
| Semantic Conventions | 1 | Span attributes use non-standard keys (`taskrun`, `namespace`); metric naming is custom with no OTel semconv alignment; schema URL v1.12.0 is present but outdated |
| Resource Attributes & Configuration | 1 | Only `service.name` set natively; no `service.version`, `telemetry.sdk.*`, or `service.instance.id`; no OTEL_* env var support |
| Trace Modeling & Context Propagation | 2 | Rich parent-child span trees for TaskRun/PipelineRun lifecycle; W3C Trace Context propagation via `tekton.dev/taskrunSpanContext` annotation; all spans are INTERNAL kind |
| Multi-Signal Observability | 1 | Traces via OTLP; metrics via Prometheus (OTLP optional); no log signal at all; no cross-signal correlation |
| Audience & Signal Quality | 2 | Span names reflect meaningful lifecycle operations; Tekton-specific metrics are actionable; some internal spans add noise |
| Stability & Change Management | 2 | Dedicated metrics migration guide documents breaking changes; telemetry is labeled "experimental" in docs; schema URL present; no explicit telemetry stability contract |

---

## Telemetry overview

### Signals observed
- **Traces**: Flowing — OTLP HTTP (`otlptracehttp`) to collector port 4318
- **Metrics**: Flowing — Prometheus scrape from controller/webhook/events-controller pods on `:9090`
- **Logs**: Not flowing via OTLP — stdout only (structured JSON via zap logger); no OTLP log export capability in v1.12.0

### Resource attributes (native, before collector enrichment)

Tekton's `createTracerProvider` function (in `pkg/tracing/tracing.go`) sets only:
```
service.name: "taskrun-reconciler"    (TaskRun reconciler)
service.name: "pipelinerun-reconciler" (PipelineRun reconciler)
```
Schema URL set at resource level: `https://opentelemetry.io/schemas/1.12.0`

No `service.version`, `service.instance.id`, `telemetry.sdk.language`, `telemetry.sdk.name`, or `telemetry.sdk.version` are set natively by Tekton.

For Prometheus metrics, the scrape target provides:
```
service.name: "tekton-pipelines-controller"  (set by Prometheus receiver from job label)
```

### Resource attributes (after collector enrichment)

After `k8sattributes` processor enrichment, the following are added:
```
k8s.container.name: tekton-pipelines-controller
k8s.deployment.name: tekton-pipelines-controller
k8s.namespace.name: tekton-pipelines
k8s.node.name: otel-eval-tekton-control-plane
k8s.pod.name: tekton-pipelines-controller-5776955795-9nz6k
k8s.pod.uid: 0a2073e3-2761-4fdf-a1bd-04b42e296b59
k8s.pod.start_time: 2026-05-05T19:12:26Z
k8s.replicaset.name: tekton-pipelines-controller-5776955795
k8s.pod.label.app.kubernetes.io/version: v1.12.0
k8s.pod.label.pipeline.tekton.dev/release: v1.12.0
k8s.pod.label.version: v1.12.0
k8s.pod.annotation.kubectl.kubernetes.io/restartedAt: ...
```

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — Configurable OTLP**

#### Evidence

**Traces:**
- Tekton emits traces via OTLP HTTP (`otlptracehttp`) to a configurable endpoint
- Configured via `config-tracing` ConfigMap with keys `enabled: "true"` and `endpoint: "http://host:port/v1/traces"`
- Default state: tracing is **disabled** (`enabled: "false"` is the default)
- The `config-observability` ConfigMap shows `tracing-protocol: grpc/http/protobuf/none` options but these keys are **not** used by the actual OTLP exporter — only `config-tracing` matters for traces
- Traces confirmed flowing: 17 JSONL lines containing spans from `taskrun-reconciler` and `pipelinerun-reconciler`

**Metrics:**
- Default export: Prometheus on `:9090` (HTTP `/metrics` endpoint)
- Optional OTLP export: `metrics-protocol: grpc` or `http/protobuf` via `config-observability`
- Metrics confirmed flowing: 160 JSONL lines with Tekton-specific metrics, Go runtime metrics, and Knative infrastructure metrics
- Services scraped: `tekton-pipelines-controller`, `tekton-pipelines-webhook`, `tekton-events-controller`

**Logs:**
- No OTLP log export. Controller logs are written to stdout/stderr as structured JSON (zap logger)
- No mechanism exists in v1.12.0 to push logs via OTLP

**Configuration discoverability:**
- `config-observability` ConfigMap has a detailed `_example` section documenting all options
- `config-tracing` ConfigMap has a minimal `_example` section
- Official docs at `tekton.dev/docs/pipelines/metrics/` document both Prometheus and OTLP options
- There is a known documentation confusion: `config-observability` shows `tracing-protocol` and `tracing-endpoint` keys in its example that are NOT used by the OTLP trace exporter

#### Checklist assessment
- Does the project support OTLP export? **Yes** — traces via OTLP HTTP, metrics via OTLP gRPC/HTTP (optional)
- Is OTLP the default or recommended path? **No** — Prometheus is default for metrics; tracing is off by default
- Is there a documented configuration path? **Yes** — documented in official docs and ConfigMap examples
- Is OTLP export configurable without code changes? **Yes** — via ConfigMap patching

#### Rationale
Level 2 is appropriate: OTLP is supported and configurable for both traces and metrics, but it is not the default path (Prometheus is default for metrics, tracing is off by default). The configuration mechanism uses Kubernetes ConfigMaps rather than standard `OTEL_*` environment variables, which is a Kubernetes-native approach but diverges from the OTEL SDK standard configuration model. The absence of any log signal via OTLP prevents a higher score.

---

### 2. Semantic Conventions

**Level: 1 — Partial / Inconsistent**

#### Evidence

##### Trace attributes

Span-level attributes observed on Tekton spans:
- `taskrun` = `"call-eval-backend-run-26"` — non-standard key (should be something like `tekton.taskrun.name`)
- `namespace` = `"default"` — non-standard key (should be `k8s.namespace.name`)

These two attributes appear only on root-level spans (`TaskRun:Reconciler`, `TaskRun:ReconcileKind`, `PipelineRun:ReconcileKind`). Child spans (`prepare`, `createPod`, `reconcile`, `updateLabelsAndAnnotations`, etc.) have **no attributes at all**.

The `pipelinerun` attribute is set on PipelineRun spans:
- `pipelinerun` = `"eval-pipeline-run-7"` — non-standard key

**Schema URL**: `https://opentelemetry.io/schemas/1.12.0` is set on the resource (via `semconv.SchemaURL` from `go.opentelemetry.io/otel/semconv/v1.12.0`). This is **outdated** — the current stable semconv is v1.27.0 (2024). The schema URL indicates the project has not updated its semconv import since at least early 2023.

**Instrumentation scope version**: Both `TaskRunReconciler` and `PipelineRunReconciler` scopes report `version: "unknown"` (no version set).

**Span kinds**: All Tekton spans use `kind=1` (INTERNAL). This is technically correct for controller-internal operations, but root reconciler spans that represent the entry point of a reconcile loop might be better modeled as SERVER spans if they represent handling of an event.

**No deprecated attributes**: The Tekton spans do not use deprecated HTTP semconv attributes — but this is because they don't emit HTTP spans at all. The HTTP spans in the trace data (`GET /`, `middleware - *`) come from the `otel-eval-backend` Node.js service, which uses deprecated attributes (`http.method`, `http.status_code`, `http.url`, `http.target`, `http.host`).

##### Metric names and attributes

Tekton-specific metric names observed:
```
tekton_pipelines_controller_taskrun_duration_seconds        (histogram)
tekton_pipelines_controller_pipelinerun_duration_seconds    (histogram)
tekton_pipelines_controller_pipelinerun_taskrun_duration_seconds (histogram)
tekton_pipelines_controller_taskrun_total                   (sum/counter)
tekton_pipelines_controller_pipelinerun_total               (sum/counter)
tekton_pipelines_controller_running_taskruns                (gauge)
tekton_pipelines_controller_running_pipelineruns            (gauge)
tekton_pipelines_controller_taskruns_pod_latency_milliseconds (histogram)
tekton_pipelines_controller_running_pipelineruns_waiting_on_pipeline_resolution (gauge)
tekton_pipelines_controller_running_pipelineruns_waiting_on_task_resolution (gauge)
tekton_pipelines_controller_running_taskruns_waiting_on_task_resolution_count (gauge)
```

Metric naming follows a custom `tekton_pipelines_controller_*` convention — not OTel semantic conventions. No OTel system/process semconv alignment. Units are **not set** (`unit: null` for all Tekton metrics). The `_milliseconds` suffix on `taskruns_pod_latency_milliseconds` is inconsistent with the `_seconds` suffix used on other duration metrics.

Data point attribute keys on Tekton metrics: `namespace`, `pipeline`, `reason`, `status`, `task` — all custom, non-semconv keys.

Infrastructure metrics (`kn_workqueue_*`, `http_client_request_duration_seconds`, `go_*`) were recently migrated from OpenCensus and now align better with standard naming conventions, but still lack units on most metrics.

##### Log attributes

No OTLP log signal. Stdout logs contain `knative.dev/traceid` (a UUID, not an OTel trace ID), `knative.dev/key`, `knative.dev/controller`, `caller`, `severity`, `timestamp` — none of these follow OTel log semantic conventions.

#### Checklist assessment
- Are span attribute keys from OTel semantic conventions? **No** — `taskrun`, `namespace`, `pipelinerun` are custom keys
- Are metric names following OTel semconv? **No** — custom `tekton_*` naming convention
- Is schema URL present? **Yes** — but outdated (v1.12.0 vs current v1.27.0)
- Are deprecated attributes used? **Not in Tekton spans** — but absence of standard attributes is also a gap
- Are metric units set? **No** — all Tekton metrics have `unit: null`

#### Rationale
Level 1 reflects that Tekton uses custom attribute keys that don't follow OTel semantic conventions, metric units are absent, the semconv version is significantly outdated, and instrumentation scope versions are not set. There is no alignment with current OTel semconv for CI/CD or pipeline operations (though no official CI/CD semconv exists, using `k8s.namespace.name` instead of `namespace` would be more consistent).

---

### 3. Resource Attributes & Configuration

**Level: 1 — Partial Identity**

#### Evidence

##### Native resource attributes

From `pkg/tracing/tracing.go`, `createTracerProvider` sets:
```go
tracesdk.WithResource(resource.NewWithAttributes(
    semconv.SchemaURL,
    semconv.ServiceNameKey.String(service),
))
```

This means only `service.name` is set natively. The SDK does NOT auto-populate `telemetry.sdk.language`, `telemetry.sdk.name`, `telemetry.sdk.version`, `service.version`, or `service.instance.id`.

Confirmed from telemetry: the Tekton resource spans contain only `service.name` before k8s enrichment (all other attributes in the collected data are collector-derived).

For metrics (Prometheus scrape path), the `service.name` is set to the Prometheus job name (`tekton-pipelines-controller`) by the collector's Prometheus receiver — not set natively by Tekton.

##### OTEL_* environment variable support

No `OTEL_*` environment variables are set on Tekton controller pods:
```yaml
env:
- name: SYSTEM_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
```

Tekton does not read `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_EXPORTER_OTLP_ENDPOINT`, or any other standard OTEL_* env vars. All configuration is via ConfigMap (`config-tracing`, `config-observability`). This is a deliberate design choice for Kubernetes-native configuration but means operators cannot use standard OTel SDK configuration patterns.

##### Identity consistency across signals

- **Traces**: `service.name` = `"taskrun-reconciler"` or `"pipelinerun-reconciler"` (set by Tekton)
- **Metrics**: `service.name` = `"tekton-pipelines-controller"` (set by Prometheus receiver from job name)
- **Logs**: No OTLP signal; stdout logs use `logger: "tekton-pipelines-controller"`

The `service.name` values differ between traces and metrics — traces use reconciler-specific names while metrics use the controller pod name. There is no `service.version` on any signal. No `service.instance.id`. No `telemetry.sdk.*` attributes on traces.

#### Checklist assessment
- Is `service.name` set natively? **Yes** — on traces only
- Is `service.version` set? **No**
- Are `telemetry.sdk.*` attributes present? **No** — not set by Tekton's custom resource construction
- Is `service.instance.id` set? **No**
- Are OTEL_* env vars supported? **No**
- Is identity consistent across signals? **No** — different `service.name` values on traces vs metrics

#### Rationale
Level 1: `service.name` is set natively on traces, which is the minimum for signal routing. However, the missing `service.version`, `telemetry.sdk.*`, and inconsistent naming across signals prevent a higher level. The deliberate avoidance of OTEL_* env vars in favor of ConfigMap-based configuration is a notable design choice that limits portability.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — Meaningful Trace Trees**

#### Evidence

##### Span structure

Tekton produces rich, hierarchical span trees for both TaskRun and PipelineRun reconciliation. Observed trace structure for a TaskRun:

```
TaskRun:Reconciler (root, INTERNAL)          ← attrs: taskrun=..., namespace=...
  └── TaskRun:ReconcileKind (INTERNAL)        ← attrs: taskrun=..., namespace=...
        ├── prepare (INTERNAL)
        │     └── updateTaskRunWithDefaultWorkspaces (INTERNAL)
        ├── reconcile (INTERNAL)
        │     └── createPod (INTERNAL)
        ├── finishReconcileUpdateEmitEvents (INTERNAL)
        │     └── updateLabelsAndAnnotations (INTERNAL)
        └── durationAndCountMetrics (INTERNAL)
```

For PipelineRuns:
```
PipelineRun:Reconciler (root, INTERNAL)
  └── PipelineRun:ReconcileKind (INTERNAL)
        ├── resolvePipelineState (INTERNAL)
        ├── runNextSchedulableTask (INTERNAL)
        │     └── createTaskRuns (INTERNAL)
        │           └── createTaskRun (INTERNAL)
        └── updatePipelineRunStatusFromInformer (INTERNAL)
```

Total unique span names observed: `TaskRun:Reconciler`, `TaskRun:ReconcileKind`, `PipelineRun:Reconciler`, `PipelineRun:ReconcileKind`, `prepare`, `reconcile`, `createPod`, `updateTaskRunWithDefaultWorkspaces`, `finishReconcileUpdateEmitEvents`, `updateLabelsAndAnnotations`, `durationAndCountMetrics`, `stopSidecars`, `resolvePipelineState`, `runNextSchedulableTask`, `createTaskRuns`, `createTaskRun`, `updatePipelineRunStatusFromInformer`.

Multiple reconcile loops for the same TaskRun appear within a single trace (as the controller reconciles the object multiple times to completion), which is correct behavior.

##### Context propagation

Tekton implements W3C Trace Context propagation via `otel.SetTextMapPropagator(propagation.TraceContext{})` (set in `pkg/tracing/tracing.go` `init()` function).

**Inbound context propagation** (from external callers): Tekton supports injecting a parent span context via the `tekton.dev/taskrunSpanContext` annotation on TaskRun objects. The `initTracing` function in `pkg/reconciler/taskrun/tracing.go` checks this annotation and extracts the W3C trace context from it. This allows external systems to inject trace context into Tekton pipelines.

**Outbound context propagation**: When Tekton creates TaskRun pods, it stores the current span context in the `tekton.dev/taskrunSpanContext` pod annotation (observed in metrics data: `k8s.pod.annotation.tekton.dev/taskrunSpanContext: {"traceparent":"00-...-01"}`). This enables downstream processes running inside the pod to continue the trace.

**Standard `traceparent` annotation**: The evaluation also injected a `traceparent` annotation directly on TaskRun objects. The `initTracing` function does NOT handle the standard `traceparent` annotation — it only handles `tekton.dev/taskrunSpanContext`. The `traceparent` annotation on TaskRun objects does NOT cause Tekton to use the external trace ID; the trace in the otel-eval-backend service that carried the `traceparent` trace ID was from the backend service's own HTTP instrumentation, not from Tekton adopting the external context.

##### Trace coherence

The trace trees tell a meaningful story of the TaskRun/PipelineRun lifecycle: from reconciler entry → kind-specific reconciliation → sub-operations (prepare, pod creation, state resolution, task scheduling) → completion. The span hierarchy is coherent and follows the actual code execution path.

**Gap**: The trace does not connect to the Kubernetes pods that execute the actual CI/CD work. There is no link between the controller's `createPod` span and any telemetry from inside the executing pod. The `tekton.dev/taskrunSpanContext` annotation mechanism enables this in theory, but it requires the pod's workload to be instrumented and aware of the annotation.

#### Checklist assessment
- Are there meaningful parent-child span relationships? **Yes** — rich hierarchical trees
- Is W3C Trace Context propagation implemented? **Yes** — via `propagation.TraceContext{}`
- Can external callers inject trace context? **Yes** — via `tekton.dev/taskrunSpanContext` annotation
- Does the project propagate context to downstream services? **Yes** — via pod annotation
- Do traces tell a complete story? **Partially** — controller lifecycle is well-traced; pod execution is not

#### Rationale
Level 2: Tekton produces well-structured trace trees with proper parent-child relationships and implements W3C Trace Context propagation both inbound and outbound. The annotation-based context propagation mechanism is a thoughtful design for a controller-based system. The main gap is that all spans are INTERNAL kind (no SERVER/CLIENT distinction), and the trace does not extend into the executing pod workloads without additional instrumentation.

---

### 5. Multi-Signal Observability

**Level: 1 — Multiple Signals Present but Disconnected**

#### Evidence

##### Signal availability

| Signal | Status | Protocol | Notes |
|--------|--------|----------|-------|
| Traces | First-class | OTLP HTTP | Disabled by default; must enable via `config-tracing` |
| Metrics | First-class | Prometheus (default) or OTLP | Enabled by default via Prometheus |
| Logs | Not available via OTLP | stdout only | Structured JSON; no OTel log export |

##### Cross-signal correlation

**Traces ↔ Metrics**: No correlation. Traces use `service.name: "taskrun-reconciler"` while metrics use `service.name: "tekton-pipelines-controller"`. There are no shared trace/span IDs on metrics data points.

**Traces ↔ Logs**: No correlation. Controller logs contain `knative.dev/traceid` which is a UUID-format internal trace ID, NOT the OTel trace ID from the OTLP trace signal. There is no mechanism to correlate a log line with an OTel span.

**Metrics ↔ Logs**: No OTel-level correlation.

The `tekton.dev/taskrunSpanContext` pod annotation stores the W3C `traceparent` header, which theoretically allows pod-level workloads to correlate with the controller trace. However, this is not automatic — it requires the workload to read the annotation and initialize an OTel tracer.

##### Collection model

- **Traces**: OTLP push from controller to collector (OTLP HTTP)
- **Metrics**: Pull (Prometheus scrape) by collector from controller/webhook/events pods on port 9090
- **Logs**: No OTel collection; stdout only

The split between push (traces) and pull (metrics) means the two signals use different collection paths and cannot easily be correlated by the collector.

#### Checklist assessment
- Are traces, metrics, and logs all available via OTLP? **No** — logs have no OTLP path
- Is there trace context on log records? **No**
- Do metrics share `service.name` with traces? **No** — different values
- Is there a unified collection model? **No** — push for traces, pull for metrics

#### Rationale
Level 1: Two signals (traces and metrics) are available but use different collection paths and have inconsistent resource attributes that prevent correlation. Logs are entirely absent from the OTel signal. There is no mechanism for cross-signal correlation in the current implementation.

---

### 6. Audience & Signal Quality

**Level: 2 — Operator-Useful Signals**

#### Evidence

##### Span naming

Span names are a mix of meaningful lifecycle operations and internal code references:

**Meaningful (operator-useful)**:
- `TaskRun:Reconciler` — clearly identifies the reconciler entry point for a TaskRun
- `TaskRun:ReconcileKind` — the main reconcile logic
- `PipelineRun:ReconcileKind` — the main reconcile logic for a PipelineRun
- `createPod` — creating the Kubernetes pod for a task step
- `prepare` — preparation phase of task execution
- `stopSidecars` — stopping sidecar containers
- `resolvePipelineState` — resolving pipeline execution state
- `runNextSchedulableTask` — scheduling the next task in a pipeline
- `createTaskRun` / `createTaskRuns` — creating child TaskRuns from a PipelineRun

**Internal/less meaningful to operators**:
- `finishReconcileUpdateEmitEvents` — very implementation-specific
- `updateLabelsAndAnnotations` — low-level Kubernetes operation
- `durationAndCountMetrics` — internal metrics recording, not a business operation
- `updateTaskRunWithDefaultWorkspaces` — low-level internal operation
- `updatePipelineRunStatusFromInformer` — very implementation-specific

The root span `TaskRun:Reconciler` is created with `span.End()` deferred immediately, meaning it ends at the same time as the first reconcile loop, but subsequent reconcile loops for the same TaskRun are attached as children of this root span. This creates a coherent per-TaskRun trace.

##### Signal-to-noise ratio

**Traces**: The volume of spans is appropriate. Each reconcile loop generates ~8-10 spans. For a TaskRun that goes through 3-4 reconcile cycles (Pending → Running → Succeeded), the total trace contains ~30-40 spans, which is manageable. The `durationAndCountMetrics` span adds noise (it's an internal implementation detail).

**Metrics**: The Tekton-specific metrics (`tekton_pipelines_controller_*`) are directly actionable for CI/CD operators:
- Duration histograms with `status` labels enable SLA monitoring
- Running count gauges enable capacity monitoring
- Pod latency metric enables scheduling performance monitoring
- The `reason` label (when enabled) enables failure categorization

Infrastructure metrics (`kn_workqueue_*`, `go_*`, `http_client_request_duration_seconds`) are useful for platform engineers but may be noisy for CI/CD operators.

##### Default usability

**Traces**: Off by default — operators must explicitly enable tracing. Once enabled, the traces are immediately useful for debugging slow TaskRuns/PipelineRuns. The `taskrun` and `namespace` attributes on root spans allow filtering by specific TaskRun.

**Metrics**: On by default (Prometheus). The Tekton-specific metrics are immediately useful without additional configuration. The `metrics.taskrun.level` and `metrics.pipelinerun.level` settings allow operators to control cardinality.

#### Checklist assessment
- Do span names reflect logical operations? **Mostly yes** — with some implementation-specific names
- Are metrics actionable for operators? **Yes** — duration, counts, and latency metrics are directly useful
- Is the signal-to-noise ratio acceptable? **Yes** — with some noisy internal spans
- Are signals useful out-of-the-box? **Metrics yes; traces require explicit enablement**

#### Rationale
Level 2: The signals are genuinely useful to operators. The Tekton-specific metrics directly measure CI/CD performance (pipeline/task duration, scheduling latency, running counts). The trace structure reflects the actual lifecycle of CI/CD operations. The main detractors are: some implementation-specific span names, the `durationAndCountMetrics` span being noise, and tracing being disabled by default.

---

### 7. Stability & Change Management

**Level: 2 — Documented with Change Communication**

#### Evidence

##### Documentation of telemetry behavior

Tekton has dedicated observability documentation:
- `tekton.dev/docs/pipelines/metrics/` — comprehensive metrics documentation listing all metric names, types, labels, and configuration options
- `tekton.dev/docs/pipelines/metrics-migration-otel/` — dedicated migration guide for the OpenCensus → OpenTelemetry transition
- The metrics doc explicitly labels all core Tekton metrics as **"experimental"** status
- Configuration options are documented in ConfigMap `_example` sections and in official docs

The tracing configuration is documented in `config-tracing` ConfigMap and referenced in the metrics doc. However, the documentation confusion between `config-observability` (which shows `tracing-protocol`/`tracing-endpoint` in its example but does NOT control the OTLP trace exporter) and `config-tracing` (which does) is a notable gap.

##### Change communication

The OpenCensus → OpenTelemetry migration (PR #9043) was documented with a comprehensive migration guide (`metrics-migration-otel.md`) that:
- Identifies breaking changes explicitly (workqueue metric renames, Go runtime metric renames)
- Provides old-name → new-name mapping tables
- Categorizes impact level (HIGH/MEDIUM/LOW)
- Provides action items for operators

This demonstrates a mature approach to communicating breaking telemetry changes.

##### Schema URL presence

- **Traces**: Schema URL `https://opentelemetry.io/schemas/1.12.0` is present on `resourceSpans` — set via `semconv.SchemaURL` from `go.opentelemetry.io/otel/semconv/v1.12.0`
- **Metrics**: Schema URL `https://opentelemetry.io/schemas/1.18.0` is present on `resourceMetrics` (set by the collector's k8s_cluster receiver)
- The trace schema URL (v1.12.0) is outdated — the current stable semconv is v1.27.0

##### Stability guarantees

- All core Tekton metrics are explicitly labeled **"experimental"** in the official documentation
- No formal stability contract for telemetry (no versioning policy for metric names or span attributes)
- The migration guide for the OpenCensus → OTel transition shows willingness to make breaking changes with documentation
- The `config-tracing` ConfigMap default endpoint (`http://jaeger-collector.jaeger.svc.cluster.local:4318/v1/traces`) suggests Jaeger as the assumed backend, which is a documentation-level coupling

#### Checklist assessment
- Is telemetry documented? **Yes** — dedicated metrics doc and migration guide
- Are breaking changes communicated? **Yes** — comprehensive migration guide for the OTel transition
- Is schema URL present? **Yes** — on traces (outdated version)
- Are there explicit stability guarantees? **No** — all metrics labeled "experimental"
- Is the documentation accurate? **Mostly** — with the notable confusion around `config-observability` vs `config-tracing`

#### Rationale
Level 2: Tekton has demonstrated good change management practices with the OpenCensus → OTel migration guide. Telemetry is documented and breaking changes are communicated. The "experimental" label on all metrics is honest about stability. The main gaps are the absence of a formal telemetry stability contract and the outdated schema URL.

---

## Key findings

### Strengths

1. **Rich trace trees for CI/CD lifecycle**: Tekton produces meaningful, hierarchical traces that accurately represent the TaskRun/PipelineRun reconciliation lifecycle. The traces are immediately useful for debugging slow or failing pipelines without requiring additional configuration beyond enabling tracing.

2. **Thoughtful context propagation design**: The `tekton.dev/taskrunSpanContext` annotation mechanism is an elegant solution for propagating trace context across the Kubernetes controller boundary into executing pods. This design acknowledges the architectural reality that CI/CD work happens in separate pods and provides a path for end-to-end tracing.

3. **Comprehensive migration documentation**: The OpenCensus → OpenTelemetry migration guide is exemplary — it provides explicit breaking change tables, impact assessments, and action items. This demonstrates that Tekton treats telemetry as a first-class concern for operators.

4. **Actionable CI/CD metrics**: The Tekton-specific metrics (`tekton_pipelines_controller_taskrun_duration_seconds`, `_pipelinerun_duration_seconds`, `_taskruns_pod_latency_milliseconds`) directly measure CI/CD performance and are immediately useful for SLA monitoring and capacity planning.

5. **Flexible metrics configuration**: The `metrics.taskrun.level` and `metrics.pipelinerun.level` settings allow operators to control metric cardinality, which is important for high-volume Tekton deployments.

### Areas for improvement

1. **Add `service.version` and `telemetry.sdk.*` resource attributes**: The native resource set is minimal — only `service.name` is set. Adding `service.version` (from the release label `v1.12.0`), `telemetry.sdk.language: "go"`, `telemetry.sdk.name: "opentelemetry"`, and `telemetry.sdk.version` would improve signal identity and allow operators to filter by version.

2. **Align span and metric attribute keys with OTel semantic conventions**: The `taskrun`, `namespace`, and `pipelinerun` span attributes should use namespaced keys. While no official CI/CD semconv exists, using `k8s.namespace.name` instead of `namespace` would be consistent with OTel conventions. Metric data point attributes should similarly be reviewed.

3. **Set metric units**: All Tekton-specific metrics have `unit: null`. Adding units (`s` for seconds, `ms` for milliseconds, `{taskrun}` for counts) would make metrics self-documenting and improve compatibility with OTel-aware backends.

4. **Resolve the `config-observability` vs `config-tracing` documentation confusion**: The `config-observability` ConfigMap's `_example` section shows `tracing-protocol: grpc` and `tracing-endpoint` keys that are NOT used by the actual OTLP trace exporter. This is a significant usability issue — operators who configure tracing via `config-observability` will see no traces. The example section should either be corrected or removed.

5. **Enable OTLP as the default for metrics or provide a unified configuration path**: Having traces use OTLP push while metrics use Prometheus pull creates operational complexity. Supporting `OTEL_EXPORTER_OTLP_ENDPOINT` as an environment variable would align with standard OTel SDK configuration and make Tekton easier to integrate with OTel-native backends.

6. **Add log correlation**: The controller logs contain `knative.dev/traceid` which is a different identifier than the OTel trace ID. Adding the OTel `traceId` and `spanId` to log records (even via stdout) would enable log-trace correlation.

### Notable observations

1. **The `config-tracing` vs `config-observability` split is a real operational footgun**: The installation research confirmed that configuring tracing via `config-observability` (which appears to support it based on the `_example` section) does not actually enable OTLP traces. The actual tracing is controlled by `config-tracing`. This was discovered by reading the source code — the documentation does not clearly distinguish these two ConfigMaps.

2. **`knative.dev/traceid` in logs is NOT the OTel trace ID**: Controller logs include `"knative.dev/traceid":"78b0f2d0-b32c-49af-b495-cac6e05e0889"` which is a UUID-format internal Knative reconciler trace ID, completely separate from the OTel hex-format trace IDs. This can mislead operators who expect log-trace correlation.

3. **The `tekton.dev/taskrunSpanContext` annotation stores W3C traceparent**: Pod annotations include `tekton.dev/taskrunSpanContext: {"traceparent":"00-...-01"}`, which is the serialized W3C trace context. This is a well-designed interface for downstream workloads to continue the trace, but it requires workloads to be OTel-aware and read Kubernetes pod annotations.

4. **Semconv v1.12.0 is significantly outdated**: The tracing code imports `go.opentelemetry.io/otel/semconv/v1.12.0` which corresponds to the OTel semconv from early 2023. The current stable semconv is v1.27.0. While this doesn't affect the current minimal attribute set, it means any future semconv-aligned attributes would start from an outdated baseline.

5. **Instrumentation scope version is not set**: Both `TaskRunReconciler` and `PipelineRunReconciler` scopes report `version: "unknown"`. Setting the scope version to the Tekton release version would improve telemetry provenance.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector (v0.150.1) with file exporter in a kind cluster named `otel-eval-tekton`
- The `k8sattributes` processor was used to enrich telemetry; native vs enriched attributes were distinguished by examining the `createTracerProvider` source code and comparing with the collected data
- Traces: 17 JSONL batches containing spans from `taskrun-reconciler`, `pipelinerun-reconciler`, and `otel-eval-backend` services
- Metrics: 160 JSONL batches containing Tekton-specific metrics, infrastructure metrics, and Kubernetes cluster metrics
- Logs: 0 JSONL records (no OTLP log export from Tekton)
- Semantic conventions were checked against OTel semconv v1.27.0 (current stable)
- Source code reviewed: `pkg/tracing/tracing.go`, `pkg/reconciler/taskrun/tracing.go`, `pkg/reconciler/taskrun/taskrun.go`
- Documentation reviewed: `tekton.dev/docs/pipelines/metrics/`, `tekton.dev/docs/pipelines/metrics-migration-otel/`, GitHub `docs/metrics.md`
- Traffic generated: 30 TaskRuns (including 5 with external trace context), 8 PipelineRuns
