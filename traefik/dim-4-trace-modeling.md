### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure
- Total root spans: 34
- Total child spans: 170
- Multi-span traces: 20
- Single-span traces: 15 (all are Traefik-internal: `/ping` health checks and `/metrics` scrapes — not real proxied traffic)
- Span kind distribution: UNSPECIFIED=0, INTERNAL=117, SERVER=63, CLIENT=24, PRODUCER=0, CONSUMER=0

##### Context propagation
W3C Trace Context is supported and propagated correctly. When the test client sends a `traceparent` header (observed in trace `4bf92f3577b34da6a3ce929d0e0e4736`), Traefik's entry-point `GET` spans carry the external span ID (`00f067aa0ba902b7`) as their `parentSpanId`, confirming that Traefik correctly extracts the incoming W3C trace context and continues the trace. This was confirmed across a concurrent load test with 5 simultaneous requests — all 5 Traefik `GET` spans correctly referenced the same external parent span ID. The instrumentation scope is `github.com/traefik/traefik`, indicating native, first-party OTel integration.

##### Async/background work
No async or background work was observed in the trace data. Traefik operates as a synchronous reverse proxy, and all observed traces represent synchronous HTTP request flows. The trace structure is: `GET` (SERVER, Traefik entry point) → `ReverseProxy` (CLIENT) → `GET /` or `GET /health` (SERVER, backend) → express middleware spans (INTERNAL). This chain is coherent and complete.

##### Span links usage
No span links were observed. Not applicable for Traefik's synchronous proxy model — parent-child relationships are used correctly and intentionally throughout.

##### Trace coherence assessment
Traces are highly coherent and tell a clear story. A user can follow a single HTTP request from Traefik's entry point through the reverse proxy call to the backend's handler and its middleware chain, all within a single trace. The pattern is:

```
GET (SERVER, Traefik, kind=2)                    ← entry point
  └── ReverseProxy (CLIENT, Traefik, kind=3)     ← outbound call
        └── GET / (SERVER, backend, kind=2)       ← backend receives
              ├── middleware - query (INTERNAL)
              ├── middleware - expressInit (INTERNAL)
              ├── middleware - jsonParser (INTERNAL)
              ├── middleware - <anonymous> (INTERNAL)
              └── request handler - / (INTERNAL)
```

Error cases (e.g., `GET /nonexistent` → 404) follow the same structure, with `ReverseProxy` spans correctly marked with `status.code=2` (ERROR) and `http.response.status_code=404`.

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do most traces consist of a single isolated span with no parent or children? | No | Only 15/35 traces are single-span, and all are internal health/metrics endpoints — not proxied traffic |
| Do requests produce multiple unrelated traces instead of one coherent trace? | No | Each proxied request produces exactly one coherent multi-span trace |
| Are root spans created arbitrarily (no consistent SERVER kind at entry points)? | No | All 34 root spans have kind=2 (SERVER), consistently |
| Is context propagation absent (no incoming traceparent support)? | No | W3C traceparent is correctly extracted and used as parent span ID |
| Does async/background work create detached traces with new trace IDs? | N/A | No async work present in this proxy model |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Do synchronous HTTP request paths produce multi-span coherent traces? | Yes | All proxied requests produce 7-8 span coherent traces (Traefik + backend) |
| Does context propagation break for async execution, background jobs, or fan-out? | N/A | No async paths present |
| Are span links used inconsistently or as a patch for propagation failures? | No | No span links used at all |
| Do retries, redirects, or internal forwarding start new traces? | No | Even 404 error paths maintain trace continuity |
| Is trace behavior undocumented or implicit? | Partially | Traefik documents OTel tracing but W3C propagation behavior is implicit |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is W3C Trace Context supported and propagated consistently at entry points? | Yes | Confirmed: external `traceparent` headers are accepted and used as `parentSpanId` |
| Are parent-child vs span links used intentionally (not as a patch)? | Yes | Parent-child relationships are used correctly throughout; no span links needed or misused |
| Are entry-point spans consistently `SERVER` kind? | Yes | All 34 root spans are kind=2 (SERVER) |
| Do traces represent logical operations rather than internal function calls? | Yes | Spans represent meaningful operations: entry point, reverse proxy, backend handler, middleware |
| Is trace topology stable across retries, fan-out, and async execution? | Yes (observed) | Concurrent load test (5 parallel requests) all maintained correct topology |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Does trace topology support complex async or graph-shaped workflows? | Partial | Synchronous proxy only; no evidence of async graph support |
| Are trace modeling decisions reviewed architecturally? | Unknown | No public evidence of architectural trace review process |
| Are trade-offs between trace completeness, cost, and clarity explicit? | No | No public documentation of sampling or cost/clarity trade-offs |
| Is trace behavior tested or validated over time? | Unknown | No evidence of trace topology regression testing |
| Do span links, events, and attributes enrich understanding intentionally? | Partial | Attributes are well-chosen (HTTP method, status, URL, peer address); no span events observed |

#### Rationale

Traefik earns **Level 2 (OpenTelemetry-Native)** for Dimension 4. The trace modeling is clearly intentional: entry points consistently use `SERVER` kind, outbound proxy calls use `CLIENT` kind, and the backend spans (from a separately instrumented service) are correctly stitched into the same trace via W3C Trace Context propagation. The `entry_point` attribute on root spans and the `ReverseProxy` span name reflect deliberate naming choices that describe logical operations rather than internal function calls.

The key evidence for Level 2 is the W3C Trace Context propagation: when an external caller sends a `traceparent` header, Traefik's entry-point span correctly sets that as its parent, creating a coherent cross-service trace without any gaps. This was observed across concurrent requests in a load test scenario.

Level 3 is not awarded because: (1) there is no evidence of architectural review of trace modeling decisions, (2) no span events are used to enrich trace understanding, (3) there is no public documentation of sampling trade-offs or trace completeness guarantees, and (4) the instrumentation scope version is reported as `?` (unknown), suggesting the version metadata is not fully populated.
