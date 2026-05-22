### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure
- Total spans: 311
- Total root spans: 141 (69 from initial step-1 pass; 141 confirmed with slurp mode)
- Total child spans: 170
- Distinct trace IDs: 142
- Multi-span traces (user traffic): 20 (7–8 spans each, plus 1 with 40 spans)
- Single-span traces: 122 — **all are internal Traefik health checks (`/ping`) or Prometheus metrics scrapes (`/metrics`); no user-traffic fragmentation**
- Span kind distribution: INTERNAL=117, SERVER=170, CLIENT=24, PRODUCER=0, CONSUMER=0

Trace topology for a standard user request (8-span trace):
```
ROOT → GET [SERVER, github.com/traefik/traefik, entry_point=web]
         └─ ReverseProxy [CLIENT, github.com/traefik/traefik]
               └─ GET / [SERVER, @opentelemetry/instrumentation-http]
                     ├─ middleware - query      [INTERNAL, @opentelemetry/instrumentation-express]
                     ├─ middleware - expressInit [INTERNAL]
                     ├─ middleware - jsonParser  [INTERNAL]
                     ├─ middleware - <anonymous> [INTERNAL]
                     └─ request handler - /     [INTERNAL]
```

The 40-span trace demonstrates fan-out: 5 parallel `GET` (SERVER) spans all share parent `00f067aa0ba902b7`, which is **not present** in the dataset — confirming Traefik correctly extracted and continued an incoming external W3C `traceparent` header. Each of the 5 Traefik SERVER spans then created a CLIENT `ReverseProxy` span that propagated the context to a backend, producing a complete 40-span cross-service trace.

##### Context propagation
W3C Trace Context (`traceparent`) is supported and propagated correctly:

- **Inbound extraction**: `entrypoint.go` calls `tracing.ExtractCarrierIntoContext(req.Context(), req.Header)` before creating the root SERVER span, so an incoming `traceparent` header is honoured. Confirmed in telemetry: the 40-span trace has 5 Traefik SERVER spans all parented to an external span ID (`00f067aa0ba902b7`) not present in the dataset.
- **Outbound injection**: `tracing.go` calls `tracing.InjectContextIntoCarrier(req)` before forwarding to backends, so the `traceparent` header is set on every proxied request. Confirmed in telemetry: backend spans (`@opentelemetry/instrumentation-http`) appear as children of Traefik `ReverseProxy` CLIENT spans.
- **Propagator**: Traefik uses `autoprop.NewTextMapPropagator()` (from `go.opentelemetry.io/contrib/propagators/autoprop`), which defaults to W3C Trace Context + Baggage, with automatic support for additional propagators via environment variable `OTEL_PROPAGATORS`.
- **Parent-based sampling**: Documented — child spans inherit the sampling decision of the parent span regardless of the local `sampleRate`. This ensures complete end-to-end trace fidelity once a trace is sampled.
- No `traceparent`, `tracestate`, or `b3` attributes appear on spans (correct — these are transport headers, not span attributes).

##### Async/background work
No async/background work spans are observed in the telemetry. Traefik is a synchronous reverse proxy; there are no goroutine-based background job spans or detached traces. All proxied requests produce continuous parent-child chains from entry point to backend. The retry middleware (visible in integration tests) correctly keeps retry attempts within the same trace, each producing sibling `ReverseProxy` CLIENT spans under the same root.

##### Span links usage
No span links observed. Absent — not needed for Traefik's synchronous proxy model. The fan-out pattern (5 parallel requests in the 40-span trace) is correctly modelled as 5 child SERVER spans under the same parent, not via links.

##### Trace coherence assessment
A user can follow a complete logical operation through the trace without stitching fragments:

1. The **entry-point SERVER span** (`GET`, kind=2) represents the inbound HTTP request received by Traefik, with full HTTP semantic attributes (`url.path`, `http.response.status_code`, `client.address`, `server.address`, `user_agent.original`, `entry_point`).
2. The **`ReverseProxy` CLIENT span** (kind=3) represents the outbound forwarding to the backend, with target URL, peer address, and response code.
3. The **backend SERVER span** and its **INTERNAL middleware children** complete the picture.

The only "fragmented" traces are `/ping` and `/metrics` — Traefik's own internal health-check and Prometheus scrape endpoints. These produce single-span traces by design (no backend to forward to), and are clearly attributable to Kubernetes liveness probes and Prometheus scraping, not to user-traffic fragmentation.

