# OpenTelemetry Support Maturity Evaluation: Tekton Pipelines

## Project overview

- **Project**: Tekton Pipelines — Kubernetes-native CI/CD primitives (Tasks, Pipelines, TaskRuns, PipelineRuns)
- **Version evaluated**: v1.12.0
- **Evaluation date**: 2025-05-05
- **Cluster**: otel-eval-tekton (kind)
- **Maturity model version**: OpenTelemetry Support Maturity Model for CNCF Projects (draft)

## Summary

| Dimension | Level | Summary |
|-----------|-------|---------|
| Integration Surface | 2 | OTLP HTTP export for traces and metrics via ConfigMap; Prometheus remains the default |
| Semantic Conventions | 1 | Custom `tekton_*` metric names and internal-style span names; schema URL set but pinned to v1.12.0 |
| Resource Attributes & Configuration | 2 | `service.name`, `telemetry.sdk.*` set natively; ConfigMap-based config only, no `OTEL_*` env var support |
| Trace Modeling & Context Propagation | 2 | Rich internal reconciler spans with correct parent-child hierarchy; W3C traceparent propagated to pod annotations |
| Multi-Signal Observability | 1 | Traces and metrics flow via OTLP; logs are stdout-only with no OTLP export path |
| Audience & Signal Quality | 1 | Span names are internal Go function names; metrics labeled "experimental"; usable but requires operator familiarity |
| Stability & Change Management | 2 | Dedicated migration guide for OpenCensus → OTel breaking change; schema URL present; metrics still marked "experimental" |

## Telemetry overview

### Signals observed
- **Traces**: Flowing — OTLP HTTP (`otlptracehttp`), pushed by `pipelinerun-reconciler` and `taskrun-reconciler`
- **Metrics**: Flowing — OTLP HTTP (`http/protobuf`), pushed by `tekton-pipelines-controller` and `tekton-events-controller`
- **Logs**: Not flowing via OTLP — Tekton logs are written to stdout only; no OTLP log export capability exists

### Resource attributes (native, before collector enrichment)

Tekton sets only a minimal set of resource attributes natively. Evidence from source (`tracing.go`) and telemetry confirms:

- `service.name` — set programmatically (e.g., `"pipelinerun-reconciler"`, `"taskrun-reconciler"`, `"tekton-pipelines-controller"`, `"events-controller"`)
- `service.version` — set to the git commit SHA (e.g., `"7798558"`), **not** the release version tag
- `telemetry.sdk.name` — `"opentelemetry"` (set by the OTel Go SDK)
- `telemetry.sdk.language` — `"go"` (set by the OTel Go SDK)
- `telemetry.sdk.version` — `"1.43.0"` (set by the OTel Go SDK)

The `service.version` value being a short git SHA rather than `"v1.12.0"` is a notable gap.

### Resource attributes (after collector enrichment)

The `k8sattributes` processor adds the following based on pod identity:
- `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.namespace.name`, `k8s.node.name`
- `k8s.deployment.name`, `k8s.replicaset.name`, `k8s.container.name`
- All pod labels (`k8s.pod.label.*`) and annotations (`k8s.pod.annotation.*`), including `tekton.dev/taskrunSpanContext`

---

## Dimension evaluations

### 1. Integration Surface

**Level: 2 — Configurable OTLP**

#### Evidence

Tekton v1.12.0 supports OTLP export for both traces and metrics, configured via two Kubernetes ConfigMaps:

- **`config-observability`** (namespace `tekton-pipelines`): Controls metrics export. Supports `metrics-protocol: prometheus | grpc | http/protobuf | none`. Default is `prometheus`. OTLP HTTP (`http/protobuf`) and gRPC (`grpc`) are available.
- **`config-tracing`** (namespace `tekton-pipelines`): Controls trace export. Keys: `enabled: "true"` and `endpoint: "<otlp-http-url>"`. Traces use `otlptracehttp` exclusively (HTTP only, not gRPC).

