### 1. Integration Surface

**Level: 2 — OpenTelemetry-Native**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅, Metrics ✅ (OTLP push + Prometheus scrape dual-channel), Logs ✅ (experimental feature gate)
- **Configuration method**: Project-specific Helm/YAML/CLI flags (`tracing.otlp.grpc`, `metrics.otlp`, `logs.access.otlp`); `OTEL_PROPAGATORS` env var is respected for propagator selection; `OTEL_EXPORTER_OTLP_ENDPOINT` is **not** documented as a configuration path — endpoint must be set via project-specific keys
- **Documentation stance**: OTLP is the **only** supported exporter in Traefik v3 for traces and metrics; legacy exporters (Jaeger, Zipkin, Datadog) were present in v2 and fully removed in v3 — the tracing overview page now reads "Traefik uses OpenTelemetry, an open standard designed for distributed tracing. Please check our dedicated OTel docs."
- **Legacy exporter status**: Removed — Jaeger/Zipkin/Datadog tracing backends existed in v2 but were dropped in v3; no legacy exporters remain for traces or metrics
- **Signals requiring adapters/sidecars**: None — all three signals flow directly from the Traefik process via OTLP gRPC to the Collector; however, Prometheus scrape is also enabled in parallel for metrics (dual-channel), and OTLP log export requires `experimental.otlpLogs: true` feature gate

**Observed telemetry details:**

| Signal | Batches | Instrumentation Scope | Transport |
|---|---|---|---|
| Traces | 27 | `github.com/traefik/traefik` (no version in scope) | OTLP gRPC |
| Metrics | 39 | `github.com/traefik/traefik v3.7.1` (OTLP push) + `prometheusreceiver v0.150.1` (scrape) | OTLP gRPC push + Prometheus pull |
| Logs | 4 | `traefik` (no version in scope) | OTLP gRPC |

**Specific observations:**

1. **Traefik v3 is OTLP-only for traces**: The v3 changelog confirms all legacy tracers were removed at the v3.0 boundary. The tracing overview now exclusively references OpenTelemetry. This is a clean break from v2.

2. **Metrics are dual-channel**: Traefik emits both OTLP push metrics (`github.com/traefik/traefik v3.7.1` scope, including OTel semantic convention metrics `http.server.request.duration` and `http.client.request.duration`) and Prometheus scrape metrics (`traefik_entrypoint_*`, `traefik_router_*`, `traefik_service_*`). The Prometheus endpoint is opt-in but equally documented alongside OTLP. Both channels were active in this evaluation. This is a notable split: OTLP metrics include semantic convention names, while Prometheus metrics use Traefik-native naming.

3. **OTLP log export is behind a feature gate**: `experimental.otlpLogs: true` must be set to enable OTLP log export. This is documented as experimental in the source code (`OTLPLogs bool` in `experimental.go`). The feature is functional and flowing but not yet stable. Log body is a JSON string (not a structured OTLP body), and `level: panic` appears in access log records for normal 200 responses — a known serialization quirk.

4. **Configuration is project-specific, not OTEL_* env vars**: Endpoint, protocol, and TLS configuration must be set via Traefik's own config hierarchy (`tracing.otlp.grpc.endpoint`, etc.). The standard `OTEL_EXPORTER_OTLP_ENDPOINT` env var is not documented as a supported configuration path. Only `OTEL_PROPAGATORS` is explicitly supported as a standard OTel env var.

5. **Rich resource attributes**: Traefik sets `service.name`, `service.version`, `telemetry.sdk.name`, `telemetry.sdk.language`, `telemetry.sdk.version`, `host.name`, `os.type`, `os.description`, `process.*` — all OTel semantic convention resource attributes populated correctly by the Go OTel SDK.

6. **W3C Trace Context propagation confirmed**: Incoming `traceparent` headers are correctly read, trace IDs preserved, new child span IDs generated, and updated `traceparent` forwarded to backends.

7. **OTel semantic convention compliance**: Traefik v3 docs state it follows "official OpenTelemetry semantic conventions v1.26.0". Span names use HTTP method names (`GET`), span kinds are SERVER/CLIENT, and metric names include both OTel semantic convention names and Traefik-native names.

#### Checklist assessment

