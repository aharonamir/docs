# System Investigation — Observability Enhancement

**Related document:** RAT.md — product requirements and business background.  
This document covers architecture, component decomposition, data flows, technical
constraints, system impact, and end-to-end scenarios for the same feature.

---

## Feature Scope

The observability enhancement extends `openjiuwen/agent_teams/observability/` with five
new capabilities delivered as independent phases:

1. **Standalone DeepAgent OTel** — make `ObservabilityRail` work outside a team context
2. **Session-Level Spans** — one root trace per session that groups all iterations
3. **Task Trajectory Tracking** — ordered step sequence per iteration as span events
4. **Metrics Infrastructure** — OTel Metrics (`MeterProvider`) for token/latency/error counters
5. **Multi-Agent Task Aggregation + Cost Tracking** — per-member contribution summaries and cost estimation

All changes are confined to `openjiuwen/agent_teams/observability/` (primary) and
`openjiuwen/harness/factory.py` (factory wiring). The separate
`openjiuwen/extensions/tracer_otel/` system for single-invoke agents is not touched.

---

## Architecture

### High-level data flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  DeepAgent (task loop)                                                   │
│                                                                          │
│  Iteration 1            Iteration 2            Iteration N               │
│  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐          │
│  │ before_iter │        │ before_iter │        │ before_iter │          │
│  │  LLM call   │        │  LLM call   │  ...   │  LLM call   │          │
│  │  tool call  │        │  tool call  │        │  tool call  │          │
│  │ after_iter  │        │ after_iter  │        │ after_iter  │          │
│  └──────┬──────┘        └──────┬──────┘        └──────┬──────┘          │
└─────────┼─────────────────────┼─────────────────────┼────────────────── ┘
          │ DeepAgentRail        │ DeepAgentRail        │ DeepAgentRail
          ▼ hooks                ▼                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  openjiuwen/agent_teams/observability/                                   │
│                                                                          │
│  rail.py ──────── ObservabilityRail                                      │
│                   ├── before_task_iteration                              │
│                   │   ├── resolve parent: session span (Ph2)             │
│                   │   │                   or team span                   │
│                   │   │                   or standalone root (Ph1)       │
│                   │   └── start + register iteration span                │
│                   └── after_task_iteration                               │
│                       ├── emit trajectory events (Ph3)                   │
│                       ├── record iteration metric (Ph4)                  │
│                       └── close iteration span (+ standalone root)       │
│                                                                          │
│  callback_handler.py ─ OtelCallbackHandler                               │
│                        ├── on_llm_start/end  → llm.call span             │
│                        │   └── record_llm_step (Ph3), MetricsRecorder (Ph4)│
│                        └── on_tool_start/end → tool.{name} span          │
│                            └── record_tool_step (Ph3), MetricsRecorder (Ph4)│
│                                                                          │
│  session_span.py ──── SessionSpanManager (Ph2)                           │
│                        ├── open_session_span → SERVER root span           │
│                        └── close_session_span                            │
│                                                                          │
│  trajectory_tracker.py TrajectoryTracker (Ph3)                           │
│  metrics.py ────────── MetricsRecorder (Ph4)                             │
│  task_aggregator.py ── TaskAggregator (Ph5)                              │
│  cost_tracker.py ───── CostTracker (Ph5)                                 │
│                                                                          │
│  monitor_handler.py ── OtelTeamMonitorHandler (existing)                 │
│                        + TaskAggregator wiring (Ph5)                     │
│  setup.py ─────────── TracerProvider + MeterProvider (Ph4)               │
│  span_context.py ───── ActiveSpanTracker, ContextVar management           │
│  semconv.py ────────── all OTel semantic convention constants             │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ OTLP export
                                       ▼
                    ┌──────────────────────────────────────┐
                    │  OTel Collector (or direct export)    │
                    │  ┌──────────────┐ ┌───────────────┐  │
                    │  │   Langfuse   │ │ Grafana (Ph4) │  │
                    │  │  (traces)    │ │  (metrics)    │  │
                    │  └──────────────┘ └───────────────┘  │
                    └──────────────────────────────────────┘
