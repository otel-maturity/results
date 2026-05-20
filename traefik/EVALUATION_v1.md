# OTel Maturity Evaluation: traefik

**Project:** traefik
**Version evaluated:** v3.7.0
**Evaluation run:** v1
**OTel Go SDK:** v1.43.0
**Signals observed:** Traces, Metrics, Logs (all via OTLP gRPC)
**Evaluation date:** 2025

---

## Summary Table

| # | Dimension | Level | Label |
|---|-----------|-------|-------|
| 1 | Integration Surface | **1** | OpenTelemetry-Aligned |
| 2 | Semantic Conventions | **1** | OpenTelemetry-Aligned |
| 3 | Resource Attributes & Configuration | **2** | OpenTelemetry-Native |
| 4 | Trace Modeling & Context Propagation | **2** | OpenTelemetry-Native |
| 5 | Multi-Signal Observability | **2** | OpenTelemetry-Native |
| 6 | Audience & Signal Quality | **1** | OpenTelemetry-Aligned |
| 7 | Stability & Change Management | **1** | OpenTelemetry-Aligned |

**Overall profile: 1–2 (Aligned → Native transition)**
Four dimensions at Level 1, three at Level 2. Traefik has made a genuine, first-party commitment to OpenTelemetry — especially for tracing — but inconsistencies across signals, missing semantic conventions in several layers, and informal change-management practices prevent a uniform Level 2 rating.

---

## Telemetry Overview

### Signals Observed

| Signal | Volume | Transport | Notes |
|--------|--------|-----------|-------|
| Traces | 204 spans (34 root, 170 child) | OTLP gRPC push | Scope: `github.com/traefik/traefik` |
| Metrics | 742 series | Dual: OTLP gRPC push + Prometheus scrape | Two coexisting naming schemes |
| Logs | 24 records | OTLP gRPC push (experimental feature gate) | 100% carry `traceId`/`spanId` |

### Resource Attributes (natively emitted by Traefik)

| Attribute | Value |
|-----------|-------|
| `service.name` | `traefik` |
| `service.version` | `3.7.0` |
| `telemetry.sdk.name` | `opentelemetry` |
| `telemetry.sdk.language` | `go` |
| `telemetry.sdk.version` | `1.43.0` |
| `host.name` | `traefik-7cd47954cc-st58x` (auto-detected) |
| `os.type` | `linux` |
| `os.description` | Full Alpine Linux + kernel string |
| `process.pid` | `1` |
| `process.executable.name` | `traefik` |
| `process.executable.path` | `/usr/local/bin/traefik` |
| `process.owner` | `traefik` |
| `process.runtime.name` | `go` |
| `process.runtime.version` | `go1.25.9` |
| `process.runtime.description` | `go version go1.25.9 linux/amd64` |

> Note: `k8s.*` attributes (`k8s.pod.name`, `k8s.namespace.name`, `k8s.node.name`, `k8s.deployment.name`, etc.) are injected by the OTel Collector `k8sattributes` processor and do not count as natively emitted.

---

## Dimension Evaluations

---

### Dimension 1: Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Key Findings

**What's working well:**
- All three signals (traces, metrics, logs) flow via native OTLP gRPC — no sidecars or adapters required
- Tracing is OTLP-exclusive in v3: legacy exporters (Jaeger, Zipkin, Datadog tracing) were fully removed at v3.0 — a strong positive signal
- The OTel Go SDK is used directly (`telemetry.sdk.name: opentelemetry`, `telemetry.sdk.version: 1.43.0`)
- `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` env vars are respected via `resource.WithFromEnv()` and `autoprop`
- W3C Trace Context propagation is confirmed working end-to-end

**Why it doesn't reach Level 2:**

