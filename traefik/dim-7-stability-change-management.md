### 7. Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence
- Traces: **Present** — `https://opentelemetry.io/schemas/1.40.0`
- Metrics: **Present** — `https://opentelemetry.io/schemas/1.18.0` (OTel collector scopes) and `https://opentelemetry.io/schemas/1.40.0` (Traefik own scope)
- Logs: **Present** — `https://opentelemetry.io/schemas/1.40.0`

All three signals carry schema URLs, indicating intentional versioning of semantic conventions.

##### Telemetry documentation
A comprehensive telemetry reference exists in the official Traefik documentation:

- **Metrics reference:** https://doc.traefik.io/traefik/reference/install-configuration/observability/metrics/
  - Full tables of all emitted metric names, types, labels, and descriptions for every supported backend (OpenTelemetry, Prometheus, Datadog, InfluxDB2, StatsD).
  - Explicitly states: *"Traefik Proxy follows [official OpenTelemetry semantic conventions v1.23.1](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.1/docs/http/http-metrics.md)"*
  - Separate sections for Global Metrics, OTel Semantic Conventions (`http.server.request.duration`, `http.client.request.duration`), EntryPoint Metrics, Router Metrics, Service Metrics.

- **Tracing reference:** https://doc.traefik.io/traefik/reference/install-configuration/observability/tracing/
  - Full configuration option table with defaults.
  - Documents span verbosity levels (`minimal` vs `detailed`).

- **Official Grafana dashboards** maintained by Traefik for both on-premises and Kubernetes deployments (IDs 17346 and 17347), demonstrating commitment to stable metric names that users can build dashboards on.

##### Release note quality for telemetry changes

Traefik maintains a per-version migration guide (`docs/content/migrate/v3.md`) that explicitly calls out telemetry-related breaking changes. Notable examples found:

**v3.5.4 — Metric renamed with OpenTelemetry:**
> *"Starting with v3.5.4, and when using OpenTelemetry, the `traefik_tls_certs_not_after_milliseconds` metric is renamed to `traefik_tls_certs_not_after_seconds`. This change aligns the metric name with its real unit precision, which is in seconds."*

**v3.5.0 — Observability section in migration guide:**
> *"Starting with v3.5.0, a new `traceVerbosity` option is available for both entrypoints and routers... Existing configurations will default to `minimal` unless overridden, which will result in fewer spans being generated than before."*
> *"Since v3.5.0, the semconv attributes `k8s.pod.name` and `k8s.pod.uid` are injected automatically in OTel resource attributes..."*

**v3.3.4 — OTel Request Duration Metric unit change:**
> *"In v3.3.4, the OpenTelemetry Request Duration metric unit has been standardized... Old Unit: Milliseconds. New Unit: Seconds."*

**v3.3 — Tracing configuration renamed:**
> *"Old: `tracing.globalAttributes`. New: `tracing.resourceAttributes`. The old option name was misleading as it specifically adds resource attributes for the collector, not global span attributes."*

**v2→v3 major migration — Tracing overhaul:**
> *"In v3, the tracing feature has been revamped and is now powered exclusively by OpenTelemetry (OTel). Traefik v3 no longer supports direct output formats for specific vendors such as Instana, Jaeger, Zipkin, Haystack, Datadog, and Elastic."*
> *"In v3, the open connections metric has been replaced with a global one... The equivalent to `traefik_entrypoint_open_connections`, `traefik_router_open_connections` and `traefik_service_open_connections` is now `traefik_open_connections`."*
> *"In v3, the `traefik_config_reloads_failure_total` and `traefik_config_last_reload_failure` metrics have been suppressed since they could not be implemented."*

Release notes also consistently tag telemetry-related PRs with labels such as `[logs, metrics, tracing]`, `[otel]`, `[tracing, otel]`, and `[metrics, tracing, accesslogs]`, making them easy to filter.

##### Stable vs experimental labeling
- **Partially present.** The documentation does not use explicit "stable" / "experimental" / "alpha" / "beta" stability badges on individual metrics or spans.
- However, there is a broader pattern of **experimental features** being gated behind an `experimental:` configuration block (e.g., `experimental.kubernetesIngressNGINX`, `experimental.fastProxy`). Observability features are not currently marked experimental.
- The metrics documentation references a pinned OTel semantic conventions version (`v1.23.1`), implying that OTel-standard metrics follow that spec's stability guarantees. Traefik-specific metrics (`traefik_*`) have no explicit stability label but are comprehensively documented.
- No formal stability tiers (e.g., "this metric is in preview") are applied to individual telemetry signals.

