### 7. Stability & Change Management

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Schema URL presence
- Traces: **Present** — `https://opentelemetry.io/schemas/1.40.0`
- Metrics: **Present** — `https://opentelemetry.io/schemas/1.18.0` (collector-side) and `https://opentelemetry.io/schemas/1.40.0` (Traefik-side)
- Logs: **Present** — `https://opentelemetry.io/schemas/1.40.0`

All three signals carry schema URLs, signaling intentional versioning of semantic conventions.

##### Telemetry documentation
Traefik has a dedicated **Observability** documentation section with multiple reference pages:
- `https://doc.traefik.io/traefik/reference/install-configuration/observability/metrics/` — lists all metric names, types, labels, and descriptions for both OTel and Prometheus formats
- `https://doc.traefik.io/traefik/reference/install-configuration/observability/tracing/` — documents span configuration options, verbosity levels, and resource attributes
- `https://doc.traefik.io/traefik/observe/overview/` — observability overview linking all signals

The metrics reference page lists every emitted metric with its name, type, label dimensions, and description (e.g., `traefik_entrypoint_requests_total`, `traefik_entrypoint_request_duration_seconds`, `http.server.request.duration`, etc.). This constitutes a clear public reference for telemetry consumers.

No explicit "stable" vs. "experimental" stability markers are applied to individual metrics or spans in the documentation.

##### Release note quality for telemetry changes

Telemetry changes are consistently tagged with `[otel]`, `[metrics]`, `[tracing]`, or `[metrics,otel]` labels in the CHANGELOG. Examples from recent releases:

- **v3.5.4** *(CHANGELOG + migration guide)*:
  > `[metrics,otel]` Rename `traefik_tls_certs_not_after_milliseconds` to `traefik_tls_certs_not_after_seconds`

- **v3.5.0** *(migration guide with Impact section)*:
  > `[middleware,tracing]` Introduce trace verbosity config and produce less spans by default. **Impact:** Existing configurations will default to `minimal` unless overridden, which will result in **fewer spans being generated than before**.

- **v3.3.4** *(migration guide entry)*:
  > `[metrics,otel]` Change request duration metric unit from millisecond to second — `traefik_(entrypoint|router|service)_request_duration_seconds` **Old Unit:** Milliseconds → **New Unit:** Seconds

- **v3.3** *(migration guide)*:
  > `[tracing]` Rename `tracing.globalAttributes` → `tracing.resourceAttributes`

- **v3.7.0-ea.2** *(CHANGELOG)*:
  > `[logs,otel]` Add OTel-conformant trace context attributes to access logs

- **v3.6.x** *(CHANGELOG)*:
  > `[otel]` Bump go.opentelemetry.io/otel dependencies; Update OpenTelemetry to v1.38.0 and semantic conventions to v1.37.0

Telemetry changes are explicitly called out in release notes and are consistently referenced back to a dedicated **migration guide** (`docs/content/migrate/v3.md`). Breaking changes include a "**Migration Required**" or "**Impact**" section with old/new values and migration steps.

##### Stable vs experimental labeling
**Partially present.** There is no per-metric or per-span stability label (e.g., "experimental", "alpha", "stable") in the telemetry reference documentation. However, Traefik uses a feature-level "experimental" concept for providers (e.g., `experimental.kubernetesgateway`, `experimental.kubernetesIngressNGINX`) which graduate to stable with explicit migration notes. This pattern does not extend to individual telemetry signals.

The migration guide does distinguish between breaking and non-breaking telemetry changes through section headers ("**Migration Required**" vs. informational notes), but there is no formal stability tier for telemetry contracts.

##### User-reported stability issues
No GitHub issues specifically about broken dashboards or alerts caused by telemetry changes were found. The metric unit change in v3.3.4 (ms → s for request duration) would have silently broken Grafana dashboards for users who did not read the migration guide; however, this was proactively documented. The v3.5.0 traceVerbosity change (fewer spans by default) was also documented with an explicit **Impact** warning.

No community reports of users discovering breakage only after the fact (reactively) were found.

##### Instrumentation scope versions
From telemetry data (Traefik v3.7.1):

**Traces:**
```
github.com/traefik/traefik   v(unknown)
@opentelemetry/instrumentation-express  v0.35.0
@opentelemetry/instrumentation-http     v0.48.0
```

**Metrics:**
```
github.com/traefik/traefik   v3.7.1
github.com/open-telemetry/opentelemetry-collector-contrib/receiver/k8sclusterreceiver  v0.150.1
github.com/open-telemetry/opentelemetry-collector-contrib/receiver/prometheusreceiver  v0.150.1
```

**Logs:**
```
traefik   v(unknown)
```

