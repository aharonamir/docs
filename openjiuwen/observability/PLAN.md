# Observability Enhancement Plan

**Last updated:** 2026-08-09  
**Status:** current-state audit and revised roadmap

---

## Current Repository Status

The repository already has substantial tracing support. The old plan and
`observability_architechture.html` are useful background, but they are stale in
several places.

### What Exists

1. **Team-agent OpenTelemetry tracing**
   - Source: `openjiuwen/agent_teams/observability/`
   - Entrypoints: `init_observability()`, `shutdown_observability()`,
     `finalize_team_trace()`, `attach_to_team_agent()`
   - Exports via OTLP/gRPC, OTLP/HTTP, console, and the current file exporter.
   - Uses the global `Runner.callback_framework` for LLM/tool callbacks.
   - Uses `ActiveSpanTracker` as a `SpanProcessor` to keep strong references to
     open spans and clean them up on cancellation/finalization.
   - Team traces include:
     - `team.{name}` root spans
     - `agent.{member}.task_iteration.N` spans
     - `agent.{member}.invoke` spans for single-round subagents
     - `llm.call` spans with prompt/completion, token usage, TTFT, reasoning
       timing, tool-call metadata, and Langfuse-compatible attributes
     - `tool.{name}` spans
     - task/member/message spans from the team monitor handler

2. **Standalone BaseAgent/workflow OTel extension**
   - Source: `openjiuwen/extensions/tracer_otel/`
   - Uses `core.session.tracer`, not `Runner.callback_framework`.
   - Creates its own `TracerProvider` and deliberately does not call
     `trace.set_tracer_provider()`, avoiding conflict with team observability.
   - Covers single-invocation Agent/Workflow tracing through
     `OtelAgentHandler`, `OtelWorkflowHandler`, span managers, and `OtelRail`.

3. **Backend/runtime bridge tracing**
   - Source: `openjiuwen/agent_teams/observability/claude/` and
     `openjiuwen/agent_teams/observability/codex/`
   - Claude bridge creates turn/tool/reasoning spans from Claude SDK stream
     chunks when team observability is initialized.
   - Codex support includes:
     - rollout trace reader for exact request/response/usage/tool payloads
     - loopback OTLP/HTTP receiver that filters Codex native spans down to
       logical model-call spans
     - bridge logic that correlates model and SDK tool records into Jiuwen
       spans

4. **Local deployment support**
   - Source: `deploy/observability/`
   - Includes Langfuse v3 docker-compose stack, OTel collector config,
     README, and a trace-upload helper.

### Important Corrections To The Old Plan

- `ObservabilityRail` is no longer the simple early-return implementation
  described in the old plan. It now has an `AgentSpanScope`, nested invoke-span
  support, stale-span cleanup, and subagent dispatch parenting.
- The main standalone DeepAgent gap still exists: both `before_task_iteration`
  and `before_invoke` skip span creation when `get_team_span()` returns `None`.
- `create_deep_agent()` and `resolve_deep_agent_parts()` do not accept
  `observability_config` today and do not auto-initialize or auto-attach
  `ObservabilityRail`.
- `SessionEvents.AGENT_SESSION_CREATED` exists and is triggered from
  `AgentSession.pre_run()` and the session controller. There is no matching
  session-close callback event in the callback event enum, so session root spans
  need a close/finalization strategy rather than assuming a close event exists.
- There is no OTel metrics infrastructure in the repository: no
  `MeterProvider`, no metrics config fields, and no metrics pipeline in the
  deploy stack.
- Token usage is recorded on spans, but there is no pricing/cost model.
- Ordered trajectory data exists elsewhere under `agent_evolving`, but team
  observability does not currently emit a per-iteration ordered LLM/tool step
  sequence.

---

## Remaining Gaps

| Gap | Current evidence | What can be added now |
| --- | --- | --- |
| Standalone DeepAgent tracing | `ObservabilityRail` requires a team span; factory does not wire observability. | Add a standalone root span path and factory opt-in. |
| Session root spans | `AGENT_SESSION_CREATED` exists, but no close event exists in `SessionEvents`. | Add a session span manager and close spans from known finalizers/shutdown paths; add a close event only if a lifecycle owner is identified. |
| Ordered task trajectory | LLM/tool spans exist, but no ordered step list is emitted on iteration spans. | Add a small tracker that records close-time LLM/tool facts as span events. |
| Metrics | No `MeterProvider`, config, instruments, or collector metrics pipeline. | Add OTel metrics behind disabled-by-default config fields. |
| Cost tracking | Usage attributes exist; no price table or totals. | Add best-effort cost estimator with explicit model-price versioning and unknown-model behavior. |
| Multi-agent aggregation | Task spans exist, but no member contribution totals. | Aggregate known LLM/tool facts by task/session/member, then stamp task/team/session spans. |
| Documentation drift | HTML plan and markdown plan describe older implementation details. | Treat this markdown as source of truth; regenerate/update HTML separately if it is still used. |

---

## Recommended Roadmap

### Phase 0 — Audit Tests And Freeze The Current Contract

Before adding behavior, add or update tests that document the current tracing
contract:

- team trace parent tree stays intact
- subagent invoke spans still nest correctly
- cancellation cleanup still drains orphaned child spans
- Codex/Claude bridge tests continue to pass
- shutdown remains idempotent

This matters because `ObservabilityRail` has accumulated real lifecycle logic.
The standalone fallback should not simplify or bypass `AgentSpanScope`.

### Phase 1 — Standalone DeepAgent OTel

