# Jiuwen Observability — Unified Requirements Analysis (RAT)

**Two lenses, two audiences:** OTel for developers, TraceHound for users.

> This document merges the OpenJiuwen observability enhancement (`RAT.md` +
> `observability_architechture.html`) and the JiuwenSwarm TraceHound viewer into a
> single requirements analysis. The two systems solve overlapping problems for
> **different people** and are documented here as two complementary layers of one
> observability capability — not as one converged subsystem.

---

## 1. Source of Demand

- **Proactive Planning** — Engineering Quality / Operational Excellence (OTel)
- **Product Requirements** — Agent Debugging & Cost Visibility (OTel)
- **Product Requirements** — End-User Session Transparency (TraceHound)

---

## 2. Two Audiences, Two Layers

This is the organizing principle of the whole document. The two systems are not two
implementations of the same thing — they are two different products for two different
audiences, fed by the same underlying agent activity.

| | **OTel (OpenJiuwen)** | **TraceHound (JiuwenSwarm)** |
|---|---|---|
| **Audience** | **Developers** — engineers who build, run, and debug agents | **Users** — people who run an agent and want to see what it did |
| **Question it answers** | *"Why did iteration 3 take 40s?"* / *"Which sub-agent burns the most tokens?"* / *"What's my p95 LLM latency?"* | *"What did my agent do?"* / *"Did it succeed?"* / *"What went wrong, and what should I do about it?"* |
| **Lens** | **Forensic + operational** — raw, granular, aggregate across sessions | **Narrative + actionable** — plain-language, turn-by-turn, single session |
| **Data source** | Instrumentation → OTLP push | Agent's own `history.jsonl` (zero instrumentation) |
| **Transport / storage** | OTLP → Langfuse / Prometheus (ClickHouse/PostgreSQL) | Local history file + `metadata.json` |
| **Surface** | Langfuse trace viewer, Grafana dashboards | Built into the JiuwenSwarm web UI (left sidebar) |
| **Unit of analysis** | span / trace / session root | **turn** (`request_id`) |
| **Primary output** | Traces, metrics, cost attributes | Session/turn replay, outcome + error classification, LLM analysis |

**The relationship:** a developer uses OTel to instrument and operate agents at scale;
a user uses TraceHound to understand a single session. The same events underlie both —
OTel captures them through instrumentation, TraceHound reads the history file the agent
writes anyway. Each layer is optimized for its audience and must not be forced into the
other's shape.

---

## 3. Demand Background

### WHY

Agents and multi-agent teams run across hundreds of sessions, but once deployed they are
opaque. Two groups are underserved, each in a different way:

- **Developers** cannot answer operational questions — what an agent did, how many tokens
  it consumed and at what cost, why one iteration was 10× slower than another, which
  sub-agent dominates token spend. For them, observability exists *only for team agents*
  and is absent for the most common case: a standalone `DeepAgent`.
- **Users** cannot see what their own agent did in a past session, whether it succeeded or
  failed, or what to do next. They need a plain-language view, not a span dump.

### WHEN

- OpenJiuwen OTel: in-progress on branch `feature-observability-enhancement`. Phase 1
  completed 2026-07-05 (commit `98f1ab2c`); Phases 2–5 remain.
- TraceHound: shipped, built into the JiuwenSwarm web server (documented 2026-07-12).

### WHAT

A unified observability capability delivered through two layers:

1. **OTel — the developer layer.** Close the tracing/metrics/cost gaps for standalone
   DeepAgents and teams through five shippable phases, all confined to
   `openjiuwen/agent_teams/observability/` and `openjiuwen/harness/factory.py`.
   The existing `extensions/tracer_otel/` single-invoke system is unchanged.
2. **TraceHound — the user layer.** A session trajectory viewer and analyser already
   available in the web UI, with turn-by-turn replay, outcome/error classification, and
   LLM-powered issue analysis.

---

## 4. Layer A — OTel (Developer-Oriented)

### 4.1 Current State

Three tracing areas are already present (as of 2026-08-09):

