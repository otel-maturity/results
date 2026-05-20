### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

The following deprecated OTel attributes (pre-semconv 1.20) were found on spans emitted by the `@opentelemetry/instrumentation-http` scope (Node.js backend spans traversing Traefik):

| Deprecated Attribute | Count |
|---|---|
| `http.method` | 24 |
| `http.status_code` | 24 |
| `http.url` | 24 |
| `http.target` | 24 |
| `http.host` | 24 |
| `http.scheme` | 24 |
| `http.flavor` | 24 |
| `http.user_agent` | 24 |
| `net.host.name` | 24 |
| `net.host.port` | 24 |
| `net.peer.port` | 24 |

Additionally, Traefik itself emits several non-standard span attributes:
- `http.client_ip` (non-standard)
- `http.status_text` (non-standard)
- `net.host.ip` (deprecated)
- `net.peer.ip` (deprecated)
- `net.transport` (deprecated)
- `entry_point` (Traefik-specific, not namespaced)
- `express.name`, `express.type` (framework-specific, non-OTel)

##### Current OTel attributes found on spans

Traefik's own instrumentation scope (`github.com/traefik/traefik`) correctly emits current OTel semconv attributes:

| Current Attribute | Count |
|---|---|
| `http.request.method` | 63 |
| `http.response.status_code` | 63 |
| `url.scheme` | 63 |
| `server.address` | 63 |
| `network.peer.address` | 63 |
| `network.peer.port` | 63 |
| `network.protocol.version` | 63 |
| `url.path` | 39 |
| `url.query` | 39 |
| `http.request.body.size` | 39 |
| `client.address` | 39 |
| `client.port` | 39 |
| `url.full` | 24 |
| `server.port` | 24 |
| `user_agent.original` | present |
| `http.route` | present |

##### Metric names and conventions

Two distinct naming patterns coexist in Traefik's metrics:

**OTel-aligned metrics** (from `github.com/traefik/traefik` scope):
- `http.server.request.duration` — follows OTel HTTP server semconv; uses `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `url.scheme`, `error.type`
- `http.client.request.duration` — follows OTel HTTP client semconv; uses same set plus `server.port`

**Proprietary Traefik metrics** (from `github.com/traefik/traefik` scope, Prometheus-style):
- `traefik_entrypoint_request_duration_seconds` — uses `code`, `entrypoint`, `method`, `protocol` (not OTel semconv)
- `traefik_entrypoint_requests_total`, `traefik_entrypoint_requests_bytes_total`, `traefik_entrypoint_responses_bytes_total`
- `traefik_router_request_duration_seconds`, `traefik_router_requests_total`, etc. — uses `code`, `method`, `protocol`, `router`, `service`
- `traefik_service_request_duration_seconds`, `traefik_service_requests_total`, etc.
- `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`

The proprietary metrics use short, Prometheus-style attribute names (`code`, `method`, `protocol`) instead of OTel semconv equivalents (`http.response.status_code`, `http.request.method`, `network.protocol.name`).

##### Log attributes

Log attributes are entirely Traefik-proprietary with PascalCase naming — no OTel semantic conventions applied:

- `ClientAddr`, `ClientHost`, `ClientPort`, `ClientUsername`
- `DownstreamContentSize`, `DownstreamStatus`
- `Duration`, `Overhead`, `OriginDuration`, `OriginContentSize`, `OriginStatus`
- `RequestMethod`, `RequestPath`, `RequestHost`, `RequestScheme`, `RequestProtocol`, `RequestPort`, `RequestAddr`, `RequestContentSize`, `RequestCount`
- `RouterName`, `ServiceName`, `ServiceAddr`, `ServiceURL`
- `KubernetesIngressName`, `KubernetesIngressNamespace`, `KubernetesServiceName`, `KubernetesServicePort`
- `RetryAttempts`, `StartLocal`, `StartUTC`, `entryPointName`
- `TraceId`, `SpanId` (duplicated as `trace_id`, `span_id` — inconsistent casing)

Log bodies are serialized as a single JSON string blob rather than structured OTel attributes. Equivalent OTel semconv attributes exist for most of these concepts (e.g., `http.request.method`, `http.response.status_code`, `url.path`, `client.address`) but are not used.

##### Cross-signal consistency

Significant inconsistency across signals for the same concepts:

| Concept | Traces (Traefik scope) | Metrics (OTel-aligned) | Metrics (Traefik proprietary) | Logs |
|---|---|---|---|---|
| HTTP method | `http.request.method` | `http.request.method` | `method` | `RequestMethod` |
| HTTP status | `http.response.status_code` | `http.response.status_code` | `code` | `DownstreamStatus` |
| Protocol | `network.protocol.version` | `network.protocol.version` | `protocol` | `RequestProtocol` |
| Router | `entry_point` (non-standard) | — | `router`, `entrypoint` | `RouterName`, `entryPointName` |
| Service | — | — | `service` | `ServiceName` |

Keys shared between traces and OTel-aligned metrics: `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`. However, no keys are shared between traces/metrics and logs.

##### Schema URL

| Signal | Schema URL |
|---|---|
| Traces | `https://opentelemetry.io/schemas/1.40.0` — present |
| Metrics | `https://opentelemetry.io/schemas/1.40.0` (Traefik scope), `https://opentelemetry.io/schemas/1.18.0` (prometheus receiver) — present but mixed versions |
| Logs | `https://opentelemetry.io/schemas/1.40.0` — present |