| Gap | Impact |
|-----|--------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` not supported | Users must configure 3 separate project-specific endpoint flags per signal instead of one standard env var |
| Metrics are not OTLP-primary | Prometheus is the Helm chart default; OTLP metrics require explicit opt-in (`--metrics.otlp=true`); Datadog, InfluxDB v2, Prometheus, and StatsD are documented as co-equal backends |
| Log OTLP is experimental | `experimental.otlpLogs: true` feature gate required — not a stable integration path |
| Per-signal inconsistency | Tracing is OTLP-only (mature), metrics is OTLP opt-in alongside 4 legacy backends, logs are OTLP experimental — three different maturity levels across three signals |

**Bottom line:** Traefik v3 has made a genuine commitment to OpenTelemetry (especially for tracing), but the metrics subsystem still treats OTLP as one of five co-equal backends with Prometheus as the default, and log OTLP remains behind a feature gate. The absence of `OTEL_EXPORTER_OTLP_ENDPOINT` support means users cannot configure the project using standard OTel env vars alone.

---

### Dimension 2: Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Key Findings

##### Traces
Traefik's own instrumentation scope (`github.com/traefik/traefik`) correctly uses **current OTel HTTP semantic conventions** on spans:
- `http.request.method`, `http.response.status_code`, `url.path`, `url.full`, `url.scheme`, `server.address`, `network.peer.address`, `network.protocol.version`, `client.address`, `user_agent.original`, `http.route`

However, **11 deprecated pre-1.20 attributes** appear on 24 spans each from the upstream `@opentelemetry/instrumentation-http` (Node.js) scope in the same pipeline:
`http.method`, `http.status_code`, `http.url`, `http.target`, `http.host`, `http.scheme`, `http.flavor`, `http.user_agent`, `net.host.name`, `net.host.port`, `net.peer.port`

##### Metrics — Two Coexisting Naming Schemes

| Metric Family | Attribute Style | OTel Aligned? |
|---|---|---|
| `http.server.request.duration` | `http.request.method`, `http.response.status_code`, `network.protocol.name/version`, `server.address`, `url.scheme`, `error.type` | ✅ Yes |
| `http.client.request.duration` | Same as above + `server.port` | ✅ Yes |
| `traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*` | `code`, `method`, `protocol`, `entrypoint`, `router`, `service` | ❌ No — Prometheus-style short names |

##### Logs — Fully Proprietary
Log attributes use **PascalCase Traefik-specific naming** with zero OTel alignment:
- `RequestMethod`, `DownstreamStatus`, `ClientAddr`, `RouterName`, `ServiceName`, `KubernetesIngressName`, etc.
- Log body is a **single serialized JSON string blob** — not structured OTel attributes
- Trace correlation fields are duplicated with inconsistent casing: `TraceId`/`trace_id`, `SpanId`/`span_id`

##### Cross-Signal Inconsistency

| Concept | Traces | OTel Metrics | Traefik Metrics | Logs |
|---|---|---|---|---|
| HTTP method | `http.request.method` | `http.request.method` | `method` | `RequestMethod` |
| HTTP status | `http.response.status_code` | `http.response.status_code` | `code` | `DownstreamStatus` |
| Protocol | `network.protocol.version` | `network.protocol.version` | `protocol` | `RequestProtocol` |

##### Schema URL
Present on all three signals (`https://opentelemetry.io/schemas/1.40.0`), but does not prevent deprecated attributes or proprietary naming from persisting.

#### Rationale

Traefik is at **Level 1** because it has **partially adopted OTel conventions** but not consistently across all signals. Traefik span attributes and two OTel-standard metrics fully conform to current semconv, demonstrating clear intent. However, the majority of `traefik_*` metrics use Prometheus-style short attribute names, log attributes are entirely proprietary PascalCase, deprecated HTTP attributes coexist in the trace pipeline, and the same concept is named three different ways across signals — all of which prevent off-the-shelf OTel dashboards and alerts from working without normalization or project-specific knowledge.

---

### Dimension 3: Resource Attributes & Configuration

**Level: 2 — OpenTelemetry-Native**

#### Key Findings

##### Native Resource Attributes (emitted by Traefik itself)

Traefik v3.7.0 uses the OTel Go SDK (v1.43.0) with automatic resource detection, emitting a rich, consistent set of attributes natively across **all three OTLP signal paths**. See the Telemetry Overview section above for the full attribute table.

##### Pipeline-Derived Attributes (Collector-added, do NOT count)
All `k8s.*` attributes (`k8s.pod.name`, `k8s.namespace.name`, `k8s.node.name`, `k8s.deployment.name`, etc.) are injected by the `k8sattributes` processor. The `service.instance.id` visible in metrics is collector-generated from the Prometheus scrape target URL — not natively emitted.

##### service.name Consistency
- **Traces:** `traefik` ✅
- **Metrics (OTLP push):** `traefik` ✅
- **Metrics (Prometheus scrape):** `traefik` ✅
- **Logs:** `traefik` ✅
- **Consistent: YES**

##### service.version
Present (`3.7.0`) across all signals ✅

##### Identity Misplacement
None — all identity correctly scoped to resource ✅

##### OTEL_* Env Var Support
Traefik configures `service.name` via its own `tracing.serviceName` static config key, not via `OTEL_SERVICE_NAME`. The OTel Go SDK inherently supports `OTEL_SERVICE_NAME`, but the **precedence between `tracing.serviceName` and `OTEL_SERVICE_NAME` is undocumented**.

