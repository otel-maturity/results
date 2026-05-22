### 6. Audience & Signal Quality

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Span naming assessment

Traefik emits spans from its own instrumentation scope (`github.com/traefik/traefik`) plus spans from the downstream backend (Node.js/Express via `@opentelemetry/instrumentation-http` and `@opentelemetry/instrumentation-express`).

**Traefik-native spans (scope: `github.com/traefik/traefik`):**
- **`GET`** (35 occurrences) — Used as the entry-point span name. This is the HTTP method alone with no route or path, making it ambiguous and operator-unfriendly. An operator cannot distinguish `/ping` health-check traffic from `/` application traffic at the span-name level alone.
- **`ReverseProxy`** (24 occurrences) — Internal component name. Describes Traefik's reverse-proxy mechanism rather than a logical operation. A better name would be something like `HTTP GET <route>` or `forward request`.

**Backend spans (scope: `@opentelemetry/instrumentation-http`, `@opentelemetry/instrumentation-express`):**
- **`GET /`**, **`GET /health`** — Good: logical HTTP method + route.
- **`middleware - query`**, **`middleware - expressInit`**, **`middleware - jsonParser`**, **`middleware - <anonymous>`** — Internal Express middleware names, not Traefik-emitted but present in the trace context.
- **`request handler - /`**, **`request handler - /health`** — Neutral: describes the handler role with route.

**Overall:** Traefik's own spans are mostly internal/ambiguous (`GET` without route context, `ReverseProxy`). The span name `GET` without a route template is a known OTel anti-pattern — it collapses all requests onto a single span name, making trace-based analysis difficult. The `ReverseProxy` span exposes an implementation component name. Some logical naming is present in the downstream backend spans but those are not Traefik's instrumentation.

- **Good (logical/user-relevant)**: `GET /` (backend), `GET /health` (backend)
- **Bad (internal/implementation)**: `ReverseProxy`, `GET` (bare method, no route), `middleware - expressInit`, `middleware - <anonymous>`
- **Overall**: Mixed — Traefik's own spans lean internal; backend spans are more logical.

---

##### Log verbosity

**Log severity distribution:**

| Severity (OTel `severityText`) | Count |
|-------------------------------|-------|
| `info`                        | 24    |

**Critical anomaly — severity mismatch:** All 24 log records carry `severityText: "info"` in the OTel envelope, but the embedded structured JSON body contains `"level": "panic"`. This is a significant signal quality defect: the OTel severity field is incorrect (or not mapped from the Traefik log level), which means any log pipeline that routes or alerts on OTel severity will silently misclassify these records. An operator relying on OTel-native log severity for alerting would miss what Traefik internally labels as `panic`-level events.

**Log content:** Each log record is a rich structured JSON access log containing:
- Client address/port, downstream/origin status codes, content sizes
- Duration, overhead, retry attempts
- Kubernetes Ingress name, namespace, service name, service address, service URL
- Router name, entry point name, request method/path/host/protocol
- Correlated `TraceId` and `SpanId` (good: trace correlation present)

**Volume assessment:** Low volume (24 records for ~24 requests) — one log record per request, consistent with access-log style. Not step-by-step internal debug logging.

**Quality:** Access-log style (one record per request/state change), which is operationally appropriate. However, the severity mismatch (`info` envelope vs `panic` body level) undermines usability for automated alert routing and log-level filtering.

---

##### Metric quality

**Traefik-native metrics (SLO-relevant):**

| Metric Name | Assessment |
|-------------|-----------|
| `traefik_entrypoint_requests_total` | ✅ SLO-relevant: request rate by entrypoint, status code, method, protocol |
| `traefik_entrypoint_request_duration_seconds` | ✅ SLO-relevant: latency histogram at entrypoint level |
| `traefik_entrypoint_requests_bytes_total` | ✅ Useful: throughput tracking |
| `traefik_entrypoint_responses_bytes_total` | ✅ Useful: response throughput |
| `traefik_router_requests_total` | ✅ SLO-relevant: request rate by router+service, status code |
| `traefik_router_request_duration_seconds` | ✅ SLO-relevant: latency by router |
| `traefik_router_requests_bytes_total` | ✅ Useful |
| `traefik_router_responses_bytes_total` | ✅ Useful |
| `traefik_service_requests_total` | ✅ SLO-relevant: downstream service request rate |
| `traefik_service_request_duration_seconds` | ✅ SLO-relevant: downstream service latency |
| `traefik_service_requests_bytes_total` | ✅ Useful |
| `traefik_service_responses_bytes_total` | ✅ Useful |
| `traefik_open_connections` | ✅ Operational: current connection count by entrypoint/protocol |
| `traefik_config_reloads_total` | ✅ Operational: config reload events |
| `traefik_config_last_reload_success` | ✅ Operational: config reload health |
| `http.server.request.duration` | ✅ OTel semantic convention histogram |
| `http.client.request.duration` | ✅ OTel semantic convention histogram |

