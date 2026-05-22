### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming assessment

Traefik emits spans from its own instrumentation scope (`github.com/traefik/traefik`). The span names observed are:

- **Good (logical/user-relevant)**:
  - `GET` — HTTP method as span name is a partial description; combined with `url.path` and `entry_point` attributes it is interpretable, though the name alone is ambiguous
  - `GET /health`, `GET /`, `GET /nonexistent` — some spans from the backend test service carry method + path as the name (from `@opentelemetry/instrumentation-http`), which is the correct HTTP semantic convention pattern

- **Bad / Neutral (internal/implementation detail)**:
  - `ReverseProxy` — this is a Traefik internal component name, not a logical operation. It exposes the implementation detail that Traefik uses a reverse proxy to forward requests. An operator unfamiliar with Traefik internals would not immediately know what this span represents
  - `middleware - query`, `middleware - expressInit`, `middleware - jsonParser`, `middleware - <anonymous>` — these come from the downstream backend service (`@opentelemetry/instrumentation-express`) and expose Express.js middleware internals including anonymous functions, which are highly implementation-centric
  - `request handler - /` — from the backend's Express instrumentation; functional but exposes framework internals
  - Bare `GET` (without path) for Traefik's own entry-point spans — the span name alone carries no routing context; the meaningful information (path, entry point, status) lives only in attributes

- **Overall**: **Mixed** — Traefik's own spans use `GET` (bare method) and `ReverseProxy` (internal component name). The `GET` name is technically aligned with OTel HTTP semantic conventions (method as span name is correct for server spans), but `ReverseProxy` is an internal name. Critically, there is no router or service name in the span name or as a dedicated attribute on trace spans (router/service context is only available in metrics), making it harder to correlate traces to routing configuration.

##### Log verbosity

| Severity Text | Severity Number | Count |
|---------------|-----------------|-------|
| `info`        | 9               | 24    |

- **Volume assessment**: Low — 24 total log records across the capture window, all at INFO severity
- **Quality**: The log records are access-log style per-request JSON objects emitted for every proxied HTTP request. They are rich and operational (contain `RouterName`, `ServiceName`, `ServiceAddr`, `Duration`, `DownstreamStatus`, `OriginStatus`, `RetryAttempts`, etc.) — well-suited for debugging individual requests
- **Critical anomaly**: The OTel `severityText` field is mapped as `info` (severityNumber=9), but the JSON body embedded inside the log record contains `"level":"panic"`. This is a **severity mismatch bug** — the log body's internal `level` field (from Traefik's logrus-based logger) says `panic` while the OTel envelope says `info`. This would mislead any alerting or filtering system that relies on OTel severity levels
- **No DEBUG/TRACE noise**: Logs are not emitted for every internal step; they appear to be access-log entries per proxied request, which is appropriate for production use
- **Log body is a JSON string**: The log body is a JSON-encoded string rather than structured OTel attributes, which means log backends cannot natively filter or index individual fields (e.g., `RouterName`, `ServiceName`) without additional parsing

##### Metric quality

| Metric Name | Type | Assessment |
|---|---|---|
| `traefik_entrypoint_requests_total` | Sum (counter) | ✅ SLO-relevant — request rate by entry point, status code |
| `traefik_entrypoint_request_duration_seconds` | Histogram | ✅ SLO-relevant — latency by entry point |
| `traefik_entrypoint_requests_bytes_total` | Sum | ✅ Operational — throughput |
| `traefik_entrypoint_responses_bytes_total` | Sum | ✅ Operational — throughput |
| `traefik_router_requests_total` | Sum | ✅ SLO-relevant — request rate by router+service |
| `traefik_router_request_duration_seconds` | Histogram | ✅ SLO-relevant — latency by router+service |
| `traefik_router_requests_bytes_total` | Sum | ✅ Operational |
| `traefik_router_responses_bytes_total` | Sum | ✅ Operational |
| `traefik_service_requests_total` | Sum | ✅ SLO-relevant — request rate per backend service |
| `traefik_service_request_duration_seconds` | Histogram | ✅ SLO-relevant — latency per backend service |
| `traefik_service_requests_bytes_total` | Sum | ✅ Operational |
| `traefik_service_responses_bytes_total` | Sum | ✅ Operational |
| `traefik_open_connections` | Gauge | ✅ Operational — active connections per entry point |
| `traefik_config_reloads_total` | Sum | ✅ Operational — config change events |
| `traefik_config_last_reload_success` | Gauge | ✅ Operational — config health |
| `http.server.request.duration` | Histogram | ✅ OTel semantic convention metric |
| `http.client.request.duration` | Histogram | ✅ OTel semantic convention metric |
| `go_*` / `process_*` | Various | ⚠️ Runtime internals — useful for infra teams but not SLO-relevant |
| `scrape_*` | Various | ⚠️ Prometheus scrape metadata — not operational signal |
| `k8s.*` | Various | ℹ️ Infrastructure metrics from collector, not Traefik-native |

