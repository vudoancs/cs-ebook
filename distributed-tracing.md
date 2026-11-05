# Distributed tracing & correlation ID

How to follow one operation across `kygo-erp-api` (NestJS, `service.name=kygo-api`) and `kygo-ai-service` (FastAPI, `service.name=kygo-ai-service`), and how to find where time went.

**Source of truth**

| Concern | File |
|---|---|
| API tracing bootstrap, sampling, trust policy | `kygo-erp-api/src/observability/tracing.ts`, `trace-context-policy.ts` |
| API correlation context / middleware | `kygo-erp-api/src/observability/correlation-context.ts`, `src/common/middlewares/request-id.middleware.ts` |
| API → Python headers | `kygo-erp-api/src/observability/correlation-headers.ts` |
| Bull / cron propagation | `kygo-erp-api/src/observability/job-telemetry.ts` |
| API JSON logs | `kygo-erp-api/src/observability/json-logger.service.ts`, `log-context.ts` |
| Python tracing / middleware / logs | `kygo-ai-service/app/observability/*`, `app/utils/logging.py` |
| Collector / Tempo / Grafana | `monitoring/otel-collector/config.yaml`, `monitoring/tempo/tempo.yaml`, `monitoring/grafana/provisioning/datasources/` |

---

## 1. Identifiers

| ID | Owner | Meaning | Lifecycle |
|---|---|---|---|
| `correlation_id` | Application | One **logical operation**: an HTTP request, a Bull job (all its attempts), one cron run | Created at the API edge (or per job/cron run). Same value across retries of that operation. New value for unrelated operations. **Not** an idempotency key — existing `jobId` (`telegram-alert:<alertId>`), order/product IDs stay as they are. |
| `trace_id` / `span_id` | OpenTelemetry | One causal call tree / one timed unit inside it | Never created by hand. W3C `traceparent` / `tracestate`. |
| `requestId` (legacy) | — | **Same value as `correlation_id`** | `req.requestId`, activity-log `requestId`, header `X-Request-ID` are kept as aliases. |

Accepted format: `^[A-Za-z0-9_-]{16,64}$` (covers UUIDs). Missing or invalid → a fresh UUID v4 replaces it (header-injection / oversized values never reach logs).

`correlation_id` is stored as span attribute `kygo.correlation_id` and as the log field `correlation_id`.

## 2. HTTP contract

**Browser / client → API**

- Request: optional `X-Correlation-ID` (fallback `X-Request-ID`).
- Response (every status, including errors and 404): `X-Correlation-ID` and `X-Request-ID` (same value). Exposed through CORS (`exposedHeaders`). Error JSON bodies from `HttpExceptionFilter` also contain `"correlationId"`. Unhandled non-HTTP exceptions return Nest's default 500 body (no `correlationId` field) but still carry the headers.
- Frontends do not show it yet; they can read `response.headers['x-correlation-id']` for a "support code".

**API → kygo-ai-service** (every call in `ai-embedding-client`, `ai-vision-client`, `ai-health-client`, `knowledge-ai-client`)

