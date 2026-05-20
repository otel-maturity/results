### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik v3.7.0 uses the OTel Go SDK (v1.43.0) with automatic resource detection, emitting a rich set of resource attributes natively across all three OTLP signals:

| Attribute | Value | Signal(s) |
|---|---|---|
| `service.name` | `traefik` | Traces, Metrics (OTLP), Logs |
| `service.version` | `3.7.0` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.name` | `opentelemetry` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.language` | `go` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.version` | `1.43.0` | Traces, Metrics (OTLP), Logs |
| `host.name` | `traefik-7cd47954cc-st58x` (pod name) | Traces, Metrics (OTLP), Logs |
| `os.type` | `linux` | Traces, Metrics (OTLP), Logs |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-7cd47954cc-st58x ...)` | Traces, Metrics (OTLP), Logs |
| `process.pid` | `1` | Traces, Metrics (OTLP), Logs |
| `process.executable.name` | `traefik` | Traces, Metrics (OTLP), Logs |
| `process.executable.path` | `/usr/local/bin/traefik` | Traces, Metrics (OTLP), Logs |
| `process.owner` | `traefik` | Traces, Metrics (OTLP), Logs |
| `process.runtime.name` | `go` | Traces, Metrics (OTLP), Logs |
| `process.runtime.version` | `go1.25.9` | Traces, Metrics (OTLP), Logs |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` | Traces, Metrics (OTLP), Logs |
| `process.command_args` | (CLI args array) | Traces, Metrics (OTLP), Logs |

##### Pipeline-derived resource attributes (added by Collector enrichment)

The following attributes are **not** emitted by Traefik itself. They are injected by the OTel Collector's `k8sattributes` processor and do **not** count toward the level:

- `k8s.namespace.name`, `k8s.pod.name`, `k8s.pod.uid`, `k8s.pod.start_time`
- `k8s.deployment.name`, `k8s.replicaset.name`, `k8s.node.name`, `k8s.container.name`
- `k8s.pod.label.*` (e.g., `app.kubernetes.io/instance`, `app.kubernetes.io/name`, `helm.sh/chart`)
- `k8s.pod.annotation.*` (e.g., `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path`)

Additionally, the Prometheus receiver auto-generates a `service.instance.id` (`traefik-metrics.traefik.svc.cluster.local:9100`) from the scrape target address — this is collector-derived, not emitted by Traefik natively.

##### service.name consistency across signals

- Traces: `traefik`
- Metrics (OTLP push): `traefik`
- Metrics (Prometheus scrape): `traefik`
- Logs: `traefik`
- **Consistent: yes** — identical value across all four signal paths

##### service.version presence

**Present** — `3.7.0` consistently across traces, metrics (OTLP), and logs. Matches the actual Traefik application version.

##### OTEL_* env var support

**Partially documented / not explicitly tested.** Traefik configures `service.name` via its own `tracing.serviceName` static config key (e.g., `tracing.serviceName: traefik` in Helm values), not via `OTEL_SERVICE_NAME`. The Traefik v3 documentation does not explicitly document `OTEL_SERVICE_NAME` or `OTEL_RESOURCE_ATTRIBUTES` as supported override mechanisms.

However, Traefik uses the standard OTel Go SDK, which inherently supports `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` at the SDK level. The interaction between Traefik's `tracing.serviceName` config and `OTEL_SERVICE_NAME` (i.e., which takes precedence) is not documented. No `OTEL_SERVICE_NAME` env var was set in the evaluation deployment for Traefik itself.

##### Identity misplacement

**None.** No `service.name`, `service.version`, or other identity attributes were found as span-level attributes. All identity is correctly scoped to the resource.

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` hard-coded or always the same generic value? | No | `service.name: traefik` is set via `tracing.serviceName` config (user-configurable) |
| Does `service.name` differ between signals? | No | Consistent `traefik` across traces, metrics, and logs |
| Are `service.version` and instance identity absent? | No | `service.version: 3.7.0` present across all signals |
| Are identity attributes placed on spans instead of resources? | No | No identity attributes found on spans |
| Is `OTEL_RESOURCE_ATTRIBUTES` ignored or overridden? | Unknown | Not tested for Traefik; config uses `tracing.serviceName` instead |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present and stable but `service.version` missing? | No | `service.version: 3.7.0` is present and consistent |
| Is configuration precedence between project config and `OTEL_*` unclear? | Yes | `tracing.serviceName` vs `OTEL_SERVICE_NAME` precedence is undocumented |
| Are Kubernetes/platform attributes only available through Collector enrichment? | Yes | k8s.* attrs are pipeline-derived; Traefik does not natively detect them |
| Does identity differ between signals or exporters? | No | Identity is consistent across all signal paths |
| Does `OTEL_RESOURCE_ATTRIBUTES` work only in some environments? | Unknown | Not documented; likely works via OTel Go SDK but untested |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present, stable, and consistent across traces/metrics/logs? | **Yes** | `traefik` on all three signals |
| Is `service.version` present and consistent? | **Yes** | `3.7.0` on all three signals |
| Are `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` respected end-to-end? | Partially | OTel Go SDK supports them; Traefik's own `tracing.serviceName` may take precedence — undocumented |
| Are identity attributes in resource scope, not duplicated on spans? | **Yes** | No identity attributes found on spans |
| Are Kubernetes attributes available via standard OTel resource detection (even if pipeline-derived)? | **Yes** (pipeline) | k8s.* attrs present via k8sattributes processor enrichment |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is resource attribute behavior explicitly documented? | **No** | Traefik docs cover `tracing.serviceName` but do not document the full resource attribute set or `OTEL_*` override behavior |
| Is configuration precedence (project defaults vs `OTEL_*`) clearly explained? | **No** | No documentation on `OTEL_SERVICE_NAME` vs `tracing.serviceName` precedence |
| Are identity changes treated as breaking changes? | Unknown | No changelog policy found for resource attribute stability |
| Are resource attributes immutable at runtime? | Likely yes | Static config; no evidence of runtime mutation |
| Does documentation explain identity behavior across shared clusters/multi-tenant deployments? | **No** | Not addressed in documentation |

#### Rationale

Traefik v3.7.0 earns **Level 2 — OpenTelemetry-Native**. It uses the OTel Go SDK natively and emits a comprehensive, consistent set of resource attributes — `service.name`, `service.version`, `telemetry.sdk.*`, `os.*`, and `process.*` — across all three OTLP signal paths (traces, metrics, logs) with identical values. Identity is correctly placed at the resource scope with no duplication on spans. The native resource attribute set is richer than most projects at this level, including process and OS metadata auto-detected by the Go SDK.

The gap preventing Level 3 is the absence of documentation governing resource attribute behavior: Traefik does not document the full set of resource attributes it emits, does not clarify the precedence between its `tracing.serviceName` config key and the standard `OTEL_SERVICE_NAME` env var, and does not address resource attribute stability or multi-tenant identity scenarios. Additionally, `service.instance.id` is absent from all natively-emitted OTLP signals (it appears only in Prometheus-scraped metrics, where it is collector-generated from the scrape target URL, not a Traefik-native emission).