#### Why Level 2 (not Level 3)

Traefik falls short of Level 3 because:
1. **No documentation** of the full resource attribute set it emits
2. **Undocumented precedence** between `tracing.serviceName` config and `OTEL_SERVICE_NAME` env var
3. **No `service.instance.id`** natively emitted on OTLP signals
4. No documented policy on resource attribute stability or multi-tenant identity behavior

---

### Dimension 4: Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Key Findings

##### Span Structure

| Metric | Count |
|--------|-------|
| Total root spans | 34 |
| Total child spans | 170 |
| Multi-span traces | 20 |
| Single-span traces | 15 (all `/ping` health checks and `/metrics` scrapes — not real proxied traffic) |
| Span kind distribution | INTERNAL=117, SERVER=63, CLIENT=24 |

##### Trace Topology
Every proxied request produces a clean, coherent 7–8 span trace:
```
GET (SERVER, Traefik, kind=2)                    ← entry point
  └── ReverseProxy (CLIENT, Traefik, kind=3)     ← outbound call
        └── GET / (SERVER, backend, kind=2)       ← backend receives
              ├── middleware - query (INTERNAL)
              ├── middleware - expressInit (INTERNAL)
              ├── middleware - jsonParser (INTERNAL)
              ├── middleware - <anonymous> (INTERNAL)
              └── request handler - / (INTERNAL)
```

##### W3C Trace Context Propagation
**Confirmed working.** When the test client sends a `traceparent` header, Traefik's entry-point `GET` span correctly uses the external span ID (`00f067aa0ba902b7`) as its `parentSpanId`. This was observed across a concurrent load test with 5 simultaneous requests — all 5 Traefik `GET` spans correctly referenced the same external parent, demonstrating robust context extraction and forwarding.

##### Error Handling
`GET /nonexistent` → 404 paths maintain full trace continuity. `ReverseProxy` spans are correctly marked with `status.code=2` (ERROR) and `http.response.status_code=404`.

##### Span Links
Not used — not needed for this synchronous proxy model. Parent-child relationships are used correctly throughout.

#### Level Rationale

**Level 2** is awarded because:
- ✅ All entry-point spans consistently use `SERVER` kind (kind=2)
- ✅ Outbound proxy calls consistently use `CLIENT` kind (kind=3)
- ✅ W3C Trace Context is extracted and propagated correctly at every entry point
- ✅ Traces represent logical operations (entry point → proxy → backend handler) rather than internal function calls
- ✅ Trace topology is stable across concurrent/parallel requests and error paths
- ✅ First-party instrumentation (`github.com/traefik/traefik` scope)

**Level 3** is not awarded because:
- ❌ No span events used to enrich trace understanding
- ❌ No public evidence of architectural trace modeling reviews
- ❌ No documented sampling trade-offs or trace completeness guarantees
- ❌ Instrumentation scope version is reported as `?` (unknown) — version metadata not fully populated

---

### Dimension 5: Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Key Findings

##### All three signals are flowing natively via OTLP

| Signal | Status | Volume | Mechanism |
|--------|--------|--------|-----------|
| Traces | Flowing | 204 spans | Native OTLP push (`github.com/traefik/traefik` scope) |
| Metrics | Flowing | 742 series | Dual: native OTLP push (OTel-semantic) + Prometheus scrape (legacy `traefik_*`) |
| Logs | Flowing | 24 records | Native OTLP push (`traefik` scope, confirmed by `observedTimeUnixNano`) |

##### Cross-signal correlation is strong
- **100% of log records** (24/24) carry both `traceId` and `spanId` as first-class OTLP fields
- Verified span-level linkage: log `spanId=ec661b5455bf83d6` matches the `GET` entry-point span in trace `9e74d9394fa15869a9fa04001b2b0c13` — logs are emitted at the exact span boundary
- **6 shared attribute keys** between OTel metrics and traces: `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`

##### The metric → trace → log path works end-to-end
A user can start from a `http.server.request.duration` spike, filter by shared attributes to narrow traces, then pivot directly to correlated access logs via the shared `traceId`/`spanId` — no manual bridging required.

#### Why not Level 3
- **Dual metric pipeline friction**: Legacy `traefik_*` Prometheus metrics use different attribute keys (`entrypoint`, `code`, `method`, `router`, `service`) vs. the OTel-semantic metrics and traces, creating a re-mapping burden for users of the legacy metrics
- **No multi-signal investigative workflow documentation** — each signal is documented in isolation
- **No cardinality or signal-selection guidance** — no explicit guidance on when to use traces vs. metrics vs. logs