**Level 0 — Instrumented**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is telemetry exported only via tool-specific or legacy exporters (Jaeger only, Prometheus scrape only)? | **NO** | OTLP is the primary and only trace exporter; metrics have OTLP push as the primary path |
| Is OTLP unsupported or available only indirectly via sidecars/adapters? | **NO** | All three signals flow natively via OTLP gRPC without any sidecar or adapter |
| Does telemetry configuration rely entirely on project-specific flags? | **PARTIAL** | Configuration uses project-specific YAML/CLI flags, but this is by design in Traefik's architecture |
| Do users need to adapt their observability stack to fit the project's model? | **NO** | Standard OTel Collector with OTLP receiver accepts all signals directly |
| Is OpenTelemetry absent from docs or treated as an afterthought? | **NO** | OTel is the only supported tracing/metrics integration in v3; prominently documented |

**Level 1 — OpenTelemetry-Aligned**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP supported alongside equally-promoted legacy exporters? | **NO** | Legacy exporters (Jaeger, Zipkin, Datadog) were fully removed in v3; OTLP is the sole option for traces |
| Are there multiple overlapping ways to configure telemetry? | **PARTIAL** | Metrics have dual-channel (OTLP + Prometheus), both documented equally; traces and logs are OTLP-only |
| Does OTLP require disabling legacy behavior or enabling experimental flags? | **PARTIAL** | Traces/metrics: no flags needed; logs require `experimental.otlpLogs: true` |
| Is OpenTelemetry integration inconsistent across signals? | **PARTIAL** | Traces/metrics are OTLP-native; logs are OTLP-capable but behind experimental flag |
| Do users need to read multiple docs pages to get a working OTLP integration? | **NO** | Each signal has a dedicated OTel doc page; configuration is straightforward |

**Level 2 — OpenTelemetry-Native**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP the default or clearly-recommended export path in docs? | **YES** | For traces: OTLP is the only option in v3 (defaults to HTTPS to localhost:4318). For metrics: OTLP is documented as the push path alongside Prometheus scrape |
| Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` respected end-to-end? | **PARTIAL** | `OTEL_PROPAGATORS` is explicitly supported; `OTEL_EXPORTER_OTLP_ENDPOINT` is not documented — endpoint must be set via project-specific config keys |
| Can users connect to an existing OTel Collector without adapters or glue code? | **YES** | Direct OTLP gRPC connection to Collector works for all three signals; confirmed in evaluation |
| Are legacy exporters clearly secondary, optional, or deprecated? | **YES** | Completely removed in v3; no legacy exporters exist for traces; Prometheus is optional alongside OTLP for metrics |
| Is telemetry configuration consistent across all signals? | **MOSTLY YES** | Traces and metrics: stable OTLP config. Logs: functional but experimental. All three signals use same OTLP gRPC endpoint pattern |

**Level 3 — OpenTelemetry-Optimized**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is the telemetry integration surface documented as a stable contract? | **NO** | OTLP log export is explicitly experimental; metric naming includes both OTel semantic convention and Traefik-native names without clear stability guarantees |
| Are telemetry integration changes reviewed like API changes? | **NO** | No evidence of telemetry-specific stability commitments or change review process in docs or changelog |
| Are breaking changes communicated with migration guidance? | **PARTIAL** | The v2→v3 migration removed all legacy tracers; migration guide exists but telemetry changes are not specially called out with migration paths |
| Does the project explicitly support diverse deployment models? | **PARTIAL** | HTTP and gRPC OTLP are both documented; TLS and insecure modes supported; no explicit guidance for local dev vs. Kubernetes vs. managed platforms |
| Are legacy integrations removed or tightly scoped with clear deprecation timelines? | **PARTIAL** | Legacy tracers were removed cleanly in v3; Prometheus metrics endpoint remains active without deprecation timeline |

#### Rationale

Traefik v3 is firmly at **Level 2 — OpenTelemetry-Native**. The most significant evidence is the complete removal of all legacy tracing backends (Jaeger, Zipkin, Datadog) in v3, making OTLP the sole trace export path. All three signals flow natively via OTLP gRPC to an OTel Collector without any adapter, sidecar, or bridge component. The project uses the OTel Go SDK with full semantic convention compliance for resource attributes, span semantics, and metric naming.

The project falls short of Level 3 for several reasons: (1) OTLP log export remains behind an `experimental.otlpLogs` feature gate and exhibits serialization quirks (JSON string body, incorrect severity level for access logs); (2) the standard `OTEL_EXPORTER_OTLP_ENDPOINT` env var is not documented as a supported configuration path — users must use Traefik-specific config keys; (3) metrics maintain a dual-channel model (OTLP push + Prometheus scrape) with no stated deprecation timeline for Prometheus; (4) there is no explicit stability contract or change-review process for the telemetry integration surface.

The partial use of standard OTel env vars (only `OTEL_PROPAGATORS` is documented) and the experimental status of log export are the primary gaps preventing a Level 3 assessment.
