### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik v3.7.1 uses the OTel Go SDK directly and emits a rich set of resource attributes natively via its own `resource.New(...)` call in `pkg/types/tracing.go`:

| Attribute | Value observed |
|---|---|
| `service.name` | `traefik` |
| `service.version` | `3.7.1` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-6d96b569d5-sthkv` (pod hostname) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-6d96b569d5-sthkv 6.17.0-1013-azure ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.10` |
| `process.runtime.description` | `go version go1.25.10 linux/amd64` |
| `process.command_args` | (CLI args array) |

Additionally, Traefik's built-in Kubernetes resource detector (`k8sdetector.go`) natively emits the following when running in a Kubernetes cluster:
- `k8s.namespace.name` — `traefik`
- `k8s.pod.uid` — `b1cfa336-e0df-4b08-8d08-ad421d2b82b4`
- `k8s.pod.name` — `traefik-6d96b569d5-sthkv`

These three k8s attributes are documented in the [Traefik tracing reference](https://doc.traefik.io/traefik/reference/install-configuration/observability/tracing/) as "automatically discovered when running in a Kubernetes cluster."

##### Pipeline-derived resource attributes (added by Collector enrichment)

The OTel Collector's `k8sattributes` processor enriches telemetry with additional Kubernetes metadata using `k8s.pod.uid` (emitted natively by Traefik) for pod association. These attributes are **not** emitted by Traefik itself:

| Attribute | Source |
|---|---|
| `k8s.node.name` | k8sattributes processor |
| `k8s.deployment.name` | k8sattributes processor |
| `k8s.replicaset.name` | k8sattributes processor |
| `k8s.container.name` | k8sattributes processor |
| `k8s.pod.start_time` | k8sattributes processor |
| `k8s.pod.label.*` (all labels) | k8sattributes processor |
| `k8s.pod.annotation.*` (all annotations) | k8sattributes processor |
| `container.id` | k8sattributes processor (container metrics) |
| `service.instance.id` (Prometheus-scraped metrics only) | Prometheus receiver (scrape endpoint URL) |

##### service.name consistency across signals

- Traces: `traefik` ✅
- Metrics (OTLP push): `traefik` ✅
- Metrics (Prometheus scrape): `traefik` ✅
- Logs: `traefik` ✅
- **Consistent: Yes** — `service.name` is identical across all three signal types and both metrics collection methods.

Note: Traces also contain `service.name: otel-eval-backend` from the backend test service, which is a separate instrumented application and not a Traefik identity issue.

##### service.version presence

**Present and consistent** — `service.version: 3.7.1` observed on traces, metrics (OTLP), and logs. This is set programmatically from `version.Version` in the Go source, ensuring it always matches the running binary.

##### OTEL_* env var support

**Partially documented:**

- **`OTEL_RESOURCE_ATTRIBUTES`**: Explicitly documented in the [Traefik tracing reference](https://doc.traefik.io/traefik/reference/install-configuration/observability/tracing/) under the `resourceAttributes` section: *"Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes."* Also documented in the metrics OTLP section.
- **`OTEL_SERVICE_NAME`**: **Not explicitly documented** in Traefik docs. However, it is **functionally supported** because the OTel Go SDK's `resource.WithFromEnv()` detector (called in `resource.New(...)`) reads `OTEL_SERVICE_NAME` and the SDK's merge order gives later detectors precedence — meaning `OTEL_SERVICE_NAME` **does override** the `tracing.serviceName` config value at runtime.
- **Configuration precedence**: Traefik provides `tracing.serviceName` (default: `"traefik"`) as its own config knob. The `OTEL_RESOURCE_ATTRIBUTES` env var is documented as an alternative. However, the precedence order between `tracing.serviceName`, `tracing.resourceAttributes`, and `OTEL_RESOURCE_ATTRIBUTES`/`OTEL_SERVICE_NAME` is not explicitly documented. Source code analysis shows env vars win over config-file values due to SDK merge order.

##### Identity misplacement

**None observed.** No `service.name`, `service.version`, or other identity attributes were found as span-level attributes. All identity attributes are correctly placed in the resource scope across all signals.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` hard-coded or always the same generic value (e.g. always "proxy", "app")? | Partially — default is `"traefik"` (configurable) | `tracing.serviceName` defaults to `"traefik"`; overridable via config or `OTEL_SERVICE_NAME` |
| Does `service.name` differ between signals? | No | `traefik` on traces, metrics, and logs |
| Are `service.version` and instance identity absent? | No | `service.version: 3.7.1` present on all signals |
| Are identity attributes placed on spans instead of resources? | No | No identity attributes found on spans |
| Is `OTEL_RESOURCE_ATTRIBUTES` ignored or overridden? | No | Documented and supported via `resource.WithFromEnv()` |

**Level 0 does NOT apply.**

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present and stable but `service.version` missing? | No | Both present and consistent |
| Is configuration precedence between project config and `OTEL_*` unclear? | Partially | `OTEL_RESOURCE_ATTRIBUTES` is documented; `OTEL_SERVICE_NAME` override behavior is not explicitly documented |
| Are Kubernetes/platform attributes only available through Collector enrichment? | Partially | `k8s.namespace.name`, `k8s.pod.uid`, `k8s.pod.name` are native; others are pipeline-derived |
| Does identity differ between signals or exporters? | No | Consistent across all signals |
| Does `OTEL_RESOURCE_ATTRIBUTES` work only in some environments? | No | Works across all OTLP-pushed signals |

**Level 1 is exceeded in most areas.**

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present, stable, and consistent across traces/metrics/logs? | **Yes** | `traefik` on all signals |
| Is `service.version` present and consistent? | **Yes** | `3.7.1` on all signals |
| Are `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` respected end-to-end? | **Yes (functionally)** | `OTEL_RESOURCE_ATTRIBUTES` documented; `OTEL_SERVICE_NAME` works via SDK's `resource.WithFromEnv()` |
| Are identity attributes in resource scope, not duplicated on spans? | **Yes** | No identity attrs on spans |
| Are Kubernetes attributes available via standard OTel resource detection? | **Yes (partially native)** | `k8s.namespace.name`, `k8s.pod.uid`, `k8s.pod.name` native; others pipeline-derived |

**Level 2 is met.**

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is resource attribute behavior explicitly documented? | Partially | `tracing.serviceName`, `tracing.resourceAttributes`, `OTEL_RESOURCE_ATTRIBUTES` are documented; `OTEL_SERVICE_NAME` override behavior is not |
| Is configuration precedence (project defaults vs `OTEL_*`) clearly explained? | No | Precedence between `tracing.serviceName` and `OTEL_SERVICE_NAME` is not documented |
| Are identity changes treated as breaking changes? | Unknown | No changelog evidence of versioning policy for resource attributes |
| Are resource attributes immutable at runtime? | Yes | Resource is built once at startup; no runtime mutation observed |
| Does documentation explain identity behavior across shared clusters/multi-tenant deployments? | No | No guidance on multi-tenant identity isolation |

**Level 3 is NOT met** — documentation gaps around `OTEL_SERVICE_NAME` precedence and multi-tenant identity guidance prevent reaching this level.

---

#### Rationale

Traefik v3.7.1 achieves **Level 2 (OpenTelemetry-Native)** for resource attributes and configuration.

The project emits a comprehensive and consistent set of native resource attributes across all three signal types (traces, metrics, logs) using the OTel Go SDK directly. `service.name` (`traefik`) and `service.version` (`3.7.1`) are stable, identical across signals, and correctly placed in the resource scope — never duplicated on individual spans or metric data points. The telemetry SDK attributes (`telemetry.sdk.*`), process attributes (`process.*`), and OS attributes (`os.*`) are all set natively.

Traefik also natively detects three Kubernetes resource attributes (`k8s.namespace.name`, `k8s.pod.uid`, `k8s.pod.name`) via its own `k8sdetector`, which is documented and distinguishes it from projects that rely entirely on pipeline enrichment for k8s identity.

`OTEL_RESOURCE_ATTRIBUTES` is explicitly documented and functional. `OTEL_SERVICE_NAME` is functionally supported (via the OTel SDK's `resource.WithFromEnv()` detector with correct merge precedence), but is not explicitly documented in Traefik's own docs — this is the primary gap preventing Level 3.

The remaining Level 3 gaps are: (1) `OTEL_SERVICE_NAME` override behavior is undocumented, (2) configuration precedence rules are not explicitly stated, and (3) there is no documentation about identity behavior in multi-tenant or shared-cluster deployments.
