### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure
- Total spans observed: 449
- Total root spans (no parentSpanId): 139
- Total child spans (have parentSpanId): 310
- Distinct trace IDs: 140
- Multi-span traces: 53 (38% of all traces)
- Single-span traces: 87 (62% of all traces)
- Span kind distribution: UNSPECIFIED=0, INTERNAL=217, SERVER=193, CLIENT=47, PRODUCER=0, CONSUMER=0
- Traefik-native span kinds (scope `github.com/traefik/traefik`): SERVER=155, CLIENT=52, INTERNAL=0

##### Span names observed
- Traefik scope (`github.com/traefik/traefik`): `GET` (SERVER, entry-point), `ReverseProxy` (CLIENT, upstream call)
- Backend scope (`@opentelemetry/instrumentation-http`): `GET /`, `GET /health` (SERVER)
- Backend scope (`@opentelemetry/instrumentation-express`): `middleware - query`, `middleware - expressInit`, `middleware - jsonParser`, `middleware - <anonymous>`, `request handler - /`, `request handler - /health` (INTERNAL)

##### Trace topology
A complete end-to-end request through Traefik produces a coherent 8-span trace:

```
GET (SERVER, Traefik)                            ← entry-point root span
  └── ReverseProxy (CLIENT, Traefik)             ← upstream call to backend
        └── GET / (SERVER, backend)              ← backend receives propagated context
              ├── middleware - query (INTERNAL)
              ├── middleware - expressInit (INTERNAL)
              ├── middleware - jsonParser (INTERNAL)
              ├── middleware - <anonymous> (INTERNAL)
              └── request handler - / (INTERNAL)
```

All 50 backend trace IDs are shared with Traefik trace IDs, confirming end-to-end context propagation across service boundaries.

##### Context propagation
W3C Trace Context (`traceparent`/`tracestate`) is fully supported and verified empirically. When a request arrives with `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`, Traefik correctly:
1. Continues the trace (preserves `traceId=4bf92f3577b34da6a3ce929d0e0e4736`)
2. Creates a new child `GET` SERVER span as a child of the injected `parentSpanId=00f067aa0ba902b7`
3. Propagates the updated `traceparent` to the backend via `ReverseProxy`
4. The backend receives and continues the trace with its own child spans

The 40-span trace (`4bf92f3577b34da6a3ce929d0e0e4736`) contains 5 complete request chains, all correctly parented under the externally injected span ID `00f067aa0ba902b7`.

Propagation is documented in Traefik v3 official docs as W3C Trace Context by default.

##### Async/background work
Traefik is a synchronous HTTP proxy — there are no async/background work patterns in its core request path. All observed traces are synchronous HTTP request/response chains. The `addInternals: true` configuration enables tracing of internal endpoints (`/ping`, `/metrics`, dashboard) which produce single-span traces (no backend forwarding), which is correct behavior for self-handled routes.

##### Span links usage
No span links observed (`links` field absent on all spans). This is appropriate — Traefik's request model is purely synchronous and does not require cross-trace linking.

##### Trace coherence assessment
A user can follow a complete logical operation through the trace:
- The `GET` SERVER span at the top represents the inbound HTTP request
- `ReverseProxy` CLIENT span represents Traefik forwarding to the upstream
- Backend spans continue the same trace ID, showing end-to-end request processing

Single-span traces fall into two categories:
1. **Legitimate single-span traces**: `/ping` (Kubernetes health probes handled internally), `/metrics` (Prometheus scrape handled internally) — these correctly have no downstream calls
2. **Batching artifacts**: Some `/` path traces appear as single Traefik spans because the backend OTLP export batches arrived in different JSONL records. The shared trace IDs confirm these are actually part of coherent multi-service traces

