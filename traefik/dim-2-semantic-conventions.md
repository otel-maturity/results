### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

All deprecated attributes originate exclusively from the **`@opentelemetry/instrumentation-http`** scope (the Node.js backend auto-instrumentation library), not from Traefik's own instrumentation. Each deprecated attribute appears on exactly 24 spans:

| Deprecated Attribute | Count | OTel Replacement |
|---|---|---|
| `http.method` | 24 | `http.request.method` |
| `http.status_code` | 24 | `http.response.status_code` |
| `http.url` | 24 | `url.full` |
| `http.target` | 24 | `url.path` + `url.query` |
| `http.host` | 24 | `server.address` + `server.port` |
| `http.scheme` | 24 | `url.scheme` |
| `http.flavor` | 24 | `network.protocol.version` |
| `http.user_agent` | 24 | `user_agent.original` |
| `net.host.name` | 24 | `server.address` |
| `net.host.port` | 24 | `server.port` |
| `net.peer.port` | 24 | `network.peer.port` |
| `net.host.ip` | present | `network.local.address` |
| `net.peer.ip` | present | `network.peer.address` |
| `net.transport` | present | `network.transport` |

Additional non-standard attribute found on spans:
- `http.status_text` (e.g., `"OK"`, `"NOT FOUND"`) — not part of OTel semantic conventions, emitted by the Node.js backend instrumentation

##### Current OTel attributes found on spans

Traefik's own instrumentation scope (`github.com/traefik/traefik`) exclusively uses **current, stable** OTel semantic conventions:

| Attribute | Example Value | OTel Semconv |
|---|---|---|
| `http.request.method` | `GET` | ✅ Current |
| `http.response.status_code` | `200`, `404` | ✅ Current |
| `url.full` | `http://10.244.0.6:3000/` | ✅ Current |
| `url.path` | `/` | ✅ Current |
| `url.query` | present | ✅ Current |
| `url.scheme` | `http` | ✅ Current |
| `server.address` | `10.244.0.6` | ✅ Current |
| `server.port` | `3000` | ✅ Current |
| `client.address` | `127.0.0.1` | ✅ Current |
| `client.port` | `34076` | ✅ Current |
| `network.peer.address` | `10.244.0.6` | ✅ Current |
| `network.peer.port` | `3000` | ✅ Current |
| `network.protocol.version` | `1.1` | ✅ Current |
| `user_agent.original` | `curl/8.18.0` | ✅ Current |
| `http.request.body.size` | numeric | ✅ Current |

Traefik-specific (domain) attributes on spans:
- `entry_point` — Traefik entrypoint name (e.g., `web`, `traefik`, `metrics`); not in OTel semconv, not documented as an extension

##### Metric names and conventions

The metric corpus contains three distinct categories:

**1. OTel-compliant metrics (current semconv names):**
- `http.server.request.duration` — ✅ Matches OTel HTTP server semconv exactly; attributes use current keys (`http.request.method`, `http.response.status_code`, `url.scheme`, `server.address`, `network.protocol.version`, `error.type`)
- `http.client.request.duration` — ✅ Matches OTel HTTP client semconv exactly; attributes use current keys

**2. Traefik-proprietary metrics (non-OTel naming, non-OTel attribute keys):**
- `traefik_entrypoint_request_duration_seconds` — Prometheus-style naming; uses `code`, `method`, `protocol`, `entrypoint` (not OTel semconv attribute names)
- `traefik_entrypoint_requests_total`, `traefik_entrypoint_requests_bytes_total`, `traefik_entrypoint_responses_bytes_total`
- `traefik_router_request_duration_seconds`, `traefik_router_requests_total`, `traefik_router_requests_bytes_total`, `traefik_router_responses_bytes_total`
- `traefik_service_request_duration_seconds`, `traefik_service_requests_total`, `traefik_service_requests_bytes_total`, `traefik_service_responses_bytes_total`
- `traefik_open_connections`, `traefik_config_reloads_total`, `traefik_config_last_reload_success`

Proprietary metric attribute keys (not OTel semconv):
- `code` (should be `http.response.status_code`)
- `method` (should be `http.request.method`)
- `protocol` (should be `network.protocol.name`)
- `entrypoint`, `router`, `service` — Traefik-domain labels, no OTel equivalent

**3. Infrastructure metrics (from OTel collector scrapers):**
- `go_*`, `process_*` — Prometheus-format Go runtime metrics (not OTel semconv)
- `k8s.*` — OTel Kubernetes semantic conventions ✅
- `scrape_*` — Prometheus scrape metadata (not OTel semconv)