**Goal:** make a standalone `DeepAgent` produce a trace when explicitly opted in.

Add:

- `observability_config: ObservabilityConfig | None = None` to
  `resolve_deep_agent_parts()` and `create_deep_agent()`.
- Factory wiring that calls `init_observability()` when needed and appends
  `ObservabilityRail()` unless an equivalent rail was supplied.
- A standalone root span path in `ObservabilityRail` when there is no team span.
  Reuse `AgentSpanScope`; do not add a separate bare-span lifecycle model.
- LLM/tool parent resolution that can use the current standalone agent/root span
  instead of returning `None`.

Expected standalone tree:

```text
agent.{card.name}
  └── agent.{card.name}.task_iteration.N
      ├── llm.call
      └── tool.{name}
```

Tests:

- standalone task-loop DeepAgent emits root, iteration, LLM, and tool spans
- single-round standalone agent behavior is explicit: either supported with
  `agent.{card.name}.invoke` or documented as out of scope
- team-agent traces are unchanged
- `shutdown_observability()` closes standalone roots and child spans

### Phase 2 — Session Root Spans

**Goal:** group activity for the same session under one trace when a reliable
session id exists.

Add:

- `session_span.py` with a `SessionSpanManager`
- open span on `SessionEvents.AGENT_SESSION_CREATED`
- parent resolution that prefers an open session span before creating team or
  standalone roots
- explicit cleanup from `shutdown_observability()` and team/agent finalization
  paths

Be honest about the constraint: there is currently no `SESSION_CLOSED` or
`AGENT_SESSION_CLOSED` event in `openjiuwen/core/runner/callback/events.py`.
Either add such an event at the real lifecycle owner or make session spans
bounded by observability shutdown/finalization. Do not leave long-lived session
spans open indefinitely.

### Phase 3 — Ordered Trajectory Events

**Goal:** make each iteration span self-describing without requiring users to
mentally reconstruct sibling span order.

Add:

- `trajectory_tracker.py`
- close-time `record_llm_step()` and `record_tool_step()` calls from
  `OtelCallbackHandler`
- `emit_to_span()` from `ObservabilityRail.after_task_iteration()` before the
  iteration span ends

Start with span events rather than a new durable store:

```text
agent.{member}.task_iteration.2
  events:
    trajectory.step {seq: 1, kind: "llm", model: "...", prompt_tokens: 123}
    trajectory.step {seq: 2, kind: "tool", name: "bash", duration_ms: 812}
```

Use the existing `agent_evolving.trajectory` types only if they fit cleanly.
Avoid coupling the tracing hot path to training/evolution storage.

### Phase 4 — OTel Metrics

**Goal:** expose operational metrics for dashboards and alerts.

Add:

- `metrics_enabled: bool = False`
- `metrics_endpoint: str = ""`
- `metrics_exporter: Literal["otlp_grpc", "otlp_http", "console"]`
- `metrics.py` with a `MeterProvider` and a small recorder API
- collector metrics pipeline and Prometheus/Grafana wiring in
  `deploy/observability/`

Initial instruments:

| Instrument | Type | Labels |
| --- | --- | --- |
| `llm.token_usage` | Counter | `model`, `agent_id`, `kind` |
| `llm.call.duration` | Histogram | `model`, `agent_id` |
| `tool.call.duration` | Histogram | `tool_name`, `agent_id` |
| `tool.call.errors` | Counter | `tool_name`, `agent_id` |
| `deepagent.task.iteration.duration` | Histogram | `agent_id`, `team_id` |
| `deepagent.task.iteration.errors` | Counter | `agent_id`, `team_id` |

Keep metrics disabled by default so tracing-only users do not get extra
provider/exporter side effects.

### Phase 5 — Cost And Aggregation

**Goal:** add useful summaries without pretending cost is exact.

Add:

- `cost_tracker.py` with a versioned model pricing table
- unknown-model behavior: set token totals but omit cost or mark cost as
  estimated/unknown
- task/member aggregation for prompt tokens, completion tokens, tool calls,
  errors, and duration
- task/team/session attributes for totals

Example task attributes:

```text
agentteam.task.total_prompt_tokens
agentteam.task.total_completion_tokens
agentteam.task.total_tool_calls
agentteam.task.estimated_cost_usd
agentteam.task.cost_pricing_version
agentteam.task.member_contributions
```

Do not hard-code volatile model names as if they are stable. Keep pricing
configuration overrideable so users can update prices without changing code.

---

## What We Can Add To The Current State

The highest-value, lowest-risk additions are:

1. **Standalone DeepAgent tracing opt-in.** This closes the largest coverage gap
   and reuses most existing code.
2. **Metrics behind a disabled-by-default config.** This adds operational value
   without affecting trace users by default.
3. **Trajectory span events.** This is small, useful, and avoids creating a new
   persistence layer.
4. **Cost estimates with explicit uncertainty.** Useful if model pricing is
   configurable and unknown models are handled honestly.

The riskiest addition is **session root spans**, not because the concept is bad,
but because the repo has a creation event without a symmetric close event. Do
this only with a clear lifecycle owner and tests proving spans close under
normal completion, cancellation, and shutdown.

---

## Verification Commands

Targeted observability tests:

```bash
make test TESTFLAGS="tests/unit_tests/agent_teams/observability/"
```

Broader callback/session regression checks:

```bash
make test TESTFLAGS="tests/unit_tests/core/runner/callback tests/unit_tests/core/session tests/unit_tests/harness"
```

Style/type checks after staging:

```bash
make check
make type-check
```