- **Team-Agent OTel** (`agent_teams/observability/`) — wired through
  `Runner.callback_framework`. Emits team roots, per-iteration agent spans, subagent
  invoke spans, LLM/tool spans, and task/member/message spans.
- **Standalone BaseAgent/Workflow** (`extensions/tracer_otel/`) — traces single-invoke
  paths via `core.session.tracer`, with its own private `TracerProvider` that
  deliberately does **not** call `set_tracer_provider()`.
- **Backend bridges** (`claude/`, `codex/`) — Claude turn/tool/reasoning spans; Codex
  rollout-trace reading, filtered native model-call spans over loopback OTLP, and
  model/tool correlation into Jiuwen spans.

**Key design constraint to preserve:** `ActiveSpanTracker` holds strong references to
open spans and drains orphaned child spans during cancellation/finalization. Future work
extends this model rather than adding a second ad hoc span lifecycle.

### 4.2 Components (Phases)

**Phase 0 — Audit tests and freeze the current contract.** Lock down the trace tree,
subagent nesting, cancellation cleanup, bridge behavior, and shutdown idempotency before
changing span ownership. No behavior change.

**Phase 1 — Standalone DeepAgent OTel. ✅ Completed.** When `team_span is None`, the rail
now creates a `SpanKind.SERVER` root per iteration with an empty `OtelContext()`, nesting
the iteration span under it; LLM/tool spans already nest via `get_current_agent_span()`.
Adds `observability_config` to `create_deep_agent()` for one-line factory wiring.

```
agent.{name}                      SERVER  (root, one per iteration)
└── agent.{name}.task_iteration.N INTERNAL
    ├── llm.call                  GENERATION
    └── tool.{name}               TOOL
```

*Known limitation:* until Phase 2, each iteration is its own trace (N disconnected traces
per user interaction in Langfuse).

**Phase 2 — Session-level spans.** A `SessionSpanManager` opens a `SpanKind.SERVER`
session span on `AGENT_SESSION_CREATED` and closes it on session end; iteration spans
(or team spans) become its children. New semconv: `SESSION_TOTAL_TOKENS`,
`SESSION_TOTAL_COST_USD`, `SESSION_AGENT_COUNT`. Risk: requires a real close/finalization
event — do not leave spans open until process shutdown unless intended.

**Phase 3 — Ordered trajectory events.** Accumulate per-iteration step records
(LLM call, tool call, decision) in a ContextVar inside `OtelCallbackHandler`, flush as
span events in `after_task_iteration`. Reuses `agent_evolving/trajectory/types.py`.
New semconv: `TRAJ_STEP_KIND`, `TRAJ_STEP_INDEX`, `TRAJ_TOOL_NAME`, `TRAJ_LLM_MODEL`,
`TRAJ_DURATION_MS`.

**Phase 4 — Metrics infrastructure.** Opt-in `MeterProvider` + `MetricsRecorder`:

| Instrument | Type | Labels |
|---|---|---|
| `gen_ai.token.usage` | Counter | model, agent_id |
| `gen_ai.llm.latency` | Histogram (ms) | model |
| `agent.tool.latency` | Histogram (ms) | tool_name |
| `agent.iteration.error` | Counter | agent_id |

Gated behind `metrics_enabled: bool = False`; `metrics_endpoint: str = ""`. Requires an
OTel Collector with a metrics pipeline (deployment out of scope).

**Phase 5 — Multi-agent aggregation + cost.** `TaskAggregator` (per-member token/tool/
duration stats emitted on `TASK_COMPLETED`) and `CostTracker` (per-session token
accumulation against a static `PRICING_TABLE`, cost emitted on session close). New
semconv: `AT_TASK_TOTAL_TOKENS`, `AT_TASK_TOTAL_COST_USD`, `AT_TASK_MEMBER_CONTRIBUTIONS`,
`SESSION_ESTIMATED_COST_USD`.

### 4.3 Deployment Stack (Developer)

