### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming assessment

Traefik emits spans from its own instrumentation scope (`github.com/traefik/traefik`) as well as spans originating from the downstream Express.js backend (via `@opentelemetry/instrumentation-express` and `@opentelemetry/instrumentation-http`).

- **Good (logical/user-relevant)**:
  - `GET` — HTTP method-based span names on the server-side entry point (partially useful; follows OTel HTTP semantic conventions for server spans)
  - `GET /health`, `GET /` — backend HTTP spans from the Express instrumentation include path, which is more descriptive
  - `ReverseProxy` — describes the proxy forwarding operation at a functional level

- **Bad (internal/implementation detail)**:
  - `ReverseProxy` — while functionally descriptive, it exposes the internal component name rather than a logical operation (e.g., no route/service context; no `http.route` attribute)
  - `middleware - query`, `middleware - jsonParser`, `middleware - expressInit`, `middleware - <anonymous>` — these come from the Express backend's auto-instrumentation and are highly internal, exposing middleware pipeline internals to trace consumers
  - `request handler - /`, `request handler - /health` — internal Express handler naming
  - Traefik's own `GET` spans lack `http.route` — they carry `url.path` (e.g., `/`, `/ping`, `/metrics`) but no route template, making grouping across parameterized routes impossible

- **Overall**: **Mixed** — Traefik's own spans are partially OTel-aligned (HTTP method naming, correct OTel semantic convention attributes like `http.request.method`, `http.response.status_code`, `url.path`) but miss critical grouping attributes (`http.route`, router name, service name). The `ReverseProxy` span is an internal component name. No `http.route` is set on any Traefik-originated span, which is a significant gap for operators trying to aggregate by route.

**Critical missing context**: Traefik spans have no `router` or `service` attribute (despite these being present in metric labels and log fields). An operator cannot identify *which router or service* handled a request from trace data alone.

##### Log verbosity

| Severity (severityText) | Count |
|------------------------|-------|
| info                   | 24    |

- **Volume assessment**: Low — only 24 log records total, one per proxied request (access log style).
- **Quality**: The logs are structured JSON access logs, emitted once per request at completion. They contain rich operational context: `RouterName`, `ServiceName`, `ServiceAddr`, `DownstreamStatus`, `OriginDuration`, `Overhead`, `Duration`, `RetryAttempts`, and trace correlation (`TraceId`, `SpanId`).
- **Severity mismatch (critical bug)**: All 24 log records have `severityText: "info"` but the embedded JSON body contains `"level": "panic"`. This is a significant signal quality defect — an operator monitoring for `panic`-level events would see them classified as `info`, while an operator filtering on `severityText` for `panic` would find nothing. This mismatch makes automated alerting on log severity unreliable.
- **Non-OTel attribute naming**: Log attributes use PascalCase (`ClientAddr`, `DownstreamStatus`, `RouterName`) rather than OTel semantic conventions (`net.peer.addr`, `http.response.status_code`, etc.), creating a disconnect between log and trace/metric attribute naming.

##### Metric quality

| Metric Name | Type | Assessment |
|------------|------|------------|
| `traefik_entrypoint_requests_total` | Counter | SLO-relevant: request rate by entrypoint, method, code, protocol |
| `traefik_entrypoint_request_duration_seconds` | Histogram | SLO-relevant: latency by entrypoint |
| `traefik_entrypoint_requests_bytes_total` | Counter | Operational: throughput |
| `traefik_entrypoint_responses_bytes_total` | Counter | Operational: response throughput |
| `traefik_router_requests_total` | Counter | SLO-relevant: request rate by router, service, method, code |
| `traefik_router_request_duration_seconds` | Histogram | SLO-relevant: latency by router |
| `traefik_router_requests_bytes_total` | Counter | Operational |
| `traefik_router_responses_bytes_total` | Counter | Operational |
| `traefik_service_requests_total` | Counter | SLO-relevant: request rate by service |
| `traefik_service_request_duration_seconds` | Histogram | SLO-relevant: latency by service |
| `traefik_service_requests_bytes_total` | Counter | Operational |
| `traefik_service_responses_bytes_total` | Counter | Operational |
| `traefik_open_connections` | Gauge | Operational: connection pressure |
| `traefik_config_reloads_total` | Counter | Operational: config change rate |
| `traefik_config_last_reload_success` | Gauge | Health signal |
| `http.server.request.duration` | Histogram | OTel semantic convention — SLO-relevant |
| `http.client.request.duration` | Histogram | OTel semantic convention — SLO-relevant |
| `go_*`, `process_*`, `scrape_*` | Various | Runtime/infrastructure — not Traefik-specific |

- **SLO-relevant metrics**: Strong set — `traefik_router_requests_total`, `traefik_router_request_duration_seconds`, `traefik_service_request_duration_seconds`, `http.server.request.duration`, `http.client.request.duration` are all directly usable for RED (Rate, Error, Duration) dashboards.
- **Metric label design**: Traefik-specific metrics use labels `router`, `service`, `method`, `code`, `protocol` — operationally meaningful. However, these use non-OTel naming (e.g., `code` instead of `http.response.status_code`), creating inconsistency with the OTel-convention metrics (`http.server.request.duration`) which use standard attribute names.
- **High-cardinality concerns**: `url.full` appears in `ReverseProxy` spans (e.g., `http://10.244.0.6:3000/`) — includes internal pod IP addresses, which is a potential cardinality concern if many backends exist. The `server.address` in `http.server.request.duration` includes pod IPs and internal service DNS names, which could fragment metrics in larger deployments.
- **Dual metric systems**: Traefik emits both Prometheus-style `traefik_*` metrics (scraped via `/metrics`) and OTel-convention `http.*.request.duration` metrics (via OTLP), with inconsistent attribute naming between them. This is a usability burden for operators.

