### 5. Multi-Signal Observability

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Signal availability
- Traces: **flowing** — OTLP gRPC push (`--tracing.otlp.grpc`) to OTel Collector; 242 spans across 27 JSONL batches; scope `github.com/traefik/traefik`
- Metrics: **flowing** — dual pipeline: OTLP gRPC push (`--metrics.otlp.grpc`) for OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`, `traefik_*`) **and** Prometheus scrape (`traefik-metrics.traefik.svc.cluster.local:9100`) for the same `traefik_*` series; 2,904 metric data points
- Logs: **flowing** — OTLP gRPC push via `--accesslog.otlp.grpc`; 24 access log records; scope `traefik`; application-level logs via `--experimental.otlpLogs` are configured but flagged as experimental

##### Cross-signal correlation
- Log records with traceId: **24 of 24 (100%)**
- Log records with spanId: **24 of 24 (100%)**
- Unique log traceIds that resolve to a span in traces: **20 of 20 (100%)**
- Shared attribute keys (traces ∩ metrics): `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`

##### Collection model per signal

| Signal | Mechanism | Scope / Instrumentation |
|--------|-----------|------------------------|
| Traces | OTLP gRPC push (native SDK) | `github.com/traefik/traefik` v3.7.1; `telemetry.sdk.language: go`, `telemetry.sdk.version: 1.43.0` |
| Metrics | OTLP gRPC push (native SDK) **+** Prometheus scrape via OTel Collector `prometheusreceiver` | Both paths produce `traefik_*` series; OTLP-only path additionally produces `http.server.request.duration` / `http.client.request.duration` (OTel semantic conventions) |
| Logs | OTLP gRPC push via `accesslog.otlp` (access logs only) | Scope `traefik`; `--experimental.otlpLogs=true` is present but covers application-level logs and remains gated behind an experimental flag |

Traefik sends all three signals directly to the OTel Collector via OTLP gRPC, with no intermediate adapter or sidecar for traces or access logs. The Prometheus scrape pipeline is a redundant/legacy path for metrics that creates a dual-pipeline situation.

##### Investigative workflow assessment

A user **can** follow the metric → trace → log path without manual bridging:

1. **Metric anomaly** — `traefik_router_requests_total` or `http.server.request.duration` spikes; attributes (`router`, `service`, `http.request.method`, `http.response.status_code`) narrow the scope.
2. **Pivot to traces** — same `http.request.method`, `http.response.status_code`, `server.address` attributes are present on spans, allowing a backend query to find the relevant trace window.
3. **Inspect logs** — every access log record carries OTLP-level `traceId` and `spanId` fields that are byte-for-byte identical to the trace IDs on the corresponding spans. The JSON body also redundantly embeds `TraceId`, `SpanId`, `trace_id`, and `span_id`. A single trace ID lookup returns both the span and its access log entry.

The workflow is seamless for **request-level investigation** (HTTP traffic). It is incomplete for **application-level investigation** (startup errors, configuration issues, routing decisions) because application logs are still behind the `--experimental.otlpLogs` flag and were not observed in the collected data.

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Is only one signal treated as first-class (typically metrics only)? | **No** | All three signals flow via OTLP |
| Are traces or logs absent, experimental, or completely undocumented? | **Partial** | Access logs are first-class; application logs via `--experimental.otlpLogs` remain experimental |
| Is there no shared context between any signals? | **No** | 100% of log records carry traceId/spanId |
| Do users need to manually correlate timestamps across unrelated tools? | **No** | Trace IDs directly link logs and spans |

Level 0 does **not** apply.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all present (all three JSONL files have data)? | **Yes** | 242 spans, 2,904 metric data points, 24 log records |
| Do some signals include correlation identifiers while others do not? | **No** | All log records carry traceId/spanId |
| Do logs lack `traceId`/`spanId` even when traces exist? | **No** | 100% of logs are correlated |
| Are signals produced by different pipelines or configurations (inconsistent)? | **Partial** | Metrics have a dual OTLP+Prometheus pipeline; application logs are experimental |
| Must users manually bridge signals (e.g. copy a trace ID into a log search)? | **No** | TraceId is embedded natively in OTLP log records |

Level 1 is met and substantially exceeded.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Are traces, metrics, AND logs all first-class (present and documented)? | **Mostly** | Traces and metrics are fully first-class; access logs are first-class; application logs remain experimental |
| Do log records automatically include `traceId` and `spanId`? | **Yes** | 24/24 (100%) log records carry both fields at the OTLP record level |
| Do metrics share attribute keys with traces for the same concepts? | **Yes** | 6 shared keys: `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme` |
| Can users pivot from metric → trace → log without manual correlation? | **Yes** (for request traffic) | Shared attributes enable metric→trace pivot; traceId in log records enables trace→log pivot |
| Do signals complement rather than duplicate each other? | **Mostly** | The dual Prometheus+OTLP metrics pipeline creates partial duplication of `traefik_*` series |

Level 2 is met.

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Are signal volume and cardinality managed intentionally across all signals? | **No** | Dual metrics pipeline (OTLP + Prometheus scrape) produces overlapping series with no documented rationale; no sampling strategy is visible |
| Are high-cardinality metrics avoided in favor of trace-driven analysis? | **No** | `traefik_router_requests_total` and `traefik_service_requests_total` are emitted alongside full-fidelity traces without explicit guidance on which to use for what |
| Are signals shaped for common investigative paths (documented workflows)? | **No** | No documented "debug a latency spike" or "investigate a 5xx error" workflow that spans all three signals |
| Is there guidance on when to use which signal? | **No** | Documentation covers each signal independently; no cross-signal decision guidance found |
| Is the balance between cost, clarity, and depth explicit? | **No** | Dual pipeline is additive, not intentionally balanced |

Level 3 is **not** met.

#### Rationale

Traefik v3.7.1 reaches **Level 2** because all three signals are present and correlated by design, not by coincidence. Traces and access logs share the same OTLP pipeline, and every access log record is stamped with the trace ID and span ID of the matching request span — a 100% correlation rate with zero manual bridging required. Metrics share six OTel semantic-convention attribute keys with traces, enabling a direct metric → trace pivot on `http.request.method`, `http.response.status_code`, and `server.address`.

The project falls short of Level 3 for three reasons:

1. **Dual metrics pipeline**: Traefik emits the same `traefik_*` metrics both via OTLP push and via a Prometheus scrape endpoint. This is additive duplication rather than intentional signal shaping, and there is no documented guidance on which path to prefer.

2. **Experimental application logs**: `--experimental.otlpLogs` is configured but gated behind an experimental flag. No application-level log records were collected in this evaluation run; only access logs (HTTP request/response records) flow as OTLP logs. This means application-level investigation (startup errors, configuration reloads, routing decisions) cannot follow the full metric → trace → log path.

3. **No cross-signal investigative workflow documentation**: Each signal is documented independently (tracing, metrics, logging as separate pages). There is no published guide describing how to move from a metric anomaly to a trace to a correlated log entry during incident investigation.
