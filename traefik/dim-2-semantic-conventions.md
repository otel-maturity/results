### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

Deprecated attributes are present on **24 spans** emitted by the backend service's Node.js `@opentelemetry/instrumentation-http` (v0.48.0) library, which is captured as part of the distributed trace. These are not Traefik-native spans but they appear in the same trace data collected from the cluster:

| Deprecated Attribute | Count | Current Equivalent |
|---|---|---|
| `http.method` | 24 | `http.request.method` |
| `http.status_code` | 24 | `http.response.status_code` |
| `http.url` | 24 | `url.full` |
| `http.target` | 24 | `url.path` + `url.query` |
| `http.host` | 24 | `server.address` + `server.port` |
| `http.scheme` | 24 | `url.scheme` |
| `http.flavor` | 24 | `network.protocol.version` |
| `http.user_agent` | 24 | `user_agent.original` |
| `net.peer.name` / `net.peer.port` | 24 each | `server.address` / `server.port` |
| `net.host.name` / `net.host.port` | 24 each | `server.address` / `server.port` |

Additionally, the following legacy-style attributes appear on the same spans:
- `http.status_text` (no OTel equivalent — non-standard)
- `net.host.ip`, `net.peer.ip` (deprecated; replaced by `network.peer.address`)
- `net.transport` (deprecated; replaced by `network.transport`)

##### Current OTel attributes found on spans

Traefik's own spans (`github.com/traefik/traefik` scope, 117 spans) use **current stable OTel semantic conventions**:

| Current Attribute | Count |
|---|---|
| `http.request.method` | 101 |
| `http.response.status_code` | 101 |
| `url.scheme` | 101 |
| `url.path` | 77 |
| `url.query` | 77 |
| `url.full` | 24 |
| `server.address` | 101 |
| `server.port` | 24 |
| `network.peer.address` | 101 |
| `network.peer.port` | 101 |
| `network.protocol.version` | 101 |
| `client.address` | 77 |
| `client.port` | 77 |
| `user_agent.original` | 77 |
| `http.request.body.size` | 77 |

Traefik-specific span attributes (domain extensions, not OTel semconv violations):
- `entry_point` — Traefik entrypoint name (no OTel equivalent)

##### Metric names and conventions

Two distinct metric naming systems coexist:

**OTel semconv-aligned (emitted by `github.com/traefik/traefik` scope):**
- `http.server.request.duration` — follows OTel `http.server.*` convention
- `http.client.request.duration` — follows OTel `http.client.*` convention
- Attributes: `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`, `error.type` ✅

**Traefik-proprietary Prometheus metrics (scraped via Prometheus receiver):**
- `traefik_entrypoint_request_duration_seconds`, `traefik_entrypoint_requests_total`, `traefik_entrypoint_requests_bytes_total`, `traefik_entrypoint_responses_bytes_total`
- `traefik_router_request_duration_seconds`, `traefik_router_requests_total`, etc.
- `traefik_service_request_duration_seconds`, `traefik_service_requests_total`, etc.
- `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`
- Attributes on Traefik Prometheus metrics: `code`, `method`, `protocol`, `entrypoint`, `router`, `service` — **proprietary, non-OTel naming** (vs. `http.response.status_code`, `http.request.method`, `network.protocol.name`)

The Traefik Prometheus metrics are scraped from a `/metrics` endpoint and converted by the OTel Collector's Prometheus receiver. These metrics use short, Prometheus-style label names (`code`, `method`, `protocol`) rather than OTel semantic convention attribute names.

**Infrastructure/runtime metrics** follow OTel conventions:
- `k8s.*` metrics: `k8s.container.cpu_limit`, `k8s.pod.phase`, etc. ✅
- `go_*` and `process_*`: Prometheus-style naming (scraped, not native OTel)

##### Log attributes

Log records from the `traefik` scope (24 records) use **PascalCase proprietary attribute names** that do not follow OTel semantic conventions:

