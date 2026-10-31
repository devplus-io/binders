---
id: 96d07743-956f-422e-a007-256f65336224
binder: binder-mvvle32a-1
folder: folder-mvw0u8o0
---
End-to-End OpenTelemetry Observability and Metrics

**Target Release:** [[V0.18]]
**Status:** Planned (P1)
**Scope:** `devplus-shared`, `devplus-backend`, `devplus-executor`, `devplus-governance-service`, `devplus-mcp`, `devplus-agent-service`, `devplus-platform`, `devplus-core-infra`, `terraform-devplus-tenant`, `terraform-helm-devplus-services`

---

## Overview & Problem Statement

DevPlus has a solid vendor-neutral OpenTelemetry foundation: an HA in-cluster OTel Collector (`devplus-core-infra/otel-collector.tf`) forwarding OTLP traces and metrics to Application Insights (`appi-devplus-<env>`), a shared Go bootstrapper (`devplus-shared/otel/otel.go`), and `otelmux`/`otelhttp`/`otelmongo` wiring in the primary Go services.

However, a deep audit of the current implementation revealed several critical bugs, broken trace boundaries, and coverage gaps that prevent end-to-end observability in production:
1. **`trace_id=` in `devplus-backend` HTTP logs is always empty (`""`):** `LoggingMiddleware` wraps `router` outside `mux.Router`, while `otelmux.Middleware("backend")` runs inside `mux.Router`. Because `r.WithContext(ctx)` inside `otelmux` passes a shallow copy of `*http.Request` to inner handlers, the outer `*http.Request` held by `LoggingMiddleware` retains the pre-`otelmux` context and logs `trace_id=` as empty on every request.
2. **~388,800 empty-polling spans/day per tenant from `executor-job-dispatcher`:** `JobDispatcher.dispatchOnce` runs every 2 seconds and unconditionally starts a root span `executor.dispatchOnce`, makes 1 K8s `ListJobs` HTTP call, and executes up to 7 MongoDB `findAndModify` queue polls — producing ~270 spans/minute per tenant even when all queues are completely idle.
3. **Broken trace continuity across MongoDB queues (`backend` → `executor`):** None of the MongoDB queue items (`DeploymentQueueItem`, `ModuleQueueItem`, `DiscoveryQueueItem`, `AdoptionJob`, `RepoAnalysisQueueItem`, `HealthQueueItem`, `CostSyncQueueItem`) persist the enqueueing request's `traceparent`/`tracestate`. Traces end at the MongoDB `insert` in `backend`, and `executor` starts an unlinked trace on its next poll tick.
4. **`devplus-mcp` has Helm OTel env vars wired, but zero OTel code:** `devplus-mcp.yaml` includes `devplus.otel-env`, but `devplus-mcp` never calls `otel.InitOTel` or wraps its HTTP server/clients in `otelhttp` (and `TestHelmChartEnvMatchesServiceReads` omitted `devplus-mcp.yaml`).
5. **`devplus-agent-service` (Python / FastAPI / LangGraph) is completely un-instrumented:** `agent-service` drops incoming `traceparent` headers from `backend`'s `AIProxyHandler` and makes dozens of unlinked callback requests to `backend` during AI/Architect runs.
6. **Lifecycle, sampling, and HTTP client gaps:**
   - `github-callback` does not trap `SIGINT`/`SIGTERM` or call `srv.Shutdown()`, so `defer otelShutdown(ctx)` never runs on pod termination.
   - `devplus-governance-service` initializes `database.NewDatabase` and `server.Register` before `otel.InitOTel`, and shuts down `otelShutdown` before `decisionLog.Close(flushCtx)` (LIFO `defer` ordering).
   - `auth-service.tf`, `github-callback.tf`, and `executor` child job specs omit `OTEL_TRACES_SAMPLER` and `OTEL_TRACES_SAMPLER_ARG` (violating the Phase 0 head-sampling contract in `otel-collector.tf`).
   - Outbound HTTP clients in `oidc_mint.go`, `auth-service/main.go`, `devplus-shared/gcpauth`, `executor/internal/ghapi`, and `executor/internal/osv` use bare `&http.Client{}` without `otelhttp.NewTransport`.
   - Zero custom domain metrics (`otel.Meter`) exist for queue depth/latency, OpenTofu execution phases, or Cedar evaluation, and `metric-alerts.tf` has no alerts on collector dropped spans or service 5xx rates.

---

## Detailed Design by Workstream

### Phase 1: Fix Existing Bugs & Lifecycle Wiring