**Runtime/infra metrics (lower operator value, not Traefik-specific):**
- `go_*` (29 metrics): Go runtime internals — useful for capacity planning but not SLO-relevant
- `process_*` (8 metrics): OS process stats
- `k8s.*` (11 metrics): Kubernetes cluster state — from OTel Collector, not Traefik itself
- `scrape_*` (4 metrics): Prometheus scrape metadata

**Metric cardinality assessment:**
- `traefik_router_requests_total` attributes: `{code, method, protocol, router, service}` — reasonable cardinality, router and service names are stable identifiers
- `traefik_entrypoint_requests_total` attributes: `{code, entrypoint, method, protocol}` — low cardinality, well-bounded
- `http.server.request.duration` attributes include `server.address` with raw IP:port values (`localhost:18080`, `10.244.0.7:8080`) — potential cardinality concern if many backend IPs are served
- `network.peer.port` in span attributes: 73 unique ephemeral port values observed — high cardinality in spans but not in metrics

**SLO-relevant metrics:** Multiple identified — request rate, latency histograms, error rates (via status code), and connection counts at entrypoint, router, and service levels.

**High-cardinality concerns:**
- `url.full` in `ReverseProxy` spans contains raw backend IP addresses (`http://10.244.0.6:3000/`) — these are pod IPs, which change on restart and could introduce cardinality issues in metric derivation
- `network.peer.port` carries ephemeral client ports (73 unique values) — not in metrics but would be problematic if ever promoted
- `server.address` in OTel histogram metrics contains raw IPs in some cases

---

##### Default production usability

**Partial usability out of the box.** An operator can answer basic questions:
- "How many requests per second is Traefik handling?" → Yes, via `traefik_entrypoint_requests_total`
- "What is the P99 latency for a specific router?" → Yes, via `traefik_router_request_duration_seconds`
- "Are there upstream service errors?" → Yes, via `traefik_service_requests_total` with status codes
- "Is Traefik's config up to date?" → Yes, via `traefik_config_last_reload_success`

However, operators face friction:
1. **Span names are not self-describing**: `GET` (bare method) and `ReverseProxy` require knowledge of Traefik's architecture to interpret correctly in distributed traces
2. **Log severity mismatch**: `severityText: "info"` for what Traefik calls `panic` means automated alerting on log severity will silently fail
3. **Mixed old/new OTel conventions**: Backend spans use deprecated `http.method`, `http.url`, `net.*` attributes (OTel semconv v1.x), while Traefik spans use current `http.request.method`, `url.*`, `network.*` — inconsistency within a single trace
4. **No documented operator dashboards or alert templates** found via public documentation search
5. **Large number of Go runtime metrics** (`go_*`, `process_*`) included by default add noise that requires filtering

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names expose internal function/class/component names? | **Partially** | `ReverseProxy` is a component name; `GET` (bare) is ambiguous |
| Are logs emitted for every internal step by default (high DEBUG/TRACE volume)? | **No** | Access-log style, one record per request |
| Is there no distinction between debug and operational signals? | **Partially** | Severity mismatch (`info` envelope vs `panic` body) blurs the distinction |
| Do users need to heavily filter telemetry before it becomes useful? | **Partially** | Metrics are largely useful; Go runtime metrics add noise; spans need route context |
| Are high-cardinality attributes emitted indiscriminately? | **Partially** | `url.full` with pod IPs; `network.peer.port` with ephemeral ports in spans |