| Log Attribute | OTel Equivalent (if any) |
|---|---|
| `RequestMethod` | `http.request.method` |
| `DownstreamStatus` | `http.response.status_code` |
| `RequestPath` | `url.path` |
| `RequestScheme` | `url.scheme` |
| `RequestHost` | `server.address` |
| `ClientAddr` | `client.address` |
| `ClientPort` | `client.port` |
| `Duration` | no direct OTel equivalent (custom) |
| `ServiceName` | no OTel equivalent (Traefik-specific) |
| `RouterName` | no OTel equivalent (Traefik-specific) |
| `KubernetesIngressName` | no OTel equivalent |
| `SpanId` / `TraceId` | duplicate of OTLP `spanId`/`traceId` fields |
| `span_id` / `trace_id` | duplicate of OTLP `spanId`/`traceId` fields (snake_case variant) |

Log bodies are **stringified JSON blobs** (single `stringValue` string), not structured OTel log records with typed attributes. The access log data is embedded as a JSON string rather than as structured OTel log attributes.

Logs do carry trace context (24/24 records have non-zero `traceId`/`spanId`), enabling correlation.

##### Cross-signal consistency

The same HTTP concept is named **differently** across signals:

| Concept | Traces (Traefik spans) | Traces (backend spans) | Metrics (OTel) | Metrics (Prometheus) | Logs |
|---|---|---|---|---|---|
| HTTP method | `http.request.method` ✅ | `http.method` ❌ | `http.request.method` ✅ | `method` ❌ | `RequestMethod` ❌ |
| HTTP status | `http.response.status_code` ✅ | `http.status_code` ❌ | `http.response.status_code` ✅ | `code` ❌ | `DownstreamStatus` ❌ |
| URL path | `url.path` ✅ | `http.target` ❌ | — | — | `RequestPath` ❌ |
| Protocol version | `network.protocol.version` ✅ | `http.flavor` ❌ | `network.protocol.version` ✅ | `protocol` ❌ | `RequestProtocol` ❌ |
| Server address | `server.address` ✅ | `net.host.name` ❌ | `server.address` ✅ | — | `RequestHost` ❌ |

Keys shared consistently across **traces (Traefik spans) and OTel metrics**: `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme` ✅

No OTel-conventional keys are shared between traces/metrics and logs.

##### Schema URL

| Signal | Schema URL |
|---|---|
| Traces | `https://opentelemetry.io/schemas/1.40.0` ✅ |
| Metrics | `https://opentelemetry.io/schemas/1.40.0` (Traefik native) and `https://opentelemetry.io/schemas/1.18.0` (Prometheus-scraped) |
| Logs | `https://opentelemetry.io/schemas/1.40.0` ✅ |

Schema URLs are present on all three signals, indicating governance intent.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|---|---|---|
| Are attribute names ad-hoc (e.g. `statusCode`, `resp_code`, `requestPath`)? | **Partially** | Log attributes use PascalCase proprietary names (`RequestMethod`, `DownstreamStatus`, `ClientAddr`); Prometheus metrics use short labels (`code`, `method`, `protocol`) |
| Are deprecated OTel attributes used (`http.method`, `http.status_code`, `http.target`)? | **Yes (mixed)** | 24 spans from backend `@opentelemetry/instrumentation-http` carry all deprecated HTTP attributes; Traefik's own spans use current attributes |
| Is the same concept named differently across signals? | **Yes** | HTTP method: `http.request.method` (traces/OTel metrics) vs `method` (Prometheus metrics) vs `RequestMethod` (logs) |
| Is semantic meaning encoded in span names rather than attributes? | **No** | Span names are standard (`GET`, `ReverseProxy`) with attributes carrying meaning |
| Do users need to consult source code to understand attribute meaning? | **Partially** | Log attributes require Traefik-specific knowledge; span/OTel metric attributes are self-descriptive |