- `X-Correlation-ID`, `X-Request-ID` from the active AsyncLocalStorage context.
- `traceparent` (+ `tracestate`) — injected by the HTTP instrumentation (the client span becomes the parent of Python's server span). `buildCorrelationHeaders()` also injects the current context as a fallback; the instrumentation overwrites it with its own child span.
- Timeouts unchanged. **No automatic retries** on these calls (embedding/LLM calls are not idempotent and cost money).
- Image downloads from CDN / OAuth / Telegram do not get these headers.

**kygo-ai-service** extracts `traceparent` (server span = child of the API's HTTP client span), reads `X-Correlation-ID` with the same validation, and echoes both headers on every response — including unhandled 500s.

**Datastores** (MongoDB, Qdrant, Redis) never see `X-Correlation-ID`; they are instrumented at the client boundary.

### Trust policy for incoming trace context

| Hop | Extract incoming `traceparent`? | Why |
|---|---|---|
| Public → `kygo-erp-api` | **No** (default `OTEL_TRUST_INCOMING_TRACE_CONTEXT=false`) | Clients could force `sampled=01` on every request (cost/DoS) or join our traces to theirs. Each public request starts a root span; our sampler decides. Set `true` only behind an internal gateway that strips/sets the header. `X-Correlation-ID` is still accepted (validated). |
| `kygo-erp-api` → `kygo-ai-service` | **Yes** | Internal caller; `ParentBased` sampler follows the API's decision so traces are complete. |

⚠️ `vps/docker-compose.yml` still publishes `8000:8000` and the AI service has no authentication. Anyone reaching that port can also send a `traceparent` with `sampled=01`. Remove the public mapping (the API uses the Docker network) — tracked as manual follow-up.

## 3. Background jobs (Bull) and cron

Queues: `ai-reindex` (`reindex-all`), `knowledge-index` (`index-article`, `delete-article`, `reindex-all`), `telegram-alert` (`send-alert`).

**Producer** (`addJobWithTelemetry`) adds one optional field to the job data — business fields and `JobOptions` (attempts, backoff, `jobId`) are unchanged:

```json
{ "articleId": "…", "requestedBy": "…",
  "_telemetry": { "correlationId": "…", "traceparent": "00-…-…-01", "tracestate": "…" } }
```

It also records a `PRODUCER` span `<queue> publish` in the request's trace.

**Consumer** (`runJobWithTelemetry`), **per attempt**:

- New **root trace** (`CONSUMER` span `<queue> process`) with a **span link** to the producer span. Attributes: `messaging.system=bull`, `messaging.destination.name`, `messaging.message.id`, `kygo.job.name`, `kygo.job.attempt`, `kygo.job.max_attempts`, `kygo.job.will_retry` on failure.
- `correlation_id` = `_telemetry.correlationId` → identical for every attempt of the job.
- Own AsyncLocalStorage context; released when the handler resolves or throws (Bull then retries as before).
- Log lines `Job attempt completed` / `Job attempt failed` with `queue`, `jobId`, `attempt`, `duration_ms`, `status`, `error_type`.
- Jobs enqueued before this change (no `_telemetry`) still run; they get a new correlation ID per attempt and no link.

**Why a new trace per attempt:** every queue here uses backoff (5 s – 10 s exponential), and `reindex-all` runs for minutes. Keeping attempts inside the HTTP request's trace would create traces lasting minutes/hours that never "finish" in Tempo, and attempts are not children of the request (it already returned 202). The link keeps the causal relationship navigable; `correlation_id` finds all attempts.

**Cron** (`telegram` digests, `employee-awards-hourly`, `notifications-cleanup-expired`): each run gets a new correlation ID and root span `cron <name>`. The 30 s `socket-session-sweep` interval is intentionally not traced (noise). `@OnEvent` listeners run inside the emitter's context and inherit its IDs.

## 4. Logs

Both services write **one JSON object per line to stdout**, collected by Promtail → Loki (7 days).

| Field | API | AI service |
|---|---|---|
| `timestamp`, `level`, `message` | ✓ | ✓ |
| `service`, `environment` | `kygo-api`, `ENVIRONMENT` | `kygo-ai-service`, `ENVIRONMENT` |
| `correlation_id`, `trace_id`, `span_id` | from context | from context |
| `context` / `logger` | Nest logger context | Python logger name |
| `operation`, `route`, `status`, `status_code`, `duration_ms` | `Request completed` — exactly one per request from `RequestIdMiddleware` (`res` `close`, context bound with `AsyncResource.bind`): includes guard 401/403, unknown-route 404, errors, client aborts (`status: aborted`); job attempt logs | `Request completed` per request |
| business IDs | metadata object (`productId`, `jobId`, `alertId`, …) | `extra=` fields |
| `error_type`, `stack` | ✓ | ✓ |

- IDs are present even when the trace is **not sampled** and when the Collector is down (non-recording spans keep valid IDs) — log search by `correlation_id` always works; the trace link may 404 in Tempo. They are **absent** when the SDK is not running at all (see the startup line `Tracing enabled|disabled`, §6) and for `/metrics`, `/health*`, `/docs` (never traced by design).
- The `Request completed` line carries the **server span** ID; business log lines carry the **active child span** ID (e.g. `AuthController.login`, `attendance.compute`) — same `trace_id`.
- Redacted keys (value replaced with `[REDACTED]`): password, token, secret, authorization, cookie, otp, api key, credential, vector, embedding, image data, file/buffer; Python also `prompt`, `content`. Request bodies, headers, search text and prompts are never logged by this code. Existing log calls that put free text in the message itself are not rewritten — keep reviewing new log lines.
- Not covered: Nest's built-in `console.log` calls elsewhere in the codebase (plain text, no IDs).

## 5. Spans you will see

| Span | Service | Key attributes |
|---|---|---|
| `POST /api/v1/admin/ai/search/image` (server) | api | `http.*`, `http.route` (template, set by `MetricsInterceptor` via RPC metadata), `kygo.correlation_id` |
| `<Controller>.<handler>` / handler name | api | Nest instrumentation (no Express layer spans — Express instrumentation is not installed, see §12.1) |
| `attendance.team_stats` › `attendance.load_employees`, `attendance.load_inputs`, `attendance.compute` | api | `kygo.employee_count`, `kygo.month_count`, `kygo.days_processed`, `kygo.shift_count`, `kygo.attendance_count`, `kygo.db.query_count`, `kygo.result_count`, `kygo.cache` — counts only |
| `products.image_search` | api | `kygo.top_k`, `kygo.input.bytes`, `kygo.result_count` |
| `ai_service.search_similar_products` → `POST` (http client) | api | `kygo.timeout_ms` (120000), `kygo.result_count` |
| `POST /search/similar-products` (server) | ai | child of the API http client span |
| `embedding.generate` | ai | `embedding.kind`, `embedding.model`, `kygo.input.bytes`/`chars`, **`embedding.lock_wait_ms`**, **`embedding.inference_ms`** |
| `qdrant.search` / `qdrant.upsert` / `qdrant.delete` → `POST`/`PUT` (httpx) | ai | `db.collection.name`, `kygo.top_k`, `kygo.result_count`, `kygo.timeout_s` |
| `results.rerank` | api (dedupe by product) / ai (`/search/similar`) | `kygo.input.count`, `kygo.result_count` |
| `mongodb.find`, `mongodb.insert`, … | api | `db.collection.name`, no statement text |
| `knowledge.search`, `knowledge.answer` | api | `kygo.top_k`, `kygo.input.chars` |
| `rag.answer` → `rag.retrieve` → `embedding.generate`, `qdrant.search`; `llm.generate` → `POST` (httpx, one per SDK attempt) | ai | `gen_ai.request.model`, `gen_ai.usage.*_tokens`, `gen_ai.response.finish_reasons`, `kygo.timeout_s` |
| `<queue> publish` / `<queue> process` | api | see §3 |

## 6. Configuration

| Variable | Service | Default | Notes |
|---|---|---|---|
| `OTEL_SDK_DISABLED` | both | `false` | `true` = no tracing (logs keep correlation IDs). |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | both | API: **none — required to enable tracing**; AI: `http://localhost:4318` | OTLP/HTTP protobuf. Docker: `http://otel-collector:4318`. |
| `OTEL_TRACES_SAMPLER_ARG` | both | `1.0` | Ratio for **root** spans (`ParentBased(TraceIdRatioBased)`). |
| `OTEL_TRUST_INCOMING_TRACE_CONTEXT` | api | `false` | See trust policy. |
| `OTEL_SERVICE_NAME` | both | `kygo-api` / `kygo-ai-service` | |
| `ENVIRONMENT` | both | `local` | `deployment.environment.name` + log `environment`. |
| `SERVICE_VERSION` | both | package version / `1.3.0` | `service.version`. |
| `LOG_LEVEL` | api | `info` | |

Exporter behaviour: batch processor, queue 2048, batch 512, export timeout 10 s (API) / 5 s HTTP timeout. Collector down → spans dropped, a WARN is logged, requests are unaffected (unit-tested in both services, and checked live against a closed port). Shutdown flushes best-effort (`enableShutdownHooks` / FastAPI lifespan).

The API needs **Node ≥ 22.15** (OTel SDK 0.300); Docker image is `node:22-alpine`. The API enables tracing only when the endpoint is set, `OTEL_SDK_DISABLED` is not `true` and Node is new enough, and logs one line at startup:

```json
{"context":"Bootstrap","message":"Tracing enabled","tracing_enabled":true,"node_version":"22.21.0","sampler_ratio":0.2,"trust_incoming_trace_context":false}
{"context":"Bootstrap","message":"Tracing disabled","tracing_enabled":false,"tracing_disabled_reason":"no_endpoint|sdk_disabled|node_unsupported", ...}
```

OTel metrics/logs exporters are off — metrics stay on Prometheus `/metrics`. **Never** add `correlation_id`, `trace_id`, `span_id`, `user_id` or `job_id` as Prometheus labels.

## 7. Sampling & retention

One policy: **head sampling at the API edge**, propagated downstream (`ParentBased`). The Collector does not sample.

- Local / verification: `1.0` (every trace).
- Production (`vps/docker-compose.yml` default): `0.2`. Trade-off: 20 % of requests have a trace; the other 80 % are **not** recorded anywhere — they cannot be recovered later. Their logs still carry `correlation_id`/`trace_id`, so log search works but "Open trace" returns *trace not found*. Raise the ratio temporarily when investigating; lower it if Tempo disk/CPU grows. Errors are **not** preferentially kept (that needs tail sampling in the Collector — not configured).
- Bull attempts and cron runs are root spans → sampled independently with the same ratio (a sampled request may have an unsampled job attempt, and vice versa).
- Retention: Tempo 72 h (`block_retention`), Loki 7 d, Prometheus 15 d.

## 8. Observability stack

`monitoring/docker-compose.yml`: `otel-collector` (contrib 0.162.0) on `kygo_app` + `monitoring_internal`, **no host port**; `tempo` (2.10.8) on `monitoring_internal` only, volume `tempo_data`. Grafana (`127.0.0.1:3030`) has datasources Prometheus, Loki (derived field `trace_id` → Tempo) and Tempo (trace → logs). Promtail parses the JSON lines; only `service`, `container_id`, `level` are labels.

Deploy note: Grafana and Promtail read provisioning/config at start — after changing them run `docker compose up -d --force-recreate grafana promtail` on the VPS. The app containers must be on `kygo_app_network` (already true) and have the OTEL env from `vps/docker-compose.yml`.

## 9. Local verification (repeatable)

Requires Docker with ~3 GB free disk, local MongoDB + Redis for the API (**do not point a local API at the production Atlas cluster**).

```bash
# 1. Telemetry backend (Collector exposed on loopback only)
docker network create kygo_app_network 2>/dev/null || true
cd monitoring && docker compose -f docker-compose.yml -f docker-compose.local.yml \
  up -d otel-collector tempo loki promtail grafana

# 2. AI service with controlled faults (dev-only script, not in the image)
cd kygo-ai-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318 FIXTURE_EMBED_DELAY_MS=1500 \
FIXTURE_PRODUCT_ID=<an existing product _id in your local Mongo> \
  .venv/bin/python scripts/run_fault_fixture.py          # 127.0.0.1:8000

# 3. API against the fixture
cd kygo-erp-api
MONGO_URL=mongodb://localhost:27017/kygo REDIS_HOST=localhost \
AI_SERVICE_URL=http://127.0.0.1:8000 OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318 \
  npm run start:dev                                       # :6001 by default
```

Scenarios:

| # | Do | Expect |
|---|---|---|
| A — slow embedding | `curl -i -H "Authorization: Bearer $ADMIN_JWT" -F file=@dress.jpg http://localhost:6001/api/v1/admin/ai/search/image` | ~1.5 s; trace shows `embedding.generate` ≈ 1500 ms dominating `products.image_search` |
| B — failed downstream | `curl -i -H "Authorization: Bearer $JWT" -H 'content-type: application/json' -d '{"question":"Thuê váy tối đa mấy ngày?"}' http://localhost:6001/api/v1/knowledge/ask` | API error body with `correlationId`; Python `llm.generate` ERROR with 3 httpx child spans (SDK retries) |
| C — job retry | restart fixture with `FIXTURE_FAIL_UPSERT=true`, publish/re-index a knowledge article in admin | 3 `knowledge-index process` traces (attempt 1..3), same `kygo.correlation_id`, each linked to the publish span |
| D — Collector down | `docker stop kygo-otel-collector`, repeat A | Request still succeeds; WARN export logs only |

Automated checks: `cd kygo-erp-api && npx jest src/observability src/common/middlewares` · `cd kygo-ai-service && .venv/bin/pytest tests/test_tracing.py`.

## 10. Diagnosing a slow request — worked example

Complaint: "image search in admin took 4 s".

1. **Get the correlation ID.** Browser DevTools → Network → the `search/image` request → response header `x-correlation-id` (or the `correlationId` field of an error body). Say `7c9e…-…`.
2. **Logs across both services** — Grafana → Explore → Loki:
   ```
   {service=~"kygo-erp-api|kygo-ai-service"} |= "7c9e…" | json
   ```
   You get the API `Request completed` line (`duration_ms: 4012`), the Python `Request completed` line for `POST /search/similar-products` and any warnings — all with the same `trace_id`.
3. **Open the trace** — click the `trace_id` derived-field link, or Explore → Tempo → TraceQL `{ span.kygo.correlation_id = "7c9e…" }`. Job attempts appear as separate traces with the same correlation ID.
4. **Read the waterfall** (example numbers):
   ```
   POST /api/v1/admin/ai/search/image        4012 ms  (api, server)
   └ products.image_search                    3890 ms
     ├ ai_service.search_similar_products     3420 ms
     │ └ POST (http client)                   3415 ms
     │   └ POST /search/similar-products      3380 ms  (ai, server)
     │     ├ embedding.generate               3110 ms  lock_wait_ms=2400 inference_ms=700
     │     └ qdrant.search                     240 ms
     │       └ POST /collections/…/query       236 ms  (httpx)
     ├ results.rerank                            1 ms
     ├ mongodb.find products                   410 ms
     └ mongodb.insert (search history)          45 ms
   ```
   - **Inclusive vs self time.** A parent's duration includes its children. `products.image_search` (3890 ms) is not "slow itself"; subtract children to get self time (≈ 14 ms). Start from the deepest wide span.
   - **Sequential vs parallel.** Children that start after the previous one ends are sequential — their durations add up (here embedding → qdrant → rerank → Mongo). Overlapping children (e.g. `POST /vision/describe` and `/vision/classify` from `analyzeFromBuffer`, run with `Promise.all`) are parallel; the parent ≈ the longest child, not the sum.
   - **Network vs server.** API http client `POST` (3415 ms) − Python server span (3380 ms) ≈ 35 ms = network + (de)serialization + upload of the image.
   - **Embedding.** `lock_wait_ms` = waiting for the single in-process inference lock (other requests were embedding at the same time) vs `inference_ms` = model time. High wait → concurrency/capacity; high inference → CPU/model size.
   - **Qdrant.** `qdrant.search` − its httpx child = client-side work; the httpx span = network + Qdrant server time. Qdrant's own server time is not visible in the trace — compare with Qdrant metrics (dashboard `ai-qdrant`).
   - **MongoDB.** `mongodb.*` spans measure what the **client** observed: connection-pool checkout wait + network round-trip + server execution. The pool wait is not a separate span. A slow `find` alone does **not** prove a missing index or an overloaded server.
5. **Confirm with database evidence** before acting: run the same filter with `.explain("executionStats")` (look at `totalDocsExamined` vs `nReturned`, `COLLSCAN`), check the Atlas Profiler / slow query log for that time window (server-side `millis`), and check connection/CPU metrics (Atlas metrics, `mongodb_connection_state`, cAdvisor CPU for `kygo-erp-api`). If server `millis` is small but the span is large → pool wait or network; if `millis` is large with COLLSCAN → index; if many concurrent requests at that time → capacity.

## 11. Troubleshooting

| Symptom | Check |
|---|---|
| No traces at all | `OTEL_SDK_DISABLED` not `true`; endpoint reachable from the container (`docker exec kygo-erp-api wget -qO- http://otel-collector:13133` health — or check Collector logs); app on `kygo_app_network`; sampling ratio not `0`. API: tracing is skipped under Jest. |
| Trace "not found" from a log link | Request was not sampled (expected at ratio < 1), or older than 72 h, or Collector/Tempo dropped it (check `kygo-otel-collector` logs for queue full / export errors). |
| API and Python spans in **different** traces | Python not receiving `traceparent`: confirm the call goes through `buildCorrelationHeaders()`; a custom axios instance without the http module patched; Python started without `instrument_app` (`OTEL_SDK_DISABLED=true` on Python only). |
| Python spans exist but API spans missing | API sampled out while Python sampled? Not possible with ParentBased unless Python got no `traceparent` — see previous row. |
| Same request, different `correlation_id` in Python | Header dropped by a proxy, or value invalid (Python generated a new one) — compare `x-correlation-id` on both responses. |
| Logs without `trace_id` | First: `{service="kygo-erp-api"} |= "Tracing "` — `Tracing disabled` + reason means no span exists anywhere (fix the env / Node version). `/metrics`, `/health*`, `/docs` are never traced. Otherwise: log written outside a request/job context (startup, intervals) — expected. |
| Many `mongodb.*` spans per request | N+1 queries — fix in code; the trace is showing real round trips. |
| Bull attempts not linked | Job enqueued before the change (no `_telemetry`), or enqueued with `queue.add` directly instead of `addJobWithTelemetry`. |
| Loki "max streams" errors | Someone added a high-cardinality label in Promtail — IDs must stay line content. |

## 12. Incident 2026-10-23 — missing trace IDs, slow attendance stats, `ApiHighP95Latency`

Production evidence: `ApiHighP95Latency` at 1.563 s; 22 slow `Request completed` lines, all with `correlation_id` and **none** with `trace_id`/`span_id`: 17× `GET /metrics` (1384–1685 ms, ~every 45 s), 2× `GET /api/v1/my-payroll/team-attendance-stats` (8673 / 8719 ms; cids `16dd8408-bd28-4108-bdee-5ed64e38645e`, `74c6a847-9208-44c1-8916-0bd502c62b6f`), 2× `/products/rental-status` (~1.05 s), 1× `/products` (1.1 s). These are filtered slow lines, not traffic ratios.

### 12.1 Missing `trace_id` / `span_id`

| Failure mode | Result of the investigation |
|---|---|
| A span exists but log context is lost | **Not the cause.** Same build run locally with tracing on (Node 23.7 and Node 22.14): `Request completed` and business lines carry the server span / child span IDs for 200, 400, 401 (guard), 404, unsampled (ratio 0), Collector unreachable, 20 concurrent requests. |
| IDs exist but the logger drops them | **Not the cause.** `currentLogContext()` → winston `enrichWithContext` keeps them (unit-tested). |
| Traces exist but export/storage fails | Would not remove IDs from logs (export is after the fact). |
| **No request span exists** | **Cause.** For business routes the IDs only disappear when the OTel SDK is not running. Since `4e02f04` (hotfix for the 500s below) the API starts the SDK only if `OTEL_EXPORTER_OTLP_ENDPOINT` is set, `OTEL_SDK_DISABLED≠true` and Node ≥ 22.15. Which condition is false on the VPS is **not verified** (no production access) — check with §12.6. `/metrics` lines have no IDs by design (ignored by the HTTP instrumentation). |

Likely candidates on the VPS (unverified): `/root/docker-compose.yml` without the OTEL variables from `vps/docker-compose.yml`; `OTEL_SDK_DISABLED=true` set during the 500 incident; or a cached `node:22-alpine` base image older than 22.15 (`docker compose up --build` does not pull a newer base).

**The 500 `stream is not readable` incident (why tracing was gated).** Confirmed locally by dumping the Express router stack: `@opentelemetry/instrumentation-express` replaces every `layer.handle` with a function named `patched`; Nest decides whether to add its default body parsers with `isMiddlewareApplied('jsonParser')`, which compares `layer.handle.name` — so with the instrumentation Nest registered a **second** json/urlencoded pair after `main.ts`'s 4 mb pair (tracing off: one pair). The exact production trigger of the 500 was **not reproduced** locally (JSON, urlencoded, multipart, gzip, chunked and >100 kb bodies all returned 4xx as expected); the same symptom with this instrumentation is reported in [getsentry/sentry-javascript#17131](https://github.com/getsentry/sentry-javascript/issues/17131). Fix (`kygo-erp-api` e121aae): Express instrumentation removed (Nest instrumentation already gives handler spans; `http.route` is set via RPC metadata), `NestFactory.create(..., { bodyParser: false })` so exactly one parser pair exists regardless of instrumentation. Regression tests: `src/observability/tracing.spec.ts`.

Other changes: startup line `Tracing enabled|disabled` (§6); `Request completed` moved to `RequestIdMiddleware` so 401/403/404/aborted requests are logged too. Binding the `close` listener matters for client aborts (verified: without it the aborted request's line lost its correlation/trace IDs); for normal completion Node already emits `close` inside the request context.

### 12.2 `GET /api/v1/my-payroll/team-attendance-stats`

Path: `MyPayrollController.getTeamAttendanceStats` → `ScheduleAuthService.ensureScheduleManager` (in-memory role check) → `AttendanceCalculationService.getTeamAttendanceStats`.

**Bottleneck (confirmed with traces on a local dataset):** the 12-month trend ran **months sequentially × employees sequentially × 3 queries** (`settings.findOne`, `shifts.find`, `attendances.find`), plus the current month again. 40 employees ⇒ **1 586 MongoDB commands** per request (13 × 2 + 40 × 13 × 3), every one a round trip, and each per-employee query was a `COLLSCAN` (`shifts`/`attendances` have no `{employee, date}` index in the schema). Duration therefore scales with *employees × months × round-trip time*.

**Fix (`kygo-erp-api` 7d4a945):** 5 queries for all 12 months (employees, settings, shop holidays over the union window, shifts and attendances with `employee $in` + date range + projection + `batchSize`), grouped in memory by employee × UTC month, then the unchanged `calculateAttendance` with preloaded data. Same payroll semantics, authorization, UTC month boundaries and ±1 day VN holiday window. No cache. Output is byte-identical to the old implementation (benchmark dataset) and equal to the per-employee path in `attendance-calculation.team-stats.spec.ts` (both target bases, joiners/leavers, missing join date, disabled employees/shifts/attendance, paid leave, early leave, boundary holidays).

**Measurements** — `src/scripts/bench-team-attendance-stats.ts`, local MongoDB 7, synthetic data (13 months, ~26 shifts/employee/month), same dataset and parameters, 1 cold + 3–5 warm runs per process; `latency` = local TCP proxy delay each way (simulated remote DB):

| Employees | Latency | Before (warm median) | After (warm median) | MongoDB commands before → after |
|---|---|---|---|---|
| 40 | 0 ms | 2 093 ms | 477 ms | 1 586 → 9 |
| 40 | 2 ms | 9 023 ms | 492 ms | 1 586 → 9 |
| 80 | 2 ms | 17 695 ms | 999 ms | 3 146 → 13 |

Cold (first) runs were 0–30 % slower than warm (before: 2 735 / 8 836 / 17 755 ms; after: 557 / 590 / 1 096 ms). After the fix the remaining time is CPU in `attendance.compute` (~390 ms for 480 employee-months, mostly `moment` string parsing in shared payroll utils — next optimisation candidate, not changed); the loop yields to the event loop after each month. **These are local numbers, not production measurements.** Production RTT to Atlas, real collection sizes and real indexes are unknown.

**Indexes:** with the batched query (12 months) a `{employee: 1, date: 1}` index made no measurable difference locally (the query returns ~90 % of the synthetic collection). Production collections hold more history, so check before deciding — do **not** add `Schema.index` (Mongoose `autoIndex` would build it on deploy):

```js
// mongosh, read-only checks
db.shifts.getIndexes(); db.attendances.getIndexes();
db.shifts.find({ employee: { $in: [/* a few ObjectIds */] }, date: { $gte: ISODate("2025-11-01"), $lt: ISODate("2026-11-01") }, enable: { $ne: false } }).explain("executionStats")
// If COLLSCAN with totalDocsExamined >> nReturned: create in a quiet hour, one collection at a time, watching Atlas metrics
db.shifts.createIndex({ employee: 1, date: 1 }, { name: "employee_1_date_1" })
db.attendances.createIndex({ employee: 1, date: 1 }, { name: "employee_1_date_1" })
// Rollback
db.shifts.dropIndex("employee_1_date_1"); db.attendances.dropIndex("employee_1_date_1")
```

### 12.3 `/metrics` and `ApiHighP95Latency`

- Histogram: `http_request_duration_seconds` (seconds), labels `method`, `route` (`normalizeRoute(req.path)`: ObjectIds/UUIDs/numbers → `:id`), `status_code`; buckets `0.005 … 0.5, 1, 2.5, 5`. Recorded by `MetricsInterceptor` for every routed request **including `/metrics`, `/ping`, `/health*`** (route values `/metrics`, `/ping`, `/health`, `/healthcheck`, `/health/password-reset`). Requests rejected by guards and unknown routes are not recorded (interceptors do not run).
- Old rule: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="kygo-erp-api"}[5m])) by (le)) > 1`, `for: 5m` — **all routes aggregated**, so `/metrics` was part of "business" P95.
- Why `/metrics` took ~1.5 s: it called `PasswordResetReadinessService.check()` on every scrape — SMTP `verify` (TLS + AUTH), then Mongo `hello`, then Redis `PING`, sequentially, cached 30 s from *completion*. With a 15 s scrape the cache missed every third scrape ⇒ one ~1.4–1.7 s scrape every ~45 s, exactly the production pattern. Reproduced locally with a fake SMTP server that answers after 1.4 s: 1.42 s on each miss, ~5 ms otherwise.
- **Did `/metrics` cause the alert?** Not proven — request volume for the window is not available here. Arithmetic: with buckets `(1, 2.5]` the reported 1.563 s means ~8 % of the window's observations were in that bucket (`1 + 1.5·(c − 0.05N)/c = 1.563` ⇒ `c ≈ 0.08N`). The `/metrics` slow scrapes alone are ~6–7 per 5 min, so they would explain the value if the API served roughly ≤ 85 requests per 5 min in that window; the 8.7 s attendance requests fall in the `+Inf` bucket and cannot by themselves produce 1.563. Run queries 4–5 in §12.4 for the incident window to confirm.
- Fixes:
  - `kygo-erp-api` a88a5c5: readiness refreshed in the background every 60 s (parallel probes, single in-flight check); `/metrics` reads the snapshot (no I/O). New gauge `password_reset_readiness_checked_timestamp_seconds`. Local: ≤ 10 ms per scrape with the 1.4 s fake SMTP.
  - `monitoring` 7f63d71: `ApiHighP95Latency` now uses `route!~"/metrics|/ping|/health|/healthcheck|/health/password-reset"` (threshold and `for` unchanged); Overview latency panel uses the same filter. `/metrics` is watched separately: `ApiMetricsScrapeSlow` (`max_over_time(scrape_duration_seconds{job="kygo-erp-api"}[10m]) > 1` for 10m), `PasswordResetReadinessStale`, and a "Monitoring endpoints" row on the NestJS API dashboard. Rules were YAML-validated only (no `promtool` available).
- Bucket resolution: between 1 s and 2.5 s the quantile is linear interpolation — "1.563 s" is not a measured latency. Adding `1.5`/`2` buckets would sharpen it (changes series; not done).

### 12.4 Grafana queries (copy-paste)

Loki stream labels: `service` (container name: `kygo-erp-api`, `kygo-ai-service`), `container_id`, `level`. The JSON field `service` (`kygo-api`) becomes `service_extracted` after `| json` — filter on the stream label, not the JSON field.

1. **Slow business requests (Loki)** — excluding monitoring routes:
   ```logql
   {service="kygo-erp-api"} |= "Request completed" | json
     | route !~ "/metrics|/ping|/health|/healthcheck|/health/password-reset"
     | duration_ms > 1000
     | line_format "{{.duration_ms}}ms {{.status_code}} {{.operation}} cid={{.correlation_id}} trace={{.trace_id}}"
   ```
2. **All logs for one correlation ID** (both services, jobs included):
   ```logql
   {service=~"kygo-erp-api|kygo-ai-service"} |= "16dd8408-bd28-4108-bdee-5ed64e38645e" | json
   ```
3. **Trace lookup (Tempo, TraceQL)** — click the `trace_id` derived field in Loki, or:
   ```traceql
   { span.kygo.correlation_id = "16dd8408-bd28-4108-bdee-5ed64e38645e" }
   { resource.service.name = "kygo-api" && span.http.route = "/api/v1/my-payroll/team-attendance-stats" && duration > 1s }
   { name = "attendance.team_stats" } | select(span.kygo.employee_count, span.kygo.db.query_count)
   ```
4. **P95 by route (Prometheus)**:
   ```promql
   histogram_quantile(0.95, sum by (le, route) (rate(http_request_duration_seconds_bucket{job="kygo-erp-api", route!~"/metrics|/ping|/health|/healthcheck|/health/password-reset"}[5m])))
   ```
   Requests slower than 1 s per route in the window: `sum by (route) (increase(http_request_duration_seconds_count{job="kygo-erp-api"}[5m])) - sum by (route) (increase(http_request_duration_seconds_bucket{job="kygo-erp-api", le="1"}[5m]))`
5. **Request volume for the same window**:
   ```promql
   sum by (route) (increase(http_requests_total{job="kygo-erp-api"}[5m]))
   sum(increase(http_requests_total{job="kygo-erp-api", route!~"/metrics|/ping|/health|/healthcheck|/health/password-reset"}[5m]))
   ```
   Set the time range to 2026-10-23 09:35–10:05 (Asia/Ho_Chi_Minh) and compare the old and new alert expressions side by side.
6. **`/metrics` scrape performance**:
   ```promql
   scrape_duration_seconds{job="kygo-erp-api"}
   max_over_time(scrape_duration_seconds{job="kygo-erp-api"}[10m])
   up{job="kygo-erp-api"}
   histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job="kygo-erp-api", route="/metrics"}[5m])))
   time() - password_reset_readiness_checked_timestamp_seconds{job="kygo-erp-api"}
   ```

### 12.5 Deploy and rollback

Pushing `main` deploys automatically (API: `docker compose -f /root/docker-compose.yml up -d --build`; monitoring: rsync + `docker compose up -d`). Order:

1. **VPS env** — copy the OTEL block of `vps/docker-compose.yml` into `/root/docker-compose.yml` if missing, and make sure `OTEL_SDK_DISABLED` is not `true` there or in `kygo-erp-api/.env`.
2. **API** — push `kygo-erp-api` (`e121aae`, `7d4a945`, `a88a5c5`, test commit). To be sure the base image is ≥ 22.15: `docker compose -f /root/docker-compose.yml build --pull kygo-erp-api && docker compose -f /root/docker-compose.yml up -d kygo-erp-api`.
3. **Monitoring** — push `monitoring` (`7f63d71`). The workflow only force-recreates Alertmanager, so on the VPS: `cd /root/monitoring && docker compose up -d --force-recreate grafana promtail && docker exec kygo-prometheus wget -qO- --post-data='' http://localhost:9090/-/reload`. (The production export still contained `GET /metrics` completion lines, which the current Promtail config drops — Promtail was probably never recreated after `a4cee43`.)
4. Verify with §12.6.

