### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming assessment

Traefik emits exactly **two span types** from its own instrumentation scope (`github.com/traefik/traefik`):

| Span Name | Kind | Count | Assessment |
|-----------|------|-------|------------|
| `GET` | Server (2) | 115 | **Neutral-to-Bad** — HTTP method alone with no path template; ambiguous without attributes |
| `ReverseProxy` | Client (3) | 44 | **Bad** — internal component name, not a user-facing operation |

- **Good (logical/user-relevant)**: None identified from Traefik's own spans. The `GET` span does carry `url.path` and `http.response.status_code` attributes which help, but the span name itself is not self-describing.
- **Bad (internal/implementation)**: `ReverseProxy` — this is a Go struct/component name exposed directly as a span name. An operator unfamiliar with Traefik internals would not know this represents "outbound call to upstream backend".
- **Neutral**: `GET` — technically OTel-conformant for HTTP server spans (the spec allows method-only names when no route template is available), but the absence of a route template (e.g. `GET /api/{resource}`) means every unique path creates a distinct-looking span only distinguishable by attributes. At the evaluation traffic volume this was low-cardinality, but in production with many routes this becomes harder to use.

**Overall**: Mixed — `GET` is spec-compliant but minimally informative; `ReverseProxy` exposes an internal implementation detail.

Note: Spans from the backend (`middleware - query`, `middleware - jsonParser`, `middleware - expressInit`, `request handler - /`, `GET /`, `GET /health`) are emitted by the Node.js backend, not Traefik itself.

##### Log verbosity

**Log severity distribution:**

| Severity | Count |
|----------|-------|
| `info`   | 24    |
| `UNSET`  | 0     |

- **Total log records**: 44 (all from access logs; 24 captured in the evaluation window shown above)
- **Volume assessment**: **Low** — one record per HTTP request, no internal step-by-step logging
- **Quality**: Access log records are per-request state-change events (one per completed request). This is good operational practice — logs fire on meaningful boundaries, not on internal processing steps.
- **Attributes per record**: 36 fields including `RouterName`, `ServiceName`, `KubernetesIngressName`, `Duration`, `DownstreamStatus`, `TraceId`/`SpanId` — rich and operationally useful.
- **Known quirk**: All access log records carry `level: panic` in the JSON body regardless of HTTP status (e.g., a normal `200 GET /` has `"level":"panic"`). This is a Traefik v3 serialization bug where the logrus panic level is hardcoded for access log entries. This is **misleading noise** — an operator monitoring log severity would see all requests as `panic`-level events.
- **Structural quirk**: The log body is a raw JSON string (not a structured OTLP object). The fields are also duplicated as extracted OTLP attributes, but the body itself is opaque to OTLP-native consumers without JSON parsing.
- **Trace ID duplication**: Both `TraceId`/`SpanId` (camelCase, in body) and `trace_id`/`span_id` (snake_case, as OTLP attributes) are present — redundant but not harmful.
- **General application logs**: Only access logs were observed flowing via OTLP. The `experimental.otlpLogs: true` feature gate was enabled; no general/internal application log records were captured in the evaluation window (INFO-level general logs appear to be low-volume by default).

##### Metric quality

**All metrics observed (77 total unique names):**

| Category | Metrics | Assessment |
|----------|---------|------------|
| **OTel semantic convention** | `http.server.request.duration`, `http.client.request.duration` | ✅ SLO-relevant — latency histograms with status, method, route |
| **Traefik-native (entrypoint)** | `traefik_entrypoint_request_duration_seconds`, `traefik_entrypoint_requests_total`, `traefik_entrypoint_requests_bytes_total`, `traefik_entrypoint_responses_bytes_total` | ✅ SLO-relevant — per-entrypoint RED metrics |
| **Traefik-native (router)** | `traefik_router_request_duration_seconds`, `traefik_router_requests_total`, `traefik_router_requests_bytes_total`, `traefik_router_responses_bytes_total` | ✅ SLO-relevant — per-router RED metrics |
| **Traefik-native (service)** | `traefik_service_request_duration_seconds`, `traefik_service_requests_total`, `traefik_service_requests_bytes_total`, `traefik_service_responses_bytes_total` | ✅ SLO-relevant — per-backend-service RED metrics |
| **Traefik-native (config)** | `traefik_config_last_reload_success`, `traefik_config_reloads_total`, `traefik_open_connections` | ✅ Operational signals |
| **Go runtime** | `go_goroutines`, `go_memstats_*`, `go_gc_*`, `go_threads`, `go_sched_*` (~28 metrics) | ⚠️ Runtime internals — useful for debugging, not for SLO alerting |
| **Process** | `process_cpu_seconds_total`, `process_resident_memory_bytes`, `process_open_fds`, etc. (~9 metrics) | ⚠️ Infrastructure-level — useful but not proxy-specific |
| **Scrape metadata** | `scrape_duration_seconds`, `scrape_samples_*`, `scrape_series_added`, `up` | ℹ️ Collector/Prometheus housekeeping |
| **K8s cluster** | `k8s.container.*`, `k8s.deployment.*`, `k8s.pod.*`, etc. | ℹ️ Collector-derived (k8s_cluster receiver), not project-native |

