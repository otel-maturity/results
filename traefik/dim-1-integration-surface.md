### 1. Integration Surface

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

- **Signals flowing via OTLP**: Traces ✅ (OTLP gRPC, scope `github.com/traefik/traefik`), Metrics ✅ (OTLP gRPC push, scope `github.com/traefik/traefik`; also via Prometheus scrape), Logs ✅ (OTLP gRPC, scope `traefik` — access logs confirmed flowing)
- **Configuration method**: Traefik-specific flags (`--tracing.otlp.grpc.endpoint`, `--metrics.otlp.grpc.endpoint`, `--accesslog.otlp.grpc.endpoint`, `--experimental.otlpLogs=true`). No `OTEL_EXPORTER_OTLP_ENDPOINT` or standard OTel SDK env vars respected for configuration.
- **Documentation stance**: OTLP is documented alongside Prometheus (metrics) and legacy Zipkin/Jaeger exporters (traces). Multiple backends are presented as co-equal options. The v3 docs dedicate distinct sections to each exporter type.
- **Legacy exporter status**: Co-equal — Prometheus is still the default/primary metrics path (enabled by default in the Helm chart); Zipkin and Jaeger trace exporters remain documented alongside OTLP. There is no deprecation notice on legacy exporters.
- **Signals requiring adapters/sidecars**: None — all three signals flow natively from Traefik to the OTel Collector via OTLP gRPC without any adapter or sidecar.

**Observed telemetry (confirmed from JSONL files):**

| Signal | Transport | Scope | Lines/Batches |
|--------|-----------|-------|---------------|
| Traces | OTLP gRPC | `github.com/traefik/traefik` | 1 batch, 31 Traefik spans (`GET`, `ReverseProxy`) |
| Metrics (OTLP push) | OTLP gRPC | `github.com/traefik/traefik` | 17 metric batches; 17 metric names including `http.server.request.duration`, `http.client.request.duration` (OTel semantic conventions) + 15 `traefik_*` names |
| Metrics (Prometheus) | Prometheus scrape | `prometheusreceiver` | Same 15 `traefik_*` metrics + Go runtime metrics |
| Logs (access) | OTLP gRPC | `traefik` | 5 batches, 30+ log records with `trace_id`/`span_id` correlation |

**Configuration required (from process args observed in telemetry):**

```
--tracing.otlp=true
--tracing.otlp.grpc=true
--tracing.otlp.grpc.endpoint=<collector>:4317
--tracing.otlp.grpc.insecure=true
--metrics.otlp=true
--metrics.otlp.grpc=true
--metrics.otlp.grpc.endpoint=<collector>:4317
--metrics.otlp.grpc.insecure=true
--metrics.prometheus=true                          # also still enabled simultaneously
--accesslog.otlp=true
--accesslog.otlp.grpc=true
--accesslog.otlp.grpc.endpoint=<collector>:4317
--experimental.otlpLogs=true                       # feature gate required for log export
```

**Key observations:**

1. **OTLP is supported but not default**: All three signals require explicit opt-in via Traefik-specific flags. Prometheus metrics are enabled by default in the Helm chart; OTLP metrics must be explicitly added.

2. **Dual metrics paths run simultaneously**: Both Prometheus scrape and OTLP push are active at the same time, producing overlapping metric data. The OTLP push adds OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`) that are not available via Prometheus scrape.

3. **Log export behind experimental feature gate**: OTLP log export requires `--experimental.otlpLogs=true` (or `experimental.otlpLogs: true` in Helm values). This is explicitly marked as non-stable, meaning it is not part of the stable integration surface.

4. **No standard OTel env var support**: Traefik does not respect `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, or `OTEL_SERVICE_NAME`. Each signal requires separate, Traefik-specific endpoint configuration flags. This is a significant departure from OTel SDK conventions.

5. **Per-signal configuration inconsistency**: Traces, metrics, and logs each have their own separate OTLP endpoint configuration blocks (`tracing.otlp`, `metrics.otlp`, `accesslog.otlp`). There is no unified OTLP exporter configuration.

6. **Prometheus still co-equal**: The Traefik Helm chart enables Prometheus metrics by default (`metrics.prometheus.entryPoint: metrics`). The OTLP metrics path is additive, not a replacement. Users must explicitly decide to use OTLP and may end up with both paths active.

7. **Rich OTel SDK integration**: Despite the configuration friction, Traefik v3 uses the OTel Go SDK natively (`telemetry.sdk.name: opentelemetry`, `telemetry.sdk.version: 1.43.0`). Resource attributes (`service.name`, `service.version`, `host.name`, `os.type`, `process.*`, `process.runtime.*`) are rich and OTel-compliant.

8. **W3C Trace Context propagation confirmed**: Traefik correctly reads incoming `traceparent` headers, preserves the trace ID, and creates a child span, forwarding the updated context to backends. This is a strong OTel integration point.

