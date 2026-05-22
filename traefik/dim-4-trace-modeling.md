### 4. Trace Modeling & Context Propagation

**Level: 2 — OpenTelemetry-Native**

#### Evidence

##### Span structure
- Total root spans: 72
- Total child spans: 170
- Multi-span traces: 20
- Single-span traces: 53
- Span kind distribution: UNSPECIFIED=0, INTERNAL=117, SERVER=120 (kind=2 across Traefik+backend), CLIENT=24, PRODUCER=0, CONSUMER=0

> Note: Kind=2 (SERVER) is used by both `github.com/traefik/traefik` (96 spans) and `@opentelemetry/instrumentation-http` (24 spans). Kind=3 (CLIENT) is used exclusively for `ReverseProxy` spans from Traefik. Kind=1 (INTERNAL) is used for Express middleware spans from `@opentelemetry/instrumentation-express`.

##### Context propagation
W3C Trace Context is actively propagated. Traefik injects the `traceparent` header when forwarding requests to backends, and the backend (Node.js/Express) correctly picks it up — evidenced by traces that span **three instrumentation scopes** in a single `traceId`:

```
github.com/traefik/traefik        | GET           | kind=SERVER  | parent=ROOT (or external)
github.com/traefik/traefik        | ReverseProxy  | kind=CLIENT  | parent=GET
@opentelemetry/instrumentation-http    | GET /         | kind=SERVER  | parent=ReverseProxy
@opentelemetry/instrumentation-express | middleware-*  | kind=INTERNAL| parent=GET /
```

The documentation explicitly states: *"Since Traefik supports parent-based sampling ratios, root spans (i.e., spans initiated by Traefik) are sampled according to this rate, while child spans inherit the sampling decision of their parent (i.e., the tracing context from incoming requests)."* This confirms intentional W3C Trace Context support.

The 40-span trace (`4bf92f3577b34da6a3ce929d0e0e4736`) shows multiple parallel requests all parented under span `00f067aa0ba902b7`, which is not present in the collected data — indicating it was an external caller whose `traceparent` was correctly accepted and continued by Traefik.

##### Async/background work
No async/background work patterns were observed in the trace data. All spans belong to synchronous HTTP request-response flows. Internal Traefik spans (`/ping`, `/metrics`) correctly produce isolated single-span traces since those are direct requests to Traefik itself with no upstream propagation or backend forwarding.

##### Span links usage
**Absent.** No span links were observed in any trace. This is appropriate — all parent-child relationships are expressed via direct parentage, not links, because the flows are synchronous request chains without fan-out or async handoffs.

##### Trace coherence assessment
Traces are highly coherent and tell a clear end-to-end story. For proxied requests (path `/`), a user can follow: **external caller → Traefik entry-point (SERVER) → Traefik ReverseProxy (CLIENT) → backend HTTP server (SERVER) → Express middleware chain (INTERNAL)**. This is a textbook distributed trace structure. The 53 "single-span" traces are all for Traefik-internal endpoints (`/ping`, `/metrics`, `/health`) that are not proxied — these are correctly isolated and not fragmented proxied traces.