Notable: the metrics scope correctly carries `v3.7.1`, tying emitted metrics to the Traefik release version. The traces scope version is `unknown`, which is a gap — trace consumers cannot determine the instrumentation version from the telemetry data itself.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names, attributes, or metric names change without notice across releases? | **No** | Metric renames (e.g., ms→s, globalAttributes→resourceAttributes) are documented in migration guide |
| Are users informed of telemetry changes only after breakage? | **No** | Migration guide entries with "Impact" sections precede the release |
| Is telemetry treated as an internal debugging aid with no stability expectations? | **No** | Dedicated telemetry reference docs, official Grafana dashboards published |
| Are changes driven by implementation refactors rather than user impact? | **Partially** | Some changes (semconv upgrades) are dependency-driven but are still documented |
| Is there no distinction between stable and experimental telemetry? | **Mostly true** | No per-signal stability labels, but breaking changes are explicitly flagged |
| Is schema URL absent from all signals? | **No** | Schema URLs present on all three signals |

→ Project clearly exceeds Level 0.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes mentioned in release notes inconsistently (sometimes, not always)? | **No** | `[otel]`/`[metrics]`/`[tracing]` tags used consistently; migration guide maintained per-version |
| Are breaking changes discovered reactively (users report broken alerts)? | **No** | Breaking changes proactively documented with "Migration Required" and "Impact" sections |
| Is stability handled differently per signal (traces more stable than metrics)? | **Partially** | Metrics have more documented changes; traces scope version is "unknown" |
| Are users expected to adapt to changes without migration guidance? | **No** | Migration steps provided (old/new values, configuration examples) |
| Is there no clear policy for what makes a telemetry change "breaking"? | **Partially** | No formal written policy, but breaking changes are consistently identified in practice |

→ Project clearly exceeds Level 1.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are telemetry changes documented clearly in release notes? | **Yes** | Tagged entries in CHANGELOG + dedicated migration guide with per-version sections |
| Is there a distinction between stable and experimental telemetry? | **Partially** | No per-signal labels, but breaking changes are flagged; feature-level experimental concept exists |
| Are breaking changes explicitly called out (not buried in general notes)? | **Yes** | Migration guide has "Migration Required" / "Impact" sections; release notes reference migration guide |
| Is migration guidance provided when spans/metrics are renamed or removed? | **Yes** | Old/new names, configuration examples, and impact statements provided |
| Are changes reviewed with downstream user impact in mind? | **Yes** | v3.5.0 traceVerbosity note explicitly states "if you rely on tracing, review your configuration" |

→ Project substantially meets Level 2.

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Is there a defined process for reviewing telemetry changes (design proposals, TEPs)? | **No** | No formal telemetry change proposal process found; changes go through standard PRs |
| Are telemetry changes evaluated for impact on usability, signal quality, and cost? | **Partially** | Some changes include impact analysis (traceVerbosity), but no systematic evaluation framework |
| Are deprecations planned and communicated with timelines? | **Partially** | Some deprecations announced (e.g., `trustForwardHeader`, `rootCAsSecrets`), but telemetry-specific deprecation timelines are not always explicit |
| Are migration paths standard practice (deprecated fields retained for multiple releases)? | **Partially** | Some fields deprecated with "will be removed in next major version", but some metric renames were immediate (no dual-emit period) |
| Are telemetry regressions detected proactively (not just reactively)? | **No evidence** | No CI-based telemetry regression testing or telemetry contract tests found |

→ Project does not yet meet Level 3.

---

#### Rationale

Traefik is assessed at **Level 2 — OpenTelemetry-Native** for the following reasons:

**Strengths supporting Level 2:**
1. **Schema URLs present on all signals** — demonstrates intentional alignment with OTel versioning conventions.
2. **Comprehensive telemetry reference documentation** — a dedicated metrics reference page lists all metric names, types, dimensions, and descriptions. This is the public contract users can build dashboards against.
3. **Proactive communication of breaking telemetry changes** — the versioned migration guide (`migrate/v3.md`) has dedicated per-version sections for telemetry changes (metric unit changes in v3.3.4 and v3.5.4, tracing configuration rename in v3.3, span verbosity behavior change in v3.5.0). Release notes consistently tag telemetry changes with `[otel]`/`[metrics]`/`[tracing]` labels.
4. **User impact awareness** — the v3.5.0 traceVerbosity change includes an explicit "Impact" section warning users that fewer spans will be produced, which is a meaningful downstream impact for alert rules and dashboards.
5. **Official Grafana dashboards** — Traefik maintains and publishes official Grafana dashboards, which creates accountability for metric stability (dashboard breakage is visible and attributable).

**Gaps preventing Level 3:**
1. **No per-signal stability labels** — there is no formal "stable" vs. "experimental" labeling at the individual metric or span level. Users cannot tell which telemetry signals are safe to build long-lived SLOs on.
2. **Some breaking changes are immediate** — the request duration unit change (ms→s in v3.3.4) and metric rename in v3.5.4 were applied without a dual-emit deprecation period. Users upgrading without reading the migration guide would have silently broken dashboards.
3. **Traces scope version is "unknown"** — the traces instrumentation scope does not carry a version, limiting consumers' ability to correlate telemetry shape with a specific release.
4. **No formal telemetry change governance** — changes go through standard PRs without a dedicated telemetry impact review checklist or proposal process (TEPs/KEPs equivalent).
5. **No proactive regression detection** — no evidence of automated telemetry contract testing or CI-based checks that would catch unintended telemetry changes before release.
