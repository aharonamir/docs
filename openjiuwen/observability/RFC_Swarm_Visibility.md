# RFC: Swarm Visibility — Unified Observability For Developers And Users

## Summary

Swarm Visibility combines two complementary observability projects:

- **Developer visibility:** OpenTelemetry tracing, metrics, cost, and aggregation for OpenJiuwen developers and operators.
- **User visibility:** TraceHound session and turn analysis for JiuwenSwarm users.

The goal is holistic visibility over agent activity without forcing the two layers into one transport, one database, or one UI. OTel remains the operational/instrumented layer. TraceHound remains the user-facing/file-based layer.

TraceHound is independently useful and independently deployable. It does **not** require the OTel enhancement, an OTel Collector, Langfuse, Prometheus, Grafana, or any span instrumentation. It reads the trajectory/history records JiuwenSwarm already writes and can provide session replay, turn analysis, outcome classification, and LLM-powered analysis even when the developer observability roadmap is not implemented.

## Problem

Agent systems are difficult to inspect after they run.

Developers need to answer operational questions:

- Which agent, model, tool, or iteration caused latency?
- Which runs failed, and where?
- How much token usage and cost came from each agent or task?
- How do standalone DeepAgents compare to team agents?

Users need to answer session questions:

- What did my agent do?
- Did it finish successfully?
- Which tools did it call?
- Why did it fail or get stuck?
- What should I do next?

Today these needs are handled by separate projects. The separation is correct, but the semantics should be aligned so a user-facing TraceHound finding can map cleanly to developer-facing OTel traces.

## Current State

### OTel Developer Layer

Existing:

- Team-agent tracing in `openjiuwen/agent_teams/observability/`.
- Standalone BaseAgent/workflow tracing in `openjiuwen/extensions/tracer_otel/`.
- Claude and Codex bridge support under `openjiuwen/agent_teams/observability/{claude,codex}/`.
- Langfuse and OTel Collector deployment support in `deploy/observability/`.

In progress:

- Standalone DeepAgent OTel opt-in.
- Session-keyed standalone roots to avoid concurrency bugs and stale context parenting.

Missing or incomplete:

- Reliable session lifecycle close contract.
- Ordered per-iteration trajectory events.
- OTel metrics.
- Cost model and multi-agent aggregation.

### TraceHound User Layer

Existing:

- Session list, turn list, and turn detail views.
- Reads `~/.jiuwenswarm/agent/sessions/<session_id>/history.jsonl`.
- Groups turns by `request_id`.
- Classifies outcomes, error categories, and query types.
- Shows token, latency, cost, model, tool, and event statistics.
- Provides LLM-powered analysis with browser-side cache invalidation.

Required preservation:

- TraceHound must remain zero additional hot-path instrumentation. It reads history files the agent already writes.
- TraceHound must continue to work independently from OTel by consuming the existing trajectory/history data path.

## Proposed Architecture

```text
Agent runtime activity
  |
  |-- Developer layer: OpenTelemetry
  |     Runner.callback_framework / DeepAgentRail / backend bridges
  |     -> spans, traces, metrics, cost attributes
  |     -> OTLP Collector -> Langfuse / Prometheus / Grafana
  |
  |-- User layer: TraceHound
  |     history.jsonl + metadata.json
  |     -> sessions, turns, timeline, outcomes, LLM analysis
  |     -> JiuwenSwarm web UI
  |
  `-- Shared semantics
        session_id, request_id, agent_id, model, tool, token usage,
        cost pricing version, error category, trajectory step