1. **`devplus-backend` — Fix `LoggingMiddleware` `trace_id` Correlation**
   - **Files:** `handlers/middleware.go`, `internal/app/app.go`, `internal/server/routes.go`
   - Either move `LoggingMiddleware` inside `r.Use(...)` *after* `r.Use(otelmux.Middleware("backend"))` (and keep a lightweight fallback for `NotFoundHandler`), or have `statusRecorder` capture the active `trace.SpanContext` from the inner request context before returning.
   - Add a unit test in `handlers/middleware_logging_test.go` asserting that when a request traverses `otelmux.Middleware` + `LoggingMiddleware`, the logged `trace_id=` is non-empty and matches the span's `TraceID`.

2. **`devplus-executor` — Eliminate Empty-Poll Span Flood in `JobDispatcher`**
   - **File:** `services/job_dispatcher.go`, `services/health_dispatch.go`, `services/cost_sync_dispatch.go`
   - Remove the unconditional `executor.dispatchOnce` root span on every 2-second tick.
   - Poll queues using a non-recording/suppressed span context (or only start `executor.dispatchJob` *after* `Dequeue` returns a non-nil queue item) so idle ticks emit zero trace spans while still recording dequeue/poll metrics.
   - Forward `OTEL_TRACES_SAMPLER` and `OTEL_TRACES_SAMPLER_ARG` alongside `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`, `OTEL_SERVICE_NAME`, and `OTEL_RESOURCE_ATTRIBUTES` in `buildJobSpec`, `buildDiscoveryJobSpec`, `buildRepoAnalysisJobSpec`, `health_dispatch.go`, and `cost_sync_dispatch.go`.

3. **`devplus-mcp` — Wire `InitOTel`, `otelhttp`, and Contract Test**
   - **Files:** `devplus-mcp/main.go`, `devplus-mcp/internal/service/service.go`, `devplus-mcp/internal/service/verify.go`, `devplus-platform/tests/contract/static_config_test.go`
   - Call `otel.InitOTel(ctx, "devplus-mcp")` in `devplus-mcp/main.go` and flush on `SIGINT`/`SIGTERM` shutdown.
   - Wrap the MCP HTTP handler in `otelhttp.NewHandler` and outbound `devplus-sdk-go` / JWKS `*http.Client` instances in `otelhttp.NewTransport`.
   - Add `"devplus-mcp.yaml": "devplus-mcp"` to `TestHelmChartEnvMatchesServiceReads` in `devplus-platform/tests/contract/static_config_test.go`.

4. **`devplus-governance-service` & `devplus-core-infra` — Fix Lifecycle, Sampling Env, and Shared Transport**
   - **`devplus-governance-service/internal/app/app.go`:** Call `otel.InitOTel(ctx, "governance")` at the very start of `Run()` (before `database.NewDatabase`, `startDecisionLog`, and `server.Register`) so `defer otelShutdown` runs *last* after `decisionLog.Close` and `govDB.Close`.
   - **`devplus-core-infra/github-callback/main.go`:** Trap `os.Interrupt` / `syscall.SIGTERM` via `signal.NotifyContext` and call `srv.Shutdown(shutdownCtx)` so `defer otelShutdown(ctx)` flushes spans cleanly on pod termination.
   - **`devplus-core-infra/auth-service.tf` & `github-callback.tf`:** Inject `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_TRACES_SAMPLER = "parentbased_traceidratio"`, and `OTEL_TRACES_SAMPLER_ARG` to match the Phase 0 head-sampling contract in `otel-collector.tf`.
   - **Shared Instrumented HTTP Client (`devplus-shared/httpclient`):** Promote `devplus-backend/internal/httpclient` to `devplus-shared/httpclient` (or wrap remaining bare `&http.Client{}` instances in `oidc_mint.go`, `auth-service/main.go`, `devplus-shared/gcpauth`, `executor/internal/ghapi`, and `executor/internal/osv` with `otelhttp.NewTransport`).

---

### Phase 2: End-to-End Trace Continuity (`backend` ↔ `executor` & `agent-service`)