---

### Dimension 6: Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Key Findings

##### Span Naming — Mixed
Traefik emits spans from its own scope (`github.com/traefik/traefik`). Span names partially follow OTel HTTP conventions:
- **Partially good**: `GET` — correct HTTP method-based naming per OTel server span conventions
- **Bad**: `ReverseProxy` — exposes internal component name with no route/service context; no `http.route` attribute set on *any* Traefik-originated span
- **Internal noise**: `middleware - query`, `middleware - expressInit`, `middleware - <anonymous>`, `middleware - jsonParser` — Express.js middleware pipeline internals surfaced in traces

**Critical gap**: Traefik spans carry `url.path` but never `http.route`. An operator cannot group traces by route template or identify which Traefik router/service handled a request from trace data alone — despite this information being present in metrics and logs.

##### Log Verbosity — Low volume, but with a critical severity bug
- **24 total log records**, structured JSON access logs — one per proxied request (operationally appropriate)
- **Severity mismatch (production hazard)**: All 24 records have `severityText: "info"` but the embedded JSON body contains `"level": "panic"`. Alert rules based on OTel log severity will silently miss these events.
- **Non-OTel attribute naming**: Log attributes use PascalCase (`RouterName`, `ClientAddr`, `DownstreamStatus`) rather than OTel semantic conventions, creating a fragmented attribute landscape across signals.

##### Metric Quality — Strongest signal layer
Traefik emits a rich set of SLO-relevant metrics:
- `traefik_router_requests_total`, `traefik_router_request_duration_seconds` — partitioned by router, service, method, code (directly usable for RED dashboards)
- `traefik_service_request_duration_seconds`, `traefik_entrypoint_requests_total` — full three-tier breakdown (entrypoint → router → service)
- OTel-convention `http.server.request.duration` and `http.client.request.duration` also present

**Concerns**: Prometheus-style `traefik_*` metrics and OTel `http.*.request.duration` metrics coexist with *different attribute naming*, requiring operators to understand both systems. `url.full` in `ReverseProxy` spans includes internal pod IPs.

##### Default Production Usability
- **Metrics**: Usable without customization for SLO monitoring
- **Traces**: Require domain knowledge — no `http.route`, no router/service context on spans
- **Logs**: Log severity mismatch makes automated alerting unreliable

#### Why Level 1 and Not Level 2
The project has genuine OTel alignment intent — semantic convention attributes on spans, OTel-format metrics, trace-correlated structured logs. But three blocking gaps prevent Level 2:
1. Missing `http.route` on server spans (no trace aggregation by route)
2. Missing router/service attributes on traces (no operational routing context)
3. Log `severityText: "info"` vs. body `"level": "panic"` mismatch (alerting defect)

---

### Dimension 7: Stability & Change Management

**Level: 1 — OpenTelemetry-Aligned**

#### Summary

Traefik demonstrates meaningful but informal telemetry change management practices. It clearly surpasses Level 0, but falls short of Level 2 due to the absence of explicit breaking-change labeling, no stable/experimental distinction, and a reactive (rather than proactive) approach to telemetry stability.

#### Key Evidence

| Area | Finding |
|------|---------|
| **Schema URLs** | ✅ Present on all three signals — traces (`1.40.0`), metrics (`1.18.0` + `1.40.0`), logs (`1.40.0`) |
| **Telemetry docs** | ✅ Comprehensive metrics reference page with all `traefik_*` metric names, types, labels; explicitly references OTel semconv v1.23.1 |
| **Span reference** | ❌ No enumeration of emitted span names or span attributes |
| **Changelog quality** | ✅ Telemetry changes consistently tagged `[otel]`, `[metrics,otel]`, `[tracing,otel]` — but ❌ no "BREAKING" labels |
| **Migration guidance** | ⚠️ v2→v3 guide covers some removals; individual metric/span renames lack inline migration notes |
| **Stable/experimental labeling** | ❌ Absent — no stability annotations on any telemetry signal |
| **User-reported breakage** | Issues #11114 (metric unit), #11230 (span name), #10928 (metric removal) — all reactive |
| **Instrumentation scope version** | ❌ Reports `vunknown` — no programmatic versioning of the instrumentation library |
| **Telemetry governance process** | ❌ None documented |