Error handling is also correct: 404 responses from the backend cause `ReverseProxy` CLIENT spans to be marked `status.code=2 (ERROR)` with `http.response.status_code=404` attribute, while the parent `GET` SERVER span remains `UNSET` (correct — the proxy itself succeeded).

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do most traces consist of a single isolated span with no parent or children? | **No** | 53 multi-span traces with full parent-child hierarchy; all 50 backend trace IDs shared with Traefik |
| Do requests produce multiple unrelated traces instead of one coherent trace? | **No** | Single trace ID spans both Traefik and backend spans for all proxied requests |
| Are root spans created arbitrarily (no consistent SERVER kind at entry points)? | **No** | All 155 Traefik root spans are consistently `kind=2 (SERVER)` |
| Is context propagation absent (no incoming traceparent support)? | **No** | W3C Trace Context propagation confirmed: incoming `traceparent` trace ID preserved, new child span created |
| Does async/background work create detached traces with new trace IDs? | **N/A** | Traefik is synchronous; no async work patterns |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Do synchronous HTTP request paths produce multi-span coherent traces? | **Yes** | 8-span traces with correct parent-child hierarchy across Traefik + backend |
| Does context propagation break for async execution, background jobs, or fan-out? | **N/A** | Traefik has no async execution paths |
| Are span links used inconsistently or as a patch for propagation failures? | **No** | No span links used; not needed for synchronous proxy model |
| Do retries, redirects, or internal forwarding start new traces? | **No** | All backend calls continue the same trace ID |
| Is trace behavior undocumented or implicit? | **No** | W3C Trace Context support is documented in Traefik v3 official docs |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is W3C Trace Context supported and propagated consistently at entry points? | **Yes** | Verified empirically: injected `traceparent` trace ID preserved across Traefik→backend boundary |
| Are parent-child vs span links used intentionally (not as a patch)? | **Yes** | Parent-child hierarchy is semantically correct; span links correctly absent |
| Are entry-point spans consistently `SERVER` kind? | **Yes** | All 155 Traefik entry-point spans are `kind=2 (SERVER)` |
| Do traces represent logical operations rather than internal function calls? | **Yes** | `GET` (inbound request), `ReverseProxy` (upstream call) are logical operations, not internal Go function names |
| Is trace topology stable across retries, fan-out, and async execution? | **Yes** | Consistent topology observed across all 53 multi-span traces |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Does trace topology support complex async or graph-shaped workflows? | **No** | Traefik is a synchronous proxy; no complex async topology needed or present |
| Are trace modeling decisions reviewed architecturally? | **Unclear** | No public evidence of architectural trace review process in Traefik project |
| Are trade-offs between trace completeness, cost, and clarity explicit? | **Partial** | `addInternals` flag exists to control internal route tracing, but no explicit cost/clarity trade-off documentation |
| Is trace behavior tested or validated over time? | **Unclear** | No publicly visible trace integration tests or trace shape validation |
| Do span links, events, and attributes enrich understanding intentionally? | **Partial** | Rich semantic attributes (`entry_point`, `url.path`, `http.response.status_code`, `network.peer.address`) present; no span events observed |

#### Rationale

Traefik v3 earns **Level 2 (OpenTelemetry-Native)** for trace modeling and context propagation.

The trace structure is clearly intentional and semantically correct: entry-point spans are consistently `SERVER` kind, upstream proxy calls are `CLIENT` kind (`ReverseProxy`), and the parent-child hierarchy accurately represents the logical request flow. W3C Trace Context propagation is fully functional — empirically verified by injecting a `traceparent` header and confirming the trace ID propagates through Traefik to the backend, with Traefik correctly creating a child span under the externally-provided parent span ID. All 50 backend trace IDs are shared with Traefik trace IDs, confirming no context loss at the service boundary.

The span naming (`GET`, `ReverseProxy`) represents logical operations rather than internal implementation details, and the two Traefik span types cleanly model the two roles Traefik plays: HTTP server (receiving the request) and HTTP client (forwarding to upstream).

Level 3 is not awarded because there is no evidence of architectural trace review processes, explicit trade-off documentation between trace completeness and cost, or automated trace shape validation in the project's CI/CD. The `addInternals` configuration option shows some awareness of trace scope control, but this falls short of the active refinement and validation expected at Level 3.