From the install plan and source (`tracing.go`): the trace exporter is hard-coded to `otlptracehttp`; there is no gRPC option for traces. Metrics support both gRPC and HTTP/protobuf.

Both signals were confirmed flowing in the evaluation cluster:
- Traces: 6 export batches, 1,621 spans across `pipelinerun-reconciler` and `taskrun-reconciler`
- Metrics: 71 export batches containing 11 Tekton-domain metrics plus infrastructure metrics

The default configuration ships with `metrics-protocol: prometheus` — operators must explicitly opt in to OTLP. No `OTEL_EXPORTER_OTLP_*` standard environment variables are honored; configuration is exclusively via the ConfigMap API.

#### Checklist assessment
- ✅ Project can export traces and metrics via OTLP
- ✅ OTLP endpoint is configurable (via ConfigMap)
- ✅ Both HTTP and gRPC are supported for metrics; HTTP only for traces
- ❌ OTLP is not the default (Prometheus is the default for metrics; tracing is disabled by default)
- ❌ Standard `OTEL_*` environment variables are not supported

#### Rationale
Level 2 is appropriate: OTLP export is available and configurable, but requires explicit operator action to enable. The absence of `OTEL_*` env var support and the Prometheus-first default prevent Level 3.

---

### 2. Semantic Conventions

**Level: 1 — Partial / Custom**

#### Evidence

##### Trace attributes
Span-level attributes are minimal and project-specific:
- `pipelinerun` — name of the PipelineRun (e.g., `"hello-pipeline-run-5hqfs"`)
- `namespace` — Kubernetes namespace (e.g., `"default"`)
- `taskrun` — name of the TaskRun (e.g., `"hello-taskrun-2hlr7"`)

These are **custom attributes**, not OTel semantic convention attributes. The current OTel semantic conventions for CI/CD pipelines are defined under `cicd.*` (e.g., `cicd.pipeline.name`, `cicd.pipeline.run.id`) — none of these are used. The resource attribute `service.name` is set correctly per OTel conventions.

**Schema URL**: `https://opentelemetry.io/schemas/1.12.0` is set on the `resourceSpans` level — this is an outdated schema version (current stable is 1.29.0+), but its presence is a positive signal.

Instrumentation scope names (`PipelineRunReconciler`, `TaskRunReconciler`) have no version set (`null`).

##### Metric names and attributes
Tekton-domain metrics use a custom `tekton_pipelines_controller_*` namespace:
- `tekton_pipelines_controller_pipelinerun_duration_seconds` (histogram)
- `tekton_pipelines_controller_pipelinerun_taskrun_duration_seconds` (histogram)
- `tekton_pipelines_controller_pipelinerun_total` (sum/counter)
- `tekton_pipelines_controller_running_pipelineruns` (gauge)
- `tekton_pipelines_controller_taskrun_duration_seconds` (histogram)
- `tekton_pipelines_controller_taskrun_total` (sum/counter)
- `tekton_pipelines_controller_taskruns_pod_latency_milliseconds` (histogram)
- `tekton_pipelines_controller_running_pipelineruns_waiting_on_pipeline_resolution` (gauge)
- `tekton_pipelines_controller_running_pipelineruns_waiting_on_task_resolution` (gauge)
- `tekton_pipelines_controller_running_taskruns` (gauge)
- `tekton_pipelines_controller_running_taskruns_waiting_on_task_resolution_count` (gauge)

These names do not follow OTel metric naming conventions (which would use `.` separators and omit redundant prefixes). The metric labels (`status`, `namespace`, `pipeline`, `task`, `taskrun`) are project-specific and not aligned with any OTel semantic convention namespace.

Infrastructure metrics (from Knative layer) include `http.client.request.duration` (histogram) with attributes `http.request.method`, `server.address`, `server.port`, `url.scheme`, `url.template` — these **do** align with current OTel HTTP semantic conventions (not deprecated `http.method`/`http.url`).

