### 2. Semantic Conventions

**Level: 1 — OpenTelemetry-Aligned**

#### Evidence

##### Deprecated attributes found on spans

Deprecated attributes are present exclusively on spans emitted by the **backend demo service** (`@opentelemetry/instrumentation-http` / `@opentelemetry/instrumentation-express` scopes), **not** on Traefik's own spans. These originate from an outdated Node.js OTel auto-instrumentation library bundled with the backend workload, not from Traefik itself:

| Deprecated Attribute | Count | Scope |
|---|---|---|
| `http.method` | 34 | `@opentelemetry/instrumentation-http` |
| `http.status_code` | 34 | `@opentelemetry/instrumentation-http` |
| `http.url` | 34 | `@opentelemetry/instrumentation-http` |
| `http.target` | 34 | `@opentelemetry/instrumentation-http` |
| `http.host` | 34 | `@opentelemetry/instrumentation-http` |
| `http.scheme` | 34 | `@opentelemetry/instrumentation-http` |
| `http.flavor` | 34 | `@opentelemetry/instrumentation-http` |
| `http.user_agent` | 34 | `@opentelemetry/instrumentation-http` |
| `http.status_text` | 34 | `@opentelemetry/instrumentation-http` |
| `http.client_ip` | 34 | `@opentelemetry/instrumentation-http` |
| `net.host.name` | 34 | `@opentelemetry/instrumentation-http` |
| `net.host.ip` | 34 | `@opentelemetry/instrumentation-http` |
| `net.host.port` | 34 | `@opentelemetry/instrumentation-http` |
| `net.peer.ip` | 34 | `@opentelemetry/instrumentation-http` |
| `net.peer.port` | 34 | `@opentelemetry/instrumentation-http` |

**Traefik's own spans (`github.com/traefik/traefik`) contain zero deprecated HTTP/net attributes.**

##### Current OTel attributes found on spans

Traefik's native spans (`github.com/traefik/traefik` scope) use current stable OTel semantic conventions:

| Attribute | Spans |
|---|---|
| `http.request.method` | `GET`, `ReverseProxy` |
| `http.response.status_code` | `GET`, `ReverseProxy` |
| `http.request.body.size` | `GET` (entrypoint) |
| `url.path` | `GET` (entrypoint) |
| `url.query` | `GET` (entrypoint) |
| `url.scheme` | `GET`, `ReverseProxy` |
| `url.full` | `ReverseProxy` |
| `network.protocol.version` | `GET`, `ReverseProxy` |
| `network.peer.address` | `GET`, `ReverseProxy` |
| `network.peer.port` | `GET`, `ReverseProxy` |
| `server.address` | `GET`, `ReverseProxy` |
| `server.port` | `ReverseProxy` |
| `client.address` | `GET` (entrypoint) |
| `client.port` | `GET` (entrypoint) |
| `user_agent.original` | `GET`, `ReverseProxy` |

Additionally, Traefik adds one custom attribute: `entry_point` (Traefik-specific, naming the entrypoint such as `web`, `traefik`, `metrics`). This is a domain concept with no OTel semconv equivalent, but it is not named in OTel style (it should ideally be `traefik.entry_point`).

Span event attributes (`exception.message`, `exception.stacktrace`, `exception.type`) follow OTel semantic conventions correctly.

##### Metric names and conventions

Traefik emits metrics from **two parallel systems simultaneously**, which is the primary semantic convention concern:

**OTel-native metrics (OTLP push, `github.com/traefik/traefik` scope) — current semconv:**
| Metric Name | Type | Attributes |
|---|---|---|
| `http.server.request.duration` | histogram | `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `url.scheme`, `error.type` |
| `http.client.request.duration` | histogram | `http.request.method`, `http.response.status_code`, `network.protocol.name`, `network.protocol.version`, `server.address`, `server.port`, `url.scheme`, `error.type` |

These two metrics use fully current OTel HTTP semconv attributes.

**Traefik-proprietary metrics (OTLP push + Prometheus scrape, `github.com/traefik/traefik` scope) — non-OTel naming:**
| Metric Name | Attributes Used |
|---|---|
| `traefik_entrypoint_requests_total` | `code`, `entrypoint`, `method`, `protocol` |
| `traefik_entrypoint_request_duration_seconds` | `code`, `entrypoint`, `method`, `protocol` |
| `traefik_entrypoint_requests_bytes_total` | `code`, `entrypoint`, `method`, `protocol` |
| `traefik_entrypoint_responses_bytes_total` | `entrypoint`, `protocol` |
| `traefik_router_requests_total` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_router_request_duration_seconds` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_router_requests_bytes_total` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_router_responses_bytes_total` | `code`, `method`, `protocol`, `router`, `service` |
| `traefik_service_requests_total` | `code`, `method`, `protocol`, `service` |
| `traefik_service_request_duration_seconds` | `code`, `method`, `protocol`, `service` |
| `traefik_service_requests_bytes_total` | `code`, `method`, `protocol`, `service` |
| `traefik_service_responses_bytes_total` | `code`, `method`, `protocol`, `service` |
| `traefik_config_reloads_total` | _(none)_ |
| `traefik_config_last_reload_success` | _(none)_ |
| `traefik_open_connections` | `entrypoint`, `protocol` |