→ Not purely Level 0 — Traefik's native spans and OTel metrics use current conventions.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|---|---|---|
| Are *some* OTel semantic conventions used (not zero)? | **Yes** | Traefik native spans (117 total) use current stable OTel HTTP semconv; `http.server.request.duration` / `http.client.request.duration` metrics use OTel attributes |
| Are deprecated and current OTel attributes mixed? | **Yes** | Deprecated attributes from `@opentelemetry/instrumentation-http` (24 spans) coexist with current attributes on Traefik spans |
| Are conventions applied to traces but not consistently to metrics/logs? | **Yes** | OTel semconv applied to Traefik native spans and OTel metrics, but Prometheus-scraped `traefik_*` metrics use proprietary labels; logs use PascalCase proprietary attributes |
| Are similar concepts named differently across signals? | **Yes** | HTTP method: 3 different names across signals; HTTP status: 3 different names |
| Are attribute types inconsistent? | **Yes** | HTTP status code: string `"200"` in Prometheus metrics (`code`), integer `200` in OTel metrics (`http.response.status_code`) |

→ **Level 1 confirmed** — OTel conventions partially adopted but inconsistently applied.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|---|---|---|
| Are **current, stable** OTel HTTP attributes used? | **Partially** | Traefik native spans use current attributes; backend spans and Prometheus metrics do not |
| Are **all** deprecated attributes removed? | **No** | 24 spans carry deprecated `http.method`, `http.status_code`, `http.target`, `http.flavor`, `net.peer.*`, `net.host.*` |
| Are attribute names consistent across traces, metrics, and logs? | **No** | HTTP method alone has 3+ different names across signals |
| Are attributes placed in the correct scope? | **Partially** | Resource attributes are correctly placed; log body embeds access log as JSON string instead of structured attributes |
| Can telemetry be interpreted using generic OTel knowledge without project-specific mapping? | **No** | Prometheus `traefik_*` metrics require Traefik-specific knowledge for `code`, `method`, `protocol`, `entrypoint`, `router`, `service` labels; log attributes require Traefik-specific knowledge |

→ Level 2 not met — deprecated attributes present, cross-signal inconsistency, proprietary log and Prometheus metric attributes.

---

#### Rationale

Traefik is assigned **Level 1 — OpenTelemetry-Aligned** because:

1. **Traefik's own spans use current OTel semconv** (117 spans from `github.com/traefik/traefik` scope) with correct attributes like `http.request.method`, `http.response.status_code`, `url.path`, `url.scheme`, `network.peer.address`, `client.address`, `user_agent.original`. This represents clear progress beyond Level 0.

2. **OTel-native metrics exist** — `http.server.request.duration` and `http.client.request.duration` are emitted with fully OTel-compliant attribute sets including `error.type`.

3. **Schema URLs are present** on all three signals (traces/metrics/logs reference `https://opentelemetry.io/schemas/1.40.0`), demonstrating governance intent.

4. **However, consistency is incomplete across all signals:**
   - The Prometheus-scraped `traefik_*` metrics (the primary operational metrics) use proprietary label names (`code`, `method`, `protocol`, `entrypoint`, `router`, `service`) instead of OTel semconv names, requiring Traefik-specific knowledge to interpret.
   - Log attributes use PascalCase proprietary naming (`RequestMethod`, `DownstreamStatus`, `ClientAddr`, `RouterName`) with no OTel alignment, and log bodies are stringified JSON blobs rather than structured OTel attributes.
   - 24 spans from the backend's `@opentelemetry/instrumentation-http` (part of the same distributed traces) carry the full set of deprecated HTTP attributes (`http.method`, `http.status_code`, `http.target`, `http.flavor`, `http.user_agent`, `net.peer.*`, `net.host.*`).
   - The same concept is named differently across signals (HTTP method: `http.request.method` vs `method` vs `RequestMethod`).

5. **The dual metric system** (OTel-native `http.server.request.duration` alongside Prometheus-scraped `traefik_entrypoint_requests_total`) means users cannot rely on a single, consistent OTel-aligned view — off-the-shelf OTel dashboards would work for the OTel metrics but not for the primary Traefik operational metrics.

To reach **Level 2**, Traefik would need to: (a) migrate `traefik_*` Prometheus metric labels to OTel semconv attribute names, (b) restructure access logs to emit structured OTel log attributes using OTel semconv names instead of PascalCase proprietary attributes, and (c) ensure no deprecated HTTP attributes appear anywhere in the trace data.