Schema URLs are declared, showing governance intent, but the mixed versions in metrics and the presence of deprecated attributes indicate the schema is not fully enforced.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|---|---|---|
| Are attribute names ad-hoc (e.g. `statusCode`, `resp_code`, `requestPath`)? | Partially | Log attributes are fully ad-hoc PascalCase (`RequestMethod`, `DownstreamStatus`, `ClientAddr`); Traefik proprietary metric attributes use short names (`code`, `method`, `protocol`) |
| Are deprecated OTel attributes used (`http.method`, `http.status_code`, `http.target`)? | Yes | 11 deprecated attributes found on 24 spans each (from upstream Node.js instrumentation in the trace pipeline) |
| Is the same concept named differently across signals? | Yes | HTTP method: `http.request.method` (traces/OTel metrics) vs `method` (Traefik metrics) vs `RequestMethod` (logs) |
| Is semantic meaning encoded in span names rather than attributes? | No | Span names are standard HTTP method + path patterns |
| Do users need to consult source code to understand attribute meaning? | Partially | Log attributes require Traefik-specific knowledge |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|---|---|---|
| Are *some* OTel semantic conventions used (not zero)? | Yes | Traefik spans use current OTel HTTP semconv; `http.server.request.duration` and `http.client.request.duration` metrics use full OTel semconv attributes |
| Are deprecated and current OTel attributes mixed? | Yes | Deprecated `http.method`, `http.status_code`, `http.target` etc. coexist with current `http.request.method`, `http.response.status_code` in the same trace pipeline |
| Are conventions applied to traces but not consistently to metrics/logs? | Yes | Traefik span attributes follow current OTel semconv; proprietary `traefik_*` metrics use short non-OTel attribute names; logs use entirely proprietary PascalCase naming |
| Are similar concepts named differently across signals? | Yes | HTTP status: `http.response.status_code` (traces, OTel metrics) vs `code` (Traefik metrics) vs `DownstreamStatus` (logs) |
| Are attribute types inconsistent? | No significant evidence | Status codes appear as integers consistently where OTel semconv is used |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|---|---|---|
| Are **current, stable** OTel HTTP attributes used (`http.request.method`, `http.response.status_code`, `url.path`, `url.full`)? | Partially | Yes in Traefik spans and OTel-aligned metrics; No in proprietary `traefik_*` metrics and logs |
| Are **all** deprecated attributes removed or gated? | No | Deprecated attributes present in trace pipeline (`@opentelemetry/instrumentation-http` scope) |
| Are attribute names consistent across traces, metrics, and logs? | No | Three different naming schemes coexist |
| Are attributes placed in the correct scope? | Partially | Resource attributes correctly follow OTel semconv; span attributes mostly correct for Traefik scope |
| Can telemetry be interpreted using generic OTel knowledge without project-specific mapping? | No | Proprietary `traefik_*` metrics and log attributes require Traefik-specific knowledge |

##### Level 3 — Semantic Extension & Stewardship

| Question | Answer | Evidence |
|---|---|---|
| Are domain-specific concepts modeled as explicit attributes? | Partially | `entry_point`, `router`, `service` are Traefik-specific concepts but inconsistently named across signals |
| Are custom attributes documented with name, type, and semantic meaning? | No | No comprehensive attribute documentation found |
| Do custom attributes extend OTel conventions rather than replace them? | Partially | Traefik spans add `entry_point` alongside OTel attributes; but proprietary metrics replace OTel attributes entirely |
| Are semantic changes versioned and reviewed? | No | No evidence of formal semconv versioning for custom attributes |
| If a first-class signal uses a proprietary schema, is it documented as an extension? | No | Proprietary metric attributes and log schema are not documented as explicit extensions |

---

#### Rationale

Traefik is assessed at **Level 1 — OpenTelemetry-Aligned** because:

1. **Partial OTel adoption**: Traefik's own tracing instrumentation (`github.com/traefik/traefik` scope) correctly uses current OTel HTTP semantic conventions for spans. The two OTel-standard metrics (`http.server.request.duration`, `http.client.request.duration`) also fully conform to OTel semconv with correct attribute names.

2. **Deprecated attributes coexist**: The trace pipeline includes spans from `@opentelemetry/instrumentation-http` (Node.js) that use 11 deprecated OTel attributes (`http.method`, `http.status_code`, `http.target`, `http.url`, `http.host`, `http.scheme`, `http.flavor`, `http.user_agent`, `net.host.name`, `net.host.port`, `net.peer.port`), which are mixed with Traefik's own current-semconv spans in the same pipeline.

3. **Proprietary metric attributes**: The majority of Traefik's metrics (`traefik_*` family) use short, Prometheus-style attribute names (`code`, `method`, `protocol`, `entrypoint`, `router`, `service`) that do not follow OTel semantic conventions, preventing use of off-the-shelf OTel dashboards for these metrics.

4. **Logs are fully proprietary**: Log attributes use PascalCase Traefik-specific naming (`RequestMethod`, `DownstreamStatus`, `ClientAddr`, etc.) with no OTel alignment. The log body is serialized as a single JSON string blob rather than structured attributes, making it impossible to use OTel-standard log queries.

5. **Cross-signal inconsistency**: HTTP method is expressed as `http.request.method` (traces/OTel metrics), `method` (Traefik metrics), and `RequestMethod` (logs) — three different names for the same concept across signals.

6. **Schema URL present** but does not enforce consistency — deprecated attributes and proprietary naming persist despite the schema URL declaration.

The project shows clear intent to adopt OTel conventions (evidenced by schema URLs, OTel-aligned span attributes, and two OTel-standard metrics) but has not completed the migration across all signals and metric families, placing it firmly at Level 1.