The `traefik_*` metrics use Prometheus-style attribute names (`code`, `method`, `protocol`) instead of OTel semconv (`http.response.status_code`, `http.request.method`, `network.protocol.name`). These metrics cover entrypoint, router, and service breakdowns which have no direct OTel semconv equivalent, but the attribute naming is not OTel-aligned.

**Infrastructure metrics** (`k8s.container.*`, `k8s.deployment.*`, `k8s.pod.*`, etc.) from the `k8sclusterreceiver` scope follow OTel semconv correctly.

##### Log attributes

Log records (Traefik access logs via OTLP, `traefik` scope) use **PascalCase proprietary attribute names** with no alignment to OTel semantic conventions:

| Log Attribute | OTel Semconv Equivalent |
|---|---|
| `RequestMethod` | `http.request.method` |
| `RequestPath` | `url.path` |
| `RequestScheme` | `url.scheme` |
| `DownstreamStatus` | `http.response.status_code` |
| `ClientHost` | `client.address` |
| `ClientPort` | `client.port` |
| `Duration` | _(no direct equivalent; `http.server.request.duration` is a metric)_ |
| `ServiceAddr` | `server.address` |
| `ServiceURL` | `url.full` |
| `RouterName` | _(Traefik-specific, no semconv)_ |
| `entryPointName` | _(Traefik-specific, no semconv)_ |
| `KubernetesIngressName` | _(Traefik-specific, no semconv)_ |
| `TraceId` / `trace_id` | _(duplicated — both camelCase and snake_case present)_ |
| `SpanId` / `span_id` | _(duplicated — both camelCase and snake_case present)_ |

The log record fields `traceId` and `spanId` are correctly populated in the OTLP log record envelope (not just as attributes), enabling trace correlation. However, the attribute keys themselves are not OTel semconv.

The log body is a **JSON string** (not a structured object), which is a deviation from recommended OTel log body practices.

##### Cross-signal consistency

The same HTTP concept is named differently across signals:

| Concept | Traces (Traefik) | Metrics (`traefik_*`) | Metrics (`http.*`) | Logs |
|---|---|---|---|---|
| HTTP method | `http.request.method` ✅ | `method` ❌ | `http.request.method` ✅ | `RequestMethod` ❌ |
| HTTP status | `http.response.status_code` ✅ | `code` ❌ | `http.response.status_code` ✅ | `DownstreamStatus` ❌ |
| Client address | `client.address` ✅ | _(absent)_ | _(absent)_ | `ClientHost` ❌ |
| URL scheme | `url.scheme` ✅ | `protocol` ❌ | `url.scheme` ✅ | `RequestScheme` ❌ |
| Network protocol | `network.protocol.version` ✅ | `protocol` ❌ | `network.protocol.name/version` ✅ | `RequestProtocol` ❌ |

There is significant **intra-metric inconsistency**: `http.server.request.duration` and `http.client.request.duration` use current OTel semconv attributes, while all `traefik_*` metrics use Prometheus-style shorthand attributes for the same concepts.

##### Schema URL

| Signal | Schema URL |
|---|---|
| Traces | `https://opentelemetry.io/schemas/1.40.0` ✅ present |
| Metrics | `https://opentelemetry.io/schemas/1.40.0` (Traefik OTLP) + `https://opentelemetry.io/schemas/1.18.0` (k8s cluster receiver) ✅ present |
| Logs | `https://opentelemetry.io/schemas/1.40.0` ✅ present |

All three signals declare a schema URL, indicating intent toward semconv governance. The Traefik-emitted signals all reference schema 1.40.0, which is a recent version.

---

#### Checklist assessment

##### Level 0 — Instrumented

| Question | Answer | Evidence |
|---|---|---|
| Are attribute names ad-hoc (e.g. `statusCode`, `resp_code`, `requestPath`)? | **Partially** | Log attributes use PascalCase proprietary names (`RequestMethod`, `DownstreamStatus`, `ClientHost`); `traefik_*` metric attributes use shorthand (`code`, `method`). Traefik's own spans do NOT use ad-hoc names. |
| Are deprecated OTel attributes used (`http.method`, `http.status_code`, `http.target`)? | **Yes (but from backend, not Traefik)** | All deprecated attributes come from the backend's `@opentelemetry/instrumentation-http`, not from Traefik's instrumentation. |
| Is the same concept named differently across signals? | **Yes** | `http.request.method` (traces) vs `method` (metrics) vs `RequestMethod` (logs). |
| Is semantic meaning encoded in span names rather than attributes? | **No** | Traefik uses meaningful span names (`GET`, `ReverseProxy`) but also carries proper attributes. |
| Do users need to consult source code to understand attribute meaning? | **Partially** | Log attributes (`DownstreamStatus`, `OriginDuration`, `Overhead`) require Traefik-specific knowledge. |