Rollback: `git revert` the commits above and push (each is self-contained; the attendance change has no data migration; the alert change is config only). If a `{employee, date}` index was created, drop it as shown in §12.2. Emergency switch for tracing without redeploying code: `OTEL_SDK_DISABLED=true` + recreate the container (logs keep `correlation_id`, lose trace IDs).

### 12.6 Remaining production verification (not done — no access)

- `docker exec kygo-erp-api node -v` (≥ 22.15) and `docker logs kygo-erp-api 2>&1 | grep '"Tracing '` → must be `Tracing enabled`.
- `docker exec kygo-erp-api printenv OTEL_SDK_DISABLED OTEL_EXPORTER_OTLP_ENDPOINT` (names only matter; do not paste other env values).
- A JSON POST (e.g. login with wrong password) returns 4xx, not 500 — confirms the body-parser fix in production.
- Loki query 1 shows `trace_id` on business lines; the derived link opens the trace in Tempo (needs ratio > 0 and a sampled request; raise `OTEL_TRACES_SAMPLER_ARG` to 1 temporarily).
- Trace of `GET /api/v1/my-payroll/team-attendance-stats` shows `attendance.*` spans with `kygo.db.query_count = 5` and its real duration on production data.
- Queries 4–5 for 2026-10-23 09:35–10:05 to settle how much `/metrics` contributed; after deploy `scrape_duration_seconds{job="kygo-erp-api"}` should stay well under 1 s.
- `getIndexes()` / `explain` on `shifts` and `attendances` (§12.2) before deciding on an index.
- `/products/rental-status` and `/products` (~1.05–1.1 s) were not investigated; use query 1 + their traces once IDs are back.