Error handling is correct: 404 responses from the backend set `status.code=ERROR` on the `ReverseProxy` CLIENT span with the `http.response.status_code=404` attribute.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do most traces consist of a single isolated span with no parent or children? | **No** | 122/142 single-span traces are exclusively `/ping` health checks and `/metrics` scrapes — not user traffic. All 20 user-traffic traces are multi-span (7–40 spans). |
| Do requests produce multiple unrelated traces instead of one coherent trace? | **No** | Each HTTP request through Traefik produces exactly one coherent trace spanning Traefik + backend. |
| Are root spans created arbitrarily (no consistent SERVER kind at entry points)? | **No** | All 141 root spans are `kind=2` (SERVER). Entry-point spans consistently carry `entry_point` attribute. |
| Is context propagation absent (no incoming traceparent support)? | **No** | The 40-span trace proves incoming `traceparent` is extracted and continued. Source code confirms `ExtractCarrierIntoContext`. |
| Does async/background work create detached traces with new trace IDs? | **N/A** | No async/background work in Traefik's proxy model. |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Do synchronous HTTP request paths produce multi-span coherent traces? | **Yes** | All 20 user-traffic traces are multi-span with correct SERVER→CLIENT→SERVER→INTERNAL topology. |
| Does context propagation break for async execution, background jobs, or fan-out? | **No** | Fan-out (40-span trace) correctly continues the external trace context across 5 parallel requests. |
| Are span links used inconsistently or as a patch for propagation failures? | **No** | No span links present; fan-out uses correct parent-child relationships. |
| Do retries, redirects, or internal forwarding start new traces? | **No** | Integration tests confirm retries produce sibling spans within the same trace. |
| Is trace behavior undocumented or implicit? | **No** | Tracing documentation covers `sampleRate`, parent-based sampling, `capturedRequestHeaders`, and OTLP configuration. |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is W3C Trace Context supported and propagated consistently at entry points? | **Yes** | `autoprop` propagator, `ExtractCarrierIntoContext` at entry, `InjectContextIntoCarrier` before forwarding. External traceparent continued in 40-span trace. |
| Are parent-child vs span links used intentionally (not as a patch)? | **Yes** | Parent-child for synchronous request flow; no span links needed or used. |
| Are entry-point spans consistently `SERVER` kind? | **Yes** | All 141 root spans are `kind=2` (SERVER). |
| Do traces represent logical operations rather than internal function calls? | **Yes** | Span names (`GET`, `ReverseProxy`) represent HTTP operations, not internal Go function names. Middleware spans (`middleware - query`, `request handler - /`) come from the backend's auto-instrumentation. |
| Is trace topology stable across retries, fan-out, and async execution? | **Yes** | Integration tests validate retry topology; fan-out observed in telemetry with correct parent inheritance. |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Does trace topology support complex async or graph-shaped workflows? | **Partial** | Fan-out is handled correctly but Traefik is a proxy, not an orchestrator. No graph-shaped async workflows in scope. |
| Are trace modeling decisions reviewed architecturally? | **Unclear** | No public ADRs or design documents specifically for trace topology found. |
| Are trade-offs between trace completeness, cost, and clarity explicit? | **Partial** | `DetailedTracingEnabled` flag controls middleware/router span verbosity. `sampleRate` with parent-based sampling is documented. No explicit documentation of cost-clarity trade-offs. |
| Is trace behavior tested or validated over time? | **Yes** | `integration/tracing_test.go` includes `TestOpenTelemetryRetry`, `TestOpenTelemetryAuth`, `TestOpenTelemetryRateLimit`, `TestOpenTelemetrySafeURL`, and others that validate span topology assertions. |
| Do span links, events, and attributes enrich understanding intentionally? | **Partial** | Rich HTTP semantic attributes on entry-point and ReverseProxy spans. No span events observed. Span links absent (not needed). `entry_point` custom attribute aids routing context. |

---

#### Rationale

Traefik v3.7.0 is assigned **Level 2 — OpenTelemetry-Native** because:

1. **W3C Trace Context propagation is correct and complete**: Traefik uses `autoprop.NewTextMapPropagator()` to extract incoming `traceparent` at entry points and inject it into outgoing proxied requests. This is validated both in source code (`ExtractCarrierIntoContext` / `InjectContextIntoCarrier`) and in the observed telemetry (the 40-span trace continues an external trace ID).

2. **Span kinds are intentional and correct**: Entry-point spans are consistently `SERVER` (kind=2), outgoing proxy calls are `CLIENT` (kind=3), and internal middleware steps are `INTERNAL` (kind=1). This reflects deliberate semantic convention adherence.

3. **Traces represent logical operations**: Span names (`GET`, `ReverseProxy`) map to HTTP-level operations. The trace topology (SERVER → CLIENT → SERVER → INTERNAL chain) tells a coherent story of an inbound request being proxied to a backend.

4. **Single-span "fragments" are by design**: The 122 single-span traces are exclusively Kubernetes liveness probes (`/ping`) and Prometheus metrics scrapes (`/metrics`) — internal Traefik endpoints with no backend forwarding. These are not fragmentation; they are correct single-operation traces.

5. **Fan-out and retries are handled correctly**: The 40-span trace demonstrates correct parent inheritance across 5 parallel backend calls. Integration tests validate that retries remain within the same trace.

Level 3 is not assigned because: (a) no architectural documentation of trace modeling decisions was found, (b) span events are absent (no error details beyond status code), (c) the `DetailedTracingEnabled` flag controls verbosity but the trade-offs are not explicitly documented, and (d) there is no evidence of trace topology being reviewed or refined over time beyond integration test coverage.
