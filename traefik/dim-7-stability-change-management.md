### 7. Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence
- Traces: **Present** — `https://opentelemetry.io/schemas/1.40.0` (on resource spans and scope spans)
- Metrics: **Present** — `https://opentelemetry.io/schemas/1.18.0` (Prometheus-scraped metrics) and `https://opentelemetry.io/schemas/1.40.0` (OTLP-pushed metrics)
- Logs: **Present** — `https://opentelemetry.io/schemas/1.40.0`

All three signals carry OTel schema URLs, signaling that Traefik intentionally tracks semantic convention versioning.

##### Telemetry documentation
A dedicated observability reference section exists at [https://doc.traefik.io/traefik/reference/install-configuration/observability/](https://doc.traefik.io/traefik/reference/install-configuration/observability/) with subsections for Metrics, Tracing, and Logs & AccessLogs. The metrics reference page lists all emitted metric names with their types, label dimensions, and descriptions. OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) are listed with their attribute sets. The tracing reference lists span attributes and resource attributes (including Kubernetes resource attributes auto-detected since v3.5.0).

There is no explicit per-metric "stability" label (e.g., "stable" vs. "experimental") in the reference, but the docs do flag OTLP log export as experimental via the `experimental.otlpLogs` feature gate.

##### Release note quality for telemetry changes

Telemetry changes are **consistently surfaced** in both the CHANGELOG and the dedicated v3 migration guide. Multiple examples found:

**CHANGELOG examples (tagged `[metrics,otel]`, `[tracing,otel]`, `[logs,otel]`):**
- `[metrics,otel] Rename traefik_tls_certs_not_after_milliseconds to traefik_tls_certs_not_after_seconds` (v3.5.4)
- `[metrics,otel] Change request duration metric unit from millisecond to second` (v3.3.4, PR #11523)
- `[tracing,otel] Use ParentBased sampler to respect parent span sampling decision` (v3.7.0-ea)
- `[logs,otel] Add OTel-conformant trace context attributes to access logs` (v3.6.11/v3.7.0-ea.2)
- `[tracing] Follow OTel semantic conventions for root span naming` (PR #11673)
- `[middleware,metrics,tracing] Upgrade to OpenTelemetry Semantic Conventions v1.26.0` (PR #10850)
- `[middleware,tracing] Introduce trace verbosity config and produce less spans by default` (PR #11870)
- `[tracing] Use ResourceAttributes instead of GlobalAttributes` (v3.3.4, PR #11515)

**Migration guide entries (https://doc.traefik.io/traefik/migrate/v3/):**
- **v3.5.4** — "Certificate Metric Renamed with OpenTelemetry": `traefik_tls_certs_not_after_milliseconds` → `traefik_tls_certs_not_after_seconds` — change is explicitly called out with old and new name.
- **v3.5.0** — "Observability / TraceVerbosity on Routers and Entrypoints": explicit impact warning — "Existing configurations will default to minimal unless overridden, which will result in fewer spans being generated than before."
- **v3.5.0** — "K8s Resource Attributes": new RBAC required for k8s.pod.name/uid injection.
- **v3.3.4** — "OpenTelemetry Request Duration Metric": metric unit changed from milliseconds to seconds — old and new unit documented, labeled "Change Details."
- **v3.2 to v3.3** — "Tracing Global Attributes": `tracing.globalAttributes` renamed to `tracing.resourceAttributes` — migration table provided.

**v2 → v3 migration guide (https://doc.traefik.io/traefik/migrate/v2-to-v3-details/):**
- "Open Connections Metric": `traefik_entrypoint_open_connections`, `traefik_router_open_connections`, `traefik_service_open_connections` → `traefik_open_connections` — old and new names listed.
- "Configuration Reload Failures Metrics": `traefik_config_reloads_failure_total` and `traefik_config_last_reload_failure` removed — explicitly called out.
- "Tracing": full vendor tracing backends removed (Jaeger, Zipkin, Datadog, etc.), replaced by pure OTLP — migration strategies provided.
- "gRPC Metrics": status code reporting behavior changed.

##### Stable vs experimental labeling
**Partially present.** OTLP log export is explicitly gated behind `experimental.otlpLogs: true`, making it clearly labeled as experimental. The v3 migration guide labels the `traceVerbosity` change as having an "Impact" block. However, there is no systematic per-metric or per-span "stable/experimental" label in the reference documentation. The OTel semconv metrics (`http.server.request.duration`) are not labeled as stable or experimental in the docs, nor are Prometheus-style `traefik_*` metrics explicitly marked stable.

##### User-reported stability issues
No open GitHub issues found specifically about broken dashboards or alerts after metric renames. The metric rename in v3.3.4 (duration unit milliseconds → seconds) was a breaking change for users with existing Grafana dashboards or alerts, but it was documented in the migration guide. The v2→v3 open connections metric restructuring was also breaking. No community reports of silent/undocumented breakage were found — suggesting the migration guide coverage is sufficient for users who read it.

##### Instrumentation scope versions
From telemetry data:

**Traces:**
| Scope | Version |
|---|---|
| `github.com/traefik/traefik` | **unknown** (no version in trace scope) |
| `@opentelemetry/instrumentation-express` | `v0.35.0` (backend) |
| `@opentelemetry/instrumentation-http` | `v0.48.0` (backend) |

**Metrics:**
| Scope | Version |
|---|---|
| `github.com/traefik/traefik` | `v3.7.0` ✅ |
| `github.com/open-telemetry/opentelemetry-collector-contrib/receiver/k8sclusterreceiver` | `v0.150.1` |
| `github.com/open-telemetry/opentelemetry-collector-contrib/receiver/prometheusreceiver` | `v0.150.1` |

Notable discrepancy: the metrics scope includes the Traefik version (`v3.7.0`), but the traces scope does **not** include a version for the `github.com/traefik/traefik` instrumentation scope. This inconsistency suggests the tracing instrumentation scope versioning is not fully wired up.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names, attributes, or metric names change without notice across releases? | **No** | Metric renames and span changes are documented in the migration guide and CHANGELOG. |
| Are users informed of telemetry changes only after breakage? | **No** | Migration guide proactively documents changes with old/new values. |
| Is telemetry treated as an internal debugging aid with no stability expectations? | **No** | Dedicated observability reference docs exist; telemetry changes get migration guide entries. |
| Are changes driven by implementation refactors rather than user impact? | **Partially** | Some changes (semconv upgrades) follow upstream OTel conventions; impact warnings are included. |
| Is there no distinction between stable and experimental telemetry? | **Partially** | OTLP logs are explicitly experimental; other signals lack explicit labels. |
| Is schema URL absent from all signals? | **No** | Schema URLs present in all three signals. |

**Level 0 is NOT assigned** — the project clearly exceeds this level.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes mentioned in release notes inconsistently (sometimes, not always)? | **No** | Telemetry changes are consistently tagged in CHANGELOG (`[metrics,otel]`, `[tracing,otel]`, etc.) and surfaced in the migration guide. |
| Are breaking changes discovered reactively (users report broken alerts)? | **No** | Changes are proactively documented before/at release. |
| Is stability handled differently per signal (traces more stable than metrics)? | **Partially** | Logs are explicitly experimental; traces/metrics are treated similarly but trace scope lacks version. |
| Are users expected to adapt to changes without migration guidance? | **No** | Migration guide provides old/new values and remediation steps for every breaking telemetry change. |
| Is there no clear policy for what makes a telemetry change "breaking"? | **Partially** | There is no formal written policy, but in practice metric renames, unit changes, and span behavior changes all get migration guide entries. |

**Level 1 is exceeded** — the project demonstrates consistent documentation of breaking changes.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes documented clearly in release notes? | **Yes** | CHANGELOG consistently uses `[metrics,otel]`, `[tracing,otel]`, `[logs,otel]` tags; migration guide has dedicated telemetry subsections per version. |
| Is there a distinction between stable and experimental telemetry? | **Partially** | OTLP logs are explicitly `experimental`; other signals lack explicit labels but the practice is consistent. |
| Are breaking changes explicitly called out (not buried in general notes)? | **Yes** | Migration guide has dedicated sections: "Certificate Metric Renamed", "OpenTelemetry Request Duration Metric", "Tracing Global Attributes", etc. |
| Is migration guidance provided when spans/metrics are renamed or removed? | **Yes** | Old and new names, old and new units, and configuration migration tables are provided for every breaking telemetry change found. |
| Are changes reviewed with downstream user impact in mind? | **Yes** | v3.5.0 TraceVerbosity change includes explicit "Impact" block warning users about fewer spans; v3.3.4 duration unit change is documented with "Change Details". |

**Level 2 is substantially met.**

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is there a defined process for reviewing telemetry changes (design proposals, TEPs)? | **No** | No formal telemetry enhancement proposal process found. Changes are tracked via PRs/issues but no dedicated governance process. |
| Are telemetry changes evaluated for impact on usability, signal quality, and cost? | **Partially** | Some changes include impact assessments (e.g., span verbosity), but no systematic quality/cost framework. |
| Are deprecations planned and communicated with timelines? | **Partially** | Deprecations are noted (e.g., `experimental.otlpLogs` flag, `tracing.globalAttributes`), but explicit removal timelines are not always given. |
| Are migration paths standard practice (deprecated fields retained for multiple releases)? | **Partially** | Some fields are retained across minor versions, but the v3.3.4 duration unit change was immediately breaking with no transitional period. |
| Are telemetry regressions detected proactively (not just reactively)? | **No** | No evidence of automated telemetry regression tests or proactive signal quality checks. |

**Level 3 is NOT met** — the project lacks a formal governance process and proactive regression detection.

---

#### Rationale

Traefik reaches **Level 2 (OpenTelemetry-Native)** for Stability & Change Management.

The evidence strongly supports this level:

1. **Schema URLs are present** in all three signals, demonstrating intentional OTel semantic convention versioning alignment.

2. **Telemetry is treated as part of the public contract.** Every breaking telemetry change found across v3.x minor versions has a dedicated entry in the official migration guide with old/new values and explicit migration instructions. Examples include the request duration metric unit change (ms→s in v3.3.4), the TLS certificate metric rename (v3.5.4), the open connections metric restructuring (v2→v3), the tracing vendor backend removal (v2→v3), and the `tracing.globalAttributes` → `tracing.resourceAttributes` rename (v3.3).

3. **Breaking changes are explicitly called out**, not buried in general release notes. The migration guide has named subsections per telemetry change, and the CHANGELOG uses consistent `[metrics,otel]`/`[tracing,otel]`/`[logs,otel]` tags.

4. **Migration guidance is provided** for every breaking telemetry change found, including old/new metric names, configuration migration tables, and impact warnings.

5. **Partial stable/experimental labeling** exists: OTLP logs are explicitly behind an `experimental` feature gate, but Prometheus-style `traefik_*` metrics and OTel traces/metrics lack explicit stability labels in the reference documentation.

The project does **not** reach Level 3 because:
- There is no formal telemetry governance process (no TEPs or design proposals for telemetry changes).
- The v3.3.4 duration unit change was immediately breaking (no deprecation period with dual reporting).
- Trace instrumentation scope lacks a version (`vunknown` in telemetry data), indicating incomplete versioning discipline.
- No evidence of proactive telemetry regression detection or signal quality evaluation frameworks.