#### What Would Lift Traefik to Level 2
1. Explicitly mark breaking telemetry changes as `BREAKING` in changelogs and migration guides
2. Add stable/experimental labels to metrics and span attributes in documentation
3. Provide inline migration guidance when metrics are renamed or removed (not just retroactively)
4. Populate the instrumentation scope version field (currently `vunknown`)

---

## Key Findings

### Top 3 Strengths

1. **First-class OTLP across all three signals with strong trace-log correlation.** Traefik v3 natively pushes traces, metrics, and logs via OTLP gRPC. Every log record carries `traceId` and `spanId` as first-class OTLP fields, with verified span-level linkage confirmed. The full metric → trace → log investigative path works end-to-end without manual bridging — a genuine Level 2 multi-signal capability.

2. **Rich, consistent resource identity via the OTel Go SDK.** Traefik emits 15 natively detected resource attributes (`service.name`, `service.version`, `telemetry.sdk.*`, `host.name`, `os.*`, `process.*`) consistently across all signal paths. `service.name` and `service.version` are present and correct on every signal, and no identity attributes are misplaced as span/metric attributes.

3. **Correct and robust W3C Trace Context propagation with clean trace topology.** Every proxied request produces a coherent 7–8 span trace with correct `SERVER`/`CLIENT` span kinds. External `traceparent` headers are correctly extracted and propagated, verified under concurrent load. Error paths (404s) maintain full trace continuity with correct `status.code=2` (ERROR) marking.

### Top 3 Areas for Improvement

1. **Add `http.route` and router/service context to traces.** Traefik spans carry `url.path` but never `http.route`. Operators cannot group traces by route template or identify which Traefik router/service handled a request from trace data alone — a critical gap given that this information is already present in metrics and logs. This is the single highest-impact trace improvement.

2. **Fix the log severity mismatch and migrate log attributes to OTel semconv.** All 24 log records carry `severityText: "info"` while the embedded JSON body contains `"level": "panic"` — a production alerting defect. Additionally, log attributes use proprietary PascalCase naming (`RequestMethod`, `DownstreamStatus`, `ClientAddr`) that is inconsistent with traces and metrics, preventing unified querying across signals.

3. **Standardize on OTLP as the primary backend and support `OTEL_EXPORTER_OTLP_ENDPOINT`.** Prometheus remains the Helm chart default for metrics; OTLP metrics require explicit opt-in; OTLP logs are behind an experimental feature gate; and `OTEL_EXPORTER_OTLP_ENDPOINT` is not supported. Users cannot configure Traefik using standard OTel env vars alone. Promoting OTLP as the default and implementing the standard env var would significantly improve operator experience.

### Notable Observations

- **Dual metric naming schemes create operational friction.** Two coexisting metric families (`http.server.request.duration` with OTel semconv attributes, and `traefik_*` with Prometheus-style short attribute names) require operators to understand both systems. The same concept (HTTP method) is named `http.request.method` in traces and OTel metrics, `method` in Traefik metrics, and `RequestMethod` in logs — four different names for the same field.

- **Instrumentation scope version is unpopulated.** The instrumentation scope reports `vunknown` (or `?`) rather than a proper semantic version. This prevents tooling from detecting instrumentation library upgrades and is a gap in telemetry provenance.

- **Log OTLP behind a feature gate limits production adoption.** Requiring `experimental.otlpLogs: true` to enable log OTLP means most production deployments will not have logs flowing through the OTel pipeline, undermining the otherwise strong cross-signal correlation capability.

- **Schema URLs are present but decorative.** Schema URL `https://opentelemetry.io/schemas/1.40.0` appears on all signals, but deprecated pre-1.20 attributes coexist in the trace pipeline and proprietary naming persists in logs — the schema URL is not enforced or leveraged for migration.

---

## Methodology Notes

- Evaluation based on live telemetry data collected from a Traefik v3.7.0 deployment in Kubernetes, with the OTel Collector configured to receive OTLP gRPC on all three signal paths.
- Trace data: 204 spans across 35 traces, including concurrent load tests and error injection (404 paths).
- Metric data: 742 series from both OTLP push and Prometheus scrape paths.
- Log data: 24 OTLP log records from access log pipeline.
- Resource attributes evaluated for native vs. collector-injected origin; only natively emitted attributes count toward Dimension 3 scoring.
- Dimension levels (0–3) follow the OTel Maturity Model rubric: 0=Not Implemented, 1=OpenTelemetry-Aligned, 2=OpenTelemetry-Native, 3=OpenTelemetry-Exemplary.
- This evaluation reflects the state of Traefik v3.7.0 with OTel Go SDK v1.43.0 at the time of data collection.