##### User-reported stability issues
- No GitHub issues specifically about broken dashboards or alerts after upgrades were found in the search results.
- The migration guide entries for metric renames (v3.3.4, v3.5.4) and unit changes suggest these were proactively documented *before* users hit breakage, rather than reactively after reports.
- The `[metrics, tracing, accesslogs] Fix ObservabilityConfig SetDefaults` bug fix (PR #12636, v3.6.9) was a correctness fix, not a silent rename.

##### Instrumentation scope versions
From telemetry data:

**Traces:**
```
github.com/traefik/traefik vunknown
@opentelemetry/instrumentation-express v0.35.0
@opentelemetry/instrumentation-http v0.48.0
```

**Metrics:**
```
github.com/traefik/traefik v3.7.0
github.com/open-telemetry/opentelemetry-collector-contrib/receiver/k8sclusterreceiver v0.150.1
github.com/open-telemetry/opentelemetry-collector-contrib/receiver/prometheusreceiver v0.150.1
```

**Logs:**
```
traefik vunknown
```

Notable: The Traefik instrumentation scope carries version `v3.7.0` in metrics (the project version), but `vunknown` in traces and logs. This inconsistency means the instrumentation library version cannot be used to detect telemetry changes for traces/logs from the scope alone. Schema URLs are present for all signals, which partially compensates.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names, attributes, or metric names change without notice across releases? | **No** | Migration guide documents all telemetry changes per minor version |
| Are users informed of telemetry changes only after breakage? | **No** | Changes are pre-announced in migration docs and release notes |
| Is telemetry treated as an internal debugging aid with no stability expectations? | **No** | Official Grafana dashboards, full metrics reference docs |
| Are changes driven by implementation refactors rather than user impact? | **No** | v3.3.4 unit change explicitly justified by naming convention alignment |
| Is there no distinction between stable and experimental telemetry? | **Partially** | No per-signal stability badges, but experimental features are gated |
| Is schema URL absent from all signals? | **No** | Schema URLs present in traces, metrics, and logs |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes mentioned in release notes inconsistently (sometimes, not always)? | **No** | Consistently documented in dedicated migration guide sections |
| Are breaking changes discovered reactively (users report broken alerts)? | **No** | Changes documented proactively in per-version migration sections |
| Is stability handled differently per signal (traces more stable than metrics)? | **Partially** | Scope version is missing for traces/logs (`vunknown`) vs. metrics (`v3.7.0`) |
| Are users expected to adapt to changes without migration guidance? | **No** | Explicit migration steps, old→new option tables, and example configs provided |
| Is there no clear policy for what makes a telemetry change "breaking"? | **Partially** | No formal written policy, but practice is consistent: renames, unit changes, and removals all get migration guide entries |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes documented clearly in release notes? | **Yes** | Dedicated per-version migration guide sections for every breaking telemetry change |
| Is there a distinction between stable and experimental telemetry? | **Partial** | No per-signal labels, but experimental config features are gated; OTel semconv version is pinned |
| Are breaking changes explicitly called out (not buried in general notes)? | **Yes** | Telemetry changes have their own named sections in the migration guide (e.g., "Certificate Metric Renamed with OpenTelemetry", "OpenTelemetry Request Duration Metric") |
| Is migration guidance provided when spans/metrics are renamed or removed? | **Yes** | Old→new name tables, config examples, and migration steps provided consistently |
| Are changes reviewed with downstream user impact in mind? | **Yes** | v3.5.0 migration note explicitly warns: "Existing configurations will default to `minimal` unless overridden, which will result in fewer spans being generated than before" |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is there a defined process for reviewing telemetry changes (design proposals, TEPs)? | **No** | No public TEP/RFC process for telemetry changes found |
| Are telemetry changes evaluated for impact on usability, signal quality, and cost? | **Partially** | Unit changes are justified by convention alignment; no formal cost/quality evaluation process |
| Are deprecations planned and communicated with timelines? | **Partially** | Some deprecations have timelines (e.g., "will be removed in the next major version"), but no multi-release deprecation windows for telemetry specifically |
| Are migration paths standard practice (deprecated fields retained for multiple releases)? | **Partially** | Config options are sometimes deprecated before removal; metric renames are immediate (no dual-emission period found) |
| Are telemetry regressions detected proactively (not just reactively)? | **No** | No evidence of automated telemetry regression testing or canary-based detection |

---

#### Rationale

Traefik scores **Level 2 — OpenTelemetry-Native**. The project treats telemetry as part of its public contract in several concrete ways:

1. **Comprehensive documentation:** A full metrics reference page lists all emitted metric names, types, labels, and descriptions per backend. Official Grafana dashboards are maintained and referenced in docs, implying a commitment to stable metric names.

2. **Proactive change communication:** The per-version migration guide (`docs/content/migrate/v3.md`) has dedicated sections for every telemetry-breaking change, including metric renames (v3.5.4), unit changes (v3.3.4), configuration renames (v3.3), span verbosity behavior changes (v3.5.0), and the complete tracing overhaul in v3.0. These are not buried in general changelogs — they have their own named headings.

3. **User impact awareness:** Migration notes explicitly describe behavioral impact (e.g., fewer spans generated by default after v3.5.0), not just the mechanical change.

4. **Schema URLs present** across all three signals, signaling intent to follow versioned OTel semantic conventions.

The project falls short of Level 3 because:
- There is no formal process (TEPs, RFCs) for reviewing telemetry changes before they land.
- No formal stability tiers (stable/experimental) are applied to individual metrics or spans.
- Metric renames appear to take effect immediately without a dual-emission deprecation period spanning multiple releases.
- Trace/log instrumentation scope versions are `vunknown`, a gap compared to metrics.
- No evidence of automated telemetry regression detection.

These gaps mean that while changes are well-communicated, users must still update dashboards and alerts on each breaking change rather than relying on a defined deprecation window or dual-emission compatibility period.