The `kn.workqueue.*` and `kn.k8s.client.*` metrics use a custom `kn.` prefix, not OTel conventions.

**Metric schema URL**: `https://opentelemetry.io/schemas/1.40.0` is set for the `tekton_pipelines_controller` scope — a much more current version than the trace schema.

##### Log attributes
No OTLP logs are emitted. Tekton logs are written to stdout in structured JSON format (via `zap`) but are not exported via OTLP.

#### Checklist assessment
- ✅ `service.name` is set correctly
- ✅ Schema URL is present on traces and metrics
- ✅ Knative-layer HTTP metrics use current OTel HTTP semconv attributes
- ❌ Span attributes use custom keys instead of OTel `cicd.*` semantic conventions
- ❌ Metric names use `_` separators and a verbose custom prefix rather than OTel naming conventions
- ❌ Instrumentation scope versions are not set
- ❌ Schema URL on traces is pinned to v1.12.0 (outdated)

#### Rationale
Level 1: The project uses OTel SDK correctly and has schema URLs, but metric names and span attributes are custom/project-specific rather than aligned with OTel semantic conventions. The Knative HTTP layer is a positive exception.

---

### 3. Resource Attributes & Configuration

**Level: 2 — SDK-standard with gaps**

#### Evidence

##### Native resource attributes
From source (`tracing.go`, `createTracerProvider`):
```go
tracesdk.WithResource(resource.NewWithAttributes(
    semconv.SchemaURL,
    semconv.ServiceNameKey.String(service),
))
```
Only `service.name` is explicitly set in the trace resource. The Go OTel SDK auto-populates `telemetry.sdk.name`, `telemetry.sdk.language`, and `telemetry.sdk.version`.

For metrics, the resource includes `service.name`, `service.version` (git SHA `"7798558"`), and SDK attributes.

**Gap**: `service.version` is set to a git commit SHA (`"7798558"`) rather than the release version (`"v1.12.0"`). This makes version-based filtering in observability backends unreliable.

**Gap**: No `service.instance.id` is set, which would be valuable for distinguishing controller replicas.

##### OTEL_* environment variable support
No support. Configuration is exclusively via the `config-observability` and `config-tracing` ConfigMaps. Standard `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`, etc. are not honored.

##### Identity consistency across signals
- **Traces**: `service.name` = `"pipelinerun-reconciler"` or `"taskrun-reconciler"` (reconciler-specific names)
- **Metrics**: `service.name` = `"tekton-pipelines-controller"` or `"events-controller"` (component-level names)

There is an **identity inconsistency** between traces and metrics: the same controller pod emits traces with `service.name = "pipelinerun-reconciler"` but metrics with `service.name = "tekton-pipelines-controller"`. This makes cross-signal correlation by `service.name` impossible without additional enrichment.

#### Checklist assessment
- ✅ `service.name` is set natively by the project
- ✅ `telemetry.sdk.*` attributes are present (via Go SDK auto-detection)
- ✅ Schema URL is set
- ❌ `service.version` uses git SHA instead of release version
- ❌ No `service.instance.id`
- ❌ `service.name` differs between traces and metrics for the same process
- ❌ No `OTEL_*` environment variable support

#### Rationale
Level 2: The project sets core resource attributes natively using the OTel SDK, but the `service.name` inconsistency between signals and the absence of `OTEL_*` env var support are meaningful gaps that prevent Level 3.

---

### 4. Trace Modeling & Context Propagation

**Level: 2 — Coherent internal traces with cross-boundary propagation**

#### Evidence

##### Span structure
Every reconciliation event produces a well-structured trace with a clear root span and a logical hierarchy of child spans. Example for a `PipelineRun:Reconciler` trace:

```
PipelineRun:Reconciler (root)
├── PipelineRun:ReconcileKind
│   ├── reconcile
│   │   ├── resolvePipelineState
│   │   ├── runNextSchedulableTask
│   │   │   ├── createTaskRuns
│   │   │   │   └── createTaskRun
│   │   │   │       └── TaskRun:ReconcileKind (cross-reconciler)
│   │   │   │           ├── prepare
│   │   │   │           │   └── updateTaskRunWithDefaultWorkspaces
│   │   │   │           ├── reconcile
│   │   │   │           │   └── createPod
│   │   │   │           ├── finishReconcileUpdateEmitEvents
│   │   │   │           │   └── updateLabelsAndAnnotations
│   │   │   │           └── durationAndCountMetrics
│   │   └── updatePipelineRunStatusFromInformer
│   ├── finishReconcileUpdateEmitEvents
│   │   └── updateLabelsAndAnnotations
│   └── durationAndCountMetrics
```

The trace captures the full lifecycle from `PipelineRun:Reconciler` through `TaskRun:ReconcileKind` child spans — this is a genuine cross-reconciler distributed trace within the same process.

All 1,621 spans observed are `kind=1` (INTERNAL). No SERVER or CLIENT spans are used, which is appropriate for an internal reconciler but means Kubernetes API calls are not individually traced.

##### Context propagation
**W3C Trace Context propagation is implemented** at the pod boundary. Tekton injects the `traceparent` header as a Kubernetes pod annotation (`tekton.dev/taskrunSpanContext`), enabling step containers to continue the trace:

```json
{"traceparent":"00-40bd776347cd69c42532e97e98ccc74b-b08100353d919233-01"}
```

Confirmed: trace ID `40bd776347cd69c42532e97e98ccc74b` appears in both the pod annotation and the active span data. This is a strong capability — step workloads that are OTel-instrumented can participate in the same trace as the Tekton controller.

From source (`tracing.go`):
```go
otel.SetTextMapPropagator(propagation.TraceContext{})
```
W3C TraceContext propagation is explicitly configured.

##### Trace coherence
Traces tell a coherent story of a reconciliation lifecycle. However:
- Kubernetes API calls (e.g., `PATCH /apis/tekton.dev/v1/pipelineruns/{name}`) are **not** traced as individual spans — they appear only in the `http.client.request.duration` metric
- There is no trace continuity between separate reconcile loops for the same PipelineRun (each reconcile starts a new root span rather than continuing a long-running trace)
- No span events or links are used

#### Checklist assessment
- ✅ Root spans are created for each reconciliation
- ✅ Parent-child relationships correctly model the reconciliation hierarchy
- ✅ W3C TraceContext propagation is configured
- ✅ Trace context is propagated to pod annotations for step workloads
- ❌ All spans are `kind=INTERNAL` — no CLIENT spans for Kubernetes API calls
- ❌ No span events for significant state transitions
- ❌ Separate reconcile loops for the same resource are not linked

#### Rationale
Level 2: Tekton has a well-modeled internal trace hierarchy and implements cross-boundary propagation via pod annotations. The lack of CLIENT spans for Kubernetes API calls and the absence of span events for lifecycle transitions are gaps, but the overall trace quality is high for an internal reconciler.

---

### 5. Multi-Signal Observability

**Level: 1 — Two signals, no correlation**

#### Evidence

##### Signal availability
- **Traces**: First-class, pushed via OTLP HTTP. Rich reconciler spans covering full lifecycle.
- **Metrics**: First-class, pushed via OTLP HTTP (or Prometheus scrape). 11 Tekton-domain metrics plus infrastructure metrics.
- **Logs**: Not available via OTLP. Tekton writes structured JSON logs to stdout using `zap`, but there is no OTLP log exporter. The `logs.jsonl` file is empty (0 bytes).

##### Cross-signal correlation
There is **no trace context in metrics** — metrics data points do not carry `traceId` or `spanId` attributes, making it impossible to correlate a specific metric data point with a trace.