- **SLO-relevant metrics**: `traefik_*_requests_total`, `traefik_*_request_duration_seconds`, `traefik_open_connections`, `traefik_config_*`, `http.server.request.duration`, `http.client.request.duration`
- **High-cardinality concerns**: Metric labels are well-controlled. Traefik uses `code`, `entrypoint`, `method`, `protocol`, `router`, `service` as label dimensions — bounded cardinality. No raw URL paths or user IDs observed in metric labels. The `server.address` in `http.server.request.duration` includes port (e.g., `localhost:18080`, `10.244.0.10:8080`), which could be a minor cardinality concern in dynamic environments but is acceptable here

##### Default production usability

**Mixed — partially usable without customization, but with notable gaps:**

1. **Traces**: The Traefik-native spans (`GET` at entry point, `ReverseProxy`) are technically functional but have a usability gap: the span name `GET` is ambiguous without reading attributes, and `ReverseProxy` exposes an implementation name. Crucially, **router and service context is absent from trace spans** — an operator cannot tell from a trace which Traefik router matched the request (this information only appears in metrics and access logs). Connecting a slow trace to a specific router/service configuration requires cross-referencing with metrics or logs.

2. **Logs**: Access log records per request are production-useful (rich context including `RouterName`, `ServiceName`, `Duration`, `OriginStatus`), but the **severity mismatch** (`info` OTel envelope vs. `panic` in body) is a production-quality defect that would cause alert fatigue or missed alerts. The JSON-as-string body format also limits native field querying.

3. **Metrics**: The traefik-specific metrics are well-designed for operator use — three layers of granularity (entrypoint → router → service) with request count, latency histograms, and byte throughput at each layer. An operator can build RED dashboards from these without deep Traefik knowledge. This is the strongest signal quality area.

4. **No sampling/filtering guidance**: No evidence of built-in sampling configuration or noise-reduction defaults for the trace signal. Kubernetes liveness probe traffic (`/ping`) and metrics scrape traffic (`/metrics`) are included in traces by default, creating noise that operators would need to filter.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names expose internal function/class/component names? | **Partially** | `ReverseProxy` is an internal component name; bare `GET` is ambiguous |
| Are logs emitted for every internal step by default? | **No** | Only access-log entries per proxied request; no DEBUG/TRACE step logging |
| Is there no distinction between debug and operational signals? | **No** | Metrics are clearly operational; logs are access-log style |
| Do users need to heavily filter telemetry before it becomes useful? | **Partially** | Liveness probe and metrics scrape spans pollute traces; router/service context missing from traces |
| Are high-cardinality attributes emitted indiscriminately? | **No** | `url.full` in `ReverseProxy` spans contains full internal backend URL (e.g., `http://10.244.0.6:3000/`) which is an IP+port — manageable in practice but not templated |