1. **Propagate Trace Context Across MongoDB Queues (`devplus-shared`, `devplus-backend`, `devplus-executor`)**
   - Add optional `TraceParent string` (`bson:"traceparent,omitempty" json:"traceparent,omitempty"`) and `TraceState string` (`bson:"tracestate,omitempty" json:"tracestate,omitempty"`) fields to:
     - `DeploymentQueueItem` (`models/deployment_queue.go`)
     - `ModuleQueueItem` (`models/module_queue.go`)
     - `DiscoveryQueueItem` (`models/discovery.go`)
     - `AdoptionJob` (`models/adoption.go`)
     - `RepoAnalysisQueueItem` (`models/repo_analysis.go`)
     - `HealthQueueItem` (`models/health.go`)
     - `CostSyncQueueItem` (`models/cost.go`)
   - Add helper `otel.InjectQueueCarrier(ctx) (traceParent, traceState string)` and `otel.ExtractQueueContext(ctx, traceParent, traceState) context.Context` in `devplus-shared/otel`.
   - When `devplus-backend` enqueues an item, stamp `TraceParent`/`TraceState` from `r.Context()`.
   - When `devplus-executor` dequeues an item in `dispatchFromQueue`, extract the parent context before starting `executor.dispatchJob` (which then forwards `TRACEPARENT` to the K8s Job pod's `executor.runJob`).

2. **Instrument `devplus-agent-service` (Python / FastAPI / LangGraph)**
   - **Files:** `devplus-agent-service/pyproject.toml`, `src/agent_service/api/main.py`, `src/agent_service/api/utils/otel.py`, `src/agent_service/domain/http.py`, `devplus-platform/k8s/helm/devplus/templates/agent-service.yaml`
   - Add `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-grpc`, `opentelemetry-instrumentation-fastapi`, `opentelemetry-instrumentation-httpx`, and `opentelemetry-instrumentation-pymongo` to `pyproject.toml`.
   - Create `init_otel(app)` in `src/agent_service/api/utils/otel.py` using the same contract as Go's `InitOTel`: no-op when `OTEL_EXPORTER_OTLP_ENDPOINT` is empty; when set, configure OTLP/gRPC trace + metric export, W3C `TraceContext` propagation, FastAPI server instrumentation, and `httpx` client instrumentation (so all tool calls back to `devplus-backend` carry `traceparent`).
   - Emit spans around LangGraph node executions and tool invocations (recording model deployment name, token counts, and run ID without logging prompt/secret content).
   - Add `{{- include "devplus.otel-env" (dict "root" . "service" "agent-service") | indent 8 }}` to `agent-service.yaml`.

---

### Phase 3: Domain Metrics, Tenant Defaults & Collector Alerting

1. **Custom Domain Metrics (`otel.Meter`)**
   - **`devplus-executor`:**
     - `devplus.executor.job.duration` (histogram, seconds; attributes: `job_kind`, `action`, `status`)
     - `devplus.executor.queue.wait_duration` (histogram, seconds from `queuedAt` to `startedAt`; attributes: `job_kind`)
     - `devplus.executor.tofu.stage_duration` (histogram, seconds; attributes: `stage` = `init`|`plan`|`apply`|`destroy`, `status`)
   - **`devplus-backend`:**
     - `devplus.backend.ws.active_connections` (up-down counter)
     - `devplus.backend.promotion.count` (counter; attributes: `status`)
   - **`devplus-governance-service`:**
     - `devplus.governance.eval.duration` (histogram, seconds; attributes: `decision` = `allow`|`deny`)
     - `devplus.governance.eval.total` (counter; attributes: `decision`, `action`)
   - **`devplus-opentofu-server`:**
     - `devplus.opentofu.state.lock_wait_duration` / `lock_conflict_total` (counter)

2. **Infrastructure & Alerting (`devplus-core-infra`, `terraform-devplus-tenant`)**
   - Enable `otel_enabled = true` on AKS tenant deployments where the `observability` collector is present.
   - Add Azure Monitor metric/log alerts in `devplus-core-infra/metric-alerts.tf` for:
     - OTel Collector exporter queue saturation / dropped spans (`otelcol_exporter_enqueue_failed_spans`, `otelcol_processor_refused_spans`).
     - Sustained HTTP 5xx rate / unhandled exceptions in Application Insights (`appi-devplus-<env>`).

---

## Acceptance Criteria ("Done When")

- [ ] `devplus-backend` `LoggingMiddleware` logs the active `trace_id` on every HTTP request (verified by unit test).
- [ ] `devplus-executor` `dispatchOnce` no longer emits trace spans on empty queue polls, and forwards `OTEL_TRACES_SAMPLER*` to all child K8s Job pods.
- [ ] Queue items carry `traceparent`/`tracestate` from `devplus-backend` to `devplus-executor`, producing a single connected trace from API request → dispatcher → K8s Job pod → `opentofu-server`.
- [ ] `devplus-mcp` initializes `otel.InitOTel`, instruments inbound/outbound HTTP with `otelhttp`, and is covered by `TestHelmChartEnvMatchesServiceReads`.
- [ ] `devplus-governance-service` and `github-callback` initialize and flush OTel in the proper lifecycle order, and `auth-service.tf` / `github-callback.tf` inject `OTEL_TRACES_SAMPLER*` and `OTEL_RESOURCE_ATTRIBUTES`.
- [ ] `devplus-agent-service` exports OTLP traces/metrics when `OTEL_EXPORTER_OTLP_ENDPOINT` is set and propagates `traceparent` across `backend` → `agent-service` → `backend` tool calls.
- [ ] Custom `otel.Meter` domain metrics are emitted by `executor`, `backend`, `governance`, and `opentofu-server`.
- [ ] Whole-repo unit test coverage stays `>= 80%` across all modified repositories.