```

### Span parent-child resolution in `before_task_iteration`

```
before_task_iteration called
         │
         ▼
  _SPAN_KEY in ctx.extra? ──yes──► return (duplicate-call guard)
         │ no
         ▼
  get_session_span() (Ph2) ──found──► use session span as parent
         │ not found
         ▼
  get_team_span() ──found and recording──► use team span as parent
         │
         ├──None──────────────────────────────────────────────────────────┐
         │                                                                │
         │ standalone path (Ph1)                                          │
         │ start SERVER span via start_span(context=OtelContext())        │
         │ store in ctx.extra[_STANDALONE_ROOT_KEY]                       │
         │ ↓                                                              │
         └──────────────────────────────────────────────────────────────►│
                                                                         │
         ▼ (all paths converge here)                                      │
  start INTERNAL iteration span (child of parent_ctx)  ◄─────────────────┘
  set_current_agent_span(iteration_span)
  ctx.extra[_SPAN_KEY] = iteration_span
```

### Design principles

**Single global `TracerProvider`, separate from `extensions/tracer_otel/`.**  
`setup.py:init_observability()` calls `set_tracer_provider()` globally. This provider
is used by the entire `agent_teams/observability/` subsystem. `extensions/tracer_otel/`
maintains its own `TracerProvider` that deliberately does not call `set_tracer_provider()`
— the two must never share a provider or interfere with each other's context.

**ContextVar-based span state, never thread-local.**  
All active span references (`_current_agent_span`, the session span registry,
trajectory step buffers) use Python `ContextVar`. This is safe for both asyncio
and multi-threaded agent tasks because each task gets its own context copy.

**`OtelContext()` (empty context) for trace isolation.**  
When starting a standalone root span, `start_span(context=OtelContext())` passes an
empty OpenTelemetry context. Without this, the span inherits whatever OTel context was
attached from the previous iteration, causing all root spans to share the same
`trace_id`. An empty context forces each standalone iteration to begin a fresh trace.
Phase 2's session span replaces this — all iterations become children of the session span
and intentionally share its `trace_id`.

**Rail stores ephemeral state in `ctx.extra`, not in `self`.**  
`ObservabilityRail` is a singleton attached to the agent. It must not hold per-iteration
state as instance attributes. All per-iteration data (iteration span, standalone root
span) is stored in `ctx.extra[key]` and popped in `after_task_iteration`, so each
iteration is fully isolated even if the same rail object is reused across thousands of
iterations.

---

### Module layout

```
openjiuwen/agent_teams/observability/
├── __init__.py              ← public exports: init_observability, shutdown_observability,
│                              ObservabilityConfig, ObservabilityRail, is_initialized
│
├── setup.py                 ← TracerProvider lifecycle; init_observability(),
│                              shutdown_observability(), get_tracer(), is_initialized()
│                              Phase 4: MeterProvider, MetricsRecorder init
│
├── config.py                ← ObservabilityConfig (Pydantic)
│                              Phase 4 adds: metrics_enabled, metrics_endpoint
│
├── semconv.py               ← all semantic convention string constants
│                              gen_ai.*, agentteam.*, deepagent.*, langfuse.*
│                              Phase 2 adds: SESSION_*
│                              Phase 3 adds: TRAJ_*
│                              Phase 5 adds: AT_TASK_TOTAL_*, SESSION_ESTIMATED_COST_USD
│
├── span_context.py          ← ActiveSpanTracker (SpanProcessor), LlmSpanState
│                              ContextVar management: _current_agent_span
│                              finalize_trace(), flush_child_spans()
│
├── rail.py                  ← ObservabilityRail(DeepAgentRail)
│                              Phase 1: standalone root span fallback
│                              Phase 2: prefer session span as parent
│                              Phase 3: call trajectory_tracker.emit_to_span()
│                              Phase 4: call MetricsRecorder.record_task_iteration()
│
├── callback_handler.py      ← OtelCallbackHandler
│                              LLM/tool/agent events from Runner.callback_framework
│                              Phase 3: call TrajectoryTracker.record_*_step()
│                              Phase 4: call MetricsRecorder.record_*()
│                              Phase 5: call CostTracker.record_usage()
│
├── monitor_handler.py       ← OtelTeamMonitorHandler (existing)
│                              task/member/message spans from TeamEvent stream
│                              Phase 5: wire TaskAggregator on TASK_* events
│
├── session_span.py          ← NEW Phase 2 — SessionSpanManager
│                              open_session_span(), close_session_span(),
│                              get_session_span()
│
├── trajectory_tracker.py    ← NEW Phase 3 — TrajectoryTracker
│                              record_llm_step(), record_tool_step(),
│                              emit_to_span(), reset()
│
├── metrics.py               ← NEW Phase 4 — MetricsRecorder
│                              MeterProvider wrapper; token counter, latency histogram,
│                              error counter; record_llm_call(), record_tool_call(),
│                              record_task_iteration()
│
├── task_aggregator.py       ← NEW Phase 5 — TaskAggregator
│                              per-task member contribution accumulation;
│                              emits aggregated span attributes on TASK_COMPLETED
│
└── cost_tracker.py          ← NEW Phase 5 — CostTracker
                               PRICING_TABLE; estimate_cost(); ContextVar accumulation
                               per session; emits session.estimated_cost_usd on close
