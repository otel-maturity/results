### 7. Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence
- Traces: **Present** — `https://opentelemetry.io/schemas/1.40.0`
- Metrics: **Present** — `https://opentelemetry.io/schemas/1.18.0` (Prometheus receiver scope) and `https://opentelemetry.io/schemas/1.40.0` (Traefik native scope)
- Logs: **Present** — `https://opentelemetry.io/schemas/1.40.0`

Schema URLs are consistently set across all three signal types, demonstrating intentional OTel semantic convention versioning.

##### Telemetry documentation
A **dedicated observability reference section** exists at:
- https://doc.traefik.io/traefik/reference/install-configuration/observability/metrics/
- https://doc.traefik.io/traefik/reference/install-configuration/observability/tracing/
- https://doc.traefik.io/traefik/observe/metrics/
- https://doc.traefik.io/traefik/observe/tracing/

The metrics reference page provides a **full table of all emitted metric names**, their types (Count/Gauge/Histogram), label dimensions, and descriptions for each of the supported backends (OTel, Prometheus, Datadog, InfluxDB2, StatsD). Traefik also maintains **official Grafana dashboards** (IDs 17346 and 17347) for on-premises and Kubernetes deployments, signalling that the metrics surface is treated as a user-facing contract. There is **no explicit "stable" vs "experimental" labeling** applied to individual metrics or spans in the documentation.

##### Release note quality for telemetry changes
Telemetry changes are **consistently tagged and documented** in the CHANGELOG. Examples from recent releases:

- **v3.5.4** (migration guide entry): *"Starting with v3.5.4, and when using OpenTelemetry, the `traefik_tls_certs_not_after_milliseconds` metric is renamed to `traefik_tls_certs_not_after_seconds`. This change aligns the metric name with its real unit precision, which is in seconds."* — Breaking change called out explicitly with a dedicated migration guide section.

