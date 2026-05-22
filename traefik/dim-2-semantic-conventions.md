### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

All 11 deprecated HTTP/network attributes appear exclusively on spans emitted by the **backend service's Node.js auto-instrumentation** (`@opentelemetry/instrumentation-http`), not by Traefik itself. Each deprecated attribute appears exactly **24 times** (one per observed request on the backend):

| Deprecated Attribute | Count | Source Scope |
|---|---|---|
| `http.method` | 24 | `@opentelemetry/instrumentation-http` |
| `http.status_code` | 24 | `@opentelemetry/instrumentation-http` |
| `http.url` | 24 | `@opentelemetry/instrumentation-http` |
| `http.target` | 24 | `@opentelemetry/instrumentation-http` |
| `http.host` | 24 | `@opentelemetry/instrumentation-http` |
| `http.scheme` | 24 | `@opentelemetry/instrumentation-http` |
| `http.flavor` | 24 | `@opentelemetry/instrumentation-http` |
| `http.user_agent` | 24 | `@opentelemetry/instrumentation-http` |
| `net.host.name` | 24 | `@opentelemetry/instrumentation-http` |
| `net.host.port` | 24 | `@opentelemetry/instrumentation-http` |
| `net.peer.port` | 24 | `@opentelemetry/instrumentation-http` |

Additional non-standard attributes also present on these spans: `net.host.ip`, `net.peer.ip`, `net.transport`, `http.status_text`, `http.client_ip`.

**Traefik's own spans** (`github.com/traefik/traefik` scope) do **not** use these deprecated attributes.

##### Current OTel attributes found on spans

Traefik's own spans (`github.com/traefik/traefik` scope) use current stable OTel HTTP semconv attributes:

| Current Attribute | Count | Notes |
|---|---|---|
| `http.request.method` | 98 | ✅ Current semconv |
| `http.response.status_code` | 98 | ✅ Current semconv |
| `url.scheme` | 98 | ✅ Current semconv |
| `server.address` | 98 | ✅ Current semconv |
| `network.protocol.version` | 98 | ✅ Current semconv |
| `network.peer.address` | 98 | ✅ Current semconv |
| `network.peer.port` | 98 | ✅ Current semconv |
| `client.address` | 74 | ✅ Current semconv |
| `client.port` | 74 | ✅ Current semconv |
| `url.path` | 74 | ✅ Current semconv |
| `url.query` | 74 | ✅ Current semconv |
| `http.request.body.size` | 74 | ✅ Current semconv |
| `url.full` | 24 | ✅ Current semconv |
| `server.port` | 24 | ✅ Current semconv |
| `user_agent.original` | present | ✅ Current semconv |

Traefik-specific span attribute: `entry_point` (non-OTel, proprietary domain label).

Spans from `@opentelemetry/instrumentation-express` also present with `express.name`, `express.type`, `http.route` attributes.

##### Metric names and conventions

