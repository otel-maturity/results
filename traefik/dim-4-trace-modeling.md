### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure
- Total root spans: 73
- Total child spans: 170
- Total spans (all services): 243
- Multi-span traces: 20
- Single-span traces: 54
- Span kind distribution: UNSPECIFIED=0, INTERNAL=117, SERVER=102, CLIENT=24, PRODUCER=0, CONSUMER=0

**Note on single-span traces:** All 54 single-span traces are legitimate — they correspond to `/ping` (Kubernetes liveness probe, no upstream backend) and `/metrics` (Prometheus scrape endpoint). These endpoints terminate at Traefik itself and correctly produce single SERVER spans. They are not fragmentation artifacts.

##### Context propagation
W3C Trace Context (`traceparent`) is documented as the **default** propagator in Traefik's OpenTelemetry documentation:

> "Traefik supports the `OTEL_PROPAGATORS` env variable to set up the propagators. The supported propagators are: tracecontext (default), baggage (default), b3, b3multi, jaeger, xray, ottrace"

This is confirmed in source code (`pkg/tracing/tracing.go`) via `autoprop.NewTextMapPropagator()` from `go.opentelemetry.io/contrib/propagators/autoprop`, which respects `OTEL_PROPAGATORS` and defaults to W3C `tracecontext + baggage`.

**Observed evidence:** Trace `4bf92f3577b34da6a3ce929d0e0e4736` (40 spans) shows 5 Traefik `GET` (SERVER) spans all correctly parented to external span `00f067aa0ba902b7` — a span ID not present in the collected JSONL, meaning it was injected via an incoming `traceparent` header. Traefik correctly extracted and honored this context, linking its spans as children of the external caller's span.

Context is also correctly propagated **outbound**: the `ReverseProxy` (CLIENT) spans inject the `traceparent` header into the upstream request, causing the backend (`otel-eval-backend`) to produce SERVER spans parented to Traefik's CLIENT span within the same trace ID.

##### Async/background work
No asynchronous or background processing paths were observed in the trace data. Traefik is a synchronous proxy; all observed request flows are synchronous HTTP proxying. No detached traces or broken context from async execution were observed.

##### Span links usage
No span links were observed in the trace data. This is appropriate — Traefik's proxy model is purely synchronous and does not require fan-out or async cross-trace linking patterns.

##### Trace coherence assessment
Traces tell a clear, user-comprehensible story. A complete proxied request produces the following coherent chain:

```
GET (traefik, SERVER, kind=2)                    ← entry point, extracts incoming traceparent
  └─ ReverseProxy (traefik, CLIENT, kind=3)      ← outgoing call to upstream, injects traceparent
       └─ GET / (otel-eval-backend, SERVER, kind=2)   ← upstream receives context
            ├─ middleware - query (INTERNAL)
            ├─ middleware - expressInit (INTERNAL)
            ├─ middleware - jsonParser (INTERNAL)
            ├─ middleware - <anonymous> (INTERNAL)
            └─ request handler - / (INTERNAL)
```

This structure is consistent across all 17 proxied-request traces (8-span and 7-span). The 7-span variant (404 error) correctly omits the `request handler` span since the route does not match. Error paths correctly set `status.code=2` on the `ReverseProxy` CLIENT span.

