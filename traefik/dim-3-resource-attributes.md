### 3. Resource Attributes & Configuration

**Level: 3 — OpenTelemetry-Optimized**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik sets the following resource attributes natively via the OTel Go SDK across all three signals (traces, metrics via OTLP push, and logs):

| Attribute | Value observed |
|---|---|
| `service.name` | `traefik` |
| `service.version` | `3.7.1` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-5fb59c7b48-c2pgw` (pod name via `resource.WithHost()`) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-5fb59c7b48-c2pgw 6.17.0-1013-azure ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.10` |
| `process.runtime.description` | `go version go1.25.10 linux/amd64` |
| `process.command_args` | (CLI args array) |

**Natively detected Kubernetes attributes** (via Traefik's built-in `K8sAttributesDetector`, not Collector enrichment):

| Attribute | Value observed |
|---|---|
| `k8s.pod.name` | `traefik-5fb59c7b48-c2pgw` |
| `k8s.pod.uid` | `d1e20c59-c3f8-42ef-8d27-fd32a5650310` |
| `k8s.namespace.name` | `traefik` |

Traefik implements a custom `K8sAttributesDetector` (in `pkg/types/k8sdetector.go`) that queries the Kubernetes API using in-cluster credentials to detect `k8s.pod.name`, `k8s.pod.uid`, and `k8s.namespace.name` natively at the source — without relying on the Collector.

**Note on Prometheus-scraped metrics:** The `service.instance.id: traefik-metrics.traefik.svc.cluster.local:9100` attribute seen in the metrics JSONL is added by the OTel Collector's Prometheus receiver (using the scrape target address as the instance ID). This is **not** natively emitted by Traefik.

##### Pipeline-derived resource attributes (added by Collector enrichment)

The following were added by the OTel Collector's `k8sattributes` processor (not emitted by Traefik):

- `k8s.container.name: traefik`
- `k8s.deployment.name: traefik`
- `k8s.replicaset.name: traefik-5fb59c7b48`
- `k8s.node.name: otel-eval-traefik-control-plane`
- `k8s.pod.start_time: 2026-05-22T07:02:02Z`
- `k8s.pod.annotation.*` (prometheus.io/path, prometheus.io/port, prometheus.io/scrape)
- `k8s.pod.label.*` (app.kubernetes.io/instance, app.kubernetes.io/managed-by, etc.)

##### service.name consistency across signals

| Signal | `service.name` value |
|---|---|
| Traces | `traefik` |
| Metrics (OTLP push) | `traefik` |
| Logs | `traefik` |
| **Consistent** | **Yes** |

##### service.version presence

Present and consistent across all three signals: `3.7.1` (the actual running app version, injected from `version.Version` at build time).

##### OTEL_* env var support

**Documented and explicitly supported in source code:**

- `OTEL_RESOURCE_ATTRIBUTES` is **documented** in the Traefik official docs under the `resourceAttributes` section for both tracing and metrics:
  > "Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes."
- `OTEL_SERVICE_NAME` is not separately called out in docs, but is honoured by the OTel Go SDK's `resource.WithFromEnv()` which is explicitly used in all three signal providers (tracing, metrics, logs).
- Source code analysis confirms that all three signal resource builders (tracing in `pkg/observability/types/tracing.go`, metrics in `pkg/observability/metrics/otel.go`, logs in `pkg/observability/types/logs.go`) follow the **same pattern** with clear precedence ordering:
  1. `resource.WithContainer()` / `WithHost()` / `WithOS()` / `WithProcess()` / `WithTelemetrySDK()` (auto-detected)
  2. `resource.WithDetectors(K8sAttributesDetector{})` (native Kubernetes detection)
  3. `resource.WithAttributes(semconv.ServiceName(serviceName), semconv.ServiceVersion(version.Version))` (project defaults)
  4. `resource.WithAttributes(resAttrs...)` (user-configured `resourceAttributes`)
  5. `resource.WithFromEnv()` — **env vars override everything above**
- The code comment explicitly states: `"Use the environment variables to allow overriding above resource attributes."`
- Configuration precedence is clear: `OTEL_*` env vars take highest priority, overriding project defaults and `resourceAttributes` config.
- The `tracing.serviceName` / `metrics.otlp.serviceName` config options are documented with a default value of `"traefik"`, giving users a Traefik-native config path as well.

##### Identity misplacement

None observed. No `service.*`, `deployment.*`, or `cloud.*` attributes were found on span attributes. Identity is correctly placed only in the resource scope.

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` hard-coded or always the same generic value? | No | `service.name: traefik` is meaningful and configurable via `tracing.serviceName` |
| Does `service.name` differ between signals? | No | Consistent `traefik` across traces, metrics, and logs |
| Are `service.version` and instance identity absent? | No | `service.version: 3.7.1` present on all signals |
| Are identity attributes placed on spans instead of resources? | No | No misplacement observed |
| Is `OTEL_RESOURCE_ATTRIBUTES` ignored or overridden? | No | Explicitly supported via `resource.WithFromEnv()` at highest priority |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present but `service.version` missing? | No | Both present on all signals |
| Is configuration precedence unclear? | No | Documented and explicit in source code |
| Are Kubernetes attributes only via Collector enrichment? | No | Traefik has a native `K8sAttributesDetector` (pod name, uid, namespace) |
| Does identity differ between signals or exporters? | No | Consistent across all three signals |
| Does `OTEL_RESOURCE_ATTRIBUTES` work only in some environments? | No | Applied uniformly across all signal providers |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present, stable, and consistent across all signals? | Yes | `traefik` on traces, metrics, and logs |
| Is `service.version` present and consistent? | Yes | `3.7.1` on all signals |
| Are `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` respected end-to-end? | Yes | `resource.WithFromEnv()` at highest priority in all signal providers |
| Are identity attributes in resource scope, not duplicated on spans? | Yes | No misplacement detected |
| Are Kubernetes attributes available via standard OTel resource detection? | Yes | Native `K8sAttributesDetector` detects `k8s.pod.name`, `k8s.pod.uid`, `k8s.namespace.name` at the source |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is resource attribute behavior explicitly documented? | Yes | Traefik docs describe `tracing.serviceName`, `tracing.resourceAttributes`, `OTEL_RESOURCE_ATTRIBUTES`, and native Kubernetes detection in the official reference docs |
| Is configuration precedence clearly explained? | Yes | Docs describe both `resourceAttributes` option and `OTEL_RESOURCE_ATTRIBUTES` env var; source code comment explicitly states env vars override all other settings |
| Are identity changes treated as breaking changes? | Yes (implied) | `tracing.serviceName` has a stable documented default of `"traefik"` and is part of the versioned Helm chart schema |
| Are resource attributes immutable at runtime? | Yes | Resource attributes are set once during provider initialization and not mutated at runtime |
| Does documentation explain identity behavior across shared clusters/multi-tenant deployments? | Partial | The `K8sAttributesDetector` docs note that automatic detection can fail in host network mode and advise using the option or env var; however, multi-tenant identity separation is not explicitly addressed |

#### Rationale

Traefik v3.7.1 achieves **Level 3 — OpenTelemetry-Optimized** for resource attributes and configuration.

The evidence is compelling across all criteria:

1. **Complete native resource attribute set**: `service.name`, `service.version`, `telemetry.sdk.*`, `host.*`, `os.*`, `process.*` are all emitted natively by the project across every signal (traces, OTLP metrics, and logs) using the OTel Go SDK's standard resource detectors.

2. **Native Kubernetes identity detection**: Traefik implements its own `K8sAttributesDetector` that queries the Kubernetes API to populate `k8s.pod.name`, `k8s.pod.uid`, and `k8s.namespace.name` at the source — not relying on Collector pipeline enrichment for core identity.

3. **Perfect signal consistency**: `service.name: traefik` and `service.version: 3.7.1` are identical across all three signal types, with no discrepancy.

4. **Explicit, documented `OTEL_*` support with clear precedence**: All three signal providers (tracing, metrics, logs) use the same resource-building pattern with `resource.WithFromEnv()` as the final (highest-priority) step. This is documented in the official Traefik reference docs and commented in the source code: *"Use the environment variables to allow overriding above resource attributes."* `OTEL_RESOURCE_ATTRIBUTES` is explicitly called out in the docs.

5. **Zero identity misplacement**: No `service.*` or identity attributes found on span attributes — all identity is correctly in the resource scope.

6. **Intentional, governed configuration**: The `tracing.serviceName` / `metrics.otlp.serviceName` options provide a Traefik-native override path, the `resourceAttributes` map supports arbitrary additions, and `OTEL_*` env vars provide the standard override mechanism — all with documented precedence.

The only minor gap at Level 3 is that multi-tenant identity separation documentation is not explicitly addressed (e.g., running multiple Traefik instances in the same cluster with distinct `service.name` values). However, the mechanism to achieve this (`OTEL_SERVICE_NAME` or `tracing.serviceName`) is clearly documented and functional.