Error cases (404 on `/nonexistent`) correctly set `status.code=2` on the `ReverseProxy` CLIENT span while the backend's SERVER spans remain `UNSET`, accurately reflecting where the error was observed.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|----------|--------|----------|
| Do most traces consist of a single isolated span with no parent or children? | **No** | 53 single-span traces are all for non-proxied internal paths (`/ping`, `/metrics`, `/health`); proxied requests produce 7–40 span multi-service traces |
| Do requests produce multiple unrelated traces instead of one coherent trace? | **No** | Proxied requests produce a single `traceId` spanning Traefik + backend |
| Are root spans created arbitrarily (no consistent SERVER kind at entry points)? | **No** | All 72 root spans are `kind=2` (SERVER) — fully consistent |
| Is context propagation absent (no incoming traceparent support)? | **No** | W3C Trace Context is propagated; the 40-span trace shows external `traceparent` accepted and continued |
| Does async/background work create detached traces with new trace IDs? | **N/A** | No async work observed |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|----------|--------|----------|
| Do synchronous HTTP request paths produce multi-span coherent traces? | **Yes** | 20 multi-span traces cover the full Traefik → backend chain |
| Does context propagation break for async execution, background jobs, or fan-out? | **Not observed** | No async patterns tested; synchronous paths are fully coherent |
| Are span links used inconsistently or as a patch for propagation failures? | **No** | No span links used at all; parent-child is used correctly |
| Do retries, redirects, or internal forwarding start new traces? | **No** | The 40-span trace shows 5 parallel backend requests all sharing the same `traceId` |
| Is trace behavior undocumented or implicit? | **No** | Traefik docs explicitly document W3C Trace Context, parent-based sampling, and OTLP export |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|----------|--------|----------|
| Is W3C Trace Context supported and propagated consistently at entry points? | **Yes** | Propagated to backends via `traceparent`; external callers' context is accepted |
| Are parent-child vs span links used intentionally (not as a patch)? | **Yes** | Parent-child used for synchronous chain; no links needed or misused |
| Are entry-point spans consistently `SERVER` kind? | **Yes** | 100% of root spans are `kind=2` (SERVER) |
| Do traces represent logical operations rather than internal function calls? | **Yes** | Span names (`GET`, `ReverseProxy`) represent routing operations, not Go internals |
| Is trace topology stable across retries, fan-out, and async execution? | **Partial** | Fan-out is stable (40-span trace); retries and async not observed |

##### Level 3 — OpenTelemetry-Optimized

| Question | Answer | Evidence |
|----------|--------|----------|
| Does trace topology support complex async or graph-shaped workflows? | **Not observed** | Only synchronous HTTP proxy flows observed |
| Are trace modeling decisions reviewed architecturally? | **Unknown** | No public evidence of architectural trace review process |
| Are trade-offs between trace completeness, cost, and clarity explicit? | **Partial** | `sampleRate` and `addInternals` options exist but no explicit cost/clarity trade-off docs |
| Is trace behavior tested or validated over time? | **Unknown** | No trace validation tests found in public documentation |
| Do span links, events, and attributes enrich understanding intentionally? | **Partial** | Attributes are well-chosen (`entry_point`, `url.path`, `http.response.status_code`); no span events observed |

---

#### Rationale

Traefik earns **Level 2 — OpenTelemetry-Native** based on strong evidence of intentional, consistent trace modeling:

1. **Consistent SERVER kind at all entry points** — every single root span (72/72) uses `kind=2`, demonstrating deliberate modeling rather than accidental instrumentation.

2. **End-to-end W3C Trace Context propagation** — traces span three independent instrumentation scopes (Traefik core, Node.js HTTP, Express) within a single `traceId`, proving that `traceparent` is correctly injected by Traefik's `ReverseProxy` CLIENT span and received by the backend. The 40-span trace further shows that externally-originated `traceparent` headers are accepted and continued.

3. **Correct CLIENT/SERVER/INTERNAL span kind semantics** — `ReverseProxy` is `CLIENT` (kind=3) because it makes an outgoing call; the backend entry is `SERVER` (kind=2); Express middleware is `INTERNAL` (kind=1). This matches OTel semantic conventions precisely.

4. **Logical span names** — `GET`, `ReverseProxy`, `GET /` represent routing operations meaningful to operators, not internal function names.

5. **Documented behavior** — the Traefik documentation explicitly covers W3C Trace Context, parent-based sampling, and OTLP export.

Level 3 is not reached because: (a) no span events are used to enrich spans, (b) async/fan-out trace topology beyond parallel synchronous requests is not demonstrated, and (c) there is no evidence of architectural trace review or validation testing processes.