The instrumentation scope is `github.com/traefik/traefik`, confirming native (not auto-instrumented) span creation by Traefik itself. Semantic conventions v1.26.0 are explicitly referenced in docs and confirmed by observed attributes (`http.request.method`, `url.path`, `server.address`, `network.peer.address`, etc.).

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do most traces consist of a single isolated span with no parent or children? | **No** | 20 multi-span traces; single-span traces are only for terminal endpoints (/ping, /metrics) |
| Do requests produce multiple unrelated traces instead of one coherent trace? | **No** | Each proxied request produces exactly one trace spanning Traefik + backend |
| Are root spans created arbitrarily (no consistent SERVER kind at entry points)? | **No** | All 73 root spans are `kind=2` (SERVER); consistent and intentional |
| Is context propagation absent (no incoming traceparent support)? | **No** | W3C traceparent extraction confirmed by trace `4bf92f...` (external parent honored) |
| Does async/background work create detached traces with new trace IDs? | **N/A** | No async paths; all flows are synchronous |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Do synchronous HTTP request paths produce multi-span coherent traces? | **Yes** | 17 multi-span traces with complete SERVER→CLIENT→SERVER chains |
| Does context propagation break for async execution, background jobs, or fan-out? | **N/A** | No async paths observed; not applicable to Traefik's proxy model |
| Are span links used inconsistently or as a patch for propagation failures? | **No** | Span links absent; not needed for synchronous proxy model |
| Do retries, redirects, or internal forwarding start new traces? | **No** | Trace IDs are consistent across Traefik and backend spans |
| Is trace behavior undocumented or implicit? | **No** | Propagators, span semantics, and configuration are explicitly documented |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is W3C Trace Context supported and propagated consistently at entry points? | **Yes** | Default propagator is `tracecontext`; confirmed by observed external parent extraction |
| Are parent-child vs span links used intentionally (not as a patch)? | **Yes** | Parent-child used correctly for synchronous calls; no span links needed or used |
| Are entry-point spans consistently `SERVER` kind? | **Yes** | All 73 root spans are `kind=2` (SERVER), with `entry_point` attribute set |
| Do traces represent logical operations rather than internal function calls? | **Yes** | Span names (`GET`, `ReverseProxy`) represent proxy operations, not Go function names |
| Is trace topology stable across retries, fan-out, and async execution? | **Yes (scoped)** | Topology is stable for all observed flows; Traefik's synchronous model limits scope |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Does trace topology support complex async or graph-shaped workflows? | **No** | Traefik is a synchronous proxy; no async topology present or needed |
| Are trace modeling decisions reviewed architecturally? | **Partial** | Source code shows deliberate design (`autoprop`, semantic conventions v1.26.0) but no public trace modeling ADRs found |
| Are trade-offs between trace completeness, cost, and clarity explicit? | **Partial** | `sampleRate` and `addInternals` options documented; no explicit cost/clarity trade-off discussion |
| Is trace behavior tested or validated over time? | **Unknown** | No trace structure tests found in public repository review |
| Do span links, events, and attributes enrich understanding intentionally? | **Partial** | Attributes are semantically correct (OTel HTTP semconv v1.26.0); no span events observed; no span links |

#### Rationale

Traefik achieves **Level 2 (OpenTelemetry-Native)**. The evidence is strong and consistent:

1. **W3C Trace Context is the default propagator** — documented explicitly and confirmed by observed trace data where an external `traceparent` was extracted and honored, producing a coherent cross-service trace.

2. **Outbound context injection works correctly** — The `ReverseProxy` (CLIENT) span injects context into upstream requests, causing the backend to produce spans within the same trace. This is the essential behavior for a proxy and it works correctly.

3. **Span kinds are intentional and correct** — Entry points are consistently `SERVER` (kind=2), outbound proxy calls are `CLIENT` (kind=3), internal middleware spans are `INTERNAL` (kind=1). This is not accidental — it reflects deliberate modeling.

4. **Trace topology is logically meaningful** — The `GET → ReverseProxy → GET /` chain represents what a user cares about: "request arrived, was proxied, upstream responded." Span names are operational, not implementation-internal.

5. **Single-span traces are not fragmentation** — The 54 single-span traces are for `/ping` and `/metrics`, which are terminal endpoints at Traefik itself. This is correct modeling, not a deficiency.

Level 3 is not reached because: (a) no span events are used, (b) no evidence of architectural trace modeling review or trace structure tests, and (c) the `addInternals` option (for tracing internal resources like ping) and sampling trade-offs are documented but not deeply analyzed. The project's synchronous proxy nature also limits the opportunity to demonstrate advanced async/fan-out trace modeling.
