### 7. Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Schema URL presence
- Traces: **Present** — `https://opentelemetry.io/schemas/1.40.0`
- Metrics: **Present** — `https://opentelemetry.io/schemas/1.18.0` and `https://opentelemetry.io/schemas/1.40.0`
- Logs: **Present** — `https://opentelemetry.io/schemas/1.40.0`

Schema URLs are present across all three signals, indicating that Traefik has adopted OTel schema versioning conventions. However, the instrumentation scope for Traefik's own spans reports `github.com/traefik/traefik vunknown` — the version field is not populated, meaning the scope itself does not carry a version signal.

##### Telemetry documentation
A dedicated observability reference exists at:
- **Metrics**: `https://doc.traefik.io/traefik/reference/install-configuration/observability/metrics/` — lists all emitted metrics with names, types, labels, and descriptions for OTel, Prometheus, Datadog, InfluxDB2, and StatsD. Includes a "Metrics Provided" section with full tables of `traefik_*` metric names and OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`). Notes that Traefik follows [OTel semantic conventions v1.23.1](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.1/docs/http/http-metrics.md).
- **Tracing**: `https://doc.traefik.io/traefik/reference/install-configuration/observability/tracing/` — documents configuration options, sampling strategy, and resource attributes. Does not enumerate emitted span names or span attributes as a reference table.

The documentation covers configuration of telemetry exporters in detail but lacks a comprehensive span attribute reference (no list of span names, span attributes, or their stability status).

##### Release note quality for telemetry changes
Telemetry changes are mentioned in release notes, tagged with `[otel]`, `[metrics,otel]`, `[tracing,otel]`, `[logs,otel]`, etc. Examples from the CHANGELOG:

