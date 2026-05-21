### 3. Resource Attributes & Configuration

**Level: 3 — OpenTelemetry-Optimized**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik v3.7.0 sets the following resource attributes natively via the OTel Go SDK across **all three signals** (traces, metrics, logs):

| Attribute | Value observed | Signal(s) |
|---|---|---|
| `service.name` | `traefik` | Traces, Metrics (OTLP), Logs |
| `service.version` | `3.7.0` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.name` | `opentelemetry` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.language` | `go` | Traces, Metrics (OTLP), Logs |
| `telemetry.sdk.version` | `1.43.0` | Traces, Metrics (OTLP), Logs |
| `host.name` | `traefik-7cd47954cc-npktt` (pod name) | Traces, Metrics (OTLP), Logs |
| `os.type` | `linux` | Traces, Metrics (OTLP), Logs |
| `os.description` | `Alpine Linux 3.23.4 (Linux ...)` | Traces, Metrics (OTLP), Logs |
| `process.pid` | `1` | Traces, Metrics (OTLP), Logs |
| `process.executable.name` | `traefik` | Traces, Metrics (OTLP), Logs |
| `process.executable.path` | `/usr/local/bin/traefik` | Traces, Metrics (OTLP), Logs |
| `process.owner` | `traefik` | Traces, Metrics (OTLP), Logs |
| `process.runtime.name` | `go` | Traces, Metrics (OTLP), Logs |
| `process.runtime.version` | `go1.25.9` | Traces, Metrics (OTLP), Logs |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` | Traces, Metrics (OTLP), Logs |
| `process.command_args` | Full CLI args array | Traces, Metrics (OTLP), Logs |
| `k8s.pod.name` | `traefik-7cd47954cc-npktt` | Traces, Metrics (OTLP), Logs |
| `k8s.pod.uid` | `65fd3a22-73fb-4c3e-a49d-4a5d20397a3d` | Traces, Metrics (OTLP), Logs |
| `k8s.namespace.name` | `traefik` | Traces, Metrics (OTLP), Logs |

**Note on native K8s attributes**: Traefik implements its own `K8sAttributesDetector` (in `pkg/types/k8sdetector.go`) that queries the Kubernetes API at startup to discover `k8s.pod.uid`, `k8s.pod.name`, and `k8s.namespace.name`. These three attributes are **project-native** — emitted by Traefik itself without any Collector enrichment.

##### Pipeline-derived resource attributes (added by Collector enrichment)

The following attributes were **added by the OTel Collector's `k8sattributes` processor** and do **not** count toward the maturity level:

- `k8s.deployment.name` — added by k8sattributes
- `k8s.replicaset.name` — added by k8sattributes
- `k8s.node.name` — added by k8sattributes
- `k8s.container.name` — added by k8sattributes
- `k8s.pod.start_time` — added by k8sattributes
- `k8s.pod.label.*` (e.g., `k8s.pod.label.app.kubernetes.io/name`) — added by k8sattributes
- `k8s.pod.annotation.*` — added by k8sattributes
- `container.id` — added by k8sattributes (container runtime)

**Note on Prometheus-scraped metrics**: The `service.instance.id` attribute (`traefik-metrics.traefik.svc.cluster.local:9100`) is set by the OTel Collector's Prometheus receiver when scraping the Traefik Prometheus endpoint — it is pipeline-derived, not native. The OTLP-pushed metrics from Traefik do not include `service.instance.id`.

##### service.name consistency across signals

- Traces: `traefik`
- Metrics (OTLP push): `traefik`
- Logs: `traefik`
- **Consistent: yes** — identical value across all three OTLP signals

##### service.version presence

**Present** — value `3.7.0` (the actual Traefik application version, not the chart version) observed consistently in traces, metrics, and logs. Set programmatically via `semconv.ServiceVersion(version.Version)` in the source.

##### OTEL_* env var support

**Documented and implemented end-to-end for all three signals.**

Source code evidence (`pkg/observability/types/tracing.go`, `pkg/observability/metrics/otel.go`, `pkg/observability/types/logs.go`):

All three signal providers use the identical resource construction pattern:
```go
resource.New(ctx,
    resource.WithContainer(),
    resource.WithHost(),
    resource.WithOS(),
    resource.WithProcess(),
    resource.WithTelemetrySDK(),
    resource.WithDetectors(K8sAttributesDetector{}),
    // Config-level values (serviceName, version, resourceAttributes)
    resource.WithAttributes(semconv.ServiceName(serviceName), semconv.ServiceVersion(version.Version)),
    resource.WithAttributes(resAttrs...),  // tracing.resourceAttributes / metrics.otlp.resourceAttributes
    // OTEL_* env vars take highest precedence — placed LAST
    resource.WithFromEnv(),
)
```

`resource.WithFromEnv()` reads both `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_SERVICE_NAME` (confirmed via OTel Go SDK source). It is positioned **last** in the resource chain, meaning it overrides all config-file and programmatic values.

**Documentation**: The official Traefik docs ([reference/install-configuration/observability/tracing](https://doc.traefik.io/traefik/reference/install-configuration/observability/tracing/)) explicitly state:
> "Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes."

**Configuration precedence** (from source code comments):
```
// The following order allows the user to override the service name and version,
// as well as any other attributes set by the above detectors.
// Use the environment variables to allow overriding above resource attributes.
```

The precedence chain is: SDK detectors < `tracing.serviceName`/`tracing.resourceAttributes` < `OTEL_RESOURCE_ATTRIBUTES`/`OTEL_SERVICE_NAME`.

**`tracing.serviceName`** defaults to `"traefik"` and is configurable via Helm values. The same pattern applies independently to metrics (`metrics.otlp.serviceName`) and logs (`logs.general.otlp.serviceName` / `logs.access.otlp.serviceName`).

**Caveat**: `OTEL_SERVICE_NAME` is not explicitly mentioned in the docs (only `OTEL_RESOURCE_ATTRIBUTES`), but it works because `resource.WithFromEnv()` in the Go SDK reads both. The docs do not document configuration precedence explicitly, though the source code comments are clear.

##### Identity misplacement

**None.** No `service.name`, `service.version`, `deployment.*`, or `cloud.*` attributes were found on spans. Identity attributes are correctly placed in the resource scope only.

Traefik span attributes are exclusively semantic-convention HTTP attributes:
- `entry_point`, `http.request.method`, `network.protocol.version`, `http.request.body.size`, `url.path`, `url.query`, `url.scheme`, `user_agent.original`, `server.address`, `network.peer.address`, `client.address`, `client.port`, `network.peer.port`, `http.response.status_code` (server spans)
- `http.request.method`, `url.full`, `url.scheme`, `network.peer.address`, `network.peer.port`, `server.address`, `server.port`, `http.response.status_code` (client/ReverseProxy spans)

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` hard-coded or always the same generic value (e.g. always "proxy", "app")? | **No** | `service.name` defaults to `"traefik"` but is configurable via `tracing.serviceName` and overridable via `OTEL_SERVICE_NAME` |
| Does `service.name` differ between signals (different value in traces vs metrics)? | **No** | `"traefik"` observed consistently in traces, OTLP metrics, and logs |
| Are `service.version` and instance identity absent? | **No** | `service.version: 3.7.0` present on all three signals |
| Are identity attributes placed on spans instead of resources? | **No** | All identity attributes are in resource scope only |
| Is `OTEL_RESOURCE_ATTRIBUTES` ignored or overridden? | **No** | `resource.WithFromEnv()` is last in the chain, giving it highest precedence |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present and stable but `service.version` missing? | **No** | Both `service.name` and `service.version` are present and consistent |
| Is configuration precedence between project config and `OTEL_*` unclear? | **Partially** | Precedence is clear in source code but only `OTEL_RESOURCE_ATTRIBUTES` is documented; `OTEL_SERVICE_NAME` is not explicitly mentioned |
| Are Kubernetes/platform attributes only available through Collector enrichment? | **No** | Traefik has a native `K8sAttributesDetector` that emits `k8s.pod.uid`, `k8s.pod.name`, `k8s.namespace.name` without Collector help |
| Does identity differ between signals or exporters? | **No** | Identical resource construction pattern across all three signal providers |
| Does `OTEL_RESOURCE_ATTRIBUTES` work only in some environments? | **No** | Works in all environments via `resource.WithFromEnv()` |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present, stable, and consistent across traces/metrics/logs? | **Yes** | `"traefik"` on all three signals |
| Is `service.version` present and consistent? | **Yes** | `"3.7.0"` on all three signals |
| Are `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` respected end-to-end? | **Yes** | `resource.WithFromEnv()` is last in all three signal providers |
| Are identity attributes in resource scope, not duplicated on spans? | **Yes** | No identity attributes found on spans |
| Are Kubernetes attributes available via standard OTel resource detection (even if pipeline-derived)? | **Yes** | Traefik's native `K8sAttributesDetector` detects `k8s.pod.uid`, `k8s.pod.name`, `k8s.namespace.name` natively |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is resource attribute behavior explicitly documented? | **Yes** | `tracing.serviceName`, `tracing.resourceAttributes`, and `OTEL_RESOURCE_ATTRIBUTES` are documented in official docs |
| Is configuration precedence (project defaults vs `OTEL_*`) clearly explained? | **Partially** | Source code comments are explicit (`"Use the environment variables to allow overriding above resource attributes"`); docs mention `OTEL_RESOURCE_ATTRIBUTES` but don't show the full precedence chain; `OTEL_SERVICE_NAME` is not documented |
| Are identity changes treated as breaking changes? | **Yes** | `service.name` defaults to `"traefik"` (stable across versions); `GlobalAttributes` was deprecated in favor of `ResourceAttributes` with backward-compat migration code |
| Are resource attributes immutable at runtime (no runtime mutation of `service.name`)? | **Yes** | Resource is built once at startup via `resource.New()` and passed to the provider; no runtime mutation observed |
| Does documentation explain identity behavior across shared clusters/multi-tenant deployments? | **Partially** | Kubernetes resource detection is documented; `K8sAttributesDetector` failure modes (host network mode) are documented with workarounds; multi-tenancy is not explicitly addressed |