There is **no trace context in logs** — logs cannot be correlated with traces at all since they are not exported via OTLP.

The `service.name` inconsistency (traces use `"pipelinerun-reconciler"`, metrics use `"tekton-pipelines-controller"`) further complicates cross-signal correlation.

##### Collection model per signal
| Signal | Protocol | Default | Notes |
|--------|----------|---------|-------|
| Metrics | OTLP HTTP or Prometheus scrape | Prometheus (port 9090) | Must opt in to OTLP |
| Traces | OTLP HTTP only | Disabled | Must opt in via ConfigMap |
| Logs | stdout (JSON/zap) | Always on | No OTLP path available |

#### Checklist assessment
- ✅ Two signals (traces, metrics) available via OTLP
- ✅ Metrics include both domain-specific and infrastructure metrics
- ❌ Logs have no OTLP export path
- ❌ No trace context in metrics data points
- ❌ No trace context in logs
- ❌ `service.name` differs between traces and metrics

#### Rationale
Level 1: Tekton provides two signals (traces and metrics) via OTLP, but they cannot be correlated with each other or with logs. The absence of any OTLP log path is a significant gap for a Level 2 rating.

---

### 6. Audience & Signal Quality

**Level: 1 — Functional but operator-oriented**

#### Evidence

##### Span naming
Span names reflect internal Go reconciler function names rather than user-facing operations:
- `PipelineRun:Reconciler`, `PipelineRun:ReconcileKind` — controller internals
- `reconcile`, `prepare`, `resolvePipelineState` — internal method names
- `finishReconcileUpdateEmitEvents`, `durationAndCountMetrics` — internal bookkeeping
- `updateLabelsAndAnnotations`, `updateTaskRunWithDefaultWorkspaces` — internal Kubernetes operations
- `createPod`, `createTaskRun`, `createTaskRuns`, `stopSidecars` — more interpretable

An operator familiar with Tekton internals can derive value from these spans. A user unfamiliar with the reconciler architecture will find the naming opaque. There are no user-facing span names like `"pipeline.run"` or `"task.execute"`.

##### Signal-to-noise ratio
The trace volume is well-proportioned — every reconcile event produces a bounded set of spans. No obvious noise spans were observed.