##### Level 1 — OpenTelemetry-Aligned

| Question | Answer | Evidence |
|---|---|---|
| Are *some* OTel semantic conventions used (not zero)? | **Yes** | Traefik's spans and `http.server/client.request.duration` metrics use current OTel semconv fully. |
| Are deprecated and current OTel attributes mixed? | **No (within Traefik's own instrumentation)** | Traefik's own spans use only current attributes. Deprecated attributes appear only in the backend service spans. |
| Are conventions applied to traces but not consistently to metrics/logs? | **Yes** | Spans are fully OTel-aligned; `traefik_*` metrics use Prometheus-style attributes; logs use proprietary PascalCase names. |
| Are similar concepts named differently across signals? | **Yes** | HTTP method: `http.request.method` (traces) ≠ `method` (metrics) ≠ `RequestMethod` (logs). |
| Are attribute types inconsistent? | **Partially** | `TraceId`/`trace_id` and `SpanId`/`span_id` are duplicated in log attributes (both camelCase and snake_case present simultaneously). |

##### Level 2 — OpenTelemetry-Native

| Question | Answer | Evidence |
|---|---|---|
| Are **current, stable** OTel HTTP attributes used (`http.request.method`, `http.response.status_code`, `url.path`, `url.full`)? | **Partially** | Yes for spans and `http.*` metrics; No for `traefik_*` metrics and all log attributes. |
| Are **all** deprecated attributes removed? | **No** | Deprecated attributes present in traces (from backend instrumentation in the same trace pipeline). |
| Are attribute names consistent across traces, metrics, and logs? | **No** | Three different naming schemes for the same concepts across the three signals. |
| Are attributes placed in the correct scope? | **Mostly yes** | Traefik spans place request attributes on spans and identity on resources correctly. |
| Can telemetry be interpreted using generic OTel knowledge without project-specific mapping? | **No** | `traefik_*` metrics require Traefik-specific knowledge of `code`, `entrypoint`, `router`, `service`. Log attributes require knowledge of Traefik's access log format. |

##### Level 3 — Semantic Extension & Stewardship

| Question | Answer | Evidence |
|---|---|---|
| Are domain-specific concepts modeled as explicit attributes? | **Partially** | `entry_point`, `router`, `service`, `entrypoint` are present but not consistently named or documented as OTel extensions. |
| Are custom attributes documented with name, type, and semantic meaning? | **No** | No formal OTel extension schema or attribute registry for Traefik-specific attributes. |
| Do custom attributes extend OTel conventions rather than replace them? | **No** | `traefik_*` metrics use `code`/`method` *instead of* OTel semconv names, not in addition to them. |
| Are semantic changes versioned and reviewed? | **No evidence** | No published semantic convention extension document found. |
| If a first-class signal uses a proprietary schema, is it documented as an extension? | **No** | Access log attribute naming is not documented as an intentional OTel extension. |

---

#### Rationale

Traefik v3.7.0 is assigned **Level 1 — OpenTelemetry-Aligned** because OTel semantic conventions are partially and inconsistently adopted across signals.

**What earns Level 1 (above Level 0):**
- Traefik's own trace spans (`github.com/traefik/traefik` scope) use exclusively current, stable OTel HTTP semantic conventions: `http.request.method`, `http.response.status_code`, `url.path`, `url.full`, `url.scheme`, `url.query`, `network.protocol.version`, `network.peer.address`, `server.address`, `client.address`, `user_agent.original`. There are zero deprecated attributes in Traefik's own spans.
- The OTLP-pushed `http.server.request.duration` and `http.client.request.duration` metrics use fully current OTel semconv attributes.
- A schema URL (`https://opentelemetry.io/schemas/1.40.0`) is declared on all three signals.
- Span event attributes (`exception.*`) follow OTel semconv.

**Why it cannot reach Level 2:**
1. **Metric naming duality**: The `traefik_*` family of metrics (the primary operational metrics for entrypoints, routers, and services) use Prometheus-style shorthand attribute names (`code`, `method`, `protocol`) rather than OTel semconv names (`http.response.status_code`, `http.request.method`, `network.protocol.name`). The same concept is named differently within the same metrics scope in the same batch.
2. **Log attributes are entirely proprietary**: All access log attributes use PascalCase Traefik-internal names (`RequestMethod`, `DownstreamStatus`, `ClientHost`, `RouterName`, `OriginDuration`) with no alignment to OTel semantic conventions. An off-the-shelf OTel dashboard cannot interpret these without a custom mapping.
3. **Cross-signal inconsistency**: HTTP method is `http.request.method` in traces, `method` in `traefik_*` metrics, and `RequestMethod` in logs — three different names for the same concept.
4. **Deprecated attributes in the trace pipeline**: While these originate from the backend's Node.js instrumentation (not Traefik itself), they are present in the collected traces data and represent a real user experience issue for anyone consuming the full trace dataset.
5. **Custom attribute `entry_point`** is not namespaced (should be `traefik.entry_point` to avoid collision) and is not documented as an OTel extension.