```

The layers should share vocabulary and identifiers, not runtime plumbing.

TraceHound is therefore not blocked by OTel phases 0-6. Those phases improve developer visibility and later correlation, while TraceHound can continue operating from trajectory/history data alone.

## Milestones And Phases

### Milestone 1 — Developer Visibility Baseline

**Phase 0: Freeze current OTel contract**

- Lock down team trace parent tree.
- Lock down subagent invoke nesting.
- Lock down cancellation cleanup and shutdown idempotency.
- Keep Claude/Codex bridge behavior covered.

**Phase 1: Standalone DeepAgent OTel**

- Add explicit `observability_config` opt-in.
- Auto-attach `ObservabilityRail` when appropriate.
- Add standalone root spans using the existing rail lifecycle.
- Ensure LLM/tool spans parent under standalone spans.
- Add session-keyed root registration for concurrency safety.

Acceptance:

- Standalone task-loop DeepAgent emits root, iteration, LLM, and tool spans.
- Single-round standalone invoke spans are supported or explicitly documented.
- Team traces remain unchanged.
- Consecutive and concurrent standalone runs produce independent traces.

### Milestone 2 — Session And Trajectory Semantics

**Phase 2: Session-level spans**

- Add session span management only with a clear close/finalization strategy.
- Prefer session roots when a reliable `session_id` exists.
- Stamp OTel spans with `session_id` and Langfuse session id.
- Close spans from known finalizers or a real session-close lifecycle event.

Open decision:

- Where exactly does a session span close? Do not leave long-lived session spans open indefinitely unless that is the explicit lifecycle contract.

**Phase 3: Ordered trajectory events**

- Record close-time LLM/tool facts.
- Emit ordered `trajectory.step` events on iteration spans.
- Align event fields with TraceHound turn records where practical.
- Keep this as span events first; do not introduce a durable trajectory store unless needed.

Acceptance:

- A developer can inspect one iteration span and see an ordered compact sequence.
- TraceHound turn event vocabulary and OTel trajectory vocabulary use compatible names for model, tool, duration, token, and error facts.

### Milestone 3 — Operational Metrics And Cost

**Phase 4: OTel metrics**

- Add disabled-by-default metrics config.
- Add `MeterProvider` setup and recorder API.
- Export token counters, LLM latency histograms, tool latency/error metrics, and iteration duration/error metrics.
- Add collector metrics pipeline and optional Prometheus/Grafana wiring.

Acceptance:

- Metrics are opt-in.
- Trace-only users do not get new provider/exporter side effects.
- Deployment docs describe how to enable dashboards.

**Phase 5: Cost and aggregation**

- Add versioned pricing table with override support.
- Handle unknown models honestly by recording token totals and omitting or marking cost as unknown.
- Aggregate prompt tokens, completion tokens, tool calls, errors, duration, and estimated cost by task/member/session.
- Share pricing semantics with TraceHound.

Acceptance:

- OTel and TraceHound use the same pricing version.
- Cost is clearly labeled as estimated.
- Multi-agent task summaries are available on task/team/session spans.

### Milestone 4 — User/Developer Correlation

**Phase 6: Cross-layer correlation**

- Preserve and expose shared identifiers where safe: `session_id`, `request_id`, trace id, span id.
- Add TraceHound links or fields that let support map a user-facing turn to developer traces.
- Map TraceHound error categories to OTel error attributes.
- Document support workflows for moving from TraceHound issue analysis to Langfuse trace inspection.

Acceptance:

- A support/debugging workflow can start in TraceHound and identify the matching OTel trace.
- Sensitive payload handling remains governed by each layer's current redaction rules.

## Non-Goals

- Do not merge `agent_teams/observability` and `extensions/tracer_otel` tracer providers.
- Do not move TraceHound onto OTLP.
- Do not require Langfuse, Prometheus, or Grafana for TraceHound.
- Do not add TraceHound-specific instrumentation to the agent hot path.
- Do not make TraceHound depend on the OTel enhancement; it must remain able to read trajectory/history records directly.
- Do not report model cost as exact when pricing is unknown or stale.

## Risks

- Session roots need a real close contract; otherwise spans may remain open too long.
- Cost tables become stale quickly; pricing must be versioned and overrideable.
- Too much semantic coupling could make TraceHound slower or OTel less precise.
- Exposing trace IDs in user UI must respect privacy and deployment boundaries.

## Initial PR Breakdown

1. **PR 1:** Phase 0 + Phase 1 standalone DeepAgent OTel and session-keyed standalone roots.
2. **PR 2:** Phase 2 session lifecycle finalization.
3. **PR 3:** Phase 3 ordered trajectory span events and shared error vocabulary.
4. **PR 4:** Phase 4 metrics provider, instruments, and collector deployment.
5. **PR 5:** Phase 5 cost table, aggregation, and TraceHound pricing alignment.
6. **PR 6:** Phase 6 TraceHound-to-OTel correlation UX/docs.

## Source Documents

- `PLAN.md`
- `TraceHound.md`
- `RAT-Unified-Observability.md`
- `observability_architechture.html`
- `swarm_visibility_architecture.html`