- **v3.3.x**: `[metrics,otel]` Rename `traefik_tls_certs_not_after_milliseconds` to `traefik_tls_certs_not_after_seconds` ([#12141](https://github.com/traefik/traefik/pull/12141)) — a breaking metric rename noted in the changelog but **no migration guidance** provided in the PR body.
- **v3.3.x**: `[otel]` Update OpenTelemetry to v1.38.0 and semantic conventions to v1.37.0 ([#12099](https://github.com/traefik/traefik/pull/12099)).
- **v3.2.x**: `[metrics,otel]` Change request duration metric unit from millisecond to second ([#11523](https://github.com/traefik/traefik/pull/11523)) — a breaking change to metric values, noted in changelog but no explicit "breaking change" call-out or migration note in the PR body.
- **v3.3.x**: `[tracing]` Follow OTel semantic conventions for root span naming ([#11673](https://github.com/traefik/traefik/pull/11673)) — span name changed from hardcoded `EntryPoint` to HTTP method per OTel semconv; noted in changelog.
- **v3.1.x**: `[metrics]` Mention missing metrics removal in the migration guide ([#10982](https://github.com/traefik/traefik/pull/10982)) — a retroactive fix to add migration notes after metrics were removed.
- **v3.6.x**: `[logs, otel]` Add OTel-conformant trace context attributes to access logs ([#12801](https://github.com/traefik/traefik/pull/12801)).

The pattern shows: telemetry changes are consistently tagged and mentioned in release notes, but breaking changes (metric renames, unit changes, span name changes) are not explicitly labeled as "breaking" or accompanied by migration guidance in the changelog entries themselves. The v3.1 retroactive migration guide fix (#10982) is evidence that missing metrics removal was not initially communicated.

##### Stable vs experimental labeling
**Absent.** There is no explicit stable/experimental/beta labeling for any telemetry signals in the documentation. The metrics reference lists all metrics without stability annotations. No versioning commitment (e.g., "these metrics are stable for the v3 lifecycle") is stated.

##### User-reported stability issues
- [Issue #11230](https://github.com/traefik/traefik/issues/11230): Users reported that Traefik hardcoded span names (e.g., `EntryPoint`) did not follow OTel conventions, breaking observability tooling expectations. PR #11673 addressed this by changing span names — itself a breaking change for users who had built dashboards/alerts on the old span name.
- [Issue #11114](https://github.com/traefik/traefik/issues/11114): Users reported the OTel request duration metric was in milliseconds despite the name suggesting seconds, leading to incorrect dashboards. Fixed in v3.2 by changing the unit (a breaking change for existing dashboards).
- [Issue #10928](https://github.com/traefik/traefik/issues/10928): Metrics removed in v3 without migration guidance; retroactively addressed in PR #10982.
- No evidence of a formal proactive regression detection process for telemetry.

##### Instrumentation scope versions
From telemetry data:
- `github.com/traefik/traefik vunknown` — Traefik's own instrumentation scope has no version set
- `@opentelemetry/instrumentation-express v0.35.0` — test workload instrumentation
- `@opentelemetry/instrumentation-http v0.48.0` — test workload instrumentation
- `github.com/traefik/traefik v3.7.0` — appears in resource spans (this is the resource version, not the scope version)

The absence of a version on the instrumentation scope (`vunknown`) is notable — it means there is no programmatic way to version-pin telemetry consumers to a specific instrumentation library version.

#### Checklist assessment

### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names, attributes, or metric names change without notice across releases? | **No** | Changes are mentioned in release notes with `[otel]`/`[metrics]`/`[tracing]` tags |
| Are users informed of telemetry changes only after breakage? | **Partially** | PR #10982 retroactively added migration notes after metrics removal was missed |
| Is telemetry treated as an internal debugging aid with no stability expectations? | **No** | Dedicated observability reference docs exist; Grafana dashboards are officially published |
| Are changes driven by implementation refactors rather than user impact? | **Partially** | Some changes (unit fixes, semconv alignment) are user-impact-driven; others lack migration guidance |
| Is there no distinction between stable and experimental telemetry? | **Yes** | No stable/experimental labeling exists |
| Is schema URL absent from all signals? | **No** | Schema URLs present across all three signals |

Level 0 criteria are **not fully met** — telemetry is documented and changes are tracked.

### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes mentioned in release notes inconsistently (sometimes, not always)? | **No** | Changes are consistently tagged and listed in CHANGELOG |
| Are breaking changes discovered reactively (users report broken alerts)? | **Yes** | Issue #11114 (metric unit), #11230 (span name), #10928 (metric removal) — all user-reported |
| Is stability handled differently per signal (traces more stable than metrics)? | **Partially** | No explicit per-signal stability policy; both traces and metrics have had breaking changes |
| Are users expected to adapt to changes without migration guidance? | **Partially** | Migration guide exists for v2→v3 but individual breaking telemetry changes lack inline migration notes |
| Is there no clear policy for what makes a telemetry change "breaking"? | **Yes** | No documented telemetry stability policy or definition of breaking change |

Level 1 criteria are **substantially met**.

### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes documented clearly in release notes? | **Yes** | Consistently tagged in CHANGELOG |
| Is there a distinction between stable and experimental telemetry? | **No** | No stable/experimental labeling |
| Are breaking changes explicitly called out (not buried in general notes)? | **No** | Metric renames, unit changes, and span name changes are listed without "BREAKING" labels |
| Is migration guidance provided when spans/metrics are renamed or removed? | **Partially** | v2→v3 migration guide covers some removals; individual PRs for metric renames lack migration notes |
| Are changes reviewed with downstream user impact in mind? | **Partially** | OTel semconv alignment is user-impact-driven, but no formal review process |

Level 2 criteria are **not met** — breaking changes are not explicitly labeled, no stable/experimental distinction, and migration guidance is inconsistent.

### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is there a defined process for reviewing telemetry changes (design proposals, TEPs)? | **No** | No evidence of a telemetry governance process |
| Are telemetry changes evaluated for impact on usability, signal quality, and cost? | **No** | No formal evaluation process found |
| Are deprecations planned and communicated with timelines? | **No** | No evidence of deprecation timelines for telemetry |
| Are migration paths standard practice (deprecated fields retained for multiple releases)? | **No** | Metric renames happen in single releases without transition periods |
| Are telemetry regressions detected proactively (not just reactively)? | **No** | No evidence of proactive telemetry regression detection |

Level 3 criteria are **not met**.

#### Rationale

Traefik scores **Level 1 — OpenTelemetry-Aligned** for the following reasons:

**Strengths:**
- Schema URLs are present across all three signals (traces, metrics, logs), reflecting intentional OTel schema versioning adoption.
- Telemetry changes are consistently tagged in the CHANGELOG (`[otel]`, `[metrics,otel]`, `[tracing,otel]`), making them discoverable.
- A comprehensive metrics reference page documents all emitted metric names, types, labels, and descriptions, including OTel semantic convention metrics.
- Traefik explicitly references OTel semantic conventions (v1.23.1) in its metrics documentation.
- An official Grafana dashboard is published and maintained, indicating awareness of downstream consumers.

**Weaknesses that prevent Level 2:**
- **No stable/experimental labeling**: No telemetry signal is explicitly designated as stable or experimental. Users cannot distinguish which metrics or spans carry stability guarantees.
- **Breaking changes are not explicitly called out**: Metric renames (`traefik_tls_certs_not_after_milliseconds` → `traefik_tls_certs_not_after_seconds`), unit changes (ms → s for request duration), and span name changes (hardcoded `EntryPoint` → HTTP method) appear in changelogs without "BREAKING CHANGE" markers or inline migration guidance.
- **Reactive rather than proactive**: Multiple breaking telemetry changes were surfaced through user bug reports (issues #11114, #11230, #10928) rather than proactive governance. The retroactive addition of migration notes for metrics removal (PR #10982) illustrates this pattern.
- **Instrumentation scope version absent**: Traefik's own instrumentation scope reports `vunknown`, preventing consumers from programmatically detecting instrumentation library version changes.
- **No telemetry stability policy**: There is no documented policy defining what constitutes a breaking telemetry change, what stability guarantees are offered, or what the deprecation process is.