> Level 0 is **not** the best fit — telemetry is more than just maintainer-centric noise.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is obvious noise reduced but defaults remain conservative? | **Yes** | No DEBUG flood, but liveness/scrape spans included by default; no sampling |
| Are some spans renamed to logical operations but others remain internal? | **Yes** | `GET` is a partially logical name; `ReverseProxy` is internal; no router/service context on spans |
| Do operators need domain knowledge to interpret span names? | **Yes** | `ReverseProxy` requires knowing Traefik's architecture; `entry_point` attribute requires Traefik config knowledge |
| Is signal quality inconsistent across traces, metrics, and logs? | **Yes** | Metrics are well-designed (SLO-ready); traces lack router/service context; logs have severity mismatch bug |
| Are logs structured but still overly detailed for operational use? | **Partially** | Logs are per-request access logs (appropriate volume), but JSON-as-string body limits usability; severity mismatch is a quality issue |

> Level 1 characteristics are **substantially met**.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names describe logical, user-relevant operations? | **Partially** | `GET` is semantically correct per OTel HTTP conventions but carries no routing context; `ReverseProxy` is not logical |
| Are telemetry defaults usable in production without customization? | **No** | Severity mismatch in logs; liveness probe noise in traces; no router/service on trace spans |
| Are logs emitted on state changes or errors — not on every internal step? | **Mostly** | Per-request access logs are appropriate, but every request is logged regardless of outcome |
| Are metrics focused on SLO-relevant signals? | **Yes** | Traefik metrics (requests_total, request_duration_seconds) are SLO-ready at all three layers |
| Can operators move from symptoms to causes without deep internal knowledge? | **Partially** | Metrics enable RED dashboards; but traces cannot be correlated to router/service config without domain knowledge |

> Level 2 is **not fully met** — the severity mismatch bug, missing router context in traces, and liveness probe noise prevent production-ready defaults.

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume, cardinality, and cost managed intentionally? | **No** | No built-in trace sampling; all requests (including health probes) traced by default |
| Is telemetry quality evaluated using objective criteria? | **No** | No evidence of instrumentation score checks or quality gates |
| Are high-cardinality attributes avoided in favor of trace-driven investigation? | **Partially** | Metric labels are well-controlled, but `url.full` in spans contains raw backend IPs |
| Are defaults refined based on user feedback? | **Unknown** | No public evidence of iterative signal quality improvements |
| Are quality regressions detectable and addressed proactively? | **No** | The severity mismatch bug (`info` envelope / `panic` body) indicates no automated quality validation |

> Level 3 is **not met**.

---

#### Rationale

Traefik is assessed at **Level 1 — OpenTelemetry-Aligned**.

The strongest evidence for this level is the **inconsistency across signal types**: Traefik's metrics are genuinely operator-focused and SLO-ready (three-tier granularity at entrypoint → router → service, with request counts and latency histograms), but the trace and log signals have notable usability gaps that prevent a clean Level 2 assignment.

**Key factors preventing Level 2:**

1. **`ReverseProxy` span name** is an internal Traefik component name, not a logical operation name. An operator unfamiliar with Traefik would not know what this span represents without reading documentation.

2. **Router and service context is absent from trace spans.** This is the most significant usability gap: Traefik routes requests through named routers and services (configured via Kubernetes Ingress annotations), but this routing context — which is the primary way operators understand traffic flow — is not present on trace spans. It appears only in metrics (`router` and `service` labels) and access logs (`RouterName`, `ServiceName`). This makes traces difficult to use for diagnosing routing-related issues without cross-referencing other signals.

3. **Severity mismatch in logs**: All 24 log records have OTel `severityText=info` but the JSON body contains `"level":"panic"`. This is a production-quality defect — alerting rules based on OTel severity would never fire on these records despite the body claiming panic-level events.

4. **Liveness probe and metrics scrape noise**: Kubernetes `/ping` health checks and Prometheus `/metrics` scrapes are included in traces by default. These are internal infrastructure operations that add trace volume without operational value for most operators.

The metrics signal alone would qualify for Level 2, but the overall telemetry package — considering all three signals together — is better characterized as Level 1: some effort toward operator usability, but signals are still shaped by internal implementation perspectives and have quality defects that require operator workarounds.