```
Agent process ── OTLP/gRPC :4317 ──> OTel Collector ──> Langfuse :3000
(opentelemetry-sdk)              (batch/truncate)      (PostgreSQL + ClickHouse)
                                     │
                                     └──> Prometheus :9090 (Phase 4) ──> Grafana :3001
```

`cd deploy/observability && docker-compose up -d`

---

## 5. Layer B — TraceHound (User-Oriented)

TraceHound is JiuwenSwarm's built-in session trajectory viewer and analyser. It inspects
any past session turn-by-turn, measures performance, diagnoses failures, and runs an
LLM-powered analysis — **without re-running the agent and without any added
instrumentation.** It is available from the **TraceHound** tab in the left sidebar; no
configuration or installation is required.

### 5.1 Core concepts

**Turn** — a single request/response cycle (`request_id` → `chat.final`), which may
contain many internal events: LLM calls, tool invocations, file reads, error recoveries,
streaming deltas.

**Event types recorded** (read from the history file, all appended in real time):
`chat.message`, `chat.tool_call`, `chat.tool_result`, `chat.final`, `chat.error`,
`chat.usage_metadata`, `chat.usage_summary`, `chat.reasoning`, `chat.delta`, `chat.file`.

**Outcome classification** (5): `completed`, `completed_with_issues`, `no_response`,
`error`, `deferred`.

**Error categories** (9): `api_auth`, `timeout`, `filesystem`, `network`, `syntax`,
`import`, `model`, `execution`, `other`.

**Query-type classification** (6): `debug`, `file_op`, `coding`, `analysis`, `question`,
`general`.

### 5.2 Three-level navigation

```
Session List → Turn List → Turn Detail
```

- **Session List** — title, agent-mode badge, last-message time, summary stats (messages,
  LLM calls, events, tokens, cache tokens, cost, avg latency, context %, models used).
- **Turn List** — session stats bar (turns, errors, tokens, LLM calls, events, cache,
  input/output cost, avg latency, avg TTFT/TPOT, max context, date range); analytics
  charts (outcome distribution, tokens/LLM-calls per turn, retry loops, duration, error
  categories, query types, top tools/skills); turn cards with a filter bar; and the LLM
  analysis panel.
- **Turn Detail** — chronological timeline of every event, with per-type rendering, a
  JSON export, and two timing annotations per card (Δ prev, elapsed).

### 5.3 LLM-powered analysis

Sends a compact session summary to a configured model and returns a priority-ranked list
of issues (1 = critical → 5 = informational), each with title, description, evidence,
impact, root cause, and recommendation. Results are cached in `localStorage` keyed on a
file-size/mtime fingerprint so cache invalidates automatically when history changes.

### 5.4 Storage & API

- **History file** — `~/.jiuwenswarm/agent/sessions/<session_id>/history.jsonl`
  (JSONL append-only; legacy `history.json` array via `JIUWENSWARM_USE_LEGACY_HISTORY_JSON=1`).
- **Metadata cache** — `metadata.json`, written after a `turns_list` request so the
  Session List renders without re-scanning full history.
- **WebSocket API** — `tracehound.turns.list`, `tracehound.turn.get`, `tracehound.analyze`
  (same connection as the rest of the JiuwenSwarm server).

---

## 6. Convergence & Shared Semantics

The layers stay separate, but they should **align on shared concepts** so a developer and
a user are describing the same underlying activity with consistent vocabulary.

| Concept | OTel (developer) | TraceHound (user) | Alignment |
|---|---|---|---|
| "One user interaction" | session root span (Phase 2) | session | same grouping intent |
| "One pass through the loop" | task iteration span / `trajectory.step` (Phase 3) | turn + its events | phase-3 steps ≈ turn events |
| Token accounting | span attributes | `usage_metadata` + per-turn totals | single source of truth |
| Cost | `CostTracker` + pricing table (Phase 5) | input/output cost | **one pricing table** |
| Failure signal | span status / error attributes | outcome + error category | map categories ↔ span attributes |

**Recommended convergence (cheap, high value):**
- Share a single pricing table between `CostTracker` and TraceHound cost rendering.
- Map TraceHound's 9 error categories onto span-level error attributes so Langfuse and
  TraceHound label failures identically.