9. **Log body quirk**: Access log records embed the full JSON as a string in the OTLP log body, rather than as a structured object. The `level: panic` field appears in all access log records regardless of HTTP status — a known Traefik serialization bug in v3.

#### Checklist assessment

**Level 0 — Instrumented**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is telemetry exported only via tool-specific or legacy exporters (Jaeger only, Prometheus scrape only)? | NO | OTLP is fully supported and working for all three signals |
| Is OTLP unsupported or available only indirectly via sidecars/adapters? | NO | Traefik exports directly to OTLP gRPC; no adapter needed |
| Does telemetry configuration rely entirely on project-specific flags? | PARTIAL | Yes — Traefik uses its own flags, not standard OTEL_* env vars; but OTLP is the target |
| Do users need to adapt their observability stack to fit the project's model? | NO | Standard OTel Collector receives data natively |
| Is OpenTelemetry absent from docs or treated as an afterthought? | NO | OTel is prominently documented in Traefik v3 observability docs |

**Level 1 — OpenTelemetry-Aligned**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP supported alongside equally-promoted legacy exporters (documented side-by-side)? | YES | Prometheus, Zipkin, Jaeger, and OTLP are all documented as peer options in Traefik docs |
| Are there multiple overlapping ways to configure telemetry (project flags AND OTEL_* variables)? | YES | Only Traefik-specific flags work; OTEL_* env vars are not respected. Both Prometheus and OTLP metrics run simultaneously |
| Does OTLP require disabling legacy behavior or enabling experimental flags? | YES | Log export requires `experimental.otlpLogs: true`; Prometheus metrics remain enabled by default even when OTLP is added |
| Is OpenTelemetry integration inconsistent across signals? | YES | Metrics: Prometheus (default) + OTLP (opt-in); Traces: OTLP only (no Prometheus equivalent); Logs: OTLP (experimental feature gate) |
| Do users need to read multiple docs pages to get a working OTLP integration? | YES | Separate docs sections for tracing, metrics, and access logs, each with their own OTLP configuration |

→ **Level 1 confirmed** — all five Level 1 characteristics are present.

**Level 2 — OpenTelemetry-Native**

| Question | Answer | Evidence |
|----------|--------|----------|
| Is OTLP the default or clearly-recommended export path in docs? | NO | Prometheus is the default for metrics; OTLP is opt-in and presented as one of several options |
| Are `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME` respected end-to-end? | NO | Traefik does not honor standard OTel SDK env vars; all config is via Traefik-specific flags |
| Can users connect to an existing OTel Collector without adapters or glue code? | YES | Direct OTLP gRPC push to collector works without adapters |
| Are legacy exporters clearly secondary, optional, or deprecated? | NO | Prometheus is the default; Zipkin/Jaeger are documented as co-equals |
| Is telemetry configuration consistent across all signals? | NO | Each signal has a separate OTLP configuration block; logs are behind a feature gate |

→ **Level 2 not met** — three of five Level 2 criteria fail.

#### Rationale

Traefik v3 is solidly at **Level 1 — OpenTelemetry-Aligned**. The project has made genuine and substantial investment in OTel: it uses the OTel Go SDK natively, supports OTLP for all three signals (traces, metrics, logs), implements W3C Trace Context propagation, emits OTel semantic convention metrics (`http.server.request.duration`, `http.client.request.duration`), and requires no adapters or sidecars for an OTel Collector integration.

However, several structural characteristics prevent it from reaching Level 2:

1. **No standard env var support**: The complete absence of `OTEL_EXPORTER_OTLP_ENDPOINT` and related env vars is a significant gap. Users cannot configure Traefik's OTLP export using the standard OTel SDK configuration model. Each signal requires separate, Traefik-specific endpoint flags.

2. **Prometheus remains the default**: The Helm chart enables Prometheus metrics by default. OTLP is additive, not a replacement. This means users who want OTLP get both paths simultaneously unless they explicitly disable Prometheus.

3. **Log export is experimental**: `experimental.otlpLogs: true` is required, marking this as a non-stable feature. This is a meaningful caveat for production use.

4. **Per-signal configuration fragmentation**: Three separate OTLP configuration blocks (`tracing.otlp`, `metrics.otlp`, `accesslog.otlp`) with no unified endpoint configuration means that changing the collector address requires updating three separate sections.

5. **Legacy exporters co-equal in docs**: Zipkin, Jaeger, and Prometheus are documented alongside OTLP without deprecation notices or a clear recommendation hierarchy.

The project is clearly moving toward deeper OTel integration (the v3 migration to native OTel SDK is a major step forward), but the integration surface still requires users to navigate Traefik-specific configuration conventions rather than the standard OTel SDK model.