```

---

## Key Sequence Diagrams

### 1. Standalone DeepAgent — two iterations (Phase 1 + Phase 2)

Phase 1 (no session span): each iteration is its own independent trace.  
Phase 2 (session span active): both iterations share the session span as root.

**Phase 1 — per-iteration independent traces:**

```
ObservabilityRail         OtelCallbackHandler     OTel SDK (spans)
       │                          │                      │
       │── before_task_iteration──►                      │
       │   (team_span is None,                           │
       │    no session span)                             │
       │                                                 │
       │   start_span("agent.X", SERVER,                 │
       │               context=OtelContext())────────────►── [root1: trace_id=AAA]
       │   ctx.extra[ROOT_KEY] = root1                   │
       │   start_span("agent.X.task_iteration.1",        │
       │               INTERNAL, parent=root1)───────────►── [iter1: parent=root1]
       │   set_current_agent_span(iter1)                 │
       │                          │                      │
       │                          │── on_llm_start ──────►── [llm1: parent=iter1]
       │                          │── on_llm_end ────────►── llm1.end()
       │                          │   record_llm_step()  │   (Ph3)
       │                          │── on_tool_start ─────►── [tool1: parent=iter1]
       │                          │── on_tool_end ───────►── tool1.end()
       │                          │   record_tool_step() │   (Ph3)
       │                          │                      │
       │── after_task_iteration──►                       │
       │   emit_to_span(iter1) (Ph3)                     │
       │   iter1.set_status(OK); iter1.end()─────────────►── iter1.end()
       │   root1.set_status(OK); root1.end()─────────────►── root1.end()
       │                                                 │   trace_id=AAA closed
       │                                                 │
       │   ── ITERATION 2 ──────────────────────────────►   (fresh trace_id=BBB)
       │   start_span("agent.X", SERVER,                 │
       │               context=OtelContext())────────────►── [root2: trace_id=BBB]
       │   ...                                           │
```

**Phase 2 — session span groups all iterations under one trace:**

```
SessionSpanManager      ObservabilityRail         OTel SDK (spans)
       │                       │                        │
       │ session_created event │                        │
       │── open_session_span()──────────────────────────►── [session: trace_id=AAA, SERVER]
       │   store in ContextVar │                        │
       │                       │                        │
       │                       │── before_task_iteration│
       │                       │   get_session_span()   │
       │                       │   returns session span  │
       │                       │   start_span("agent.X.task_iteration.1",
       │                       │               INTERNAL,│
       │                       │               parent=session span)
       │                       │───────────────────────►── [iter1: trace_id=AAA, parent=session]
       │                       │── after_task_iteration─►── iter1.end()
       │                       │                        │   (session span stays open)
       │                       │── before_task_iteration│   (iteration 2)
       │                       │───────────────────────►── [iter2: trace_id=AAA, parent=session]
       │                       │── after_task_iteration─►── iter2.end()
       │                       │                        │
       │ session_closed event  │                        │
       │── close_session_span()─────────────────────────►── session.end()
       │                                                │   trace_id=AAA closed
```

---

### 2. LLM call — span parent resolution in OtelCallbackHandler

```
OtelCallbackHandler         span_context.py           OTel SDK
       │                           │                      │
       │── on_llm_start ──────────►│                      │
       │   _get_parent_context_    │                      │
       │   for_llm_tool()          │                      │
       │                           │ get_current_agent_span()
       │                           │ returns iter_span (set by rail.before)
       │                           │                      │
       │   set_span_in_context(    │                      │
       │     iter_span,            │                      │
       │     get_current())        │                      │
       │                           │                      │
       │   start_span("llm.call",  │                      │
       │     GENERATION,           │                      │
       │     parent=iter_span_ctx)─────────────────────►── [llm_span: parent=iter_span]
       │   store llm_span in       │                      │
       │   LlmSpanState (ContextVar)                      │
       │                           │                      │
       │── on_llm_end ────────────►│                      │
       │   set token attributes    │                      │
       │   set status OK           │                      │
       │   llm_span.end() ─────────────────────────────►── llm_span.end()
       │   record_llm_step() (Ph3) │                      │
       │   MetricsRecorder.        │                      │
       │     record_llm_call() (Ph4)                      │
       │   CostTracker.            │                      │
       │     record_usage() (Ph5)  │                      │