---

#### Rationale

Traefik v3.7.0 reaches **Level 3 — OpenTelemetry-Optimized** based on the following evidence:

1. **Consistent identity across all signals**: `service.name: traefik` and `service.version: 3.7.0` are present and identical on traces, OTLP metrics, and logs. The same resource construction pattern is applied in all three signal providers (`tracing.go`, `metrics/otel.go`, `logs.go`).

2. **OTEL_* env vars respected with highest precedence**: `resource.WithFromEnv()` is explicitly placed last in the resource chain in all three signal providers, with source code comments confirming this is intentional to allow env var override of all config-file values.

3. **Documented configuration**: `tracing.serviceName`, `tracing.resourceAttributes`, and `OTEL_RESOURCE_ATTRIBUTES` are documented in the official reference docs. The source code comments explain the override semantics.

4. **Stable identity across versions**: The `service.name` default (`"traefik"`) is stable and the deprecated `GlobalAttributes` was migrated to `ResourceAttributes` with backward-compat code, indicating identity stability is treated as a concern.

5. **Native K8s resource detection**: Traefik implements its own `K8sAttributesDetector` that natively emits `k8s.pod.uid`, `k8s.pod.name`, and `k8s.namespace.name` by querying the Kubernetes API — this is not pipeline-derived.

6. **No identity misplacement**: All identity attributes are strictly in the resource scope; no `service.*` attributes were found on spans.

The two minor gaps that prevent a "perfect" Level 3 are:
- `OTEL_SERVICE_NAME` is not explicitly documented (only `OTEL_RESOURCE_ATTRIBUTES`), though it works via the Go SDK.
- Configuration precedence is not shown as a formal table in the docs (though the source comments are clear).
- Multi-tenant/shared-cluster identity guidance is absent.

These gaps are minor and do not affect the practical behavior. The overall resource attribute design is intentional, consistent, and well-governed.
