### 3. Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Native resource attributes (emitted by the project)

Traefik v3.7.0 uses the OTel Go SDK (v1.43.0) and emits a rich set of resource attributes natively via OTLP push for all three signals (traces, metrics, logs). The following are confirmed as **project-native** (present on OTLP-pushed telemetry before any Collector enrichment):

| Attribute | Value observed |
|---|---|
| `service.name` | `traefik` |
| `service.version` | `3.7.0` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-7cd47954cc-rrd6j` (pod name) |
| `os.type` | `linux` |
| `os.description` | `Alpine Linux 3.23.4 (Linux traefik-7cd47954cc-rrd6j 6.17.0-1013-azure ...)` |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |
| `process.command_args` | CLI args array |

Additionally, per Traefik's own documentation, it **natively auto-detects** Kubernetes resource attributes when running in a Kubernetes cluster:
- `k8s.namespace.name` — auto-detected by Traefik
- `k8s.pod.uid` — auto-detected by Traefik
- `k8s.pod.name` — auto-detected by Traefik

These three are therefore **project-native** (not exclusively pipeline-derived), though the k8sattributes processor also adds/confirms them.

##### Pipeline-derived resource attributes (added by Collector enrichment)

The following attributes were added by the OTel Collector's `k8sattributes` processor and are **not** emitted natively by Traefik:

- `k8s.container.name` — added by k8sattributes
- `k8s.deployment.name` — added by k8sattributes
- `k8s.node.name` — added by k8sattributes
- `k8s.replicaset.name` — added by k8sattributes
- `k8s.pod.start_time` — added by k8sattributes
- `k8s.pod.label.*` (e.g., `k8s.pod.label.app.kubernetes.io/name`, `k8s.pod.label.helm.sh/chart`) — added by k8sattributes
- `k8s.pod.annotation.*` (e.g., `k8s.pod.annotation.prometheus.io/scrape`) — added by k8sattributes
- `container.id` — added by k8sattributes (from container runtime)
- `os.version` — added by k8sattributes (on trace signal only)

For Prometheus-scraped metrics (via the Collector's Prometheus receiver), the following are **additionally** added by the Collector's Prometheus receiver (not by Traefik):
- `service.instance.id` = `traefik-metrics.traefik.svc.cluster.local:9100` (Prometheus scrape target)
- `server.address` / `server.port` / `url.scheme` (scrape target metadata)
- `container.image.name` / `container.image.tag` (from cAdvisor/kubelet)

##### service.name consistency across signals

- **Traces**: `traefik`
- **Metrics** (OTLP push): `traefik`
- **Logs**: `traefik`
- **Consistent**: ✅ Yes — `service.name = "traefik"` across all three signals

Note: The trace file also contains spans from `otel-eval-backend` (the test backend service), which is a separate service and expected.

##### service.version presence

✅ **Present and consistent** — `service.version = "3.7.0"` observed on all three signals (traces, metrics, logs). The version matches the deployed app version (`v3.7.0`).

##### OTEL_* env var support

**`OTEL_RESOURCE_ATTRIBUTES`**: **Documented and supported**. The official Traefik v3 documentation explicitly states:
> "Traefik also supports the `OTEL_RESOURCE_ATTRIBUTES` env variable to set up the resource attributes."

This is documented for both tracing (`tracing.resourceAttributes`) and metrics (`metrics.otlp.resourceAttributes`).

**`OTEL_SERVICE_NAME`**: **Not explicitly documented** by Traefik. The project uses `tracing.serviceName` (CLI: `--tracing.serviceName`, default: `"traefik"`) and `metrics.otlp.serviceName` as its own configuration mechanism. These are Traefik-specific config options that set `service.name`. Since Traefik uses the standard OTel Go SDK resource detection, `OTEL_SERVICE_NAME` may be honoured at the SDK level, but this is **not explicitly documented** in Traefik's own docs.

**Configuration precedence**: Partially documented. Traefik has its own `tracing.serviceName` and `metrics.otlp.serviceName` options (both default to `"traefik"`). The relationship between these Traefik-specific options and `OTEL_SERVICE_NAME` is not clearly documented. `OTEL_RESOURCE_ATTRIBUTES` is documented as a supported alternative to `tracing.resourceAttributes`/`metrics.otlp.resourceAttributes`.

**Kubernetes resource auto-detection**: Documented. Traefik auto-detects `k8s.namespace.name`, `k8s.pod.uid`, and `k8s.pod.name` when running in Kubernetes. The docs note this can fail if running in host network mode, with guidance to use the config option or env var as fallback.

**Deployment configuration observed**: `--tracing.serviceName=traefik` is explicitly set in the deployment args, confirming the service name is set via Traefik's own config mechanism (not via `OTEL_SERVICE_NAME`).

##### Identity misplacement

✅ **No misplacement detected.** No `service.*`, `deployment.*`, or `cloud.*` attributes were found on span attributes (wrong scope). All identity attributes are correctly placed in the resource scope. No `env`, `environment`, `version`, `region`, or `cluster` attributes were found duplicated on spans.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` hard-coded or always the same generic value? | No | `service.name = "traefik"` is the project name, not a generic value like "app" or "proxy" |
| Does `service.name` differ between signals? | No | `traefik` across traces, metrics, and logs |
| Are `service.version` and instance identity absent? | No | `service.version = "3.7.0"` present on all signals |
| Are identity attributes placed on spans instead of resources? | No | All identity attributes in resource scope |
| Is `OTEL_RESOURCE_ATTRIBUTES` ignored or overridden? | No | Explicitly documented as supported |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present and stable but `service.version` missing? | No | `service.version` is present and consistent |
| Is configuration precedence between project config and `OTEL_*` unclear? | Partially | `OTEL_RESOURCE_ATTRIBUTES` is documented; `OTEL_SERVICE_NAME` precedence vs `tracing.serviceName` is not documented |
| Are Kubernetes/platform attributes only available through Collector enrichment? | Partially | k8s.namespace.name, k8s.pod.uid, k8s.pod.name are natively auto-detected; others (k8s.deployment.name, k8s.node.name, etc.) are pipeline-derived |
| Does identity differ between signals or exporters? | No | Consistent across all signals |
| Does `OTEL_RESOURCE_ATTRIBUTES` work only in some environments? | No | Documented as universally supported |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is `service.name` present, stable, and consistent across traces/metrics/logs? | ✅ Yes | `traefik` on all three signals |
| Is `service.version` present and consistent? | ✅ Yes | `3.7.0` on all three signals |
| Are `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES` respected end-to-end? | Partially | `OTEL_RESOURCE_ATTRIBUTES` is documented and supported; `OTEL_SERVICE_NAME` is not explicitly documented (project uses `tracing.serviceName` instead) |
| Are identity attributes in resource scope, not duplicated on spans? | ✅ Yes | No misplacement found |
| Are Kubernetes attributes available via standard OTel resource detection? | ✅ Yes | k8s.namespace.name, k8s.pod.uid, k8s.pod.name auto-detected natively; extended k8s attributes via k8sattributes pipeline |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is resource attribute behavior explicitly documented? | Partially | `tracing.resourceAttributes`, `metrics.otlp.resourceAttributes`, `OTEL_RESOURCE_ATTRIBUTES`, and Kubernetes auto-detection are documented; `OTEL_SERVICE_NAME` is not |
| Is configuration precedence (project defaults vs `OTEL_*`) clearly explained? | No | The relationship between `tracing.serviceName` / `metrics.otlp.serviceName` and `OTEL_SERVICE_NAME` is not documented |
| Are identity changes treated as breaking changes? | Unknown | No evidence of versioning policy for resource attributes in changelogs |
| Are resource attributes immutable at runtime? | Yes (likely) | No evidence of runtime mutation of `service.name`; it's set at startup via static config |
| Does documentation explain identity behavior across shared clusters/multi-tenant deployments? | No | No documentation on multi-tenant identity separation |