Metrics include some potentially high-cardinality concerns: `tekton_pipelines_controller_taskruns_pod_latency_milliseconds` has `task` and `taskrun` labels that could be unbounded (acknowledged in the documentation: see [#9393](https://github.com/tektoncd/pipeline/issues/9393)).

##### Default usability
- Metrics are marked **"experimental"** in the official documentation. This signals instability to operators.
- The default `metrics-protocol: prometheus` means OTLP is not used out of the box.
- Tracing is **disabled by default** — operators must explicitly configure it.
- No pre-built dashboards or alert rules are shipped with the project.
- The Tekton documentation does include a metrics reference table and a migration guide, which aids operator onboarding.

#### Checklist assessment
- ✅ Span names are consistent and follow a pattern
- ✅ Metrics cover the key operational dimensions (duration, count, running, pod latency)
- ❌ Span names are internal function names, not user-facing operation names
- ❌ All Tekton metrics are labeled "experimental" in documentation
- ❌ No pre-built dashboards or example queries shipped
- ❌ Tracing is opt-in and disabled by default
- ❌ High-cardinality risk acknowledged but not resolved for pod latency metric

#### Rationale
Level 1: The telemetry is functional and informative for Tekton contributors and experienced operators, but the internal naming, experimental status, and opt-in-only posture limit out-of-the-box usability.

---

### 7. Stability & Change Management

**Level: 2 — Breaking changes documented with migration guide**

#### Evidence

##### Documentation of telemetry behavior
The `docs/metrics.md` file provides a comprehensive reference table for all metrics, including types, labels, and status (experimental). The `config-observability.yaml` and `config-tracing.yaml` ConfigMaps include inline documentation of all supported keys.

##### Change communication
A dedicated `docs/metrics-migration-otel.md` migration guide was published for the OpenCensus → OpenTelemetry migration (PR [#9043](https://github.com/tektoncd/pipeline/pull/9043)). This guide:
- Explicitly calls out breaking changes with "Action Required" flags
- Provides a full mapping table of old → new metric names
- Covers configuration key changes (`metrics.backend-destination` → `metrics-protocol`)
- Distinguishes high-impact (infrastructure metrics) from low-impact (core Tekton metrics) changes

The v1.12.0 release notes do not mention telemetry changes, as the OTel migration was in a prior release.

##### Schema URL presence
- Traces: `https://opentelemetry.io/schemas/1.12.0` (set on `resourceSpans`)
- Metrics (`tekton_pipelines_controller` scope): `https://opentelemetry.io/schemas/1.40.0`
- Metrics (`go.opentelemetry.io/contrib/instrumentation/runtime` scope): `https://opentelemetry.io/schemas/1.40.0`
- Metrics (`knative.dev/pkg/observability/metrics/k8s` scope): `https://opentelemetry.io/schemas/1.40.0`
- Prometheus-scraped metrics: no schema URL (expected for Prometheus receiver)

The trace schema URL is pinned to `v1.12.0` of the OTel semconv spec (the version of the semconv Go package imported in `tracing.go`: `semconv "go.opentelemetry.io/otel/semconv/v1.12.0"`). This is outdated but consistently applied.

##### Stability guarantees
All Tekton metrics are explicitly labeled "experimental" in the documentation, meaning no backward-compatibility guarantee is made. The migration guide acknowledges that metric names changed as a breaking change. There is no published stability policy for telemetry signals.

#### Checklist assessment
- ✅ Metrics are documented with a reference table
- ✅ Configuration options are documented in ConfigMaps and docs
- ✅ A dedicated migration guide exists for the OTel transition
- ✅ Schema URLs are present on all OTLP exports
- ❌ All metrics are labeled "experimental" — no stability guarantee
- ❌ Trace schema URL is pinned to outdated semconv v1.12.0
- ❌ No published telemetry stability policy or versioning commitment

#### Rationale
Level 2: The project demonstrates good change management practices (dedicated migration guide, documented breaking changes) and has schema URLs on all exports. The "experimental" label on all metrics and the absence of a formal stability commitment prevent Level 3.

---

## Key findings

### Strengths

1. **Genuine OTLP-native implementation**: Both traces and metrics are exported via the OTel Go SDK using OTLP HTTP, not via a Prometheus bridge or third-party adapter. The implementation uses `otlptracehttp` and the Knative OTel metrics layer directly.

2. **W3C TraceContext propagation to pod workloads**: Tekton injects the `traceparent` header as a pod annotation (`tekton.dev/taskrunSpanContext`), enabling step containers to participate in the controller's distributed trace. This is a sophisticated and valuable capability for pipeline observability.

3. **Rich reconciler trace hierarchy**: Each PipelineRun and TaskRun reconciliation produces a well-structured multi-level trace that accurately models the controller's execution path, including cross-reconciler spans (PipelineRun → TaskRun within the same trace).

4. **Dedicated OTel migration documentation**: The `metrics-migration-otel.md` guide clearly communicates the OpenCensus → OpenTelemetry breaking changes with action items, metric name mapping tables, and configuration changes. This is exemplary change management.

5. **Schema URLs present on all OTLP exports**: Both traces and metrics carry `schemaUrl`, enabling schema-aware processing in OTel pipelines.

### Areas for improvement

1. **Add OTLP log export**: Tekton's structured `zap` logs are valuable for debugging but are completely absent from OTLP. Adding an OTLP log exporter (or at minimum documenting how to collect logs alongside traces) would enable cross-signal correlation and complete the three-signal picture.

2. **Align `service.name` between traces and metrics**: The same controller pod emits traces with `service.name = "pipelinerun-reconciler"` and metrics with `service.name = "tekton-pipelines-controller"`. This prevents cross-signal correlation by `service.name`. A consistent identity (e.g., `"tekton-pipelines-controller"` for both) with reconciler differentiation via a span attribute would resolve this.

3. **Set `service.version` to the release tag**: Currently `service.version = "7798558"` (a git SHA). Using the release version (`"v1.12.0"`) would make version-based filtering in observability backends reliable.

4. **Adopt OTel CI/CD semantic conventions for span attributes**: Span attributes `pipelinerun` and `namespace` should be mapped to `cicd.pipeline.run.id` and `k8s.namespace.name` (or equivalent OTel semconv keys). This would enable generic OTel tooling to understand Tekton traces without project-specific knowledge.

5. **Make OTLP the default or provide a guided setup**: Prometheus is the default for metrics and tracing is disabled by default. Providing a "quickstart" that enables OTLP out of the box (or at minimum a prominent documentation path) would lower the barrier to OTel adoption for Tekton operators.

6. **Graduate metrics from "experimental" status**: All Tekton metrics carry an "experimental" label in documentation, signaling no stability guarantee. Stabilizing the core metrics (`pipelinerun_duration_seconds`, `taskrun_duration_seconds`, `pipelinerun_total`, `taskrun_total`) would give operators confidence to build dashboards and alerts.

### Notable observations

- **`service.version` is a git SHA, not a release tag**: This is easy to overlook but has practical impact. Operators filtering by version in Datadog, Grafana, or Jaeger will find `"7798558"` instead of `"v1.12.0"`.

- **The trace schema URL is pinned to OTel semconv v1.12.0**: The Go import in `tracing.go` is `semconv "go.opentelemetry.io/otel/semconv/v1.12.0"`. The SDK version is 1.43.0 but the semconv package version is not updated. This creates a schema URL mismatch with the metrics layer which uses v1.40.0.

- **Knative infrastructure metrics use current OTel HTTP semconv**: The `http.client.request.duration` metric (from the Knative layer) uses `http.request.method`, `server.address`, `server.port`, `url.scheme`, `url.template` — all current (non-deprecated) OTel HTTP semconv attributes. This is better semconv alignment than the Tekton-domain metrics.

- **Pod latency metric has unbounded cardinality**: `tekton_pipelines_controller_taskruns_pod_latency_milliseconds` includes `task` and `taskrun` labels that grow with every TaskRun. This is acknowledged in the documentation (issue [#9393](https://github.com/tektoncd/pipeline/issues/9393)) but not yet resolved.

- **Tracing supports basic auth for the OTLP endpoint**: The `config-tracing` ConfigMap supports a `credentialsSecret` key for HTTP Basic Auth on the OTLP endpoint. This is a useful security feature for Tekton operators sending traces to authenticated collectors.

---

## Methodology notes

- Telemetry was collected using an OpenTelemetry Collector (v0.150.1) with file exporter in a local kind cluster (`otel-eval-tekton`)
- The `k8sattributes` processor was used to enrich telemetry; native vs. enriched attributes were distinguished by comparing the `service.name`-only resource in traces with the full enriched resource in the JSONL files
- 8 TaskRuns and 4 PipelineRuns (2-task sequential) were executed to generate representative telemetry
- Semantic conventions were checked against the current stable OTel specification (HTTP semconv: `http.request.method`, `http.response.status_code`, `url.path`, `url.full`)
- Source code was reviewed at tag `v1.12.0` on GitHub for `pkg/tracing/tracing.go`, `pkg/apis/config/tracing.go`, `pkg/apis/config/metrics.go`, and `cmd/controller/main.go`
- Documentation was reviewed at `docs/metrics.md` and `docs/metrics-migration-otel.md` at tag `v1.12.0`
