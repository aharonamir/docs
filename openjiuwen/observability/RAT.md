# Requirements Analysis — Observability Metrics & Cost

---

## Source of Demand

- **Proactive Planning** — Engineering Quality / Operational Excellence
- **Product Requirements** — OpenJiuwen Platform / Agent Debugging & Cost Visibility

---

## Demand Background

### WHY

OpenJiuwen runs single agents and multi-agent teams across hundreds of sessions, and the
observability layer already produces structured traces for both. Two operational gaps
remain after tracing is in place:

**1. No time-series metrics** — Token usage, LLM/tool latency, and error rates are visible
only as per-span trace data. Operators cannot answer trend questions ("is token spend
growing?", "which tool regressed this week?", "what is the p95 iteration time?") without
re-querying individual traces. There is no OTel `MeterProvider` output to scrape into
Prometheus/Grafana.

**2. Cost and per-run usage are not aggregated** — Providers return cost on many LLM
responses and those values flow through to span attributes, but:

- When a provider omits cost, nothing estimates it.
- There is no aggregated per-trace count of tokens, tool calls, and estimated cost stamped
  on the run/task root span, so a single agent run (N LLM + M tool spans) has no
  "what did this run cost" answer, and a team task has no "what did this task cost" answer.

These gaps mean cost attribution and capacity planning require manual span-by-span
inspection rather than structured metrics and rollup data.

### WHEN

In progress as of 2026-09-03. The August 2026 observability restructure and the
tracing/trajectory work are **done** (out of scope here); the remaining roadmap is the
metrics + cost work described below, implemented on the stacked `feat-otel-*` branches:

- Metrics (config, recorder, runtime, emission, deploy): PR #1099, head `feat-otel-05-deploy`.
- Cost estimation + usage rollup: PR #1100, head `feat-otel-08-rollup`.

### WHAT

The observability runtime was unified into one shared layer, and the earlier phase plan
(session spans, task trajectory, `tracer_otel` split) was superseded. Already delivered:
standalone DeepAgent and sub-agent tracing, ordered trajectory events
(`OJ_TRAJECTORY_RECORD_KIND`), and provider-reported cost passthrough
(`OJ_GEN_AI_USAGE_{INPUT,OUTPUT,TOTAL}_COST`) on LLM spans. The remaining scope is:

- **Part A — OTel Metrics**: disabled-by-default metrics instruments emitted from the
  existing callback handler and agent rail.
- **Part B — Cost estimation & aggregation**: a versioned pricing table that fills cost
  when the provider omits it, plus a trace-keyed rollup stamped on single-agent
  (`openjiuwen.run.*`) and team (`agentteam.task.*`) root spans.

Source of truth for implementation steps: `docs/openjiuwen/observability/PLAN.md`.

---

**Component 1 — Disabled-by-default OTel metrics (Part A)**

Adds an OTel `MeterProvider` alongside the existing trace provider, emitting token usage,
latency, and error counters as time-series.

- `ObservabilityConfig` gains `metrics_enabled: bool = False`, `metrics_endpoint: str = ""`
  (empty → fall back to `endpoint`), `metrics_exporter: Literal["otlp_grpc", "otlp_http", "console"]`
  (default `otlp_grpc`).
- A `MetricsRecorder` (in `extensions/observability/metrics.py`) builds one
  `MeterProvider` + `PeriodicExportingMetricReader`. It is **self-contained**: it never
  calls `metrics.set_meter_provider`, so it coexists with any other meter provider and
  with tests that build two recorders.
- `ObservabilityRuntime` owns the recorder: it is created in **both** `initialize()`
  branches (first-init and already-initialized) when `metrics_enabled`, and shut down /
  cleared in `shutdown()` and the rollback path. `setup.py` exposes
  `get_metrics_recorder()` / `is_metrics_enabled()`.

Instruments:

| Instrument | Type | Labels |
|---|---|---|
| `llm.token_usage` | Counter | model, agent_id, kind (prompt\|completion) |
| `llm.call.duration` | Histogram (ms) | model, agent_id |
| `tool.call.duration` | Histogram (ms) | tool_name, agent_id |
| `tool.call.errors` | Counter | tool_name, agent_id |
| `deepagent.task.iteration.duration` | Histogram (ms) | agent_id, team_id |
| `deepagent.task.iteration.errors` | Counter | agent_id, team_id |

Emission points are the existing close sites: `OtelCallbackHandler._emit_llm_metrics`
(LLM close, before `span.end()`, covering stream + non-stream), `_emit_tool_metrics`
(tool finish/error), and `AgentObservabilityRail._emit_iteration_metrics`
(`after_task_iteration` / error). Every emission is wrapped in `try/except` +
`logger.warning` so metrics can never raise into the agent/tool execution path.

Deploy: the collector config adds a `metrics` pipeline (otlp → `prometheus` + `debug`) on
the existing `8889` port, `docker-compose.yml` adds a `prometheus` service, and a
`prometheus.yml` scrape config targets `otel-collector:8889`.

---

**Component 2 — Versioned cost estimation (Part B)**

Fills the gap when a provider omits cost on an LLM response.

`cost_tracker.py` provides a versioned per-1M-token price table:

- `ModelPrice(input_usd_per_1m, output_usd_per_1m)` and `CostEstimate(...)` frozen dataclasses.
- `estimate_cost(model, prompt_tokens, completion_tokens) -> CostEstimate` — cost only when
  the model is in the table; unknown models return zero cost with `known=False` — **never
  guessed**.
- `register_model_prices(version, prices)` is a **full-replacement** register (not a merge),
  so a registered version is a single authoritative snapshot; `PRICING_VERSION` labels it.
- Prices are static and must be updated manually when providers change pricing.

In `callback_handler._record_usage_attrs`, after the provider-cost passthrough, if the
provider reported no cost and the model is known, `estimate_cost` writes
`OJ_GEN_AI_USAGE_{INPUT,OUTPUT,TOTAL}_COST` on the span.

---

**Component 3 — Trace-keyed usage rollup on root spans (Part B)**

Aggregates LLM/tool facts per trace and stamps them on the root span so each run/task
carries its own totals.

`usage_aggregation.py` provides a thread-safe `UsageAccumulator` keyed by `trace_id`
(singleton via `get_accumulator()`):

- `accumulate_llm(trace_id, *, prompt, completion, cost)` — called once per LLM span close.
- `accumulate_tool(trace_id, *, is_error)` — called at tool finish/error, including the
  authoritative single-agent tool spans closed by `AgentObservabilityRail`.
- `snapshot(trace_id)` / `clear(trace_id)`, plus a shared `drain_rollup(trace_id)`
  accessor that snapshots-and-clears atomically so no finalizer can leak an accumulator
  entry.

Stamping (attribute names live in `semconv.py` — no hardcoded literals):

- **Single-agent root** — `close_agent_run_span` (`harness/observability/run_span.py`)
  stamps `openjiuwen.run.total_prompt_tokens`, `.total_completion_tokens`,
  `.total_tool_calls`, `.estimated_cost_usd` (`OJ_RUN_*` constants) when the rollup is
  non-empty.
- **Team root** — `finalize_trace` (`agent_teams/observability/span_context.py`) — the team
  root's real close point — drains and stamps `agentteam.task.total_prompt_tokens`,
  `.total_completion_tokens`, `.total_tool_calls`, `.estimated_cost_usd`
  (`AT_TASK_*` constants), then clears the entry.

**Known limitation (accepted, no new channel):** the rollup is **process-local**. With the
default `spawn_mode="process"`, teammates run in separate processes, each with its own
accumulator, so the leader's `finalize_trace` only sees usage that ran in the leader
process. `agentteam.task.*` totals are therefore best-effort/incomplete across processes.
No cross-process aggregation channel is added.

---

### Requirement Type

☑ **Functionality** (new metrics instruments, new cost estimation, new rollup aggregation)
☑ **Operation and Maintenance** (observability and cost visibility for production agents)

---

## Needs Assessment

### Constraints

**Metrics stay disabled by default:**
`metrics_enabled=False` is the default. Tracing-only users must get zero extra provider or
exporter side effects; a `metrics_enabled=False` acquisition creates no `MeterProvider`.

**No new pip dependencies:**
`opentelemetry-sdk`, `opentelemetry-api`, `opentelemetry-exporter-otlp-proto-grpc/-http`
are already in the `observability` extra (`pyproject.toml:99-102`). Nothing new is added.

**Metrics must never raise into the hot path:**
Every emission is wrapped in `try/except` + `logger.warning`, mirroring the existing
`SafeSpanProcessor` pattern. A recorder failure (or absent recorder) degrades to a no-op.

**Single global `TracerProvider`:**
OpenTelemetry allows exactly one global trace provider per process, and whichever runtime
initializes first owns it. `extensions/observability/demand.py` coordinates this with
per-runtime *acquire* / *release* demand counting across the single-agent and team
runtimes; the provider is shut down only when the last demand is gone.

**The metrics recorder is self-contained:**
`MetricsRecorder` uses its own provider's meter directly and never calls
`metrics.set_meter_provider`, avoiding conflict with coexisting meter providers.

**Cost table is versioned and static:**
Prices are not fetched at runtime; unknown models are marked (`known=False`) and never
guessed. The table must be updated manually when provider pricing changes.

**Rollup must be drained, never left to grow:**
Accumulator entries are keyed by `trace_id` and must be snapshot-and-cleared (via
`drain_rollup`) at the root finalizer. Team totals are best-effort across processes
(see Component 3 known limitation); no aggregation channel is added in scope.

### Impact of Requirement Implementation on Existing Systems

**`ObservabilityConfig` — new optional fields (additive):**
`metrics_enabled`, `metrics_endpoint`, `metrics_exporter` are added with defaults. Existing
configs that do not set them behave as before.

**`ObservabilityRuntime` — recorder ownership (additive):**
When `metrics_enabled=False`, no recorder exists and nothing changes. When enabled, the
runtime builds and owns the recorder in both `initialize()` branches, matching the
already-initialized second-runtime case.

**`OtelCallbackHandler` / `AgentObservabilityRail` — guarded emission (additive):**
LLM/tool/iteration metric emission and rollup accumulation are added at existing close
sites. With no recorder, `get_metrics_recorder()` returns `None` and all calls no-op.

**Root-span stamping (additive):**
`close_agent_run_span` and `finalize_trace` stamp rollup attributes only when the
`drain_rollup` snapshot is non-empty; traces without accumulated usage are unchanged. The
naming constants (`OJ_RUN_*`, `AT_TASK_*`) are added to `semconv.py`.

**Existing observability tests:**
Retained. New suites cover the recorder, cost tracker, usage aggregation, emission, and
the finalize-trace stamping paths.

**Deploy:**
`otel-collector-config.yaml` adds a `metrics` pipeline; the `traces` pipeline and the
Langfuse export are unchanged. `docker-compose.yml` adds a `prometheus` service. Existing
trace-only deployments are unaffected.

### External Dependencies

| Dependency | Required for | Notes |
|---|---|---|
| `opentelemetry-api` / `opentelemetry-sdk` | All parts | Already in `pyproject.toml`. SDK provides `MeterProvider`/`Counter`/`Histogram`. |
| `opentelemetry-exporter-otlp-proto-grpc` / `-http` | Metrics export | Already pinned in the `observability` extra. |
| OTel Collector | Metrics export | Must be running with the updated config (adds a metrics pipeline). Existing deployments need the config change to export metrics. |
| Prometheus | Metrics scraping | New `prom/prometheus` service in `docker-compose.yml`; optional if the collector is elsewhere. |
| Grafana | Dashboards | Optional; required only if metric dashboards are needed. |
| Langfuse | Traces | Unchanged; no new setup required for traces. |