**SLO-relevant metrics**: `http.server.request.duration`, `http.client.request.duration`, `traefik_entrypoint_request_duration_seconds`, `traefik_router_request_duration_seconds`, `traefik_service_request_duration_seconds`, `traefik_*_requests_total` (with status code labels)

**High-cardinality concerns**:
- `http.server.request.duration` uses `server.address` as a label (includes port), which could cause cardinality explosion in multi-tenant or high-route environments. In this evaluation, 6 unique label combinations were observed.
- `ReverseProxy` spans carry `url.full` with the full upstream URL including pod IP (`http://10.244.0.6:3000/`). In production with pod churn this is **high-cardinality** — each pod restart generates a new IP, creating unbounded unique span attribute values.
- `client.port` and `network.peer.port` appear as span attributes on server spans — ephemeral port numbers are inherently high-cardinality and provide no diagnostic value.
- Traefik-native Prometheus metrics use `router=` and `service=` labels (e.g., `router=demo-otel-eval-backend@kubernetes`) — these are bounded by configuration and are acceptable.

##### Default production usability

**Can an operator use this telemetry without heavy customization?**

**Partially yes.** The metric story is strong: Traefik provides a well-structured three-layer metric hierarchy (entrypoint → router → service) covering the full RED (Rate, Errors, Duration) pattern, plus OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`). An operator can build SLO dashboards and alerts from these metrics without any customization.

**However, the trace story has usability gaps:**

1. **`ReverseProxy` span name** requires internal knowledge — an operator must know this represents the outbound upstream call.
2. **`GET` span name** (server) carries no route template — all requests to different paths share the same span name, making trace search/filtering by operation less useful without inspecting `url.path` attributes.
3. **`url.full` with pod IPs** in `ReverseProxy` spans is high-cardinality in production and leaks infrastructure details that are not meaningful to operators.
4. **`client.port` / `network.peer.port`** on server spans are ephemeral and noisy.

**The log story is operationally rich but has a critical bug**: the `level: panic` serialization error on all access log records would cause false alerts in any monitoring system that watches log severity levels.

**OTLP log export is experimental** (`experimental.otlpLogs: true` required) — this is not production-stable, and operators cannot rely on it without accepting instability risk.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names expose internal function/class/component names? | **Partially** | `ReverseProxy` is an internal Go component name; `GET` is spec-compliant but not route-aware |
| Are logs emitted for every internal step by default (high volume of DEBUG/TRACE)? | **No** | Only access logs observed; one record per request; INFO severity only |
| Is there no distinction between debug and operational signals? | **No** | Metrics are clearly structured; logs are access-only by default |
| Do users need to heavily filter telemetry before it becomes useful? | **No** | Metrics are immediately usable; traces require understanding of `ReverseProxy` |
| Are high-cardinality attributes emitted indiscriminately? | **Partially** | `url.full` with pod IPs and `client.port`/`network.peer.port` are high-cardinality; `url.path` is present on spans |

**Level 0 does not apply** — telemetry is not purely maintainer-centric; metrics are clearly operator-oriented.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is obvious noise reduced but defaults remain conservative? | **Yes** | No DEBUG/TRACE flood; but `client.port`, `network.peer.port`, and pod-IP `url.full` add noise |
| Are some spans renamed to logical operations but others remain internal? | **Yes** | `GET` (HTTP method, spec-compliant but minimal); `ReverseProxy` (internal name) |
| Do operators need domain knowledge to interpret span names? | **Yes** | `ReverseProxy` requires knowing Traefik's architecture |
| Is signal quality inconsistent across traces, metrics, and logs? | **Yes** | Metrics are strong; traces have naming issues; logs have the `panic` severity bug |
| Are logs structured but still overly detailed for operational use? | **Partially** | 36 attributes per access log record is verbose but all are operationally relevant; the `panic` severity bug is the bigger concern |

**Level 1 applies** — OTel SDK is used, semantic conventions are partially followed, but signal quality is inconsistent and there are usability gaps.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names describe logical, user-relevant operations? | **No** | `GET` lacks route template; `ReverseProxy` is an internal component name |
| Are telemetry defaults usable in production without customization? | **Partially** | Metrics yes; traces have high-cardinality attributes; logs have the `panic` bug |
| Are logs emitted on state changes or errors — not on every internal step? | **Yes** | Access logs are per-request boundary events |
| Are metrics focused on SLO-relevant signals rather than raw internal counters? | **Mostly** | Core Traefik metrics are SLO-relevant; Go runtime metrics are bundled in |
| Can operators move from symptoms to causes without deep internal knowledge? | **Partially** | Metrics → identify affected router/service; traces → `ReverseProxy` name requires Traefik knowledge |

**Level 2 does not fully apply** — span naming and the `panic` severity log bug prevent this level.

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume, cardinality, and cost managed intentionally? | **No** | High-cardinality `url.full` (pod IPs), `client.port`, `network.peer.port` not filtered |
| Is telemetry quality evaluated using objective criteria? | **No** | No evidence of instrumentation scoring or quality gates |
| Are high-cardinality attributes avoided in favor of trace-driven investigation? | **No** | Pod IP in `url.full`, ephemeral ports in span attributes |
| Are defaults refined based on user feedback? | **Partially** | `addInternals: false` default shows some intentionality; but known issues (pod IPs, `panic` log level) persist |
| Are quality regressions detectable and addressed proactively? | **No** | `level: panic` bug in access logs is a known regression |

**Level 3 does not apply.**

---

#### Rationale

Traefik v3.7.0 is assessed at **Level 1 (OpenTelemetry-Aligned)**. The project has made genuine investment in OTel: it uses the OTel Go SDK natively, emits OTLP traces, metrics, and logs, follows HTTP semantic conventions for its metric names (`http.server.request.duration`, `http.client.request.duration`), and provides a well-structured three-layer metric hierarchy (entrypoint/router/service) that directly supports SLO alerting.

However, several signal quality issues prevent Level 2:

1. **`ReverseProxy` span name** — an internal Go component name is used as the client span name for upstream backend calls. This is the clearest indicator of maintainer-centric design: the name makes sense to a Traefik developer who knows the `ReverseProxy` struct, but not to an operator who just wants to understand "Traefik called the backend."

2. **`GET` span name without route template** — while technically spec-compliant (OTel HTTP spec allows method-only names when no route template is available), the lack of route templating means all requests share the same span name. An operator cannot filter traces by "which route was slow" without inspecting attributes — reducing trace usability for the primary diagnostic use case.

3. **High-cardinality `url.full` with pod IPs** — `ReverseProxy` spans carry the full upstream URL including the pod IP address (e.g., `http://10.244.0.6:3000/`). In production, pod churn creates unbounded unique values for this attribute, making span-level aggregation unreliable and increasing storage costs.

4. **Ephemeral port attributes** — `client.port` and `network.peer.port` appear on server spans. These are always unique per connection and add noise without diagnostic value.

5. **`level: panic` in access logs** — all access log records carry `"level":"panic"` in the JSON body regardless of HTTP status. This is a known serialization bug that would trigger false severity alerts in any monitoring system that uses log level as a signal.

6. **OTLP log export is experimental** — the feature gate `experimental.otlpLogs: true` is required, indicating this is not production-stable.

The project shows clear OTel adoption intent and the metric layer is genuinely operator-oriented, but the trace naming and log quality issues mean an operator cannot rely on the telemetry defaults without customization or domain knowledge.