- Keep `trajectory.step` (Phase 3) and TraceHound's turn-event model semantically
  aligned, but **do not** force them onto one data path — OTel is push/instrumented,
  TraceHound is file-based/zero-overhead.

**Not converged (deliberately):**
- Transport, storage, and UI surface remain distinct — each is built for its audience.
- TraceHound's LLM analysis and outcome classification are *user-facing* capabilities and
  have no OTel equivalent; they are not part of the developer roadmap.

---

## 7. Needs Assessment

### 7.1 Constraints

- **Two OTel systems must remain separate.** `agent_teams/observability/` uses
  `set_tracer_provider()` globally (DeepAgents + teams); `extensions/tracer_otel/` keeps
  a private `TracerProvider` and must not call `set_tracer_provider()`. Code changes must
  not set the provider from `extensions/`.
- **`ObservabilityRail` is DeepAgent-only.** It is a `DeepAgentRail`; single-invoke
  agents use `extensions/tracer_otel/OtelRail`. Phase 1 factory wiring applies only when
  `enable_task_loop=True`.
- **Phase 1 roots are per-iteration, not per-session** — resolved by Phase 2.
- **Metrics need collector infrastructure** (Phase 4); disabled by default until deployed.
- **Cost uses a static pricing table** — not fetched at runtime; must be updated manually.
- **OTel API/SDK version mismatch (pre-existing).** `opentelemetry-api 1.39.1` alongside
  `opentelemetry-sdk 1.42.1` silently broke 14 tests; a compat shim in `setup.py` was
  added in Phase 1. Permanent fix: align versions in `pyproject.toml`.
- **TraceHound imposes zero in-path overhead** — it reads files the agent already writes.
  This property must be preserved; no instrumentation may be added to the hot path for its sake.

### 7.2 Impact on existing systems

- **`ObservabilityRail`** — additive only; standalone path gated on `team_span is None`,
  never true for team agents.
- **`create_deep_agent()`** — new optional `observability_config: Optional[Any] = None`.
- **`ObservabilityConfig`** — new optional `metrics_enabled` / `metrics_endpoint` (Phase 4).
- **Existing tests** — 14 pre-existing observability tests restored; 8 new Phase 1 tests
  bring the total to 24; later phases add without removing.
- **`extensions/tracer_otel/`** — untouched.
- **`deploy/observability/`** — Phase 4 adds a metrics pipeline; traces pipeline unchanged.
- **TraceHound** — unchanged by the OTel work; it is a separate consuming system.

### 7.3 External dependencies

| Dependency | Required for | Notes |
|---|---|---|
| `opentelemetry-api` / `-sdk` | All OTel phases | Version alignment needed (API 1.39.1 vs SDK 1.42.1). |
| `opentelemetry-exporter-otlp` | All OTel phases | Exports to Langfuse / collector. |
| Langfuse | OTel Phases 1–3 (traces) | Already configured for team agents. |
| OTel Collector | OTel Phase 4 (metrics) | Must be deployed with a metrics pipeline. |
| Grafana | OTel Phase 4 (dashboards) | Optional. |
| `opentelemetry-sdk` metrics API | OTel Phase 4 | `MeterProvider`, `Counter`, `Histogram` in 1.42.1. |
| JiuwenSwarm web server | TraceHound | Built-in; no extra dependency. |
| Configured LLM | TraceHound analysis | Optional; analysis panel errors without one. |

---

## 8. Non-Goals & Open Decisions

- **Non-goal:** merging the two OTel providers, or converging OTel and TraceHound onto a
  single data path.
- **Non-goal:** re-instrumenting the agent to serve TraceHound; its zero-overhead,
  file-based model is a requirement, not an implementation detail.
- **Open:** where the session span closes (Phase 2 needs a real lifecycle-close event).
- **Open:** whether the shared pricing table lives in `agent_teams/observability/` or in a
  shared package both systems can import.

---

*Document version: v1.0 (unified)*
*Last updated: 2026-08-30*