---

#### Rationale

Traefik v3.7.0 is assessed at **Level 2 — OpenTelemetry-Native**.

**Strengths supporting Level 2:**
- `service.name = "traefik"` and `service.version = "3.7.0"` are present, stable, and **consistent across all three signals** (traces, metrics, logs) via native OTLP export.
- A rich set of process/runtime/OS resource attributes is emitted natively via the OTel Go SDK.
- Traefik natively auto-detects Kubernetes resource attributes (`k8s.namespace.name`, `k8s.pod.uid`, `k8s.pod.name`) without requiring pipeline enrichment.
- All identity attributes are correctly placed in resource scope (no misplacement on spans).
- `OTEL_RESOURCE_ATTRIBUTES` is explicitly documented and supported for both tracing and metrics.
- The project uses OTel Go SDK v1.43.0 (current) with full OTLP support across all three signals.

**Why not Level 3:**
- `OTEL_SERVICE_NAME` is **not explicitly documented** by Traefik. The project uses its own `tracing.serviceName` / `metrics.otlp.serviceName` config options (both separate, both defaulting to `"traefik"`). The precedence relationship between these Traefik-specific options and standard `OTEL_SERVICE_NAME` is undocumented.
- The separation of `tracing.serviceName` and `metrics.otlp.serviceName` into two distinct config options (rather than a single resource-level configuration) is a non-standard pattern that could lead to identity divergence if configured independently.
- No documentation on resource attribute versioning policy (breaking changes), multi-tenant identity, or runtime immutability guarantees.
- `service.instance.id` is absent from traces and logs natively (present in Prometheus-scraped metrics only, where it is set to the scrape target address — a Collector-derived value, not a pod identity).