##### Log attributes

Log records use **Traefik-proprietary PascalCase attribute names** — not aligned with OTel semantic conventions:

| Log Attribute | Semantic Meaning | OTel Equivalent |
|---|---|---|
| `RequestMethod` | HTTP method | `http.request.method` |
| `DownstreamStatus` | Response status code | `http.response.status_code` |
| `RequestPath` | URL path | `url.path` |
| `RequestScheme` | URL scheme | `url.scheme` |
| `RequestHost` | Server address | `server.address` |
| `ClientAddr` | Client address+port | `client.address` + `client.port` |
| `ClientHost` | Client address | `client.address` |
| `ClientPort` | Client port | `client.port` |
| `Duration` | Request duration (ns) | — (no direct OTel log equiv.) |
| `ServiceAddr` | Backend address | `server.address` |
| `RouterName` | Traefik router | — (domain-specific) |
| `ServiceName` | Traefik service | — (domain-specific) |
| `entryPointName` | Traefik entrypoint | — (domain-specific) |
| `TraceId` / `trace_id` | Trace ID (duplicated!) | OTel `traceId` field |
| `SpanId` / `span_id` | Span ID (duplicated!) | OTel `spanId` field |

Notable issues:
- Log body is a **JSON-encoded string blob** rather than structured attributes — the entire access log is serialized as a single string value, with attributes also extracted as top-level keys
- `TraceId`/`SpanId` are duplicated (both PascalCase and snake_case variants present)
- Severity is `"info"` for all records but the embedded JSON shows `"level": "panic"` — a mismatch
- Log attribute names use inconsistent casing (`entryPointName` vs `ClientAddr` vs `trace_id`)

##### Cross-signal consistency

Shared attribute keys between traces and metrics (✅ consistent naming):
- `http.request.method`, `http.response.status_code`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`

However, the **Traefik proprietary metrics** use completely different names for the same concepts:
- Traces: `http.request.method` → Traefik metrics: `method`
- Traces: `http.response.status_code` → Traefik metrics: `code`
- Traces: `network.protocol.version` → Traefik metrics: `protocol`

**No shared attribute keys between traces/metrics and logs.** Logs use proprietary PascalCase names (`RequestMethod`, `DownstreamStatus`) while traces use OTel semconv (`http.request.method`, `http.response.status_code`).

##### Schema URL

| Signal | Schema URL | Notes |
|---|---|---|
| Traces | `https://opentelemetry.io/schemas/1.40.0` | ✅ Present, current |
| Metrics | `https://opentelemetry.io/schemas/1.40.0` (Traefik) | ✅ Present for OTel metrics |
| Metrics | `https://opentelemetry.io/schemas/1.18.0` (collector) | ✅ Present, older version |
| Logs | `https://opentelemetry.io/schemas/1.40.0` | ✅ Present |

Schema URLs are present on all signals, indicating intent to follow OTel semconv governance.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|---|---|---|
| Are attribute names ad-hoc (e.g. `statusCode`, `resp_code`, `requestPath`)? | **Partially** | Traefik's own trace spans use proper OTel names; logs use PascalCase proprietary names (`RequestMethod`, `DownstreamStatus`); Traefik metrics use `code`, `method`, `protocol` |
| Are deprecated OTel attributes used (`http.method`, `http.status_code`, `http.target`)? | **Yes** | 11 deprecated attributes present (from `@opentelemetry/instrumentation-http` backend scope) |
| Is the same concept named differently across signals? | **Yes** | `http.request.method` (traces/OTel metrics) vs `method` (Traefik metrics) vs `RequestMethod` (logs) |
| Is semantic meaning encoded in span names rather than attributes? | **Partially** | Most span names are HTTP method + path (e.g., `GET`, `GET /health`), but attributes carry the full context |
| Do users need to consult source code to understand attribute meaning? | **Partially** | Traefik-domain attributes (`entry_point`, `router`, `service`) require Traefik knowledge |

> Level 0 is **not** the right assignment — OTel conventions are substantially adopted in Traefik's own spans and the two OTel-named metrics.

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|---|---|---|
| Are *some* OTel semantic conventions used (not zero)? | **Yes** | Traefik spans use current OTel HTTP semconv; `http.server.request.duration` and `http.client.request.duration` use correct OTel metric names and attribute keys |
| Are deprecated and current OTel attributes mixed (`http.status_code` AND `http.response.status_code`)? | **Yes** | Both present in the same trace dataset; deprecated from Node.js backend scope, current from Traefik scope |
| Are conventions applied to traces but not consistently to metrics/logs? | **Yes** | Traefik proprietary metrics (`traefik_*`) use `code`/`method`/`protocol`; logs use PascalCase non-OTel names |
| Are similar concepts named differently across signals? | **Yes** | `http.request.method` (traces) vs `method` (Traefik metrics) vs `RequestMethod` (logs) |
| Are attribute types inconsistent (HTTP status as string vs int)? | **Minor** | `http.response.status_code` is integer in traces/OTel metrics; `code` is string in Traefik metrics |

