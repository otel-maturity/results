### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability
- Traces: **flowing** — OTLP gRPC push (`github.com/traefik/traefik` scope; `@opentelemetry/instrumentation-http` and `@opentelemetry/instrumentation-express` scopes from the backend service). 239 spans across 88 unique trace IDs.
- Metrics: **flowing** — dual pipeline: (1) OTLP gRPC push from Traefik native (`github.com/traefik/traefik` scope, emitting both OTel-semantic `http.server.request.duration` / `http.client.request.duration` and Prometheus-prefixed `traefik_*` metrics); (2) Prometheus scrape via OTel Collector `prometheusreceiver` (same `traefik_*` series plus Go runtime and process metrics). 2,742 metric series total.
- Logs: **flowing** — OTLP gRPC push of access logs via `--experimental.otlpLogs=true` + `--accesslog.otlp=true` (`traefik` scope). 72 log records across 4 JSONL lines.

##### Cross-signal correlation
- Log records with traceId: **24 of 24 (100%)**
- Log records with spanId: **24 of 24 (100%)**
- Matching trace IDs (logs ∩ traces): **20 of 20 unique log trace IDs found in trace data (100%)**
- Shared attribute keys (traces ∩ metrics):
  - `http.request.method`
  - `http.response.status_code`
  - `network.protocol.version`
  - `server.address`
  - `server.port`
  - `url.scheme`

##### Collection model per signal

| Signal | Export mechanism | Scope / Receiver |
|--------|-----------------|------------------|
| Traces | OTLP gRPC push → OTel Collector | `github.com/traefik/traefik` (Traefik native) |
| Metrics | OTLP gRPC push → OTel Collector **and** Prometheus scrape | `github.com/traefik/traefik` (OTLP) + `prometheusreceiver` (scrape) |
| Logs (access logs) | OTLP gRPC push → OTel Collector (experimental) | `traefik` scope |

All three signals are emitted by Traefik itself directly over OTLP gRPC to the same OTel Collector endpoint (`otel-collector-opentelemetry-collector.opentelemetry.svc.cluster.local:4317`). This is a single-pipeline, unified export model — not three separate tools.

The Prometheus scrape is an additional redundant path for metrics; the OTLP-native metrics (`http.server.request.duration`, `http.client.request.duration`) use OTel semantic conventions, while the Prometheus-scraped metrics use Traefik's legacy `traefik_*` naming.

OTLP logs are gated behind `--experimental.otlpLogs=true` (experimental flag in Traefik v3), but they are fully functional and actively flowing in this deployment.

##### Investigative workflow assessment

A user **can** follow the metric → trace → log path without manual bridging:

1. **Metric anomaly** (e.g., spike in `http.server.request.duration` or `traefik_router_requests_total` with `code=500`): filterable by `router`, `service`, `http.request.method`, `http.response.status_code`.
2. **Pivot to trace**: the same `http.request.method`, `http.response.status_code`, `server.address`, and `url.scheme` attributes appear on spans, allowing a backend (e.g., Grafana Tempo) to find correlated traces without manual ID copying.
3. **Pivot to log**: every access log record carries the OTLP-native `traceId` and `spanId` fields in the log envelope (verified: 100% correlation). The log body also embeds `TraceId`, `SpanId`, `trace_id`, and `span_id` as JSON fields, providing redundant linkage. A user can jump directly from a span to its corresponding access log entry.

The only gap preventing Level 3 is: (a) the dual-pipeline metric model (Prometheus scrape + OTLP push) creates redundant series with inconsistent naming conventions (`traefik_*` vs OTel semantic names), and (b) there is no documented investigative workflow guide that explicitly describes the metric → trace → log path, nor guidance on when to prefer traces over metrics for high-cardinality analysis.

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is only one signal treated as first-class (typically metrics only)? | **No** | All three signals are present and actively flowing via OTLP |
| Are traces or logs absent, experimental, or completely undocumented? | **No** | Traces are fully supported; logs are experimental but functional and documented |
| Is there no shared context between any signals? | **No** | 100% of log records carry traceId/spanId; 6 shared attribute keys between metrics and traces |
| Do users need to manually correlate timestamps across unrelated tools? | **No** | All signals share the same OTLP pipeline and resource attributes |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all present (all three JSONL files have data)? | **Yes** | 239 spans, 2742 metric series, 72 log records |
| Do some signals include correlation identifiers while others do not? | **No** | All three signals carry consistent resource attributes; logs carry traceId/spanId |
| Do logs lack `traceId`/`spanId` even when traces exist? | **No** | 24/24 (100%) of log records carry both traceId and spanId |
| Are signals produced by different pipelines or configurations (inconsistent)? | **Partially** | Metrics have a dual path (OTLP push + Prometheus scrape); traces and logs are OTLP-only |
| Must users manually bridge signals (e.g. copy a trace ID into a log search)? | **No** | traceId is embedded in the OTLP log envelope natively |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all first-class (present and documented)? | **Yes** | All three documented in Traefik's observability overview; unified config example shows `accessLog`, `metrics.otlp`, `tracing.otlp` together |
| Do log records automatically include `traceId` and `spanId`? | **Yes** | 100% of access log records carry both fields in the OTLP envelope; trace IDs verified to match actual spans |
| Do metrics share attribute keys with traces for the same concepts? | **Yes** | 6 shared keys: `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme` |
| Can users pivot from metric → trace → log without manual correlation? | **Yes** | Shared attributes enable metric→trace pivot; OTLP traceId in logs enables trace→log pivot |
| Do signals complement rather than duplicate each other? | **Mostly** | Metrics aggregate request counts/durations; traces provide per-request detail; logs provide full access log context — complementary. Minor duplication from dual Prometheus+OTLP metric paths |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume and cardinality managed intentionally across all signals? | **Partially** | Metrics have cardinality controls (`addRoutersLabels`, `addServicesLabels` flags); no evidence of trace sampling strategy beyond `sampleRate=1` (100%) |
| Are high-cardinality metrics avoided in favor of trace-driven analysis? | **No** | Both `traefik_*` Prometheus-style metrics (with `router`, `service` labels) and OTel-semantic metrics are emitted; no guidance on preferring traces for high-cardinality dimensions |
| Are signals shaped for common investigative paths (documented workflows)? | **No** | Traefik docs describe each signal independently; no "investigating a latency spike" or "debugging a 5xx error" cross-signal workflow guide exists |
| Is there guidance on when to use which signal? | **No** | Documentation describes what each signal does but not when to prefer one over another |
| Is the balance between cost, clarity, and depth explicit? | **No** | No documentation on cost trade-offs between Prometheus scrape vs OTLP metrics, or trace sampling vs log verbosity |

#### Rationale

Traefik v3 meets **Level 2 (OpenTelemetry-Native)** cleanly. All three signals are first-class and exported via a unified OTLP pipeline to the same collector. The correlation quality is excellent: 100% of access log records carry both `traceId` and `spanId` in the OTLP envelope (not just embedded in the log body), and all 20 unique trace IDs found in logs match verified spans in the trace data. Six attribute keys are shared between metrics and traces, enabling metric-to-trace pivots. The Traefik documentation presents all three signals together with a unified configuration example.

Level 3 is not reached because: (1) the metric pipeline is duplicated (Prometheus scrape + OTLP push), creating naming inconsistencies (`traefik_*` vs OTel semantic conventions) without documented rationale; (2) trace sampling is set to 100% with no evidence of intentional cost/cardinality management across signals; and (3) there is no cross-signal investigative workflow documentation — signals are documented in isolation rather than as a coordinated debugging experience.