**Level 0 verdict:** Does not fully apply — log volume is appropriate and metrics are operationally useful. But some Level 0 characteristics are present in traces and log severity handling.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Is obvious noise reduced but defaults remain conservative? | **Yes** | Access logs only (no step-by-step debug), but Go runtime metrics add bulk |
| Are some spans renamed to logical operations but others remain internal? | **Yes** | Entry-point spans use `GET` (partially logical), `ReverseProxy` is internal |
| Do operators need domain knowledge to interpret span names? | **Yes** | Must know what `ReverseProxy` means in Traefik's architecture; `GET` without route requires checking attributes |
| Is signal quality inconsistent across traces, metrics, and logs? | **Yes** | Metrics are strong; traces are weak (naming, mixed conventions); logs have severity bug |
| Are logs structured but still overly detailed for operational use? | **Partially** | Structured JSON access logs are appropriate in content, but severity mapping is broken |

**Level 1 verdict:** Substantially met. Traefik has made real effort toward OTel alignment (structured logs with trace correlation, OTel semantic conventions in metrics, histogram-based duration metrics, router/service/entrypoint metric segmentation) but signal quality is inconsistent — traces expose internal naming, and the log severity mismatch is a production-usability defect.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Do span names describe logical, user-relevant operations? | **No** | `GET` (bare) and `ReverseProxy` do not describe operations from a user perspective; missing route template in entry-point span names |
| Are telemetry defaults usable in production without customization? | **Partially** | Metrics yes; traces require operator knowledge; log severity is broken |
| Are logs emitted on state changes or errors — not every internal step? | **Yes** | Access-log pattern is appropriate |
| Are metrics focused on SLO-relevant signals? | **Yes** | Strong set of request rate, latency, and error metrics |
| Can operators move from symptoms to causes without deep internal knowledge? | **No** | Trace span naming requires Traefik architecture knowledge; `ReverseProxy` and bare `GET` spans are ambiguous |

**Level 2 verdict:** Not met. The span naming deficiency and log severity bug prevent this level from being awarded.

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume, cardinality, and cost managed intentionally? | **No** | No evidence of intentional cardinality management; pod IPs in `url.full` |
| Is telemetry quality evaluated using objective criteria? | **No** | No public evidence of instrumentation score checks or quality gates |
| Are high-cardinality attributes avoided in favor of trace-driven investigation? | **No** | Ephemeral ports and pod IPs present in span attributes |
| Are defaults refined based on user feedback? | **Unknown** | No public evidence found |
| Are quality regressions detectable and addressed proactively? | **No** | Log severity mismatch (`info` vs `panic`) suggests no regression detection |

**Level 3 verdict:** Not met.

---

#### Rationale

Traefik is awarded **Level 1 — OpenTelemetry-Aligned**.

The project has made genuine progress toward OTel alignment: it uses OTel semantic conventions for HTTP metrics (current semconv `http.request.method`, `network.*`, `url.*` in its own spans), emits well-structured access logs with trace correlation, and provides a rich set of SLO-relevant metrics segmented at entrypoint, router, and service levels with clear descriptions. The metric set (`traefik_router_requests_total`, `traefik_entrypoint_request_duration_seconds`, etc.) is genuinely useful for operators building SLOs.

However, three issues prevent advancement to Level 2:

1. **Span naming is not operator-friendly**: Traefik's entry-point spans are named `GET` (bare HTTP method) without a route template, making them indistinguishable across different endpoints at the span-name level. The `ReverseProxy` span exposes an internal component name rather than a logical operation. An operator cannot build meaningful trace-based dashboards or alerts without understanding Traefik's internal architecture.

2. **Log severity mismatch is a production defect**: All 24 log records have `severityText: "info"` in the OTel envelope but `"level": "panic"` in the structured body. This means any OTel-native log routing, filtering, or alerting based on severity will silently misclassify these records. This is not a minor inconsistency — it breaks a fundamental contract of the logging signal.

3. **Mixed OTel semantic convention generations**: Within a single distributed trace, Traefik's spans use current OTel semconv (`http.request.method`, `url.full`) while the backend spans (from a different instrumentation library) use deprecated conventions (`http.method`, `http.url`, `net.*`). While the backend is not Traefik's responsibility, the inconsistency within a single trace view reduces usability for operators.

The signal quality is inconsistent across the three telemetry pillars: metrics are strong (close to Level 2), logs are structurally appropriate but have a severity bug, and traces require domain knowledge to interpret. This profile — real effort and partial success, but with meaningful gaps in usability — is characteristic of Level 1.