> Level 1 criteria are **substantially met**.

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|---|---|---|
| Are **current, stable** OTel HTTP attributes used (`http.request.method`, `http.response.status_code`, `url.path`, `url.full`)? | **Partially** | Yes for Traefik's own spans and OTel metrics; No for Traefik's proprietary metrics and logs |
| Are **all** deprecated attributes removed or gated? | **No** | 11 deprecated attributes present (from Node.js backend; but they co-exist in the same telemetry pipeline) |
| Are attribute names consistent across traces, metrics, and logs? | **No** | Three different naming schemes across the three signals |
| Are attributes placed in the correct scope (request metadata on spans, identity on resources)? | **Mostly** | Resource attributes correctly placed; `entry_point` on spans is reasonable |
| Can telemetry be interpreted using generic OTel knowledge without project-specific mapping? | **Partially** | Traefik spans yes; Traefik metrics no (`code`, `method`, `protocol`, `entrypoint`, `router`, `service` require Traefik knowledge) |

> Level 2 criteria are **not met** — the Traefik-proprietary metric family and logs prevent full OTel-native classification.

##### Level 3 — Semantic Extension & Stewardship

| Question | Answer | Evidence |
|---|---|---|
| Are domain-specific concepts modeled as explicit attributes (not overloaded span names)? | **Partially** | `entry_point`, `router`, `service` are explicit; but not documented as OTel extensions |
| Are custom attributes documented with name, type, and semantic meaning? | **No** | No evidence of formal OTel extension documentation for Traefik-domain attributes |
| Do custom attributes extend OTel conventions rather than replace them? | **No** | Traefik metrics replace OTel attribute names (`code` instead of `http.response.status_code`) |
| Are semantic changes versioned and reviewed? | **No** | No evidence of versioned semconv governance |
| If a first-class signal uses a proprietary schema where OTel semconv exists, is it explicitly documented as an extension? | **No** | Traefik proprietary metrics use non-OTel names without explicit extension documentation |

> Level 3 criteria are **not met**.

---

#### Rationale

Traefik is assigned **Level 1 — OpenTelemetry-Aligned** based on the following reasoning:

**What Traefik does well:** Traefik's own tracer (`github.com/traefik/traefik` scope) exclusively uses current, stable OTel HTTP semantic conventions on spans — `http.request.method`, `http.response.status_code`, `url.full`, `url.path`, `url.scheme`, `server.address`, `client.address`, `network.peer.*`, `user_agent.original`. The two OTel-named metrics (`http.server.request.duration`, `http.client.request.duration`) are correctly named and use correct OTel attribute keys. Schema URLs are present on all three signals, and resource attributes follow OTel k8s/process conventions. Traefik clearly has OTel awareness and has partially adopted current conventions.

**Why not Level 2:** Three significant inconsistencies prevent Level 2:

1. **Proprietary metric family:** The `traefik_*` metric family (13 metrics) uses Prometheus-style naming and non-OTel attribute keys (`code`, `method`, `protocol`, `entrypoint`, `router`, `service`) for concepts where OTel semconv exists. A user cannot apply generic OTel dashboards to these metrics without mapping.

2. **Log attributes are entirely proprietary:** All 36 log attributes use Traefik's own PascalCase naming (`RequestMethod`, `DownstreamStatus`, `ClientAddr`, etc.) — none align with OTel semantic conventions. The log body is a serialized JSON string rather than structured attributes, further reducing usability.

3. **Deprecated attributes co-exist in the pipeline:** While the deprecated attributes (`http.method`, `http.status_code`, `http.target`, `http.flavor`, `net.host.*`, `net.peer.*`) come from the Node.js backend's `@opentelemetry/instrumentation-http` (not Traefik itself), they are present in the same telemetry dataset. The evaluation must consider what a consumer receives, not just what Traefik emits directly.

**Cross-signal inconsistency** is the clearest Level 1 indicator: the concept of "HTTP method" has three different names across the three signals — `http.request.method` (traces), `method` (Traefik metrics), and `RequestMethod` (logs).