```

---

### 3. Team task aggregation — Phase 5

```
TeamEvent stream        OtelTeamMonitorHandler    TaskAggregator        OTel SDK
       │                        │                      │                     │
       │── TASK_ASSIGNED ───────►                      │                     │
       │   task_id="t-1"        │── init_task("t-1")──►                     │
       │                        │   start_span("task.t-1")──────────────────►── [task_span]
       │                        │                      │                     │
       │── AGENT_CALLED ────────►                      │                     │
       │   member="A"           │── record_member_start("t-1","A")──►        │
       │                        │                      │ accumulate[t-1][A]  │
       │                        │                      │ = {tokens:0, ...}   │
       │                        │                      │                     │
       │  (OtelCallbackHandler receives A's LLM/tool events)                 │
       │  MetricsRecorder accumulates tokens per agent_id                    │
       │                        │                      │                     │
       │── AGENT_DONE ──────────►                      │                     │
       │   member="A"           │── record_member_end  │                     │
       │   tokens_used=1500     │   ("t-1","A",        │                     │
       │                        │    tokens=1500,       │                     │
       │                        │    tool_calls=3,      │                     │
       │                        │    duration_ms=4200)─►                     │
       │                        │                      │ accumulate[t-1][A]  │
       │                        │                      │ = {tokens:1500,     │
       │                        │                      │    tool_calls:3,    │
       │                        │                      │    duration_ms:4200}│
       │                        │                      │                     │
       │── TASK_COMPLETED ──────►                      │                     │
       │   task_id="t-1"        │── finalize("t-1") ──►                     │
       │                        │                      │ compute totals:     │
       │                        │                      │  total_tokens=1500  │
       │                        │                      │  members={A:{...}}  │
       │                        │◄── aggregated attrs ─│                     │
       │                        │   set on task_span    │                     │
       │                        │── task_span.end()─────────────────────────►── task_span.end()
       │                        │   AT_TASK_TOTAL_TOKENS=1500               │
       │                        │   AT_TASK_MEMBER_CONTRIBUTIONS={...}      │
```

---

## Component Breakdown

### Existing components (modified by this feature)

| Component | File | Role | What changes |
|---|---|---|---|
| `ObservabilityRail` | `rail.py` | `DeepAgentRail`; starts/closes per-iteration agent spans | Ph1: standalone fallback; Ph2: prefer session span; Ph3: emit trajectory; Ph4: record iteration metric |
| `OtelCallbackHandler` | `callback_handler.py` | Creates LLM/tool child spans from `Runner.callback_framework` events | Ph3: `record_*_step()`; Ph4: `MetricsRecorder.record_*()`; Ph5: `CostTracker.record_usage()` |
| `OtelTeamMonitorHandler` | `monitor_handler.py` | Creates task/member/message spans from `TeamEvent` stream | Ph5: wire `TaskAggregator` on `TASK_*` events |
| `init_observability` / `setup.py` | `setup.py` | `TracerProvider` lifecycle; compat shim for OTel API/SDK mismatch | Ph2: register session event handlers; Ph4: init `MeterProvider` + `MetricsRecorder`; Ph5: init `TaskAggregator` + `CostTracker` |
| `ObservabilityConfig` | `config.py` | Pydantic config: endpoint, Langfuse auth, enabled flag | Ph4: add `metrics_enabled: bool = False`, `metrics_endpoint: str = ""` |
| `semconv.py` | `semconv.py` | String constants for all OTel attribute keys | Ph2: `SESSION_*`; Ph3: `TRAJ_*`; Ph5: `AT_TASK_TOTAL_*`, `SESSION_ESTIMATED_COST_USD` |
| `create_deep_agent()` | `harness/factory.py` | Factory for `DeepAgent` + task loop | Ph1: `observability_config` optional kwarg; auto-init + auto-attach `ObservabilityRail` |

### New components

| Component | File | Phase | Key interface |
|---|---|---|---|
| `SessionSpanManager` | `session_span.py` | 2 | `open_session_span(session_id, tracer)` → `Span`; `close_session_span(session_id)`; `get_session_span(session_id)` → `Span \| None` |
| `TrajectoryTracker` | `trajectory_tracker.py` | 3 | `record_llm_step(model, prompt_tokens, completion_tokens, ttft_ms)`; `record_tool_step(tool_name, input_len, output_len, duration_ms)`; `emit_to_span(span)`; `reset()` |
| `MetricsRecorder` | `metrics.py` | 4 | `record_llm_call(model, prompt_tokens, completion_tokens, duration_ms, agent_id)`; `record_tool_call(tool_name, duration_ms, agent_id, error)`; `record_task_iteration(duration_ms, agent_id, is_error)` |
| `TaskAggregator` | `task_aggregator.py` | 5 | `init_task(task_id)`; `record_member_start(task_id, member_id)`; `record_member_end(task_id, member_id, tokens, tool_calls, duration_ms)`; `finalize(task_id)` → `dict` |
| `CostTracker` | `cost_tracker.py` | 5 | `PRICING_TABLE: dict[str, tuple[float, float]]`; `estimate_cost(model, input_tokens, output_tokens) → float`; `record_usage(model, prompt_tokens, completion_tokens)`; `get_session_cost() → float` |

### OTel instruments (Phase 4)

| Instrument name | Type | Unit | Labels | Recorded by |
|---|---|---|---|---|
| `gen_ai.token.usage` | Counter | `{token}` | `model`, `agent_id`, `token_type` (`prompt`/`completion`) | `MetricsRecorder.record_llm_call()` |
| `gen_ai.llm.request.duration` | Histogram | `ms` | `model` | `MetricsRecorder.record_llm_call()` |
| `agent.tool.call.duration` | Histogram | `ms` | `tool_name` | `MetricsRecorder.record_tool_call()` |
| `agent.tool.call.errors` | Counter | `{call}` | `tool_name`, `agent_id` | `MetricsRecorder.record_tool_call(error=True)` |
| `agent.task.iteration.duration` | Histogram | `ms` | `agent_id` | `MetricsRecorder.record_task_iteration()` |
| `agent.task.iteration.errors` | Counter | `{iteration}` | `agent_id` | `MetricsRecorder.record_task_iteration(is_error=True)` |

---

## Testing

| Phase | Test file | Approach | Key assertions |
|---|---|---|---|
| 1 | `test_standalone_rail.py` | `InMemorySpanExporter`; mock agent with `team_name=""` | ROOT+INTERNAL span pair; same `trace_id`; `agent.name` attribute; OK/ERROR status; per-iteration trace isolation; no-duplicate guard |
| 2 | `test_session_span.py` | `InMemorySpanExporter`; fire mock `SessionEvents` | Session span opens on create event; iteration spans parent to session; all share `trace_id`; session span closes on close event; no session span → falls back to standalone root |
| 3 | `test_trajectory_tracker.py` | `InMemorySpanExporter`; call `record_*_step()` then `emit_to_span()` | Span events contain steps in order; `TRAJ_STEP_INDEX` increments; ContextVar is cleared after `emit_to_span()`; `reset()` clears without emitting |
| 4 | `test_metrics.py` | `InMemoryMetricReader`; call `MetricsRecorder.record_*()` | Counter increments match calls; histogram data points recorded; no metrics emitted when `metrics_enabled=False` |
| 5 | `test_task_aggregator.py` | Unit test; call `init_task` / `record_member_*` / `finalize` | Totals computed correctly for N members; `finalize()` dict matches expected schema |
| 5 | `test_cost_tracker.py` | Unit test; call `record_usage()` / `get_session_cost()` | Known model → expected cost; unknown model → 0.0; accumulation is additive across calls |

All unit tests live in `tests/unit_tests/agent_teams/observability/` and use only
`InMemorySpanExporter` / `InMemoryMetricReader` — no real OTel Collector or Langfuse
connection required. All 24 currently passing tests must continue to pass after each
phase.

---

## Technical Constraints

**`ObservabilityRail` is a `DeepAgentRail`, not an `AgentRail`.**  
It hooks into the DeepAgent *task loop* (`before_task_iteration` / `after_task_iteration`).
It does not apply to single-invoke agents (agents without a task loop). Single-invoke
agents are covered by `extensions/tracer_otel/OtelRail`, which is an `AgentRail` and
must not be modified as part of this feature.

**Factory wiring only applies when `enable_task_loop=True`.**  
When `observability_config` is passed to `create_deep_agent()` and `enable_task_loop`
is `False`, `init_observability()` is still called (to set up the provider), but
`ObservabilityRail` is not prepended to the rails list because there is no task loop for
it to hook into.

**`ctx.extra` keys must not collide with other rails.**  
`_SPAN_KEY = "_otel_agent_span"` and `_STANDALONE_ROOT_KEY = "_otel_standalone_root_span"`
are string keys in `ctx.extra`. Any other rail that writes the same key will silently
corrupt the span tracking. These keys are defined as module-level constants in `rail.py`
and should never be reused elsewhere.

**Standalone root span uses `OtelContext()` (empty) — not `get_current()`.**  
Passing `context=OtelContext()` to both `start_span()` and `set_span_in_context()` is
mandatory. If `get_current()` is used instead, the OTel context from a previous iteration
(or from an outer scope) is inherited, causing all root spans to share the same
`trace_id`. This was discovered and fixed during Phase 1 development.

**Phase 2 `SessionSpanManager` uses a dict registry, not a single ContextVar.**  
A `ContextVar` holding a single span would break if multiple sessions run concurrently
in the same process (e.g. a multi-user server). The `SessionSpanManager` uses a
`dict[session_id, Span]` protected by a `threading.Lock`, keyed by session ID.
`get_session_span(session_id)` returns `None` for unknown session IDs — `rail.py`
must handle `None` gracefully.

**Phase 4 `MetricsRecorder` is a singleton; `MeterProvider` is separate from `TracerProvider`.**  
The `MetricsRecorder` is initialized once by `init_observability()` when
`config.metrics_enabled` is `True`. It wraps its own `MeterProvider`. The existing
`TracerProvider` is not reused for metrics — OTel separates the two APIs. Calling
`get_metrics_recorder()` when metrics are disabled must return a no-op recorder, not
raise an exception.

**Phase 5 `CostTracker.PRICING_TABLE` is static.**  
Prices are hardcoded as `{model_id: (input_price_per_1k, output_price_per_1k)}`. The
table covers common models (e.g. `claude-sonnet-4-6`, `claude-haiku-4-5`, `gpt-4o`).
Unknown model IDs produce a cost of `0.0`. The table must be updated manually when
providers change pricing. No runtime API call is made.

**OTel API/SDK version compat shim (pre-existing, resolved in Phase 1).**  
`setup.py` patches `TraceFlags.RANDOM_TRACE_ID` and `TraceFlags.random_trace_id` if
absent (for `opentelemetry-api < 1.42` installed alongside `opentelemetry-sdk 1.42`).
`_CompatIdGenerator(RandomIdGenerator)` gates `is_trace_id_random()` on the attribute's
presence. This shim must not be removed until the API and SDK versions are aligned in
`pyproject.toml`.

---

## Impact on Existing Systems

### `ObservabilityRail` — additive behavior change

Team agents: **no change.** The standalone fallback path is gated on `team_span is None`
after `get_session_span()` also returns `None`. For team agents, `get_team_span()` always
returns a recording span, so the new code paths are never entered.

Standalone DeepAgents: were **no-op** before Phase 1. Now produce spans. This is purely
additive — callers that do not call `init_observability()` see no spans (same as before),
because `_tracer().start_span()` returns a no-op span when no provider is configured.

### `create_deep_agent()` — backwards compatible

`observability_config: Optional[Any] = None` is added with a default of `None`.
All existing call sites that omit this parameter are unaffected. No existing parameter
is renamed or removed.

### `ObservabilityConfig` — backwards compatible (Phase 4)

`metrics_enabled: bool = False` and `metrics_endpoint: str = ""` are added as optional
Pydantic fields with defaults. Existing configs that do not set these fields deserialize
correctly with metrics disabled.

### Existing observability unit tests

All 14 pre-existing observability unit tests were silently broken before Phase 1 by the
OTel API/SDK version mismatch (`TraceFlags.RANDOM_TRACE_ID` absent). The compat shim
added in Phase 1 restored them. The 8 new Phase 1 tests bring the total to 24. Each
phase will add tests without removing existing ones.

### `extensions/tracer_otel/`

Not touched. Its `TracerProvider`, `OtelRail`, and `init_otel_tracer()` are unchanged.
The two systems continue to coexist: `extensions/tracer_otel/` does not call
`set_tracer_provider()`; `agent_teams/observability/` does.

### `deploy/observability/otel-collector-config.yaml` (Phase 4)

A metrics pipeline (receiver: `otlp`, exporter: `prometheus` or `otlp`) is added.
The existing traces pipeline is unchanged. Deployments that do not enable metrics are
unaffected — the new pipeline is inert if no metrics are exported to it.

### Performance

Tracing adds one `start_span()` + `end()` call per iteration (rail) and per LLM/tool
event (callback handler). These are synchronous in the hot path; the `BatchSpanProcessor`
exports asynchronously in a background thread. No blocking I/O on the agent's execution
path.

Phase 3 trajectory tracking accumulates step dicts in a ContextVar list per iteration.
Memory overhead is proportional to the number of LLM/tool steps in one iteration, which
is bounded by the task loop's step limit.

Phase 4 metrics use OTel's `Counter.add()` and `Histogram.record()` — both O(1) and
lock-free in the SDK implementation.

---

## End-to-End Scenarios

### Scenario A — Standalone DeepAgent, 2 iterations, Phase 1 + Phase 2

**Context:** A developer creates a `DeepAgent` with `observability_config` set. The agent
runs a task that takes 2 iterations to complete. Phase 1 only: two separate traces.
Phase 2 enabled: one trace with the session span as root.

**Step-by-step (Phase 2 active):**

```
1. FACTORY: create_deep_agent(model, enable_task_loop=True,
                observability_config=ObservabilityConfig(enabled=True, ...))
   SETUP:   init_observability(config) → TracerProvider created, OTLP exporter registered
            Session event listener registered (Ph2)
   FACTORY: ObservabilityRail() prepended to rails list

2. SESSION: AgentSession.start() fires SessionEvents.AGENT_SESSION_CREATED
   SESSION_SPAN_MGR: open_session_span("session-abc", tracer)
            → start_span("session.session-abc", SERVER, context=OtelContext())
            → session_span stored in registry["session-abc"]
            → trace_id = AAA

3. TASK LOOP — Iteration 1:
   RAIL:    before_task_iteration(ctx)
            get_session_span("session-abc") → session_span (trace_id=AAA)
            start_span("agent.my_agent.task_iteration.1", INTERNAL,
                        parent=session_span_ctx)
            → iter1_span (trace_id=AAA, parent=session_span)
            set_current_agent_span(iter1_span)

   LLM:     Runner.callback_framework fires on_llm_start
   CALLBACK: _get_parent_context_for_llm_tool() → iter1_span
             start_span("llm.call", GENERATION, parent=iter1_span)
             → llm1_span (trace_id=AAA, parent=iter1_span)
   LLM:     on_llm_end → set token attrs, llm1_span.end()
             record_llm_step(model, prompt=800, completion=200, ttft_ms=340) (Ph3)
             MetricsRecorder.record_llm_call(...) (Ph4)
             CostTracker.record_usage(...) (Ph5)

   TOOL:    on_tool_start → start_span("tool.bash", TOOL, parent=iter1_span)
   TOOL:    on_tool_end   → tool1_span.end()
             record_tool_step(...) (Ph3)

   RAIL:    after_task_iteration(ctx)
            TrajectoryTracker.emit_to_span(iter1_span) (Ph3)
            → 2 span events added to iter1_span (llm_step, tool_step)
            MetricsRecorder.record_task_iteration(...) (Ph4)
            iter1_span.set_status(OK); iter1_span.end()
            no standalone root → skip

4. TASK LOOP — Iteration 2: identical to step 3, produces iter2_span (trace_id=AAA)

5. SESSION: AgentSession.close() fires session_closed event
   SESSION_SPAN_MGR: close_session_span("session-abc")
            session_span.set_status(OK); session_span.end()
            → trace_id=AAA fully closed

6. LANGFUSE: receives one trace (trace_id=AAA)
   Span tree:
   session.session-abc                     SERVER    (root)
   ├── agent.my_agent.task_iteration.1     INTERNAL
   │   ├── llm.call                        GENERATION
   │   │   events: [traj.step.0: llm, traj.step.1: tool] (Ph3)
   │   └── tool.bash                       TOOL
   └── agent.my_agent.task_iteration.2     INTERNAL
       ├── llm.call                        GENERATION
       └── tool.bash                       TOOL
```

**Error path — iteration raises exception:**

```
   RAIL:    after_task_iteration(ctx)
            ctx.exception is not None
            iter_span.set_status(ERROR, str(ctx.exception)); iter_span.end()
            standalone_root (if any): set_status(ERROR); standalone_root.end()
            MetricsRecorder.record_task_iteration(is_error=True) (Ph4)
```

---

### Scenario B — Team agent task with per-member aggregation, Phase 5

**Context:** A team of two agents (A and B) works on a task. The system should produce
a task span with aggregated token/tool/cost attributes for both members.

**Step-by-step:**

```
1. TEAM RUNNER: starts team session → team_span opened (existing behavior)

2. TEAM EVENT: TASK_ASSIGNED(task_id="t-99", assignee="A")
   MONITOR_HANDLER: start_span("task.t-99", SPAN, parent=team_span)
   TASK_AGGREGATOR: init_task("t-99")
                    accumulate["t-99"] = {}

3. AGENT A runs — ObservabilityRail + OtelCallbackHandler fire as normal
   CALLBACK: on_llm_end  → CostTracker.record_usage("claude-sonnet-4-6", 1200, 400)
   CALLBACK: on_tool_end → MetricsRecorder.record_tool_call("bash", 120ms, "A")

4. TEAM EVENT: AGENT_DONE(task_id="t-99", member="A", tokens=1600, tool_calls=2, duration_ms=5000)
   TASK_AGGREGATOR: record_member_end("t-99", "A", tokens=1600, tool_calls=2, duration_ms=5000)
                    accumulate["t-99"]["A"] = {tokens:1600, tool_calls:2, duration:5000}

5. AGENT B runs similarly
   TEAM EVENT: AGENT_DONE(task_id="t-99", member="B", tokens=900, tool_calls=1, duration_ms=2800)
   TASK_AGGREGATOR: accumulate["t-99"]["B"] = {tokens:900, tool_calls:1, duration:2800}

6. TEAM EVENT: TASK_COMPLETED(task_id="t-99")
   TASK_AGGREGATOR: finalize("t-99")
                    total_tokens = 1600 + 900 = 2500
                    members = {A:{...}, B:{...}}
   MONITOR_HANDLER: task_span.set_attribute("agentteam.task.total_tokens", 2500)
                    task_span.set_attribute("agentteam.task.member_contributions",
                                            '{"A":{"tokens":1600,...},"B":{...}}')
                    task_span.set_attribute("agentteam.task.total_cost_usd",
                                            CostTracker.estimate_cost(...))
                    task_span.end()

7. LANGFUSE: task span shows:
   agentteam.task.total_tokens = 2500
   agentteam.task.total_cost_usd = 0.0138   (example)
   agentteam.task.member_contributions = "{...}"
```

---

## External Dependencies

### Python packages

| Package | Version | Used by | Status |
|---|---|---|---|
| `opentelemetry-api` | ≥ 1.39 (align with SDK) | All phases | Already in `pyproject.toml`; version should be pinned to match SDK |
| `opentelemetry-sdk` | 1.42.1 | All phases | Already in `pyproject.toml` |
| `opentelemetry-exporter-otlp-proto-grpc` | Any current | All phases | Already in `pyproject.toml`; exports spans to Langfuse / Collector |
| `opentelemetry-sdk` metrics API | 1.42.1 | Phase 4 | `MeterProvider`, `Counter`, `Histogram` — already available in SDK 1.42.1; no new package needed |

### Infrastructure

| Service | Required for | Setup required | Notes |
|---|---|---|---|
| Langfuse | Phases 1–3 (trace UI) | Already configured for team agents | No new setup; standalone agent traces appear automatically once spans are exported |
| OTel Collector | Phase 4 (metrics pipeline) | New: add metrics pipeline to `otel-collector-config.yaml`; deploy Collector if not already running | Traces can bypass Collector (direct OTLP to Langfuse); metrics require Collector → Prometheus/Grafana |
| Prometheus | Phase 4 (metrics storage) | New: scrape target for OTel Collector Prometheus exporter | Optional if a different metrics backend is used |
| Grafana | Phase 4 (metrics dashboards) | Optional | Only needed if dashboards for token/latency/cost metrics are desired |
