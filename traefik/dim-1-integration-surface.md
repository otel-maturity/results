### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅ (25 batches), Metrics ✅ (24 batches — both OTLP push and Prometheus scrape), Logs ✅ (4 batches — access logs and general logs via OTLP)
- **Configuration method**: Project-specific CLI flags (`--tracing.otlp.grpc.*`, `--metrics.otlp.*`, `--accesslog.otlp.*`, `--experimental.otlpLogs=true`) — `OTEL_*` standard env vars are **not** the primary configuration surface; only `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are supported as supplementary env vars
- **Documentation stance**: OTLP is the **only** documented tracing backend in v3.x. For metrics, OTLP is listed first and given equal documentation alongside Prometheus, Datadog, InfluxDB2, and StatsD. Logs OTLP export is documented but behind an experimental feature gate.
- **Legacy exporter status**: For tracing — **removed** (Jaeger/Zipkin direct exporters were removed in v3.0; only OTLP remains). For metrics — Prometheus is **co-equal** (enabled by default in the Helm chart). For logs — OTLP is **experimental** (requires `experimental.otlpLogs: true` flag).
- **Signals requiring adapters/sidecars**: None — all signals flow natively via OTLP gRPC to the OTel Collector. Prometheus metrics were also collected via Collector scrape (dual-path), but this is supplementary rather than required.

**Detailed observations:**

**Traces (OTLP — fully native):**
- Traefik v3 uses the OTel Go SDK directly (`go.opentelemetry.io/otel`) and emits spans via `otlptracehttp`/`otlptracegrpc` exporters natively.
- Instrumentation scope: `github.com/traefik/traefik` (no version in scope attribute — minor quirk).
- Span types observed: `GET` (server spans, kind=2) and `ReverseProxy` (client spans) — consistent with OTel HTTP semantic conventions v1.26.0 (explicitly cited in docs).
- W3C Trace Context propagation verified end-to-end: `traceparent` correctly propagated to backend with same trace ID and new child span ID.
- `OTEL_PROPAGATORS` env var is supported for selecting propagators (tracecontext, baggage, b3, jaeger, xray, ottrace).
- Resource attributes set natively: `service.name=traefik`, `service.version=3.7.0`, `telemetry.sdk.name=opentelemetry`, `telemetry.sdk.language=go`, `telemetry.sdk.version=1.43.0`, plus full process/OS attributes.
- In v3.7.0 docs, the tracing overview explicitly states: *"Traefik uses OpenTelemetry"* and refers only to the OTLP configuration page — no legacy backends remain.

**Metrics (dual-path — OTLP push + Prometheus scrape):**
- OTLP push is opt-in (`--metrics.otlp=true`); Prometheus is **enabled by default** in the Helm chart.
- Both were active in this evaluation; 17 OTLP-native metric names observed from the `github.com/traefik/traefik` scope, including OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) alongside Traefik-specific names (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`).
- The Prometheus endpoint is exposed on a dedicated port (9100) via a separate ClusterIP service — it is a first-class citizen, not deprecated.
- Docs list OpenTelemetry first but present Prometheus, Datadog, InfluxDB2, and StatsD as co-equal options.
- `OTEL_RESOURCE_ATTRIBUTES` env var is documented as a supplementary way to set resource attributes.
- Helm chart docs explicitly show how to **disable Prometheus** to use OTLP-only: `metrics: { prometheus: null, otlp: { enabled: true } }`.

**Logs (OTLP — experimental):**
- Both general logs and access logs can be exported via OTLP, but this requires `--experimental.otlpLogs=true` feature gate.
- The docs carry an explicit `!!! warning: This is an experimental feature.` notice.
- Access log OTLP export has a known quirk: the log body is a raw JSON string (not a structured object), and `level: panic` appears in the JSON body for normal 200 responses — a serialization bug.
- Log records include rich attributes: `TraceId`, `SpanId`, `trace_id`, `span_id` (duplicated in both camelCase and snake_case), `RouterName`, `ServiceName`, `KubernetesIngressName`, etc.
- The instrumentation scope for logs is `traefik` (no version) — inconsistent with the `github.com/traefik/traefik` scope used for traces and metrics.