**OTel semantic convention metric names (current):**
- `http.server.request.duration` — ✅ OTel semconv histogram; attributes use current keys (`http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `url.scheme`, `error.type`)
- `http.client.request.duration` — ✅ OTel semconv histogram; same attribute set

**Traefik-proprietary metric names (Prometheus-style, non-OTel semconv naming):**
- `traefik_entrypoint_request_duration_seconds` — Prometheus naming convention, not OTel semconv
- `traefik_entrypoint_requests_total` — Prometheus naming convention
- `traefik_entrypoint_requests_bytes_total` — Prometheus naming convention
- `traefik_entrypoint_responses_bytes_total` — Prometheus naming convention
- `traefik_router_request_duration_seconds` — Prometheus naming convention
- `traefik_router_requests_total`, `_bytes_total`, `_responses_bytes_total`
- `traefik_service_request_duration_seconds` — Prometheus naming convention
- `traefik_service_requests_total`, `_bytes_total`, `_responses_bytes_total`
- `traefik_open_connections` — Prometheus naming convention
- `traefik_config_reloads_total`, `traefik_config_last_reload_success`

**Traefik-specific metric attribute labels (non-OTel semconv):**
- `entrypoint`, `router`, `service` — Traefik-domain labels, no OTel equivalent
- `method`, `code`, `protocol` — bare abbreviations instead of OTel semconv keys (`http.request.method`, `http.response.status_code`, `network.protocol.name`)

**Infrastructure metrics (OTel semconv-aligned):**
- `k8s.*` metrics — ✅ Use OTel k8s semconv naming
- `go_*`, `process_*` — Prometheus Go/process collector naming (scraped, not OTel-native)

**Note:** Two metric naming paradigms coexist: OTel semconv (`http.server.request.duration`) and Prometheus-style Traefik metrics (`traefik_*`). The Traefik-proprietary metrics use attribute keys (`method`, `code`, `protocol`) that do not match OTel semconv keys used in traces (`http.request.method`, `http.response.status_code`).

##### Log attributes

Log records carry Traefik's **access log fields** as structured attributes — entirely proprietary, non-OTel naming:

```
ClientAddr, ClientHost, ClientPort, ClientUsername
DownstreamContentSize, DownstreamStatus
Duration, Overhead
entryPointName
KubernetesIngressName, KubernetesIngressNamespace, KubernetesServiceName, KubernetesServicePort
OriginContentSize, OriginDuration, OriginStatus
RequestAddr, RequestContentSize, RequestCount, RequestHost, RequestMethod
RequestPath, RequestPort, RequestProtocol, RequestScheme
RetryAttempts, RouterName
ServiceAddr, ServiceName, ServiceURL
SpanId / span_id, TraceId / trace_id   ← duplicated with different casing
StartLocal, StartUTC
```

Key issues:
- **All attribute names are PascalCase or camelCase proprietary names** — none follow OTel log semconv (e.g., `http.request.method`, `http.response.status_code`, `url.path`, `client.address`)
- **Duplicate trace correlation keys**: both `TraceId`+`SpanId` (PascalCase) and `trace_id`+`span_id` (snake_case) appear — inconsistent casing within the same record
- Log body is a **JSON string blob** rather than structured OTel attributes
- 24 log records carry valid trace context (traceId/spanId populated)

##### Cross-signal consistency

| Concept | Traces (Traefik scope) | Metrics (OTel) | Metrics (Traefik) | Logs |
|---|---|---|---|---|
| HTTP method | `http.request.method` ✅ | `http.request.method` ✅ | `method` ❌ | `RequestMethod` ❌ |
| HTTP status code | `http.response.status_code` ✅ | `http.response.status_code` ✅ | `code` ❌ | `DownstreamStatus` ❌ |
| URL path | `url.path` ✅ | — | — | `RequestPath` ❌ |
| URL scheme | `url.scheme` ✅ | `url.scheme` ✅ | `protocol` ❌ | `RequestScheme` ❌ |
| Network protocol | `network.protocol.version` ✅ | `network.protocol.version` ✅ | `protocol` ❌ | `RequestProtocol` ❌ |
| Server address | `server.address` ✅ | `server.address` ✅ | — | `RequestHost` ❌ |
| Entry point | `entry_point` (custom) | `entrypoint` (custom) | `entrypoint` | `entryPointName` |

**Observations:**
- Traefik's own spans and OTel metrics (`http.server.request.duration`, `http.client.request.duration`) share 6 common OTel semconv keys consistently
- Traefik's proprietary Prometheus-style metrics use abbreviated, non-OTel attribute keys (`method`, `code`, `protocol`) that do not match trace attributes
- Log attributes share zero OTel semconv keys with traces or metrics
- The entry point concept is named differently across all three signals (`entry_point` vs `entrypoint` vs `entryPointName`)

##### Schema URL

| Signal | Schema URL | Notes |
|---|---|---|
| Traces | `https://opentelemetry.io/schemas/1.40.0` | ✅ Present, current |
| Metrics | `https://opentelemetry.io/schemas/1.40.0` (Traefik) + `https://opentelemetry.io/schemas/1.18.0` (collector) | ✅ Present, mixed versions |
| Logs | `https://opentelemetry.io/schemas/1.40.0` | ✅ Present, current |

Schema URLs are present across all signals, indicating intent toward semconv governance.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|---|---|---|
| Are attribute names ad-hoc (e.g. `statusCode`, `resp_code`, `requestPath`)? | **Partial** | Log attributes are fully ad-hoc (`RequestMethod`, `DownstreamStatus`, `ClientAddr`). Traefik metric labels (`method`, `code`) are abbreviated. Traefik span attributes are OTel-aligned. |
| Are deprecated OTel attributes used (`http.method`, `http.status_code`, `http.target`)? | **Yes (third-party)** | 11 deprecated attributes present, but only from the backend Node.js auto-instrumentation, not Traefik itself |
| Is the same concept named differently across signals? | **Yes** | HTTP method: `http.request.method` (traces) vs `method` (Traefik metrics) vs `RequestMethod` (logs) |
| Is semantic meaning encoded in span names rather than attributes? | **No** | Traefik uses `GET`, `ReverseProxy` as span names but carries proper attributes |
| Do users need to consult source code to understand attribute meaning? | **Partial** | Log attributes require Traefik-specific knowledge; span/OTel metric attributes are self-describing |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|---|---|---|
| Are *some* OTel semantic conventions used (not zero)? | **Yes** | Traefik spans and `http.server/client.request.duration` metrics use current OTel semconv fully |
| Are deprecated and current OTel attributes mixed? | **Yes (across scopes)** | Deprecated on backend spans (`@opentelemetry/instrumentation-http`), current on Traefik spans — mixed in the same trace pipeline |
| Are conventions applied to traces but not consistently to metrics/logs? | **Yes** | Traces: OTel-aligned; Traefik proprietary metrics: non-OTel keys; Logs: fully proprietary |
| Are similar concepts named differently across signals? | **Yes** | HTTP method, status code, scheme all named differently across traces/Traefik-metrics/logs |
| Are attribute types inconsistent? | **Minor** | `http.response.status_code` is int on spans, string (`code=200`) on Traefik Prometheus metrics |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|---|---|---|
| Are **current, stable** OTel HTTP attributes used? | **Partial** | Traefik spans: ✅ yes; Traefik Prometheus metrics: ❌ no (`method`, `code`, `protocol`); Logs: ❌ no |
| Are **all** deprecated attributes removed? | **No** | 11 deprecated attributes present from backend auto-instrumentation (different scope but same pipeline) |
| Are attribute names consistent across traces, metrics, and logs? | **No** | Significant inconsistency between Traefik's Prometheus metrics and OTel metric/trace attribute keys; logs fully diverge |
| Are attributes placed in the correct scope? | **Mostly** | Resource attributes properly scoped; span attributes well-structured |
| Can telemetry be interpreted using generic OTel knowledge without project-specific mapping? | **Partial** | Traefik spans and OTel metrics: yes; Traefik Prometheus metrics and logs: no |

##### Level 3 — Semantic Extension & Stewardship

| Question | Answer | Evidence |
|---|---|---|
| Are domain-specific concepts modeled as explicit attributes? | **Partial** | `entry_point`, `router`, `service` are explicit but inconsistently named across signals |
| Are custom attributes documented with name, type, and semantic meaning? | **Partial** | Traefik docs describe telemetry configuration but do not provide a formal attribute registry |
| Do custom attributes extend OTel conventions rather than replace them? | **No** | Traefik Prometheus metrics use entirely non-OTel attribute keys alongside OTel-named metrics |
| Are semantic changes versioned and reviewed? | **No evidence** | No semconv versioning or changelog found for Traefik-specific attributes |
| Are proprietary schemas explicitly documented as extensions? | **No** | Traefik access log format is not documented as an OTel extension |

---

#### Rationale

Traefik is assigned **Level 1 — OpenTelemetry-Aligned** because it partially adopts OTel semantic conventions, but with significant inconsistencies across signals and naming paradigms.

**What earns Level 1 (not Level 0):**
- Traefik's own spans (`github.com/traefik/traefik` scope) use fully current OTel HTTP semconv attributes (`http.request.method`, `http.response.status_code`, `url.path`, `url.scheme`, `server.address`, `network.protocol.version`, `user_agent.original`, `client.address`, etc.) — no deprecated attributes from Traefik itself
- The OTel metrics `http.server.request.duration` and `http.client.request.duration` use correct current semconv attribute keys
- Schema URLs are present on all three signals, indicating intentional OTel alignment
- Trace context (traceId/spanId) is propagated into logs, enabling correlation

**Why Level 2 is not reached:**
1. **Dual metric naming paradigms**: The primary Traefik metrics (`traefik_*`) use Prometheus-style naming with non-OTel attribute keys (`method`, `code`, `protocol`, `entrypoint`, `router`, `service`) that conflict with the OTel semconv keys used in traces. A user cannot use a single OTel dashboard for both Traefik metric types without key normalization.
2. **Log attributes are fully proprietary**: Access log fields (`RequestMethod`, `DownstreamStatus`, `ClientAddr`, `RequestPath`, etc.) share no attribute names with OTel semconv. The log body is a JSON string blob rather than structured OTel attributes.
3. **Deprecated attributes in pipeline**: 11 deprecated attributes from `@opentelemetry/instrumentation-http` (the backend service) appear in the same trace pipeline — while not Traefik's own instrumentation, they exist within the evaluated telemetry corpus.
4. **Cross-signal inconsistency**: The same concept (HTTP method, status code, entry point) is named differently across traces, Traefik metrics, and logs — requiring project-specific knowledge to correlate.
5. **Duplicate trace context keys in logs**: Both `TraceId`/`SpanId` (PascalCase) and `trace_id`/`span_id` (snake_case) appear in the same log record — a sign of ad-hoc integration rather than governed semconv adoption.