- **v3.7.0-ea.2** (changelog): *`[logs, otel]` Add OTel-conformant trace context attributes to access logs (#12801)* — Additive OTel change documented in release notes.

- **v3.7.0** (changelog): *`[logs, metrics, tracing]` Bump go.opentelemetry.io/otel (#13100)* — OTel SDK bumps are tracked per-release.

- **v3.7.0-ea.1** (changelog): *`[accesslogs, otel]` Allow Stdio access logs alongside OTLP logging (#12307)* — New telemetry capability documented.

- **v3.6.0-era** (changelog): *`[metrics, otel]` Rename `traefik_tls_certs_not_after_milliseconds` to `traefik_tls_certs_not_after_seconds` (#12141)* — Metric rename visible in CHANGELOG.

- **v3.5.0** (migration guide): *TraceVerbosity on Routers and Entrypoints* — New `traceVerbosity` option introduced; migration guide explicitly notes **"If you rely on tracing, review your configuration to explicitly set the desired verbosity level. Existing configurations will default to `minimal` unless overridden, which will result in fewer spans being generated than before."* Impact is called out for users.

- **v2-to-v3 migration guide** (detailed): Dedicated "Observability" section documents:
  - Open Connections metric replaced: `traefik_entrypoint_open_connections`, `traefik_router_open_connections`, `traefik_service_open_connections` → `traefik_open_connections`
  - `traefik_config_reloads_failure_total` and `traefik_config_last_reload_failure` metrics removed (with explanation)
  - gRPC metric status code behavior changed
  - Tracing revamped — vendor-specific exporters (Datadog, Jaeger, Zipkin, etc.) removed in favour of pure OTel; migration paths (OTLP ingestion endpoints or OTel collector) are documented
  - InfluxDB v1 provider removed; remediation steps provided
  - Access log `ServiceURL` field type change noted

Release notes consistently use the `[otel]`, `[metrics]`, `[tracing]`, `[logs]` tags to categorise telemetry changes, making them easy to scan.

##### Stable vs experimental labeling
**Absent at the individual signal level.** There is no per-metric or per-span "stable"/"experimental" badge in Traefik's docs. However, at the feature level, Traefik does use an `experimental` flag for whole provider features (e.g. the Kubernetes Ingress NGINX provider was marked experimental until v3.6.2). The OTel semantic conventions version pinned in the schema URL (`1.40.0`) provides indirect versioning intent. No formal telemetry stability tiers are defined.

##### User-reported stability issues
- No prominent GitHub issues found specifically about broken dashboards or alerts after telemetry changes.
- The v3.5.4 metric rename (`traefik_tls_certs_not_after_milliseconds` → `traefik_tls_certs_not_after_seconds`) is the most impactful recent breaking telemetry change; it was proactively documented in both the CHANGELOG and the migration guide before release.
- The v3.5.0 traceVerbosity change (fewer spans by default) was similarly documented with explicit impact notes.
- No evidence of users discovering telemetry breakage reactively without prior notice.

##### Instrumentation scope versions
From telemetry data:

**Traces:**
| Scope Name | Version |
|---|---|
| `github.com/traefik/traefik` | `unknown` (no version set) |
| `@opentelemetry/instrumentation-express` | `v0.35.0` |
| `@opentelemetry/instrumentation-http` | `v0.48.0` |

**Metrics:**
| Scope Name | Version |
|---|---|
| `github.com/traefik/traefik` | `v3.7.1` |
| `github.com/open-telemetry/opentelemetry-collector-contrib/receiver/k8sclusterreceiver` | `v0.150.1` |
| `github.com/open-telemetry/opentelemetry-collector-contrib/receiver/prometheusreceiver` | `v0.150.1` |

Notable: Traefik's own metrics scope carries a proper semantic version (`v3.7.1`), directly tying the instrumentation to the release. The traces scope does not carry a version, which is a minor gap.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names, attributes, or metric names change without notice across releases? | **No** | Metric renames are documented in CHANGELOG and migration guide (e.g. v3.5.4 `_milliseconds` → `_seconds`). |
| Are users informed of telemetry changes only after breakage? | **No** | Changes are announced in advance in release notes and migration guides. |
| Is telemetry treated as an internal debugging aid with no stability expectations? | **No** | Official Grafana dashboards exist; metrics are documented with full reference tables. |
| Are changes driven by implementation refactors rather than user impact? | **Partially** | Some changes are OTel semconv alignment; impact is still documented. |
| Is there no distinction between stable and experimental telemetry? | **Yes** | No per-signal stable/experimental labeling exists. |
| Is schema URL absent from all signals? | **No** | Schema URLs present on all three signals. |

→ Project clearly exceeds Level 0.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes mentioned in release notes inconsistently (sometimes, not always)? | **No** | `[otel]`, `[metrics]`, `[tracing]` tags are consistently applied in CHANGELOG. |
| Are breaking changes discovered reactively (users report broken alerts)? | **No** | Metric renames and span behavior changes are documented proactively. |
| Is stability handled differently per signal (traces more stable than metrics)? | **Partially** | Trace scope version is absent; metrics scope has explicit version. |
| Are users expected to adapt to changes without migration guidance? | **No** | Dedicated migration guide sections exist for all significant telemetry changes. |
| Is there no clear policy for what makes a telemetry change "breaking"? | **Yes** | No published formal policy distinguishing breaking vs non-breaking telemetry changes. |

→ Project clearly exceeds Level 1.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes documented clearly in release notes? | **Yes** | Consistent `[otel]`/`[metrics]`/`[tracing]` tagging in CHANGELOG; dedicated migration guide sections. |
| Is there a distinction between stable and experimental telemetry? | **No** | No per-signal labeling. Feature-level experimental flags exist but not per metric/span. |
| Are breaking changes explicitly called out (not buried in general notes)? | **Yes** | Metric renames and span behavior changes get dedicated migration guide sections (e.g. v3.5.4, v3.5.0, v2→v3 observability section). |
| Is migration guidance provided when spans/metrics are renamed or removed? | **Yes** | Explicit before/after metric names, remediation steps, and configuration examples provided. |
| Are changes reviewed with downstream user impact in mind? | **Yes** | v3.5.0 migration note explicitly warns about fewer spans; v2→v3 migration explains impact on dashboards/monitoring. |

→ Project meets Level 2 substantially.

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is there a defined process for reviewing telemetry changes (design proposals, TEPs)? | **No** | No formal telemetry enhancement proposal process found. |
| Are telemetry changes evaluated for impact on usability, signal quality, and cost? | **Partially** | v3.5.0 traceVerbosity change reduced default span count with explicit rationale; but no formal review gate. |
| Are deprecations planned and communicated with timelines? | **Partially** | Feature deprecations (e.g. `trustForwardHeader`) are documented; metric deprecations are less formal. |
| Are migration paths standard practice (deprecated fields retained for multiple releases)? | **Partially** | Some deprecations span multiple releases; metric renames are not always dual-emitted during transition. |
| Are telemetry regressions detected proactively (not just reactively)? | **Unknown** | No public evidence of telemetry regression test suite or dashboard-based regression detection. |

→ Project does not yet reach Level 3.

---

#### Rationale

Traefik is assessed at **Level 2 — OpenTelemetry-Native**.

The project treats telemetry as a meaningful user-facing contract. Schema URLs are present on all three signals. The metrics reference documentation is comprehensive, listing every emitted metric with its type, dimensions, and description. Telemetry changes are consistently tagged in the CHANGELOG using `[otel]`, `[metrics]`, and `[tracing]` prefixes, making them easy to identify. Breaking changes — such as the v3.5.4 metric rename (`traefik_tls_certs_not_after_milliseconds` → `traefik_tls_certs_not_after_seconds`) and the v3.5.0 traceVerbosity default change — receive dedicated sections in the migration guide with explicit impact statements and remediation steps. The v2→v3 migration guide contains a full "Observability" section enumerating every metric and tracing breaking change, including removal of vendor-specific trace exporters. Official Grafana dashboards signal that metrics are considered a stable integration surface.

The project falls short of Level 3 primarily because:
1. There is no formal per-signal stable/experimental labeling — users cannot tell which metrics are guaranteed stable vs subject to change.
2. No formal telemetry change review process (design proposals, TEPs) is publicly documented.
3. Metric renames do not appear to follow a dual-emission deprecation window (old and new name emitted simultaneously across multiple releases).
4. The traces instrumentation scope does not carry a version (`unknown`), a minor gap compared to the versioned metrics scope.
5. No evidence of proactive telemetry regression detection (e.g. automated dashboard tests or signal quality gates in CI).