**Configuration surface analysis:**
- Configuration is entirely via Traefik-native CLI flags / YAML/TOML config — **not** via `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, or `OTEL_SERVICE_NAME`.
- The deployment shows 40+ `--tracing.*`, `--metrics.*`, and `--accesslog.*` CLI arguments — there is no shortcut via standard OTel env vars.
- `OTEL_PROPAGATORS` (propagator selection) and `OTEL_RESOURCE_ATTRIBUTES` (additional resource attributes) are the only standard OTel env vars supported.
- No adapters, sidecars, or bridge components are needed — Traefik connects directly to the OTel Collector.

#### Checklist assessment

**Level 0 — Instrumented**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is telemetry exported only via tool-specific or legacy exporters (Jaeger only, Prometheus scrape only)? | **NO** | OTLP gRPC is the primary path for traces and logs; Prometheus is co-equal for metrics but OTLP push also works |
| Is OTLP unsupported or available only indirectly via sidecars/adapters? | **NO** | OTLP is natively supported for all three signals via the OTel Go SDK |
| Does telemetry configuration rely entirely on project-specific flags? | **PARTIAL** | Configuration uses project-specific flags, but they map directly to OTel SDK concepts (endpoint, protocol, headers) |
| Do users need to adapt their observability stack to fit the project's model? | **NO** | Any OTel Collector or OTLP-compatible backend works without adapters |
| Is OpenTelemetry absent from docs or treated as an afterthought? | **NO** | OpenTelemetry is the sole tracing backend in v3.x; prominently documented for all signals |

→ Does **not** qualify as Level 0.

**Level 1 — OpenTelemetry-Aligned**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP supported alongside equally-promoted legacy exporters (documented side-by-side)? | **PARTIAL** | For tracing: no legacy exporters remain (v3.0 removed Jaeger/Zipkin). For metrics: Prometheus, Datadog, InfluxDB2, StatsD are co-equal |
| Are there multiple overlapping ways to configure telemetry (project flags AND OTEL_* variables)? | **PARTIAL** | Only `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are supplementary; primary config is project-specific flags |
| Does OTLP require disabling legacy behavior or enabling experimental flags? | **PARTIAL** | Traces: no. Metrics: no (but Prometheus is default). Logs: yes — `experimental.otlpLogs=true` required |
| Is OpenTelemetry integration inconsistent across signals? | **PARTIAL** | Traces are fully stable/native; metrics has dual-path (OTLP + Prometheus, both active); logs OTLP is experimental |
| Do users need to read multiple docs pages to get a working OTLP integration? | **YES** | Separate docs pages for tracing, metrics, and logs OTLP configuration; no single "connect to OTel Collector" guide |

→ Qualifies as Level 1, but substantially exceeds it for traces.

**Level 2 — OpenTelemetry-Native**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP the default or clearly-recommended export path in docs? | **YES (traces), PARTIAL (metrics), NO (logs)** | Tracing docs show only OTLP; metrics docs list OTLP first but Prometheus is default; logs OTLP is experimental |
| Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` respected end-to-end? | **NO** | These standard env vars are not supported; only `OTEL_RESOURCE_ATTRIBUTES` and `OTEL_PROPAGATORS` are |
| Can users connect to an existing OTel Collector without adapters or glue code? | **YES** | Direct OTLP gRPC/HTTP to any OTel Collector; verified in evaluation |
| Are legacy exporters clearly secondary, optional, or deprecated? | **YES (traces), NO (metrics)** | Jaeger/Zipkin removed in v3. Prometheus is still the default metrics backend |
| Is telemetry configuration consistent across all signals? | **NO** | Traces: stable OTLP. Metrics: dual-path (Prometheus default + OTLP opt-in). Logs: experimental OTLP feature gate |

→ Mostly meets Level 2 for the integration surface, with notable gaps in standard env var support and log stability.

**Level 3 — OpenTelemetry-Optimized**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is the telemetry integration surface documented as a stable contract? | **NO** | Logs OTLP is explicitly experimental; no stability guarantees stated for telemetry configuration API |
| Are telemetry integration changes reviewed like API changes? | **NO** | No evidence of formal telemetry API governance; OTLP config changed between v2 and v3 with removals |
| Are breaking changes communicated with migration guidance? | **PARTIAL** | CHANGELOG notes breaking changes; migration guides exist for v2→v3 but not specific to telemetry |
| Does the project explicitly support diverse deployment models (local dev, Kubernetes, managed platforms)? | **YES** | Helm chart values, YAML/TOML, and CLI args documented; Kubernetes-specific resource attribute auto-detection |
| Are legacy integrations removed or tightly scoped with clear deprecation timelines? | **PARTIAL** | Jaeger/Zipkin removed in v3 (good). Prometheus remains co-equal with no deprecation timeline. Logs OTLP still experimental |

→ Does **not** qualify as Level 3.

#### Rationale

Traefik v3 is assigned **Level 2 — OpenTelemetry-Native** based on the following reasoning:

**Why not Level 1:** Traefik v3 removed all legacy tracing exporters (Jaeger, Zipkin) and made OTLP the sole tracing integration point. The tracing surface is unambiguously OTel-native — no adapters, no sidecars, no bridges. All three signals flow via OTLP gRPC to a standard OTel Collector without any glue code. This substantially exceeds Level 1's "OTLP supported alongside equally-promoted legacy exporters" characterization.

**Why Level 2:** The project meets the core Level 2 criteria: OTLP is the default/only path for traces, users can connect to any OTel Collector without adapters, the OTel Go SDK is used natively, and W3C Trace Context propagation works correctly end-to-end. The project is clearly OTel-oriented in its v3 design philosophy.

**Why not Level 3:** Three significant gaps prevent Level 3:
1. **No standard `OTEL_EXPORTER_OTLP_ENDPOINT` / `OTEL_SERVICE_NAME` support** — the primary configuration surface is Traefik-specific CLI flags, not the standard OTel SDK env vars. Users cannot configure Traefik's telemetry export with the same env vars they use for other OTel-instrumented services.
2. **Prometheus metrics is the default** — the Helm chart enables Prometheus by default, and OTLP metrics push is opt-in. For a fully OTel-native project, OTLP should be the default or at minimum co-equal without requiring explicit disablement of the legacy path.
3. **Logs OTLP is experimental** — requiring `experimental.otlpLogs=true` means the logs integration surface is not stable. The access log OTLP export also has a known serialization quirk (JSON body string, `level: panic` in normal 200 responses), indicating immaturity.

The combination of a fully native OTel tracing surface, functional OTLP metrics push (even if not default), and working OTLP log export (even if experimental) makes Level 2 the appropriate assignment. The project is clearly on a trajectory toward Level 3 as logs OTLP stabilizes and standard env var support potentially expands.
