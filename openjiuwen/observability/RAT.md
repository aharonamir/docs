# Requirements Analysis — Observability Enhancement

---

## Source of Demand

- **Proactive Planning** — Engineering Quality / Operational Excellence
- **Product Requirements** — OpenJiuwen Platform / Agent Debugging & Cost Visibility

---

## Demand Background

### WHY

OpenJiuwen can run agents and multi-agent teams across hundreds of sessions. However,
once an agent is deployed, there is no structured way to answer basic operational
questions:

- What did a specific agent do during its last run?
- How many tokens did it consume, and what did that cost?
- Why did iteration 3 of a task loop take 40 seconds when iteration 1 took 4?
- Which sub-agent in a team is responsible for most of the token spend?

The root problem is that observability exists for *team agents* but is absent or
incomplete in three key scenarios:

**1. Standalone DeepAgents** — A `DeepAgent` running outside a team (the most common
developer use case) produces zero traces. `ObservabilityRail` hard-returns when
`team_span is None`, making the entire rail a no-op. Developers have no visibility into
what their agent actually did during a multi-iteration task loop.

**2. Cross-iteration sessions** — When a DeepAgent runs multiple iterations, each
produces an isolated trace. There is no root span that groups all iterations under a
single session, so a single user interaction appears as N disconnected traces in Langfuse
instead of one coherent view.

**3. Token and cost accounting** — Token usage is visible on individual LLM spans, but
there is no aggregated count per task, per agent, per session, or per team member. There
is no cost estimation. When a workflow costs $2 to run, nothing reports that.

These gaps mean that debugging and cost attribution in production require manual log
inspection rather than structured observability tooling.

### WHEN

In-progress feature on branch `feature-observability-enhancement`.
Phase 1 was completed on 2026-07-05. Phases 2–5 are the remaining scope.

### WHAT

The feature closes all five gaps through five independently shippable phases. All changes
are confined to `openjiuwen/agent_teams/observability/` and `openjiuwen/harness/factory.py`.
The existing `extensions/tracer_otel/` system for single-invoke agents is unchanged.

---

**Component 1 — Standalone DeepAgent OTel (Phase 1)**

Makes `ObservabilityRail` work for a `DeepAgent` that is not inside a team.

When `team_span is None`, the rail now creates a `SpanKind.SERVER` root span per
iteration using an empty `OtelContext()` so each iteration starts a fully independent
trace. The iteration span nests under it. All LLM and tool child spans already nest
under the iteration span via `get_current_agent_span()` — no changes needed in
`callback_handler.py`.

Also adds `observability_config` to `create_deep_agent()` so callers wire observability
from the factory in one line, without manually calling `init_observability()` or manually
prepending `ObservabilityRail()` to the rails list.

Resulting span tree for a standalone DeepAgent:

```
agent.{name}                      SERVER  (root, one per iteration)
└── agent.{name}.task_iteration.N INTERNAL
    ├── llm.call                  GENERATION
    └── tool.{name}               TOOL
```

**Status: Completed. Committed as `98f1ab2c`.**

---

**Component 2 — Session-Level Spans (Phase 2)**

Groups all iterations of a session under a single long-lived `SpanKind.SERVER` session
span, so a user's entire interaction appears as one trace in Langfuse.

A new `SessionSpanManager` opens a session span on `SessionEvents.AGENT_SESSION_CREATED`
and closes it on session end. The iteration span from Phase 1 (or the team span, for team
agents) becomes a child of the session span. For standalone DeepAgents, this replaces
the per-iteration root spans created in Phase 1.

New semconv attributes: `SESSION_TOTAL_TOKENS`, `SESSION_TOTAL_COST_USD`,
`SESSION_AGENT_COUNT`.

---

**Component 3 — Task Trajectory Tracking (Phase 3)**

Emits the ordered sequence of reasoning steps within one iteration as span events on the
agent iteration span. Enables answering "what exactly happened in this iteration, in order"
from a single span view in Langfuse.

Step records (LLM call, tool call, decision) are accumulated per iteration in a ContextVar
inside `OtelCallbackHandler`, then flushed as OTel span events in `after_task_iteration`.
Reuses `agent_evolving/trajectory/types.py` as the data model — no new data structures.

New semconv: `TRAJ_STEP_KIND`, `TRAJ_STEP_INDEX`, `TRAJ_TOOL_NAME`, `TRAJ_LLM_MODEL`,
`TRAJ_DURATION_MS`.

---

**Component 4 — Metrics Infrastructure (Phase 4)**

Adds OTel Metrics (`MeterProvider`) alongside traces, making token usage, latency, and
error rates queryable as time-series metrics in Grafana.

A `MetricsRecorder` singleton wraps the `MeterProvider`. It records:

| Instrument | Type | Labels |
|---|---|---|
| `gen_ai.token.usage` | Counter | model, agent_id |
| `gen_ai.llm.latency` | Histogram (ms) | model |
| `agent.tool.latency` | Histogram (ms) | tool_name |
| `agent.iteration.error` | Counter | agent_id |

Gated behind `config.metrics_enabled: bool = False` — opt-in only.

---

**Component 5 — Multi-Agent Task Aggregation + Cost Tracking (Phase 5)**