##### Default production usability

Partially usable out of the box, but with notable gaps:

- **Metrics are the strongest signal**: The `traefik_router_*` and `traefik_service_*` metrics with router/service labels give operators a clear picture of traffic patterns, latency, and errors per routing rule. This is directly actionable for SLO monitoring.
- **Traces lack routing context**: An operator receiving a `GET` span from Traefik cannot determine which router or service handled the request without cross-referencing with logs. The absence of `http.route`, `router`, or `service` attributes on trace spans is a significant operational gap. Grouping traces by route (a fundamental tracing use case) is not possible.
- **Log severity bug is a production hazard**: All access log entries are misclassified as `info` when the body says `panic`. Alert rules based on OTel log severity will not fire correctly.
- **Mixed OTel conventions**: Some spans use OTel stable HTTP conventions (`http.request.method`, `http.response.status_code`), some use deprecated ones (`http.method`, `http.status_code`, `http.flavor`), and logs use neither. An operator building dashboards or alerts across signals faces a fragmented attribute landscape.
- **No `http.route` on server spans**: Without route templating, high-traffic APIs would produce one series per unique URL path, making aggregation and alerting unreliable.

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names expose internal function/class/component names? | Partially | `ReverseProxy` is an internal component name; `middleware - expressInit`, `middleware - <anonymous>` are fully internal |
| Are logs emitted for every internal step by default (high volume of DEBUG/TRACE)? | No | Logs are access-log style: one record per request completion |
| Is there no distinction between debug and operational signals? | No | Metrics and logs are clearly operational; there is some debug noise in Express middleware spans |
| Do users need to heavily filter telemetry before it becomes useful? | Partially | Metrics are usable directly; traces require filtering out middleware spans and health check noise |
| Are high-cardinality attributes emitted indiscriminately? | Partially | `url.full` includes pod IPs; `server.address` includes internal addresses; no raw query strings |

**Level 0 is not the right assignment** — telemetry is not purely maintainer-centric; there is clear operational intent.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is obvious noise reduced but defaults remain conservative? | Yes | No DEBUG/TRACE flood; but health check spans (`/ping`) and internal middleware spans are included without filtering |
| Are some spans renamed to logical operations but others remain internal? | Yes | `GET` follows HTTP conventions; `ReverseProxy` is partially logical; middleware spans are internal |
| Do operators need domain knowledge to interpret span names? | Yes | `ReverseProxy` requires knowing Traefik internals; no router/service context on spans requires Traefik knowledge |
| Is signal quality inconsistent across traces, metrics, and logs? | Yes | Metrics are strong; traces lack routing context; logs have severity mismatch and non-OTel attribute names |
| Are logs structured but still overly detailed for operational use? | Partially | 30+ fields per log record; most are operationally relevant but non-OTel named; severity bug is a defect |

**Level 1 is the correct assignment.**

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names describe logical, user-relevant operations? | Partially | `GET` is conventional but incomplete without `http.route`; `ReverseProxy` is internal |
| Are telemetry defaults usable in production without customization? | No | Missing `http.route` on traces; severity mismatch in logs; dual metric naming systems |
| Are logs emitted on state changes or errors — not on every internal step? | Yes | Access-log style, one per request |
| Are metrics focused on SLO-relevant signals rather than raw internal counters? | Mostly | `traefik_router_*` and `traefik_service_*` metrics are SLO-oriented; runtime `go_*` metrics add noise |
| Can operators move from symptoms to causes without deep internal knowledge? | No | Traces lack router/service context; log severity mismatch breaks alerting; dual metric systems require expertise |

**Level 2 is not met** — the missing `http.route` on spans and the log severity mismatch are blocking defects for production usability without customization.

#### Rationale

Traefik scores **Level 1 (OpenTelemetry-Aligned)**. The project has made genuine effort toward OTel alignment — it uses OTel semantic convention attribute names on traces (e.g., `http.request.method`, `network.protocol.version`, `url.path`), emits both OTel-convention metrics (`http.server.request.duration`) and rich Traefik-specific operational metrics (`traefik_router_*`, `traefik_service_*`), and produces structured access logs with trace correlation. These are meaningful improvements over a purely maintainer-centric baseline.

However, several gaps prevent Level 2:

1. **No `http.route` on server spans** — Traefik's own `GET` spans carry `url.path` but not `http.route`. This makes trace-based aggregation by route template impossible, which is a fundamental requirement for operator usability.
2. **No router/service context on traces** — Despite metrics and logs including `router` and `service` identifiers, Traefik spans carry only `entry_point`. An operator cannot correlate a slow trace to a specific Traefik router or backend service without external knowledge.
3. **Log severity mismatch** — All access log records are emitted with `severityText: "info"` while the body JSON contains `"level": "panic"`. This is a production-impacting defect that breaks severity-based alerting.
4. **Dual metric naming systems** — Prometheus-style `traefik_*` metrics and OTel-convention `http.*.request.duration` metrics coexist with different attribute naming, requiring operators to understand both systems.
5. **Mixed OTel convention versions** — Spans mix stable (`http.request.method`) and deprecated (`http.method`, `http.flavor`, `http.status_code`) attribute names, indicating incomplete migration.

The metrics layer is the strongest signal — a knowledgeable operator can build effective dashboards and alerts from `traefik_router_*` and `traefik_service_*` metrics. But the trace and log layers require domain knowledge and workarounds to use effectively in production.
