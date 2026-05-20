### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability
- Traces: **flowing** — OTLP push via Traefik's native OTel SDK (`github.com/traefik/traefik` scope); 204 spans across 9 JSONL lines
- Metrics: **flowing** — dual pipeline: Traefik-native OTLP push (`github.com/traefik/traefik` scope) for OTel-semantic metrics (`http.server.request.duration`, `http.client.request.duration`) plus legacy Prometheus-style metrics (`traefik_*`); 742 metric series across 9 JSONL lines
- Logs: **flowing** — OTLP push via Traefik's native log exporter (`traefik` scope, with `observedTimeUnixNano` confirming native OTLP); 24 log records across 3 JSONL lines

##### Cross-signal correlation
- Log records with traceId: **24 of 24 (100%)**
- Log records with spanId: **24 of 24 (100%)**
- Shared attribute keys (traces ∩ metrics): `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`
- Verified span-level linkage: log `spanId=ec661b5455bf83d6` matches the `GET` entry-point span in trace `9e74d9394fa15869a9fa04001b2b0c13`, confirming logs are emitted at the exact span boundary

##### Collection model per signal
- **Traces**: Traefik pushes spans natively over OTLP to the OTel Collector. Scope name `github.com/traefik/traefik` (no version pinned in scope metadata). Spans cover entry-point, router, middleware, and reverse-proxy hops.
- **Metrics**: Two sub-pipelines coexist. (1) Native OTel metrics (`http.server.request.duration`, `http.client.request.duration`) pushed via OTLP using OTel semantic conventions. (2) Legacy Prometheus-style metrics (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`) scraped by the OTel Collector `prometheusreceiver`. The OTel metrics share attribute keys with traces; the Prometheus-style metrics use non-standard keys (`entrypoint`, `router`, `service`, `code`, `method`, `protocol`).
- **Logs**: Traefik emits structured access-log records natively over OTLP. Each log record carries `traceId` and `spanId` as first-class OTLP fields. The log body is a JSON string that also redundantly embeds `TraceId`, `SpanId`, `trace_id`, and `span_id` fields. Log attributes include Traefik-specific fields (`RouterName`, `ServiceName`, `entryPointName`, `RequestPath`, etc.).

##### Investigative workflow assessment
A user can follow the **metric → trace → log** path without manual bridging:

1. **Metric anomaly** (e.g. spike in `http.server.request.duration` for `server.address=localhost:18080`) → filter by shared attributes (`http.request.method`, `http.response.status_code`, `server.address`) to narrow candidate traces.
2. **Pivot to trace** — the shared attribute keys (`http.request.method`, `http.response.status_code`, `server.address`, `server.port`, `url.scheme`) are identical between OTel metrics and trace spans, enabling direct drill-down into the specific trace.
3. **Pivot to log** — every log record carries the exact `traceId` and `spanId` of the Traefik entry-point span, so a trace viewer can surface the correlated access log in one click.

The only friction point is the **dual metric pipeline**: the legacy `traefik_*` Prometheus metrics use different attribute keys (`entrypoint`, `code`, `method`) than the OTel-semantic metrics and traces, so users relying on those legacy metrics must mentally re-map attribute names. This does not break correlation but reduces fluency.

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is only one signal treated as first-class (typically metrics only)? | **No** | All three signals are present and non-trivial |
| Are traces or logs absent, experimental, or completely undocumented? | **No** | Traces: 204 spans; Logs: 24 records, both via native OTLP |
| Is there no shared context between any signals? | **No** | Logs carry traceId/spanId; metrics share attribute keys with traces |
| Do users need to manually correlate timestamps across unrelated tools? | **No** | Shared trace context enables automated correlation |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all present? | **Yes** | All three JSONL files have data |
| Do some signals include correlation identifiers while others do not? | **No** | All three carry or share trace context |
| Do logs lack `traceId`/`spanId` even when traces exist? | **No** | 100% of log records carry both `traceId` and `spanId` |
| Are signals produced by different pipelines or configurations (inconsistent)? | **Partially** | Legacy `traefik_*` metrics use a Prometheus scrape pipeline with different attribute keys, but OTel metrics and traces share keys |
| Must users manually bridge signals? | **No** | Shared traceId/spanId enables direct pivot |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all first-class (present and documented)? | **Yes** | All three signals present with native OTLP export |
| Do log records automatically include `traceId` and `spanId`? | **Yes** | 24/24 (100%) log records carry both fields |
| Do metrics share attribute keys with traces for the same concepts? | **Yes** | 6 shared keys: `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme` |
| Can users pivot from metric → trace → log without manual correlation? | **Yes** | Shared attributes enable metric→trace pivot; shared traceId/spanId enables trace→log pivot |
| Do signals complement rather than duplicate each other? | **Mostly** | Logs add access-log detail (client IP, latency breakdown, Kubernetes context) not in traces; metrics aggregate what traces detail. Minor redundancy: traceId appears in both OTLP log fields and the log body JSON. |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume and cardinality managed intentionally across all signals? | **No** | Dual metric pipeline (OTel + legacy Prometheus) creates cardinality duplication for the same concepts |
| Are high-cardinality metrics avoided in favor of trace-driven analysis? | **No** | Legacy `traefik_*` metrics include per-router and per-service dimensions; no documented guidance to use traces for high-cardinality analysis |
| Are signals shaped for common investigative paths (documented workflows)? | **No** | No documented multi-signal debugging workflow found |
| Is there guidance on when to use which signal? | **No** | Observability documentation covers configuration of each signal independently, not cross-signal usage patterns |
| Is the balance between cost, clarity, and depth explicit? | **No** | No explicit cost/cardinality guidance |

#### Rationale

Traefik reaches **Level 2 (OpenTelemetry-Native)** because all three signals are present as native OTLP outputs, and they are intentionally correlated: every access-log record carries the `traceId` and `spanId` of its corresponding entry-point span (verified by matching span IDs across signals), and the OTel-semantic metrics share six attribute keys with trace spans enabling direct metric-to-trace pivots.

The project does not reach Level 3 because: (a) the dual metric pipeline (OTel-semantic + legacy Prometheus-style) creates attribute-key inconsistency that degrades fluency for users of the legacy metrics; (b) there is no documented investigative workflow guiding users on how to move across signals; and (c) cardinality and signal-selection guidance is absent.