For a team task, produces a summary of each sub-agent's contribution (tokens, tool calls,
duration) as span attributes on the task span. At session close, emits an estimated
`session.estimated_cost_usd` attribute using a built-in pricing table.

Two new modules:

- `TaskAggregator` — listens to `TeamEvent.TASK_*`, accumulates per-member stats, emits
  aggregated attributes on `TASK_COMPLETED`.
- `CostTracker` — accumulates token usage per session using `PRICING_TABLE`
  (input/output price per 1K tokens for common models); emits cost on session close.

New semconv: `AT_TASK_TOTAL_TOKENS`, `AT_TASK_TOTAL_COST_USD`,
`AT_TASK_MEMBER_CONTRIBUTIONS`, `SESSION_ESTIMATED_COST_USD`.

---

### Requirement Type

☑ **Functionality** (new tracing paths, new metrics instruments, new aggregation logic)
☑ **Operation and Maintenance** (observability and cost visibility for production agents)

---

## Needs Assessment

### Constraints

**Two OTel systems must remain separate:**
`agent_teams/observability/` uses `set_tracer_provider()` globally and is designed for
DeepAgents and team agents. `extensions/tracer_otel/` has its own `TracerProvider` that
deliberately does NOT call `set_tracer_provider()` to avoid conflict. Merging them is
out of scope; both systems must coexist. Code changes must not call `set_tracer_provider`
from `extensions/`.

**`ObservabilityRail` only applies to DeepAgent task loops:**
`ObservabilityRail` is a `DeepAgentRail`. It does not cover single-invoke agents (those
use `extensions/tracer_otel/OtelRail`, an `AgentRail`). Phase 1's factory wiring
(`observability_config` in `create_deep_agent()`) applies only when `enable_task_loop=True`.

**Phase 1 root spans are per-iteration, not per-session:**
Until Phase 2 ships, each DeepAgent iteration produces its own independent trace. In
Langfuse this appears as N separate traces for one user interaction. This is a known
limitation of Phase 1 alone; it is resolved by Phase 2.

**Metrics require collector infrastructure (Phase 4):**
The `MeterProvider` and OTLP metrics export require an OTel Collector to be running and
configured with a metrics pipeline. The Collector configuration (`otel-collector-config.yaml`)
is updated as part of Phase 4, but deploying the Collector is outside the scope of this
requirement. Metrics are disabled by default (`metrics_enabled=False`) until the
infrastructure is in place.

**Cost tracking uses a static pricing table:**
`CostTracker.PRICING_TABLE` is a dict of known models and their prices per 1K tokens.
It is not fetched from the model provider at runtime. Prices may drift as providers
adjust pricing. The table must be updated manually when pricing changes.

**OTel API/SDK version mismatch (pre-existing, resolved in Phase 1):**
`opentelemetry-api 1.39.1` was installed alongside `opentelemetry-sdk 1.42.1`. The SDK
referenced `TraceFlags.RANDOM_TRACE_ID` not present in API 1.39. This silently broke all
14 pre-existing observability tests via a caught exception. A compat shim was added to
`setup.py` in Phase 1. The permanent fix is to align the API and SDK versions in
`pyproject.toml`.

### Impact of Requirement Implementation on Existing Systems

**`ObservabilityRail` — behavior change, additive only:**
Standalone DeepAgents previously received no spans. They now receive a root span and an
iteration span per iteration. Team agent behavior is unchanged: the new standalone code
path is gated on `team_span is None`, which is never true for team agents.

**`create_deep_agent()` — new optional parameter:**
`observability_config: Optional[Any] = None` is added with a default of `None`.
All existing callers that do not pass this parameter are unaffected.

**`ObservabilityConfig` — new optional fields (Phase 4):**
`metrics_enabled: bool = False` and `metrics_endpoint: str = ""` are added with defaults.
Existing configs that do not set these fields continue to work as before.

**Existing observability tests:**
All 14 pre-existing observability unit tests now pass (they were silently broken by the
OTel API/SDK mismatch; the Phase 1 compat shim restored them). The 8 new Phase 1 tests
bring the total to 24. Each subsequent phase will add tests without removing existing ones.

**`extensions/tracer_otel/`:**
Not touched. Its `TracerProvider` and `OtelRail` are unchanged.

**`deploy/observability/`:**
Phase 4 adds a metrics pipeline to `otel-collector-config.yaml`. The traces pipeline is
unchanged. Existing deployments that do not enable metrics are unaffected.

### External Dependencies

| Dependency | Required for | Notes |
|---|---|---|
| `opentelemetry-api` | All phases | Already in `pyproject.toml`. Version should be aligned with `opentelemetry-sdk`. |
| `opentelemetry-sdk` | All phases | Already in `pyproject.toml`. Current version: 1.42.1. |
| `opentelemetry-exporter-otlp` | All phases | Already in `pyproject.toml`. Exports spans to Langfuse / collector. |
| Langfuse | Phases 1–3 (traces) | Already configured for team agents. No new setup required. |
| OTel Collector | Phase 4 (metrics) | Must be deployed and configured with a metrics pipeline before metrics are usable. |
| Grafana | Phase 4 (metrics dashboards) | Optional. Required only if metric dashboards are needed. |
| `opentelemetry-sdk` metrics API | Phase 4 | `MeterProvider`, `Counter`, `Histogram` — already available in `opentelemetry-sdk 1.42.1`. |
